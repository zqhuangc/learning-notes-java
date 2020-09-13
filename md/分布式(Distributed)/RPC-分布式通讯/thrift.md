RPC本质上是一种 Inter-process communication（IPC）——进程间通信的形式。常见的进程间通信方式如管道、共享内存是同一台物理机上的两个进程间的通信，而RPC就是两个在不同物理机上的进程之间的通信。概括的说，RPC就是在一台机器上调用另一台机器上的方法，这种调用在远程机器上对代码的执行就像在本机上对代码的执行一样，只是迁移了一个执行环境而已.  
  
这些RPC工具大多使用“接口描述语言” —— interface description language (IDL) 来提供跨平台跨语言的服务调用。现在生产中用的最多的**IDL**是Google开源的protobuf。


## Thrift
Thrift通过一个中间语言(IDL, 接口定义语言)来定义RPC的接口和数据类型，然后通过一个编译器生成不同语言的代码（目前支持C++,Java, Python, PHP, Ruby, Erlang, Perl, Haskell, C#, Cocoa, Smalltalk和OCaml）,并由生成的代码负责RPC协议层和传输层的实现。  

协议栈的其他模块都是Thrift的运行时模块：

* 底层IO模块，负责实际的数据传输，包括Socket，文件，或者压缩数据流等。

* TTransport负责以字节流方式发送和接收Message，是底层IO模块在Thrift框架中的实现，每一个底层IO模块都会有一个对应TTransport来负责Thrift的字节流(Byte Stream)数据在该IO模块上的传输。例如TSocket对应Socket传输，TFileTransport对应文件传输。

* TProtocol主要负责结构化数据组装成Message，或者从Message结构中读出结构化数据。TProtocol将一个有类型的数据转化为字节流以交给TTransport进行传输，或者从TTransport中读取一定长度的字节数据转化为特定类型的数据。如int32会被TBinaryProtocol Encode为一个四字节的字节数据，或者TBinaryProtocol从TTransport中取出四个字节的数据Decode为int32。

* TServer负责接收Client的请求，并将请求转发到Processor进行处理。TServer主要任务就是高效的接受Client的请求，特别是在高并发请求的情况下快速完成请求。

* Processor(或者TProcessor)负责对Client的请求做出相应，包括RPC请求转发，调用参数解析和用户逻辑调用，返回值写回等处理步骤。Processor是服务器端从Thrift框架转入用户逻辑的关键流程。Processor同时也负责向Message结构中写入数据或者读出数据。

Thrift的模块设计非常好，在每一个层次都可以根据自己的需要选择合适的实现方式。同时也应该注意到Thrift目前的特性并不是在所有的程序语言中都支持。例如C++实现中有TDenseProtocol没有TTupleProtocol，而Java实现中有TTupleProtocol没有TDenseProtocol。  

* 使用thrift流程
```
(1). 利用IDL定义数据结构及服务
(2). 利用代码生成工具将(1)中的IDL编译成对应语言（如C++、JAVA），编译后得到基本的框架代码
(3). 在(2)中框架代码基础上完成完整代码（纯C++代码、JAVA代码等）
```

### 数据类型
* Base Types（基本类型）  
* Structs（结构体）  
* Containers（容器）  
强类型
```
有三种可用类型：

list<type>:映射为STL vector，Java ArrayList，或脚本语言中的native array。。
set<type>: 映射为为STL set，Java HashSet，Python中的set，或PHP/Ruby中的native dictionary。
Map<type1,type2>：映射为STL map，Java HashMap，PHP associative array，或Python/Ruby dictionary。
```
* Exceptions（异常）  
* Services（服务）  
```java
service <name> {
<returntype> <name>(<arguments>)
[throws (<exceptions>)]
...
}

//例子：

service StringCache {
void set(1:i32 key, 2:string value),
string get(1:i32 key) throws (1:KeyNotFound knf),
void delete(1:i32 key)
}
```

### Tserver
Thrift目前支持的Server模型包括：

