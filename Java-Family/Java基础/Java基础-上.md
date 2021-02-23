
## String
### 不可变性
一旦一个string对象在内存(堆)中被创建出来，他就**无法被修改**。特别要注意的是，String类的所有方法都没有改变字符串本身的值，都是**返回了一个新的对象**。

如果你需要一个可修改的字符串，应该使用**StringBuffer**（线程安全）或者**StringBuilder**。否则会有大量时间浪费在垃圾回收上，因为每次试图修改都有新的string对象被创建出来。

### SubString
- JDK 6 

  在jdk 6 中，String类包含三个成员变量：char value[]， int offset，int count。当调用substring方法的时候，会创建一个新的string对象，但是这个string的值仍然指向堆中的**同一个字符数组**。这两个对象中只有count和offset 的值是不同的。

  如果你有一个很长很长的字符串，但是当你使用substring进行切割的时候你只需要很短的一段。这可能导致**性能问题**，因为你需要的只是一小段字符序列，但是你却引用了整个字符串（因为这个非常长的字符数组一直在被引用，所以无法被回收，就可能导致**内存泄露**。

- JDK 7 

  在JDK 7中的subString方法，其使用new String创建了一个**新字符串**，避免对老字符串的引用。从而解决了内存泄露问题。

### 字符串拼接
虽然字符串是**不可变**的，但是还是可以通过**新建字符串**的方式来进行字符串的拼接。

常用的字符串拼接方式有五种，分别是使用 **+** 、使用**concat**、使用**StringBuilder**、使用**StringBuffer**以及使用**StringUtils.join**。

由于字符串拼接过程中会创建新的对象，所以如果要在一个循环体中进行字符串拼接，就要考虑内存问题和效率问题。

因此，经过对比，我们发现，直接使用StringBuilder的方式是效率最高的。因为StringBuilder天生就是设计来定义可变字符串和字符串的变化操作的。

但是，还要强调的是：

1、如果不是在循环体中进行字符串拼接的话，直接使用+就好了。

2、如果在并发场景中进行字符串拼接的话，要使用**StringBuffer**来代替StringBuilder。

### switch对String的支持
switch是通过**equals()**和**hashCode()** 方法来实现的。记住，switch中**只能使用整型**，比如byte。short，char以及int。还好hashCode()方法返回的是**int**，而不是long。进行switch的实际是哈希值，然后通过使用**equals**方法比较进行安全检查，这个检查是必要的，因为哈希可能会发生碰撞。

### 运行时常量池（intern）
在JVM运行时区域的方法区中，有一块区域是运行时常量池，主要用来存储编译期生成的各种字面量和符号引用。
- **字面量**：字符串、final的常量值、基本数据类型的**值**
- **符号引用**：类的全限定名、字段名和属性、方法名和属性
```java
String s = new String("liqg");
```
在**编译期**，符号引用s和字面量liqg会被加入到Class文件的常量池中；

在**运行期**，要在Java**堆中**创建一个字符串对象的，而这个对象所对应的**字符串字面量**是保存在字符串常量池中的。

**对象的符号引用s是保存在Java虚拟机栈上的，他保存的是堆中刚刚创建出来的的字符串对象的引用。**

所以，new String()**创建了两个对象**，一个在常量池，一个在堆中。

- **intern()**
String对象的intern函数，它的作用：

    - 将字符串字面量放入常量池（如果池没有的话）
    - 返回这个常量的引用
  

在程序中得到的字符串是只有在**运行期**才能确定的，在编译期是无法确定的，那么也就没办法在编译期被加入到常量池中。这时候，对于那种可能经常使用的字符串，使用**intern**进行定义，每次JVM运行到这段代码的时候，就会直接把常量池中该字面值的引用返回，这样就可以**减少大量字符串对象的创建**了。

### 为什么String要用final修饰

- 不可改变，**执行效率**高

  只要声明成final ，JVM才不用对相关方法在**虚函数表**中查询，而**直接定位**到string类的相关方法上，提高了**执行效率**。基础类以保证执行效率为第一要素。

- **安全性**

  String这个对象基本是被所有的类对象都会使用的到了，**如果可以被复写**，就会很乱套，比如map的key ，如果是一个string为key的话，String如果可以改变的话，会**产生很多不可控的后果**。

