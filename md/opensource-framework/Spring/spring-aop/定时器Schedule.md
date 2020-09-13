

### Timer和TimerTask

Timer和TimerTask可以作为线程实现的第三种方式，在JDK1.3的时候推出。但是自从JDK1.5之后不再推荐时间，而是使用ScheduledThreadPoolExecutor代替

Timer运行在后台，可以执行任务一次，或定期执行任务。TimerTask类继承了Runnable接口，因此具备多线程的能力。一个Timer可以调度任意多个TimerTask，所有任务都存储在一个队列中顺序执行，如果需要多个TimerTask并发执行，则需要创建两个多个Timer。

一个Timer定时器，是单线程的

###### 终止Timer线程

调用Timer.cancle()方法。可以在程序任何地方调用，甚至在TimerTask中的run方法中调用；
设置Timer对象为null，其会自动终止；
用System.exit方法，整个程序终止。



Timer线程的缺点（这个就重要了）

* Timer线程不会捕获异常，所以TimerTask抛出的未检查的异常会终止timer线程。如果Timer线程中存在多个计划任务，其中一个计划任务抛出未检查的异常，则会引起整个Timer线程结束，从而导致其他计划任务无法得到继续执行。
* Timer线程时基于绝对时间(如：2014/02/14 16:06:00)，因此计划任务对系统的时间的改变是敏感的。(举个例子，假如你希望任务1每个10秒执行一次，某个时刻，你将系统时间提前了6秒，那么任务1就会在4秒后执行，而不是10秒后)
* Timer是单线程，如果某个任务很耗时，可能会影响其他计划任务的执行。
  Timer执行程序是有可能延迟1、2毫秒，如果是1秒执行一次的任务，1分钟有可能延迟60毫秒，一小时延迟3600毫秒，相当于3秒（如果你的任务对时间敏感，这将会有影响） ScheduledThreadPoolExecutor的时间会更加的精确

### ScheduledThreadPoolExecutor（JDK全新定时器调度,核心）

ScheduledThreadPoolExecutor是JDK1.5以后推出的类，用于实现定时、重复执行的功能，官方文档解释要优于Timer

scheduleAtFixedRate
是以上一个任务开始的时间计时，period时间过去后，检测上一个任务是否执行完毕，如果上一个任务执行完毕，则当前任务立即执行，如果上一个任务没有执行完毕，则需要等上一个任务执行完毕后立即执行。
执行周期是 initialDelay 、initialDelay+period 、initialDelay + 2 * period} 、 … 如果延迟任务的执行时间大于了 period，比如为 5s，则后面的执行会等待5s才回去执行
scheduleWithFixedDelay
是以上一个任务结束时开始计时，period时间过去后，立即执行, 由上面的运行结果可以看出，第一个任务开始和第二个任务开始的间隔时间是 第一个任务的运行时间+period（永远是这么多）

> 注意： 通过ScheduledExecutorService执行的周期任务，如果任务执行过程中抛出了异常，那么过ScheduledExecutorService就会停止执行任务，且也不会**再周期地执行该任务了**。所以你如果想保住任务都一直被周期执行，那么catch一切可能的异常。



### TriggerContext

接口表示触发的上下文。它能够获取上次任务`原本的计划时间`/`实际的执行时间`以及`实际的完成时间`

```java
//@since 3.0 我们发现每个方法都有可能返回null(比如首次执行)
public interface TriggerContext {
	// 上次预计的执行时间
	@Nullable
	Date lastScheduledExecutionTime();
	// 上次真正执行时间
	@Nullable
	Date lastActualExecutionTime();
	// 上次完成的时间
	@Nullable
	Date lastCompletionTime();

}
```

唯一实现类：`SimpleTriggerContext`

### Trigger

`TaskScheduler`中将会使用到`Trigger`对象，所以先对它进行分析

`Trigger`接口用于计算任务的下次执行时间。

```java
public interface Trigger {
	//获取下次执行时间
	@Nullable
	Date nextExecutionTime(TriggerContext triggerContext);
}
```

#### CronTrigger#nextExecutionTime

通过Cron表达式来生成调度计划。

```java
scheduler.schedule(task, new CronTrigger("0 15 9-17 * * MON-FRI"));
```

> Spring对cron表达式的支持，是由`CronSequenceGenerator`来实现的，不依赖于别的框架。

```java
@Override
	public Date nextExecutionTime(TriggerContext triggerContext) {
		Date date = triggerContext.lastCompletionTime();
		// 这里面有个处理：如果data为null，相当于任务已经完成了
		if (date != null) {
			// 拿到上一次预定执行的时间
			Date scheduled = triggerContext.lastScheduledExecutionTime();
			// 如果预定执行的时间为null（比如第一次）或者上一次还在data之后，那就取当前时间嘛
			if (scheduled != null && date.before(scheduled)) {
				// Previous task apparently executed too early...
				// Let's simply use the last calculated execution time then,
				// in order to prevent accidental re-fires in the same second.
				date = scheduled;
			}
		}
		// 如果任务还没有完成，那就以当前时间去计算下一个时间
		else {
			date = new Date();
		}
		return this.sequenceGenerator.next(date);
	}
```

