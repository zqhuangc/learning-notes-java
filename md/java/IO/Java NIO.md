InetSocketAddress

```java
public NIOServer(int port) throws IOException{

        this.port = port;

        //先把通道打开
        ServerSocketChannel server = ServerSocketChannel.open();

        //设置高速公路的关卡
        server.bind(new InetSocketAddress(this.port));
        // 默认为 true 阻塞
        // 设为 false 非阻塞
        server.configureBlocking(false);

        //开门迎客，排队叫号大厅开始工作
        selector = Selector.open();

        //告诉服务叫号大厅的工作人员，你可以接待了（事件）
        server.register(selector, SelectionKey.OP_ACCEPT);

        System.out.println("服务已启动，监听端口是：" + this.port);
    }
```



## Java NIO架构

核心主要有三个，分别是：Selector、Channel和Buffer,我们先来看看它们之间的关系：

它们之间的关系很清晰，一个线程对应着一个Selector，一个Selector对应着多个Channel，一个Channel对应着一个Buffer，当然这只是通常的做法，一个Channel也可以对应多个Selector，一个Channel对应着多个Buffer。

### Selector

```
java.nio.channels.spi.SelectorProvider#openSelector
SelectorProvider#provider()
```

Selector——选择器用于监听多个管道的事件，使用传统的阻塞IO时我们可以方便的知道什么时候可以进行读写，而使用非阻塞通道，我们需要一些方法来知道什么时候通道准备好了，选择器正是为这个需要而诞生的。
Selector 一般称 为选择器 ，当然你也可以翻译为 多路复用器 。它是Java NIO核心组件中的一个，用于检查一个或多个NIO Channel（通道）的状态是否处于可读、可写。如此可以实现单线程管理多个channels,也就是可以管理多个网络链接。
使用Selector的好处在于： 使用更少的线程来就可以来处理通道了， 相比使用多个线程，避免了线程上下文切换带来的开销。

传统的Java IO在面对大量IO请求的时候有心无力，因为每个维护每一个IO请求都需要一个线程，这带来的问题就是，系统资源被极度消耗，吞吐量直线下降，引起系统相关问题，那么Java NIO是如何解决这个问题的呢？答案就是Selector，简单来说它对应着多路IO复用中的监管角色，它负责统一管理IO请求，监听相应的IO事件，并通知对应的线程进行处理，这种模式下就无需为每个IO请求单独分配一个线程，另外也减少线程大量阻塞，资源利用率下降的情况

### Channel
Channel——管道实际上就像传统IO中的流，到任何目的地(或来自任何地方)的所有数据都必须通过一个 Channel 对象。一个 Buffer 实质上是一个容器对象。
每一种基本 Java 类型都有一种缓冲区类型：

Channel作为一种全新的设计（双向的），它帮助系统以相对小的代价来保持IO请求数据传输的处理，但是它并不真正存放数据，它总是结合着缓存区（Buffer）一起使用，另外Channel主要有以下四种：
FileChannel：读写文件时使用的通道
DatagramChannel：传输UDP连接数据时的通道,与Java IO中的DatagramSocket对应
SocketChannel：传输TCP连接数据时的通道，与Java IO中的Socket对应
ServerSocketChannel: 监听套接词连接时的通道，与Java IO中的ServerSocket对应
当然其中最重要以及最常用的就是SocketChannel和ServerSocketChannel，也是Java NIO的精髓，ServerSocketChannel可以设置成非阻塞模式，然后结合Selector就可以实现多路复用IO，使用一个线程管理多个Socket连接

通道和流一样都是需要基于物理文件的，而每个流或者通道都通过文件指针操作文件，这里说的「通道是双向的」也是有前提的，那就是通道基于随机访问『RandomAccessFile』的可读可写文件指针。『RandomAccessFile』是既可读又可写的，所以基于它的通道是双向的，所以，「通道是双向的」这句话是有前提的，不能断章取义。

```java
//创建一个 TCP 套接字通道
SocketChannel channel = SocketChannel.open();
//调整通道为非阻塞模式
channel.configureBlocking(false);
//向选择器注册一个通道
SelectionKey key = channel.register(selector, SelectionKey.OP_READ);

```

##### Package java.nio.channels

* Channel

Channel表现了一个可以进行IO操作的通道，该interface定义了以下方法：

boolean isOpen()
​       该Channel是否是打开的。

void close()
​       关闭这个Channel，相关的资源会被释放。

* ReadableByteChannel

定义了一个可从中读取byte数据的channel interface。

int read(ByteBuffer dst)
从channel中读取byte数据并写到ByteBuffer中。返回读取的byte数。

* WritableByteChannel

定义了一个可向其写byte数据的channel interface。

int write(ByteBuffer src)
​    从ByteBuffer中读取byte数据并写到channel中。返回写出的byte数。

* ByteChannel

ByteChannel并没有定义新的方法，它的作用只是把ReadableByteChannel和WritableByteChannel合并在一起。

* ScatteringByteChannel

