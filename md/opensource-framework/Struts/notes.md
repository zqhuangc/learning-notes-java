![1600919687190](..\..\image\struts执行原理-filter-interceptor.png)

回顾Struts2的执行流程图？  
请求->过滤器(web.xml)->拦截器(struts2) Action


#### Struts2动态方法调用
动态方法调用: 通过 url 动态调用 Action 中的方法
​     如果Action中存在多个方法时，我们可以使用!+方法名调用指定方法
默认情况下, Struts 的动态方法调用处于激活状态, 若想禁用该功能, 则可以在 struts.xml 文件中添加如下 constant 元素:
​      <constant name="struts.enable.DynamicMethodInvocation" value="false"/>
实现


注意
​     如果全局和局部有同名的result，那么局部会覆盖全局的result。
​     同一个应用中每次请求Struts2框架都会创建一个新的Action实例。

#### 类型转换
Struts2中为什么要类型转换？
​      HTML表单采集数据  提交表单  Action
​      底层依赖HTTP传递数据，而HTTP协议中 没有 “类型” 的概念. 每一项
​      表单输入只可能是一个字符串或一个字符串数组。因此在服务器端Action
​      中 必须把 String 转换为业务需要的特定的数据类型
Struts2中如何传递请求参数给Action?
​      Struts2框架会将表单的参数以同名的方式设置给对应Action的属性中。
​      该工作主要是由Parameters拦截器做的。而该拦截器中已经自动的实现了
​      String到基本数据类型之间的转换工作。类似于: Beanutils工具。
案例
​      1、注册表单(ServletActionContext先获取参数后自动同名设置获取)
​      2、Action中定义同名属性并提供get和set方法
​      3、检测是否转换


String到基本数据类型的转换是自动的。
String到Date日期类型的转换是有条件的。
默认输入框输入的格式必须是yyyy-MM-dd，其他格式无法转换。


#### Struts2中如何自定义类型转换器？
接口实现类
Struts2中如何配置自定义转换器?
1、自定义转换器继承StrutsTypeConverter
2、重写convertFromString和convertToString方法
3、注册转换器
3.1 在Action所在包中建立 
Action名-conversion.properties
3.2 在3.1文件中添加以下数据
需要转换的字段名=自定义转换器类的权限定名
birthday=cn.itcast.convertor.DateTypeConvertor
总结
以上的转换器注册时候是与Action的名字相耦合的,因此只能在自己的Action中内部使 
用，称之为局部转换器注册方式。如何定义全局类型转换器呢？



#### Struts2中如何自定义全局类型转换器？
实现的接口和继承的类都是相同的，本质上就是配置的方式不同。
实现
1、自定义转换器继承StrutsTypeConverter
2、重写convertFromString和convertToString方法
3、注册转换器
3.1 在项目src目录下建立以下固定文件 
​        xwork-conversion.properties   
3.2 在3.1文件中添加以下数据
​        需要转换的类类型=转换器类的权限定名
​        如:  java.util.Date= cn.itcast.converter.DateConverter
总结
该拦截器负责对错误信息进行拦截器<interceptor name="conversionError“ 
class="org.apache.struts2.interceptor.StrutsConversionErrorInterceptor"/>

### 

### OGNL表达式
OGNL是Object Graphic Navigation Language（对象图导航语言）的缩写，它是一个开源项目。 Struts2框架使用OGNL作为默认的表达式语言。

OGNL表达式之#号
### \#号的作用

      #号主要用于访问访问Map栈信息，不使用#号主要用于访问List(对象栈)信息。
<s:property value="#request.username"/>
<s:property value="#request.userpsw"/>
<s:property value="address"/>    // 获取对象栈信息(默认从栈顶检索)
Struts2的property 标签中value属性值会特意的将其中的值以OGNL表达式的方式进行运行。


%
$


#### UI标签
