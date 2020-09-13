# [Javassist之字节码读取](https://zhuanlan.zhihu.com/p/38106338)

Javassist是一个用于处理 Java 字节码的类库。Java字节码是一个以二进制文件进行存储的class文件。每一个class文件都包含一个Javal类或者是接口。

Javassist.Ctclass是一个对class文件的一个抽象表示形式。一个CtClass(编译时类)对象是一个处理class文件的句柄。下面这段程序是一个非常简单的例子：

```java
ClassPool pool = ClassPool.getDefault();
CtClass cc = pool.get("test.Rectangle");
cc.setSuperclass(pool.get("test.Point"));
cc.writeFile();
```

这段程序首先获取一个ClassPool对象，它控制着Javassist对字节码的修改。ClassPool对象是一个表示class文件的CtClass对象的容器。它读取一个类文件，用于构造CtClass对象，并记录构建的对象以响应后面的访问。

要去改变一个类的定义，开发者必须要先从ClassPool对象中，获得一个表示该类的CtClass对象的引用。该类可以通过ClassPool的get()方法获取。在上面的展示的程序示例中，调用ClassPool的get()方法，返回了一个用于表示test.Rectangle类的CtClass对象cc。ClassPool.getDefault()方法的作用是扫描默认系统路径的类，返回一个ClassPool对象。

站在源码的角度，ClassPool是一个使用类的名称当作key，对应的CtClass为value的hash表。ClassPool的get()方法根据传入的类名参数，在hash表中查询到该类名对应的CtClass对象并返回。如果没有查询到该类名对应的CtClass对象，get()方法将会读取类文件来构造一个新的CtClass对象，这个对象会被添加到hash表中，然后返回。

从ClassPool对象中获取的CtClass对象是可以被改变的（关于如何修改CtClass的详细说明将会在下文展示）。在上面的例子中，test.Rectangle的父类被改变成了test.Point。这个改变在最后调用CtClass()的writeFile()方法时会被写入源class文件。

writeFile()方法将CtClass对象转变成一个class文件，并写入本地磁盘。Javassist也提供了直接获取修改过的字节码的方法。调用toBytecode()方法，直接获取字节码：

```java
byte[] b = cc.toBytecode();
```

也可以直接读取CtClass：

```java
Class clazz = cc.toClass();
```

toClass()方法会请求当前线程的上下文类加载器去加载CtClass表示的类文件。它返回一个表示被加载类的java.lang.Class对象。更加详细的说明，请看下面的部分。

### 定义一个新的类

要重新定义一个类，必须调用ClassPool的makeClass()方法。

```java
ClassPool pool = ClassPool.getDefault();
CtClass cc = pool.makeClass("Point");
```

这段代码定义了一个Point类，它没有包含任何的属性或方法。Point类可以通过调用CtNewMethod的工厂方法来创建方法，并使用CtClass的addMethod()将该方法添加到Point类中。

makeClass()无法创建一个新的接口，但ClassPool的makeInterface()方法可以。接口的方法可以通过CtNewMethod的abstractMethod()方法创建。注意，一个接口方法是一个抽象的方法。

### 类的冻结

如果一个CtClass对象通过writeFile()，toClass()，或者toBytecode()转换到一个类文件，Javassist将会冻结CtClass对象。对CtClass对象的进一步改变将不被允许。因为JVM不会允许重新加载一个类，所以当开发者尝试去改变一个已经加载的类时，将会发出警告。

一个冻结的CtClass可以被解除，解冻之后就可以允许对类的定义进行修改。例如下面这个例子：

```java
CtClasss cc = ...;
    :
cc.writeFile();
cc.defrost();
cc.setSuperclass(...);    // OK since the class is not frozen.
```

在调用defrost()方法之后，CtClass对象就可以再次被修改。

