## AbstractApplicationContext#finishRefresh

```
protected void finishRefresh() {
		// Clear context-level resource caches (such as ASM metadata from scanning).
		clearResourceCaches();

		// Initialize lifecycle processor for this context.
		initLifecycleProcessor();

		// Propagate refresh to lifecycle processor first.
		getLifecycleProcessor().onRefresh();

		// Publish the final event.
		publishEvent(new ContextRefreshedEvent(this));

		// Participate in LiveBeansView MBean, if active.
		LiveBeansView.registerApplicationContext(this);
	}
	
	
DefaultLifecycleProcessor#onRefresh();
public void onRefresh() {
		startBeans(true);
		this.running = true;
	}
	
private void startBeans(boolean autoStartupOnly) {
		Map<String, Lifecycle> lifecycleBeans = getLifecycleBeans();
		Map<Integer, LifecycleGroup> phases = new HashMap<>();
		lifecycleBeans.forEach((beanName, bean) -> {
		    // 
			if (!autoStartupOnly || (bean instanceof SmartLifecycle && ((SmartLifecycle) bean).isAutoStartup())) {
				int phase = getPhase(bean);
				LifecycleGroup group = phases.get(phase);
				if (group == null) {
					group = new LifecycleGroup(phase, this.timeoutPerShutdownPhase, lifecycleBeans, autoStartupOnly);
					phases.put(phase, group);
				}
				group.add(beanName, bean);
			}
		});
		if (!phases.isEmpty()) {
			List<Integer> keys = new ArrayList<>(phases.keySet());
			Collections.sort(keys);
			for (Integer key : keys) {
				phases.get(key).start();
			}
		}
	}
	
LifecycleGroup#start()
public void start() {
			if (this.members.isEmpty()) {
				return;
			}
			if (logger.isDebugEnabled()) {
				logger.debug("Starting beans in phase " + this.phase);
			}
			Collections.sort(this.members);
			for (LifecycleGroupMember member : this.members) {
				doStart(this.lifecycleBeans, member.name, this.autoStartupOnly);
			}
		}

DefaultLifecycleProcessor#doStart
private void doStart(Map<String, ? extends Lifecycle> lifecycleBeans, String beanName, boolean autoStartupOnly) {
		Lifecycle bean = lifecycleBeans.remove(beanName);
		if (bean != null && bean != this) {
			String[] dependenciesForBean = getBeanFactory().getDependenciesForBean(beanName);
			for (String dependency : dependenciesForBean) {
				doStart(lifecycleBeans, dependency, autoStartupOnly);
			}
			if (!bean.isRunning() &&
					(!autoStartupOnly || !(bean instanceof SmartLifecycle) || ((SmartLifecycle) bean).isAutoStartup())) {
				if (logger.isTraceEnabled()) {
					logger.trace("Starting bean '" + beanName + "' of type [" + bean.getClass().getName() + "]");
				}
				try {
					bean.start();
				}
				catch (Throwable ex) {
					throw new ApplicationContextException("Failed to start bean '" + beanName + "'", ex);
				}
				if (logger.isDebugEnabled()) {
					logger.debug("Successfully started bean '" + beanName + "'");
				}
			}
		}
	}
```



## Spring Lifecycle

### SmartLifecycle

Lifecycle接口定义了每个对象的重要方法，每个对象都有自己的生命周期需求，如下：

```
public interface Lifecycle {  
  void start();  
  void stop();  
  boolean isRunning();  
}  
```

任何spring管理的对象都可以实现这个接口。那么，当ApplicationContext自身启动和停止时，它将自动调用上下文内所有生命周期的实现。通过委托给`LifecycleProcessor``来做这个工作。`注意`LifecycleProcessor``自身扩展了Lifecycle接口。它也增加了两个其他的方法来与上下文交互，使得可以刷新和关闭。`

```
public interface LifecycleProcessor extends Lifecycle {  
  void onRefresh();  
  void onClose();  
}

public interface SmartLifecycle extends Lifecycle, Phased {

	int DEFAULT_PHASE = Integer.MAX_VALUE;

	default boolean isAutoStartup() {
		return true;
	}

	default void stop(Runnable callback) {
		stop();
		callback.run();
	}
	
	@Override
	default int getPhase() {
		return DEFAULT_PHASE;
	}

}

```

启动和关闭调用的顺序是很重要的。如果两个对象之间存在依赖关系，依赖类要在其依赖类后启动，依赖类也要在其依赖类前停止。然而有时候其之间的依赖关系不是那么直接。你可能仅仅知道某种类型的对象应该在另一类型对象前启动。在那些情况下，SmartLifecycle接口定义了另一个选项，在其父类接口Phased中定义命名为getPhase()方法。



```
public interface Phased {  
  int getPhase();  
} 
public interface SmartLifecycle extends Lifecycle, Phased {  
  boolean isAutoStartup();  
  void stop(Runnable callback);  
}  
```

当启动时，有最低phase的对象首先启动，并且停止时，按照相反的顺序结束。因此，实现了`SmartLifecycle``接口并且其`getPhase()方法返回`Integer.MIN_VALUE``的一个对象将是首先被启动并且最后停止。与其相反对应的对象，Integer.MAX_VALUE的phase的值，将指明最后启动和最先停止（可能是其依赖其他对象工作运行）。当考虑phase值时，了解任何普通Lifecycle对象（没有实现SmartLifecycle其值将是0）的默认phase也很重要。因此，任何负数phase值将表示对象应该在那些标准组件前启动（并其之后停止），并且对于正数的phase值按照相反顺序启动停止。`

`如你在SmartLifecycle接口中定义的stop方法内有一回调参数。任何实现类在其关闭完成后必须调用回调的run方法。必须的时候由于实现了LifecycleProcessor接口的实现类可以进行异步关闭操作，DefaultLifecycleProcessor对于在每个phase调用那个回调内的对象组将等待一个超时时间。默认的每个phase的超时是30秒。你可以通过在上下文内定义一个命名为`lifecycleProcessor的bean重写默认的生命周期处理器实例。如果你仅仅想修改超时时间，如下定义将会很有用：

```
<bean id="lifecycleProcessor"class="org.springframework.context.support.DefaultLifecycleProcessor">  
  <!-- timeout value in milliseconds -->  
  <property name="timeoutPerShutdownPhase" value="10000"/>  
</bean>  

```

LifecycleProcessor 接口定义了回调方法来刷新和关闭上下文。后者仅简单地做关闭处理如同直接地调用stop方法，但是当关闭上下文时，这将起作用。刷新回调使得 SmartLifecycle bean的另一个功能起作用。当上下文刷新时（在所有的对象实例化和初始化后），将调用那个回调，并且在那个点上，默认的生命周期处理器将检查每个SmartLifecycle对象的isAutoStartup()方法的返回值。如果是true，那么对象将在那个点上启动而不是等一个上下文的明确调用或者等其自己的start()方法（不像上下文的刷新，上下文启动对于标准的上下文实现不是自动发生的）phase值与依赖关系一样将如上所述决定了启动顺序。