- 可以**共用同一个实例**

  在多线程中共享一个不可变对象而**不用担心线程安全**问题。当代码中出现字面量形式创建字符串对象时，JVM首先会对这个字面量进行检查，如果**字符串常量池**中存在相同内容的字符串对象的引用，则将这个引用返回，否则新的字符串对象被创建，然后将这个引用放入字符串常量池，并返回该引用。

- **设计思想**

  语言本身就是一种设计。任何设计思想都是会遵守一些既定的规则，这样才能体现**一致性**。无论是人类语言，还是机器语言，都有它们的约束规则。

  Long, Double, Integer 之类的全都是final的 ，程序的基石是不可被改变的

## Lambda
Java 8 引入的 Lambda 表达式的主要作用就是**简化部分匿名内部类**的写法。Lambda 表达式的另一个依据是**类型推断机制**。在上下文信息足够的情况下，编译器可以推断出参数表的类型，而不需要显式指名。

### 常见用法
#### 无参函数的简写
``` java
new Thread(new Runnable() {
    @Override
    public void run() {
        System.out.println("Hello");
    }
}).start();

/** ===== 简化后 ===== **/
new Thread(() -> System.out.println("Hello")).start();
```

#### 单参函数的简写
```java
view.setOnClickListener(new View.OnClickListener() {
    @Override
    public void onClick(View v) {
        v.setVisibility(View.GONE);
    }
});

/** ===== 简化后 ===== **/
view.setOnClickListener(v -> v.setVisibility(View.GONE));
```

#### 多参函数的简写
```java
List<Integer> list = Arrays.asList(1, 2, 3);
Collections.sort(list, new Comparator<Integer>() {
    @Override
    public int compare(Integer o1, Integer o2) {
        return o1.compareTo(o2);
    }
});

/** ===== 简化后 ===== **/
Collections.sort(list, (o1, o2) -> o1.compareTo(o2));

```

### 方法引用
#### 引用静态方法
```java
// 符合如下格式
([变量1, 变量2, ...]) -> 类名.静态方法名([变量1, 变量2, ...])

// 定义一个静态方法
public class Utils {
    public static int compare(Integer o1, Integer o2) {
        return o1.compareTo(o2);
    }
}

// 一般写法
Collections.sort(list, (o1, o2) -> Utils.compare(o1, o2));

// 简化写法
Collections.sort(list, Utils::compare);
```

#### 引用对象的方法
```java
Collections.sort(list, (o1, o2) -> compare(o1, o2));

Collections.sort(list, this::compare);
```

#### 引用类的方法
```java
// 符合如下格式
(变量1[, 变量2, ...]) -> 变量1.实例方法([变量2, ...])

// 一般写法
Collections.sort(list, (o1, o2) -> o1.compareTo(o2));

// 简化写法
Collections.sort(list, Integer::compareTo);
```

#### 引用构造方法
```java
// 符合如下的格式
([变量1, 变量2, ...]) -> new 类名([变量1, 变量2, ...])
// 可以简化成
类名::new

// 一般写法
Function<Integer, ArrayList> function = n -> new ArrayList(n);

// 简化写法
Function<Integer, ArrayList> function = ArrayList::new;
```

### 实现原理
```java
public class LambdaTest {
    public static void main(String[] args) {
        new Thread(() -> System.out.println("Hello World")).start();
    }
}
```
通过 javap 对该文件反编译后的结果：
```java
public static void main(java.lang.String[]);
  Code:
     0: new           #2                  // class java/lang/Thread
     3: dup
     4: invokedynamic #3,  0              // InvokeDynamic #0:run:()Ljava/lang/Runnable;
     9: invokespecial #4                  // Method java/lang/Thread."<init>":(Ljava/lang/Runnable;)V
    12: invokevirtual #5                  // Method java/lang/Thread.start:()V
    15: return
```
发现 Lambda 表达式被封装成了主类的一个私有方法，并通过 **invokedynamic** 指令进行调用。

**invokedynamic**：JAVA7中增加了invokedynamic指令，这是JAVA为了实现『**动态类型语言**』支持而做的一种改进。动态类型语言是判断变量值的**类型信息**，变量没有类型信息，变量值才有类型信息，这是动态语言的一个重要特征。

