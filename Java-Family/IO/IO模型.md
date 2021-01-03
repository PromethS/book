# 第 1 章 Java IO

## 1.1 基础概念

**1 操作系统与内核**

![img](https://520li.oss-cn-hangzhou.aliyuncs.com/img/20200525152359)

**操作系统**：管理计算机硬件与软件资源的系统软件**内核**：操作系统的核心软件，负责管理系统的进程、内存、设备驱动程序、文件和网络系统等等，为应用程序提供对计算机硬件的安全访问服务

**2 内核空间和用户空间**

为了避免用户进程直接操作内核，保证内核安全，操作系统将内存寻址空间划分为两部分：**内核空间（Kernel-space）**，供内核程序使用**用户空间（User-space）**，供用户进程使用为了安全，内核空间和用户空间是隔离的，即使用户的程序崩溃了，内核也不受影响

**3 数据流**

![image-20210103172417199](https://520li.oss-cn-hangzhou.aliyuncs.com/img/book/20210103172421.png)

计算机中的数据是基于随着时间变换高低电压信号传输的，这些数据信号连续不断，有着固定的传输方向，类似水管中水的流动，因此抽象数据流(I/O流)的概念：**指一组有顺序的、有起点和终点的字节集合**，

抽象出数据流的作用：**实现程序逻辑与底层硬件解耦**，通过引入数据流作为程序与硬件设备之间的抽象层，面向通用的数据流输入输出接口编程，而不是具体硬件特性，程序和底层硬件可以独立灵活替换和扩展

## 1.2 I/O 模型

I/O 模型简单的理解：就是用什么样的通道进行数据的发送和接收，很大程度上决定了程序通信的性能

Java 共支持 3 种网络编程模型/IO 模式：BIO、NIO、AIO

Java BIO ： 同步并阻塞(传统阻塞型)，服务器实现模式为一个连接一个线程，即客户端有连接请求时服务器 端就需要启动一个线程进行处理，如果这个连接不做任何事情会造成不必要的线程开销

![img](https://520li.oss-cn-hangzhou.aliyuncs.com/img/book/20201227214032.image)

Java NIO ： 同步非阻塞，服务器实现模式为一个线程处理多个请求(连接)，即客户端发送的连接请求都会注册到多路复用器上，多路复用器轮询到连接有 I/O 请求就进行处理

![img](https://520li.oss-cn-hangzhou.aliyuncs.com/img/book/20201227214048.image)

Java AIO(NIO.2) ： 异步非阻塞，AIO 引入异步通道的概念，采用了 Proactor 模式，简化了程序编写，有效的请求才启动线程，它的特点是先由操作系统完成后才通知服务端程序启动线程去处理，一般适用于连接数较多且连接时间较长的应用

## 1.3 BIO、NIO、AIO 适用场景分析

BIO 方式适用于**连接数目比较小且固定**的架构，这种方式对服务器资源要求比较高，并发局限于应用中，JDK1.4 以前的唯一选择，但程序简单易理解。

NIO 方式适用于**连接数目多且连接比较短**（轻操作）的架构，比如聊天服务器，弹幕系统，服务器间通讯等。 编程比较复杂，JDK1.4 开始支持。

AIO 方式使用于**连接数目多且连接比较长**（重操作）的架构，比如相册服务器，充分调用 OS 参与并发操作， 编程比较复杂，JDK7 开始支持。

## 1.4 Java BIO 基本介绍

Java BIO 就是传统的 java io 编程，其相关的类和接口在 java.io

BIO(blocking I/O) ： 同步阻塞，服务器实现模式为一个连接一个线程，即客户端有连接请求时服务器端就需 要启动一个线程进行处理，如果这个连接不做任何事情会造成不必要的线程开销，可以通过线程池机制改善(实 现多个客户连接服务器)。

BIO 方式适用于连接数目比较小且固定的架构，这种方式对服务器资源要求比较高，并发局限于应用中，JDK1.4 以前的唯一选择，程序简单易理解

## 1.5 Java BIO 工作机制

![img](https://520li.oss-cn-hangzhou.aliyuncs.com/img/book/20201227214335.image)

**对 BIO 编程流程的梳理 **

1. 服务器端启动一个 ServerSocket
2. 客户端启动 Socket 对服务器进行通信，默认情况下服务器端需要对每个客户 建立一个线程与之通讯
3. 客户端发出请求后, 先咨询服务器是否有线程响应，如果没有则会等待，或者被拒绝
4. 如果有响应，客户端线程会等待请求结束后，在继续执行

## 1.6 Java BIO 应用实例

1. 使用 BIO 模型编写一个服务器端，监听 6666 端口，当有客户端连接时，就启动一个线程与之通讯。
2. 要求使用线程池机制改善，可以连接多个客户端.
3. 服务器端可以接收客户端发送的数据(telnet 方式即可)

```java
public static void main(String[] args) throws IOException {
        ExecutorService executorService = Executors.newCachedThreadPool();
        ServerSocket serverSocket = new ServerSocket(6666);
        System.out.println("服务器启动了");
        while (true) {
            System.out.println("等待连接...");
            //从服务器中接受到一个服务器连接
            final Socket socket = serverSocket.accept();
            System.out.println("连接到了一个客户端");
            executorService.execute(new Runnable() {
                public void run() {
                    handler(socket);
                }
            });
        }
    }
```

根据接收到的socket，获取数据

```java
 //从socket中，获取数据
    public static void handler(Socket socket) {
        try {

            //System.out.println("线程ID="+Thread.currentThread().getId()+" 线程名字="+Thread.currentThread().getName());

            //1. 定义个byte数组用来接受数据
            byte[] bytes = new byte[1024];

            //2. socket的输入流对象
            InputStream inputStream = socket.getInputStream();

            //3. 循环读取来自客户端的发送的数据，这里可以循环对inputSteam对象进行读取
            while (true){

                System.out.println("线程ID="+Thread.currentThread().getId()+" 线程名字="+Thread.currentThread().getName());

                //4. 每次使用数组对象来接收输入流中的数据
                System.out.println("read...");
                //这里需要注意的是：在read方法中会阻塞
                int read = inputStream.read(bytes);

                //5. 如果读取的数据长度不，则把bytes数组对象，转换为字符串对象
                if(read!=-1){
                    String message = new String(bytes,0,read);

                    System.out.println("客户端发送的消息："+message);
                }else {
                    break;
                }
            }

        } catch (IOException e) {
            e.printStackTrace();
        }finally {
            System.out.println("关闭和client的连接");

            try {
                socket.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }
```

## 1.7 Java BIO 问题分析

1. 每个请求都需要创建独立的线程，与对应的客户端进行数据 Read，业务处理，数据 Write
2. 当并发数较大时，需要创建大量线程来处理连接，系统资源占用较大。
3. 连接建立后，如果当前线程暂时没有数据可读，则线程就阻塞在 Read 操作上，造成线程资源浪费

# 第 2 章 Java NIO 编程

## 2.1 Java NIO 基本介绍

1. Java NIO 全称 java non-blocking IO，是指 JDK 提供的新 API。从 JDK1.4 开始，Java 提供了一系列改进的 输入/输出的新特性，被统称为 NIO(即 New IO)，是同步非阻塞的
2. NIO 相关类都被放在 java.nio 包及子包下，并且对原 java.io 包中的很多类进行改写。
3. NIO 有三大核心部分：Channel(通道)，Buffer(缓冲区), Selector(选择器)
4. NIO 是 **面向缓冲区** ，或者面向块编程的。数据读取到一个它稍后处理的缓冲区，需要时可在缓冲区中前后 移动，这就增加了处理过程中的灵活性，使用它可以提供非阻塞式的高伸缩性网络
5. Java NIO 的非阻塞模式，使一个线程从某通道发送请求或者读取数据，但是它仅能得到目前可用的数据，如果 目前没有数据可用时，就什么都不会获取，而不是保持线程阻塞，所以直至数据变的可以读取之前，该线程可 以继续做其他的事情。 非阻塞写也是如此，一个线程请求写入一些数据到某通道，但不需要等待它完全写入，这个线程同时可以去做别的事情。
6. 通俗理解：NIO 是可以做到用一个线程来处理多个操作的。假设有 10000 个请求过来,根据实际情况，可以分配 50 或者 100 个线程来处理。不像之前的阻塞 IO 那样，非得分配 10000 个。
7. HTTP2.0 使用了多路复用的技术，做到同一个连接并发处理多个请求，而且并发请求的数量比 HTTP1.1 大了好 几个数量级
8. 简单案例说明

```java
public class BasicBuffer {
    public static void main(String[] args) {

        //声明一个IntBuffer，用于存放int类型的数据，容量为5
        IntBuffer intBuffer = IntBuffer.allocate(5);

        //默认为写模式，直接使用put即可写入数据
        for(int i=0;i<intBuffer.capacity();i++){
            intBuffer.put(i*2);
        }

        //关键代码，用于翻转读模式，写模式
        intBuffer.flip();

        while (intBuffer.hasRemaining()){
            int i = intBuffer.get();
            System.out.println("buffer中的结果为："+i);
        }
    }
}
```

## 2.2 NIO 和 BIO 的比较

1. BIO 以**流的方式**处理数据,而 NIO 以**块的方式**处理数据,块 I/O 的效率比流 I/O 高很多
2. BIO 是**阻塞**的，NIO 则是**非阻塞**的
3. BIO 基于**字节流和字符流进**行操作，而 NIO 基于 **Channel(通道)和 Buffer(缓冲区)**进行操作，数据总是从通道 读取到缓冲区中，或者从缓冲区写入到通道中。Selector(选择器)用于监听多个通道的事件（比如：连接请求， 数据到达等），因此使用单个线程就可以监听多个客户端通道

## 2.3 NIO 三大核心原理示意图

Selector 、 Channel 和 Buffer之间的关系 ![img](https://520li.oss-cn-hangzhou.aliyuncs.com/img/book/20201227221458.image)

1. 每个 channel 都会对应一个 Buffer
2. Selector 对应一个线程， 一个线程对应多个 channel(连接)
3. 该图反应了有三个 channel 注册到 该 selector //程序
4. 程序切换到哪个 channel 是有事件决定的, Event 就是一个重要的概念
5. Selector 会根据不同的事件，在各个通道上切换
6. Buffer 就是一个内存块 ， 底层是有一个数组
7. 数据的读取写入是通过 Buffer, 这个和 BIO , BIO 中要么是输入流，或者是 输出流, 不能双向，但是 NIO 的 Buffer 是可以读也可以写, 需要 flip 方法切换 channel 是双向的, 可以返回底层操作系统的情况, 比如 Linux ， 底层的操作系统 通道就是双向的

## 2.4 缓冲区

### 基本介绍

缓冲区（Buffer）：缓冲区本质上是一个**可以读写数据的内存块**，可以理解成是一个**容器对象(含数组)**，该对象提供了一组方法，可以更轻松地使用内存块，，缓冲区对象内置了一些机制，能够跟踪和记录缓冲区的状态变化情况。Channel 提供从文件、网络读取数据的渠道，但是读取或写入的数据都必须经由 Buffer，如图

 ![img](https://520li.oss-cn-hangzhou.aliyuncs.com/img/book/20201227221608.image)

### Buffer 类及其子类

1. 在 NIO 中，Buffer 是一个顶层父类，它是一个抽象类, 类的层级关系图:

![img](https://520li.oss-cn-hangzhou.aliyuncs.com/img/book/20201227221625.image)

1. Buffer 类定义了所有的缓冲区都具有的四个属性来提供关于其所包含的数据元素的信息:

```java
private int mark = -1;
private int position = 0;
private int limit;
private int capacity;
```

| 属性     | 描述                                                         |
| -------- | ------------------------------------------------------------ |
| Capacity | 容量，即可以容纳的最大数据量；在缓冲区创建时被设定并且不能改变 |
| Limit    | 表示缓冲区的当前终点，不能对缓冲区超过极限的位置进行读写操作。并且极限是可以修改的 |
| Position | 位置，下一个要读或写的元素的索引，每次读写缓冲区数据时，都会改变该值，为下次读写作准备 |

3.buffer类方法一览表

```java
# jdk 1.4时，引入的API
public final int capacity() //返回此缓冲区的容量
public final int position() //返回此缓冲区的位置
public final Buffer position(int newPosition) //设置缓冲区的位置
public final int limit() //返回此缓冲区的限制
public final Buffer limit(int newLimit) //设置此缓冲区的限制

public final Buffer clear() //清空此缓存区，即将各个标记恢复到初始状态，但是数据没有真正擦除
public final Buffer flip() //反正此缓冲区
public final boolean hasRemaining() //告知在当前位置和限制之间是否有数据
public abstract boolean isReadOnly() //告知此缓冲区是否为只读缓冲区


//jdk1.6 引入的API
public abstract boolean hasArray() //告知此缓冲区是否具有可访问的底层实现数组
public abstract Object array() //返回此缓冲区的底层实现数组
```

### ByteBuffer

从前面可以看出对于 Java 中的基本数据类型(boolean 除外)，都有一个 Buffer 类型与之相对应，最常用的自 然是 ByteBuffer 类（二进制数据），该类的主要方法如下

```java
public static ByteBuffer allocateDirect(int capacity) //创建直接缓冲区
public static ByteBuffer allocate(int capacity) //设置缓冲区的初始容量
public abstract byte get() //从当前位置position上get，get之后，position自动+1
public abstract byte get(int index) //从绝对位置get
public abstract ByteBuffer put(byte b) //从当前位置上添加，put之后，position会自动+1
public abstract ByteBuffer put(int index, byte b) //绝对位置上put
```

## 2.5 通道(Channel)

### 基本介绍

1. NIO 的通道类似于流，但有些区别如下：

- 通道可以**同时进行读写**，而流只能读或者只能写
- 通道可以实现异步读写数据
- 通道可以从缓冲读数据，也可以写数据到缓冲:

1. BIO 中的 stream 是单向的，例如 FileInputStream 对象只能进行读取数据的操作，而 NIO 中的通道(Channel) 是双向的，可以读操作，也可以写操作。
2. Channel 在 NIO 中是一个接口 `public interface Channel extends Closeable{}`
3. 常 用 的 Channel 类 有 ：

- FileChannel：用于文件读写
- DatagramChannel：用于 UDP 数据包收发
- ServerSocketChannel：用于服务端 TCP 数据包收发
- SocketChannel：用于客户端 TCP 数据包收发

### FileChannel 类

> FileChannel 主要用来对本地文件进行 IO 操作，常见的方法有

```java
public int read(ByteBuffer dst) ，从通道读取数据并放到缓冲区中 
public int write(ByteBuffer src) ，把缓冲区的数据写到通道中 
public long transferFrom(ReadableByteChannel src, long position, long count)，从目标通道中复制数据到当前通道
public long transferTo(long position, long count, WritableByteChannel target)，把数据从当前通道复制给目标通道
```

### FileChannel 基本案例

#### 2.5.1  本地文件写数据

> 案例：使用前面学习后的 ByteBuffer(缓冲) 和 FileChannel(通道)， 将 "hello,你好吗" 写入到 file01.txt 中

```java
public class NIOFileChannel {
    public static void main(String[] args) throws IOException {

        String message="hello,你好吗";

        //创建一个输出流
        FileOutputStream fileOutputStream = new FileOutputStream("d:\\a.txt");

        //通过输出流获得一个通道
        FileChannel channel = fileOutputStream.getChannel();

        //创建一个缓冲区
        ByteBuffer byteBuffer = ByteBuffer.allocate(1024);

        //把消息放入到缓冲区中
        byteBuffer.put(message.getBytes());

        /**
         * Before flip:
         *
         * position=15
         * limit=1024
         * capacity=1024
         */

        //现在要读取byteBuffer中的数据了，需要进行读写切换
        byteBuffer.flip();

        /**
         * After flip
         *
         * position=0
         * limit=15
         * capacity=1024
         */

        //往通道里面写数据
        channel.write(byteBuffer);
        fileOutputStream.close();
    }
}
```

#### 2.5.2 本地文件读数据

> 案例：使用前面学习后的 ByteBuffer(缓冲) 和 FileChannel(通道)， 将file01.txt中的文本，写入到屏幕中

```java
public class NIOFileChannel02 {
    public static void main(String[] args) throws IOException {

        //创建文件的输入流
        File file = new File("d:\\a.txt");

        FileInputStream fileInputStream = new FileInputStream(file);

        //通过inputStream 获取对应的Channel
        FileChannel fileChannel = fileInputStream.getChannel();

        //创建一个缓冲区
        ByteBuffer byteBuffer=ByteBuffer.allocate((int) file.length());

        //文件通道把数据读入到缓冲区里面
        fileChannel.read(byteBuffer);

        //直接读取缓冲区的数组，底层就是hb byte数组
        //final byte[] hb; // Non-null only for heap buffers
        byte[] array = byteBuffer.array();

        String message = new String(array);

        System.out.println(message);
    }
}
```

#### 2.5.3 文件的拷贝

> 使用一个 Buffer 完成文件读取、写入,使用 FileChannel(通道) 和 方法 read , write，完成文件的拷贝

```java
public class NIOFileChannel03 {
    public static void main(String[] args) throws IOException {
        FileInputStream fileInputStream = new FileInputStream("1.txt");
        FileChannel fileChannel01 = fileInputStream.getChannel();

        FileOutputStream fileOutputStream = new FileOutputStream("2.txt");
        FileChannel fileChannel02 = fileOutputStream.getChannel();

        ByteBuffer byteBuffer = ByteBuffer.allocate(512);

        while (true){

            byteBuffer.clear();

            /**
             * 读取之前：
             *
             * position=0
             * limit=512
             * capacity=512
             */
            int read = fileChannel01.read(byteBuffer);

            /**
             * 读取之后：
             *
             * position=31
             * limit=512
             * capacity=512
             */
            if(read==-1){
                //读完了
                break;
            }

            byteBuffer.flip();

            /**
             * 翻转之后，切换读取模式：
             *
             * position=0
             * limit=31
             * capacity=512
             */
            fileChannel02.write(byteBuffer);

            /**
             * 读取模式，
             *
             * position=31
             * limit=31
             * capacity=512
             */
        }

        fileInputStream.close();
        fileOutputStream.close();
    }
}
```

#### 2.5.4 拷贝文件 transferFrom 方法

> 上个案例使用了ByteBuffer完成了，文件复制，现在直接使用Channel的transferFrom方法，完成文件拷贝

```java
public class NIOFileChannel04 {
    public static void main(String[] args) throws IOException {

        FileInputStream fileInputStream = new FileInputStream("E:\\aa.docx");
        FileChannel sourceChannel = fileInputStream.getChannel();

        FileOutputStream fileOutputStream = new FileOutputStream("E:\\aa1.doc");
        FileChannel destChannel = fileOutputStream.getChannel();

        destChannel.transferFrom(sourceChannel,0,sourceChannel.size());

        fileInputStream.close();
        sourceChannel.close();

        fileOutputStream.close();
        destChannel.close();
    }
}
```

## 2.6  注意事项和细节

### 2.6.1  bytebuffer异常

> ByteBuffer 支持类型化的 put 和 get, put 放入的是什么数据类型，get 就应该使用相应的数据类型来取出，否则可能有 BufferUnderflowException 异常

```java
public class NIOByteBufferPutGet {
    public static void main(String[] args) {
        ByteBuffer buffer = ByteBuffer.allocate(64);

        //使用类型化方式放入数据
        buffer.putInt(10);
        buffer.putLong(10L);
        buffer.putChar('你');
        buffer.putShort((short) 11);
        /*
            limit = position;
            position = 0;
            mark = -1;
        */
        buffer.flip();

        //System.out.println(buffer.getLong()); //BufferUnderflowException
        System.out.println(buffer.getInt());
        System.out.println(buffer.getLong());
        System.out.println(buffer.getChar());
        System.out.println(buffer.getShort());
    }
}
```

### 2.6.2  readOnlyBuffer

> buffer中有一个asReadOnlyBuffer方法，用于获得一个只读buffer，此buffer只能用作于读取数据

```java
public class ReadOnlyBuffer {
    public static void main(String[] args) {

        ByteBuffer buffer = ByteBuffer.allocate(64);

        for (int i = 0; i < 64; i++) {
            buffer.put((byte)i);
        }

        buffer.flip();

        //只读buffer
        ByteBuffer readOnlyBuffer = buffer.asReadOnlyBuffer();

        while (readOnlyBuffer.hasRemaining()){

            System.out.println(readOnlyBuffer.get());
        }
    }
}
```

### 2.6.3 堆外内存进行修改

> NIO 还提供了 MappedByteBuffer， 可以让文件直接在内存（堆外的内存）中进行修改， 操作系统不需要拷贝一次， 而如何同步到文件 由 NIO 来完成

```java
public class MappedByteBufferTest {
    public static void main(String[] args) throws IOException {
        
        //因为RandomAccessFile可以自由访问文件的任意位置，所以如果我们希望只访问文件的部分内容，那就可以使用RandomAccessFile类
        RandomAccessFile randomAccessFile = new RandomAccessFile("1.txt", "rw");

        FileChannel channel = randomAccessFile.getChannel();

        /*** 参数 1: FileChannel.MapMode.READ_WRITE 使用的读写模式
           * 参数 2： 0 ： 可以直接修改的起始位置
           * 参数 3: 5: 是映射到内存的大小(不是索引位置) ,即将 1.txt 的多少个字节映射到内存
           * 可以直接修改的范围就是 0-5
           * 实际类型 DirectByteBuffer */
        MappedByteBuffer mappedByteBuffer = channel.map(FileChannel.MapMode.READ_WRITE, 0, 5);

        //直接把channel中的数据映射到内存中，在内存中进行修改，减少了一次内存的拷贝，
        //只要进行一次map映射即可

        mappedByteBuffer.put(0,(byte)'H');
        mappedByteBuffer.put(1,(byte)'9');

        randomAccessFile.close();

        System.out.println("修改成功~");
    }
}
```

### 2.6.4 分散 & 聚合

> 在实际的开发当中，当建立通道，开始传输数据的时候，一个buffer可能不够用，这个时候，我们可以使用buffer数组来进行数据传输，这个时候，就有了读取的分散，和写入的聚合

- Scattering：将数据写入到 buffer 时，可以采用 buffer 数组，依次写入 [分散]
- Gathering: 从 buffer 读取数据时，可以采用 buffer 数组，依次读[聚合]

```java
public class ScatteringAndGatheringTest {
    public static void main(String[] args) throws IOException {

        ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();
        InetSocketAddress inetSocketAddress = new InetSocketAddress(8000);

        //绑定端口到 socket ，并启动
        serverSocketChannel.bind(inetSocketAddress);

        //创建一个ByteBuffer数组，使用数组来进行数据传输工作
        ByteBuffer[] byteBuffers = new ByteBuffer[2];
        byteBuffers[0]=  ByteBuffer.allocate(5);
        byteBuffers[1]=  ByteBuffer.allocate(3);

        //等待客户端连接，这里会进行阻塞
        SocketChannel socketChannel = serverSocketChannel.accept();

        int messageLength=8;

        while (true){
            int byteRead=0;

            while (byteRead<messageLength){

                //读取的数量
                //读取的时候，传入一个缓冲数组
                long l = socketChannel.read(byteBuffers);
                byteRead+=l; //累计读取的字节数

                System.out.println("ByteRead="+byteRead);

                Arrays.asList(byteBuffers)
                        .stream()
                        .map(buffer->"position="+buffer.position()+",limit="+buffer.limit())
                        .forEach(System.out::println);
            }

            //对所有的buffer进行flip
            Arrays.asList(byteBuffers).forEach(byteBuffer -> byteBuffer.flip());

            //将数据读出显示到客户端
            long byteWrite=0;
            while (byteWrite<messageLength){
                long l = socketChannel.write(byteBuffers);
                byteWrite+=l;
            }

            //将所有的buffer进行clear
            Arrays.asList(byteBuffers).forEach(byteBuffer -> byteBuffer.clear());

            System.out.println("byteRead:=" + byteRead + " byteWrite=" + byteWrite + ", messagelength" + messageLength);
        }
    }
}
```

## 2.7 选择器 Selector

### 2.7.1 基本介绍

1. Java 的 NIO,用非阻塞的 IO 方式.可以用一个线程,处理多个的客户端连接,就会使用到 Selector(选择器)
2. Selector **能够检测多个注册的通道上是否有事件发生**(注意:多个 Channel 以事件的方式可以注册到同一个 Selector),如果有事件发生,便获取事件然后针对每个事件进行相应的处理.这样就可以只用一个单线程去管 理多个通道,也就是管理多个连接和请求.【示意图】
3. **只有在 连接/通道 真正有读写事件发**时,才会进行读写,就大大地减少了系统开销,并且不必为每个连接都 创建一个线程,不用去维护多个线程
4. 避免了多线程之间的**上下文切换**导致的开销

### 2.7.2  示意图和特点说明

![img](https://520li.oss-cn-hangzhou.aliyuncs.com/img/book/20201227222559.image)

1. Netty 的 IO 线程 NioEventLoop 聚合了**Selector(选择器,也叫多路复用器)**,可以同时并发处理成百上千个客户端连接.
2. 当线程从某客户端 Socket 通道进行读写数据时,若没有数据可用时,该线程可以进行其他任务.
3. 线程通常将非阻塞 IO 的**空闲时间用于在其他通道上**执行 IO 操作,所以单独的线程可以管理多个输入和输出通道.
4. 由于读写操作都是**非阻塞**的,这就可以充分提升 IO 线程的运行效率,避免由于频繁 I/O 阻塞导致的线程挂起.
5. 一个 I/O 线程可以并发处理 N 个客户端连接和读写操作,这从根本上解决了传统同步阻塞 I/O 一连接一线程模型,架构的性能,弹性伸缩能力和可靠性都得到了极大的提升

### 2.7.3 select、poll、epoll

JDK中NIO使用多路复用的IO模型，通过把多个IO阻塞复用到一个**select的阻塞**上，实现系统在单线程中可以同时处理多个客户端请求，节省系统开销，在JDK1.4和1.5 update10版本之前，JDK的Selector基于**select/poll模型**实现，在JDK 1.5 update10以上的版本，底层使用**epoll**代替了select/poll

  - **select**==>时间复杂度O(n)

    当IO事件发生时，只能无差别轮询所有流，找出能操作的流。当**流的数量多了**以后，且只有少数socket有数据的时候，会导致效率下降。而且受限于所持有的**文件句柄(fd)** 数量，默认值是**1024**个。

  - **poll**==>时间复杂度O(n)

    与select**基本相同**，只是没有fd数量限制，因为采用的是**链表存储**

  - **epoll**==>时间复杂度O(1)

     基于**事件驱动**，会把哪个流发生的I/O事件通知我们，**不需要轮询**，只有Linux支持。

    epoll支持打开的文件描述符数量不在受限制。select/poll使用**轮询**方式遍历整个文件描述符的集合，epoll基于每个文件描述符的**callback函数**回调。两种触发模式
       - **水平触发**（EPOLLLT）,默认模式

         只要这个fd还有**数据可读**，每次epoll_wait都会返回它的事件，提醒用户程序去操作

    - **边缘触发**（EPOLLET）

      只会**提示一次**，直到下次再有数据流入之前都不会再提示了，无论fd中是否还有数据可读，所以在read时一定要**读取完数据**。

    - 水平触发模式中，系统中如果有大量你不需要读写的就绪文件描述符，但每次调用epoll_wait都会返回，这样会大大降低处理程序检索自己关心的就绪文件描述符的效率。

      边缘触发模式中，当被监控的文件描述符上有可读写事件发生时，epoll_wait()才会通知处理程序去读写。**边缘触发比水平触发效率高，系统不会充斥大量你不关心的就绪文件描述符**

- **总结** 
  - 表面上看epoll的性能最好，但是在**连接数少且十分活跃**情况下，**epoll的性能不如select与poll**，因为epoll的通知机制需要很多函数回调
  - select/poll效率低是因为每次都需要轮询。但低效是相对的，视情况而定，也可通过良好的设计改善
  - select、poll、epoll**本质都是同步I/O**，因为都需要在数据就绪时自己负责进行读写，这个**读写的过程是堵塞**的。而**异步I/O是不需要自己读写**的，会把数据从内核直接拷贝到用户空间
  - **redis**使用的是epoll模式，Rocketmq、dubbo使用的是nio；**nginx**使用的是AIO（异步IO）

### 2.7.4 JDK epoll的bug

JDK NIO的BUG，例如臭名昭著的epoll bug，它会导致Selector空轮询，最终导致CPU 100%。官方声称在JDK1.6版本的update18修复了该问题，但是直到JDK1.7版本该问题仍旧存在，只不过该BUG发生概率降低了一些而已，它并没有被根本解决。

**Selector BUG出现的原因**

若Selector的轮询结果为空，也没有wakeup或新消息处理，则发生空轮询，CPU使用率100%，

**Netty的解决办法**

- 对Selector的select操作周期进行统计，每完成一次空的select操作进行一次计数，
- 若在某个周期内连续发生N次空轮询，则触发了epoll死循环bug。
- 重建Selector，判断是否是其他线程发起的重建请求，若不是则将原SocketChannel从旧的Selector上去除注册，重新注册到新的Selector上，并将原来的Selector关闭。

参考：https://blog.csdn.net/hemeinvyiqiluoben/article/details/82941571

## 2.8 NIO原理分析图

NIO非阻塞网络编程相关的(Selector、SelectionKey、ServerScoketChannel Socket Channel)关系梳理图 

![img](https://520li.oss-cn-hangzhou.aliyuncs.com/img/book/20201227222855.image) 



1)当客户端连接时，会通过服务器套接字通道得到套接字通道

2)选择器进行监听选择方法，返回有事件发生的通道的个数。

3)将socketChannel注册到Selector上，register(Selector sel，int ops)，一个selector上可以注册多个SocketChannel

4)注册后返回一个SelectionKey，会和该Selector关联（集合)

5)进一步得到各个SelectionKey（有事件发生)

6)在通过SelectionKey反向获取SocketChannel，方法channel（)

7)可以通过得到的channel，完成业务处理

