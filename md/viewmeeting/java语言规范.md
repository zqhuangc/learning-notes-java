* 数值表示

十六进制   0x      0X

十进制

八进制   0

二进制   0b   0B





char 可表示一个中文字符   char=‘中’

```
"a"+"b"+"c" == "abc"  一个 String 对象（java 优化）
("a"+"b"+"c").intern() == "abc".intern()
var + "a"  两个
```







类型变量：在类、接口、方法和构造器中用作类型的非限定标识符

可声明为泛化的类型参数   作用域  边界

泛化声明

参数化类型集

```
泛型类或泛型接口的声明定义了一个参数化类型集
参数化类型是形式为 C<T1，...,Tn> 的类或接口， 其中 C 是泛型名，而 <T1，...,Tn> 是表示该泛型的特定参数化形式的类型引元列表

类型引元可以是引用类型或通配符
```





类型变量擦除是其最左边界的擦除

每种其他类型的擦除都是该类型自身



原生类型：引用类型泛型名，数组类型



变量

类型转换







* 访问权限






完全限定名 包名+类/接口名



静态导入   import static typename.identifier

























# [翻车原因](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&scene=21#wechat_redirect)

为什么使用 `YYYY-MM-dd` 格式化 `"2020-12-31"` 时间时，打印的结果是**错误** 的 `"2021-12-31"`呢？

我们打开 https://docs.oracle.com/javase/8/docs/api/java/time/format/DateTimeFormatter.html#patterns 文档，看看 `YYYY` 的定义描述，就非常好理解背后的原因













# END