继承了ReadableByteChannel并提供了同时往几个ByteBuffer中写数据的能力。

* GatheringByteChannel

继承了WritableByteChannel并提供了同时从几个ByteBuffer中读数据的能力。

* InterruptibleChannel

用来表现一个可以被异步关闭的Channel。这表现在两方面：

1． 当一个InterruptibleChannel的close()方法被调用时，其它block在这个InterruptibleChannel的IO操作上的线程会接收到一个AsynchronousCloseException。

2． 当一个线程block在InterruptibleChannel的IO操作上时，另一个线程调用该线程的interrupt()方法会导致channel被关闭，该线程收到一个ClosedByInterruptException，同时线程的interrupt状态会被设置。

 

非阻塞IO
非阻塞IO的支持可以算是NIO API中最重要的功能，非阻塞IO允许应用程序同时监控多个channel以提高性能，这一功能是通过Selector，SelectableChannel和SelectionKey这3个类来实现的。

SelectableChannel 代表了可以支持非阻塞IO操作的channel，可以将其注册在 Selector 上，这种注册的关系由 SelectionKey 这个类来表现（见UML图）。Selector这个类通过select()函数，给应用程序提供了一个可以同时监控多个IO channel的方法：

 

应用程序通过调用select()函数，让Selector监控注册在其上的多个SelectableChannel，当有channel的IO操作可以进行时，select()方法就会返回以让应用程序检查channel的状态，并作相应的处理。

* SelectableChannel

这个抽象类是所有支持非阻塞IO操作的channel（如DatagramChannel、SocketChannel）的父类。SelectableChannel可以注册到一个或多个Selector上以进行非阻塞IO操作。

SelectableChannel可以是blocking和non-blocking模式（所有channel创建的时候都是blocking模式），只有non-blocking的SelectableChannel才可以参与非阻塞IO操作。

SelectableChannel configureBlocking(boolean block)
​       设置blocking模式。

boolean isBlocking()
​       返回blocking模式。

通过register()方法，SelectableChannel可以注册到Selector上。

int validOps()
返回一个bit mask，表示这个channel上支持的IO操作。当前在SelectionKey中，用静态常量定义了4种IO操作的bit值：OP_ACCEPT，OP_CONNECT，OP_READ和OP_WRITE。

SelectionKey register(Selector sel, int ops)
将当前channel注册到一个Selector上并返回对应的SelectionKey。在这以后，通过调用Selector的select()函数就可以监控这个channel。ops这个参数是一个bit mask，代表了需要监控的IO操作。

SelectionKey register(Selector sel, int ops, Object att)
这个函数和上一个的意义一样，多出来的att参数会作为attachment被存放在返回的SelectionKey中，这在需要存放一些session state的时候非常有用。

boolean isRegistered()
​       该channel是否已注册在一个或多个Selector上。


SelectableChannel还提供了得到对应SelectionKey的方法

SelectionKey keyFor(Selector sel)
返回该channe在Selector上的注册关系所对应的SelectionKey。若无注册关系，返回null。


Selector
Selector可以同时监控多个SelectableChannel的IO状况，是非阻塞IO的核心。

Selector open()
​       Selector的一个静态方法，用于创建实例。

在一个Selector中，有3个SelectionKey的集合：
1． key set代表了所有注册在这个Selector上的channel，这个集合可以通过keys()方法拿到。
2． Selected-key set代表了所有通过select()方法监测到可以进行IO操作的channel，这个集合可以通过selectedKeys()拿到。
3． Cancelled-key set代表了已经cancel了注册关系的channel，在下一个select()操作中，这些channel对应的SelectionKey会从key set和cancelled-key set中移走。这个集合无法直接访问。

 

以下是select()相关方法的说明：

int select()
监控所有注册的channel，当其中有注册的IO操作可以进行时，该函数返回，并将对应的SelectionKey加入selected-key set。

int select(long timeout)
​       可以设置超时的select()操作。

int selectNow()
​       进行一个立即返回的select()操作。

Selector wakeup()
​       使一个还未返回的select()操作立刻返回。

SelectionKey
代表了Selector和SelectableChannel的注册关系。

Selector定义了4个静态常量来表示4种IO操作，这些常量可以进行位操作组合成一个bit mask。

int OP_ACCEPT
有新的网络连接可以accept，ServerSocketChannel支持这一非阻塞IO。

int OP_CONNECT
​       代表连接已经建立（或出错），SocketChannel支持这一非阻塞IO。

int OP_READ
int OP_WRITE
​       代表了读、写操作。


以下是其主要方法：

Object attachment()
返回SelectionKey的attachment，attachment可以在注册channel的时候指定。

Object attach(Object ob)
​       设置SelectionKey的attachment。

SelectableChannel channel()
​       返回该SelectionKey对应的channel。

Selector selector()
​       返回该SelectionKey对应的Selector。

void cancel()
​       cancel这个SelectionKey所对应的注册关系。

int interestOps()
​       返回代表需要Selector监控的IO操作的bit mask。