## 2.9 NIO快速入门

> 编写一个 NIO 入门案例,实现服务器端和客户端之间的数据简单通讯(非阻塞)

服务端

```java
import java.io.IOException;
import java.net.InetSocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.*;
import java.util.Iterator;
import java.util.Set;

public class NIOServer {
    public static void main(String[] args) throws IOException {

        //1. 创建ServerSocketChannel，这个是服务端的通道
        //用于产生 为每个客户端都生成一个SocketChannel通道
        ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();

        //2. 创建一个最重要的Selector对象，用于对各种连接请求进行管理
        Selector selector = Selector.open();

        //绑定一个端口6666，在服务端进行监听
        serverSocketChannel.bind(new InetSocketAddress(6666));

        //设置为非阻塞模式
        serverSocketChannel.configureBlocking(false);

        //3. 注册到selector上面，全程由select进行调度使用
        serverSocketChannel.register(selector, SelectionKey.OP_ACCEPT);

        while (true){
            //select方法是一种阻塞方法，必须至少有一个 chanel被选中才能返回
            if(selector.select(1000)==0){
                System.out.println("服务器等待一秒，无连接");
                continue;
            }

            //如果返回的>0, 就获取到相关的 selectionKey 集合
            // 1.如果返回的>0, 表示已经获取到关注的事件
            // 2. selector.selectedKeys() 返回关注事件的集合
            // 通过 selectionKeys 反向获取通道
            Set<SelectionKey> selectionKeys = selector.selectedKeys();

            Iterator<SelectionKey> keyIterator = selectionKeys.iterator();
            //对选择到的key进行遍历，通过key可以反向的获取通道
            while (keyIterator.hasNext()){
                SelectionKey key = keyIterator.next();

                //根据key对应的通道发生的事件做相应处理

                //如果是OP_Accept,有新的连接客户端到来
                if(key.isAcceptable()){
                    //为该客户端生成一个socketChannel
                    SocketChannel socketChannel = serverSocketChannel.accept();

                    System.out.println("客户端连接成功，生成了一个socketChannel"+socketChannel.hashCode());

                    //因为我们的ServerSocketChannel是非阻塞的，所以我们新生成的socketChannel
                    //也要设置成非阻塞的
                    socketChannel.configureBlocking(false);

                    //生成了一个新的通道之后，要为这个通道注册到选择器上，这样选择器才能够监听事件
                    //要监听读事件
                    socketChannel.register(selector,SelectionKey.OP_READ, ByteBuffer.allocate(1024));

                }

                if(key.isReadable()){
                    //通过key反向的得到了通道
                    SocketChannel socketChannel = (SocketChannel)key.channel();

                    //通过key反向的得到了缓冲区
                    ByteBuffer byteBuffer = (ByteBuffer) key.attachment();

                    //把通道中的信息内容，读取到缓冲区中
                    socketChannel.read(byteBuffer);

                    System.out.println("from 客户端:"+new String(byteBuffer.array()));
                }

                //当我们处理完一个通道后，要把当前通道给移除，防止重复操作
                keyIterator.remove();
            }
        }
    }
}
```

