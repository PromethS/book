**目录**

[toc]
---
## IO系统
Java IO 是一套Java用来读写数据（输入和输出）的API。大部分程序都要处理一些输入，并由输入产生一些输出。Java提供了java.io包。

java中io系统分为Bio，Nio，Aio三种io模型。BIO是**面向流**，NIO是**面向缓冲区**。
### 阻塞IO模型（BIO）
最**传统**的IO模型，即在读写数据时会发生堵塞现象。当用户发起IO请求后，内核会查看数据是否就绪，如未就绪则会等待数据就绪，线程处于**堵塞**状态，用户线程交出CPU，当数据就绪后，内核会将数据拷贝到用户线程，并返回结果给用户线程，用户线程解除block状态。典型的例子为：data = socket.read()

#### 核心概念
BIO中操作的流主要有两大类，字节流和字符流，两类根据流的方向都可以分为**输入流**和**输出流**。

**字节流**主要用来处理字节或二进制对象，**字符流**用来处理字符文本或字符串

- 输入字节流：**InputStream**
- 输入字符流：**Reader**
- 输出字节流：**OutputStream**
- 输出字符流：**Writer**

在Linux中，当应用进程调用**recvfrom**方法调用数据的时候，如果内核没有把数据准备好不会立刻返回，而是会经历**等待数据准备就**绪，数据从**内核复制到用户空间**之后再返回，这期间应用进程一直阻塞直到返回，所以被称为阻塞IO模型

### 非阻塞IO模型
当用户发起IO请求后，并**不需要等待**，而是立刻得到一个结果，如果结果为error时，表示数据并未就绪，则会**再发起**一次IO请求，**直到**获取到结果数据。在非阻塞IO模型中，用户线程需要**不断询问**内核数据是否就绪，也就是说用户线程并不会交出CPU，而是一直while循环中读取数据并占用CPU，这样会导致**CPU占用率**非常高。