SelectionKey interestOps(int ops)
​       设置interestOps。

int readyOps()
​       返回一个bit mask，代表在相应channel上可以进行的IO操作。


ServerSocketChannel
支持非阻塞操作，对应于java.net.ServerSocket这个类，提供了TCP协议IO接口，支持OP_ACCEPT操作。

ServerSocket socket()
​       返回对应的ServerSocket对象。

SocketChannel accept()
​       接受一个连接，返回代表这个连接的SocketChannel对象。

 

SocketChannel
支持非阻塞操作，对应于java.net.Socket这个类，提供了TCP协议IO接口，支持OP_CONNECT，OP_READ和OP_WRITE操作。这个类还实现了ByteChannel，ScatteringByteChannel和GatheringByteChannel接口。

DatagramChannel和这个类比较相似，其对应于java.net.DatagramSocket，提供了UDP协议IO接口。


Socket socket()
​       返回对应的Socket对象。

boolean connect(SocketAddress remote)
boolean finishConnect()
connect()进行一个连接操作。如果当前SocketChannel是blocking模式，这个函数会等到连接操作完成或错误发生才返回。如果当前SocketChannel是non-blocking模式，函数在连接能立刻被建立时返回true，否则函数返回false，应用程序需要在以后用finishConnect()方法来完成连接操作。

Pipe
包含了一个读和一个写的channel(Pipe.SourceChannel和Pipe.SinkChannel)，这对channel可以用于进程中的通讯。

FileChannel
用于对文件的读、写、映射、锁定等操作。和映射操作相关的类有FileChannel.MapMode，和锁定操作相关的类有FileLock。值得注意的是FileChannel并不支持非阻塞操作。

 

Channels
这个类提供了一系列static方法来支持stream类和channel类之间的互操作。这些方法可以将channel类包装为stream类，比如，将ReadableByteChannel包装为InputStream或Reader；也可以将stream类包装为channel类，比如，将OutputStream包装为WritableByteChannel。

### Buffer

定义了一个可以线性存放primitive type数据的容器接口。Buffer主要包含了与类型（byte, char…）无关的功能。值得注意的是Buffer及其子类都不是线程安全的。

Buffer对应着某一个Channel，从Channel中读取数据或者向Channel中写数据，Buffer与数组很类似（子类内部维护了一个相应类型数组），但是它提供了更多的特性，方便我们对Buffer中的数据进行操作，我们先来看一下Buffer分配时的类别（这里不是指Buffer的具体数据类型）即Direct Buffer和Heap Buffer，那么为什么要有这两种类别的Buffer呢？我们先来看看它们的特性：
* Direct Buffer：  unsafe
直接分配在系统内存中；  
不需要花费将数据库从内存拷贝到Java内存中的成本；  
虽然Direct Buffer是直接分配中系统内存中的，但当它被重复利用时，只有真正需要数据的那一页数据会被装载到真是的内存中，其它的还存在在虚拟内存中，不会造成实际内存的资源浪费；  
可以结合特定的机器码，一次可以有顺序的读取多字节单元；  
因为直接分配在系统内存中，所以它不受Java GC管理，不会自动回收；  
创建以及销毁的成本比较高；  
* Heap Buffer：  
分配在Java Heap，受Java GC管理生命周期，不需要额外维护；  
创建成本相对较低；  
根据它们的特性，我们可以大致总结出它们的适用场景：  
如果这个Buffer可以重复利用，而且你也想多个字节操作，亦或者你对性能要求很高，可以选择使用Direct Buffer，但其编码相对来说会比较复杂，需要注意的点也更多，反之则用Heap Buffer，

  - Capacity：顾名思义它的含义是容量，代表着Buffer的最大容量，与数组的Size很类似，初始化不可更改，除非你改变的Buffer的结构；
  - Limit：顾名思义它的含义是界限，代表着Buffer的目前可使用的最大限制，写模式下，一般Limit等于Capacity，读模式下需要你自己控制它的值结合position读取想要的数据；
  - `Position`：顾名思义它的含义是位置，代表着Buffer目前操作的位置，通俗来说，就是你下次对Buffer进行操作的起始位置；
mark 用于记录当前position的前一个位置或者默认是-1，用于重复读。private int mark = -1;
address 用于操作直接内存，区别于 jvm 内存。long address;
？？？？？？？？？
写模式下，所谓写模式就是将缓存区中的内容写入通道。position 代表下一个字节应该被写出去的字节在缓存区中的位置，limit 表示最后一个待写字节在缓存区的位置。
读模式下，所谓读模式就是从通道读取数据到缓存区。position 代表下一个读出来的字节应当存储在缓存区的位置，limit 等于 capacity。
？？？？？？？

#### 属性：

capacity
这个Buffer最多能放多少数据。capacity一般在buffer被创建的时候指定。

limit
在Buffer上进行的读写操作都不能越过这个下标。当写数据到buffer中时，limit一般和capacity相等，当读数据时，limit代表buffer中有效数据的长度。

