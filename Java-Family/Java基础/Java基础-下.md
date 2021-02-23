## 代理
### 反射 [参考](https://www.jianshu.com/p/9be58ee20dee)
JAVA反射机制是在**运行状态中**，对于任意一个类，都能够知道这个**类的所有属性和方法**；对于任意一个对象，都能够调用它的**任意方法和属性**

#### 反射机制的相关类

| 类名          | 用途                                             |
| ------------- | ------------------------------------------------ |
| Class类       | 代表类的实体，在运行的Java应用程序中表示类和接口 |
| Field类       | 代表类的成员变量（成员变量也称为类的属性）       |
| Method类      | 代表类的方法                                     |
| Constructor类 | 代表类的构造方法                                 |

#### Class类
##### 获得类相关的方法
| 方法                          | 用途                                                         |
| ----------------------------- | ------------------------------------------------------------ |
| asSubclass(Class<U> clazz)    | 把传递的类的对象转换成代表其子类的对象                       |
| Cast                          | 把对象转换成代表类或是接口的对象                             |
| getClassLoader()              | 获得类的加载器                                               |
| getClasses()                  | 返回一个数组，数组中包含该类中所有公共类和接口类的对象       |
| getDeclaredClasses()          | 返回一个数组，数组中包含该类中所有类和接口类的对象（破坏private） |
| **forName**(String className) | 根据类名返回类的对象                                         |
| getName()                     | 获得类的完整路径名字                                         |
| newInstance()                 | 创建类的实例                                                 |
| getPackage()                  | 获得类的包                                                   |
| getSimpleName()               | 获得类的名字                                                 |
| getSuperclass()               | 获得当前类继承的父类的名字                                   |
| getInterfaces()               | 获得当前类实现的类或是接口                                   |

##### 获得类中属性相关的方法
| 方法                          | 用途                            |
| ----------------------------- | ------------------------------- |
| getField(String name)         | 获得某个公有的属性对象          |
| getFields()                   | 获得所有公有的属性对象          |
| getDeclaredField(String name) | 获得某个属性对象（破坏private） |
| getDeclaredFields()           | 获得所有属性对象                |

其他方法略...


### 静态代理
```java
// 代理的目标接口
public interface HelloSerivice {
    public void say();
}

// 代理的目标对象
public class HelloSeriviceImpl implements HelloSerivice{

    @Override
    public void say() {
        System.out.println("hello world");
    }
}
```
定义**代理对象**
```java
public class HelloSeriviceProxy implements HelloSerivice{

    private HelloSerivice target;
    public HelloSeriviceProxy(HelloSerivice target) {
        this.target = target;
    }

    @Override
    public void say() {
        System.out.println("记录日志");
        target.say();
        System.out.println("清理数据");
    }
}
```
这就是一个简单的静态的代理模式的实现。代理模式中的所有角色（**代理对象、目标对象、目标对象的接口**）等都是在编译期就确定好的。

### 动态代理
Java中，实现动态代理有两种方式：

1、**JDK动态代理**：java.lang.reflect 包中的Proxy类和InvocationHandler接口提供了生成动态代理类的能力。限制是目标类必须实现一个或多个接口。

2、**Cglib动态代理**：Cglib (Code Generation Library )是一个第三方代码生成类库，运行时在内存中动态生成一个子类对象从而实现对目标对象功能的扩展。

在**Spring AOP**中，如果目标类未实现接口则采用Cglib。

代理的目标类
```java
public class UserServiceImpl implements UserService {

    @Override
    public void add() {
        // TODO Auto-generated method stub
        System.out.println("--------------------add----------------------");
    }
}
```
#### JDK动态代理
```java
public class MyInvocationHandler implements InvocationHandler {

    private Object target;

    public MyInvocationHandler(Object target) {

        super();
        this.target = target;

    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        PerformanceMonior.begin(target.getClass().getName()+"."+method.getName());
        //System.out.println("-----------------begin "+method.getName()+"-----------------");
        Object result = method.invoke(target, args);
        //System.out.println("-----------------end "+method.getName()+"-----------------");
        PerformanceMonior.end();
        return result;
    }

    public Object getProxy(){

        return Proxy.newProxyInstance(Thread.currentThread().getContextClassLoader(), target.getClass().getInterfaces(), this);
    }

}

public static void main(String[] args) {

  UserService service = new UserServiceImpl();
  MyInvocationHandler handler = new MyInvocationHandler(service);
  UserService proxy = (UserService) handler.getProxy();
  proxy.add();
}
```

