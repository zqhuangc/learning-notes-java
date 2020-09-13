# SpEL

SpelExpressionParser#parseExpression

Expression 

SpEL的几个概念：

表达式（“干什么”）：SpEL的核心，所以表达式语言都是围绕表达式进行的
解析器（“谁来干”）：用于将字符串表达式解析为表达式对象
上下文（“在哪干”）：表达式对象执行的环境，该环境可能定义变量、定义自定义函数、提供类型转换等等
root根对象及活动上下文对象（“对谁干”）：root根对象是默认的活动上下文对象，活动上下文对象表示了当前表达式操作的对象



步骤解释：

按照SpEL支持的语法结构，写出一个expressionStr
准备一个表达式解析器ExpressionParser，调用方法parseExpression()对它进行解析。这一步至少完成了如下三件事：
使用一个专门的断词器Tokenizer，将给定的表达式字符串拆分为Spring可以认可的数据格式
根据断词器处理的操作结果生成相应的语法结构
在这处理过程之中就需要进行表达式的对错检查（语法格式不对要精准报错出来）
将已经处理好后的表达式定义到一个专门的对象Expression里，等待结果
由于表达式内可能存在占位符变量${}，所以还不太适合马上直接getValue()（若不需要解析占位符那就直接getValue()也是可以拿到值的）。所以在计算之前还得设置一个表达式上下文对象`EvaluationContext`（这一步步不是必须的）
替换好占位符内容后，利用表达式对象计算出最终的结果



## ExpressionParser：表达式解析器

```java
// @since 3.0
public interface ExpressionParser {
	// 他俩都是把字符串解析成一个Expression对象~~~~  备注expressionString都是可以被repeated evaluation的
	Expression parseExpression(String expressionString) throws ParseException;
	Expression parseExpression(String expressionString, ParserContext context) throws ParseException;
}
```



#### ParserContext







```java
public class SpelExpressionParser extends TemplateAwareExpressionParser {
	private final SpelParserConfiguration configuration;
	public SpelExpressionParser() {
		this.configuration = new SpelParserConfiguration();
	}
	public SpelExpressionParser(SpelParserConfiguration configuration) {
		Assert.notNull(configuration, "SpelParserConfiguration must not be null");
		this.configuration = configuration;
	}

	// 最终都是委托给了Spring的内部使用的类：InternalSpelExpressionParser--> 内部的SpEL表达式解析器~~~
	public SpelExpression parseRaw(String expressionString) throws ParseException {
		return doParseExpression(expressionString, null);
	}
	// 这里需要注意：因为是new的，所以每次都是一个新对象，所以它是线程安全的~
	@Override
	protected SpelExpression doParseExpression(String expressionString, @Nullable ParserContext context) throws ParseException {
		return new InternalSpelExpressionParser(this.configuration).doParseExpression(expressionString, context);
	}

}
```





### Expression

**示的是表达式对象。**`能够根据上下文对象对自身进行计算的表达式。`

```java
// @since 3.0   表达式计算的通用抽象。  该接口提供的方法非常非常之多~~~ 但不要怕大部分都是重载的~~~
public interface Expression {
	
	// 返回原始表达式的字符串~~~
	String getExpressionString();

	// 使用一个默认的标准的context执行计算~~~
	@Nullable
	Object getValue() throws EvaluationException;
	// SpEL内部帮你转换~~~  使用的是默认的context
	@Nullable
	<T> T getValue(@Nullable Class<T> desiredResultType) throws EvaluationException;

	// 根据指定的根对象计算此表达式
	@Nullable
	Object getValue(Object rootObject) throws EvaluationException;
	@Nullable
	<T> T getValue(Object rootObject, @Nullable Class<T> desiredResultType) throws EvaluationException;

	// 根据指定的上下文:EvaluationContext来计算值~~~  rootObject：跟对象
	@Nullable
	Object getValue(EvaluationContext context) throws EvaluationException;
	// 以rootObject作为表达式的root对象来计算表达式的值。 
	// root对象：比如parser.parseExpression("name").getValue(person);相当于去person里拿到name属性的值。这个person就叫root对象
	@Nullable
	Object getValue(EvaluationContext context, Object rootObject) throws EvaluationException;
	@Nullable
	<T> T getValue(EvaluationContext context, @Nullable Class<T> desiredResultType) throws EvaluationException;
	@Nullable
	<T> T getValue(EvaluationContext context, Object rootObject, @Nullable Class<T> desiredResultType)
			throws EvaluationException;

	// 返回可传递给@link setvalue的最一般类型
	@Nullable
	Class<?> getValueType() throws EvaluationException;
	@Nullable
	Class<?> getValueType(Object rootObject) throws EvaluationException;
	@Nullable
	Class<?> getValueType(EvaluationContext context) throws EvaluationException;
	@Nullable
	Class<?> getValueType(EvaluationContext context, Object rootObject) throws EvaluationException;


	@Nullable
	TypeDescriptor getValueTypeDescriptor() throws EvaluationException;
	@Nullable
	TypeDescriptor getValueTypeDescriptor(Object rootObject) throws EvaluationException;
	@Nullable
	TypeDescriptor getValueTypeDescriptor(EvaluationContext context) throws EvaluationException;
	@Nullable
	TypeDescriptor getValueTypeDescriptor(EvaluationContext context, Object rootObject) throws EvaluationException;

	// 确定是否可以写入表达式，即可以调用setValue（）
	boolean isWritable(Object rootObject) throws EvaluationException;
	boolean isWritable(EvaluationContext context) throws EvaluationException;
	boolean isWritable(EvaluationContext context, Object rootObject) throws EvaluationException;

	// 在提供的上下文中将此表达式设置为提供的值。
	void setValue(Object rootObject, @Nullable Object value) throws EvaluationException;
	void setValue(EvaluationContext context, @Nullable Object value) throws EvaluationException;
	void setValue(EvaluationContext context, Object rootObject, @Nullable Object value) throws EvaluationException;

}
```



