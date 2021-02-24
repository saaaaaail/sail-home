---
title: io详解
date: 2020-07-24 21:47:47
tags:
- 计算机网络
- io
---

# 网络io详解
## io模型
一个数据操作通常包括:
- 等待数据到达，复制到内核的缓冲区
- 把数据从内核缓冲区复制到应用程序缓冲区
在Linux网络模型中介绍了5种io模型:
- 阻塞式io
- 非阻塞式io
- 多路复用io
- 信号驱动io
- 异步io


### 阻塞式io
用户线程/进程发出io系统调用read，cpu会检查内核空间数据有没有复制到内核缓冲区，如果数据ok了，就将数据由内核缓冲区复制到应用程序的缓冲区，应用程序读出数据；如果数据还没有准备好，用户线程/进程就会阻塞并让出cpu，直到内核将数据填充到应用程序的缓冲区并唤醒进程/线程。

此过程中进程/线程会阻塞，但是cpu会调度其他进程执行，所以cpu的利用率挺高的。只是执行io系统调用的进程/线程会一直阻塞住。

![阻塞式io](阻塞式io.png)

### 非阻塞式io
用户进程发起io系统调用，如果数据没有准备好，就立即返回error状态码。应用程序可以继续执行。但是如果应用程序一定需要io数据的话，就必须不停地轮询进行io系统调用，一旦获得数据就跳出自旋，使用数据。这种方式虽然非阻塞了，但是轮询又浪费了cpu，cpu效率不高。

![非阻塞式io](非阻塞式io.png)

### 多路复用io
这种方式多路指的是多个io资源，复用指的是一个线程管理多个io资源。当这些资源没有一个是可读可写的时候，线程会被阻塞并等待，当一个io资源可读可写说明数据已经就绪，复制到了内核缓冲区中，唤醒线程以非阻塞的方式read，将内核缓冲区的数据复制到用户程序缓冲区中。

可以使用多进程/多线程+阻塞io的方式实现多路复用io的情形，但是如果连接数过多，线程创建与切换会影响cpu的使用率。

![多路复用io](多路复用io.png)

### 信号驱动io
在信号驱动io模型中，会给socket注册一个信号驱动的函数sigaction，数据未就绪，进程会继续执行，内核在数据到达时向应用进程发送SIGIO信号，然后信号处理函数会将数据从内核复制到应用进程中，然后在这个函数中进行io处理。这种方式一般用于UDP，类似于待处理的数据。

### 异步io
在异步io模型中，用户线程发起一个aio_read()以后会立即返回，线程不会阻塞可以去做其他的事情，内核会等待数据准备完成，然后将数据由内核拷贝到应用程序，这一切完成以后会通知用户线程，用户线程可以直接去使用数据了。

异步io与信号驱动io的区别是:
- 在异步io模型中，收到信号表示io两阶段操作都已完成，直接使用数据即可。
- 在信号驱动io模型中，收到信号表示数据已经准备好，需要用户线程进行read系统调用去获取数据。

![异步io](异步io.png)

### 总结
  
前四个io模型均为同步io，只有最后一个是真正的异步io。

- 同步io中，数据由内核缓冲区复制到应用程序缓冲区中，应用程序会阻塞。
- 异步io中，io操作第二阶段也不会阻塞。

在同步io中，只有阻塞式io的第一阶段会被阻塞。

## io多路复用详解
select/poll/epoll

### 谈谈select
select方法是linux里的一种系统调用方法。select的fd_set是一个1024大小的长整型数组。select与poll都是水平触发模式。即本次轮询结束时有就绪的fd没有处理，下一次再调用会立即返回。
- 将多个socket注册到select中的fd_set中
- 使用copy_from_user将fd集合从用户空间拷贝到内核空间
- 注册文件描述符的回调函数__pollwait
- 然后就是轮询fd集合调用poll方法，也就是上面的回调函数，回调函数会返回一个读写是否就绪的掩码，根据这个掩码给fd_set赋值
- 如果遍历完fd_set后没有就绪的io设备，就挂起当前进程，直到有设备就绪或者主动超时的时候唤醒当前进程，重新遍历fd_set，判断有没有就绪的fd
- 如果有就绪的fd了，select立即返回，然后还要再去遍历一遍fd_set获得就绪的文件描述符，读取文件，这一步就是将fd_set由内核再拷贝到用户空间。