#### cglib动态代理
```java
public class CglibProxy implements MethodInterceptor{
    private Enhancer enhancer = new Enhancer();
    public Object getProxy(Class clazz){
        //设置需要创建子类的类  
        enhancer.setSuperclass(clazz);
        enhancer.setCallback(this);
        //通过字节码技术动态创建子类实例  
        return enhancer.create();
    }
    //实现MethodInterceptor接口方法  
    public Object intercept(Object obj, Method method, Object[] args,
                            MethodProxy proxy) throws Throwable {
        System.out.println("前置代理");
        //通过代理类调用父类中的方法  
        Object result = proxy.invokeSuper(obj, args);
        System.out.println("后置代理");
        return result;
    }
}

public class DoCGLib {
    public static void main(String[] args) {
        CglibProxy proxy = new CglibProxy();
        //通过生成子类的方式创建代理类  
        UserServiceImpl proxyImp = (UserServiceImpl)proxy.getProxy(UserServiceImpl.class);
        proxyImp.add();
    }
}
```

cglib完全避开了java自带的反射机制带来的问题，cglib不仅仅可以代理接口，同时可以给普通类进行代理，同时它采用fastclass机制为代理类和被代理类各生成一个Class，这个Class会为代理类或被代理类的方法分配一个index(int类型)。
 这个index当做一个入参，FastClass就可以**直接定位要调用的方法直接进行调用**，这样**省去了反射调用**，所以调用效率比JDK动态代理通过反射调用高。

**注：**1->final class无法支持，因为无法继承；2->必须要有个无参的构造方法

> 在JDK 1.8以前CGLIB的性能要比动态代理高，但是自1.8以后动态代理比CGLIB更快

## 序列化与反序列化

对象序列化机制（object serialization）是Java语言内建的一种**对象持久化方式**，通过对象序列化，可以把对象的状态保存为**字节数组**，并且可以在有需要的时候将这个字节数组通过**反序列化**的方式再转换成对象。对象序列化可以很容易的在JVM中的活动对象和字节数组（流）之间进行转换。

在Java中，对象的序列化与反序列化被广泛应用到**RMI**(远程方法调用)及**网络传输**中。

### Serializable（Externalizable）
#### Serializable

通过实现 java.io.Serializable 接口以**启用其序列化功能**。未实现此接口的类将无法使其任何状态序列化或反序列化。可序列化类的**所有子类型**本身都是可序列化的。**序列化接口没有方法或字段**，仅用于标识可序列化的语义。

当试图对一个对象进行序列化的时候，如果遇到不支持 Serializable 接口的对象。在此情况下，将抛出 **NotSerializableException**。

#### Externalizable接口
Externalizable继承了Serializable，该接口中定义了两个抽象方法：**writeExternal()** 与**readExternal()**。当使用Externalizable接口来进行序列化与反序列化的时候需要开发人员**重写**writeExternal()与readExternal()方法。

>实现Externalizable接⼜的类必须要提供⼀个public的⽆参的构造器。如果没有的话会抛出异常：**java.io.InvalidClassException**

### serialVersionUID
在进⾏反序列化时， JVM会把传来的字节流中的**serialVersionUID**与本地相应实体类的serialVersionUID进⾏⽐较， **如果相同就认为是⼀致的**， 可以进⾏反序列化， 否则就会出现序列化版本不⼀致的异常**InvalidCastException**。

这样做是为了**保证安全**， 因为⽂件存储中的内容可能被篡改。

