# 运行时数据区域

## 线程私有

### 程序计数器

```Java
一块较小的内存，不会发生OutOfMemory(OOM)。当前线程所执行的字节码行号的符号指示器。线程私有，每一个线程（一个内核只会执行一条线程中的指令）都有单独的程序计数器  
在JVM规范中规定，如果线程执行的是非native方法，则程序计数器中保存的是当前需要执行的指令的地址；如果线程执行的是native方法，则程序计数器中的值是undefined。

由于程序计数器中存储的数据所占空间的大小不会随程序的执行而发生改变，因此，对于程序计数器是不会发生内存溢出现象(OutOfMemory)的。
```
### 本地方法栈

```Java
为虚拟机使用到的Native方法服务(java调用非java代码的接口)，
//Sun HotSpot虚拟机中上述两个栈已合二为一
```
### java虚拟机栈

```java
描述Java方法执行的内存模型，每个方法执行时都会创建一个栈帧，用于存储局部变量表、操作数栈、动态链接。方法出口等信息。一般栈都可扩展、

局部变量表（编译时分配，运行时不变）存放编译时期的基本数据类型、对象引用（不等同于对象本身）。64位的long和double占两个局部变量空间，其他占一个

// 栈帧
Java栈中存放的是一个个的栈帧，每个栈帧对应一个被调用的方法，在栈帧中包括局部变量表(Local Variables)、操作数栈(Operand Stack)、指向当前方法所属的类的运行时常量池（运行时常量池的概念在方法区部分会谈到）的引用(Reference to runtime constant pool)、方法返回地址(Return Address)和一些额外的附加信息。
// 工作过程
当线程执行一个方法时，就会随之创建一个对应的栈帧，并将建立的栈帧压栈。当方法执行完毕之后，便会将栈帧出栈。因此可知，线程当前执行的方法所对应的栈帧必定位于Java栈的顶部。

栈帧构成部分：
1. 局部变量表：用来存储方法中的局部变量（包括在方法中声明的非静态变量以及函数形参）。对于基本数据类型的变量，则直接存储它的值，对于引用类型的变量，则存的是指向对象的引用。局部变量表的大小在编译器就可以确定其大小了，因此在程序执行期间局部变量表的大小是不会改变的。

2. 操作数栈：栈最典型的一个应用就是用来对表达式求值。想想一个线程执行方法的过程中，实际上就是不断执行语句的过程，而归根到底就是进行计算的过程。因此可以这么说，程序中的所有计算过程都是在借助于操作数栈来完成的。

3. 指向运行时常量池的引用：因为在方法执行的过程中有可能需要用到类中的常量，所以必须要有一个引用指向运行时常量。

4. 方法返回地址：当一个方法执行完毕之后，要返回之前调用它的地方，因此在栈帧中必须保存一个方法返回地址。

5. 附加消息
```
## 线程共享

### 堆

```java
https://www.tuicool.com/articles/FbI3En 
主要存放对象实例，GC（Garbage Collection）主要区域，细分：新生代（Eden、From Survivor。To Suvivor）、老年代

内存空间只需逻辑连续
当前虚拟机都按可扩展实现（通过-Xmx和-Xms控制）
```
### 方法区

```java
存储虚拟机加载的类信息、常量、静态变量、即时编译器（JIT）编译后的代码等数据。内存空间只需逻辑连续。（若内存回收主要是常量池的回收和对类型的卸载） 本地方法区存在一块特殊的内存区域，叫常量池（Constant Pool）

永久代（JDK8被移除，转为在直接内存中的元数据）：方法区的一种实现，有-XX：MaxPermSize的上限，易遇到内存溢出；J9和JRockit只要不触到进程可用内存上限就没问题，JDK1.7(JDK8)中 字符串常量池从永久代移除，使用 MetaSpace 来保存类加载之后的类信息，字符串常量池也被移动到 JAVA 堆

Native Memory
运行时常量池相比Class文件常量池，多了动态性，运行时也能将常量放进池，因此常用String和intern（）
```

## 堆外内存（直接内存）

这是一块物理内存，专门用于 JVM 和 IO 设备打交道，Java 底层使用 C 语言的 API 调用操作系统与 IO 设备进行交互。  
例如，Java 内存中有一个字节数组，现在调用流将它写入磁盘文件，那么 JVM 首先会将这个字节数组先拷贝一份到堆外内存中，然后调用 C 语言 API 指明将某个连续地址范围的数据写入磁盘。  
读操作也是类似，而 JVM 额外做的拷贝工作也是有意义的，因为 JVM 是基于自动垃圾回收机制运行的，所有内存中的数据会在 GC 时不停的被移动，如果你调用系统 API 告诉操作系统将内存某某位置的内存写入磁盘，而此时发生 GC 移动了该部分数据，GC 结束后操作系统是不是就写错数据了。  
所以，JVM 对于与外围 IO 设备交互的情况下，都会将内存数据复制一份到堆外内存中，然后调用系统 API 间接的写入磁盘，读也是类似的。由于堆外内存不受 GC 管理，所以用完一定得记得释放。  **内存不连续问题**

**JDK1.4 新加入NIO类，引入了一种基于通道和缓冲区的I/O方式，它可以使用Native函数库直接分配堆外内存，然后通过一个存储在java堆中的DirectByteBuffer对象作为这块内存的引用进行操作，避免java堆和Native堆中来回复制数据。** 









# JMM内存模型

主内存
工作内存（缓存）



工作内存从主内存拷贝缓存需要操作的数据，操作完成后将改动的数据写回主内存

指令  Lock