### 为什么局部变量需要final
Lambda 表达(匿名类) 会在另一个线程中执行，Lambda 表达式(匿名类) 中的其实是局部变量的一个**拷贝**。

如果没有要求lambda表达式外部变量为final修饰，那么开发者会误以为外部变量的值能够在lambda表达式中被改变，而这实际是不可能的，所以要求外部变量为final是在编译期以强制手段确保用户不会在lambda表达式中做修改原变量值的操作。


## Stream
### Stream介绍
Stream 使用一种类似用 SQL 语句从数据库查询数据的直观方式来提供一种对 Java **集合运算和表达的高阶抽象**。

这种风格将要处理的元素集合看作**一种流**，流在**管道中传输**，并且可以在管道的节点上进行处理，比如筛选，排序，聚合等。

Stream有以下特性及**优点**：
- **无存储**。Stream不是一种数据结构，它只是某种数据源的一个视图，数据源可以是一个数组，Java容器或I/O channel等。
- 为**函数式编程**而生。对Stream的任何修改**都不会修改背后的数据源**，比如对Stream执行过滤操作并不会删除被过滤的元素，而是会产生一个不包含被过滤元素的新Stream。
- **惰式执行**。Stream上的操作并不会立即执行，只有等到用户真正需要结果的时候才会执行。
- **可消费性**。Stream只能被**消费一次**，一旦遍历过就会失效，就像容器的迭代器那样，想要再次遍历必须重新生成。