#### SpelExpression

```java
public class SpelExpression implements Expression {

	// 在编译表达式之前解释该表达式的次数。
	private static final int INTERPRETED_COUNT_THRESHOLD = 100;
	// 放弃前尝试编译表达式的次数
	private static final int FAILED_ATTEMPTS_THRESHOLD = 100;

	private final String expression;
	// AST：抽象语法树~
	private final SpelNodeImpl ast; // SpelNodeImpl的实现类非常非常之多~~~
	private final SpelParserConfiguration configuration;

	@Nullable
	private EvaluationContext evaluationContext; // 如果没有指定，就会用默认的上下文 new StandardEvaluationContext()

	// 如果该表达式已经被编译了，就会放在这里 @since 4.1  Spring内部并没有它的实现类  尴尬~~~编译是要交给我们自己实现？？？
	@Nullable
	private CompiledExpression compiledAst;

	// 表达式被解释的次数-达到某个限制时可以触发编译
	private volatile int interpretedCount = 0;
	// 编译尝试和失败的次数——使我们最终放弃了在似乎不可能编译时尝试编译它的尝试。
	private volatile int failedAttempts = 0;
	
	// 唯一构造函数~
	public SpelExpression(String expression, SpelNodeImpl ast, SpelParserConfiguration configuration) {
		this.expression = expression;
		this.ast = ast;
		this.configuration = configuration;
	}
	...
	// 若没有指定，这里会使用默认的StandardEvaluationContext上下文~
	public EvaluationContext getEvaluationContext() {
		if (this.evaluationContext == null) {
			this.evaluationContext = new StandardEvaluationContext();
		}
		return this.evaluationContext;
	}
	...
	@Override
	@Nullable
	public Object getValue() throws EvaluationException {
		// 如果已经被编译过，就直接从编译后的里getValue即可~~~~
		if (this.compiledAst != null) {
			try {
				EvaluationContext context = getEvaluationContext();
				return this.compiledAst.getValue(context.getRootObject().getValue(), context);
			} catch (Throwable ex) {
				// If running in mixed mode, revert to interpreted
				if (this.configuration.getCompilerMode() == SpelCompilerMode.MIXED) {
					this.interpretedCount = 0;
					this.compiledAst = null;
				} else {
					throw new SpelEvaluationException(ex, SpelMessage.EXCEPTION_RUNNING_COMPILED_EXPRESSION);
				}
			}
		}

		ExpressionState expressionState = new ExpressionState(getEvaluationContext(), this.configuration);
		// 比如此处SeEl是加法+，所以ast为：OpPlus语法树去处理的~~~
		Object result = this.ast.getValue(expressionState);
	
		// 检查是否需要编译它~~~
		checkCompile(expressionState);
		return result;
	}
	... // 备注：数据转换都是EvaluationContext.getTypeConverter() 来进行转换
	// 注意：此处的TypeConverter为`org.springframework.expression`的  只有一个实现类：StandardTypeConverter
	// 它内部都是委托给ConversionService去做的，具体是`DefaultConversionService`去做的~

	@Override
	@Nullable
	public Class<?> getValueType() throws EvaluationException {
		return getValueType(getEvaluationContext());
	}
	@Override
	@Nullable
	public Class<?> getValueType(Object rootObject) throws EvaluationException {
		return getValueType(getEvaluationContext(), rootObject);
	}
	@Override
	@Nullable
	public Class<?> getValueType(EvaluationContext context, Object rootObject) throws EvaluationException {
		ExpressionState expressionState = new ExpressionState(context, toTypedValue(rootObject), this.configuration);
		TypeDescriptor typeDescriptor = this.ast.getValueInternal(expressionState).getTypeDescriptor();
		return (typeDescriptor != null ? typeDescriptor.getType() : null);
	}
	...
	@Override
	public boolean isWritable(Object rootObject) throws EvaluationException {
		return this.ast.isWritable(new ExpressionState(getEvaluationContext(), toTypedValue(rootObject), this.configuration));
	}
	...
	@Override
	public void setValue(Object rootObject, @Nullable Object value) throws EvaluationException {
		this.ast.setValue(new ExpressionState(getEvaluationContext(), toTypedValue(rootObject), this.configuration), value);
	}
	...
}
```



#### `EvaluationContext`：评估/计算的上下文



### SpEL中的PropertyAccessor（重要）





































