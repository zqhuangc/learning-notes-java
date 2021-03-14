Unicode 6.0.0

字节序（byte ordering）

数据类型：原始类型和引用类型

原始类型：

引用类型：类类型，接口类型，数组类型

原始值和引用值：变量类型、参数传递、方法返回和运算操作

编译期类型检查



原始类型，虚拟机的字节码指令本身就可以确定它的指令操作数的类型是什么，如 iadd，ladd，fadd、dadd。



java虚拟机是直接支持对象的，这里的对象可以是动态分配的某个类的实例，也可以指某个数组

虚拟机中使用 reference 类型来表示对某个对象的引用，可看做指向对象的指针。每一个对象都可能存在多个指向它的引用，对象的操作、传递和检查都通过引用它的 reference 类型的数据来进行



* 原始类型

原始类型： 数值类型、boolean 类型、returnAddress类型

数值类型：整数类型（byte 8，short 16，int 32，long 64，char 16 无符号），浮点类型（float，double）

二进制补码整数，默认 0

> returnAddress类型是指向某个操作码（opcode）的指针，此操作码与 Java虚拟机 指令相对应，不能直接与数据类型相对应   （jsr，ret，jsr_w指令，jdk7 已禁用，导致 returnAddress **名存实亡**）



newarray 指令  创建数组

boolean 类型数组的访问和修改共用 byte 类型数组的  baload 和 bastore 指令



* 引用类型

类类型（class type），数组类型（array type），接口类型（interface type）

分别指向动态创建的类实例、数组实例和实现了某个接口的类实例和数组实例

默认值 null

数组类型的组件类型（component type），元素类型（原生，类，接口）



* pc 寄存器（program counter 程序计数器）

  一条java 虚拟机线程只会执行一个方法的代码，当前方法， 若方法不是 native 的，pc 保存Java虚拟机正在执行的字节码指令的地址


* java 虚拟机栈

  与线程同时创建，用于存储栈帧，用于存储局部变量与一些尚未算好的结果。栈帧可以在堆中分配。

  StackOverflowError、OutOfMemoryError



自动内存管理

方法区：存储每一个类的结构信息，如运行时常量池、字段、和方法数据、构造函数和普通方法的字节码内容，还包括一些在类、实例、接口初始化时用到的特殊方法



运行时常量池是 class 文件中每一个类或接口的常量池表（constant_pool table）的运行时表示形式





### 栈帧

每一个栈帧都有自己的本地变量表（local variable）、操作数栈（operand stack）和指向当前方法所属的类的运行时常量池的引用



#### 局部变量表

局部变量使用索引来进行定位。首个局部变量的索引值为 0，存储实例方法所在对象的引用 this



#### 操作数栈

LIFO

ajva虚拟机提供字节码指令从局部变量表或者对象实例的字段中复制常量或变量值到操作数栈，也提供字节码指令从栈获取数据、处理数据以及结果重新入栈

long 和 double 占用两个单位索引

dup 和 swap 指令  

不关注操作数具体数据类型 raw type





#### 动态链接

在 class 文件里面，一个方法若要调用其他方法，或者访问成员变量，需要通过符号引用来（symbolic reference）表示

每一个栈帧（§2.6）内部都包含一个指向运行时常量池（§2.5.5）的引用来支持当前方法
的代码实现动态链接（Dynamic Linking）。在 Class 文件里面，描述一个方法调用了其他方法，
或者访问其成员变量是通过符号引用（Symbolic Reference）来表示的，**动态链接的作用**就是
将这些符号引用所表示的方法转换为实际方法的直接引用。类加载的过程中将要解析掉尚未被解析
的符号引用，并且将变量访问转化为访问这些变量的存储结构所在的运行时内存位置的正确偏移
量。
由于动态链接的存在，通过晚期绑定（Late Binding）使用的其他类的方法和变量在发生
变化时，将不会对调用它们的方法构成影响。





类加载的过程中





#### 浮点模式

ACC_STRICT



#### 初始化方法的特殊命名

名为\<init> 的特殊实例初始化方法

