## 使用Thread的wait方法与notify方法时遇到的IllegalMonitorStateException  
[<博客主页](https://jeremieastray.github.io)  
  

以下代码是用来作同步处理使用的，当线程城运行后，this.currentThread.wait()这句会让线程等待，减少性能损耗。当我有数据要处理时调用handleObject来唤醒线程。

```
    public synchronized void handleObject(RpcInvocation rpcInvocation) {
        this.rpcInvocation = rpcInvocation;
        this.currentThread.notify();
    }

    @Override
    public void run() {
        try {
            this.currentThread = Thread.currentThread();
            while (this.running) {
                this.currentThread.wait();
                if (this.rpcInvocation != null) {
                   //处理rpcInvocation 
                }
            }
        } catch (IOException | ClassNotFoundException | InterruptedException e) {
            logger.error(e.getMessage(), e);
        } finally {
            //结束
        }
    }
```

不过这段代码运行到第三行时，会出现IllegalMonitorStateException，这是因为调用handleObject的线程不是调用wait()方法的线程，而调用handleObject的线程没有权去notify()，因此会出现IllegalMonitorStateException。  
出现这个Exception的原因有下面几个：  
1、当前线程不含有当前对象的锁资源的时候，调用obj.wait()方法;  
2、当前线程不含有当前对象的锁资源的时候，调用obj.notify()方法。  
3、当前线程不含有当前对象的锁资源的时候，调用obj.notifyAll()方法 。  

解决方案，当wait()和notify()方法都在同一个锁对象下，则能正常解锁：
```
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
            while (this.running) {
                synchronized (this.currentThread) {
                    this.currentThread.wait();
                }
                if (this.rpcInvocation != null) {
                   //处理rpcInvocation 
                }
            }
        } catch (IOException | ClassNotFoundException | InterruptedException e) {
            logger.error(e.getMessage(), e);
        } finally {
            //结束
        }
    }
```