如果ClassPool.doPruning被设置成true，在Javassist冻结CtClass对象的时候，Javassist将会对CtClass对象包含的数据结构进行精简。通过修剪丢弃对象不需要的属性（attribute_info结构）来减少内存的消耗。例如，丢弃Code_attribute结构（方法体）。因此，当CtClass对象被精简之后，方法的字节码除了方法名，签名，以及注解之外将不能被访问。精简的CtClass对象无法再次被解冻。ClassPool.doPruning的默认值是false。

对于一些并不想被精简的特殊CtClass，必须提前调用stopPruning()：

```java
CtClasss cc = ...;
cc.stopPruning(true);
    :
cc.writeFile();                             // convert to a class file.
// cc is not pruned.
```

CtClass对象cc没有被精简。因此它可以在writeFile()被调用之后解冻。

**注意：**在进行debug的时候，可能需要暂时停止精简或者冻结或者将修改过的类文件写入磁盘。debugWriteFile()方法可以方便的实现此功能。它可以停止精简，写类文件，解冻，同时再次精简。

### 类扫描路径

由静态方法ClassPool.getDefault() 返回的默认 ClassPool 与 JVM 有相同的扫描路径。如果程序在 JBoss 或者 Tomcat 之类的 web 应用程序中，ClassPoll 对象可能无法找到用户的类，因为这些 Web 应用服务器使用多个类加载器以及系统类加载器。在这种情况下，就必须注册额外的类扫描路径到 ClassPool 中。可以使用如下方式进行注册：

```java
pool.insertClassPath(new ClassClassPath(this.getClass()));
```

这行代码注册了一类路径，它被用来加载this所指代的对象的类。this.getClass()参数可以被替换成任何一个类对象。用于加载由已经注册的该类对象表示的类的类路径。

你可以注册一个目录名称来作为类扫描路径。例如，下面的代码将/usr/local/javalib添加到类扫描路径：

```java
ClassPool pool = ClassPool.getDefault();
pool.insertClassPath("/usr/local/javalib");
```

用户可以添加的扫描路径不仅仅是目录，也可以是一个URL：

```java
ClassPool pool = ClassPool.getDefault();
ClassPath cp = new URLClassPath("www.javassist.org", 80, "/java/", "org.javassist.");
pool.insertClassPath(cp);
```

这段代码将"[http://www.javassist.org:80/java/](https://link.zhihu.com/?target=http%3A//www.javassist.org/java/)"添加到类扫描路径。这个URL仅仅被用来扫描属于 org.javassist 包的类。例如，想要加载一个 org.javassist.test.Main，它的类文件可以这样获得：

```java
http://www.javassist.org:80/java/org/javassist/test/Main.class
```

此外，你可以直接传入一个字节数组到 ClassPool 对象中，由这个数组构造出一个 CtClass 对象。可以使用ByteArrayClassPath 来做到。例如：

```java
ClassPool cp = ClassPool.getDefault();
byte[] b = a byte array;
String name = class name;
cp.insertClassPath(new ByteArrayClassPath(name, b));
CtClass cc = cp.get(name);
```

所获得的 CtClass 对象表示由b所引用的类文件定义的类。如果调用了get()方法，ClassPool会从给定的ByteArrayClassPath 中读取一个类文件，传入给get()的类名参数与所指定的类名相同。

如果你不知道完整的类名，那么你可以使用ClassPool的makeClass()：

```java
ClassPool cp = ClassPool.getDefault();
InputStream ins = an input stream for reading a class file;
CtClass cc = cp.makeClass(ins);
```

makeClass() 方法返回一个由所给的输入流构造的 CtClass 对象。你可以使用 makeClass() 将类文件直接传入到 ClassPool 对象中。如果扫描路径包含很大的 jar 文件，这可能会提升性能。因为 ClassPool 对象只会在需要的时候才会读取类文件，所以它可能会因为每一个类文件而重复扫描整个 jar 文件。makeClass() 可以被用来优化重复扫描的情况。由 makeClass() 构造的 CtClass 将一直被 ClassPool 对象所持有，所对应的类文件也将不会再被读取。