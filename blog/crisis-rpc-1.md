## Crisis-rpc之一动态代理
Crisis-rpc项目链接:https://github.com/JeremieAstray/crisis-rpc

Crisis-rpc中，有两个地方用到了动态代理，一个是代理rpc服务，一个是代理加载rpc返回的对象，以下是对这两个代理对象进行详细的描述：

## 一、rpc服务对象
代理要用于注入的类，代理rpc接口，动态生成rpc服务对象，并且注册到spring容器中。

```
clazzMap.forEach((serviceName, clazzList) -> clazzList.forEach(clazz -> {
    boolean isInterface = clazz.isInterface();
    if (isInterface) {
        //jdk方案 代理服务
        Object o = Proxy.newProxyInstance(clazz.getClassLoader(), new Class[]{clazz}, (proxy, method, params) -> {
            RpcInvocation rpcInvocation = new RpcInvocation();
            rpcInvocation.setServerName(serviceName);
            rpcInvocation.setClientId(UUID.randomUUID().toString());
            rpcInvocation.setDestClazz(clazz.getName());
            rpcInvocation.setParams(params);
            rpcInvocation.setMethod(method.getName());
            rpcInvocation.setParamsType(method.getParameterTypes());
            rpcInvocation.setReturnType(method.getReturnType());
            return rpcClientMap.get(serviceName).invoke(rpcInvocation);
        });

        //cglib方案
        /*Enhancer hancer = new Enhancer();
        hancer.setInterfaces(new Class[]{clazz});
        hancer.setCallback((InvocationHandler) (o, method, params) -> {
            RpcInvocation rpcDto = new RpcInvocation();
            rpcDto.setClientId(UUID.randomUUID().toString());
            rpcDto.setDestClazz(clazz.getName());
            rpcDto.setParams(params);
            rpcDto.setMethod(method.getName());
            rpcDto.setParamsType(method.getParameterTypes());
            rpcDto.setReturnType(method.getReturnType());
            return rpcClientMap.get(serviceName).invoke(rpcDto);
        });
        Object o = hancer.create();*/
        //作为单例注册到spring容器中
        beanFactory.registerSingleton(clazz.getName(), o);
    }
}));
```
应用层使用的时候，跟原来spring注入的使用方式一致，因此此框架可以做到对代码在无嵌入的时候进行操作，只需要在使用之时在spring的applicationContext.xml上面配置即可。
```
@Autowired
private UserService userService;
```


## 二、延迟加载rpc返回的对象
这里使用cglib进行动态代理，原理跟hibernate读取数据库对象的lazy loading一致。
* 当调用rpc服务后，如果没有立即使用返回的对象，就会先执行下面的代码，然后等到调用返回的对象的时候再对其进行加载使用（等待rpc请求），以此降低调用rpc服务等待的时间。
* 当然，这个功能可以配置不使用。

```
hancer.setCallback(new ProxyHandler() {
     @Override
     public Object intercept(Object obj, Method method, Object[] params, MethodProxy proxy) throws Throwable {
         try {
             //第一次调用时
             if (this.isFirst() && this.getObject() == null) {
                 try {
                     Object o = getCache(rpcInvocation.getServerName()).getIfPresent(rpcInvocation.getClientId());
                     if (o != null) {
                         this.setObject(o);
                     } else {
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
             if (this.getObject() != null) {
                 if ("finalize".equals(method.getName())) {
                     this.finalize();
                     return null;
                 }
                 return method.invoke(this.getObject(), params);
             }
             return null;
         } catch (Exception e) {
             logger.error(e.getMessage(), e);
             return null;
         } finally {
             getCache(rpcInvocation.getServerName()).invalidate(rpcInvocation.getClientId());
         }
     }
 });
```

## 总结
在这一部分的实现里面，我学到了Java里面动态代理类的方式，同时也体验了一次hibernate延迟加载的问题，以前我使用hibernate过程中遇到的延迟加载的问题在这里面也会遇到。  
另外，我也发现除了JDK代理方式也有其他诸如cglib的动态代理方式能够对Java类进行动态代理。这里面就会涉及性能问题，各种方式使用的问题，如JDK代理只能对接口进行代理，cglib则没有这个限制。  
完成这个框架的过程也是我深入研究技术的一个过程，我希望我的技术也跟随着这个框架一起进步，这个框架能够慢慢被我完善，直到有一天能够派上用场。  

Crisis-rpc项目链接:https://github.com/JeremieAstray/crisis-rpc