#### PeriodicTrigger

用于定期执行的Trigger；它有两种模式：

* fixedRate：两次任务开始时间之间间隔指定时长
* fixedDelay: 上一次任务的结束时间与下一次任务开始时间``间隔指定时长

可见这两种情况的区别就在于，在决定下一次的执行计划时是否要考虑上次任务在什么时间执行完成。 默认情况下PeriodicTrigger使用了fixedDelay模式。

* period: long类型，表示间隔时长，注意在fixedRate与fixedDelay两种模式下的不同含义
* timeUnit: TimeUnit类型，表示间隔时长的单位，如毫秒等；默认是毫秒
* initialDelay: long类型，表示启动任务后间隔多长时间开始执行第一次任务
* fixedRate: boolean类型，表示是否是fixedRate，为True时是fixedRate，否则是fixedDelay，默认为False

### CronTask

### TaskScheduler

Spring任务调度器的核心接口，定义了执行定时任务的主要方法，主要根据任务的不同触发方式调用不同的执行逻辑，**其实现类都是对JDK原生的定时器或线程池组件进行包装，并扩展额外的功能**。

> TaskScheduler用于对Runnable的任务进行调度，它包含有多种触发规则。

```java
public interface TaskScheduler {

	// 提交任务调度请求 
	// Runnable task：待执行得任务
	// Trigger trigger：使用Trigger指定任务调度规则
	@Nullable
	ScheduledFuture<?> schedule(Runnable task, Trigger trigger);

	// @since 5.0  这里使用的Instant 类，其实最终也是转换成了Date
	default ScheduledFuture<?> schedule(Runnable task, Instant startTime) {
		return schedule(task, Date.from(startTime));
	}
	//  提交任务调度请求   startTime表示它的执行时间
	//  注意任务只执行一次，使用startTime指定其启动时间  
	ScheduledFuture<?> schedule(Runnable task, Date startTime);

	// @since 5.0
	default ScheduledFuture<?> scheduleAtFixedRate(Runnable task, Instant startTime, Duration period) {
		return scheduleAtFixedRate(task, Date.from(startTime), period.toMillis());
	}
	// 使用fixedRate的方式提交任务调度请求    任务首次启动时间由传入参数指定 
	// task　待执行的任务　 startTime　任务启动时间    period　两次任务启动时间之间的间隔时间，默认单位是毫秒
	ScheduledFuture<?> scheduleAtFixedRate(Runnable task, Date startTime, long period);

	// @since 5.0
	default ScheduledFuture<?> scheduleAtFixedRate(Runnable task, Duration period) {
		return scheduleAtFixedRate(task, period.toMillis());
	}
	// 使用fixedRate的方式提交任务调度请求 任务首次启动时间未设置，任务池将会尽可能早的启动任务
	// task 待执行任务 
	// period 两次任务启动时间之间的间隔时间，默认单位是毫秒
	ScheduledFuture<?> scheduleAtFixedRate(Runnable task, long period);

	// @since 5.0
	default ScheduledFuture<?> scheduleWithFixedDelay(Runnable task, Instant startTime, Duration delay) {
		return scheduleWithFixedDelay(task, Date.from(startTime), delay.toMillis());
	}
	//  使用fixedDelay的方式提交任务调度请求  任务首次启动时间由传入参数指定 
	// delay 上一次任务结束时间与下一次任务开始时间的间隔时间，单位默认是毫秒 
	ScheduledFuture<?> scheduleWithFixedDelay(Runnable task, Date startTime, long delay);
	// @since 5.0
	default ScheduledFuture<?> scheduleWithFixedDelay(Runnable task, Duration delay) {
		return scheduleWithFixedDelay(task, delay.toMillis());
	}
	// 使用fixedDelay的方式提交任务调度请求 任务首次启动时间未设置，任务池将会尽可能早的启动任务 
	// delay 上一次任务结束时间与下一次任务开始时间的间隔时间，单位默认是毫秒 
	ScheduledFuture<?> scheduleWithFixedDelay(Runnable task, long delay);

}
```

> 备注：TaskScheduler的另一实现类`TimerManagerTaskScheduler`在Spring5.0之后就被直接移除了

#### ThreadPoolTaskScheduler

包装Java Concurrent中的`ScheduledThreadPoolExecutor`类，**大多数场景下都使用它来进行任务调度**。

除实现了TaskScheduler接口中的方法外，它还包含了一些对ScheduledThreadPoolExecutor进行操作的接口