实例初始化方法只能在实例的初始化期间，通过 invokespecial 指令调用



\<clinit> 方法想成为类或接口初始化方法 ，须设置 ACC_STATIC 标志





* 异常

athrow 字节码指令

由 Java 虚拟机执行的每一个方法都会配有零至多个异常处理器（Exception Handlers）

在 Class 文件里面，每个方法的异常处理器都存
储在一个表中





## 字节码指令集

Java 虚拟机的指令由一个字节长度的、代表着某种特定操作含义的操作码（Opcode）以及
跟随其后的零至多个代表此操作所需参数的操作数（Operands）所构成。虚拟机中许多指令并不
包含操作数，只有一个操作码



对于大部分为与数据类型相关的字节码指令，他们的操作码助记符中都有特殊的字符来表明专
门为哪种数据类型服务：**i 代表对 int 类型的数据操作，l 代表 long，s 代表 short，b 代表 byte，**
**c 代表 char，f 代表 float，d 代表 double，a 代表 reference。**也有一些指令的助记符中没
有明确的指明操作类型的字母，例如 arraylength 指令，它没有代表数据类型的特殊字符，但
操作数永远只能是一个数组类型的对象。还有另外一些指令，例如无条件跳转指令 goto 则是与数
据类型无关的。



并非每种数据类型和每一种操作都有对应的指令。有一些单独的指令可以在必
要的时候用来将一些不支持的类型转换为可被支持的类型。





#### 加载和存储指令

  将一个局部变量加载到操作栈的指令包括有：iload、iload_<n>、lload、lload_<n>、
fload、fload_<n>、dload、dload_<n>、aload、aload_<n>
  将一个数值从操作数栈存储到局部变量表的指令包括有：istore、istore_<n>、
lstore、lstore_<n>、fstore、fstore_<n>、dstore、dstore_<n>、astore、
astore_<n>
  将一个常量加载到操作数栈的指令包括有：bipush、sipush、ldc、ldc_w、ldc2_w、
aconst_null、iconst_m1、iconst_<i>、lconst_<l>、fconst_<f>、dconst_<d>
  扩充局部变量表的访问索引的指令：wide



#### 运算指令

算术指令包括：
  加法指令：iadd、ladd、fadd、dadd
  减法指令：isub、lsub、fsub、dsub
  乘法指令：imul、lmul、fmul、dmul
  除法指令：idiv、ldiv、fdiv、ddiv
  求余指令：irem、lrem、frem、drem
  取反指令：ineg、lneg、fneg、dneg
  位移指令：ishl、ishr、iushr、lshl、lshr、lushr
  按位或指令：ior、lor
  按位与指令：iand、land
  按位异或指令：ixor、lxor
  局部变量自增指令：iinc
  比较指令：dcmpg、dcmpl、fcmpg、fcmpl、lcmp



#### 类型转换指令

Java 虚拟机直接支持（译者注：“直接支持”意味着转换时无需显式的转换指令）以下数值
的宽化类型转换（Widening Numeric Conversions，小范围类型向大范围类型的安全转换）：
  int 类型到 long、float 或者 double 类型
  long 类型到 float、double 类型
  float 类型到 double 类型

窄化类型转换（Narrowing Numeric Conversions）指令包括有：i2b、i2c、i2s、l2i、
f2i、f2l、d2i、d2l 和 d2f。窄化类型转换可能会导致转换结果产生不同的正负号、不同的数
量级，转换过程很可能会导致数值丢失精度。



#### 对象创建与操作

  创建类实例的指令：new

  创建数组的指令：newarray，anewarray，multianewarray
  访问类字段（static 字段，或者称为类变量）和实例字段（非 static 字段，或者成为
实例变量）的指令：getfield、putfield、getstatic、putstatic
  把一个数组元素加载到操作数栈的指令：baload、caload、saload、iaload、laload、
faload、daload、aaload
  将一个操作数栈的值储存到数组元素中的指令：bastore、castore、sastore、