position
读/写操作的当前下标。当使用buffer的相对位置进行读/写操作时，读/写会从这个下标进行，并在操作完成后，buffer会更新下标的值。

mark
一个临时存放的位置下标。调用mark()会将mark设为当前的position的值，以后调用reset()会将position属性设置为mark的值。mark的值总是小于等于position的值，如果将position的值设的比mark小，当前的mark值会被抛弃掉

这些属性总是满足以下条件：
0 <= mark <= position <= limit <= capacity

limit和position的值除了通过limit()和position()函数来设置，也可以通过下面这些函数来改变：

Buffer clear()
把position设为0，把limit设为capacity，一般在把数据写入Buffer前调用。

Buffer flip()
把limit设为当前position，把position设为0，一般在从Buffer读出数据前调用。

Buffer rewind()
把position设为0，limit不变，一般在把数据重写入Buffer前调用。

Buffer对象有可能是只读的，这时，任何对该对象的写操作都会触发一个ReadOnlyBufferException。isReadOnly()方法可以用来判断一个Buffer是否只读。

#### ByteBuffer

在Buffer的子类中，ByteBuffer是一个地位较为特殊的类，因为在java.io.channels中定义的各种channel的IO操作基本上都是围绕ByteBuffer展开的。


ByteBuffer定义了4个static方法来做创建工作：

ByteBuffer allocate(int capacity)
创建一个指定capacity的ByteBuffer。

ByteBuffer allocateDirect(int capacity)
创建一个direct的ByteBuffer，这样的ByteBuffer在参与IO操作时性能会更好（很有可能是在底层的实现使用了DMA技术），相应的，创建和回收direct的ByteBuffer的代价也会高一些。isDirect()方法可以检查一个buffer是否是direct的。

ByteBuffer wrap(byte [] array)
ByteBuffer wrap(byte [] array, int offset, int length)
把一个byte数组或byte数组的一部分包装成ByteBuffer。


ByteBuffer定义了一系列get和put操作来从中读写byte数据，如下面几个：

byte get()
ByteBuffer get(byte [] dst)
byte get(int index)


ByteBuffer put(byte b)
ByteBuffer put(byte [] src)
ByteBuffer put(int index, byte b)

这些操作可分为绝对定位和相对定位两种，相对定位的读写操作依靠position来定位Buffer中的位置，并在操作完成后会更新position的值。

在其它类型的buffer中，也定义了相同的函数来读写数据，唯一不同的就是一些参数和返回值的类型。


除了读写byte类型数据的函数，ByteBuffer的一个特别之处是它还定义了读写其它primitive数据的方法，如：

int getInt()
​    从ByteBuffer中读出一个int值。

ByteBuffer putInt(int value)
​    写入一个int值到ByteBuffer中。


读写其它类型的数据牵涉到字节序问题，ByteBuffer会按其字节序（大字节序或小字节序）写入或读出一个其它类型的数据（int,long…）。字节序可以用order方法来取得和设置：

ByteOrder order()
​       返回ByteBuffer的字节序。

ByteBuffer order(ByteOrder bo)
​       设置ByteBuffer的字节序。


ByteBuffer另一个特别的地方是可以在它的基础上得到其它类型的buffer。如：

CharBuffer asCharBuffer()
为当前的ByteBuffer创建一个CharBuffer的视图。在该视图buffer中的读写操作会按照ByteBuffer的字节序作用到ByteBuffer中的数据上。

用这类方法创建出来的buffer会从ByteBuffer的position位置开始到limit位置结束，可以看作是这段数据的视图。视图buffer的readOnly属性和direct属性与ByteBuffer的一致，而且也只有通过这种方法，才可以得到其他数据类型的direct buffer。


ByteOrder
用来表示ByteBuffer字节序的类，可将其看成java中的enum类型。主要定义了下面几个static方法和属性：


ByteOrder BIG_ENDIAN
​       代表大字节序的ByteOrder。

ByteOrder LITTLE_ENDIAN
​       代表小字节序的ByteOrder。

ByteOrder nativeOrder()
​       返回当前硬件平台的字节序。

MappedByteBuffer
ByteBuffer的子类，是文件内容在内存中的映射。这个类的实例需要通过FileChannel的map()方法来创建。

## NIO和传统的IO有什么区别呢?

1. IO是面向流的，NIO是面向块（缓冲区）的。  

IO面向流的操作一次一个字节地处理数据。一个输入流产生一个字节的数据，一个输出流消费一个字节的数据。，导致了数据的读取和写入效率不佳；
NIO面向块的操作在一步中产生或者消费一个数据块。按块处理数据比按(流式的)字节处理数据要快得多，同时数据读取到一个它稍后处理的缓冲区，需要时可在缓冲区中前后移动。这就增加了处理过程中的灵活性。通俗来说，NIO采取了“预读”的方式，当你读取某一部分数据时，他就会猜测你下一步可能会读取的数据而预先缓冲下来。

2. IO是阻塞的，NIO是非阻塞的。  

