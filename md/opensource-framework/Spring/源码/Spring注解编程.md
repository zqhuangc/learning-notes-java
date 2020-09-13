

# AutowireCandidateResolver

## @Qualifier--- 按类别批量依赖注入

### QualifierAnnotationAutowireCandidateResolver



```java
// @since 2.5
public class QualifierAnnotationAutowireCandidateResolver extends GenericTypeAwareAutowireCandidateResolver {
	// 是个List，可以知道它不仅仅只支持org.springframework.beans.factory.annotation.Qualifier
	private final Set<Class<? extends Annotation>> qualifierTypes = new LinkedHashSet<>(2);
	private Class<? extends Annotation> valueAnnotationType = Value.class;


	// 空构造：默认支持的是@Qualifier以及JSR330标准的@Qualifier
	public QualifierAnnotationAutowireCandidateResolver() {
		this.qualifierTypes.add(Qualifier.class);
		try {
			this.qualifierTypes.add((Class<? extends Annotation>) ClassUtils.forName("javax.inject.Qualifier", QualifierAnnotationAutowireCandidateResolver.class.getClassLoader()));
		} catch (ClassNotFoundException ex) {
			// JSR-330 API not available - simply skip.
		}
	}
	
	// 非空构造：可自己额外指定注解类型
	// 注意：如果通过构造函数指定qualifierType，上面两种就不支持了，因此不建议使用
	// 而建议使用它提供的addQualifierType() 来添加~~~
	public QualifierAnnotationAutowireCandidateResolver(Class<? extends Annotation> qualifierType) {
	... // 省略add/set方法	

	// 这是个最重要的接口方法~~~  判断所提供的Bean-->BeanDefinitionHolder 是否是候选的
	// （返回true表示此Bean符合条件）
	@Override
	public boolean isAutowireCandidate(BeanDefinitionHolder bdHolder, DependencyDescriptor descriptor) {
		// 1、先看父类：bean定义是否允许依赖注入、泛型类型是否匹配
		boolean match = super.isAutowireCandidate(bdHolder, descriptor);
		// 2、若都满足就继续判断@Qualifier注解~~~~
		if (match) {
			// 3、看看标注的@Qualifier注解和候选Bean是否匹配~~~（本处的核心逻辑）
			// descriptor 一般封装的是属性写方法的参数，即方法参数上的注解
			match = checkQualifiers(bdHolder, descriptor.getAnnotations());
			// 4、若Field/方法参数匹配，会继续去看看参数所在的方法Method的情况
			// 若是构造函数/返回void。 进一步校验标注在构造函数/方法上的@Qualifier限定符是否匹配
		if (match) {
				MethodParameter methodParam = descriptor.getMethodParameter();
				// 若是Field，methodParam就是null  所以这里是需要判空的
				if (methodParam != null) {
					Method method = methodParam.getMethod();
					// method == null表示构造函数 void.class表示方法返回void
					if (method == null || void.class == method.getReturnType()) {
						// 注意methodParam.getMethodAnnotations()方法是可能返回空的
						// 毕竟构造方法/普通方法上不一定会标注@Qualifier等注解呀~~~~
						// 同时警示我们：方法上的@Qualifier注解可不要乱标
						match = checkQualifiers(bdHolder, methodParam.getMethodAnnotations());
					}
				}
			}
		}
		return match;
	}
        //checkQualifiers
	...
}
```



qualifierTypes是支持调用者自己指定的（默认只支持@Qualifier类型）
只有类型匹配、Bean定义匹配、泛型匹配等全部Ok了，才会使用@Qualifier去更加精确的匹配
descriptor.getAnnotations()的逻辑是：
- 如果DependencyDescriptor描述的是字段（Field），那就去字段里拿注解们
- 若描述的是方法参数（MethodParameter），那就返回的是方法参数的注解
- 步骤3的match = true表示Field/方法参数上的限定符是匹配的





说明：能走到`isAutowireCandidate()`方法里来，那它肯定是标注了`@Autowired`注解的（才能被`AutowiredAnnotationBeanPostProcessor`后置处理），所以`descriptor.getAnnotations()`返回的数组长度至少为1



##### `checkQualifiers()`方法：