iastore、fastore、dastore、aastore
  取数组长度的指令：arraylength
  检查类实例类型的指令：instanceof、checkcast

#### 操作数栈管理指令
Java 虚拟机提供了一些用于直接操作操作数栈的指令，包括：pop、pop2、dup、dup2、
dup_x1、dup2_x1、dup_x2、dup2_x2 和 swap。



#### 方法调用和返回指令
以下四条指令用于方法调用：
invokevirtual 指令用于调用对象的实例方法，根据对象的实际类型进行分派（虚方法分
派），这也是 Java 语言中最常见的方法分派方式。
invokeinterface 指令用于调用接口方法，它会在运行时搜索一个实现了这个接口方法的
对象，找出适合的方法进行调用。
invokespecial 指令用于调用一些需要特殊处理的实例方法，包括实例初始化方法（§
2.9）、私有方法和父类方法。
invokestatic 指令用于调用类方法（static 方法）。
而方法返回指令则是根据返回值的类型区分的，包括有 ireturn（当返回值是 boolean、
byte、char、short 和 int 类型时使用）、lreturn、freturn、dreturn 和 areturn，另
外还有一条 return 指令供声明为 void 的方法、实例初始化方法、类和接口的类初始化方法使用。

#### 控制转移指令

控制转移指令可以让 Java 虚拟机有条件或无条件地从指定指令而不是控制转移指令的下一
条指令继续执行程序。控制转移指令包括有：
  条件分支：ifeq、iflt、ifle、ifne、ifgt、ifge、ifnull、ifnonnull、if_icmpeq、
if_icmpne、if_icmplt, if_icmpgt、if_icmple、if_icmpge、if_acmpeq 和
if_acmpne。
  复合条件分支：tableswitch、lookupswitch
  无条件分支：goto、goto_w、jsr、jsr_w、ret

#### 抛出异常
在程序中显式抛出异常的操作会由 athrow 指令实现，除了这种情况，还有别的异常会在其
他 Java 虚拟机指令检测到异常状况时由虚拟机自动抛出。



#### 同步

管程（Monitor）

虚拟机可以从方法常量池中的方法表结构（method_info Structure

）
中的 ACC_SYNCHRONIZED 访问标志区分一个方法是否同步方法。当方法调用时，调用指令将会
检查方法的 ACC_SYNCHRONIZED 访问标志是否被设置，如果设置了，执行线程将先持有管程，
然后再执行方法，最后再方法完成（无论是正常完成还是非正常完成）时释放管程。在方法执行期
间，执行线程持有了管程，其他任何线程都无法再获得同一个管程。如果一个同步方法执行期间抛
出了异常，并且在方法内部无法处理此异常，那这个同步方法所持有的管程将在异常抛到同步方法
之外时自动释放。
同步一段指令集序列通常是由 Java 语言中的 synchronized 块来表示的，Java 虚拟机的
指令集中有 monitorenter 和 monitorexit 两条指令来支持 synchronized 关键字的语义，
正确实现 synchronized 关键字需要编译器与 Java 虚拟机两者协作支持



# 编译

Oracle 的 JDK 包括两部分内容：一部分是将 Java 源代码编译成 Java 虚拟机的指令集的编译器，另一部分是用于 Java 虚拟机的运行时环境。





javap

所有指令的格式如下：

```
<index> <opcode> [<operand1> [<operand2>...]] [<comment>]

<index>是 code[]数组中的指令的操作码的索引，此处的 code[]数组就是存储当前方法的
Java 虚拟机字节码的 Code 属性中的 code[]数组（§4.7.3）。也可以认为<index>是相对于
方法起始处的字节偏移量。<opcode>为指令的操作码的助记符号，<operandN>是指令的操作数，
一条指令可以有 0 至多个操作数。<comment>为行尾的语法注释，譬如：

8 bipush 100 // Push int constant 100

在每一行中，在表示运行时常量池索引的操作数前，会井号（’#’）开头，在指令后的注释中，
会带有这个操作数的描述
10 ldc #1 // Push float constant 100.0
```