1.  TSimpleServer：使用阻塞IO的单线程服务器，主要用于调试
2.  TThreadedServer：使用阻塞IO的多线程服务器。每一个请求都在一个线程里处理，并发访问情况下会有很多线程同时在运行。
3.  TThreadPoolServer：使用阻塞IO的多线程服务器，使用线程池管理处理线程。
4.  TNonBlockingServer：使用非阻塞IO的多线程服务器，使用少量线程既可以完成大并发量的请求响应，必须使用TFramedTransport。

TServer对象通常如下工作：

1） 使用TServerTransport获得一个TTransport
2） 使用TTransportFactory，可选地将原始传输转换为一个适合的应用传输（典型的是使用TBufferedTransportFactory）
3） 使用TProtocolFactory，为TTransport创建一个输入和输出
4） 调用TProcessor对象的process()方法

### TTransport  
Thrift最底层的传输可以使用Socket，File和Zip来实现  
在TTransport这一层，数据是按字节流(Byte Stream)方式处理的，即传输层看到的是一个又一个的字节，并把这些字节按照顺序发送和接收。

### TProtocol
TProtocol的主要任务是把TTransport中的字节流转化为数据流(Data Stream),在TProtocol这一层就会出现具有数据类型的数据，如整型，浮点数，字符串，结构体等。    
Thrift 可以让用户选择客户端与服务端之间传输通信协议的类别，在传输协议上总体划分为文本 (text) 和二进制 (binary) 传输协议，为节约带宽，提高传输效率，一般情况下使用二进制类型的传输协议为多数。常用协议有以下几种：
```java
TBinaryProtocol: 二进制格式
TCompactProtocol: 高效率的、密集的二进制编码格式
TJSONProtocol: 使用 JSON 的数据编码协议进行数据传输
TSimpleJSONProtocol: 提供JSON只写协议, 生成的文件很容易通过脚本语言解析。
TDebugProtocol: 使用易懂的可读的文本格式，以便于debug
```

### TProcessor/Processor
Processor是TServer从Thrift框架转到用户逻辑的关键流程.

TProcessor对于一次RPC调用的处理过程可以概括为：

1. TServer接收到RPC请求之后，调用TProcessor.process进行处理
2. TProcessor.process首先调用TTransport.readMessageBegin接口，读出RPC调用的名称和RPC调用类型。如果RPC调用类型是RPC Call，则调用TProcessor.process_fn继续处理，对于未知的RPC调用类型，则抛出异常。
3. TProcessor.process_fn根据RPC调用名称到自己的processMap中查找对应的RPC处理函数。如果存在对应的RPC处理函数，则调用该处理函数继续进行请求响应。不存在则抛出异常。  
  a) 在这一步调用的处理函数，并不是最终的用户逻辑。而是对用户逻辑的一个包装。  
  b) processMap是一个标准的std::map。Key为RPC名称。Value是对应的RPC处理函数的函数指针。 processMap的初始化是在Processor初始化的时候进行的。Thrift虽然没有提供对processMap做修改的API，但是仍可以通过继承TProcessor来实现运行时对processMap进行修改，以达到打开或关闭某些RPC调用的目的。

RPC处理函数是RPC请求处理的最后一个步骤，它主要完成以下三个步骤：
```
a) 调用RPC请求参数的解析类，从TProtocol中读入数据完成参数解析。不管RPC调用的参数有多少个，Thrift都会将参数放到一个Struct中去。Thrift会检查读出参数的字段ID和字段类型是否与要求的参数匹配。对于不符合要求的参数都会跳过。这样，RPC接口发生变化之后，旧的处理函数在不做修改的情况，可以通过跳过不认识的参数，来继续提供服务。进而在RPC框架中提供了接口的多Version支持。

b) 参数解析完成之后，调用用户逻辑，完成真正的请求响应。

c) 用户逻辑的返回值使用返回值打包类进行打包，写入TProtocol。
```

### Thrift RPC Version
### Thrift Generator