```
// 将给定的限定符注释与候选bean定义匹配。命名中你发现：这里是负数形式，表示多个注解一起匹配
	// 此处指的限定符，显然默认情况下只有@Qualifier注解
	protected boolean checkQualifiers(BeanDefinitionHolder bdHolder, Annotation[] annotationsToSearch) {
		// 很多人疑问为何没标注注解返回的还是true？
		// 请参照上面我的解释：methodParam.getMethodAnnotations()方法是可能返回空的，so...可以理解了吧
		if (ObjectUtils.isEmpty(annotationsToSearch)) {
			return true;
		}
		SimpleTypeConverter typeConverter = new SimpleTypeConverter();

		// 遍历每个注解（一般有@Autowired+@Qualifier两个注解）
		// 本文示例的两个注解：@Autowired+@LoadBalanced两个注解~~~（@LoadBalanced上标注有@Qualifier）
		for (Annotation annotation : annotationsToSearch) {
			Class<? extends Annotation> type = annotation.annotationType();
			boolean checkMeta = true; // 是否去检查元注解
			boolean fallbackToMeta = false;

			// isQualifier方法逻辑见下面：是否是限定注解（默认的/开发自己指定的）
			// 本文的org.springframework.cloud.client.loadbalancer.LoadBalanced是返回true的
			if (isQualifier(type)) {
				// checkQualifier：检查当前的注解限定符是否匹配
				if (!checkQualifier(bdHolder, annotation, typeConverter)) {
					fallbackToMeta = true; // 没匹配上。那就fallback到Meta去吧
				} else {
					checkMeta = false; // 匹配上了，就没必要校验元数据了喽~~~
				}
			}

			// 开始检查元数据（如果上面匹配上了，就不需要检查元数据了）
			// 比如说@Autowired注解/其它自定义的注解（反正就是未匹配上的），就会进来一个个检查元数据
			// 什么时候会到checkMeta里来：如@A上标注有@Qualifier。@B上标注有@A。这个时候限定符是@B的话会fallback过来
			if (checkMeta) {
				boolean foundMeta = false;
				// type.getAnnotations()结果为元注解们：@Documented、@Retention、@Target等等
				for (Annotation metaAnn : type.getAnnotations()) {
					Class<? extends Annotation> metaType = metaAnn.annotationType();
					if (isQualifier(metaType)) {
						foundMeta = true; // 只要进来了 就标注找到了，标记为true表示从元注解中找到了
						// Only accept fallback match if @Qualifier annotation has a value...
						// Otherwise it is just a marker for a custom qualifier annotation.
						// fallback=true(是限定符但是没匹配上才为true)但没有valeu值
						// 或者根本就没有匹配上，那不好意思，直接return false~
						if ((fallbackToMeta && StringUtils.isEmpty(AnnotationUtils.getValue(metaAnn))) || !checkQualifier(bdHolder, metaAnn, typeConverter)) {
							return false;
						}
					}
				}
				// fallbackToMeta =true你都没有找到匹配的，就返回false的
				if (fallbackToMeta && !foundMeta) {
					return false;
				}
			}
		}
		// 相当于：只有所有的注解都木有返回false，才会认为这个Bean是合法的~~~
		return true;
	}

	// 判断一个类型是否是限定注解   qualifierTypes：表示我所有支持的限定符
	// 本文的关键在于下面这个判断语句：类型就是限定符的类型 or @Qualifier标注在了此注解上（isAnnotationPresent）
	protected boolean isQualifier(Class<? extends Annotation> annotationType) {
		for (Class<? extends Annotation> qualifierType : this.qualifierTypes) {
			// 类型就是限定符的类型 or @Qualifier标注在了此注解上（isAnnotationPresent）
			if (annotationType.equals(qualifierType) || annotationType.isAnnotationPresent(qualifierType)) {
				return true;
			}
		}
		return false;
}

/**-------------------------------------------------*/
/**----------------checkQualifier-------------------*/
/**-------------------------------------------------*/
// 检查某一个注解限定符，是否匹配当前的Bean
	protected boolean checkQualifier(BeanDefinitionHolder bdHolder, Annotation annotation, TypeConverter typeConverter) {
		// type：注解类型 bd：当前Bean的RootBeanDefinition 
		Class<? extends Annotation> type = annotation.annotationType();		
		RootBeanDefinition bd = (RootBeanDefinition) bdHolder.getBeanDefinition();
	
		// ========下面是匹配的关键步骤=========
		// 1、Bean定义信息的qualifiers字段一般都无值了（XML时代的配置除外）
		// 长名称不行再拿短名称去试了一把。显然此处 qualifier还是为null的
		AutowireCandidateQualifier qualifier = bd.getQualifier(type.getName());
		if (qualifier == null) {
			qualifier = bd.getQualifier(ClassUtils.getShortName(type));
		}
		
		//这里才是真真有料的地方~~~请认真看步骤
		if (qualifier == null) {
			// First, check annotation on qualified element, if any
			// 1、词方法是从bd标签里拿这个类型的注解声明，非XML配置时代此处targetAnnotation 为null
			Annotation targetAnnotation = getQualifiedElementAnnotation(bd, type);
			// Then, check annotation on factory method, if applicable
			// 2、若为null。去工厂方法里拿这个类型的注解。这方法里标注了两个注解@Bean和@LoadBalanced，所以此时targetAnnotation就不再为null了~~
			if (targetAnnotation == null) {
				targetAnnotation = getFactoryMethodAnnotation(bd, type);
			}

			// 若本类木有，还会去父类去找一趟
			if (targetAnnotation == null) {
				RootBeanDefinition dbd = getResolvedDecoratedDefinition(bd);
				if (dbd != null) {
					targetAnnotation = getFactoryMethodAnnotation(dbd, type);
				}
			}


			// 若xml、工厂方法、父里都还没找到此方法。那好家伙，回退到还去类本身上去看
			// 也就是说，如果@LoadBalanced标注在RestTemplate上，也是阔仪的
			if (targetAnnotation == null) {
				// Look for matching annotation on the target class
				...
			}
		
			// 找到了，并且当且仅当就是这个注解的时候，就return true了~
			// Tips：这里使用的是equals，所以即使目标的和Bean都标注了@Qualifier属性，value值相同才行哟~~~~
			// 简单的说：只有value值相同，才会被选中的。否则这个Bean就是不符合条件的
			if (targetAnnotation != null && targetAnnotation.equals(annotation)) {
				return true;
			}
		}

		// 赞。若targetAnnotation还没找到，也就是还没匹配上。仍旧还不放弃，拿到当前这个注解的所有注解属性继续尝试匹配
		Map<String, Object> attributes = AnnotationUtils.getAnnotationAttributes(annotation);
		if (attributes.isEmpty() && qualifier == null) {
			return false;
		}
		... // 详情不描述了。这就是为什么我们吧@Qualifier标注在某个类上面都能生效的原因 就是这里做了非常强大的兼容性~
	}

// =================它最重要的两个判断=================
if (targetAnnotation != null && targetAnnotation.equals(annotation));

// Fall back on bean name (or alias) match
if (actualValue == null && attributeName.equals(AutowireCandidateQualifier.VALUE_KEY) &&
					expectedValue instanceof String && bdHolder.matchesName((String) expectedValue));

```