Java虚拟机经常利用操作码隐式包含某些操作数，譬如指令iconst_<i> 中的i表示的int
常量 −1、0、1、2、3、4、5。iconst_0 表示把 int 型的 0 值压入操作数栈





很多数值常量，以及对象、字段和方法，都是通过当前类的运行时常量池进行访问的





如果传递了 n 个参数给某个实例方法，则当前栈帧会按照约定的顺序接收这些参数，将它们
保存为方法的第 1 个至第 n 个局部变量之中。





方法调用

对普通实例方法调用是在运行时根据对象类型进行分派的（相当于在 C++中所说的“虚方法”），这类方法通过调用 invokevirtual 指令实现，每条 invokevirtual 指令都会带有一个表示索引的参数，**运行时常量池在该索引处的项为某个方法的符号引用**，这个符号引用可以提供方法所在对象的类型的内部二进制名称、方法名称和方法描述符



invokevirtual 指令操作数（在上面示例中的运行时常量池索引#4）不是 Class 实例中方
法指令的偏移量。编译器并不需要了解 Class 实例的内部布局，它只需要产生方法的符号引用并
保存于运行时常量池即可，这些运行时常量池项将会在执行时转换成调用方法的实际地址。Java
虚拟机指令集中其他指令在访问 Class 实例时也采用相同的方式。



invokestatic 指令用于调用类方法。

invokespecial 指令用于调用实例初始化方法，它也可以用
来调用父类方法和私有方法。

所有使用 invokespecial 指令调用的方法都需要 this 作为第一个参数，保存在
第一个局部变量之中。

**如果编译器要调用某个方法，必须先产生这个方法描述符，描述符中包含了方法实际参数和返**
**回类型。**编译器在方法调用时不会处理参数的类型转换问题，只是简单地将参数的压入操作数栈，
且不改变其类型。通常，编译器会先把一个方法所在的对象的引用压入操作数栈，方法参数则按顺
序跟随这个对象之后入栈。编译器在生成 invokevirtual 指令时，也会生成这条指令所引用的
描述符，这个描述符提供了方法参数和返回值的信息。作为方法解析（§5.4.3.3）时的一个特
殊处理过程，一个用于调用 java.lang.invoke.MethodHandle 的 invoke()或者
invokeExact()方法的 invokevirtual 指令会提供一个方法描述符，这个方法描述符符合语
法规则，并且在描述符中确定的类型信息将会被解析。



编译器会使用 tableswitch 和 lookupswitch 指令来生成 switch 语句的编译代码。
tableswitch 指令用于表示 switch 结构中的 case 语句块，可以高效地从索引表中确定 case
语句块的分支偏移量。当 switch 语句中提供的条件值不能从索引表中确定任何一个 case 语句块
的分支偏移量时， default 分支将起作用。





注解：

如果编译器遇到一个被注解的、声明为在运行时可见的包，那编译器将会生成一个表示接口的
Class 文件，内部形式为（§4.2.1）“package-name.package-info”。这个接口有默认的
访问权限（“package-private”），并且没有父接口。它的 ClassFile 结构（§4.1）中的
ACC_INTERFACE 和 ACC_ABSTRACT 的标志（表 4.1）会被自动设置。如果 Class 文件的版本
号小于 50.0，则不设置 ACC_SYNTHETIC 标志；如果 Class 文件的版本号为 50.0 或更高，则
必须设置 ACC_SYNTHETIC 标志。





# Class 文件格式

每个 Class 文件都是由 8 字节为单位的字节流组成，所有的 16 位、32 位和 64 位长度的数
据将被构造成 2 个、4 个和 8 个 8 字节单位来表示。

各种内部表示名称 描述符和签名 常量池 字段 方法 属性 格式检查 约束 验证