```java
public class ThreadPoolTaskScheduler extends ExecutorConfigurationSupport
		implements AsyncListenableTaskExecutor, SchedulingTaskExecutor, TaskScheduler {
	...
	// 默认的size 是1
	private volatile int poolSize = 1;
	private volatile boolean removeOnCancelPolicy = false;
	@Nullable
	private volatile ErrorHandler errorHandler;
	// 内部持有一个JUC的ScheduledExecutorService 的引用
	@Nullable
	private ScheduledExecutorService scheduledExecutor;
	...
	
	// 初始化线程池的执行器~~~~ 该方法的父类是ExecutorConfigurationSupport
	// 它定义了一些线程池的默认配置~~~~~
	@Override
	protected ExecutorService initializeExecutor(
			ThreadFactory threadFactory, RejectedExecutionHandler rejectedExecutionHandler) {
		// 我们发现，如果set PoolSize，那么它的size就是1
		this.scheduledExecutor = createExecutor(this.poolSize, threadFactory, rejectedExecutionHandler);

		if (this.removeOnCancelPolicy) {
			if (this.scheduledExecutor instanceof ScheduledThreadPoolExecutor) {
				((ScheduledThreadPoolExecutor) this.scheduledExecutor).setRemoveOnCancelPolicy(true);
			} else {
				logger.info("Could not apply remove-on-cancel policy - not a Java 7+ ScheduledThreadPoolExecutor");
			}
		}

		return this.scheduledExecutor;
	}
	// 就是new一个ScheduledThreadPoolExecutor  来作为最终执行任务的执行器
	protected ScheduledExecutorService createExecutor(
			int poolSize, ThreadFactory threadFactory, RejectedExecutionHandler rejectedExecutionHandler) {

		return new ScheduledThreadPoolExecutor(poolSize, threadFactory, rejectedExecutionHandler);
	}
	...
	
	//获取当前活动的线程数 委托给ScheduledThreadPoolExecutor来做得
	public int getActiveCount() {
		if (this.scheduledExecutor == null) {
			// Not initialized yet: assume no active threads.
			return 0;
		}
		return getScheduledThreadPoolExecutor().getActiveCount();
	}

	// 显然最终就是交给ScheduledThreadPoolExecutor去执行了~~~
	// 提交执行一次的任务
	// submit\submitListenable方法表示：提交执行一次的任务，并且返回一个Future对象供判断任务状态使用
	@Override
	public void execute(Runnable task) {
		Executor executor = getScheduledExecutor();
		try {
			executor.execute(errorHandlingTask(task, false));
		}
		catch (RejectedExecutionException ex) {
			throw new TaskRejectedException("Executor [" + executor + "] did not accept task: " + task, ex);
		}
	}
	...

}
```

> 使用它前必须得先调用initialize()【初始化方法】，有shutDown()方法，执行完后可以关闭线程

#### ConcurrentTaskScheduler

以单个线程方式执行定时任务，适用于简单场景；（以当前线程执行任务。如果任务简单，可以直接使用这个类来执行。快捷方便。）

#### DefaultManagedTaskScheduler

它继承自`ConcurrentTaskScheduler`，在ConcurrentTaskScheduler基础上增加了JNDI的支持。它`@since 4.0`

### ScheduledExecutorService

#### DelegatedScheduledExecutorService



### SchedulingConfigurer#configureTasks



### ScheduledTask

#### ScheduledFutureTask

定时任务类，内部包装了一个Runnable。

```java
// @since 4.3  发现这个类出现得还是比较晚得
public final class ScheduledTask {
	// 任务，其实就是很简单的包装了 Runnable。
	// 常见的子类有 TriggerTask、CronTask（主要是支持的CronTrigger、cron表达式）、
	// FixedDelayTask、FixedRateTask、IntervalTask(前两者得父类)
	private final Task task;
	@Nullable
	volatile ScheduledFuture<?> future;

	ScheduledTask(Task task) {
		this.task = task;
	}
	 //@since 5.0.2
	public Task getTask() {
		return this.task;
	}
	// 取消任务
	public void cancel() {
		ScheduledFuture<?> future = this.future;
		if (future != null) {
			future.cancel(true);
		}
	}
	@Override
	public String toString() {
		return this.task.toString();
	}
}
```





# @EnableScheduling

```java
//@since 3.1
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Import(SchedulingConfiguration.class)
@Documented
public @interface EnableScheduling {

}
```



### SchedulingConfiguration

它的效果同XML中的`<task:annotation-driven>`

```java
@Configuration
@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
public class SchedulingConfiguration {

	@Bean(name = TaskManagementConfigUtils.SCHEDULED_ANNOTATION_PROCESSOR_BEAN_NAME)
	@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
	public ScheduledAnnotationBeanPostProcessor scheduledAnnotationProcessor() {
		return new ScheduledAnnotationBeanPostProcessor();
	}

}
```



## ScheduledAnnotationBeanPostProcessor

Scheduled注解后处理器，项目启动时会扫描所有标记了`@Scheduled`注解的方法，封装成`ScheduledTask`注册起来。这个处理器是处理定时任务的核心类

如果是代理类会拿到原类型，需要的只是提取出 @Scheduled 标注方法，先缓存起来，之后会由应用主动调用

#### postProcessAfterInitialization（处理 @Scheduled，顺序问题）

##### processScheduled

标注 @Scheduled 的 method --> runnable --> schduledtask 

 registrar#scheduletask

#### finishRegistration（最后配置 Scheduler 、SchedulingConfigurer 和 ScheduledTaskRegistrar，并执行定时任务）