#### select缺点
- select每次执行都要将fd集合由用户空间拷贝到内核空间，开销太大。
- 每次都是轮询整个fd集合，效率不好。
- 能一次管理的文件描述符有限1024个。

### 谈谈poll
poll与select类似，解决了select的最后一个缺点就是数量限制，fd_set由数组修改为链表实现。

### 谈谈epoll
epoll解决了select前两个缺点。将原先的一个函数拆分为3部分:
- 调用epoll_create方法创建一个epoll对象
- 调用epoll_ctl向epoll对象中添加socket集合
- 调用epoll_wait等待事件发生

```c++
//这就是一个epoll对象
struct eventpoll {
　　...
　　/*红黑树的根节点，这棵树中存储着所有添加到epoll中的事件，
　　也就是这个epoll监控的事件*/
　　struct rb_root rbr;
　　/*双向链表rdllist保存着将要通过epoll_wait返回给用户的、满足条件的事件*/
　　struct list_head rdllist;
　　...
};

//对每个事件都会包装成epitem节点
struct epitem {
　　...
　　//红黑树节点
　　struct rb_node rbn;
　　//双向链表节点
　　struct list_head rdllink;
　　//事件句柄等信息
　　struct epoll_filefd ffd;
　　//指向其所属的eventepoll对象
　　struct eventpoll *ep;
　　//期待的事件类型
　　struct epoll_event event;
　　...
}; // 这里包含每一个事件对应着的信息。

```

#### epoll_create
在调用epoll_create方法的时候，内核会构建一个eventpoll结构体，包含两个与epoll相关的成员:
- struct rb_root rbr 是构建红黑树的头节点，用于存储epoll_ctl传来的socket
- struct list_head rdllist 是构建一个双向链表，保存epoll_wait返回给用户的、满足就绪条件的事件

#### epoll_ctl
epoll_ctl就是epoll的事件注册函数。用于将文件描述符的添加到epoll对象的红黑树中，这样的话，添加、删除、修改事件的时候从红黑树中寻找也非常有效率，然后向内核注册事件的回调函数，用于数据就绪中断发生的时候将事件节点添加到双端链表里面。这个方法在调用时会将所有的fd拷贝到内核空间，而不是在epoll_wait的时候重复拷贝，保证了整个过程中只拷贝一次。

#### epoll_wait
epoll_wait就会遍历双链表里面有没有就绪的事件节点，不用再遍历所有的文件描述符集合，然后将这里面的事件复制到用户态内存的event数组中，然后将事件数量返回给用户，然后应用程序就可以遍历event数组，进行对应事件的读写操作。  

例子看看：   
```c++
    for( ; ; )
    {
        nfds = epoll_wait(epfd,events,20,500);
        for(i=0;i<nfds;++i)
        {
            if(events[i].data.fd==listenfd) //有新的连接
            {
                connfd = accept(listenfd,(sockaddr *)&clientaddr, &clilen); //accept这个连接
                ev.data.fd=connfd;
                ev.events=EPOLLIN|EPOLLET;
                epoll_ctl(epfd,EPOLL_CTL_ADD,connfd,&ev); //将新的fd添加到epoll的监听队列中
            }
            else if( events[i].events&EPOLLIN ) //接收到数据，读socket
            {
                n = read(sockfd, line, MAXLINE)) < 0    //读
                ev.data.ptr = md;     //md为自定义类型，添加数据
                ev.events=EPOLLOUT|EPOLLET;
                epoll_ctl(epfd,EPOLL_CTL_MOD,sockfd,&ev);//修改标识符，等待下一个循环时发送数据，异步处理的精髓
            }
            else if(events[i].events&EPOLLOUT) //有数据待发送，写socket
            {
                struct myepoll_data* md = (myepoll_data*)events[i].data.ptr;    //取数据
                sockfd = md->fd;
                send( sockfd, md->ptr, strlen((char*)md->ptr), 0 );        //发送数据
                ev.data.fd=sockfd;
                ev.events=EPOLLIN|EPOLLET;
                epoll_ctl(epfd,EPOLL_CTL_MOD,sockfd,&ev); //修改标识符，等待下一个循环时接收数据
            }
            else
            {
                //其他的处理
            }
        }
    }
```

