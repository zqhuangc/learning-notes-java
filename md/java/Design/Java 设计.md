# [Java SE Technologies](https://www.oracle.com/technetwork/java/javase/tech/index.html)

## Java 接口设计

### 通用设计 - 类/接口名

+ 模式：（形容词）+名词
  - 举例：
    - 单名词：java.lang.String
    - 双名词：java.util.ArrayList
    -  形容词+名词：java.util.LinkedList

### 通用设计 - 可访问性

#### 四种修饰符

- public：开放 API 使用场景
  - 举例：java.lang.String
- default(默认）：仅在当前 package 下使用，属于私有 API
  - 举例：java.io.FileSystem
- protected：不能用于修饰最外层 class
- private：不能用于修饰最外层 class

### 通用设计 - 可继承性

+ final：final 不具备继承性，仅⽤用于实现类，不能与 abstract 关键字同时修饰类
  - 举例：java.lang.String

+ 非 final：最常见/默认的设计⼿手段，可继承性依赖于可访问性
  - 举例：java.io.FileSystem

### 具体类设计

#### 常见场景

- 功能组件
  - HashMap
- 接口/抽象类实现
  - HashMap <- AbstractMap <- Map
- 数据对象
  - POJO
- 工具辅助
  - *Utils
  - ViewHelper
  - Helper

#### 命名模式
* 前缀：“Default”、“Generic”、“Common”、“Basic”
* 后缀：“Impl”



### 抽象类设计

#### 常见场景

- 接口通用实现（模板模式）
  - AbstractList
  - AbstractSet
  - AbstractMap
- 状态/行为继承

#### 常见模式

* 抽象程度介于类与接口之间（Java 8+ 可完全由接⼝口代替）

  - 以 “Abstract” 或 “Base” 类名前缀

    - java.util.AbstractCollection

    - javax.sql.rowset.BaseRowSet

### 接口设计

#### 常见场景
* 上下游系统（组件）通讯契约
  - API
  - RPC
* 常量定义

#### 常见模式

* 无状态（Stateless）
* 完全抽象（ < Java 8 ）
* 局部抽象（ Java 8+ ）
* 单一抽象（ Java 8 函数式接口）
  - Serializable
  - Cloneable
  - AutoCloseable
  - EventListener

### 内置类设计
#### 常见场景

* 临时数据存储类：java.lang.ThreadLocal.ThreadLocalMap

* 特殊用途的 API 实现：java.util.Collections.UnmodifiableCollection
* Builder 模式（接口）：java.util.stream.Stream.Builder



## Java枚举设计

- 枚举(enum) 实际是 final class
- 枚举(enum) 成员修饰符为 public static final
- `values()` 是 Java 编译器做的**字节码提升**
  javap --v 全类名



### 枚举类

* 场景：Java 枚举（enum）引⼊入之前的模拟枚举实现类
* 模式：
  - 成员用常亮表示，并且类型为当前类型
  - 常用关键字 final 修饰类
  - 非 public 构造器器

### Java 枚举

+ 基本特性
  - 类结构（强类型）
  - 继承 java.lang.Enum
  - 不可显示地继承和被继承

### “枚举类” V.S Java 枚举

![](https://ws1.sinaimg.cn/large/006xzusPly1g5tgbhgb07j30gy07pq3j.jpg)

成员设计

构造器设计

 方法设计

 

## Java 泛型设计

Java 泛型属于编译时处理，运行时擦写。

spring   ResolvableType     字节码处理



MethodHandle

invokeDynamic

### 泛型使用场景

* 编译时强类型检查
* 避免类型强转
* 实现通用算法

### 泛型类型
A generic type is a generic class or interface that is parameterized over types.

* 调用泛型类型
* 实例化泛型
* Java 7 Diamond 语法（泛型类型省略）
* 类型参数命名约定 

### 类型参数命名约定

* E：表示集合元素（Element）
* V：表示数值（Value）
* K：表示键（Key）
* T：表示类型

### 泛型有界类型参数

* 单界限
* 多界限
* 泛型方法和有界类型参数

### 泛型通配类型

* 上界通配类型
* 下界通配类型

### 泛型类型擦写
Generics were introduced to the Java language to provide tighter type checks at compile time and to support generic programming. To implement generics, the Java compiler applies type erasure to:

- Replace all type parameters in generic types with their bounds or `Object` if the type parameters are unbounded.
  The produced bytecode, therefore, contains only ordinary classes, interfaces, and methods.
- Insert type casts if necessary to preserve type safety.
- Generate bridge methods to preserve polymorphism in extended generic types.
  Type erasure ensures that no new classes are created for parameterized types; consequently, generics incur no
  runtime overhead.





## Java 方法设计

#### 方法名设计（英语，动名形副组合）

- 单元：一个类或者一组类（组件）

  - 类采用名词结构
    - 动词过去式+名词
      - ContextRefreshedEvent
    - 动词ing + 名词
      - InitializingBean
    - 形容词 + 名词
      - ConfigurableApplicationContext

- 执行：某个方法

  - 方法命名：动词

    - execute
    - callback
    - run

  - 方法参数：名词

  - 异常：

    - 根（顶层）异常

      - Throwable
        - checked 类型：Exception
        - unchecked类型：RuntimeException
        - 不常见：Error

    - Java 1.4 `java.lang.StackTraceElement`

      - 添加异常原因（cause）

        - 反模式：吞掉某个异常

        - 性能：注意 `fillInStackTrace()`

          方法的开销，避免异常栈调用深度

          - 方法一：JVM 参数控制栈深度（物理屏蔽）
          - 方法二：logback 日志框架控制堆栈输出深度（逻辑屏蔽）

#### 方法返回类型设计 （需要抽象，越具体，越难通用）

* 原则
  1. 返回值需要抽象，越具体，越难通用，除Object
  2. 若是集合只读，确保只读unmodifiableXXX，不要 []
  3. 非只读，返回快照（推荐用ArrayList），保证原生的

#### 方法参数类型、名称、数量的设计





## Java 函数式设计

### 理解 @FunctionalInterface

用于函数式接⼝口类型声明的信息注解类型，这些接口的实例被 Lambda 表示式、方法引用或构造器器引用创建。函数式接口只能有一个抽象方法，并排除接口默认方法以及声明中覆盖 Object 的公开方法的统计。同时，@FunctionalInterface 不能标注在注解、类以及枚举上。如果违背以上规则，那么接口不能视为函数式接口，当标注@FunctionalInterface 后，会引起编译错误。

不过，如果任一接口满足以上函数式接口的要求，无论接口声明中是否标注@FunctionalInterface，均能被编译器器视作函数式接口。

### 函数式接口类型

* 提供类型 – Supplier<T>
* 消费类型 – Consumer<T>
* 转换类型 – Function<T,R>
*  断定类型 – Predicate<T>
* 隐藏类型 - Action

### 函数式接口设计

#### Supplier<T> 接口定义
* 基本特点：只出不进
* 编程范式：作为方法/构造参数、方法返回值
* 使用场景：数据来源，代码替代接口

#### Consumer<T> 接口设计

* 基本特点：只进不出
* 编程范式：作为方法/构造参数
* 使用场景：执行 Callback

#### Function<T,R> 接口设计

* 基本特点：有进有出
* 编程范式：作为方法/构造参数
* 使用场景：类型转换、业务处理理等

#### Predicate<T> 接口设计

* 基本特点：boolean 类型判断
* 编程范式：作为方法/构造参数
* 使用场景：过滤、对象比较等



###函数式在框架中的运用
### Stream API 设计

#### 基本操作
* 转换：Stream#map(Function)
* 过滤：Stream#filter(Predicate)
* 排序：
  - Stream#sorted()
  - Stream#sorted(Comparator)

#### 类型
* 串行 Stream（默认类型）
* 并行 Stream
* 转换并行 Stream：Stream#parallel()
* 是否为并行 Stream：Stream#isParallel()

####  高级操作
* Collect 操作
* 分组操作
* 聚合操作
* flatMap 操作
* reduce 操作



### 匿名内置类

基本特性：

- 无名称类
- 声明位置(执行模块)：
  - static block
  - 实例block
  - 方法
  - 构造器
- 并非特殊的类结构
  - 类全名称：${package}.${declared_class}.${num}

### 函数式接口

基本特性：

- 所有的函数式接口都引用一段执行代码
- 函数式接口没有固定的类型，固定模式（ SCFP = Supplier + Consumer + Function + Predicate） + Action
- 利用方法引用来实现模式匹配
