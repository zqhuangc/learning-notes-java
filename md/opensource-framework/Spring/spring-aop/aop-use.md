

通过类匹配模式串声明切点，within()函数定义的连接点是针对目标类而言的，而非针对运行期对象的类型而言，这一点和execution()是相同的。

但是within()和execution()函数不同的是，within()所指定的连接点最小范围只能是类，而execution()所指定的连接点可以大到包，小到方法入参。 所以从某种意义上讲，execution()函数功能涵盖了within()函数的功能



```
文档里面是这么说的：

@target: Limits matching to join points (the execution of methods when using Spring AOP) where the class of the executing object has an annotation of the given type.

@within: Limits matching to join points within types that have the given annotation (the execution of methods declared in types with the given annotation when using Spring AOP).

看不出来啥区别，文档后面还举了个例子：

@target(org.springframework.transaction.annotation.Transactional)

Any join point (method execution only in Spring AOP) where the target object has a @Transactional annotation

@within(org.springframework.transaction.annotation.Transactional)

Any join point (method execution only in Spring AOP) where the declared type of the target object has an @Transactional annotation
```

stackoverflow上有老外提到了这个问题：https://stackoverflow.com/questions/52992365/spring-creates-proxy-for-wrong-classes-when-using-aop-class-level-annotation/53452483

When using spring AOP with class level annotations, spring context.getBean seems to always create and return a proxy or interceptor for every class, wether they have the annotation or not.

This behavior is only for class level annotation. For method level annotations, or execution pointcuts, if there is no need for interception, getBean returns a POJO.

对于这个问题，答案可以参考下aspectj的文档：https://www.eclipse.org/aspectj/doc/released/adk15notebook/annotations-pointcuts-and-advice.html#runtime-type-matching-and-context-exposure

The this(), target(), and args() pointcut designators allow matching based on the runtime type of an object, as opposed to the statically declared type. In AspectJ 5, these designators are supplemented with three new designators : @this() (read, "this annotation"), @target(), and @args().

The forms of @this() and @target() that take a single annotation name are analogous to their counterparts that take a single type name. They match at join points where the object bound to this (or target, respectively) has an annotation of the specified type.

@within(Foo)

Matches any join point where the executing code is defined within a type which has an annotation of type Foo.

结合之前的文档，我们可以得出结论，**@target是匹配的对象的运行时类型，而@within是匹配的定义正在执行的代码(方法)的类，一个方法定义在哪个类中这个是明确可以知道的，但是对象的运行时类型只有在运行时才能知道，所以@target才给所有的bean都生成了代理类**，如果无法生成代理类就会出问题了，比如：



**.@target匹配的是执行方法的对象的class上是否有注解，@within匹配的是定义方法的对象的class上是否有注解，由于运行时多态的特性，执行方法的对象和定义方法的对象可能不是同一个对象。**

**- 2.定义方法的类是可以提前知道的，这个是静态的，但是执行方法的类是无法提前知道的，是运行时动态的，因此，在@target的场景下，spring aop给所有的bean都生成了代理类，如果无法生成代理就会报错。比如：cglib遇到final的时候。**

**- 3.@target一定要慎用，可以用execution(\*  (@cn.javass..Secure *).*(..))来代替。**



### **@within和@annotation的区别：**

@within 对象级别

@annotation 方法级别