总的来说:   
select/poll/epoll都是对linux里设备驱动的poll函数的封装，用来在注册事件的回调函数，并且将当前进程挂当对应设备驱动的等待队列上，由于会依次循环不同事件的poll函数，所以认为每一个事件如果没有就绪都会将进程添加到其等待队列里。

##### 水平触发与边缘触发
因为读写操作是用户去做的，如果epoll_wait返回了就绪的读写事件，但是用户不去读写的话，就会面临一些问题，所以用下面两种模式来约束一下。
- 水平触发模式 :只要这个文件描述的缓冲区数据没有读干净，那么每次epoll_wait都会返回读事件，提醒用户操作。
- 边缘触发模式 :当有io事件发生，如果是可读的，就必须将这个文件描述符一直读到空，否则下次不会返回余下的数据，会丢掉事件，但是数据遗留在缓冲区，所以如果读写操作不是非阻塞的，那么最后一次没有数据在就绪的时候就会阻塞住。

## java中的网络io模型
java中的io模型包括BIO、NIO、AIO三种。
### BIO
BIO是java中Socket的经典的网络编程模型，对于客户端而言就是在某时间去与服务端建立连接，在单线程下对于服务端而言包括下面步骤:
- socket.accept() 这里会阻塞住等待客户端的连接
- socket.getIputStream().read(buffer) 这个方法就是一个阻塞式io，如果数据没有准备好（比如客户端由键盘输入数据，输入完成并发送到达之前，服务端都是阻塞的），这里会阻塞住等待。
- 接受到数据后处理，客户端断开连接

可以看出，BIO在一次通信过程中会阻塞两次。  
所以BIO要进行并发必须是一个io连接对应一个线程，所以BIO服务器的问题就是大量的连接建立但是又不发送消息长时间占用连接，会使服务器性能下降。

```java
    //BIO 服务端代码
    public class Server {
        public static void main(String[] args) {
        byte[] buffer = new byte[1024];
            try{
                ServerSocket serverSocket = newServerSocket(8080);
                System.out.println("服务器已启动并监听8080端口");
                while(true) {
                    System.out.println();
                    System.out.println("服务器正在等待连接...");
                    Socket socket = serverSocket.accept();
                    System.out.println("服务器已接收到连接请求...");
                    System.out.println();
                    System.out.println("服务器正在等待数据...");
                    socket.getInputStream().read(buffer);
                    System.out.println("服务器已经接收到数据");
                    System.out.println();
                    String content = newString(buffer);
                    System.out.println("接收到的数据:"+ content);
                }
            } catch(IOException e) {
                // TODO Auto-generated catch block
                e.printStackTrace();
            }
        }
    }

    //BIO 客户端代码
    public class Consumer {
        public static void main(String[] args) {
            try{
                Socket socket = newSocket("127.0.0.1",8080);
                String message = null;
                Scanner sc = newScanner(System.in);
                message = sc.next();
                socket.getOutputStream().write(message.getBytes());
                socket.close();
                sc.close();
            } catch(IOException e) {
                // TODO Auto-generated catch block
                e.printStackTrace();
            }
        }
    }
```
### NIO
jdk1.4开始引入了NIO类库，这里的NIO指的是New IO，NIO有标准输入输出NIO与网络编程NIO，主要使用Selector多路复用器来实现，windows通过select实现，linux通过epoll实现。