```
ClassFile {
    u4 magic;
    u2 minor_version;
    u2 major_version;
    u2 constant_pool_count;
    cp_info constant_pool[constant_pool_count-1];// 
    u2 access_flags;// 掩码标志，表示某个类或者接口的访问权限及基础属性。
    u2 this_class;
    u2 super_class;
    u2 interfaces_count;
    u2 interfaces[interfaces_count];// 接口表，interfaces[]数组中的每个成员的值必须是一个对 constant_pool 表中项目的一个有效索引值
    u2 fields_count;
    field_info fields[fields_count]; // 字段表，fields[]数组中的每个成员都必须是一个 fields_info 结构
    u2 methods_count;
    method_info methods[methods_count];// 方法表，methods[]数组中的每个成员都必须是一个 method_info 结构
    u2 attributes_count;
    attribute_info attributes[attributes_count];// 属性表，attributes 表的每个项的值必须是 attribute_info 结构
}
```



```
字段描述符
字符  类型  含义
B  byte  有符号字节型数
C  char  Unicode 字符，UTF-16 编码
D  double  双精度浮点数
F  float  单精度浮点数
I  int  整型数
J  long  长整数
S  short  有符号短整数
Z  boolean  布尔值 true/false
L Classname;  reference  一个名为<Classname>的实例
[  reference  一个一维数组

方法描述符



field_info {
    u2 access_flags;
    u2 name_index;
    u2 descriptor_index;
    u2 attributes_count;
    attribute_info attributes[attributes_count];
}

method_info {
    u2 access_flags;
    u2 name_index;
    u2 descriptor_index;
    u2 attributes_count;
    attribute_info attributes[attributes_count];
}

attribute_info {
    u2 attribute_name_index;
    u4 attribute_length;
    u1 info[attribute_length];
}
```





注解类型必定带有 ACC_ANNOTATION 标记，如果设置了 ANNOTATION 标记，ACC_INTERFACE 也必须被同时设置。





所有方法调用的参数都必须与方法描述符相兼容（JLS ）

方法调用指令的目标实例的类型必须与指令所指定的类或接口的类型相兼容



编译时检查 版本偏差（Version Skew）





对于不包含 **StackMapTable** 属性的 Class 文件（这样的 Class 文件其版本号必须小于或
等于 49.0）需要使用类型推断的方式来验证。



方法分支跳转一定不能超过 code[]数组的范围

所有控制流指令的目标都应该是某条指令的起始处

方法会显式声明自己局部变量表的大小，指令所访问或修改的局部变量索引绝不能大于这
个限制值。



数据流分析器（Data-Flow Analyzer）为每条指令都设置一个“变更位”（Changed Bit），
用来表示指令是否需要被检测。



## 运行时常量池

 constant_pool 表     

运行时常量池中的所有引用最初都是符号引用。



### 属性

SourceFile

SourceDebugExtension

* LineNumberTable

```
LineNumberTable_attribute {
    u2 attribute_name_index;
    u4 attribute_length;
    u2 line_number_table_length;
    {  		u2 start_pc;
    		u2 line_number;
    } line_number_table[line_number_table_length];
}
```



* LocalVariableTable

```
LocalVariableTable_attribute {
    u2 attribute_name_index;
    u4 attribute_length;
    u2 local_variable_table_length;
    {  	u2 start_pc;
    	u2 length;
    	u2 name_index;
    	u2 descriptor_index;
    	u2 index;
	} local_variable_table[local_variable_table_length];
}
```



LocalVariableTypeTable

```
LocalVariableTypeTable_attribute {
	u2 attribute_name_index;
	u4 attribute_length;
	u2 local_variable_type_table_length;
	{  	u2 start_pc;
		u2 length;
		u2 name_index;
		u2 signature_index;
		u2 index;
	}local_variable_type_table[local_variable_type_table_length];
}
```



* Deprecated



* RuntimeVisibleAnnotations

  用于保存 Java 语言中的类、字段或方法的运行时的可见注解（Annotations）。