### transient
transient 关键字的作用是**控制变量的序列化**，在变量声明前加上该关键字，可以**阻止**该变量被序列化到文件中，在被反序列化后，transient 变量的值被设为初始值，如 int 型的是 0，对象型的是 null。


### 自定义序列化策略
在序列化过程中，如果被序列化的类中定义了**writeObject** 和 **readObject** 方法，虚拟机会试图调用对象类里的 writeObject 和 readObject 方法，进行用户**自定义**的序列化和反序列化。

### 破坏单例模式
1.通过对Singleton的序列化与反序列化得到的对象是一个新的对象，这就破坏了Singleton的单例性。

只要在Singleton类中定义**readResolve**就可以解决该问题：
```java
package com.hollis;
import java.io.Serializable;
/**
 * Created by hollis on 16/2/5.
 * 使用双重校验锁方式实现单例
 */
public class Singleton implements Serializable{
    private volatile static Singleton singleton;
    private Singleton (){}
    public static Singleton getSingleton() {
        if (singleton == null) {
            synchronized (Singleton.class) {
                if (singleton == null) {
                    singleton = new Singleton();
                }
            }
        }
        return singleton;
    }

    private Object readResolve() {
        return singleton;
    }
}
```
> 因为反序列化过程中，在反序列化执行过程中会执行到ObjectInputStream#readOrdinaryObject方法，这个方法会判断对象是否包含readResolve方法，如果包含的话会直接调用这个方法获得对象实例。

2.基于反射机制，也可以破坏privite，解决办法：
```java
// 反射破坏单例
Class<Singleton> singleClass = (Class<Singleton>)Class.forName("com.dev.interview.Singleton");
Constructor<Singleton> constructor = singleClass.getDeclaredConstructor(null);
constructor.setAccessible(true);
Singleton singletonByReflect = constructor.newInstance();

// 解决办法
private Singleton() {
    if (singleton != null) {
        throw new RuntimeException("Singleton constructor is called... ");
    }
}
```

## 注解
### 元注解
一共有**4**个元注解：
- **@Target**：表示该注解可以用于什么地方
- **@Retention**： 表示在什么级别保存该注解信息
    - SOURCE：注解仅存在于源码中，在class字节码文件中不包含
    - CLASS：默认的保留策略，注解会在class字节码文件中存在，但运行时无法获取
    - RUNTIME：注解会在class字节码文件中存在，在运行时可以通过反射获取到
- **@Documented**：将此注解包含在javadoc中
- **@Inherited**：允许子类继承父类中的注解

### 自定义注解
```java
// 通过@interface修饰
// 注解名大写开头
// 可以定义成员变量，并且添加默认值
public @interface EnableAuth {
    String name() default "猿天地";
}
```

### Spring常用注解
- **@Configuration**：把一个类作为一个IoC容器，它的某个方法头上如果注册了@Bean，就会作为这个Spring容器中的Bean
- **@Lazy(true)**：表示延迟初始化
- **@Service**：用于标注业务层组件、
- **@Controller**：用于标注控制层组件
- **@Repository**：用于标注数据访问组件，即DAO组件。
- **@Component**：泛指组件，当组件不好归类的时候，我们可以使用这个注解进行标注。
- @Scope：用于指定scope作用域的（用在类上）
- **@PostConstruct**：用于指定初始化方法（用在方法上）
- **@PreDestory**：用于指定销毁方法（用在方法上）
- @DependsOn：定义Bean初始化及销毁时的顺序
- @Primary：自动装配时当出现多个Bean候选者时，被注解为@Primary的Bean将作为首选者，否则将抛出异常
- **@Autowrite**：默认**按类型装配**，如果我们想使用按名称装配，可以结合@Qualifier注解一起使用。如下：@Autowrite @Qualifier("personDaoBean") 存在多个实例配合使用
- **@Resource**：默认**按名称装配**，当找不到与名称匹配的bean才会按类型装配。