io是以字节流的方式处理数据，nio是以块的方式处理数据。  
其中Buffer和Channel是标准NIO的核心对象，写数据或者读数据都要通过一个Channel对象，而Buffer比较类似于要发的数据的载体。在NIO中准备发送的数据要先放到Buffer中，Buffer通过Channel传递数据，然后再通过Buffer从Channel中读取数据，再从Buffer中取数据，最后清理Buffer。


#### Buffer
Buffer是一个数组，通常是字节数组，但是也有其他基本数据类型的Buffer。   
Buffer中使用几个变量来保存当前状态
- capacity 容量 缓冲区能容纳的元素数量
- position 当前位置，指向的是下一次发生读取、写入操作的起始位置
- limit 界限，指的是Buffer里有效数据结束以后的第一个索引的位置

```java
//介绍完几个参数来看看方法flip，将limit设置为之前数据的最后一个，将position设置为0，意味着后面的操作从第一个数据开始，一直到所有有效数据结束
public final Buffer flip() {
    limit = position;
    position = 0;
    mark = -1;
    return this;
}
//clear 将postion置0，limit置为最大值
public final Buffer clear() {
    position = 0;
    limit = capacity;
    mark = -1;
    return this;
}
```

Buffer操作顺序:  
- 写数据到Buffer
- 调用filp()方法，转换Buffer的模式
- 从Buffer中读数据
- clear()或compact()，清理缓冲区，clear会清空所有数据，compact()会清除已经读过的数据。未读的数据会移动到缓冲区的起始位置，新写的数据跟在未读的数据后面



#### Channel
Channel是一个对象，用来读取和写入数据。
- Channel是双向的，可读可写
- Channel具有异步操作的能力
- Channel读写通过Buffer

Channel对象主要有以下几种:
- FileChannel
- DatagramChannel
- SocketChannel
- ServerSocketChannel 

#### 例子
```java
public static void copyFileUseNIO(String src,String dst) throws IOException{
    //声明源文件和目标文件
            FileInputStream fi=new FileInputStream(new File(src));
            FileOutputStream fo=new FileOutputStream(new File(dst));
            //获得传输通道channel
            FileChannel inChannel=fi.getChannel();
            FileChannel outChannel=fo.getChannel();
            //获得容器buffer
            ByteBuffer buffer=ByteBuffer.allocate(1024);
            while(true){
                //判断是否读完文件
                int eof =inChannel.read(buffer);
                if(eof==-1){
                    break;  
                }
                //重设一下buffer的position=0，limit=position
                buffer.flip();
                //开始写
                outChannel.write(buffer);
                //写完要重置buffer，重设position=0,limit=capacity
                buffer.clear();
            }
            inChannel.close();
            outChannel.close();
            fi.close();
            fo.close();
}     
```
#### Selector      
Selector多路复用器提供了一种单线程管理多个连接的非阻塞io方法。将Channel与Channel事件一起注册到Selector中，使Selector监听这个Channel的这个事件。当事件发生select立即返回，并获得就绪事件的集合，在java中就绪事件使用SelectedKey包装，从key中获得Channel，并判断就绪的事件是什么来进行具体地操作。

Selector底层是调用linux的epoll 和poll方法来实现的，EPollArrayWrapper中封装了epoll的三个方法，有时间再看。

