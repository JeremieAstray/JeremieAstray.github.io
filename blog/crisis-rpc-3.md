##Crisis-rpc之三传输对象与序列化
Crisis-rpc项目链接:https://github.com/JeremieAstray/crisis-rpc 
这一篇博客讲的是Crisis-rpc如何定义rpc传输的对象，参数定义，序列化的方式等。
##一、传输对象的定义
传输对象分为二个RpcInvocation和RpcResult。
RpcInvocation包含了服务的id、服务名、服务要调用的类、调用的方法、返回参数类型、调用的类型和调用的参数。  
RpcResult包含了服务的id、服务名、调用状态、返回的对象或者Exception。  
代码如下：
```
public class RpcInvocation implements Serializable {
    private String clientId;
    private String destClazz;
    private String method;
    private Class[] paramsType;
    private Object[] params;
    private Class returnType;
    private String serverName;
}
```
```
public class RpcResult implements Serializable {

    private String clientId;
    private Status status;
    private Object returnPara;
    private Exception exception;
    private String serverName;
}
```

##二、序列化工具
Crisis-rpc使用Java的序列化方式对对象进行序列化，这里拆成两种，将对象序列化为字节数组和字符串以及其反序列化。
```
public class SerializeTool {

    private static final Logger logger = LoggerFactory.getLogger(SerializeTool.class);

    //对象序列化为字符串
    public static String objectToString(Object object) {
        ByteArrayOutputStream byteArrayOutputStream = null;
        ObjectOutputStream objectOutputStream = null;
        try {
            byteArrayOutputStream = new ByteArrayOutputStream();
            objectOutputStream = new ObjectOutputStream(byteArrayOutputStream);
            objectOutputStream.writeObject(object);
            objectOutputStream.flush();
            return byteArrayOutputStream.toString("ISO-8859-1");
        } catch (IOException e) {
            logger.error("objectToString error", e);
        } finally {
            try {
                if (objectOutputStream != null)
                    objectOutputStream.close();
                if (byteArrayOutputStream != null)
                    byteArrayOutputStream.close();
            } catch (IOException e) {
                logger.error("close stream error", e);
            }
        }
        return null;
    }

    //字符串反序列化为对象
    public static Object stringToObject(String string) {
        ByteArrayInputStream byteArrayInputStream = null;
        ObjectInputStream objectInputStream = null;
        try {
            byte[] objectBytes = string.getBytes("ISO-8859-1");
            byteArrayInputStream = new ByteArrayInputStream(objectBytes);
            objectInputStream = new ObjectInputStream(byteArrayInputStream);
            return objectInputStream.readObject();
        } catch (IOException | ClassNotFoundException e) {
            logger.error("stringToObject error", e);
        } finally {
            try {
                if (objectInputStream != null)
                    objectInputStream.close();
                if (byteArrayInputStream != null)
                    byteArrayInputStream.close();
            } catch (IOException e) {
                logger.error("close stream error", e);
            }
        }
        return null;
    }

    //字节数组反序列化为对象
    public static Object byteArrayToObject(byte[] byteArray) throws EOFException {
        ByteArrayInputStream byteArrayInputStream = null;
        ObjectInputStream objectInputStream = null;
        try {
            byteArrayInputStream = new ByteArrayInputStream(byteArray);
            objectInputStream = new ObjectInputStream(byteArrayInputStream);
            return objectInputStream.readObject();
        } catch (EOFException e) {
            throw e;
        } catch (IOException | ClassNotFoundException e) {
            logger.error("byteArrayToObject error", e);
        } finally {
            try {
                if (objectInputStream != null)
                    objectInputStream.close();
                if (byteArrayInputStream != null)
                    byteArrayInputStream.close();

            } catch (IOException e) {
                logger.error("close stream error", e);
            }
        }
        return null;
    }

    //对象序列化为字节数组
    public static byte[] objectToByteArray(Object object) {
        ByteArrayOutputStream byteArrayOutputStream = null;
        ObjectOutputStream objectOutputStream = null;
        try {
            byteArrayOutputStream = new ByteArrayOutputStream();
            objectOutputStream = new ObjectOutputStream(byteArrayOutputStream);
            objectOutputStream.writeObject(object);
            objectOutputStream.flush();
            return byteArrayOutputStream.toByteArray();
        } catch (IOException e) {
            logger.error("objectToByteArray error", e);
        } finally {
            try {
                if (objectOutputStream != null)
                    objectOutputStream.close();
                if (byteArrayOutputStream != null)
                    byteArrayOutputStream.close();
            } catch (IOException e) {
                logger.error("close stream error", e);
            }
        }
        return null;
    }
}
```
##总结
这一部分我自己定义了rpc的传输对象和序列化方式。  
其实，这里写的序列化方式不是十分的完善，序列化的方式过于暴力，而且这样子序列化传输数据会有安全问题（局域网内部的话其实可以忽略），因此后面也需要继续对其进行优化。  

Crisis-rpc项目链接:https://github.com/JeremieAstray/crisis-rpc 