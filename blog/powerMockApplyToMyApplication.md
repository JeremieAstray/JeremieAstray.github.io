## powerMock+spring+testng/junit应用(mock静态、私有方法) 

###  1、背景：  
某项目需要mock掉对redis的存储，但是在目标类中该方法是private的，mockito是做不到对private方法进行mock的。  
  
而使用powerMock的时候，如果mock static方法或者private方法  
1)junit需要在测试类加上@RunWith(PowerMockRunner.class)注解  
```
@RunWith(PowerMockRunner.class)
public class MockPrivateLogicTest {
```
2)testng需要测试类继承PowerMockTestCase类
```
public class MockPrivateLogicTest extends PowerMockTestCase {
```
  
而如果这样写之后就会跟spring加载相冲突：
1)junit如果要spring注入需要在类加上@RunWith(SpringJUnit4ClassRunner.class)  
```
@RunWith(SpringJUnit4ClassRunner.class)
public class MockPrivateLogicTest {
```
2)testng如果要spring注入需要类继承AbstractTestNGSpringContextTests类  
```
public class MockPrivateLogicTest extends AbstractTestNGSpringContextTests {
```
  
因此powerMock跟spring加载相冲突了  

### 2、解决方案：  
  
1)junit  
junit只需要加上powerMock兼容性的注解即可：@PowerMockRunnerDelegate(SpringJUnit4ClassRunner.class)  
```
//如果要mock static方法或者private方法要加上这个注解
@PrepareForTest(MockPrivateLogic.class)
//委托给spring
@PowerMockRunnerDelegate(SpringJUnit4ClassRunner.class)
//使用PowerMockRunner运行单元测试
@RunWith(PowerMockRunner.class)
public class MockPrivateLogicTest {
    @Autowired
    public MockPrivateLogic mockPrivateLogic;
    
    @Test
    public MockPrivateLogic mockMockPrivateLogic(){
        final Map<String, String> redisMap = Maps.newHashMap();
        
        //mock对象
        MockPrivateLogic MockPrivateLogic = PowerMockito.spy(this.MockPrivateLogic);
        
        //mock setCache这个私有方法
        PowerMockito.doAnswer(new Answer<Void>() {
            @Override
            public Void answer(InvocationOnMock invocationOnMock) {
                Object[] params = invocationOnMock.getArguments();
                String key = "TEMP_" + params[0];
                String value = params[1].toString();
                redisMap.put(key, value);
                return null;
            }
        }).when(MockPrivateLogic, "setCache",
                Mockito.any(String.class)
                , Mockito.any(String.class));
                
                
        //...具体实现
        //当调用MockPrivateLogic的方法调用到setCache的时候就会运行mock的代码
    }
}
```  
  
2)testng 
testng使用折中的方案：继承PowerMockTestCase，然后手动加载spring  
```
//PowerMock需要排除掉javax.management.*，否则spring加载的时候会出现ClassNotCastException
@PowerMockIgnore({"javax.management.*"})
//如果要mock static方法或者private方法要加上这个注解
@PrepareForTest(MockPrivateLogic.class)
//测试类继承PowerMockTestCase
public class MockPrivateLogicTest extends PowerMockTestCase {

    //手动加载MockPrivateLogic，不需要加@Autowire注解
    private MockPrivateLogic mockPrivateLogic;

    @BeforeClass
    public void initBaseConfig() {
        //手动加载spring
        ApplicationContext context = new ClassPathXmlApplicationContext("classpath*:META-INF/spring/*.xml");
        MockPrivateLogic = (MockPrivateLogic) context.getBean("mockPrivateLogic");
        //对注解了@Mock的对象进行模拟
        MockitoAnnotations.initMocks(this);
    }
    
    //mock代码跟junit的一致
}

```
  
ps:官方给出的解决方案是在测试类加上下面的代码，但是我运行失败  
会出现applicationContext加载失败的异常（github上面也有人直接提了这个bug了）  

```
@ContextHierarchy({
        @ContextConfiguration(locations = "classpath*:META-INF/spring/*.xml")
})
//如果要mock static方法或者private方法要加上这个注解
@PrepareForTest(MockPrivateLogic.class)
public class MockPrivateLogicTest extends AbstractTestNGSpringContextTests {

    @Autowire
    private MockPrivateLogic MockPrivateLogic;

    @ObjectFactory
    public IObjectFactory getObjectFactory() {
        return new org.powermock.modules.testng.PowerMockObjectFactory();
    }
    
    //mock代码跟junit的一致
}
```

### 3、顺手提供一下pom依赖（网上很多教程都没有。。。版本自己去中央仓库找吧，各个库的版本要相匹配哦）：   
  
junit:  
```
    <dependency>
        <groupId>junit</groupId>
        <artifactId>junit</artifactId>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.mockito</groupId>
        <artifactId>mockito-all</artifactId>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.powermock</groupId>
        <artifactId>powermock-api-mockito</artifactId>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.powermock</groupId>
        <artifactId>powermock-module-junit</artifactId>
        <scope>test</scope>
    </dependency>
```
  
testng:  
```
    <dependency>
        <groupId>org.testng</groupId>
        <artifactId>testng</artifactId>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.mockito</groupId>
        <artifactId>mockito-all</artifactId>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.powermock</groupId>
        <artifactId>powermock-api-mockito</artifactId>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.powermock</groupId>
        <artifactId>powermock-module-testng</artifactId>
        <scope>test</scope>
    </dependency>
```