@Component指的是组件，
@Controller，@Repository和@Service 注解都被@Component修饰，用于代码中区分表现层，持久层和业务层的组件，代码中组件不好归类时可以使用@Component来标注


## 泛型
Java泛型（ generics） 是JDK 5中引⼊的⼀个新特性， 允许在定义类和接口的时候使⽤**类型参数**。

泛型最⼤的好处是可以提⾼**代码的复⽤性**。

### 类型擦除
类型擦除指的是通过**类型参数合并**，将泛型类型实例关联到**同一份字节码上**。编译器只为泛型类型**生成一份字节码**，并将其实例关联到这份字节码上。类型擦除的关键在于从泛型类型中清除类型参数的相关信息，并且再必要的时候添加类型检查和类型转换的方法。 

类型擦除可以简单的理解为将泛型java代码转换为**普通java代码**，只不过编译器更直接点，将泛型java代码直接转换成普通java字节码。 

- **虚拟机中没有泛型**，只有普通类和普通方法,所有泛型类的类型参数在编译时都会被擦除,泛型类并没有自己独有的Class类对象。比如并不存在List<String>.class或是List<Integer>.class，而只有List.class。
- 创建泛型对象时请**指明类型**，让编译器尽早的做参数检查
- 不要忽略编译器的警告信息，那意味着潜在的**ClassCastException**等着你
- **静态变量是被泛型类的所有实例所共享的**。对于声明为MyClass<T>的类，访问其中的静态变量的方法仍然是 MyClass.myStaticVar。不管是通过new MyClass<String>还是new MyClass<Integer>创建的对象，都是共享一个静态变量。
- 泛型的类型参数不能用在Java异常处理的catch语句中。因为异常处理是由JVM在运行时刻来进行的。由于类型信息被擦除，JVM是无法区分两个异常类型MyException<String>和MyException<Integer>的。对于JVM来说，它们都是 MyException类型的。也就无法执行与异常对应的catch语句。

### 泛型中K T V E ？ object等的含义
- **E** - Element (在集合中使用，因为集合中存放的是元素)
- **T** - Type（Java 类）
- **K** - Key（键）
- **V** - Value（值）
- **N** - Number（数值类型）
- **?** - 表示不确定的java类型（无限制通配符类型）

### 上下界限定符extends 和 super
- **<? extends T>**：是指 “上界通配符（Upper Bounds Wildcards）”，即泛型中的类必须为当前类的子类或当前类。
- **<? super T>**：是指 “下界通配符（Lower Bounds Wildcards）”，即泛型中的类必须为当前类或者其父类。

## SPI
- **API**(Application Programming Interface)

  大多数情况下，都是**实现方**来制定接口并完成对接口的不同实现，调用方仅仅依赖却无权选择不同实现。

- **SPI**(Service Provider Interface)

  调用方来制定接口，实现方来针对接口来实现不同的实现。调用方来**选择自己需要的实现方**。

### 如何定义SPI
- 步骤1、定义一组接口 (假设是org.foo.demo.IShout)，并写出接口的一个或**多个实现**
```java
public interface IShout {
    void shout();
}
public class Cat implements IShout {
    @Override
    public void shout() {
        System.out.println("miao miao");
    }
}
public class Dog implements IShout {
    @Override
    public void shout() {
        System.out.println("wang wang");
    }
}
```

- 步骤2、在 **src/main/resources/** 下建立 **/META-INF/services** 目录， 新增一个以**接口命名的文件** (org.foo.demo.IShout文件)，内容是要应用的实现类
```java
org.foo.demo.animal.Dog
org.foo.demo.animal.Cat
```

- 步骤3、使用 **ServiceLoader** 来加载配置文件中指定的实现。
```java
public class SPIMain {
    public static void main(String[] args) {
        ServiceLoader<IShout> shouts = ServiceLoader.load(IShout.class);
        for (IShout s : shouts) {
            s.shout();
        }
    }
}
// wang wang, miao miao
```