```java
//可以看到SelectionKey里面关联了Channel、Selector、以及监听的事件、就绪的事件等。
public abstract class SelectionKey {
    ... ...
    public abstract int interestOps();

    public abstract SelectionKey interestOps(int ops);

    public abstract int readyOps();
    //监听读操作
    public static final int OP_READ = 1 << 0;
    //监听写操作
    public static final int OP_WRITE = 1 << 2;
    //监听connect操作
    public static final int OP_CONNECT = 1 << 3;
    //监听accept操作
    public static final int OP_ACCEPT = 1 << 4;
    ... ...
    // -- Attachments --

    private volatile Object attachment = null;

    private static final AtomicReferenceFieldUpdater<SelectionKey,Object>
        attachmentUpdater = AtomicReferenceFieldUpdater.newUpdater(
            SelectionKey.class, Object.class, "attachment"
        );

    public final Object attach(Object ob) {
        return attachmentUpdater.getAndSet(this, ob);
    }

    public final Object attachment() {
        return attachment;
    }
}

public class SelectionKeyImpl extends AbstractSelectionKey {
    //SelectionKey关联的Channel
    final SelChImpl channel;
    //SelectionKey关联的Selector
    public final SelectorImpl selector;
    private int index;
    //关联的事件
    private volatile int interestOps;
    //就绪的事件
    private int readyOps;

    SelectionKeyImpl(SelChImpl var1, SelectorImpl var2) {
        this.channel = var1;
        this.selector = var2;
    }

    private void ensureValid() {
        if (!this.isValid()) {
            throw new CancelledKeyException();
        }
    }

    public int interestOps() {
        this.ensureValid();
        return this.interestOps;
    }
    /**
    * 修改SelectionKey关联的事件
    */
    public SelectionKey interestOps(int var1) {
        this.ensureValid();
        return this.nioInterestOps(var1);
    }

    public int readyOps() {
        this.ensureValid();
        return this.readyOps;
    }

    public void nioReadyOps(int var1) {
        this.readyOps = var1;
    }

    public int nioReadyOps() {
        return this.readyOps;
    }

    public SelectionKey nioInterestOps(int var1) {
        if ((var1 & ~this.channel().validOps()) != 0) {
            throw new IllegalArgumentException();
        } else {
            this.channel.translateAndSetInterestOps(var1, this);
            this.interestOps = var1;
            return this;
        }
    }

    public int nioInterestOps() {
        return this.interestOps;
    }
}
```

例：   
```java
public class SelectorDemo {

    public static void main(String[] args) throws IOException {


        Selector selector = Selector.open();
        ServerSocketChannel socketChannel = ServerSocketChannel.open();
        socketChannel.bind(new InetSocketAddress(8080));
        socketChannel.configureBlocking(false);
        socketChannel.register(selector, SelectionKey.OP_ACCEPT);

        while (true) {
            int ready = selector.select();
            if (ready == 0) {
                continue;
            } else if (ready < 0) {
                break;
            }

            Set<SelectionKey> keys = selector.selectedKeys();
            Iterator<SelectionKey> iterator = keys.iterator();
            while (iterator.hasNext()) {

                SelectionKey key = iterator.next();
                if (key.isAcceptable()) {

                    ServerSocketChannel channel = (ServerSocketChannel) key.channel();
                    SocketChannel accept = channel.accept();
                    if (accept == null) {
                        continue;
                    }
                    accept.configureBlocking(false);
                    accept.register(selector, SelectionKey.OP_READ);
                } else if (key.isReadable()) {
                    // 读事件
                    deal((SocketChannel) key.channel(), key);
                } else if (key.isWritable()) {
                    // 写事件
                    resp((SocketChannel) key.channel(), key);
                }
                // 注：处理完成后要从中移除掉
                iterator.remove();
            }
        }
        selector.close();
        socketChannel.close();
    }

    private static void deal(SocketChannel channel, SelectionKey key) throws IOException {

        ByteBuffer buffer = ByteBuffer.allocate(1024);
        ByteBuffer responseBuffer = ByteBuffer.allocate(1024);

        int read = channel.read(buffer);

        if (read > 0) {
            buffer.flip();
            responseBuffer.put(buffer);
        } else if (read == -1) {
            System.out.println("socket close");
            channel.close();
            return;
        }

        key.interestOps(SelectionKey.OP_READ | SelectionKey.OP_WRITE);
        key.attach(responseBuffer);
    }

    private static void resp(SocketChannel channel, SelectionKey key) throws IOException {

        ByteBuffer buffer = (ByteBuffer) key.attachment();
        buffer.flip();

        channel.write(buffer);
        if (!buffer.hasRemaining()) {
            key.attach(null);
            key.interestOps(SelectionKey.OP_READ);
        }
    }
}
```