afterSingletonsInstantiated(bean生命周期中最后的扩展点) -->  finishRegistration --> registrar#afterproperties



```java
// 首先：非常震撼的是，它实现的接口非常的多。还好的是，大部分接口我们都很熟悉了。
// MergedBeanDefinitionPostProcessor：它是个BeanPostProcessor
// DestructionAwareBeanPostProcessor：在销毁此Bean的时候，会调用对应方法
// SmartInitializingSingleton：它会在所有的单例Bean都完成了初始化后，调用这个接口的方法
// EmbeddedValueResolverAware, BeanNameAware, BeanFactoryAware, ApplicationContextAware：都是些感知接口
// DisposableBean：该Bean销毁的时候会调用
// ApplicationListener<ContextRefreshedEvent>：监听容器的`ContextRefreshedEvent`事件
// ScheduledTaskHolder：维护本地的ScheduledTask实例
public class ScheduledAnnotationBeanPostProcessor
		implements ScheduledTaskHolder, MergedBeanDefinitionPostProcessor, DestructionAwareBeanPostProcessor,
		Ordered, EmbeddedValueResolverAware, BeanNameAware, BeanFactoryAware, ApplicationContextAware,
		SmartInitializingSingleton, ApplicationListener<ContextRefreshedEvent>, DisposableBean {

	/**
	 * The default name of the {@link TaskScheduler} bean to pick up: "taskScheduler".
	 * <p>Note that the initial lookup happens by type; this is just the fallback
	 * in case of multiple scheduler beans found in the context.
	 * @since 4.2
	 */
	 // 看着注释就知道，和@Async的默认处理一样~~~~先类型  在回退到名称
	public static final String DEFAULT_TASK_SCHEDULER_BEAN_NAME = "taskScheduler";
	// 调度器（若我们没有配置，它是null的）
	@Nullable
	private Object scheduler;
	
	// 这些都是Awire感知接口注入进来的~~
	@Nullable
	private StringValueResolver embeddedValueResolver;
	@Nullable
	private String beanName;
	@Nullable
	private BeanFactory beanFactory;
	@Nullable
	private ApplicationContext applicationContext;

	// ScheduledTaskRegistrar：ScheduledTask注册中心，ScheduledTaskHolder接口的一个重要的实现类，维护了程序中所有配置的ScheduledTask
	// 内部会处理调取器得工作，因此我建议先移步，看看这个类得具体分析
	private final ScheduledTaskRegistrar registrar = new ScheduledTaskRegistrar();

	// 缓存，没有被标注注解的class们  
	// 这有个技巧，使用了newSetFromMap，自然而然的这个set也就成了一个线程安全的set
	private final Set<Class<?>> nonAnnotatedClasses = Collections.newSetFromMap(new ConcurrentHashMap<>(64));

	// 缓存对应的Bean上  里面对应的 ScheduledTask任务。可议有多个哦~~
	// 注意：此处使用了IdentityHashMap
	private final Map<Object, Set<ScheduledTask>> scheduledTasks = new IdentityHashMap<>(16);

	// 希望此processor是最后执行的~
	@Override
	public int getOrder() {
		return LOWEST_PRECEDENCE;
	}

	 //Set the {@link org.springframework.scheduling.TaskScheduler} that will invoke the scheduled methods
	 // 也可以是JDK的ScheduledExecutorService(内部会给你包装成一个TaskScheduler)
	 // 若没有指定。那就会走默认策略：去从起中先按照类型找`TaskScheduler`该类型(或者ScheduledExecutorService这个类型也成)的。
	 // 若有多个该类型或者找不到，就安好"taskScheduler"名称去找 
	 // 再找不到，就用系统默认的：
	public void setScheduler(Object scheduler) {
		this.scheduler = scheduler;
	}
	...
	
	// 此方法会在该容器内所有的单例Bean已经初始化全部结束后，执行
	@Override
	public void afterSingletonsInstantiated() {
		// Remove resolved singleton classes from cache
		// 因为已经是最后一步了，所以这个缓存可议清空了
		this.nonAnnotatedClasses.clear();
		
		// 在容器内运行，ApplicationContext都不会为null
		if (this.applicationContext == null) {
			// Not running in an ApplicationContext -> register tasks early...
			// 如果不是在ApplicationContext下运行的，那么就应该提前注册这些任务
			finishRegistration();
		}
	}

	// 兼容容器刷新的时间（此时候容器硬启动完成了）   它还在`afterSingletonsInstantiated`的后面执行
	@Override
	public void onApplicationEvent(ContextRefreshedEvent event) {
		// 这个动作务必要做：因为Spring可能有多个容器，所以可能会发出多个ContextRefreshedEvent 事件
		// 显然我们只处理自己容器发出来得事件，别的容器发出来我不管~~
		if (event.getApplicationContext() == this.applicationContext) {
			// Running in an ApplicationContext -> register tasks this late...
			// giving other ContextRefreshedEvent listeners a chance to perform
			// their work at the same time (e.g. Spring Batch's job registration).
			// 为其他ContextRefreshedEvent侦听器提供同时执行其工作的机会（例如，Spring批量工作注册）
			finishRegistration();
		}
	}

	private void finishRegistration() {
		// 如果setScheduler了，就以调用者指定的为准~~~
		if (this.scheduler != null) {
			this.registrar.setScheduler(this.scheduler);
		}

		// 这里继续厉害了：从容器中找到所有的接口`SchedulingConfigurer`的实现类（我们可议通过实现它定制化scheduler）
		if (this.beanFactory instanceof ListableBeanFactory) {
			Map<String, SchedulingConfigurer> beans =
					((ListableBeanFactory) this.beanFactory).getBeansOfType(SchedulingConfigurer.class);
			List<SchedulingConfigurer> configurers = new ArrayList<>(beans.values());
			
			// 同@Async只允许设置一个不一样的是，这里每个都会让它生效
			// 但是平时使用，我们自顶一个类足矣~~~
			AnnotationAwareOrderComparator.sort(configurers);
			for (SchedulingConfigurer configurer : configurers) {
				configurer.configureTasks(this.registrar);
			}
		}
		
		// 至于task是怎么注册进registor的，请带回看`postProcessAfterInitialization`这个方法的实现
		// 有任务并且registrar.getScheduler() == null，那就去容器里找来试试~~~
		if (this.registrar.hasTasks() && this.registrar.getScheduler() == null) {
			...
			// 这块逻辑和@Async的处理一毛一样。忽略了 主要看看resolveSchedulerBean()这个方法即可
		}
        // 定时任务执行
		this.registrar.afterPropertiesSet();
	}
            
	// 从容器中去找一个
	private <T> T resolveSchedulerBean(BeanFactory beanFactory, Class<T> schedulerType, boolean byName) {
		// 若按名字去查找，那就按照名字找
		if (byName) {
			T scheduler = beanFactory.getBean(DEFAULT_TASK_SCHEDULER_BEAN_NAME, schedulerType);

			// 这个处理非常非常有意思，就是说倘若找到了你可以在任意地方直接@Autowired这个Bean了，可以拿这个共用Scheduler来调度我们自己的任务啦~~
			if (this.beanName != null && this.beanFactory instanceof ConfigurableBeanFactory) {
				((ConfigurableBeanFactory) this.beanFactory).registerDependentBean(
						DEFAULT_TASK_SCHEDULER_BEAN_NAME, this.beanName);
			}
			return scheduler;
		}
		// 按照schedulerType该类型的名字匹配resolveNamedBean  底层依赖：getBeanNamesForType
		else if (beanFactory instanceof AutowireCapableBeanFactory) {
			NamedBeanHolder<T> holder = ((AutowireCapableBeanFactory) beanFactory).resolveNamedBean(schedulerType);
			if (this.beanName != null && beanFactory instanceof ConfigurableBeanFactory) {
				((ConfigurableBeanFactory) beanFactory).registerDependentBean(holder.getBeanName(), this.beanName);
			}
			return holder.getBeanInstance();
		}
		// 按照类型找
		else {
			return beanFactory.getBean(schedulerType);
		}
	}


	@Override
	public void postProcessMergedBeanDefinition(RootBeanDefinition beanDefinition, Class<?> beanType, String beanName) {
	}
	@Override
	public Object postProcessBeforeInitialization(Object bean, String beanName) {
		return bean;
	}

	// Bean初始化完成后执行。去看看Bean里面有没有标注了@Scheduled的方法~~
	@Override
	public Object postProcessAfterInitialization(final Object bean, String beanName) {
		// 拿到目标类型（因为此类有可能已经被代理过）
		Class<?> targetClass = AopProxyUtils.ultimateTargetClass(bean);
		// 这里对没有标注注解的类做了一个缓存，防止从父去扫描（毕竟可能有多个容器，可能有重复扫描的现象）
		if (!this.nonAnnotatedClasses.contains(targetClass)) {

			// 如下：主要用到了MethodIntrospector.selectMethods  这个内省方法工具类的这个工具方法，去找指定Class里面，符合条件的方法们
			Map<Method, Set<Scheduled>> annotatedMethods = MethodIntrospector.selectMethods(targetClass,
					(MethodIntrospector.MetadataLookup<Set<Scheduled>>) method -> {
					
						//过滤Method的核心逻辑就是是否标注有此注解（Merged表示标注在父类、或者接口处也是ok的）
						Set<Scheduled> scheduledMethods = AnnotatedElementUtils.getMergedRepeatableAnnotations(
								method, Scheduled.class, Schedules.class);
						return (!scheduledMethods.isEmpty() ? scheduledMethods : null);
					});
			if (annotatedMethods.isEmpty()) {
				this.nonAnnotatedClasses.add(targetClass);
				if (logger.isTraceEnabled()) {
					logger.trace("No @Scheduled annotations found on bean class: " + bean.getClass());
				}
			}
			// 此处相当于已经找到了对应的注解方法~~~
			else {
				// Non-empty set of methods
				// 这里有一个双重遍历。因为一个方法上，可能重复标注多个这样的注解~~~~~
				// 所以最终遍历出来后，就交给processScheduled(scheduled, method, bean)去处理了
				annotatedMethods.forEach((method, scheduledMethods) ->
						scheduledMethods.forEach(scheduled -> processScheduled(scheduled, method, bean)));
				if (logger.isDebugEnabled()) {
					logger.debug(annotatedMethods.size() + " @Scheduled methods processed on bean '" + beanName + "': " + annotatedMethods);
				}
			}
		}
		return bean;
	}

	// 这个方法就是灵魂了。就是执行这个注解，最终会把这个任务注册进去，并且启动的~~~
	protected void processScheduled(Scheduled scheduled, Method method, Object bean) {
		try {
			// 标注此注解的方法必须是无参的方法
			Assert.isTrue(method.getParameterCount() == 0, "Only no-arg methods may be annotated with @Scheduled");

			// 拿到最终要被调用的方法  做这么一步操作主要是防止方法被代理了
			Method invocableMethod = AopUtils.selectInvocableMethod(method, bean.getClass());
			// 把该方法包装成一个Runnable 线程~~~
			Runnable runnable = new ScheduledMethodRunnable(bean, invocableMethod);
			boolean processedSchedule = false;
			String errorMessage = "Exactly one of the 'cron', 'fixedDelay(String)', or 'fixedRate(String)' attributes is required";

			// 装载任务，这里长度定为4，因为Spring认为标注4个注解还不够你用的？
			Set<ScheduledTask> tasks = new LinkedHashSet<>(4);

			// Determine initial delay
			// 计算出延时多长时间执行 initialDelayString 支持占位符如：@Scheduled(fixedDelayString = "${time.fixedDelay}")
			// 最终拿到一个initialDelay值,Long型的
			long initialDelay = scheduled.initialDelay();
			String initialDelayString = scheduled.initialDelayString();
			if (StringUtils.hasText(initialDelayString)) {
				Assert.isTrue(initialDelay < 0, "Specify 'initialDelay' or 'initialDelayString', not both");
				if (this.embeddedValueResolver != null) {
					initialDelayString = this.embeddedValueResolver.resolveStringValue(initialDelayString);
				}
				if (StringUtils.hasLength(initialDelayString)) {
					try {
						initialDelay = parseDelayAsLong(initialDelayString);
					}
					catch (RuntimeException ex) {
						throw new IllegalArgumentException(
								"Invalid initialDelayString value \"" + initialDelayString + "\" - cannot parse into long");
					}
				}
			}

			// Check cron expression
			// 解析cron
			String cron = scheduled.cron();
			if (StringUtils.hasText(cron)) {
				String zone = scheduled.zone();
				// cron也可以使用占位符。把它配置在配置文件里就成~~~zone也是支持占位符的
				if (this.embeddedValueResolver != null) {
					cron = this.embeddedValueResolver.resolveStringValue(cron);
					zone = this.embeddedValueResolver.resolveStringValue(zone);
				}
				if (StringUtils.hasLength(cron)) {
					Assert.isTrue(initialDelay == -1, "'initialDelay' not supported for cron triggers");
					processedSchedule = true;
					TimeZone timeZone;
					if (StringUtils.hasText(zone)) {
						timeZone = StringUtils.parseTimeZoneString(zone);
					}
					else {
						timeZone = TimeZone.getDefault();
					}

					// 这个相当于，如果配置了cron，它就是一个task了，就可以吧任务注册进registrar里面了
					// 这里面的处理是。如果已经有调度器taskScheduler了，那就立马准备执行了															
					tasks.add(this.registrar.scheduleCronTask(new CronTask(runnable, new CronTrigger(cron, timeZone))));
				}
			}

			// At this point we don't need to differentiate between initial delay set or not anymore
			if (initialDelay < 0) {
				initialDelay = 0;
			}
			...
			// 下面就不再说了，就是解析fixed delay、fixed rated、
			
			// Check whether we had any attribute set
			Assert.isTrue(processedSchedule, errorMessage);

			// Finally register the scheduled tasks
			// 最后吧这些任务都放在全局属性里保存起来~~~~
			// getScheduledTasks()方法是会把所有的任务都返回出去的~~~ScheduledTaskHolder接口就一个Set<ScheduledTask> getScheduledTasks();方法嘛
			synchronized (this.scheduledTasks) {
				Set<ScheduledTask> registeredTasks = this.scheduledTasks.get(bean);
				if (registeredTasks == null) {
					registeredTasks = new LinkedHashSet<>(4);
					this.scheduledTasks.put(bean, registeredTasks);
				}
				registeredTasks.addAll(tasks);
			}
		}
		catch (IllegalArgumentException ex) {
			throw new IllegalStateException(
					"Encountered invalid @Scheduled method '" + method.getName() + "': " + ex.getMessage());
		}
	}

	private static long parseDelayAsLong(String value) throws RuntimeException {
		if (value.length() > 1 && (isP(value.charAt(0)) || isP(value.charAt(1)))) {
			return Duration.parse(value).toMillis();
		}
		return Long.parseLong(value);
	}

	private static boolean isP(char ch) {
		return (ch == 'P' || ch == 'p');
	}

	 //@since 5.0.2 获取到所有的任务。包含本实例的，以及registrar（手动注册）的所有任务
	@Override
	public Set<ScheduledTask> getScheduledTasks() {
		Set<ScheduledTask> result = new LinkedHashSet<>();
		synchronized (this.scheduledTasks) {
			Collection<Set<ScheduledTask>> allTasks = this.scheduledTasks.values();
			for (Set<ScheduledTask> tasks : allTasks) {
				result.addAll(tasks);
			}
		}
		result.addAll(this.registrar.getScheduledTasks());
		return result;
	}

	// Bean销毁之前执行。移除掉所有的任务，并且取消所有的任务
	@Override
	public void postProcessBeforeDestruction(Object bean, String beanName) {
		Set<ScheduledTask> tasks;
		synchronized (this.scheduledTasks) {
			tasks = this.scheduledTasks.remove(bean);
		}
		if (tasks != null) {
			for (ScheduledTask task : tasks) {
				task.cancel();
			}
		}
	}
	...
}

```

