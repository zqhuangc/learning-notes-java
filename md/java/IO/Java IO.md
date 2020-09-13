## 各种流

new File

#### 字节流

FileInputStream  
FileOutputStream  
ByteArrayInputStream  

#### 字符流  
Writer out =new FileWriter(f,true);  

#### 管道流  
管道流主要可以进行两个线程之间的通信。
PipedOutputStream 管道输出流
PipedInputStream 管道输入流

#### 打印流  
PrintStream

#### 输入输出重定向  
System.setOut

####  缓冲流
BufferedReader
注意： BufferedReader只能接受字符流的缓冲区，因为每一个中文需要占据两个字节，所以需要将System.in这个字节输入流变为字符输入流，采用：
BufferedReader buf = new BufferedReader(new InputStreamReader(System.in));

####  Scanner类
Scanner scan = new Scanner(System.in);

####  数据操作流：

DataOutputStream、DataInputStream

#### 合并流：

SequenceInputStream 主要用来将2个流合并在一起，比如将两个txt中的内容合并为另外一个txt。

#### 文件压缩

* ZipOutputStream类
  java中的每一个压缩文件都是可以使用ZipFile来进行表示的
* ZipInputStream类
#### 对象的序列化
对象序列化就是把一个对象变为二进制数据流的一种方法。
一个类要想被序列化，就行必须实现java.io.Serializable接口。虽然这个接口中没有任何方法，就如同之前的cloneable接口一样。实现了这个接口之后，就表示这个类具有被序列化的能力。

* Externalizable接口

被Serializable接口声明的类的对象的属性都将被序列化，但是如果想自定义序列化的内容的时候，就需要实现Externalizable接口。

## 字符流

### Reader

 用于读取字符流的抽象类。子类必须实现的方法只有 read(char[], int, int) 和 close()。  
|—BufferedReader ：从字符输入流中读取文本，缓冲各个字符，从而实现字符、数组和行的高效读取。 可以指定缓冲区的大小，或者可使用默认的大小。大多数情况下，默认值就足够大了。  
|—LineNumberReader ：跟踪行号的缓冲字符输入流。此类定义了方法 setLineNumber(int) 和getLineNumber()，它们可分别用于设置和获取当前行号。  
|—InputStreamReader ：是字节流通向字符流的桥梁：它使用指定的 charset 读取字节并将其解码为字符。它使用的字符集可以由名称指定或显式给定，或者可以接受平台默认的字符集。  
|—FileReader： ：用来读取字符文件的便捷类。此类的构造方法假定默认字符编码和默认字节缓冲区大小 都是适当的。要自己指定这些值，可以先在 FileInputStream 上构造一个InputStreamReader。 
|—CharArrayReader  
|—StringReader  

### Writer

 写入字符流的抽象类。子类必须实现的方法仅有 write(char[], int, int)、flush() 和 close()。 


|—BufferedWriter： ：将文本写入字符输出流，缓冲各个字符，从而提供单个字符、数组和字符串的高效写入。  
|—OutputStreamWriter ：是字符流通向字节流的桥梁：可使用指定的 charset 将要写入流中的字符编码成字节。它使用的字符集可以由名称指定或显式给定，否则将接受平台默认的字符集。  
|—FileWriter： ：用来写入字符文件的便捷类。此类的构造方法假定默认字符编码和默认字节缓冲区大小都是可接受的。要自己指定这些值，可以先在 FileOutputStream 上构造一个 OutputStreamWriter。  
|—PrintWrite  
|—CharArrayWriter  
|—StringWriter  

## 字节流

### InputStream

是表示字节输入流的所有类的超类。   
|— FileInputStream： ：从文件系统中的某个文件中获得输入字节。哪些文件可用取决于主机环境。FileInputStream 用于读取诸如图像数据之类的原始字节流。要读取字符流，请考虑使用 FileReader。  
|— FilterInputStream： ：包含其他一些输入流，它将这些流用作其基本数据源，它可以直接传输数据或提供一些额外的功能。  
|— BufferedInputStream ：该类实现缓冲的输入流。  
|— Stream ：  
|— ObjectInputStream ：  
|— PipedInputStream  

### OutputStream

此抽象类是表示输出字节流的所有类的超类。   
|— FileOutputStream ：文件输出流是用于将数据写入 File 或 FileDescriptor 的输出流。  
|— FilterOutputStream ：此类是过滤输出流的所有类的超类。  
|— BufferedOutputStream ：该类实现缓冲的输出流。  
|— PrintStream ：  
|— DataOutputStream ：  
|— ObjectOutputStream ：  
|— PipedOutputStream：  

