[<博客主页](https://jeremieastray.github.io)  
  
## Crisis-rpc之四线程同步与rpc连接实现
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
            
