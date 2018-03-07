## Crisis-rpc之四线程同步与rpc连接实现  
[<博客主页](https://jeremieastray.github.io)  
  
Crisis-rpc项目链接:https://github.com/JeremieAstray/crisis-rpc 
rpc连接即为使用各种连接方式进行网络通讯。通过不同的网络通讯来调用rpc服务。  
这里我实现了几种连接方式：bio、nio、mina、netty和http。  
这里我将针对bio进行详细描述，其它方式均是使用类似的方式进行rpc调用。  
## 一、线程同步
RpcClient为接受RpcInvocation，并且使用不同连接方式进行rpc调用的类。在动态代理一章中曾说道过，rpc返回的对象是可以进行延迟加载的，所以这里就不详细说明了，这里主要说明rpc对象调用时的等待。  
ps:如果返回类的类型为final、static、基本类型、void等，只能通过同步的方式调用rpc服务  
这里的代码描述的就是主线程获取结果时，进行等待操作，等待rpc服务线程完成后唤醒  
```
if (this.isFirst() && this.getObject() == null) {
    try {
        Object o = getCache(rpcInvocation.getServerName()).getIfPresent(rpcInvocation.getClientId());
        if (o != null) {
            this.setObject(o);
        } else {
            //等待
            synchronized (rpcInvocation) {
                rpcInvocation.wait(DEFAULT_TIMEOUT);
            }
            o = getCache(rpcInvocation.getServerName()).getIfPresent(rpcInvocation.getClientId());
        }
        if (o != null) {
            this.setObject(o);
        }
        this.setFirst(false);
    } finally {
        lockMap.remove(rpcInvocation.getClientId());
    }
}
```
```
/**
* 当不能生成代理对象时,使用同步获取rpcDto的方式
*
* @param rpcInvocation
* @return
*/
public Object getObject(RpcInvocation rpcInvocation) {
   try {
       //等待
       synchronized (rpcInvocation) {
           rpcInvocation.wait(DEFAULT_TIMEOUT);
       }
       Object returnObject = getCache(rpcInvocation.getServerName()).getIfPresent(rpcInvocation.getClientId());
       if (returnObject == null && rpcInvocation.getReturnType().isPrimitive()) {
           return this.getDefaultPrimitiveValue(Type.getType(rpcInvocation.getReturnType()).getSort());
       } else {
           return returnObject;
       }
   } catch (InterruptedException e) {
       logger.error(e.getMessage(), e);
   } finally {
       getCache(rpcInvocation.getServerName()).invalidate(rpcInvocation.getClientId());
       lockMap.remove(rpcInvocation.getClientId());
   }
   if (rpcInvocation.getReturnType().isPrimitive()) {
       return this.getDefaultPrimitiveValue(Type.getType(rpcInvocation.getReturnType()).getSort());
   } else {
       return null;
   }
}
```


客户端的RpcHandler为处理Rpc结果的类，当有效的rpc结果返回时它会把结果返回结果放到缓存中，并且唤醒主线程的rpc等待（可以设置等待时间），让主线程继续运行。
```
public class RpcHandler {
    public static void handleMessage(Object message) {
        if (message instanceof RpcResult) {
            RpcResult rpcResult = (RpcResult) message;
            if (rpcResult.getStatus() == RpcResult.Status.SUCCESS) {
                if (rpcResult.getReturnPara() != null) {
                    RpcClient.getCache(rpcResult.getServerName()).put(rpcResult.getClientId(), rpcResult.getReturnPara());
                }
                Object lock = RpcClient.lockMap.get(rpcResult.getClientId());
                if (lock != null) {
                    synchronized (lock) {
                        lock.notify();
                    }
                }
            }
        }
    }
}
```

服务端的RpcHandler  
服务端也是需要让spring预先加载bean,然后再rpc调用过程中，调用spring容器里面对应的对象

```
public static RpcResult handleMessage(Object message, ApplicationContext applicationContext) {
    RpcResult rpcResult = new RpcResult();
    if (message instanceof RpcInvocation) {
        RpcInvocation rpcInvocation = (RpcInvocation) message;
        setRpcContext(rpcInvocation);
        rpcResult.setServerName(rpcInvocation.getServerName());
        try {
            //获取需要被调用的class(applicationContext中)和方法
            Class clazz = Class.forName(rpcInvocation.getDestClazz());
            Object o1 = applicationContext.getBean(clazz);
            Method method = clazz.getMethod(rpcInvocation.getMethod(), rpcInvocation.getParamsType());
            Object result;
            //反射调用
            result = method.invoke(o1, rpcInvocation.getParams());

            //返回值
            rpcResult.setReturnPara(result);
            rpcResult.setStatus(RpcResult.Status.SUCCESS);
            rpcResult.setClientId(rpcInvocation.getClientId());
        } catch (Exception e) {
            logger.error(e.getMessage(), e);
            rpcResult.setClientId(rpcInvocation.getClientId());
            rpcResult.setStatus(RpcResult.Status.ERR0R);
            rpcResult.setReturnPara(null);
            rpcResult.setException(e);
        }
    } else {
        rpcResult.setReturnPara(null);
        rpcResult.setStatus(RpcResult.Status.ERR0R);
    }
    return rpcResult;
}
```


## 二、bio
这里的bio连接我使用了类似与数据库线程池的方式，让应用程序持有多条bio连接，程序可以异步或同步调用这些连接进行通讯。  
1、socket连接池  
连接池实现有几个要点：

 - 获取连接是要线程安全的  
 - 用完连接后需要手动释放连接，释放连接这里使用id进行标记  
 - 初始化连接池使用工厂方式  
 - 程序结束后需要释放所有的连接，因此这里面也有线程同步问题  

```
public class SocketPool<T extends PoolObject> {
    private static final Logger logger = LoggerFactory.getLogger(SocketPool.class);

    private Deque<T> poolBeanLinkedListPool = new ConcurrentLinkedDeque<>();
    private PoolBeanFactory<T> poolBeanFactory;
    private Map<Integer, T> integerTMap = new ConcurrentHashMap<>();

    //private static long timeout = 180000;//ms

    private AtomicInteger size = new AtomicInteger(0);
    private static int INITIAL_SIZE = 10;
    private static int INCREASE_SIZE = 20;
    private static int MAX_SIZE = 20;
    private Lock lock = new ReentrantLock();


    public SocketPool(PoolBeanFactory<T> poolBeanFactory) {
        if (poolBeanFactory == null) {
            throw new NullPointerException("poolBeanFactory should now be null!");
        }
        this.poolBeanFactory = poolBeanFactory;
        this.newPoolBeans(INITIAL_SIZE);
    }


    private void newPoolBeans(int size) {
        try {
            this.lock.lock();
            this.size.addAndGet(size);
            for (int i = 0; i < size; i++) {
                T connection = this.poolBeanFactory.init();
                this.poolBeanLinkedListPool.add(connection);
                this.integerTMap.put(connection.getId(), connection);
                new Thread(connection).start();
            }
        } finally {
            this.lock.unlock();
        }
    }

    public synchronized T getConnection() {
        if (this.poolBeanLinkedListPool.isEmpty()) {
            if (this.size.get() < MAX_SIZE) {
                increasePoolSize();
            } else {
                while (this.poolBeanLinkedListPool.isEmpty()) {
                    //暂时设定等待一会
                    try {
                        Thread.sleep(50);
                    } catch (InterruptedException e) {
                        logger.error(e.getMessage(), e);
                    }
                }
            }
        }
        //取出连接池中一个连接
        return this.poolBeanLinkedListPool.removeFirst(); // 删除第一个连接返回
    }

    //将连接放回连接池
    public void releaseConnection(int id) {
        if (this.integerTMap.get(id) != null) {
            //System.out.println("释放连接 id：" + id);
            this.poolBeanLinkedListPool.add(this.integerTMap.get(id));
        }
    }

    public void removeConnection(int id) {
        if (this.integerTMap.get(id) != null) {
            this.poolBeanLinkedListPool.remove(this.integerTMap.get(id));
            this.integerTMap.remove(id);
        }
    }

    private void increasePoolSize() {
        //System.out.println("init size : " + Math.min(MAX_SIZE - this.size.get(), INCREASE_SIZE));
        this.newPoolBeans(Math.min(MAX_SIZE - this.size.get(), INCREASE_SIZE));
    }

    public void killConnection() {
        try {
            this.lock.lock();
            this.integerTMap.values().forEach(PoolObject::killConnection);
            this.integerTMap.clear();
            this.poolBeanLinkedListPool.clear();
        } finally {
            this.lock.unlock();
        }
    }
}
```
#### socket线程  
每个线程就是一条连接，初始化后放置与连接池当中，有一个id用于标识。  
调用时，调用handleObject方法接受RpcInvocation来调用rpc服务。  
如果没有使用该线线程，该线程会处于线程等待：this.currentThread.wait();  
ps:这里面要注意另外一个问题:[使用Thread的wait方法与notify方法时遇到的IllegalMonitorStateException](http://blog.csdn.net/a549058481/article/details/60780664)  

```
public class SocketBioRpcThread implements PoolObject {
    private static final Logger logger = LoggerFactory.getLogger(SocketBioRpcThread.class);

    private static AtomicInteger ID = new AtomicInteger(0);

    private int id;
    private volatile RpcInvocation rpcInvocation;
    private Socket socket = null;
    private ObjectOutputStream objectOutputStream = null;
    private ObjectInputStream objectInputStream = null;
    private volatile boolean running = true;
    private Thread currentThread;

    public SocketBioRpcThread(int port, String host) {
        this.id = ID.incrementAndGet();
        try {
            socket = new Socket(host, port);
        } catch (IOException e) {
            logger.error(e.getMessage(), e);
        }
    }

    public synchronized void handleObject(RpcInvocation rpcInvocation) {
        this.rpcInvocation = rpcInvocation;
        synchronized (this.currentThread) {
            this.currentThread.notify();
        }
    }

    @Override
    public void run() {
        try {
            this.currentThread = Thread.currentThread();
            this.objectInputStream = new ObjectInputStream(socket.getInputStream());
            this.objectOutputStream = new ObjectOutputStream(socket.getOutputStream());
            while (this.running) {
                synchronized (this.currentThread) {
                    this.currentThread.wait();
                }
                if (this.rpcInvocation != null) {
                    this.objectOutputStream.writeObject(this.rpcInvocation);
                    this.objectOutputStream.flush();
                    Object o = objectInputStream.readObject();
                    if(o instanceof  RpcEndSignal){
                        this.rpcInvocation = null;
                        this.running = false;
                        SocketBioRpcBean.socketBioRpcThreadSocketPool.releaseConnection(this.getId());
                    }else {
                        RpcHandler.handleMessage(o);
                        this.rpcInvocation = null;
                        SocketBioRpcBean.socketBioRpcThreadSocketPool.releaseConnection(this.getId());
                    }
                }
            }
        } catch (EOFException e) {
            logger.debug("socket连接结束");
        } catch (IOException | ClassNotFoundException | InterruptedException e) {
            logger.error(e.getMessage(), e);
        } finally {
            if (objectOutputStream != null) {
                try {
                    objectOutputStream.flush();
                    objectOutputStream.close();
                } catch (IOException e) {
                    logger.error(e.getMessage(), e);
                }
            }
            try {
                if (objectInputStream != null)
                    objectInputStream.close();
            } catch (IOException e) {
                logger.error(e.getMessage(), e);
            }
            try {
                if (socket != null)
                    socket.close();
            } catch (IOException e) {
                logger.error(e.getMessage(), e);
            }
            SocketBioRpcBean.socketBioRpcThreadSocketPool.removeConnection(this.getId());
        }
    }

    @Override
    public int getId() {
        return this.id;
    }

    @Override
    public void killConnection() {
        this.running = false;
        this.rpcInvocation = new RpcEndSignal();
        synchronized (this.currentThread) {
            this.currentThread.notify();
        }
    }
}
```

服务端接收bio连接，每接受一次连接将会新建一条线程处理，线程置于线程池当中

```
try {
    this.serverSocket = new ServerSocket(serverPort);
    this.runningSignal = true;
    while (this.runningSignal) {
        Socket socket = this.serverSocket.accept();
        RpcSocket rpcSocket = new RpcSocket(socket, this.applicationContext);
        this.rpcSocketList.add(rpcSocket);
        this.executor.execute(rpcSocket);
    }
} catch (IOException e) {
    logger.error(e.getMessage(), e);
} finally {
    this.executor.shutdown();
    try {
        logger.debug("关闭BioRpc服务");
        if (this.serverSocket != null) {
            this.serverSocket.close();
        }
    } catch (IOException e) {
        logger.error(e.getMessage(), e);
    }
}
```

服务端连接处理  
需要注意一点的是，客户端有可能会发出结束连接的请求，所以这里会有一个处理RpcEndSignal的特殊处理。
```
public class RpcSocket implements Runnable {

    public ObjectOutputStream objectOutputStream = null;
    public ObjectInputStream objectInputStream = null;
    private static final Logger logger = LoggerFactory.getLogger(RpcSocket.class);
    private Socket socket;
    private ApplicationContext applicationContext;
    private volatile boolean running = true;

    public RpcSocket(Socket sockek, ApplicationContext applicationContext) {
        this.socket = sockek;
        this.applicationContext = applicationContext;
    }

    @Override
    public void run() {
        InetSocketAddress remoteAddress = null;
        try {
            remoteAddress = (InetSocketAddress) socket.getRemoteSocketAddress();
            MonitorStatus.remoteHostsList.add(remoteAddress.getHostString() + ":" + remoteAddress.getPort());
            objectOutputStream = new ObjectOutputStream(this.socket.getOutputStream());
            objectInputStream = new ObjectInputStream(this.socket.getInputStream());
            while (this.running) {
                Object o = objectInputStream.readObject();
                if (o instanceof RpcEndSignal) {
                    objectOutputStream.writeObject(new RpcEndSignal());
                    objectOutputStream.flush();
                    this.closeThread();
                } else {
                    RpcHandler.setRpcContextAddress(socket.getLocalSocketAddress(), socket.getRemoteSocketAddress());
                    RpcResult rpcResult = RpcHandler.handleMessage(o, applicationContext);
                    objectOutputStream.writeObject(rpcResult);
                    objectOutputStream.flush();
                }
            }
        } catch (IOException | ClassNotFoundException e) {
            logger.error(e.getMessage(), e);
        } finally {
            if (remoteAddress != null) {
                MonitorStatus.remoteHostsList.remove(remoteAddress.getHostString() + ":" + remoteAddress.getPort());
            }
            if (objectOutputStream != null) {
                try {
                    objectOutputStream.flush();
                    objectOutputStream.close();
                } catch (IOException e) {
                    logger.error(e.getMessage(), e);
                }
            }
            try {
                if (objectInputStream != null)
                    objectInputStream.close();
            } catch (IOException e) {
                logger.error(e.getMessage(), e);
            }
            try {
                if (!socket.isClosed())
                    socket.getInputStream().close();
            } catch (IOException e) {
                logger.error(e.getMessage(), e);
            }
            try {
                logger.debug(socket.getInetAddress() + " close!");
                socket.close();
            } catch (IOException e) {
                logger.error(e.getMessage(), e);
            }

        }
    }

    public void closeThread() {
        this.running = false;
    }
}
```

## 总结：
这一部分的实现让我学习到了不少关于多线程同步之间的问题，还有客户端与服务端之间的协同的问题。  
在C/S编程当中，我们不能只考虑客户端或者服务端一边的问题，或者盲目的采用捕捉exception的方式去解决问题，我们必须争取的处理好会遇到的各种通讯情况。  

Crisis-rpc项目链接:https://github.com/JeremieAstray/crisis-rpc 