客户端

```java
import java.io.IOException;
import java.net.InetSocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.SocketChannel;

public class NIOClient {
    public static void main(String[] args) throws IOException {
        SocketChannel socketChannel = SocketChannel.open();
        socketChannel.configureBlocking(false);

        InetSocketAddress inetSocketAddress = new InetSocketAddress("127.0.0.1", 6666);


        if(!socketChannel.connect(inetSocketAddress)){
            while (!socketChannel.finishConnect()){
                System.out.println("因为连接需要时间,客户端不会阻塞,可以做其它工作..");
            }
        }

        //连接成功，发送数据
        String str="hello,你好吗";
        ByteBuffer buffer = ByteBuffer.wrap(str.getBytes());
        socketChannel.write(buffer);

        System.in.read();
    }
}
```

![img](https://520li.oss-cn-hangzhou.aliyuncs.com/img/book/20201227223409.image)

## 2.10  SelectionKey

1. SelectionKey，表示 Selector 和网络通道的注册关系, 共四种:

int **OP_ACCEPT**：有新的网络连接可以 accept，值为 16

int **OP_CONNECT**：代表连接已经建立，值为 8

int **OP_READ**：代表读操作，值为 1

int **OP_WRITE**：代表写操作，值为 4

源码中：

```java
public static final int OP_READ = 1 << 0; 
public static final int OP_WRITE = 1 << 2; 
public static final int OP_CONNECT = 1 << 3; 
public static final int OP_ACCEPT = 1 << 4;
```

## 2.11 ServerSocketChannel

1. ServerSocketChannel 在服务器端**监听新的客户端 Socket 连接**,主要的职责就是为每一个新的连接过来的客户端，生成一个socketChannel
2. 相关方法

```java
public abstract class ServerSocketChannel  extends AbstractSelectableChannel
    implements NetworkChannel
{
  
  //得到一个ServerSocketChannel通道
  public static ServerSocketChannel open() 
  
  //设置服务端口号
  public final ServerSocketChannel bind(SocketAddress local)
  
  //设置阻塞或者非阻塞模式
  public abstract SelectableChannel configureBlocking(boolean block)
  
  //接收一个连接，返回待代表这个连接的通道对象
  //如果这个通道是非阻塞模式，则这个方法在没有等待连接的时候会立即返回
  //如果是则模式，则会一直阻塞直到一个新的连接可用或者一个io错误发生了 
  public abstract SocketChannel accept()
  
  //注册一个选择器，并且设置监听事件，返回一个SelectionKey
  public final SelectionKey register(Selector sel, int ops)
}
```

## 2.12 SocketChannel

1. SocketChannel，网络 IO 通道，**具体负责进行读写操作**。NIO 把缓冲区的数据写入通道，或者把通道里的数 据读到缓冲区。主要的职责就是用来交换数据用的

```java
public abstract class SocketChannel
    extends AbstractSelectableChannel
    implements ByteChannel, ScatteringByteChannel, GatheringByteChannel, NetworkChannel
{

  //得到一个SocketChannel通道
  public static SocketChannel open()
  
  //设置阻塞模式或者非阻塞模式
  public final SelectableChannel configureBlocking(boolean block)
  
  //连接服务器，如果连接成功，会直接返回true
  public abstract boolean connect(SocketAddress remote) 
  
  //如果上面的方法连接失败，接下来就要通过该方法完成连接操作
  public abstract boolean finishConnect() 
}
```

## 2.13 NIO 打造群聊系统

- 编写一个 NIO 群聊系统，实现服务器端和客户端之间的数据简单通讯（非阻塞）
- 实现多人群聊
- 服务器端：可以监测用户上线，离线，并实现消息转发功能
- 客户端：通过 channel 可以无阻塞发送消息给其它所有用户，同时可以接受其它用户发送的消息(有服务器转发 得到)

**服务端代码**

```java
/**
 * 最终达到一个什么效果
 * <p>
 * 1. 开启一个服务端，监听端口
 * 2. 一旦客户端进行连接了，就创建一个通道进行监听
 * 3. 服务端中显示上线通知
 * 4. 客户端发送消息，其他客户端可以接收到消息
 * 5. 客户端下线了，服务端能够收到
 */
public class GroupChatServer {

    //定义属性，定义服务端中最为重要的几个组件
    private Selector selector; //选择器
    private ServerSocketChannel listenChannel; //服务端的Socket通道
    private static final int PORT = 5555;   //绑定的端口号


    /**
     * 一旦创建一个实例，就立刻生成一个服务端实例，绑定端口，进行监听连接事件
     */
    public GroupChatServer() {
        //在构造函数中，构建好信息
        try {
            //创建服务端对象的最为重要的几步：

            //第一步：不管是服务端还是客户端都必须创建一个选择器
            selector = Selector.open();

            //第二步：服务端，必须得创建一个ServerSocketChannel通道，用于针对客户端生成一个通道
            listenChannel = ServerSocketChannel.open();

            //第三步：socket():Retrieves a server socket associated with this channel.
            //服务端通道对应的socket，进行绑定
            listenChannel.socket().bind(new InetSocketAddress(PORT));

            //必须得设置成非阻塞模式，否则会报异常
            listenChannel.configureBlocking(false);

            //将该listenChannel 注册到selector中
            listenChannel.register(selector, SelectionKey.OP_ACCEPT);
        } catch (IOException e) {
            e.printStackTrace();
        }

    }


    /**
     * 监听来自客户端的请求，为每一个连接的客户端新创建一个通道，并且注册读取的事件
     */
    public void listen() {
        try {

            //循环监听，搞一个死循环
            while (true) {

                //这里一直处于阻塞状态，直到有事件生成
                int count = selector.select();

                //当通道接收到当前有数据来了的时候
                if (count > 0) {

                    //在选择器对象中，获取到选择的Key对象集合
                    Iterator<SelectionKey> iterator = selector.selectedKeys().iterator();

                    //对集合对象进行遍历
                    while (iterator.hasNext()) {

                        //得到了最重要的对象 SelectionKey
                        SelectionKey selectionKey = iterator.next();

                        //如果当前的事件是连接事件，则要为客户端新增一个通道对象
                        if (selectionKey.isAcceptable()) {

                            //这里的accept会针对客户端的连接事件，新增一个socket通道
                            //这里不会阻塞，因为当前已经知道了有一个连接事件到来了，会直接创建
                            SocketChannel socketChannel = listenChannel.accept();

                            //把当前的通道对象设置为非阻塞
                            socketChannel.configureBlocking(false);

                            //当前的这个socket对象，注册到选择器中，用于监听读取事件
                            socketChannel.register(selector, SelectionKey.OP_READ);

                            //上线提示：
                            System.out.println(socketChannel.getRemoteAddress() + " 已经上线了~");
                        }

                        //当通道是可读状态
                        if (selectionKey.isReadable()) {
                            readData(selectionKey);
                        }

                        //移除当前的key
                        iterator.remove();
                    }

                }

            }

        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    /**
     * 读取来自客户端的消息
     *
     * @param selectionKey
     */
    private void readData(SelectionKey selectionKey) {

        //把通道对象抽出来，因为一旦过程中发生了异常了，在finally中可以close掉
        SocketChannel channel = null;

        try {
            //根据key得到通道对象
            channel = (SocketChannel) selectionKey.channel();

            //创建buffer对象，用于接收来自客户端的数据信息
            ByteBuffer byteBuffer = ByteBuffer.allocate(1024);

            int readCount = channel.read(byteBuffer);

            if(readCount>0){
                //如果当前有数据,直接把缓冲区转换为字符串对象
                String msg = new String(byteBuffer.array());

                System.out.println("from "+channel.getRemoteAddress()+"说："+msg);

                //把该消息转发给其他的客户端
                sendMsgToOtherClient(msg,channel);

            }

        } catch (IOException e) {
            e.printStackTrace();

            try {

                //？？？疑问点：为什么这里是可以通过read事件来获取，当前客户端离线了呢？
                //可以通过代码调试，来知道
                /**
                 * 回答：当客户端主动断开连接的时候，也会发送一个可读事件的通知
                 * 但是当这个关闭了的通道，尝试去读取数据的到缓冲区的时候，就会报错：
                 * java.io.IOException: 远程主机强迫关闭了一个现有的连接。
                 * */
                System.out.println(channel.getRemoteAddress()+"离线了~");

                //客户端离线了之后，要取消注册key
                selectionKey.cancel();

                //关闭通道
                channel.close();

            } catch (IOException ex) {
                ex.printStackTrace();
            }
        }


    }

    /**
     * 把消息发给其他的客户端
     * @param msg
     * @param channel
     */
    private void sendMsgToOtherClient(String msg, SocketChannel channel) throws IOException {
        System.out.println("服务器转发消息中...");

        //获取注册到selector上面的SocketChannel，并且排除Self
        for (SelectionKey selectedKey : selector.keys()) {

            //根据每个key获取所对应的Channel
            Channel targetChannel = selectedKey.channel();

            //发给别人，不要发给自己
            if(targetChannel instanceof SocketChannel && targetChannel!=channel){

                //这里得到了其他的通道
                SocketChannel destChannel = (SocketChannel) targetChannel;

                //准备好一个ByteBuffer进行数据传送
                ByteBuffer byteBuffer = ByteBuffer.wrap(msg.getBytes());

                //往通道里面写数据
                destChannel.write(byteBuffer);
            }
        }

    }


    public static void main(String[] args) {

        GroupChatServer groupChatServer = new GroupChatServer();
        groupChatServer.listen();


    }
}
```

**客户端代码**

```java
public class GroupChatClient {

    private final String HOST="127.0.0.1";
    private final Integer PORT=5555;
    private Selector selector;
    private SocketChannel socketChannel;
    private String userName;

    //构造函数
    public GroupChatClient() throws IOException {

        //不管是服务端，还是客户端，第一件事情就是创建一个Selector对象
        selector = Selector.open();

        //连接服务器
        socketChannel= SocketChannel.open(new InetSocketAddress(HOST,PORT));

        //设置socket通道为非阻塞
        socketChannel.configureBlocking(false);

        //把当前的通道，进行注册
        socketChannel.register(selector, SelectionKey.OP_READ);

        //获取客户端的用户信息
        userName = socketChannel.getLocalAddress().toString().substring(1);

        System.out.println(userName+" is ok");
    }


    //发送信息的方法
    public void send(String msg){
        msg=userName+"说："+msg;
        try {
            ByteBuffer byteBuffer = ByteBuffer.wrap(msg.getBytes());
            socketChannel.write(byteBuffer);
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    //读取来自服务端的消息
    public void read(){
        try {
            //获取可读取的通道
            int selectCount = selector.select();
            if(selectCount>0){

                //获取当前选择中，所有有事件发生的SelectionKey
                Iterator<SelectionKey> iterator = selector.selectedKeys().iterator();
                while (iterator.hasNext()){

                    SelectionKey selectionKey = iterator.next();

                    //如果当前的key的类型是可读状态
                    if(selectionKey.isReadable()){

                        //获取对应的通道
                        SocketChannel channel = (SocketChannel) selectionKey.channel();

                        //构建一个buffer
                        ByteBuffer byteBuffer = ByteBuffer.allocate(1024);

                        //从通道中读取数据到buffer中
                        channel.read(byteBuffer);

                        //展示数据
                        System.out.println(new String(byteBuffer.array()));
                    }
                    iterator.remove();
                }
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    public static void main(String[] args) throws IOException {
        //创建一个客户端
        GroupChatClient groupChatClient = new GroupChatClient();

        //新建一个线程，用于，不停的读取来服务器发来的数据
        new Thread(() -> {
            while (true){
                groupChatClient.read();
                try {
                    Thread.sleep(3000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }).start();

        //新建一个输入流
        Scanner scanner = new Scanner(System.in);
        while (scanner.hasNext()){
            String line = scanner.nextLine();
            groupChatClient.send(line);
        }
    }
}
```

# 第 3 章 高性能I/O优化

> 零拷贝是网络编程的关键，很多性能优化都离不开。

我们先来看一下IO模型是如何一步一步的演化的

## 3.1 传统的IO模型

> DMA：全称叫**直接内存存取**（Direct Memory Access），他的作用就是不需要经过CPU进行数据传输（分担CPU工作），将数据从一个地址空间复制到另外一个地址空间。

在传统的IO模型中，有3次上下文切换，2次Cpu拷贝

该图流程如下：

- 从磁盘中，使用直接内存拷贝到内核缓冲
- 内存缓冲使用CPU拷贝到用户模式的缓冲区中
- 业务处理后，再次使用CPU拷贝到 socket的缓冲中

最后使用DMA Copy拷贝到协议栈中

![image-20201227225410670](https://520li.oss-cn-hangzhou.aliyuncs.com/img/book/20201227225413.png)

## 3.2 mmap优化

> mmap 通过内存映射，将`文件映射到内核缓冲区`，同时，用户空间可以`共享内核空间的数据`。这样，在进行网络传输时，就可以减少内核空间到用户空间的拷贝次数

![image-20201227225448098](https://520li.oss-cn-hangzhou.aliyuncs.com/img/book/20201227225450.png)

**优点：**即使频繁调用，**使用小块文件传输，效率也很高**。

**缺点：**不能很好的利用 DMA 方式，会比 sendfile 多消耗 CPU，内存安全性控制复杂，需要避免 JVM crash问题。

## 3.3  sendFile 优化

> Linux 2.1 版本 提供了 sendFile 函数，其基本原理如下：`数据根本不经过用户态`，直接从内核缓冲区进入到 Socket Buffer，同时，由于和用户态完全无关，就减少了一次上下文切换

![image-20201227225519454](https://520li.oss-cn-hangzhou.aliyuncs.com/img/book/20201227225521.png)

### 3.3.1 sendFile再度优化

> Linux 在 2.4 版本中，做了一些修改，避免了从内核缓冲区拷贝到 Socket buffer 的操作，直接拷贝到协议栈， 从而再一次减少了数据拷贝

```
这里其实有 一次 cpu 拷贝 kernel buffer -> socket buffer 但是，拷贝的信息很少，比如 lenght , offset , 消耗低，可以忽略
```

![image-20201227225541010](https://520li.oss-cn-hangzhou.aliyuncs.com/img/book/20201227225543.png)

**优点：**可以利用 DMA 方式，消耗 CPU 较少，**大块文件传输效率高**，无内存安全性问题。

**缺点：**小块文件效率低于 mmap 方式，只能是 BIO 方式传输，不能使用 NIO。

KAFKA的索引文件（index file）使用mmap+write 方式，data文件使用sendfile 。

## 3.4  零拷贝的再次理解

1. 我们说零拷贝，是`从操作系统的角度`来说的。因为内核缓冲区之间，没有数据是重复的（只有 kernel buffer 有 一份数据）。
2. 零拷贝不仅仅带来更少的数据复制，还能带来其他的性能优势，例如`更少的上下文切换`，更少的 CPU 缓存伪共享以及无 CPU 校验和计算。

### 3.4.1  mmap 和 sendFile 的区别

1. **mmap 适合小数据量读写，sendFile 适合大文件传输**。

   > RocketMQ 基于 mmap+write 实现零拷贝，适用于**业务级消息**这种小块文件的数据持久化和传输 。
   >
   > Kafka 基于 sendfile 这种零拷贝方式，适用于系统日志消息这种**高吞吐量**的大块文件的数据持久化和传输。
   >
   > > KAFKA的索引文件使用mmap+write 方式，data文件使用sendfile
   >
   > Netty 的零拷贝分为两种：
   >
   > - 基于操作系统实现的零拷贝，底层基于FileChannel的transferTo方法
   > - 基于Java 层操作优化，对数组缓存对象(ByteBuf )进行封装优化，通过对ByteBuf数据建立数据视图，支持ByteBuf 对象**合并，切分**，当底层仅保留一份数据存储，减少不必要拷贝

2. mmap 需要 4 次上下文切换，3 次数据拷贝；sendFile 需要 3 次上下文切换，最少 2 次数据拷贝。

3. sendFile 可以利用 DMA 方式，减少 CPU 拷贝，mmap 则不能（必须从内核拷贝到 Socket 缓冲区）

![image-20200519002833382](https://520li.oss-cn-hangzhou.aliyuncs.com/img/image-20200519002833382.png)

### 3.4.2  NIO 零拷贝案例

**服务端代码**

```java
import java.io.IOException;
import java.net.InetSocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.ServerSocketChannel;
import java.nio.channels.SocketChannel;

public class NIOServer {
    public static void main(String[] args) throws IOException {

        //创建一个端口进行监听
        InetSocketAddress inetSocketAddress = new InetSocketAddress(7001);
        ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();
        serverSocketChannel.socket().bind(inetSocketAddress);

        //创建一个buffer进行接收
        ByteBuffer byteBuffer = ByteBuffer.allocate(1024);

        while (true) {

            //接收来自客户端的请求
            SocketChannel socketChannel = serverSocketChannel.accept();

            int readCount = 0;

            while (readCount != -1) {
                try
                {
                    readCount = socketChannel.read(byteBuffer);
                }catch (Exception ex){
                    break;
                }

                //把缓冲区恢复为默认状态
                byteBuffer.rewind();
            }
        }
    }
}
```

**客户端代码**

```java
import java.io.FileInputStream;
import java.io.IOException;
import java.net.InetSocketAddress;
import java.nio.channels.FileChannel;
import java.nio.channels.SocketChannel;

public class NIOClient {
    public static void main(String[] args) throws IOException {
        SocketChannel socketChannel = SocketChannel.open();

        socketChannel.connect(new InetSocketAddress("127.0.0.1",7001));

        String fileName="FSCapture90.zip";

        FileChannel channel = new FileInputStream(fileName).getChannel();

        long startTime = System.currentTimeMillis();
        //在 linux 下一个 transferTo 方法就可以完成传输
        // 在 windows 下 一次调用 transferTo 只能发送 8m , 就需要分段传输文件, 而且要主要
        long transferCount= channel.transferTo(0, channel.size(), socketChannel);
        long endTime = System.currentTimeMillis()-startTime;
        System.out.println(" 发 送 的 总 的 字 节 数 =" + transferCount + " 耗 时 :" + (System.currentTimeMillis() - startTime));

        channel.close();
    }
}
```

##  3.5  page cache

页缓存（PageCache)是操作系统对文件的缓存，用来减少对磁盘的 I/O 操作，以页为单位的，内容就是磁盘上的物理块，页缓存能帮助**程序对文件进行顺序读写的速度几乎接近于内存的读写速度**，主要原因就是由于OS使用PageCache机制对读写访问操作进行了性能优化：

**页缓存读取策略**：当进程发起一个读操作 （比如，进程发起一个 read() 系统调用），它首先会检查需要的数据是否在页缓存中：

- 如果在，则放弃访问磁盘，而直接从页缓存中读取
- 如果不在，则内核调度块 I/O 操作从磁盘去读取数据，并读入紧随其后的少数几个页面（不少于一个页面，通常是三个页面），然后将数据放入页缓存中

> MySql中默认B+树每个节点大小为16K，每个page默认是4K，在读取page时默认会读入随后的几个page（通常是三个），所以每个节点设置为16K，可以确保每次可以把节点的数据都加载到内存中。

**页缓存写策略**：当进程发起write系统调用写数据到文件中，先写到页缓存，然后方法返回。此时数据还没有真正的保存到文件中去，Linux 仅仅将页缓存中的这一页数据标记为“脏”，并且被加入到脏页链表中

然后，由flusher 回写线程周期性将脏页链表中的页写到磁盘，让磁盘中的数据和内存中保持一致，最后清理“脏”标识。在以下三种情况下，脏页会被写回磁盘:

- 空闲内存低于一个特定阈值
- 脏页在内存中驻留超过一个特定的阈值时
- 当用户进程调用 sync() 和 fsync() 系统调用时

RocketMQ中，ConsumeQueue逻辑消费队列存储的数据较少，并且是顺序读取，在page cache机制的预读取作用下，Consume Queue文件的读性能几乎接近读内存，即使在有消息堆积情况下也不会影响性能，提供了2种消息刷盘策略：

- 同步刷盘：在消息真正持久化至磁盘后RocketMQ的Broker端才会真正返回给Producer端一个成功的ACK响应
- 异步刷盘，能充分利用操作系统的PageCache的优势，只要消息写入PageCache即可将成功的ACK返回给Producer端。消息刷盘采用后台异步线程提交的方式进行，降低了读写延迟，提高了MQ的性能和吞吐量

Kafka实现消息高性能读写也利用了页缓存，这里不再展开