#### nio的零拷贝

- 要等待dma将数据从硬件设备里面拷贝到

要解释零拷贝，首先了解一下，普通的操作系统io数据拷贝过程:
- 由磁盘/socket缓冲区将数据拷贝到内核空间缓冲区。(等待数据就绪) DMA
- 由内核缓冲区拷贝到应用程序的缓冲区。(读数据) cpu参与复制
- 由应用程序的缓冲区拷贝到内核缓冲区。(写数据) cpu参与复制
- 由内核缓冲区拷贝数据到硬盘/网络socket等io设备。DMA    

可以看到数据流转过程中经过两次读/写，经过了两次内核态与用户态的切换，所以如果数据仅仅的简单的传输，可以使用零拷贝减少不必要的复制过程。

linux2.4以前 通过sendFile系统调用传送文件:
- 通过 DMA copy 将数据从磁盘读取到 kernel buffer 中 
- 通过 CPU copy 将数据从 内核缓冲区 copy 到 sokcet的内核缓冲区 中
- 通过DMA 将socket内核缓冲区的数据拷贝到网卡设备中

可见减少了内核态与用户态切换，减少了一次cpu的复制操作。

linux2.4及以后对 sendFile模式进行了改善:  
- 通过DMA将网卡或者磁盘的数据拷贝到内核缓冲区中
- CPU不再拷贝数据，而是将位于内核缓冲区的数据的位置和偏移量发给socket 的内核缓冲区  (这一步cpu不拷贝数据，只是把数据地址发给socket的内核)
- DMA通过socket内核缓冲区里的数据在内核缓冲区里的内存地址的数据拷贝到网卡磁盘里

这样就实现了零拷贝，CPU不再参与数据的拷贝，内核态与用户态之间的数据拷贝次数为0，两次拷贝数据通过DMA完成。

java.nio中通过Channel的transferTo方法实现,将数据由一个Channel传入到另一个可写的Channel
```java
    public void transferTo(long position,long count,WritableByteChannel target);
```

最终依赖sendFile系统调用
```c++
#include <sys/socket.h>
ssize_t sendfile(int out_fd, int in_fd, off_t *offset, size_t count);
```

例:
```java
public static void fileCopyWithFileChannel(File fromFile, File toFile) {
        FileInputStream fileInputStream = null;
        FileOutputStream fileOutputStream = null;
        FileChannel fileChannelInput = null;
        FileChannel fileChannelOutput = null;
        try {
            fileInputStream = new FileInputStream(fromFile);
            fileOutputStream = new FileOutputStream(toFile);
            //得到fileInputStream的文件通道
            fileChannelInput = fileInputStream.getChannel();
            //得到fileOutputStream的文件通道
            fileChannelOutput = fileOutputStream.getChannel();
            //将fileChannelInput通道的数据，写入到fileChannelOutput通道
            fileChannelInput.transferTo(0, fileChannelInput.size(), fileChannelOutput);
        } catch (IOException e) {
            e.printStackTrace();
        } finally {
            try {
                if (fileInputStream != null) {
                    fileInputStream.close();
                }
                if (fileChannelInput != null) {
                    fileChannelInput.close();
                }
                if (fileOutputStream != null) {
                    fileOutputStream.close();
                }
                if (fileChannelOutput != null) {
                    fileChannelOutput.close();
                }
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }
```