对于传统的IO，当一个线程调用read() 或 write()时，该线程被阻塞，直到有一些数据被读取，或数据完全写入。该线程在此期间不能再干任何事情了。
而对于NIO，使用一个线程发送读取数据请求，没有得到响应之前，线程是空闲的，此时线程可以去执行别的任务，而不是像IO中那样只能等待响应完成。

## NIO和IO适用场景
NIO是为弥补传统IO的不足而诞生的，但是尺有所短寸有所长，NIO也有缺点，因为NIO是面向缓冲区的操作，每一次的数据处理都是对缓冲区进行的，那么就会有一个问题，在数据处理之前必须要判断缓冲区的数据是否完整或者已经读取完毕，如果没有，假设数据只读取了一部分，那么对不完整的数据处理没有任何意义。所以每次数据处理之前都要检测缓冲区数据。
* 那么NIO和IO各适用的场景是什么呢？  

如果需要管理同时打开的成千上万个连接，这些连接每次只是发送少量的数据，例如聊天服务器，这时候用NIO处理数据可能是个很好的选择。
而如果只有少量的连接，而这些连接每次要发送大量的数据，这时候传统的IO更合适。使用哪种处理数据，需要在数据的响应等待时间和检查缓冲区数据的时间上作比较来权衡选择。

### RandomAccessFile



RandomAccessFile的绝大多数功能，但不是全部，已经被JDK 1.4的nio的"内存映射文件(memory-mapped files)"给取代了，你该考虑一下是不是用"内存映射文件"来代替RandomAccessFile了。
一、作用：  
随机流（RandomAccessFile）不属于IO流，支持对文件的读取和写入随机访问。

二、随机访问文件原理：  

首先把随机访问的文件对象看作存储在文件系统中的一个大型 byte 数组，然后通过指向该 byte 数组的光标或索引（即：文件指针 FilePointer）在该数组任意位置读取或写入任意数据。

三、相关方法说明： 

1、对象声明：RandomAccessFile raf = newRandomAccessFile(File file, String mode);
​    其中参数 mode 的值可选 "r"：可读，"w" ：可写，"rw"：可读性；  
2、获取当前文件指针位置：int RandowAccessFile.getFilePointer();  
3、改变文件指针位置（相对位置、绝对位置）：    
1> 绝对位置：RandowAccessFile.seek(int index);  
2> 相对位置：RandowAccessFile.skipByte(int step);  相对当前位置  
4、给写入文件预留空间：RandowAccessFile.setLength(long len);

### 内存映射：

内存映射文件能让你创建和修改那些因为太大而无法放入内存的文件。有了内存映射文件，你就可以认为文件已经全部读进了内存，然后把它当成一个非常大的数组来访问。这种解决办法能大大简化修改文件的代码。
fileChannel.map(FileChannel.MapMode mode, long position, long size)将此通道的文件区域直接映射到内存中。注意，你必须指明，它是从文件的哪个位置开始映射的，映射的范围又有多大；也就是说，它还可以映射一个大文件的某个小片断。

MappedByteBuffer是ByteBuffer的子类，因此它具备了ByteBuffer的所有方法，但新添了force()将缓冲区的内容强制刷新到存储设备中去、load()将存储设备中的数据加载到内存中、isLoaded()位置内存中的数据是否与存储设置上同步。

尽管映射写似乎要用到 FileOutputStream，但是映射文件中的所有输出 必须使用 RandomAccessFile，但如果只需要读时可以使用FileInputStream，写映射文件时一定要使用随机访问文件，可能写时要读的原因吧。



这个功能主要是为了提高大文件的读写速度而设计的。内存映射文件(memory-mappedfile)能让你创建和修改那些大到无法读入内存的文件。有了内存映射文件，你就可以认为文件已经全部读进了内存，然后把它当成一个非常大的数组来访问了。将文件的一段区域映射到内存中，比传统的文件处理速度要快很多。内存映射文件它虽然最终也是要从磁盘读取数据，但是它并不需要将数据读取到OS内核缓冲区，而是直接将进程的用户私有地址空间中的一部分区域与文件对象建立起映射关系，就好像直接从内存中读、写文件一样，速度当然快了。



Java NIO AsynchronousFileChannel异步文件通
Java7中新增了AsynchronousFileChannel作为nio的一部分。AsynchronousFileChannel使得数据可以进行异步读写。



NIO通讯服务端步骤：

1、创建ServerSocketChannel,为它配置非阻塞模式

2、绑定监听，配置TCP参数，录入backlog大小等

3、创建一个独立的IO线程，用于轮询多路复用器Selector

4、创建Selector，将之前的ServerSocketChannel注册到Selector上，并设置监听标识位SelectionKey.ACCEPT

5、启动IO线程，在循环体中执行Selector.select()方法，轮询就绪的通道

6、当轮询到处于就绪的通道时，需要进行判断操作位，如果是ACCEPT状态，说明是新的客户端介入，则调用accept方法接受新的客户端。

7、设置新接入客户端的一些参数，并将其通道继续注册到Selector之中。设置监听标识等