![image](https://520li.oss-cn-hangzhou.aliyuncs.com/img/clipboard-1590144192230.png)

### 多路复用IO模型（NIO）
- 目前使用比较多的IO模型，会有**一个线程不断去轮询**多个socket的状态，只有当socket真正有**读写事件**时，才真正调用实际的IO操作。只需要**一个线程就可以管理多个socket**，系统不需要建立新的进程或线程，并且只有真正有socket读写事件时，才会占用IO资源，大大减少了**资源的占用**。该模型适用于**连接数比较多**的场景

  在JAVA NIO中是通过selector.select()去查询每个通道是否有事件到达，如果没有事件则一直阻塞在那里。

  ![image](https://520li.oss-cn-hangzhou.aliyuncs.com/img/clipboard-1590144203505.png)

    - 相比较非阻塞IO模型**效率高**是因为，在非阻塞IO模型中是用户线程轮询数据状态，而多路复用IO模型，只有一个线程去轮询，并且是在**内核**进行的，这个效率比用户线程要高很多
    - **缺点**：当事件响应体很大时，就会导致后续的事件迟迟得不到处理，并且会影响新的事件轮询。

- **select、poll、epoll**  [参考](https://www.cnblogs.com/aspirant/p/9166944.html) 

  JDK中NIO使用多路复用的IO模型，通过把多个IO阻塞复用到一个**select的阻塞**上，实现系统在单线程中可以同时处理多个客户端请求，节省系统开销，在JDK1.4和1.5 update10版本之前，JDK的Selector基于**select/poll模型**实现，在JDK 1.5 update10以上的版本，底层使用**epoll**代替了select/poll
  
    - **select**==>时间复杂度O(n)
      
      当IO事件发生时，只能无差别轮询所有流，找出能操作的流。当**流的数量多了**以后，且只有少数socket有数据的时候，会导致效率下降。而且受限于所持有的**文件句柄(fd)** 数量，默认值是**1024**个。
      
    - **poll**==>时间复杂度O(n)
      
      与select**基本相同**，只是没有fd数量限制，因为采用的是**链表存储**
      
    - **epoll**==>时间复杂度O(1)
      
    基于**事件驱动**，会把哪个流发生的I/O事件通知我们，**不需要轮询**，只有Linux支持。
      
        - epoll支持打开的文件描述符数量不在受限制
        - select/poll使用**轮询**方式遍历整个文件描述符的集合，epoll基于每个文件描述符的**callback函数**回调
        - 两种**触发模式**
            - **水平触发**（EPOLLLT）,默认模式
            
              只要这个fd还有**数据可读**，每次epoll_wait都会返回它的事件，提醒用户程序去操作
            
            - **边缘触发**（EPOLLET）
            
              只会**提示一次**，直到下次再有数据流入之前都不会再提示了，无论fd中是否还有数据可读，所以在read时一定要**读取完数据**。
            
            - **比较**
            
              水平触发模式中，系统中如果有大量你不需要读写的就绪文件描述符，但每次调用epoll_wait都会返回，这样会大大降低处理程序检索自己关心的就绪文件描述符的效率。
            
              边缘触发模式中，当被监控的文件描述符上有可读写事件发生时，epoll_wait()才会通知处理程序去读写。**边缘触发比水平触发效率高，系统不会充斥大量你不关心的就绪文件描述符**
  
- **总结** 
    
    - 表面上看epoll的性能最好，但是在**连接数少且十分活跃**情况下，**epoll的性能不如select与poll**，因为epoll的通知机制需要很多函数回调
    - select/poll效率低是因为每次都需要轮询。但低效是相对的，视情况而定，也可通过良好的设计改善
    - select、poll、epoll**本质都是同步I/O**，因为都需要在数据就绪时自己负责进行读写，这个**读写的过程是堵塞**的。而**异步I/O是不需要自己读写**的，会把数据从内核直接拷贝到用户空间
    - **redis**使用的是epoll模式，Rocketmq、dubbo使用的是nio；**nginx**使用的是AIO（异步IO）

#### 核心概念
- **Buffer**（缓冲区）

  Buffer是一个**对象**，它包含一些要写入或者读出的数据，在NIO中**所有数据都是用缓存区**处理的，在读数据的时候要从缓冲区中读，写数据的时候会先写到缓冲区中，缓冲区本质上是一块可以写入数据，然后可以从中读取数据的**一个数组**，提供了对数据的结构化访问以及在内部维护了读写位置等信息。

- **Channel**（通道）

  Channel（通道）数据总是**从通道**读取到缓冲区，或者从缓冲区写入到通道中，Channel只负责**运输数据**，而操作数据是Buffer。通道与流类似，不同地方：

    - 通道是**双向**的，可以同时进行读，写操作，而流是单向流动的，只能写入或者读取
    - 流的读写是阻塞的，通道可以**异步读写**

- **Selector**（多路复用选择器）

  Selector是NIO编程的基础，主要作用就是将多个Channel**注册**到Selector上，如果Channel上发生读或写事件，Channel就处于就绪状态，就会被Selector轮询出来，然后通过SelectionKey就可以获取到已经就绪的Channel集合，进行IO操作了。

Selector与Channel，Buffer之间的关系：

![image](https://520li.oss-cn-hangzhou.aliyuncs.com/img/clipboard-1590144229672.png)

### 异步IO模型（AIO）
**最理想**的IO模型，当用户线程发起read请求时，立刻就可以去做其他事了，内核会等待数据准备完成，然后将数据从内核拷贝到用户线程，并给用户线程**发送一个信号**，告诉它read操作完成了。

只需要先发起一个请求，当接受到内核返回的成功信号时则表示IO操作完成了，可以直接去使用数据了。异步IO需要**底层操作系统**的支持，在JAVA 7中提供了Asynchronous IO

![image](https://520li.oss-cn-hangzhou.aliyuncs.com/img/clipboard-1590144249384.png)

#### 核心概念
aio通过**异步通道**实现异步操作，异步通道提供了两种方式获取操作结果：
- 通过**Future**类来获取异步操作的结果，不过要注意的是future.get()是阻塞方法，会阻塞线程
- 通过**回调**的方式进行异步，通过传入一个**CompletionHandler**的实现类进行回调，CompletionHandler定义了两个方法，completed和failed两方法分别对应成功和失败

## BIO详细解读
### 概述
- 使用**InputStreamReader**可以将输入字节流转化为输入字符流
```java
Reader reader  =  new InputStreamReader(inputStream);
```
- 使用**OutputStreamWriter**可以将输出字节流转化为输出字符流
```java
Writer writer = new OutputStreamWriter(outputStream)
```
- 在使用**字节流**的时候，**InputStream**和**OutputStream**都是**抽象类**，我们实例化的都是他们的子类，每一个子类都有自己的作用范围。
  ![image](https://520li.oss-cn-hangzhou.aliyuncs.com/img/clipboard-1590144267042.png)

- 在使用**字符流**的时候也是，**Reader**和**Writer**都是**抽象类**，我们实例化的都是他们的子类，每一个子类都有自己的作用范围。

  ![image](https://520li.oss-cn-hangzhou.aliyuncs.com/img/clipboard-1590144275763.png)

### 以读写文件为例
- 从数据源中**读取**数据
    - 输入字节流：**InputStream**
    ```java
    public static void main(String[] args) throws Exception{
        File file = new File("D:/a.txt");
        InputStream inputStream = new FileInputStream(file);
        byte[] bytes = new byte[(int) file.length()];
        inputStream.read(bytes);
        System.out.println(new String(bytes));
        inputStream.close();
    }
    ```
    - 输入字符流：**Reader**
    ```java
    public static void main(String[] args) throws Exception{
        File file = new File("D:/a.txt");
        Reader reader = new FileReader(file);
        char[] bytes = new char[(int) file.length()];
        reader.read(bytes);
        System.out.println(new String(bytes));
        reader.close();
    }
    ```
- **输出**到目标媒介
    - 输出字节流：**OutputStream**
    ```java
    public static void main(String[] args) throws Exception{
        String var = "hai this is a test";
        File file = new File("D:/b.txt");
        OutputStream outputStream = new FileOutputStream(file);
        outputStream.write(var.getBytes());
        outputStream.close();
    }
    ```
    - 输出字符流：**Writer**
    ```java
    public static void main(String[] args) throws Exception{
        String var = "hai this is a test";
        File file = new File("D:/b.txt");
        Writer writer = new FileWriter(file);
        writer.write(var);
        writer.close();
    }
    ```
    
### 缓存流
在使用InputStream的时候，都是**一个字节一个字节**的读或写，而BufferedInputStream为输入字节流提供了**缓冲区**，读数据的时候会一次读取一块数据放到缓冲区里，当缓冲区里的数据被读完之后，输入流会再次填充数据缓冲区，直到输入流被读完，有了缓冲区就能够提高很多**io速度**。
- **BufferedInputStream**

  使用方式将输入流包装到BufferedInputStream中
```java
/**
 * inputStream 输入流
 * 1024 内部缓冲区大小为1024byte
 */
BufferedInputStream bufferedInputStream = new BufferedInputStream(inputStream,1024);
```
- **BufferedOutputStream**

  BufferedOutputStream可以为输出字节流提供缓冲区，作用与BufferedInputStream类似
```java
/**
 * outputStream 输出流
 * 1024 内部缓冲区大小为1024byte
 */
BufferedOutputStream bufferedOutputStream = new BufferedOutputStream(outputStream,1024);
```
- **BufferedReader**
```java
BufferedReader bufferedReader = new BufferedReader(reader,1024);
```
- **BufferedWriter**
```java
BufferedWriter bufferedWriter = new BufferedWriter(writer,1024);
```

## NIO详细解读
使用原生NIO类库十分复杂，NIO的类库和Api繁杂，使用麻烦，需要对网络编程十分熟悉，才能编写出高质量的NIO程序，所以并不建议直接使用原生NIO进行网络编程，而是使用一些成熟的框架，比如**Netty**
### 复制文件为例
```java
FileInputStream fileInputStream=new FileInputStream(new File(src));
FileOutputStream fileOutputStream=new FileOutputStream(new File(dst));
//获取输入输出channel通道
FileChannel inChannel=fileInputStream.getChannel();
FileChannel outChannel=fileOutputStream.getChannel();
//创建容量为1024个byte的buffer
ByteBuffer buffer=ByteBuffer.allocate(1024);
while(true){
    //从inChannel里读数据，如果读不到字节了就返回-1，文件就读完了
    int eof =inChannel.read(buffer);
    if(eof==-1){
        break;
    }
    //将Buffer从写模式切换到读模式
    buffer.flip();
    //开始往outChannel写数据
    outChannel.write(buffer);
    //清空buffer
    buffer.clear();
}
inChannel.close();
outChannel.close();
fileInputStream.close();
fileOutputStream.close();
```
### Socket-服务端
```java
//创建多路复用选择器Selector
Selector selector=Selector.open();
//创建一个通道对象Channel，监听9001端口
ServerSocketChannel channel = ServerSocketChannel.open().bind(new InetSocketAddress(9001));
//设置channel为非阻塞
channel.configureBlocking(false);
//
/**
 * 1.SelectionKey.OP_CONNECT：连接事件
 * 2.SelectionKey.OP_ACCEPT：接收事件
 * 3.SelectionKey.OP_READ：读事件
 * 4.SelectionKey.OP_WRITE：写事件
 *
 * 将channel绑定到selector上并注册OP_ACCEPT事件
 */
channel.register(selector,SelectionKey.OP_ACCEPT);

while (true){
    //只有当OP_ACCEPT事件到达时，selector.select()会返回（一个key），如果该事件没到达会一直阻塞
    selector.select();
    //当有事件到达了，select()不在阻塞，然后selector.selectedKeys()会取到已经到达事件的SelectionKey集合
    Set keys = selector.selectedKeys();
    Iterator iterator = keys.iterator();
    while (iterator.hasNext()){
        SelectionKey key = (SelectionKey) iterator.next();
        //删除这个SelectionKey，防止下次select方法返回已处理过的通道
        iterator.remove();
        //根据SelectionKey状态判断
        if (key.isConnectable()){
            //连接成功
        } else if (key.isAcceptable()){
            /**
             * 接受客户端请求
             *
             * 因为我们只注册了OP_ACCEPT事件，所以有客户端链接上，只会走到这
             * 我们要做的就是去读取客户端的数据，所以我们需要根据SelectionKey获取到serverChannel
             * 根据serverChannel获取到客户端Channel，然后为其再注册一个OP_READ事件
             */
            // 1，获取到ServerSocketChannel
            ServerSocketChannel serverChannel = (ServerSocketChannel) key.channel();
            // 2，因为已经确定有事件到达，所以accept()方法不会阻塞
            SocketChannel clientChannel = serverChannel.accept();
            // 3，设置channel为非阻塞
            clientChannel.configureBlocking(false);
            // 4，注册OP_READ事件
            clientChannel.register(key.selector(),SelectionKey.OP_READ);
        } else if (key.isReadable()){
            // 通道可以读数据
            /**
             * 因为客户端连上服务器之后，注册了一个OP_READ事件发送了一些数据
             * 所以首先还是需要先获取到clientChannel
             * 然后通过Buffer读取clientChannel的数据
             */
            SocketChannel clientChannel = (SocketChannel) key.channel();
            ByteBuffer byteBuffer = ByteBuffer.allocate(BUF_SIZE);
            long bytesRead = clientChannel.read(byteBuffer);
            while (bytesRead>0){
                byteBuffer.flip();
                System.out.println("client data ："+new String(byteBuffer.array()));
                byteBuffer.clear();
                bytesRead = clientChannel.read(byteBuffer);
            }

            /**
             * 我们服务端收到信息之后，我们再给客户端发送一个数据
             */
            byteBuffer.clear();
            byteBuffer.put("客户端你好，我是服务端，你看这NIO多难".getBytes("UTF-8"));
            byteBuffer.flip();
            clientChannel.write(byteBuffer);
        } else if (key.isWritable() && key.isValid()){
            //通道可以写数据
        }
    }
}
```
### Socket-客户端
```java
Selector selector = Selector.open();
SocketChannel clientChannel = SocketChannel.open();
//将channel设置为非阻塞
clientChannel.configureBlocking(false);
//连接服务器
clientChannel.connect(new InetSocketAddress(9001));
//注册OP_CONNECT事件
clientChannel.register(selector, SelectionKey.OP_CONNECT);
while (true){
    //如果事件没到达就一直阻塞着
    selector.select();
    Iterator<SelectionKey> iterator = selector.selectedKeys().iterator();
    while (iterator.hasNext()){
        SelectionKey key = iterator.next();
        iterator.remove();
        if (key.isConnectable()){
            /**
             * 连接服务器端成功
             *
             * 首先获取到clientChannel，然后通过Buffer写入数据，然后为clientChannel注册OP_READ时间
             */
            clientChannel = (SocketChannel) key.channel();
            if (clientChannel.isConnectionPending()){
                clientChannel.finishConnect();
            }
            clientChannel.configureBlocking(false);
            ByteBuffer byteBuffer = ByteBuffer.allocate(1024);
            byteBuffer.clear();
            byteBuffer.put("服务端你好，我是客户端，你看这NIO难吗".getBytes("UTF-8"));
            byteBuffer.flip();
            clientChannel.write(byteBuffer);
            clientChannel.register(key.selector(),SelectionKey.OP_READ);
        } else if (key.isReadable()){
            //通道可以读数据
            clientChannel = (SocketChannel) key.channel();
            ByteBuffer byteBuffer = ByteBuffer.allocate(BUF_SIZE);
            long bytesRead = clientChannel.read(byteBuffer);
            while (bytesRead>0){
                byteBuffer.flip();
                System.out.println("server data ："+new String(byteBuffer.array()));
                byteBuffer.clear();
                bytesRead = clientChannel.read(byteBuffer);
            }
        } else if (key.isWritable() && key.isValid()){
            //通道可以写数据
        }
    }
}
```

## AIO详细解读
### Socket-服务端
```java
AsynchronousServerSocketChannel server = AsynchronousServerSocketChannel.open().bind(new InetSocketAddress("127.0.0.1", 9001));
//异步接受请求
server.accept(null, new CompletionHandler<AsynchronousSocketChannel, Void>() {
    //成功时
    @Override
    public void completed(AsynchronousSocketChannel result, Void attachment) {
        try {
            ByteBuffer buffer = ByteBuffer.allocate(1024);
            buffer.put("我是服务端，客户端你好".getBytes());
            buffer.flip();
            result.write(buffer, null, new CompletionHandler<Integer, Void>(){
                @Override
                public void completed(Integer result, Void attachment) {
                    System.out.println("服务端发送消息成功");
                }

                @Override
                public void failed(Throwable exc, Void attachment) {
                    System.out.println("发送失败");
                }
            });

            ByteBuffer readBuffer = ByteBuffer.allocate(1024);
            result.read(readBuffer, null, new CompletionHandler<Integer, Void>() {
                //成功时调用
                @Override
                public void completed(Integer result, Void attachment) {
                    System.out.println(new String(readBuffer.array()));
                }
                //失败时调用
                @Override
                public void failed(Throwable exc, Void attachment) {
                    System.out.println("读取失败");
                }
            });

        } catch (Exception e) {
            e.printStackTrace();
        }
    }
    //失败时
    @Override
    public void failed(Throwable exc, Void attachment) {
        exc.printStackTrace();
    }
});
//防止线程执行完
TimeUnit.SECONDS.sleep(1000L);
```
### Socket-客户端
```java
AsynchronousSocketChannel client = AsynchronousSocketChannel.open();
Future<Void> future = client.connect(new InetSocketAddress("127.0.0.1", 9001));
//阻塞，获取连接
future.get();

ByteBuffer buffer = ByteBuffer.allocate(1024);
//读数据
client.read(buffer, null, new CompletionHandler<Integer, Void>() {
    //成功时调用
    @Override
    public void completed(Integer result, Void attachment) {
        System.out.println(new String(buffer.array()));
    }
    //失败时调用
    @Override
    public void failed(Throwable exc, Void attachment) {
        System.out.println("客户端接收消息失败");
    }
});

ByteBuffer writeBuffer = ByteBuffer.allocate(1024);
writeBuffer.put("我是客户端，服务端你好".getBytes());
writeBuffer.flip();
//阻塞方法
Future<Integer> write = client.write(writeBuffer);
Integer r = write.get();
if(r>0){
    System.out.println("客户端消息发送成功");
}
//休眠线程
TimeUnit.SECONDS.sleep(1000L);
```