Spring怎么发现Task、执行Task的流程



### ScheduledTaskRegistrar#scheduleTasks（执行定时任务）

ScheduledAnnotationBeanPostProcessor#afterSingletonsInstantiated(bean生命周期中最后的扩展点) -->  ScheduledAnnotationBeanPostProcessor#finishRegistration --> ScheduledTaskRegistrar#afterPropertiesSet --> ScheduledTaskRegistrarscheduleTask

#### contextLifecycleScheduledTaskRegistrar

`ScheduledTask`注册中心，`ScheduledTaskHolder`接口的一个重要的实现类，维护了程序中所有配置的`ScheduledTask`。

```java
//@since 3.0 它在Spring3.0就有了
// 这里面又有一个重要的接口：我们可议通过扩展实现此接口，来定制化属于自己的ScheduledTaskRegistrar 下文会有详细介绍
// 它实现了InitializingBean和DisposableBean，所以我们也可以把它放进容器里面
public class ScheduledTaskRegistrar implements ScheduledTaskHolder, InitializingBean, DisposableBean {
	
	// 任务调度器
	@Nullable
	private TaskScheduler taskScheduler;
	// 该类事JUC包中的类
	@Nullable
	private ScheduledExecutorService localExecutor;

	// 对任务进行分类 管理
	@Nullable
	private List<TriggerTask> triggerTasks;
	@Nullable
	private List<CronTask> cronTasks;
	@Nullable
	private List<IntervalTask> fixedRateTasks;
	@Nullable
	private List<IntervalTask> fixedDelayTasks;
	private final Map<Task, ScheduledTask> unresolvedTasks = new HashMap<>(16);
	private final Set<ScheduledTask> scheduledTasks = new LinkedHashSet<>(16);

	// 调用者可议自己指定一个TaskScheduler 
	public void setTaskScheduler(TaskScheduler taskScheduler) {
		Assert.notNull(taskScheduler, "TaskScheduler must not be null");
		this.taskScheduler = taskScheduler;
	}

	// 这里，如果你指定的是个TaskScheduler、ScheduledExecutorService都是阔仪得
	// ConcurrentTaskScheduler也是一个TaskScheduler的实现类
	public void setScheduler(@Nullable Object scheduler) {
		if (scheduler == null) {
			this.taskScheduler = null;
		}
		else if (scheduler instanceof TaskScheduler) {
			this.taskScheduler = (TaskScheduler) scheduler;
		}
		else if (scheduler instanceof ScheduledExecutorService) {
			this.taskScheduler = new ConcurrentTaskScheduler(((ScheduledExecutorService) scheduler));
		}
		else {
			throw new IllegalArgumentException("Unsupported scheduler type: " + scheduler.getClass());
		}
	}

	@Nullable
	public TaskScheduler getScheduler() {
		return this.taskScheduler;
	}
	// 将触发的任务指定为可运行文件（任务）和触发器对象的映射  typically custom implementations of the {@link Trigger} interface
	// org.springframework.scheduling.Trigger
	public void setTriggerTasks(Map<Runnable, Trigger> triggerTasks) {
		this.triggerTasks = new ArrayList<>();
		triggerTasks.forEach((task, trigger) -> addTriggerTask(new TriggerTask(task, trigger)));
	}

	// 主要处理` <task:*>`这种配置
	public void setTriggerTasksList(List<TriggerTask> triggerTasks) {
		this.triggerTasks = triggerTasks;
	}
	public List<TriggerTask> getTriggerTaskList() {
		return (this.triggerTasks != null? Collections.unmodifiableList(this.triggerTasks) :
				Collections.emptyList());
	}

	// 这个一般是最常用的 CronTrigger：Trigger的一个实现类。另一个实现类为PeriodicTrigger
	public void setCronTasks(Map<Runnable, String> cronTasks) {
		this.cronTasks = new ArrayList<>();
		cronTasks.forEach(this::addCronTask);
	}
	public void setCronTasksList(List<CronTask> cronTasks) {
		this.cronTasks = cronTasks;
	}
	public List<CronTask> getCronTaskList() {
		return (this.cronTasks != null ? Collections.unmodifiableList(this.cronTasks) :
				Collections.emptyList());
	}
	...
	// 判断是否还有任务
	public boolean hasTasks() {
		return (!CollectionUtils.isEmpty(this.triggerTasks) ||
				!CollectionUtils.isEmpty(this.cronTasks) ||
				!CollectionUtils.isEmpty(this.fixedRateTasks) ||
				!CollectionUtils.isEmpty(this.fixedDelayTasks));
	}


	/**
	 * Calls {@link #scheduleTasks()} at bean construction time.
	 */
	// 这个方法很重要：开始执行所有已经注册的任务们~~
	@Override
	public void afterPropertiesSet() {
		scheduleTasks();
	}
    
	@SuppressWarnings("deprecation")
	protected void scheduleTasks() {
		// 这一步非常重要：如果我们没有指定taskScheduler ，这里面会new一个newSingleThreadScheduledExecutor
		// 显然它并不是是一个真的线程池，所以他所有的任务还是得一个一个的One by One的执行的  请务必注意啊~~~~
		// 默认是它：Executors.newSingleThreadScheduledExecutor() 所以肯定串行啊
		if (this.taskScheduler == null) {
			this.localExecutor = Executors.newSingleThreadScheduledExecutor();
			this.taskScheduler = new ConcurrentTaskScheduler(this.localExecutor);
		}


		// 加下来做得事，就是借助TaskScheduler来启动每个任务
		// 并且把启动了的任务最终保存到scheduledTasks里面~~~ 后面还会介绍TaskScheduler的两个实现
		if (this.triggerTasks != null) {
			for (TriggerTask task : this.triggerTasks) {
				addScheduledTask(scheduleTriggerTask(task));
			}
		}
		if (this.cronTasks != null) {
			for (CronTask task : this.cronTasks) {
				addScheduledTask(scheduleCronTask(task));
			}
		}
		if (this.fixedRateTasks != null) {
			for (IntervalTask task : this.fixedRateTasks) {
				addScheduledTask(scheduleFixedRateTask(task));
			}
		}
		if (this.fixedDelayTasks != null) {
			for (IntervalTask task : this.fixedDelayTasks) {
				addScheduledTask(scheduleFixedDelayTask(task));
			}
		}
	}

	private void addScheduledTask(@Nullable ScheduledTask task) {
		if (task != null) {
			this.scheduledTasks.add(task);
		}
	}
	@Override
	public Set<ScheduledTask> getScheduledTasks() {
		return Collections.unmodifiableSet(this.scheduledTasks);
	}

	// 销毁： task.cancel()取消所有的任务  调用的是Future.cancel()
	// 并且关闭线程池localExecutor.shutdownNow()
	@Override
	public void destroy() {
		for (ScheduledTask task : this.scheduledTasks) {
			task.cancel();
		}
		if (this.localExecutor != null) {
			this.localExecutor.shutdownNow();
		}
	}

}

```