8、如果轮询的通道操作位是READ，则进行读取，构造Buffer对象等

9、更细节的还有数据没发送完成继续发送的问题



### 小结

Selectorkey，OP_write/read/accepted/connected

FileChannel，serversocketchannel，socketchannel

bytebuffer.put()/get(), 0<=position<=limit<=capacity

wrap，slice,flip,clear

allocate

allocatedirect

asreadonlybuffer

mappedbytebuffer



GatheringByteChannel

IOUtil

# 源码分析

## 创建

Selector#open()

SelectorProvider#provider#openSelector

ServerSocketChannel#open()

SelectorProvider#provider#openServerSocketChannel



pipe.open

创建一个pipe对象，目的是为了实现Selector的唤醒操作。看下这个Pipe类，它很老，在1.4就有了，可见它是伴随着JDK NIO一起出现的，它是一个单向的管道数据结构，关联了一对儿Channel，一个是可写Channel，JDK叫它sinkChannel（sink），一个是可读Channel，JDK叫它sourceChannel（source）。

pipe的设计主要是为了实现Selector的唤醒机制

```java
WindowsSelectorImpl(SelectorProvider sp) throws IOException {
        super(sp);
        pollWrapper = new PollArrayWrapper(INIT_CAP);
        wakeupPipe = Pipe.open();
        wakeupSourceFd = ((SelChImpl)wakeupPipe.source()).getFDVal();

        // Disable the Nagle algorithm so that the wakeup is more immediate
        SinkChannelImpl sink = (SinkChannelImpl)wakeupPipe.sink();
        (sink.sc).socket().setTcpNoDelay(true);
        wakeupSinkFd = ((SelChImpl)sink).getFDVal();

        pollWrapper.addWakeupSocket(wakeupSourceFd, 0);
    }

ServerSocketChannelImpl(SelectorProvider sp) throws IOException {
        super(sp);
        this.fd =  Net.serverSocket(true);
        this.fdVal = IOUtil.fdVal(fd);
    }

```



## 注册事件

> channel#register(selector,SelectionKey.OP_xxx)  -->
>
> SelectorImpl#register
>
> WindowsSelectorImpl#implRegister(SelectionKeyImpl)
>
> newKeys.addLast(ski);
>
>
>
> SelectionKeyImpl#interestOps
>
> WindowsSelectorImp#setEventOps(this);
>
> updateKeys.addLast(ski);



```java
// AbstractSelectableChannel#
public final SelectionKey register(Selector sel, int ops, Object att)
        throws ClosedChannelException
    {
        if ((ops & ~validOps()) != 0)
            throw new IllegalArgumentException();
        if (!isOpen())
            throw new ClosedChannelException();
        synchronized (regLock) {
            if (isBlocking())
                throw new IllegalBlockingModeException();
            synchronized (keyLock) {
                // re-check if channel has been closed
                if (!isOpen())
                    throw new ClosedChannelException();
                SelectionKey k = findKey(sel);
                if (k != null) {
                    k.attach(att);
                    k.interestOps(ops);
                } else {
                    // New registration
                    k = ((AbstractSelector)sel).register(this, ops, att);
                    addKey(k);
                }
                return k;
            }
        }
    }
    
// ServerSocketChannel#validOps()
public final int validOps() {
        return SelectionKey.OP_ACCEPT;
    }

监听轮询前面创建的那个poll数组——pollWrapper，里面保存的已经注册到当前Selector的Channel的fd，该过程可能会阻塞。
```



> PollArrayWrapper
>
> FdMap
>
> WindowsSelectorImp
>
> ServerSocketChannelImpl中kill

```java
subSelector.processSelectedKeys(updateCount, action)

private int processSelectedKeys(long updateCount, Consumer<SelectionKey> action) {
          int numKeysUpdated = 0;
          numKeysUpdated += processFDSet(updateCount, action, readFds,
                                           Net.POLLIN,
                                           false);
            numKeysUpdated += processFDSet(updateCount, action, writeFds,
                                           Net.POLLCONN |
                                           Net.POLLOUT,
                                           false);
            numKeysUpdated += processFDSet(updateCount, action, exceptFds,
                                           Net.POLLIN |
                                           Net.POLLCONN |
                                           Net.POLLOUT,
                                           true);
            return numKeysUpdated;
        }
private void setWakeupSocket() {
  this.setWakeupSocket0(this.wakeupSinkFd);
}
private native void setWakeupSocket0(int var1);
Java_sun_nio_ch_WindowsSelectorImpl_setWakeupSocket0(JNIEnv *env, jclass this,
                                                jint scoutFd) {
    /* Write one byte into the pipe */
    const char byte = 1;
    send(scoutFd, &byte, 1, 0);
}

adjustThreadsCount调整轮询线程数量


```



## 轮询

Selector#select()