### 实现原理
ServiceLoader类的签名类的成员变量：
```java
public final class ServiceLoader<S> implements Iterable<S>{
private static final String PREFIX = "META-INF/services/";

// 代表被加载的类或者接口
private final Class<S> service;

// 用于定位，加载和实例化providers的类加载器
private final ClassLoader loader;

// 创建ServiceLoader时采用的访问控制上下文
private final AccessControlContext acc;

// 缓存providers，按实例化的顺序排列
private LinkedHashMap<String,S> providers = new LinkedHashMap<>();

// 懒查找迭代器
private LazyIterator lookupIterator;

......
}
```
实现流程：
- 应用程序调用ServiceLoader.**load**方法，先创建一个新的ServiceLoader，并实例化该类中的成员变量，包括：
    - loader(ClassLoader类型，类加载器)
    - acc(AccessControlContext类型，访问控制器)
    - providers(LinkedHashMap类型，用于缓存加载成功的类)
    - lookupIterator(实现迭代器功能)
- 应用程序通过**迭代器接口**获取对象实例，ServiceLoader先判断成员变量providers对象中(LinkedHashMap类型)**是否有缓存**实例对象，如果有缓存，直接返回。 如果没有缓存，执行**类的装载**：
    - 读取**META-INF/services/** 下的配置文件，获得所有能被实例化的类的名称
    - 通过反射方法**Class.forName()** 加载类对象，并用instance()方法将类实例化
    - 把实例化后的类缓存到providers对象中(LinkedHashMap类型）
    - 然后返回实例对象


## 异常
### Error和Exception
- Error表⽰**系统级的错误**， 是java运⾏环境内部错误或者硬件问题， 不能指望程序来处理这样的问题， 除了退出运⾏外别⽆选择， 它是Java虚拟机抛出的。
- Exception 表⽰程序需要捕捉、 **需要处理的异常**， 是由与程序设计的不完善⽽出现的问题， 程序必须处理的问题。

### 异常类型
- **受检异常**
```java
// 在声明的时候定义了异常，调用者一定要处理异常，否则将编译不通过
public void test() throw new Exception{ }
```
- **非受检异常**

  ⼀般是运⾏时异常， 继承⾃**RuntimeException**。这种异常⼀般可以理解为是**代码原因**导致的。 ⽐如发⽣空指针、 数组越界等。 所以， 只要代码写的没问题， 这些异常都是可以避免的。

### 异常相关关键字
- **try**-⽤来指定⼀块预防所有异常的程序；
- **catch**⼦句紧跟在try块后⾯， ⽤来指定你想要捕获的异常的类型；
- **finally**为确保⼀段代码不管发⽣什么异常状况都要被执⾏；
- **throw**语句⽤来明确地抛出⼀个异常；
- **throws**⽤来声明⼀个⽅法可能抛出的各种异常；

### 自定义异常
通过继承**Exception**的⼦类的⽅式实现。编写⾃定义异常类实际上是继承⼀个API标准异常类， ⽤新定义的异常处理信息**覆盖原有信息**的过程。

### try-with-resources
常规写法：
```java
public static void main(String[] args) {
    BufferedReader br = null;
    try {
        String line;
        br = new BufferedReader(new FileReader("d:\\hollischuang.xml"));
        while ((line = br.readLine()) != null) {
            System.out.println(line);
        }
    } catch (IOException e) {
        // handle exception
    } finally {
        try {
            if (br != null) {
                br.close();
            }
        } catch (IOException ex) {
            // handle exception
        }
    }
}
```

使用try-with-resources语句：
```java
public static void main(String... args) {
    try (BufferedReader br = new BufferedReader(new FileReader("d:\\ hollischuang.xml"))) {
        String line;
        while ((line = br.readLine()) != null) {
            System.out.println(line);
        }
    } catch (IOException e) {
        // handle exception
    }
}
```

底层实现，其实背后的原也很简单，那些我们没有做的关闭资源的操作，编译器都帮我们做了：
```java
public static transient void main(String args[])
    {
        BufferedReader br;
        Throwable throwable;
        br = new BufferedReader(new FileReader("d:\\ hollischuang.xml"));
        throwable = null;
        String line;
        try
        {
            while((line = br.readLine()) != null)
                System.out.println(line);
        }
        catch(Throwable throwable2)
        {
            throwable = throwable2;
            throw throwable2;
        }
        if(br != null)
            if(throwable != null)
                try
                {
                    br.close();
                }
                catch(Throwable throwable1)
                {
                    throwable.addSuppressed(throwable1);
                }
            else
                br.close();
            break MISSING_BLOCK_LABEL_113;
            Exception exception;
            exception;
            if(br != null)
                if(throwable != null)
                    try
                    {
                        br.close();
                    }
                    catch(Throwable throwable3)
                      {
                        throwable.addSuppressed(throwable3);
                    }
                else
                    br.close();
        throw exception;
        IOException ioexception;
        ioexception;
    }
}
```

### finally和return的执行顺序
因为return表⽰的是要整个⽅法体返回， 所以，**finally中的语句会在return之前执⾏**。但是在finally中数据**修改效果会不生效**。
```java
// 测试 修改值类型
static int f() {
    int ret = 0;
    try {
        return ret;  // 返回 0，finally内的修改效果不起作用
    } finally {
        ret++;
        System.out.println("finally执行");
    }
}

// 测试 修改引用类型
static int[] f2(){
    int[] ret = new int[]{0};
    try {
        return ret;  // 返回 [1]，finally内的修改效果起了作用
    } finally {
        ret[0]++;
        System.out.println("finally执行");
    }
}

```


## 时间处理
### SimpleDateFormat
SimpleDateFormat是Java提供的一个格式化和解析日期的工具类。它允许进行格式化（日期 -> 文本）、解析（文本 -> 日期）和规范化。

在使用SimpleDateFormat的时候，需要通过字母来描述时间元素，并组装成想要的日期和时间模式。常用的时间元素和字母的对应表如下：
![image-20200522182253341](https://520li.oss-cn-hangzhou.aliyuncs.com/img/image-20200522182253341.png)

模式字母通常是重复的，其数量确定其精确表示。如下表是常用的输出格式的表示方法。
![image-20200522182306485](https://520li.oss-cn-hangzhou.aliyuncs.com/img/image-20200522182306485.png)

#### SimpleDateFormat线程安全性
![image-20200522182323851](https://520li.oss-cn-hangzhou.aliyuncs.com/img/image-20200522182323851.png)不安全的原因：SimpleDateFormat中的format方法在执行过程中，会使用一个成员变量**calendar**来保存时间。这其实就是问题的关键。
![image-20200522182338374](https://520li.oss-cn-hangzhou.aliyuncs.com/img/image-20200522182338374.png)

### Java 8中的时间处理
在Java8中， 新的时间及⽇期API位于java.time包中：
- **Instant**： 时间戳
- Duration： 持续时间， 时间差
- LocalDate： 只包含⽇期， ⽐如： 2016-10-20
- LocalTime： 只包含时间， ⽐如： 231210
- **LocalDateTime**： 包含⽇期和时间， ⽐如： 2016-10-20 231421
- **Period**： 时间段
- ZoneOffset： 时区偏移量， ⽐如： +8:00
- ZonedDateTime： 带时区的时间
- Clock： 时钟， ⽐如获取⽬前美国纽约的时间

```java
// 获取当前时间
LocalDate today = LocalDate.now();
int year = today.getYear();
int month = today.getMonthValue();
int day = today.getDayOfMonth();
System.out.printf("Year : %d Month : %d day : %d t %n", year,month, day);


// 创建指定日期的时间
LocalDate date = LocalDate.of(2018, 01, 01);


// 检查闰年
LocalDate nowDate = LocalDate.now();
//判断闰年
boolean leapYear = nowDate.isLeapYear();


// 计算两个⽇期之间的天数和⽉数
Period period = Period.between(LocalDate.of(2018, 1, 5),LocalDate.of(2018, 2, 5));
```