#### nio的直接内存
DirectByteBuffer从内存中分配一段连续的内存，对内存的操作与普通Buffer没有区别，区别在于是使用Unsafe类native方法直接分配的堆外内存。那么堆外内存的释放也由Unsafe类的native方法完成。
```java
//注意DirectByteBuffer继承了MappedByteBuffer内存映射的类，而普通的HeapBuffer没有实现内存映射的类
class DirectByteBuffer extends MappedByteBuffer implements DirectBuffer {
    protected static final Unsafe unsafe = Bits.unsafe();
    private static final long arrayBaseOffset;
    protected static final boolean unaligned;
    private final Object att;
    private final Cleaner cleaner;
    ......
```

##### 那么为什么要使用直接内存呢？
首先了解一下数据的拷贝过程，对于Channel的write方法都依赖于IOUtil的write方法：
```java
static int write(FileDescriptor var0, ByteBuffer var1, long var2, NativeDispatcher var4) throws IOException {
    //如果是DirectBuffer，直接写
    if (var1 instanceof DirectBuffer) {
        return writeFromNativeBuffer(var0, var1, var2, var4);
    } else {
        //非DirectBuffer
        //获取已经读取到的位置
        int var5 = var1.position();
        //获取可以读到的位置
        int var6 = var1.limit();

        assert var5 <= var6;
        //申请一个源buffer可读大小的DirectByteBuffer
        int var7 = var5 <= var6 ? var6 - var5 : 0;
        ByteBuffer var8 = Util.getTemporaryDirectBuffer(var7);

        int var10;
        try {

            var8.put(var1);
            var8.flip();
            var1.position(var5);
            //通过DirectBuffer写
            int var9 = writeFromNativeBuffer(var0, var8, var2, var4);
            if (var9 > 0) {
                var1.position(var5 + var9);
            }

            var10 = var9;
        } finally {
            //回收分配的DirectByteBuffer
            Util.offerFirstTemporaryDirectBuffer(var8);
        }

        return var10;
    }
}
```
可以看到，当Buffer是DirectBuffer时会直接调用writeFromNativeBuffer方法，当Buffer是HeapBuffer时会先申请一个直接内存，然后将堆内存拷贝到直接内存，然后在进行write操作系统调用。

##### 可以知道，nio在使用堆内存进行io操作之前会先拷贝到堆外内存，是为什么呢？
因为指向native方法的线程被认为是处于safepoint，可能会发生GC，导致堆内存重排。

传统的BIO的write方法数据使用字节数组传递，而NIO的write方法为了提高效率，数据仅仅通过内存地址传递，所以如果在native方法时发生了GC就会导致HeapBuffer的内存地址变化，因此要使用DirectBuffer来暂存保证内存地址不会被修改。

所以HeapBuffer数据拷贝的顺序应该是
> 网络 –> 临时的DirectByteBuffer –> 应用 HeapByteBuffer –> 临时的DirectByteBuffer –> 网络

如果直接使用DirectByteBuffer的数据拷贝顺序是
> 网络 –> 应用 DirectByteBuffer –> 网络

##### 申请堆外内存意味着java需要对这一块内存进行管理
DirectByteBuffer中使用Bits对象申请堆外内存，其中有一个变量totalCapacity记录已经分配的堆外内存的总大小，然后jvm通过-XX:MaxDirectMemorySize参数设置堆外内存的大小限制。

在DirectByteBuffer的构造方法里会预分配堆外内存，如果分配空间不足就会调用System.gc()方法来触发full gc期待能回收一点堆外内存，如果gc以后还是不足会抛OOM(因此如果堆外内存不够的话也是会抛OOM的),如果内存足够了就会使用Unsafe类去分配内存，返回内存基址。

##### DirectByteBuffer的回收
DirectByteBuffer是一个冰山对象，存在于堆里的对象DirectByteBuffer很小，包括内存基地址、大小等属性以及Cleaner对象用来做内存回收，但其指向了一大堆内存。