#### ReschedulingRunnable

### SchedulingConfigurer（提供扩展，没有默认）

@Scheduled

```java
@EnableScheduling
@Configuration
public class ScheduldConfig implements SchedulingConfigurer {

	// 下面是推荐的配置，当然你也可以简单粗暴的使用它：  开启100个核心线程  足够用了吧
	// taskRegistrar.setScheduler(Executors.newScheduledThreadPool(100));
    @Override
    public void configureTasks(ScheduledTaskRegistrar taskRegistrar) {
        // 方式一：可议直接使用一个TaskScheduler  然后设置上poolSize等参数即可  （推荐）
        //ThreadPoolTaskScheduler taskScheduler = new ThreadPoolTaskScheduler();
        //taskScheduler.setPoolSize(10);
        //taskScheduler.initialize(); // 记得调用啊
        //taskRegistrar.setTaskScheduler(taskScheduler);

        // 方式二：ScheduledExecutorService(使用ScheduledThreadPoolExecutor,它继承自JUC的ThreadPoolExecutor，是一个和任务调度相关的线程池)
        ScheduledThreadPoolExecutor poolExecutor = new ScheduledThreadPoolExecutor(10);
        taskRegistrar.setScheduler(poolExecutor);

        // =========示例：这里，我们也可以注册一个任务，一直执行的~~~=========
        taskRegistrar.addFixedRateTask(() -> System.out.println("执行定时任务1: " + new Date()), 1000);

    }
}
```

其实这里给了我们`打开了一个思路`。通过这我们可以捕获到`ScheduledTaskRegistrar`，从而我们可以通过接口动态的去改变任务的执行时间、以及对任务的增加、删、改、查等操作



Task在平时业务开发中确实使用非常的广泛，但在分布式环境下，其实已经很少使用Spring自带的定时器了，而使用分布式任务调度框架：Elastic-job、xxl-job等

另外说几点使用细节：

标注@Scheduled注解的方法必须无入数
cron、fixedDelay、fixedRate注解属性必须至少一个
若在分布式环境（或者集群环境）直接使用Spring的Scheduled，请使用分布式锁或者保证任务的幂等