注意：Class.isAnnotationPresent(Class<? extends Annotation> annotationClass)表示annotationClass是否标注在此类型上（此类型可以是任意Class类型）。
此方法不具有传递性：比如注解A上标注有@Qualifier，注解B上标注有@A注解，那么你用此方法判断@B上是否有@Qualifier它是返回false的（即使都写了@Inherited注解，因为和它没关系）



checkQualifiers()方法它会检查标注的所有的注解（循环遍历一个个检查），规则如下：

若是限定符注解（自己就是@Qualifier或者isAnnotationPresent），匹配上了，就继续看下一个注解
- 也就说@Qualifier所标注的注解也算是限定符(isQualifier() = true)
  若是限定符注解但是没匹配上，那就fallback。继续看看标注在它身上的限定符注解（如果有）能否匹配上，若匹配上了也成
  若不是限定符注解，也是走fallback逻辑
  总之：若不是限定符注解直接忽略。若有多个限定符注解都生效，必须全部匹配上了，才算做最终匹配上。



# AnnotationUtils



# AnnotationConfigUtils 和 ConfigurationClassUtils

BeanFactoryUtils

BeanFactoryAnnotationUtils#isQualifierMatch



# AnnotationAttributes：Map的封装，在实际使用时会有更好的体验（LinkedHashMap<String, Object> ）

它的获取一般这么来：

* AnnotatedTypeMetadata#getAnnotationAttributes
* AnnotationAttributes#fromMap
* AnnotatedElementUtils#getMergedAnnotationAttributes等系列方法
  new AnnotationAttributes



# Type   泛型

<https://blog.csdn.net/f641385712/article/details/88789847>

`LocalVariableTableParameterNameDiscoverer`

原理是：通过ASM提供的通过字节码获取方法的参数名称，Spring给我们集成了这个提供了这个类，我们只需要简单的使用即可。





SimpleAutowireCandidateResolver没有任何实现，就是一个Adapter
GenericTypeAwareAutowireCandidateResolver：基础实现，有核心逻辑方法：checkGenericTypeMatch(BeanDefinitionHolder bdHolder, DependencyDescriptor descriptor)，

作用为：根据给定的候选bean定义，将给定的依赖类型与其泛型类型信息匹配。（这就是泛型依赖注入的核心匹配逻辑，所以这列其实也是Spring4.0后才推出来的）



GenericTypeResolver

**ResolvableType为所有的java类型提供了统一的数据结构以及API ，换句话说，一个ResolvableType对象就对应着一种java类型。** 下面我们看看具体的使用



## ImportAware#setImportMetadata







## @EnableConfigurationProperties

###  EnableConfigurationPropertiesRegistrar

#### ConfigurationPropertiesBean

#### ConfigurationPropertiesBindingPostProcessor（BeanPostProcessor）

初始化前绑定

#### ConfigurationPropertiesBinder

##### Binder

#### ConfigurationPropertiesBeanDefinitionValidator



#### ConfigurationBeanFactoryMetadata