doSelect中begin完毕后，调用subSelector的poll方法轮询；若是poll上有事件就绪，那么就不会阻塞，继续往下进行；若poll上没有事件就绪就会等待SelectThread上的事件就绪，通过threadFinished将其唤醒；若是SelectThread上也没有事件就绪就会一直阻塞，除非被外部唤醒，或者调用的是select的单参方法，会阻塞到超时结束。



epll模型（Linux内核）



```java
OP_ACCEPT，它本质就是read事件，即当有新客户端连接请求到该服务器，对应的服务端Channel的fd里会有数据进来，此时poll0方法会被触发，也就是所谓的被唤醒，它会轮询数组里所有的fd，这也是select的缺点之一，轮询一遍才能找到这个就绪的fd，然后解除阻塞，立即返回。对应到上层就是Selector的select方法会立即返回。

线程调用了该Selector的wakeup方法，那么也会唤醒select方法
poll0

@Override
    protected int doSelect(Consumer<SelectionKey> action, long timeout)
        throws IOException
    {
        assert Thread.holdsLock(this);
        this.timeout = timeout; // set selector timeout
        processUpdateQueue();
        processDeregisterQueue();
        if (interruptTriggered) {
            resetWakeupSocket();
            return 0;
        }
        // Calculate number of helper threads needed for poll. If necessary
        // threads are created here and start waiting on startLock
        adjustThreadsCount(); // 1
        finishLock.reset(); // reset finishLock
        // Wakeup helper threads, waiting on startLock, so they start polling.
        // Redundant threads will exit here after wakeup.
        startLock.startThreads(); //2
        // do polling in the main thread. Main thread is responsible for
        // first MAX_SELECTABLE_FDS entries in pollArray.
        try {
            begin();
            try {
                subSelector.poll();// 
            } catch (IOException e) {
                finishLock.setException(e); // Save this exception
            }
            // Main thread is out of poll(). Wakeup others and wait for them
            if (threads.size() > 0)
                finishLock.waitForHelperThreads();
          } finally {
              end();
          }
        // Done with poll(). Set wakeupSocket to nonsignaled  for the next run.
        finishLock.checkForException();
        processDeregisterQueue();
        int updated = updateSelectedKeys(action);
        // Done with poll(). Set wakeupSocket to nonsignaled  for the next run.
        resetWakeupSocket();
        return updated;
    }
    

EPollSelectorImpl.java
注册说的比较抽象，其实就是将某个Channel和一个Selector关联起来，结合epoll机制，就是让I/O多路复用器能和某个Socket的fd绑定，只不过这里可以绑定多个Socket的fd
```

文件描述符（fd） 遍历检测

ServerSocketChannelImpl 操作系统的监听套接字fd

第141行注释清晰的写明：阻塞模式的fd，也就是Channel，无法使用register实现事件驱动

LT工作方式

对于I/O请求知道一个流程，以输入为例，一个输入操作通常包括两个阶段：

1、俗称的等待数据准备好：即网卡有数据且数据已经被复制到操作系统内核，也有人把这一步叫等待数据就绪

2、从操作系统内核向进程用户空间复制这份数据，也有人把这一步叫从内核拷贝数据到用户空间



Linux的read、write、recv和recvfrom等API都是阻塞的I/O系统调用，所谓阻塞就是当函数不能成功执行完时，程序就会一直停在这里等待

Linux对一个文件描述符指定的文件或设备有两种工作方式：阻塞与非阻塞：

1、阻塞方式：指试图对fd进行读写时，如果没有数据可读，或者暂时不可写，程序就进入等待状态，直到fd可读或者可写为止。

2、非阻塞方式：是指如果fd没有数据可读，或者不可写，读/写函数可以立即返回，不会等结果。比较朴素的是使用轮询机制，反复请求，拿不到数据就返回一个错误信号，这种非阻塞没什么实际的用处，所以谈论非阻塞I/O，其实大部分还是指的I/O多路复用技术


  select/poll 执行流程如下：

1、先从用户空间拷贝fd_set到内核空间

2、注册一个回调函数，将当前进程挂起（不是休眠）到对应I/O设备的等待队列里，不同的I/O设备都有自己的进程等待队列

3、循环遍历所有要监听的fd对应设备上的等待队列，让当前进程询问队列里有无I/O事件就绪，每次轮询发现有状态变化（I/O事件就绪）的fd，就会回调，唤醒挂起的进程，否则直到超时结束，或者没有设置超时就会一直阻塞。

4、每执行完一次select（一般业务代码也会循环使用select不断的去监听I/O事件），它会把fd_set从内核空间拷贝回用户空间



1、epoll将用户关心的fd放到了Linux内核里的一个事件表，而不是像select/poll那样，每次调用都需要从用户空间复制fd到内核。内核将持久维护加入的fd，减少了内核和用户空间复制数据的性能开销

2、当一个fd有I/O事件就绪（比如读事件），epoll机制无须遍历整个被侦听的fd集合，只要遍历那些被内核I/O事件唤醒而加入就绪队列的fd集合，减少了无用功

3、epoll机制支持的最大fd上限远远大于1024，在1G内存的机器上是10万左右，具体数目可以cat/proc/sys/fs/file-max查看

