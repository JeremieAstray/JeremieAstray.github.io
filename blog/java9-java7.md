## Java9正式版都已经发布了，我们还在用Java7?  
[<博客主页](https://jeremieastray.github.io)  
  
首先，我这里要恭贺Java9终于不再跳票，发布了正式版。这意味着在Java的时间线中，Java8已经成为旧时代的产品了。    

## 现在，我们来回顾一下Java8对比Java7特有的我们比较常用的特性  
  
### 1.Lambda表达式

正如Java7的泛型至于Java6，函数式编程是Java8最大的，也可以说是划时代的一个特性。使用Lambda表达式进行开发可以令代码简洁，开发快速。而且语法接近自然语言，易于理解，也让程序员更方便地进行代码管理。  
  
遍历数组  
```
//老版本java写法
List<String> strList = Arrays.asList( "a", "b", "d" );
for (String str: strList) {
    System.out.println(str);
}
//Java8写法
Arrays.asList( "a", "b", "d" ).forEach(System.out::println);
```
给数组进行排序： 
```
//老版本java写法
List<String> strList2 = Arrays.asList("a", "b", "d");
Collections.sort(strList, new Comparator<String>() {
    @Override
    public int compare(String o1, String o2) {
        return o1.compareTo(o2);
    }
});
//Java8写法
Arrays.asList( "a", "b", "d" ).sort(Comparator.naturalOrder());
```

Lambda表达式作为Java 8的最大卖点，它有潜力吸引更多的开发者加入到JVM平台，并在纯Java编程中使用函数式编程的概念。如果你需要了解更多Lambda表达式的细节，可以参考[官方文档](https://docs.oracle.com/javase/tutorial/java/javaOO/lambdaexpressions.html)。  
    
### 2.接口的默认方法与静态方法  

Java的接口可以支持实现默认方法和静态方法。	由于JVM上的默认方法的实现在字节码层面提供了支持，因此效率非常高。默认方法允许在不打破现有继承体系的基础上改进接口。  
```
private interface Defaulable {
    // Interfaces now allow default methods, the implementer may or 
    // may not implement (override) them.
    default String notRequired() { 
        return "Default implementation"; 
    }        
}
```
  
### 3.扩展注解的支持  

Java 8扩展了注解的上下文。现在几乎可以为任何东西添加注解：局部变量、泛型类、父类与接口的实现，就连方法的异常也能添加注解。  
  
```
public class Annotations {
    @Retention( RetentionPolicy.RUNTIME )
    @Target( { ElementType.TYPE_USE, ElementType.TYPE_PARAMETER } )
    public @interface NonEmpty {        
    }

    public static class Holder< @NonEmpty T > extends @NonEmpty Object {
        public void method() throws @NonEmpty Exception {            
        }
    }

    @SuppressWarnings( "unused" )
    public static void main(String[] args) {
        final Holder< String > holder = new @NonEmpty Holder< String >();        
        @NonEmpty Collection< @NonEmpty String > strings = new ArrayList<>();        
    }
}
```
  
### 4.Streams  

极大地提高了对于集合操作的简便性。Streams类结合Lambda表达式一起用30行代码可以用10行代码简明扼要地表达出来。  
  
使用流进行元素分组：  
```
//老版本java写法
List<Foo> fooList = Arrays.asList(foo1, foo2, foo3);

Map<Integer, List<Foo>> fooGroup = new HashMap<>();

for (Foo foo : fooList) {
    if (!fooGroup.containsKey(foo.getType())) {
        fooGroup.put(foo.getType(),new ArrayList<>());
    }
    fooGroup.get(foo.getType()).add(foo);
}

//Java8写法
Map<Integer,List<Foo>> fooGroup = Arrays.asList(foo1,foo2,foo3)
                                        .stream
                                        .collect(Collectors.groupingBy(Foo::getType));

class Foo {
    private int type;
    private String desc;
}
```  

数组过滤并求和：    
```
//老版本java写法
 long totalPointsOfOpenTasksTemp = 0;
 for (Foo foo : tasks) {
     if (foo != null && foo.getType() == 1) {
         int point = foo.getPoints();
         totalPointsOfOpenTasksTemp += point;
     }
 }

//Java8写法
long totalPointsOfOpenTasks = tasks
                .stream()
                .filter(Objects::nonNull)
                .filter(task -> task.getType() == 1)
                .mapToInt(Foo::getPoints)
                .sum();

```  

Stream API、Lambda表达式还有接口默认方法和静态方法支持的方法引用，是Java 8对软件开发的现代范式的响应。  

### 5.Optional类  

Optional类用于包装返回的参数，它可以有效防止空指针问题。  
```
public Optional<Foo> getFoo() {
        //some logic to create foo
        Foo foo = someLogic();
        return Optional.of(foo);
    }

public void invoker(){
    Optional<Foo> fooOptional = getFoo();
    if(fooOptional.isPresent()) {
        System.out.println("foo is valid");
    }else {
        System.err.println("foo is not valid");
    }
}
```
  
### 6.Date/Time API  

我们可以更方便地使用Java API来处理时间参数，而不去要去依赖common工具包的DateUtils了。  

```
LocalDateTime localDateTime = LocalDateTime.now();
String time = localDateTime.format(DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss"));
LocalDateTime localDateTimeTemp = LocalDateTime.parse(time,DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss"));
localDateTimeTemp.plusHours(1L);
boolean equal = localDateTimeTemp.isEqual(localDateTime);
```
    
## 那么Java9有什么新的特性值得我们去关注呢？  
  
### 1.Java 平台级模块系统  

引入了一种新的Java编程组件模块，它是一个命名的、自描述的代码和数据集合。  
    
### 2.Linking  

你可以使用jlink工具创建针对应用程序进行优化的最小运行时映像而不需要使用完全加载 JDK 安装版本。  
      
### 3.JShell  

可以从控制台启动 jshell ，并直接启动输入和执行 Java 代码。    
      
### 4.改进的 Javadoc  

Javadoc 现在支持在 API 文档中的进行搜索。另外，Javadoc 的输出现在符合兼容 HTML5 标准。    
      
### 5.集合工厂方法  

看两行代码就能看出来这个特性了(我们用的工具guava也有类似的实现)  
```
    Set<Integer> ints = Set.of(1, 2, 3);  
    List<String> strings = List.of("first", "second");  
```  
      
### 6.改进的 Stream API  

Stream 接口中添加了 4 个新的方法：dropWhile, takeWhile, ofNullable, iterate方法。  
大家可以自己使用来体验一下。    
      
### 7.私有接口方法  

Java 8 为我们带来了接口的默认方法。 接口现在也可以包含行为，而不仅仅是方法签名。 但是，如果在接口上有几个默认方法，代码几乎相同的时候，我们就需要一些私有的方法去复用这些代码，而Java9就给我们带来了这个：  
```  
public interface MyInterface {  
  
    void normalInterfaceMethod();  
   
    default void interfaceMethodWithDefault() {  init(); }  
   
    default void anotherDefaultMethod() { init(); }  
   
    // This method is not part of the public API exposed by MyInterface
    private void init() { System.out.println("Initializing"); }  
}  
```  