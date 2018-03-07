## Crisis-rpc之二spring配置初始化与获取  
[<博客主页](https://jeremieastray.github.io)  
  
Crisis-rpc项目链接:https://github.com/JeremieAstray/crisis-rpc  
Crisis-rpc中要使用spring来进行服务的初始化，以下就是描述该实现：  
## spring配置
首先要从spring配置文件说起。 
Crisis-rpc框架可以配置多个rpc服务，每个服务可以指定ip，端口，使用什么连接实现，是否延迟加载，要进行代理的包等。

ps:当程序结束前，要让程序调用destroy方法(其调用了线程池的shutdown)，使线程池结束，否则会出现程序一直结束不了的问题。

```
<bean class="com.jeremie.spring.rpc.client.proxy.RpcInitializer" init-method="init" destroy-method="destroy">
    <property name="serviceConfigList">
        <list>
            <bean id="minaServerConfig" class="com.jeremie.spring.rpc.remote.config.ServiceConfig">
                <property name="defaultIp" value="127.0.0.1"/>
                <property name="defaultPort" value="8091"/>
                <property name="method" value="mina"/>
                <property name="name" value="adol-impl"/>
                <property name="lazyLoading" value="true"/>
                <property name="loadTimeout" value="1000"/>
                <property name="packages">
                    <list>
                        <value>com.jeremie.spring.adol.service</value>
                    </list>
                </property>
            </bean>
            <bean id="httpServerConfig" class="com.jeremie.spring.rpc.remote.config.ServiceConfig">
                <property name="defaultIp" value="127.0.0.1"/>
                <property name="defaultPort" value="8095"/>
                <property name="method" value="http"/>
                <property name="name" value="home-impl"/>
                <property name="lazyLoading" value="true"/>
                <property name="loadTimeout" value="1000"/>
                <property name="packages">
                    <list>
                        <value>com.jeremie.spring.home.service</value>
                    </list>
                </property>
            </bean>
        </list>
    </property>
</bean>
```
## Java配置类

ServiceConfig配置类：
```
public class ServiceConfig implements Serializable {

    private String defaultIp;
    private int defaultPort;
    private int defaultNioClientPort;
    private boolean lazyLoading = false;
    private Long loadTimeout = 500L;

    private String name;

    private String method;
}
```

初始化rpc服务的代码
```
private void beforeInit() {
    try {
        for (ServiceConfig serviceConfig : serviceConfigList) {
            RpcClient rpcClient = RpcClientEnum.getRpcClientInstance(serviceConfig.getName(), serviceConfig.getMethod(), serviceConfig.isLazyLoading(), serviceConfig.getLoadTimeout());
            this.rpcClientMap.put(serviceConfig.getName(), rpcClient);
            if (rpcClient.getRpcBean() != null) {
                RpcBean rpcBean = rpcClient.getRpcBean();
                rpcBean.setAppName(serviceConfig.getName());
                rpcBean.setClientPort(serviceConfig.getDefaultNioClientPort());
                rpcBean.setHost(serviceConfig.getDefaultIp());
                rpcBean.setPort(serviceConfig.getDefaultPort());
                this.rpcBeanList.add(rpcClient.getRpcBean());
            }
            rpcClient.init();
        }
    } catch (Exception e) {
        logger.error(e.getMessage(), e);
    }
}
```

这个是我对于自己实现的每种连接的一个枚举类，使用该枚举类可以减少if else的使用，同时也可以清晰的看清楚我当前使用的是哪种连接。
```
public enum RpcClientEnum {

    NIO("nio", "com.jeremie.spring.rpc.remote.nio.SocketNioRpcClient", "com.jeremie.spring.rpc.remote.nio.NioRpcBean"),
    BIO("bio", "com.jeremie.spring.rpc.remote.socket.SocketBioRpcClient", "com.jeremie.spring.rpc.remote.socket.SocketBioRpcBean"),
    MINA("mina", "com.jeremie.spring.rpc.remote.mina.MinaRpcClient", "com.jeremie.spring.rpc.remote.mina.MinaRpcBean"),
    NETTY("netty", "com.jeremie.spring.rpc.remote.netty.NettyRpcClient", "com.jeremie.spring.rpc.remote.netty.NettyRpcBean"),
    HTTP("http", "com.jeremie.spring.rpc.remote.http.HttpRpcClient", "com.jeremie.spring.rpc.remote.http.HttpRpcBean");

    private String name;
    private String clientClazzName;
    private String beanClazzName;

    RpcClientEnum(String name, String clientClazzName, String beanClazzName) {
        this.name = name;
        this.clientClazzName = clientClazzName;
        this.beanClazzName = beanClazzName;
    }

    public static RpcClient getRpcClientInstance(String serverName, String clientName, boolean lazyLoading, Long cacheTimeout) throws ClassNotFoundException, IllegalAccessException, InstantiationException, NoSuchMethodException, InvocationTargetException {
        if (clientName == null) {
            throw new NullPointerException();
        }
        for (RpcClientEnum rpcClientEnum : RpcClientEnum.values()) {
            if (clientName.equals(rpcClientEnum.name)) {
                RpcClient rpcClient = (RpcClient) Class.forName(rpcClientEnum.clientClazzName).getDeclaredConstructor(String.class, Boolean.class, Long.class).newInstance(serverName, lazyLoading, cacheTimeout);
                if (!"".equals(rpcClientEnum.beanClazzName)) {
                    rpcClient.setRpcBean((RpcBean) Class.forName(rpcClientEnum.beanClazzName).newInstance());
                }
                return rpcClient;
            }
        }
        throw new ClassNotFoundException("can not case to rpc client class name " + clientName + " !");
    }

}
```

## 总结：
在这一部分的实现里面，我学习了spring配置一个bean对应bean所需要的东西，还有spring初始化bean，结束beans的流程。  
关于枚举，其实是枚举是一个非常好用的东西，只要合适的使用它，它会让你的代码更加优雅。  


Crisis-rpc项目链接:https://github.com/JeremieAstray/crisis-rpc
