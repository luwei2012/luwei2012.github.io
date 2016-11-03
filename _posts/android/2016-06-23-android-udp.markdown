---
layout: post
title: "Android UDP服务"
modified:
categories: android
description: "我不重复制造轮子，我只是代码的搬运工"
tags: [UDP]
image:
  feature:
  credit:
  creditlink:
comments:
share:
date: 2016-06-23T09:19:36+08:00
---

### 已经有的轮子：KryoNet

[KryoNet](https://github.com/EsotericSoftware/kryonet) is a Java library that provides a clean and simple API for efficient TCP and UDP client/server network 
communication using NIO. KryoNet uses the Kryo serialization library to automatically and efficiently transfer object graphs across the network.

正如KryoNet的描述，这个框架已经提供了可用的udp服务器和客户端实现，但是使用的是[Kryo](https://github.com/EsotericSoftware/kryo)来做字节的序列化和反序列化。
我只是想要一个UDP服务器的demo，不想绑定Kryo，而且由于项目需求，字节码解析协议需要定制，于是就有了这篇博客。


### 一个完整的UDP服务器Demo应该有什么？

先帖一发[代码库](https://luoqiaowei@bitbucket.org/luoqiaowei/udpdemo.git)。这里使用的bibucket建仓库，实在是点不开github...
一个完整的UDP服务器应该能做到下面的事情：

Server端：

1. 端口监听。防止dos攻击、报文过滤等等都应该做在这一层。
2. 连接池。接收到UDP报文后肯定需要创建连接然后发送ACK或者其他数据，如果不考虑性能，连接池可以省略，我的demo里面也没有实现，都是直接new出来的。
3. 连接记录。虽然都说UDP是无连接的协议，但是大部分需求都是需要记录UDP连接的，这样才能发送广播之类的消息；同时如果想要使用UDP来实现一套IM系统，肯定还是需要连接记录的。
4. 数据解析处理。
5. 可靠的数据传递。(超时重发、超重发次数离线)
6. 心跳检测。独立的线程遍历连接记录，发现有心跳超时的记录就"断开连接"，连接池回收。

Client端：

1. 端口监听。逻辑跟服务器一样一样。
2. 建立连接。不要以为UDP服务无连接就不需要这一步，一般都是发送一个心跳包，等待服务器的ACK，如果等到了就说明链路通了，否则需要提示用户无法建立连接
3. 可靠的数据传递。超时重发、超重发次数离线并且重连、超重连次数通知上层逻辑处理。
4. 数据解析处理。
5. 心跳。

目前我的Demo里面实现的功能很简陋，Server端什么都没有，收到消息直接返回ACK。这个Server本身也只是在项目中用来测试客户端是否正确，并没有花时间来做。Client端相对丰满点，实现了：

1. 端口监听。没有防DOS攻击等复杂逻辑，简单地收取报文。
2. 建立连接。
3. 可靠的数据传递。只做了超时重发、超重发次数重连，没有保存本地数据库。
4. 数据解析处理。
5. 心跳

### UDP连接框架
<figure>
	<img src="/images/android/UDP/UDP_Connection.png" alt="UDP连接ER图">
	<figcaption>UDP连接ER图</figcaption>
</figure>
首先需要熟悉下[Java里面的NIO](http://tutorials.jenkov.com/java-nio/selectors.html)，有一个对应的[翻译](http://ifeve.com/overview/)，最主要的是要看到第9篇，
这样就知道了如果搭建一个简单地UDP服务器了。
这些最基础的UDP连接，对应封装在UDPConnection类中：
{% highlight logos %}
public class UDPConnection {
    InetSocketAddress connectedAddress;
    DatagramChannel datagramChannel;
    final ByteBuffer readBuffer, writeBuffer;
    private final Serialization serialization;
    private SelectionKey selectionKey;
    private final Object writeLock = new Object();
    ......
}
{% endhighlight %}
从成员变量里面就可以看出来，与NIO的教程里面说的一样有三个角色：`buffer`、`Channel`和`SelectionKey`。同时还有另一个角色：`Serialization`，也就是我们下面着重会讲的数据解析协议，这个类规定了

1. 如何从ByteBuffer转化为我们需要的Java对象
2. 如何从Java对象转化为ByteBuffer

{% highlight logos %}
    public UDPConnection(Serialization serialization, int writeBufferSize, int objectBufferSize) {
        this.serialization = serialization;
        readBuffer = ByteBuffer.allocate(objectBufferSize);
        writeBuffer = ByteBuffer.allocateDirect(writeBufferSize);
    }

    public void bind(Selector selector, InetSocketAddress localPort) throws IOException {
        close();
        readBuffer.clear();
        writeBuffer.clear();
        try {
            datagramChannel = selector.provider().openDatagramChannel();
            datagramChannel.socket().bind(localPort);
            datagramChannel.configureBlocking(false);
            selectionKey = datagramChannel.register(selector, SelectionKey.OP_READ);
        } catch (IOException ex) {
            close();
            throw ex;
        }
    }
{% endhighlight %}
先看构造函数和bind。构造函数中传入了具体的`Serialization`，以及各个缓存的大小。
bind函数作用就是在给定的端口上监听UDP报文，这里使用`selector.provider().openDatagramChannel()`而不是`DatagramChannel.open()`还是有一点区别的，方便了我们扩展代码。
{% highlight logos %}
    public void connect(Selector selector, InetSocketAddress remoteAddress) throws IOException {
        close();
        readBuffer.clear();
        writeBuffer.clear();
        try {
            datagramChannel = selector.provider().openDatagramChannel();
            datagramChannel.configureBlocking(false);
            datagramChannel.socket().bind(null);
            datagramChannel.socket().connect(remoteAddress);
            selectionKey = datagramChannel.register(selector, SelectionKey.OP_READ);
            connectedAddress = remoteAddress;
        } catch (Exception ex) {
            close();
            IOException ioEx = new IOException("Unable to connect to: " + remoteAddress);
            ioEx.initCause(ex);
            throw ioEx;
        }
    }
{% endhighlight %}
看到这个方法可能会让人有点困惑，UDP是无连接的，为何还会有connect方法？这里的connect并不是TCP中的建立连接的意思，而是把Channel与该地址绑定，这样以后写数据的时候就会默认写给connect中传入的地址。具体描述可参见[官方文档](http://tool.oschina.net/uploads/apidocs/jdk-zh/java/nio/channels/DatagramChannel.html#connect(java.net.SocketAddress))
{% highlight logos %}
       public InetSocketAddress readFromAddress() throws IOException {
            DatagramChannel datagramChannel = this.datagramChannel;
            if (datagramChannel == null) throw new SocketException("ConnectionWrapper is closed.");
            if (!datagramChannel.isConnected())
                return (InetSocketAddress) datagramChannel.receive(readBuffer); // always null on Android >= 5.0
            datagramChannel.read(readBuffer);
            return connectedAddress;
        }
    
        public PacketMessage readObject() {
            readBuffer.flip();
            try {
                try {
                    PacketMessage object = serialization.read(readBuffer);
                    if (readBuffer.hasRemaining())
                        throw new Exception("Incorrect number of bytes (" + readBuffer.remaining()
                                + " remaining) used to deserialize object: " + object);
                    if (DEBUG) info("readObject object : " + object.getClass().getName());
                    return object;
                } catch (Exception ex) {
                    throw new RuntimeException("Error during deserialization.", ex);
                }
            } finally {
                readBuffer.clear();
            }
        }
    
        /**
         * This method is thread safe.
         */
        public int send(PacketMessage object, SocketAddress address) throws IOException {
            DatagramChannel datagramChannel = this.datagramChannel;
            if (datagramChannel == null) throw new SocketException("ConnectionWrapper is closed.");
            synchronized (writeLock) {
                try {
                    try {
                        if (DEBUG) info("send object : " + object.getClass().getName());
                        serialization.write(writeBuffer, object);
                    } catch (Exception ex) {
                        throw new RuntimeException("Error serializing object of type: " + object.getClass().getName(), ex);
                    }
                    writeBuffer.flip();
                    int length = writeBuffer.limit();
                    datagramChannel.send(writeBuffer, address);
                    boolean wasFullWrite = !writeBuffer.hasRemaining();
                    return wasFullWrite ? length : -1;
                } finally {
                    writeBuffer.clear();
                }
            }
        }
{% endhighlight %}
这三个方法中`readFromAddress`是从通道中读取数据到`ByteBuffer`，`readObject`使用规定的`Serialization`从`ByteBuffer`转为Java对象，`send`将Java对象写入通道。有了这些基础方法后就可以在上层设计UDP连接框架了。

这里使用了静态代理的设计思路，`UDPConnection`是`IConnection`的一个实现，`ConnectionWrapper`是`IConnection`的一个代理；`ConnectionWrapper`中使用的是`UDPConnection`的实现。`ConnectionWrapper`中还有一个属性是`EndPoint`类型的对象，`ConnectionWrapper`中主要使用的是`EndPoint`的`reconnect`实现。

EndPoint定义了一个UDP端的控制接口。

1. `Serialization getSerialization();` 获取数据序列化的实现
2. `addListener(Listener listener);` 添加上面所说的五中状态的监听
3. `removeListener(Listener listener);`删除状态监听
4. `run();`继承的`Runnable`，读取消息的循环，运行在独立线程中
5. `reconnect();`重连
6. `start();`开启一个线程运行`run()`
7. `stop();`停止run()并重置Selector状态
8. `close();`状态置为关闭，关闭通道，重置`Selector`
9. `update(int timeout);`读取一次数据的实现，`run`中会循环调用该方法
10. `getUpdateThread();`获取读取数据的线程，这是为了判断建立连接的线程与读取数据的线程是否相同，建立连接的过程会block线程，因此不能使用同一个线程，否则会卡死。

这些接口里面只有1、2、3、5、6、7、8是提供给外部控制Client状态的。

下面我们来走一遍建立连接的过程:

1. 创建Client

   `client = new Client() {};` 
        
2. 配置状态回调，启动Client，开始监听UDP报文

   `client.start();`
   
   `client.addListener(new com.transport.Listener());`

3. 开启另一个线程开始连接服务器

{% highlight logos %}
   new Thread("Connect") {
       public void run() {
           try {
               client.connect(host, Integer.parseInt(port)); 
           } catch (IOException ex) {
               ex.printStackTrace();
               System.exit(1);
           }
       }
   }.start();
{% endhighlight %}   

需要注意的是必须要先启动Client后在建立连接，因为我们这里建立连接做的事情是1.给Channel绑定服务器的地址 2.给服务器发送一个心跳包，然后block线程，等待心跳ACK，超时断开连接；收到ACK唤醒线程，通知状态监听

服务器的逻辑与之类似：

1. 创建Server

   `server = new Server() {};`
   
2. 配置状态回调，启动Client，开始监听UDP报文，收到报文后默认返回ACK

   `server.addListener(new Listener());`
   
   `server.start();` 
        
3. 绑定端口，服务器没有建立连接的步骤，不会造成线程block

   `server.bind(54555);`


### 可靠地数据传输

要实现可靠地数据传输需要实现两个特性：1.断线重连 2.数据包ACK，超时重发 3.心跳

#### 断线重连

断线重连发生在下面几种情况下：

1. 从通道读取数据失败。我们认为数据通道被破坏了，需要重新创建。
2. ACK检查时发现了超出重发次数的数据包。我们认为服务器失联了，需要重联，确认连接可用。
3. 发送数据包失败。与1一样，认为数据通道破坏了。

对应的代码调用可以全局搜索`reconnect();`的调用

#### 超时重发
上面在介绍`EndPoint`的时候说过了`update(int timeout)`，运行在独立的线程中，这个方法做了3件事情：

1. 检测`Selector`是否有数据通道处于ready状态，如果有，进入读取数据的逻辑。
2. 检测并发送心跳包。
3. 检测等待ACK的包，是否有等待的包超时了，有并且已经超出重发次数上线，重连；没有超出重发次数上线则重发数据包，并将该数据包移出等待ACK的列表。

我们在发送数据包时，需要将每一个发送成功的数据包保存在等待ACK的列表中，在收到了对应的ACK后才能将之移除；重连时需要把还在等待ACK的数据包重发一次。

#### 心跳

心跳也是在`update`函数中检测的，目前定义的是一分钟一次，在update函数中判断距离上次发送心跳的时间，到了一分钟就发送一次心跳，然后重置最近一次发送心跳的时间。这里需要注意的
是重置发送心跳的时间是写在发送数据包的函数中的，因为我们有可能会出现心跳包重发的情况，因此在发送数据包的时候来判断是否属心跳包，以此为依据来更新最新一次数据包的发送时间。
重连的时候不会发送还在等待ACK的心跳包，心跳包会直接抛弃。

### 数据解析协议 

首先需要看下序列化接口的定义：

{% highlight logos %}
public interface Serialization {

    void write(ByteBuffer buffer, PacketMessage packet);

    PacketMessage read(ByteBuffer buffer);

    /**
     * The fixed number of bytes that will be written by {@link #writeLength(ByteBuffer, int)} and read by
     * {@link #readLength(ByteBuffer)}.
     */
    int getLengthLength();

    void writeLength(ByteBuffer buffer, int length);

    int readLength(ByteBuffer buffer);
}
{% endhighlight %} 

这里我们规定了`ByteBuffer`与`PacketMessage`之间的互相转化接口，具体实现写在`PacketSerialization`中。这里不太想赘述了，代码中已经写得很详细了。

### 如何运行Demo

我们虽然导入Android Studio后是一个Android工程，但是不能启动模拟器来运行，因为Android部分的代码都被我删掉了。我们可以运行代码里面的ChatClient
和ChatServer两个Java勒种的main方法，运行方法不需要我再说了吧，附上截图：
<figure>
	<img src="/images/android/UDP/ChatServer.png" alt="服务器运行图"> 
	<img src="/images/android/UDP/ChatClient01.png" alt="客户端设置服务器地址"> 
	<img src="/images/android/UDP/ChatClient02.png" alt="客户端设置服务器端口"> 
	<img src="/images/android/UDP/ChatClient03.png" alt="客户端运行图"> 
</figure>