规律总结  
IO流中的对象：其实很简单，就是读取和写入。但是因为功能的不同，流的体系中提供 N 多的对象。那么开始时，到 底该用哪个对象更为合适呢？这就需要明确流的操作规律。 
1 ，明确源和目的。  
数据源：就是需要读取，可以使用两个体系：InputStream、Reader；  
数据汇：就是需要写入，可以使用两个体系：OutputStream、Writer；  
2 ，操作的数据是否是纯文本数据？  
如果是：数据源：Reader 数据汇：Writer
如果不是：数据源：InputStream 数据汇：OutputStream  
3 ，虽然确定了一个体系，但是该体系中有太多的对象，到底用哪个呢？明确操作的数据设备。
数据源对应的设备：硬盘(File)，内存(数组)，键盘(System.in)  
数据汇对应的设备：硬盘(File)，内存(数组)，控制台(System.out)。  
4 ，需要在基本操作上附加其他功能吗？比如缓冲。  
根据规律实例化演示  
需求：读取键盘录入，将数据存储到一个文件中。  
规律分析  
1，明确体系：  
源：InputStream ，Reader  
目的：OutputStream ，Writer  
2，明确数据：  
源：是纯文本吗？是 Reader  
目的；是纯文本吗？是 Writer  
3，明确设备：
源：键盘，System.in  
目的：硬盘，FileWriter  
InputStream in = System.in;  
FileWriter fw = new FileWriter(“a.txt”);  
4，需要额外功能吗？  
需要，因为源明确的体系时Reader。可是源的设备是System.in。所以为了方便于操作文本数据，将源转成字符流。需要转换流。  
InputStreamReader InputStreamReader isr = new InputStreamReader(System.in); 
FileWriter fw = new FileWriter(“a.txt”); 
BufferedReader bufr = new BufferedReader(new InputStreamReader(System.in)); 
BufferedWriter bufw = new BufferedWriter(new FileWriter(“a.txt”));

## 区别

### 关于字节流和字符流的区别  
实际上字节流在操作的时候本身是不会用到**缓冲区**的，是文件本身的直接操作的，但是字符流在操作的时候下后是会用到缓冲区的，是通过缓冲区来操作文件的。  
读者可以试着将上面的字节流和字符流的程序的最后一行关闭文件的代码注释掉，然后运行程序看看。你就会发现使用字节流的话，文件中已经存在内容，但是使用字符流的时候，文件中还是没有内容的，这个时候就要刷新缓冲区。 

### 使用字节流好还是字符流好呢？  
字节流。首先因为硬盘上的所有文件都是以字节的形式进行传输或者保存的，包括图片等内容。但是字符只是在内存中才会形成的，所以在开发中，字节流使用广泛



## File.separator
https://blog.csdn.net/chindroid/article/details/7735832
在Windows下的路径分隔符和Linux下的路径分隔符是不一样的，当直接使用绝对路径时，跨平台会暴出“No such file or diretory”的异常。

比如说要在temp目录下建立一个test.txt文件，在Windows下应该这么写：
File file1 = new File ("C:\tmp\test.txt");
在Linux下则是这样的：
File file2 = new File ("/tmp/test.txt");
如果要考虑跨平台，则最好是这么写：
File myFile = new File("C:" + File.separator + "tmp" + File.separator, "test.txt");
File类有几个类似separator的静态字段，都是与系统相关的，在编程时应尽量使用。
separatorChar
public static final char separatorChar
与系统有关的默认名称分隔符。此字段被初始化为包含系统属性 file.separator 值的第一个字符。在 UNIX 系统上，此字段的值为 '/'；在 Microsoft Windows 系统上，它为 '\'。
separator
public static final String separator
与系统有关的默认名称分隔符，为了方便，它被表示为一个字符串。此字符串只包含一个字符，即 separatorChar。
pathSeparatorChar
public static final char pathSeparatorChar
与系统有关的路径分隔符。此字段被初始为包含系统属性 path.separator 值的第一个字符。此字符用于分隔以路径列表 形式给定的文件序列中的文件名。在 UNIX 系统上，此字段为 ':'；在 Microsoft Windows 系统上，它为 ';'。
pathSeparator
public static final String pathSeparator
与系统有关的路径分隔符，为了方便，它被表示为一个字符串。此字符串只包含一个字符，即 pathSeparatorChar。