```java
##### Package java.nio.charset
这个包定义了Charset及相应的encoder和decoder。可以将Charset，CharsetDecoder和CharsetEncoder理解成一个Abstract Factory模式的实现

Charset
代表了一个字符集，同时提供了factory method来构建相应的CharsetDecoder和CharsetEncoder。


Charset提供了以下static的方法：

SortedMap availableCharsets()
       返回当前系统支持的所有Charset对象，用charset的名字作为set的key。

boolean isSupported(String charsetName)
       判断该名字对应的字符集是否被当前系统支持。

Charset forName(String charsetName)
       返回该名字对应的Charset对象。

 

Charset中比较重要的方法有：
String name()
       返回该字符集的规范名。

Set aliases()
       返回该字符集的所有别名。

CharsetDecoder newDecoder()
       创建一个对应于这个Charset的decoder。

CharsetEncoder newEncoder()
       创建一个对应于这个Charset的encoder。

CharsetDecoder
将按某种字符集编码的字节流解码为unicode字符数据的引擎。

CharsetDecoder的输入是ByteBuffer，输出是CharBuffer。进行decode操作时一般按如下步骤进行
1． 调用CharsetDecoder的reset()方法。（第一次使用时可不调用）
2． 调用decode()方法0到n次，将endOfInput参数设为false，告诉decoder有可能还有新的数据送入。
3． 调用decode()方法最后一次，将endOfInput参数设为true，告诉decoder所有数据都已经送入。
4． 调用decoder的flush()方法。让decoder有机会把一些内部状态写到输出的CharBuffer中。

CharsetDecoder reset()
       重置decoder，并清除decoder中的一些内部状态。

CoderResult decode(ByteBuffer in, CharBuffer out, boolean endOfInput)
从ByteBuffer类型的输入中decode尽可能多的字节，并将结果写到CharBuffer类型的输出中。根据decode的结果，可能返回3种CoderResult：CoderResult.UNDERFLOW表示已经没有输入可以decode；CoderResult.OVERFLOW表示输出已满；其它的CoderResult表示decode过程中有错误发生。根据返回的结果，应用程序可以采取相应的措施，比如，增加输入，清除输出等等，然后再次调用decode()方法。

CoderResult flush(CharBuffer out)
有些decoder会在decode的过程中保留一些内部状态，调用这个方法让这些decoder有机会将这些内部状态写到输出的CharBuffer中。调用成功返回CoderResult.UNDERFLOW。如果输出的空间不够，该函数返回CoderResult.OVERFLOW，这时应用程序应该扩大输出CharBuffer的空间，然后再次调用该方法。

CharBuffer decode(ByteBuffer in)
一个便捷的方法把ByteBuffer中的内容decode到一个新创建的CharBuffer中。在这个方法中包括了前面提到的4个步骤，所以不能和前3个函数一起使用。

 
decode过程中的错误有两种：malformed-input CoderResult表示输入中数据有误；unmappable-character CoderResult表示输入中有数据无法被解码成unicode的字符。如何处理decode过程中的错误取决于decoder的设置。对于这两种错误，decoder可以通过CodingErrorAction设置成：

1． 忽略错误
2． 报告错误。（这会导致错误发生时，decode()方法返回一个表示该错误的CoderResult。）
3． 替换错误，用decoder中的替换字串替换掉有错误的部分。

 

CodingErrorAction malformedInputAction()
       返回malformed-input的出错处理。

CharsetDecoder onMalformedInput(CodingErrorAction newAction)
       设置malformed-input的出错处理。

CodingErrorAction unmappableCharacterAction()
       返回unmappable-character的出错处理。

CharsetDecoder onUnmappableCharacter(CodingErrorAction newAction)
       设置unmappable-character的出错处理。

String replacement()
       返回decoder的替换字串。

CharsetDecoder replaceWith(String newReplacement)
       设置decoder的替换字串。

CharsetEncoder
将unicode字符数据编码为特定字符集的字节流的引擎。其接口和CharsetDecoder相类似。

CoderResult
描述encode/decode操作结果的类。

 
CodeResult包含两个static成员：

CoderResult OVERFLOW
       表示输出已满

CoderResult UNDERFLOW
       表示输入已无数据可用。


其主要的成员函数有：

boolean isError()
boolean isMalformed()
boolean isUnmappable()
boolean isOverflow()
boolean isUnderflow()
       用于判断该CoderResult描述的错误。

 

int length()
       返回错误的长度，比如，无法被转换成unicode的字节长度。

void throwException()
       抛出一个和这个CoderResult相对应的exception。

 

CodingErrorAction
表示encoder/decoder中错误处理方法的类。可将其看成一个enum类型。有以下static属性：

CodingErrorAction IGNORE
       忽略错误。

CodingErrorAction REPLACE
       用替换字串替换有错误的部分。

CodingErrorAction REPORT
报告错误，对于不同的函数，有可能是返回一个和错误有关的CoderResult，也有可能是抛出一个CharacterCodingException。
```

