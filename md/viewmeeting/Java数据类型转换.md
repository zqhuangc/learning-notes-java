**1 数据类型转换的种类**
  java数据类型的转换一般分三种,分别是:
  (1). 简单数据类型之间的转换
  (2). 字符串与其它数据类型的转换
  (3). 其它实用数据类型转换

**2 简单数据类型之间的转换**
  在Java中整型、实型、字符型被视为简单数据类型，这些类型由低级到高级分别为(byte，short，char)--int--long--float--double

  简单数据类型之间的转换又可以分为：
  ●低级到高级的自动类型转换
  ●高级到低级的强制类型转换
  ●包装类过渡类型能够转换

**2.1自动类型转换**
  低级变量可以直接转换为高级变量，笔者称之为自动类型转换,例如，下面的语句可以在Java中直接通过：
byte b;
int i=b;
long l=b;
float f=b;
double d=b;
  但是将double型变量赋值给float变量，不加强转的话会报错。


 如果低级类型为char型，向高级类型（整型）转换时，会转换为对应ASCII码值，例如r

  char c='c';
  int i=c;
  System.out.println("output:" i);
输出：output:99;
对于byte,short,char三种类型而言，他们是平级的，因此不能相互自动转换，可以使用下述的强制类型转换。

short i=99;

char c=(char)i;

System.out.println("output:" c);
输出：output:c;


但根据笔者的经验，byte,short,int三种类型都是整型，因此如果操作整型数据时，最好统一使用int型。

**2.2强制类型转换**
  将高级变量转换为低级变量时，情况会复杂一些，你可以使用强制类型转换。即你必须采用下面这种语句格式：

int i=99;

byte b=(byte)i;

char c=(char)i;

float f=(float)i;
double d = (double)f;
  可以想象，这种转换肯定可能会导致溢出或精度的下降，因此笔者并不推荐使用这种转换。


**2.3包装类过渡类型转换**
   在我们讨论其它变量类型之间的相互转换时，我们需要了解一下Java的包装类，所谓包装类，就是可以直接将简单类型的变量表示为一个类，在执行变量类型的相互转换时，我们会大量使用这些包装类。Java共有六个包装类，分别是Boolean、Character、Integer、Long、Float和Double，从字面上我们就可以看出它们分别对应于 boolean、char、int、long、float和double。而String和Date本身就是类。所以也就不存在什么包装类的概念了。
   在进行简单数据类型之间的转换（自动转换或强制转换）时，我们总是可以利用包装类进行中间过渡。
   一般情况下，我们首先声明一个变量，然后生成一个对应的包装类，就可以利用包装类的各种方法进行类型转换了。例如：

例1，当希望把float型转换为double型时：
  float f1=100.00f;
  Float F1=new float(f1);
  Double d1=F1.doubleValue();
//F1.doubleValue()为Float类的返回double值型的方法

​    当希望把double型转换为int型时：

  double d1=100.00; 
  Double D1=new Double(d1);
  int i1=D1.intValue();

  当希望把int型转换为double型时，自动转换：

​    int i1=200;
​    double d1=i1;

  简单类型的变量转换为相应的包装类，可以利用包装类的构造函数。即：
Boolean(boolean value)、Character(char value)、Integer(int value)、Long(long value)、Float(float value)、Double(double value)
而在各个包装类中，总有形为××Value()的方法，来得到其对应的简单类型数据。利用这种方法，也可以实现不同数值型变量间的转换，例如，对于一个双精度实型类，intValue()可以得到其对应的整型变量，而doubleValue()可以得到其对应的双精度实型变量。

3 字符串型与其它数据类型的转换
  通过查阅类库中各个类提供的成员方法可以看到，几乎从java.lang.Object类派生的所有类提供了toString()方法，即将该类转换为字符串。例如：Characrer,Integer,Float,Double,Boolean,Short等类的toString()方法toString()方法用于将字符、整数、浮点数、双精度数、逻辑数、短整型等类转换为字符串。如下所示：

int i1=10;
float f1=3.14f;
double d1=3.1415926;
Integer I1=new Integer(i1);
//生成Integer类r
Float F1=new Float(f1);
//生成Float类r
Double D1=new Double(d1);
//生成Double类r
//分别调用包装类的toString()方法转换为字符串
String si1=I1.toString();
String sf1=F1.toString();
String sd1=D1.toString();
Sysytem.out.println("si1" si1);
Sysytem.out.println("sf1" sf1);
Sysytem.out.println("sd1" sd1);

**4、将字符型直接做为数值转换为其它数据类型** 

 将字符型变量转换为数值型变量实际上有两种对应关系，在我们在第一部分所说的那种转换中，实际上是将其转换成对应的ASCII码，但是我们有时还需要另一种转换关系，例如，'1'就是指的数值1，而不是其ASCII码，对于这种转换，我们可以使用Character的getNumericValue(char ch)方法。

**5、Date类与其它数据类型的相互转换**
​      整型和Date类之间并不存在直接的对应关系，只是你可以使用int型为分别表示年、月、日、时、分、秒，这样就在两者之间建立了一个对应关系，在作这种转换时，你可以使用Date类构造函数的三种形式：
Date(int year, int month, int date)：以int型表示年、月、日
Date(int year, int month, int date, int hrs, int min)：以int型表示年、月、日、时、分
Date(int year, int month, int date, int hrs, int min, int sec)：以int型表示年、月、日、时、分、秒r
在长整型和Date类之间有一个很有趣的对应关系，就是将一个时间表示为距离格林尼治标准时间1970年1月1日0时0分0秒的毫秒数。对于这种对应关系，Date类也有其相应的构造函数：Date(long date)
获取Date类中的年、月、日、时、分、秒以及星期你可以使用Date类的getYear()、getMonth()、getDate()、getHours()、getMinutes()、getSeconds()、getDay()方法，你也可以将其理解为将Date类转换成int。
而Date类的getTime()方法可以得到我们前面所说的一个时间对应的长整型数，与包装类一样，Date类也有一个toString()方法可以将其转换为String类。
有时我们希望得到Date的特定格式，例如20020324，我们可以使用以下方法，首先在文件开始引入，

import java.text.SimpleDateFormat;
import java.util.*;
java.util.Date date = new java.util.Date();/
/如果希望得到YYYYMMDD的格式
SimpleDateFormat sy1=new SimpleDateFormat("yyyyMMDD");
String dateFormat=sy1.format(date);
//如果希望分开得到年，月，日
SimpleDateFormat sy=new SimpleDateFormat("yyyy");
SimpleDateFormat sm=new SimpleDateFormat("MM");
SimpleDateFormat sd=new SimpleDateFormat("dd");
String syear=sy.format(date);
String smon=sm.format(date);
String sday=sd.format(date);


附加：

在java中除了这些转换之外基本数据类型还可以被隐式的转换成String,例如：

System.out.print("转换"+100);//如果在数据前面有字符串用+连接//就会隐式的转换成

String


1.字符型转时间型：2005-10-1
reportdate_str ＝ “2005-10-01”；
reportdate_str ＝ reportdate_str ＋ “ 00:00:00.0”
reportdate = java.sql.Timestamp.valueOf(reportdate_str);

2.时间型转字符型：
V_DATE =  reportdate.toString();

3.将long型转化为String型
​    long APP_UNIT = (long) userview.getDEPT_ID();
​    String S_APP_UNIT = String.valeOf(APP_UNIT);
  基本类型s都可以通过String.valeOf(s)来转化为String型。