![image-20200522181047557](https://520li.oss-cn-hangzhou.aliyuncs.com/img/image-20200522181047557.png)

Stream上的所有操作分为两类：中间操作和结束操作，中间操作只是一种标记，只有结束操作才会触发实际计算。

- **中间操作**又可以分为无状态的(Stateless)和有状态的(Stateful)
    - 无状态中间操作是指元素的处理不受前面元素的影响，
    - 有状态的中间操作必须等到所有元素处理之后才知道最终结果，
- **结束操作**又可以分为短路操作和非短路操作。
    - 短路操作是指不用处理全部元素就可以返回结果，比如找到第一个满足条件的元素。


### Stream的创建
#### 通过已有的集合来创建流
```java
List<String> strings = Arrays.asList(Hello", "HelloWorld");
Stream<String> stream = strings.stream();
```
通过一个已有的List创建一个流。除此以外，还有一个**parallelStream**方法，可以为集合创建一个**并行流**。

#### 通过Stream创建流
```java
Stream<String> stream = Stream.of("Hello", "HelloWorld");
```

### Stream中间操作
Stream有很多中间操作，多个中间操作可以连接起来形成一个流水线，每一个中间操作就像流水线上的一个工人，每人工人都可以对流进行加工，加工后得到的结果还是一个流。

#### filter
```java
// filter 方法用于通过设置的条件过滤出元素
List<String> strings = Arrays.asList("Hollis", "", "HollisChuang", "H", "hollis");
strings.stream().filter(string -> !string.isEmpty()).forEach(System.out::println);
//Hollis, HollisChuang, H, hollis
```

#### map
```java
// map 方法用于映射每个元素到对应的结果
List<Integer> numbers = Arrays.asList(3, 2, 2, 3, 7, 3, 5);
numbers.stream().map( i -> i*i).forEach(System.out::println);
//9,4,4,9,49,9,25
```

#### limit/skip
```java
// limit 返回 Stream 的前面 n 个元素；skip 则是扔掉前 n 个元素
List<Integer> numbers = Arrays.asList(3, 2, 2, 3, 7, 3, 5);
numbers.stream().limit(4).forEach(System.out::println);
//3,2,2,3
```

#### sorted
```java
// sorted 方法用于对流进行排序
List<Integer> numbers = Arrays.asList(3, 2, 2, 3, 7, 3, 5);
numbers.stream().sorted().forEach(System.out::println);
//2,2,3,3,3,5,7
```

#### distinct
```java
// distinct主要用来去重
List<Integer> numbers = Arrays.asList(3, 2, 2, 3, 7, 3, 5);
numbers.stream().distinct().forEach(System.out::println);
//3,2,7,5
```

### Stream最终操作
Stream的中间操作得到的结果还是一个Stream，需要把流计算出流中元素的个数、将流装换成集合等。在最终操作之后，**不能再次使用流**，也不能在使用任何中间操作，否则将抛出异常。

#### forEach
```java
// forEach用来迭代流中的每个数据
Random random = new Random();
random.ints().limit(10).forEach(System.out::println);
```

#### count
```java
// count用来统计流中的元素个数
List<String> strings = Arrays.asList( "Hello", "HelloWorld", "Hollis");
System.out.println(strings.stream().count());
```

#### collect
```java
// collect就是一个归约操作，可以接受各种做法作为参数，将流中的元素累积成一个汇总结果
List<String> strings = Arrays.asList("Hollis", "HollisChuang", "hollis","Hollis666", "Hello", "HelloWorld", "Hollis");
strings  = strings.stream().filter(string -> string.startsWith("Hollis")).collect(Collectors.toList());
System.out.println(strings);
//Hollis, HollisChuang, Hollis666, Hollis
```

### 实现原理 [参考](https://mp.weixin.qq.com/s/RIH2Saml6FtkIn9HxFaVcA)
#### 操作如何记录
一个完整的操作是**<数据来源，操作，回调函数>**构成的三元组。Stream中使用**Stage**的概念来描述一个完整的操作，并用某种实例化后的**PipelineHelper**来代表Stage，将具有先后顺序的各个Stage连到一起，就构成了整个流水线。

![image-20200522181138057](https://520li.oss-cn-hangzhou.aliyuncs.com/img/image-20200522181138057.png)

通过Collection.stream()方法得到Head也就是stage0，紧接着调用一系列的中间操作，不断产生**新的Stream**。这些Stream对象以**双向链表**的形式组织在一起，构成整个流水线，由于每个Stage都记录了**前一个Stage和本次的操作以及回调函数**，依靠这种结构就能建立起对数据源的所有操作。

#### 操作如何叠加
Stage并不知道后面Stage到底执行了哪种操作，以及回调函数是哪种形式，使用Sink接口完成Stage的调用关系。Sink接口包含的方法：

![image-20200522181159199](https://520li.oss-cn-hangzhou.aliyuncs.com/img/image-20200522181159199.png)

每个Stage都会将自己的操作封装到一个Sink里，前一个Stage只需调用后一个Stage的accept()方法即可，并不需要知道其内部是如何处理的。

Stream.sorted()是一个**有状态**的中间操作，其对应的Sink.begin()方法可能创建一个放结果的容器，而accept()方法负责将元素添加到该容器，最后end()负责对容器进行排序。

Stream.findFirst()是短路操作，只要找到一个元素，cancellationRequested()就应该返回true，以便调用者尽快结束查找。

Sink的四个接口方法常常**相互协作**，共同完成计算任务。实际上Stream API内部实现的的本质，就是如何**重载Sink**的这四个接口方法。
```java
// Stream.map()，调用该方法将产生一个新的Stream

public final <R> Stream<R> map(Function<? super P_OUT, ? extends R> mapper) {

    ...

    return new StatelessOp<P_OUT, R>(this, StreamShape.REFERENCE,

                                 StreamOpFlag.NOT_SORTED | StreamOpFlag.NOT_DISTINCT) {

        @Override /*opWripSink()方法返回由回调函数包装而成Sink*/

        Sink<P_OUT> opWrapSink(int flags, Sink<R> downstream) {

            return new Sink.ChainedReference<P_OUT, R>(downstream) {

                @Override

                public void accept(P_OUT u) {

                    R r = mapper.apply(u);// 1. 使用当前Sink包装的回调函数mapper处理u

                    downstream.accept(r);// 2. 将处理结果传递给流水线下游的Sink

                }

            };

        }

    };

}
```
```java
// Stream.sort()方法用到的Sink实现
// 1.首先beging()方法告诉Sink参与排序的元素个数，方便确定中间结果容器的的大小；
// 2.之后通过accept()方法将元素添加到中间结果当中，最终执行时调用者会不断调用该方法，直到遍历所有元素；
// 3.最后end()方法告诉Sink所有元素遍历完毕，启动排序步骤，排序完成后将结果传递给下游的Sink；
// 4.如果下游的Sink是短路操作，将结果传递给下游时不断询问下游cancellationRequested()是否可以结束处理。


class RefSortingSink<T> extends AbstractRefSortingSink<T> {

    private ArrayList<T> list;// 存放用于排序的元素

    RefSortingSink(Sink<? super T> downstream, Comparator<? super T> comparator) {

        super(downstream, comparator);

    }

    @Override

    public void begin(long size) {

        ...

        // 创建一个存放排序元素的列表

        list = (size >= 0) ? new ArrayList<T>((int) size) : new ArrayList<T>();

    }

    @Override

    public void end() {

        list.sort(comparator);// 只有元素全部接收之后才能开始排序

        downstream.begin(list.size());

        if (!cancellationWasRequested) {// 下游Sink不包含短路操作

            list.forEach(downstream::accept);// 2. 将处理结果传递给流水线下游的Sink

        }

        else {// 下游Sink包含短路操作

            for (T t : list) {// 每次都调用cancellationRequested()询问是否可以结束处理。

                if (downstream.cancellationRequested()) break;

                downstream.accept(t);// 2. 将处理结果传递给流水线下游的Sink

            }

        }

        downstream.end();

        list = null;

    }

    @Override

    public void accept(T t) {

        list.add(t);// 1. 使用当前Sink包装动作处理t，只是简单的将元素添加到中间列表当中

    }

}
```

#### 叠加之后的操作如何执行
下表给出了各种有返回结果的Stream结束操作：

![image-20200522181221047](https://520li.oss-cn-hangzhou.aliyuncs.com/img/image-20200522181221047.png)

- 对于表中返回boolean或者Optional的操作（Optional是存放 一个 值的容器）的操作，由于值返回一个值，只需要在对应的Sink中记录这个值，等到执行结束时返回就可以了。
- 对于归约操作，最终结果放在用户调用时**指定的容器**中（容器类型通过收集器指定）。collect(), reduce(), max(), min()都是归约操作，虽然max()和min()也是返回一个Optional，但事实上底层是通过调用reduce()方法实现的。
- 对于返回是数组的情况，毫无疑问的结果会放在数组当中。这么说当然是对的，但在最终返回数组之前，结果其实是存储在一种叫做**Node**的数据结构中的。Node是一种多叉树结构，元素存储在树的叶子当中，并且一个叶子节点可以存放多个元素。这样做是为了**并行执行**方便。

### 为什么要求集合是final的
因为Stream是懒操作的，只有触发结果操作时才开始执行。如果在中间对集合发生操作将影响整个结果。

## 自动拆装箱
### 基本数据类型
基本类型，或者叫做**内置类型**，是Java中不同于类(Class)的特殊类型。它们是我们编程中使用最频繁的类型。

Java基本类型共有八种，基本类型可以分为三类：
>字符类型**char**
>
>布尔类型**boolean**
>
>数值类型**byte**、**short**、**int**、**long**、**float**、**double**。

数值类型又可以分为整数类型byte、short、int、long和浮点数类型float、double。

#### 使用基本类型的**好处**

在Java语言中，new一个对象是存储在堆里的，我们通过栈中的引用来使用这些对象；所以，对象本身来说是比较消耗资源的。

对于经常用到的类型，如int等，如果我们每次使用这种变量的时候都需要new一个Java对象的话，就会比较笨重。所以，和C++一样，Java提供了基本数据类型，这种数据的变量**不需要使用new创建**，他们不会在堆上创建，而是**直接在栈内存中存储**，因此会更加高效。

#### 取值范围
- **byte**：byte用1个字节来存储，范围为-128(-2^7)到127(2^7-1)，在变量初始化的时候，byte类型的默认值为0。
- **short**：short用2个字节存储，范围为-32,768 (-2^15)到32,767 (2^15-1)，在变量初始化的时候，short类型的默认值为0，一般情况下，因为Java本身转型的原因，可以直接写为0。
- **int**：int用4个字节存储，范围为-2,147,483,648 (-2^31)到2,147,483,647 (2^31-1)，在变量初始化的时候，int类型的默认值为0。
- **long**：long用8个字节存储，范围为-9,223,372,036,854,775,808 (-2^63)到9,223,372,036, 854,775,807 (2^63-1)，在变量初始化的时候，long类型的默认值为0L或0l，也可直接写为0。

**超出范围**：在溢出的时候并不会抛异常，也没有任何提示。只是**取值**会不正确。

### 包装类型
Java语言是一个面向对象的语言，但是Java中的基本数据类型却是不面向对象的，这在实际使用时存在很多的不便，为了解决这个不足，在设计类时为每个基本数据类型设计了一个对应的类进行代表，这样八个和基本数据类型对应的类统称为包装类(Wrapper Class)。


| 基本数据类型 | 包装类    |
| ------------ | --------- |
| byte         | Byte      |
| boolean      | Boolean   |
| short        | Short     |
| char         | Character |
| int          | Integer   |
| long         | Long      |
| float        | Float     |
| double       | Double    |

### 自动拆箱与自动装箱
在Java SE5中，为了减少开发人员的工作，Java提供了自动拆箱与自动装箱功能。

自动装箱: 就是将基本数据类型自动转换成对应的包装类。

自动拆箱：就是将包装类自动转换成对应的基本数据类型。

>自动装箱都是通过包装类的**valueOf()** 方法来实现的.自动拆箱都是通过包装类对象的**xxxValue()** 来实现的。

#### 哪些地方会自动拆装箱
- 将基本数据类型放入集合类
- 包装类型和基本类型的大小比较
- 包装类型的运算
- 三目运算符的使用
- 函数参数与返回值

#### 带来的问题
- 包装对象的数值比较，不能简单的使用==，虽然-128到127之间的数字可以，但是这个范围之外还是需要使用**equals**比较。
- 有些场景会进行自动拆装箱，由于自动拆箱，如果包装类对象为null，那么自动拆箱时就有可能抛出**NPE**。
- 如果一个**for循环**中有大量拆装箱操作，会浪费很多资源。

## Integer缓存机制
在Java 5中引入的一个有助于节省内存、提高性能的功能。

>适用于整数值区间**-128 至 +127**。
>
>只适用于自动装箱。**使用构造函数创建对象不适用**。
### 实现原理
把基本数据类型自动转换成封装类对象的过程叫做自动装箱，相当于使用valueOf方法。下文为JDK8的valueOf实现：
```java
public static Integer valueOf(int i) {
    if (i >= IntegerCache.low && i <= IntegerCache.high)
        return IntegerCache.cache[i + (-IntegerCache.low)];
    return new Integer(i);
}
```
**IntegerCache**是Integer类中定义的一个private static的内部类。
```java
private static class IntegerCache {
    static final int low = -128;
    static final int high;
    static final Integer cache[];

    static {
        // high value may be configured by property
        int h = 127;
        String integerCacheHighPropValue =
            sun.misc.VM.getSavedProperty("java.lang.Integer.IntegerCache.high");
        if (integerCacheHighPropValue != null) {
            try {
                int i = parseInt(integerCacheHighPropValue);
                i = Math.max(i, 127);
                // Maximum array size is Integer.MAX_VALUE
                h = Math.min(i, Integer.MAX_VALUE - (-low) -1);
            } catch( NumberFormatException nfe) {
                // If the property cannot be parsed into an int, ignore it.
            }
        }
        high = h;

        cache = new Integer[(high - low) + 1];
        int j = low;
        for(int k = 0; k < cache.length; k++)
            cache[k] = new Integer(j++);

        // range [-128, 127] must be interned (JLS7 5.1.7)
        assert IntegerCache.high >= 127;
    }

    private IntegerCache() {}
}
```
最大值127可以通过 **-XX:AutoBoxCacheMax=size** 修改。

### 其他缓存的对象
这种缓存行为不仅适用于Integer对象。我们针对所有的**整数类型**的类都有类似的缓存机制。
- 有ByteCache用于缓存Byte对象
- 有ShortCache用于缓存Short对象
- 有LongCache用于缓存Long对象
- 有CharacterCache用于缓存Character对象

**Byte, Short, Long**有固定范围: -128 到 127。对于**Character**, 范围是 0 到 127。除了Integer以外，这个范围都不能改变。

## 枚举
### 枚举的用法
在java语言中还没有引入枚举类型之前，表示枚举类型的常用模式是声明一组**int常量**。但是存在**类型安全性**和**程序可读性**两方面不足。

Java定义枚举类型的语句很简约。它有以下**特点**：：
- 1）使用关键字enum
- 2）类型名称，比如这里的Season
- 3）一串允许的值，比如上面定义的春夏秋冬四季
- 4）枚举可以单独定义在一个文件中，也可以嵌在其它Java类中
- 5）枚举可以实现一个或多个接口（Interface）
- 6）可以定义新的变量
- 7）可以定义新的方法
- 8）可以定义根据具体枚举值而相异的类

### 枚举的实现
```java
public final class T extends Enum
{
    private T(String s, int i)
    {
        super(s, i);
    }
    public static T[] values()
    {
        T at[];
        int i;
        T at1[];
        System.arraycopy(at = ENUM$VALUES, 0, at1 = new T[i = at.length], 0, i);
        return at1;
    }

    public static T valueOf(String s)
    {
        return (T)Enum.valueOf(demo/T, s);
    }

    public static final T SPRING;
    public static final T SUMMER;
    private static final T ENUM$VALUES[];
    static
    {
        SPRING = new T("SPRING", 0);
        SUMMER = new T("SUMMER", 1);
        ENUM$VALUES = (new T[] {
            SPRING, SUMMER
        });
    }
}
```
通过反编译后代码我们可以看到，public final class T extends Enum，说明，该类是**继承了Enum类**的，同时**final**关键字告诉我们，这个类也是不能被继承的。

当我们使用enmu来定义一个枚举类型的时候，编译器会**自动**帮我们创建一个final类型的类继承Enum类，所以枚举类型**不能被继承**。

### 枚举与单例
>使用枚举实现单例的方法虽然还没有广泛采用，但是单元素的枚举类型已经成为实现Singleton的最佳方法。

**双重校验锁**实现单例：
```java
public class Singleton {
    private static volatile Singleton singleton;
    
    private Singleton() {}
    
    public static Singleton instance() {
        if (singleton == null) {
            synchronized (Singleton.class) {
                if (singleton == null) {
                    singleton = new Singleton();
                }
            }
        }
        return singleton
    }
    
}
```
枚举实现单例：
```java
public enum Singleton {
    INSTANCE;
}
```

#### 枚举解决线程安全问题
枚举的实现：
```java
public final class T extends Enum
{
    //省略部分内容
    public static final T SPRING;
    public static final T SUMMER;
    private static final T ENUM$VALUES[];
    static
    {
        SPRING = new T("SPRING", 0);
        SUMMER = new T("SUMMER", 1);
        ENUM$VALUES = (new T[] {
            SPRING, SUMMER
        });
    }
}
```
**static**类型的属性会在类被加载之后被初始化，Java**类的加载和初始化过程**都是线程安全的（因为虚拟机在加载枚举的类的时候，会使用ClassLoader的**loadClass**方法，而这个方法使用**同步代码块**保证了线程安全）。所以，创建一个enum类型是线程安全的。

在对枚举类反序列化时，会禁用writeObject、readObject等方法，而是调用枚举类的valueOf()，所以可以保证枚举类的单例。

#### 枚举解决反序列化破坏单例的问题
在序列化的时候Java仅仅是将枚举对象的**name属性**输出到结果中，反序列化的时候则是通过java.lang.Enum的**valueOf**方法来根据名字查找枚举对象。同时，编译器是不允许任何对这种序列化机制的定制的，因此禁用了**writeObject、readObject、readObjectNoData、writeReplace和readResolve**等方法。

普通的Java类的反序列化过程中，会通过反射调用类的默认构造函数来初始化对象。所以，即使单例中构造函数是私有的，也会被**反射给破坏掉**。由于反序列化后的对象是**重新new**出来的，所以这就破坏了单例。但是，**枚举的反序列化并不是通过反射实现的**。

枚举类的valueOf函数：
```java
public static <T extends Enum<T>> T valueOf(Class<T> enumType,String name) {  
    T result = enumType.enumConstantDirectory().get(name);  
    if (result != null)  
        return result;  
    if (name == null)  
        throw new NullPointerException("Name is null");  
    throw new IllegalArgumentException(  
        "No enum const " + enumType +"." + name);  
}  
```



参考：http://hollischuang.gitee.io/tobetopjavaer/