```
RuntimeVisibleAnnotations_attribute {
	u2 attribute_name_index; // 常量池的一个有效索引,字符串
“RuntimeVisibleAnnotations”
	u4 attribute_length;
	u2 num_annotations; // 当前结构表示的运行时可见注解的数量
	annotation annotations[num_annotations]; //annotations[]数组的每个成员的值表示一个程序元素的唯一的运行时可见注解。
}

annotation {
	u2 type_index; // 常量池的一个有效索引,表示一个字段描述符（表示一个注解类型，和当前 annotation 结构表示的注解一致）
	u2 num_element_value_pairs;
	{  	u2 element_name_index;
		element_value value;
	} element_value_pairs[num_element_value_pairs]
}

element_value 结构是一个可辨识联合体（Discriminated Union） ① ，用于表示“元素
-值”的键值对中的值。它被用来描述所有注解（包括 RuntimeVisibleAnnotations、
RuntimeInvisibleAnnotations、RuntimeVisibleParameterAnnotations 和
untimeInvisibleParameterAnnotations

element_value {
	u1 tag;
	union {
        u2 const_value_index;
        {  	u2 type_name_index;
       		u2 const_name_index;
        } enum_const_value;
        u2 class_info_index;
        annotation annotation_value;
        {  	u2 num_values;
        	element_value values[num_values];
        } array_value;
     } value;
}
element_value
tag  值  元素类型
s  String
e  enum constant
c  class
@  annotation type
[  array



RuntimeVisibleParameterAnnotations 属性是一个变长属性，位于 method_info结构的属性表中。用于保存对应方法的参数的所有运行时可见 Java 语言注解。
```





* BootstrapMethods 

如果某个 ClassFile 结构的常量池中有至少一个 CONSTANT_InvokeDynamic_info（§
4.4.10）项，那么这个 ClassFile 结构的属性表中必须有一个明确的 BootstrapMethods 属
性。



### 约束

### 限制 65536



# 类加载

## 创建和加载

Java 虚拟机的启动是通过引导类加载器（Bootstrap Class Loader ）创建一
个初始类（Initial Class）来完成

数组类型没有外部的二进制表示；它们都是由 Java 虚拟机创建，而不是
通过类加载器加载的。



Java 虚拟机支持两种类加载器：Java 虚拟机提供的引导类加载器（Bootstrap Class
Loader）和用户自定义类加载器（User-Defined Class Loader）。每个用户自定义的类加
载器应该是抽象类 ClassLoader 的某个子类的实例。应用程序使用用户自定义类加载器是为了
便于扩展 Java 虚拟机的功能，支持动态加载并创建类。当然，它也可以从用户自定义的数据来源
来获取类的二进制表示并创建类。例如，用户自定义类加载器可以通过网络下载、动态产生或是从
一个加密文件中提取类的信息。



在 Java 虚拟机运行时，类或接口不仅仅是由它的名称来确定，而是由一个值对：二进制名称
和它的定义类加载器共同确定。每个这样的类或接口都归属于独立的运行时包结构（Runtime Package）。类或接口的运行时包结构由包名及类或接口的定义类加载器来决定。

通常使用标识<N，Ld>来表示一个类或接口，这里的 N 表示类或接口的名称，Ld 表示类
或接口的定义类加载器。



##　链接

###　准备

准备（Preparation）阶段的任务是为类或接口的静态字段分配空间，并用默认值初始化这
些字段（§2.3，§2.4）。这个阶段不会执行任何的虚拟机字节码指令。在初始化阶段（§5.5）
会有显式的初始化器来初始化这些静态字段，所以准备阶段不做这些事情。



### 解析

Java 虚拟机指令 anewarray、checkcast、getfield、getstatic、instanceof、invokedynamic、invokeinterface、invokespecial、invokestatic、invokevirtual、ldc、ldc_w、multianewarray、new、putfield 和 putstatic 将符号引用指向运行时常量池。执行上述任何一条指令都需要对它的符号引用的进行解析。
解析（Resolution）是根据运行时常量池的符号引用来动态决定具体的值的过程。





### 初始化

每个类或接口 C，都有一个唯一的初始化锁 LC。如何实现从 C 到 LC 的映射可由 Java 虚拟机
实现自行决定。例如，LC 可以是 C 的 Class 对象，或者是与 Class 对象相关的管程（Monitor）。













































#END