DirectByteBuffer对象的Cleaner对象实现了PhantomReference接口
```java
public class Cleaner extends PhantomReference<Object> {
    private static final ReferenceQueue<Object> dummyQueue = new ReferenceQueue();
    private static Cleaner first = null;
    private Cleaner next = null;
    private Cleaner prev = null;
    private final Runnable thunk;

    private Cleaner(Object var1, Runnable var2) {
        super(var1, dummyQueue);
        this.thunk = var2;
    }
    ......
    public void clean() {
        if (remove(this)) {
            try {
                this.thunk.run();
            } catch (final Throwable var2) {
                AccessController.doPrivileged(new PrivilegedAction<Void>() {
                    public Void run() {
                        if (System.err != null) {
                            (new Error("Cleaner terminated abnormally", var2)).printStackTrace();
                        }

                        System.exit(1);
                        return null;
                    }
                });
            }

        }
    }
```
Cleaner实现了虚引用接口，其构造方法将DirectByteBuffer对象与虚引用队列传递给了父类，也就是给DirectByteBuffer对象加上了虚引用。当DirectByteBuffer对象被回收以后，就会将cleaner对象放到虚引用队列里面，然后使用一个ReferenceHandle线程处理这个队列，如果从队列里面取出了Cleaner对象就执行它的clean方法，执行trunk.run方法，这个trunk其实是Deallocator类实现了Runnable接口的嵌套类，其run方法会调用unsafe.freeMemory(long address)方法来释放堆外内存。

当然还有个问题就是，如果DirectByteBuffer对象经过几次Y gc进入了老年代以后很难回收掉，就会长期占用堆外内存。因此使用堆外内存要求手动回收。


#### nio的内存映射
linux中使用mmap内存映射，将文件映射到应用程序的缓冲区，那么文件位于哪儿？文件描述符经由我们open系统调用后获得，自然是位于内核缓冲区，将位于内核空间的文件映射到应用程序缓冲区，减少了由内核空间到用户空间的数据拷贝，以及用户态与内核态的切换。

- 发出mmap系统调用，导致用户空间到内核空间的上下文切换(第一次上下文切换)。通过DMA引擎将磁盘文件中的内容拷贝到内核空间缓冲区中(第一次拷贝: hard drive ——> kernel buffer)。
- mmap系统调用返回，导致内核空间到用户空间的上下文切换(第二次上下文切换)。接着用户空间和内核空间共享这个缓冲区，而不需要将数据从内核空间拷贝到用户空间。因为用户空间和内核空间共享了这个缓冲区数据，所以用户空间就可以像在操作自己缓冲区中数据一般操作这个由内核空间共享的缓冲区数据。
- 发出write系统调用，导致用户空间到内核空间的上下文切换(第三次上下文切换)。将数据从内核空间缓冲区拷贝到内核空间socket相关联的缓冲区(第二次拷贝: kernel buffer ——> socket buffer)。
- write系统调用返回，导致内核空间到用户空间的上下文切换(第四次上下文切换)。通过DMA引擎将内核空间socket缓冲区中的数据传递到协议引擎(第三次拷贝: socket buffer ——> protocol engine)

在java中使用FileChannel.map方法来将文件映射到MappedByteBuffer中，有三种模式:
- FileChannel.MapMode.READ_ONLY:产生只读缓冲区，对缓冲区的写入操作将导致ReadOnlyBufferException
- FileChannel.MapMode.READ_WRITE:产生可写缓冲区，任何修改将在某个时刻由内核缓冲区写回到磁盘文件中，而这某个时刻是依赖OS的，其他映射同一个文件的程序可能不能立即看到这些修改，多个程序同时进行文件映射的确切行为是依赖于系统的，但是它是线程安全的
- FileChannel.MapMode.PRIVATE:产生可写缓冲区，但任何修改是缓冲区私有的，不会回到文件中

这种方式处理大文件很方便，避免了文件的拷贝，而且由于零拷贝无法修改数据，因此有数据修改需求的时候可以使用内存映射缓冲区来做，修改完的数据就是映射到的内核空间的数据。
