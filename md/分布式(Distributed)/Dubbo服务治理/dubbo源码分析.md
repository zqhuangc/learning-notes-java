# Dubbo 源码解读要点（注意不同版本的区别，以官网文档为参考）

首先我们要关注的是服务的发布和服务的消费这两个主要的流程，那么就可以基于这个点去找到源码分析的突破口。那么自然而然我们就可以想到spring的配置

## Spring对外留出的扩展

dubbo是基于 spring 配置来实现服务的发布的，那么一定是基于spring的扩展来写了一套自己的标签，那么spring是如何解析这些配置呢？

* 在spring中定义了两个接口
  - NamespaceHandler: 注册一堆 BeanDefinitionParser，利用他们来进行解析
  - BeanDefinitionParser:用于解析每个 element 的内容

 

Spring默认会加载jar包下的`META-INF/spring.handlers`文件寻找对应的`NamespaceHandler`。 

Dubbo-config 模块下的 dubbo-config-spring 

## Dubbo的接入实现

Dubbo中spring扩展就是使用spring的自定义类型，所以同样也有NamespaceHandler、BeanDefinitionParser。而NamespaceHandler是DubboNamespaceHandler

```java
public class DubboNamespaceHandler extends NamespaceHandlerSupport {
   static {
      Version.checkDuplicate(DubboNamespaceHandler.class);
   }
   public void init() {
        registerBeanDefinitionParser("application", new DubboBeanDefinitionParser(ApplicationConfig.class, true));
        registerBeanDefinitionParser("module", new DubboBeanDefinitionParser(ModuleConfig.class, true));
        registerBeanDefinitionParser("registry", new DubboBeanDefinitionParser(RegistryConfig.class, true));
        registerBeanDefinitionParser("monitor", new DubboBeanDefinitionParser(MonitorConfig.class, true));
        registerBeanDefinitionParser("provider", new DubboBeanDefinitionParser(ProviderConfig.class, true));
        registerBeanDefinitionParser("consumer", new DubboBeanDefinitionParser(ConsumerConfig.class, true));
        registerBeanDefinitionParser("protocol", new DubboBeanDefinitionParser(ProtocolConfig.class, true));
        registerBeanDefinitionParser("service", new DubboBeanDefinitionParser(ServiceBean.class, true));
        registerBeanDefinitionParser("reference", new DubboBeanDefinitionParser(ReferenceBean.class, false));
        registerBeanDefinitionParser("annotation", new DubboBeanDefinitionParser(AnnotationBean.class, true));
    }
}

```



BeanDefinitionParser全部都使用了DubboBeanDefinitionParser，如果我们想看<dubbo:service>的配置，就直接看DubboBeanDefinitionParser中

这个里面主要做了一件事，把不同的配置分别转化成spring容器中的bean对象

application对应ApplicationConfig

registry对应RegistryConfig

monitor对应MonitorConfig

provider对应ProviderConfig

consumer对应ConsumerConfig

…

为了在spring启动的时候，也相应的启动provider发布服务注册服务的过程，而同时为了让客户端在启动的时候自动订阅发现服务，加入了两个bean

ServiceBean、ReferenceBean。分别继承了 ServiceConfig 和 ReferenceConfig

同时还分别实现了InitializingBean、DisposableBean, ApplicationContextAware, ApplicationListener, BeanNameAware

**InitializingBean**接口为bean提供了初始化方法的方式，它只包括afterPropertiesSet方法，凡是继承该接口的类，在初始化bean的时候会执行该方法。

**DisposableBean** bean被销毁的时候，spring容器会自动执行destory方法，比如释放资源

**ApplicationContextAware** 实现了这个接口的bean，当spring容器初始化的时候，会自动的将ApplicationContext注入进来

**ApplicationListener**  ApplicationEvent事件监听，spring容器启动后会发一个事件通知

**BeanNameAware** 获得自身初始化时，本身的bean的id属性

 

那么基本的实现思路可以整理出来了

1. 利用spring的解析收集xml中的配置信息，然后把这些配置信息存储到serviceConfig中
2. 调用ServiceConfig的export方法来进行服务的发布和注册



### 注解驱动

#### xml 配置

激活 ：new DubboBeanDefinitionParser(AnnotationBean.class, true)

# 前提 ：了解 netty 工作原理源码

# 服务的发布

ServiceConfigBase

ConfigurableMetadataServiceExporter#export

> DubboBootstrapApplicationListener#onContextRefreshedEvent
>
> DubboBootstrap#start --> 
>
> DubboBootstrap#exportServices --> 
>
> ServiceConfig#export

## ServiceBean



旧版：ServiceBean 是服务发布的切入点，通过 afterPropertiesSet 方法，调用export()方法进行发布。

export 为父类 ServiceConfig 中的方法，所以跳转到 SeviceConfig 类中的 export 方法

新版：服务导出的入口方法是 ServiceBean 的 onApplicationEvent。onApplicationEvent 是一个事件响应方法，该方法会在收到 Spring 上下文刷新事件后执行服务导出操作。

```java
public void onApplicationEvent(ContextRefreshedEvent event) {
    // 是否有延迟导出 && 是否已导出 && 是不是已被取消导出
    if (isDelay() && !isExported() && !isUnexported()) {
        // 导出服务
        export();
    }
}
```

* delay的使用

我们发现，delay的作用就是延迟暴露，而延迟的方式也很直截了当，Thread.sleep(delay)

```java
// -☆- ServiceBean
private boolean isDelay() {
    // 获取 delay
    Integer delay = getDelay();
    ProviderConfig provider = getProvider();
    if (delay == null && provider != null) {
        // 如果前面获取的 delay 为空，这里继续获取
        delay = provider.getDelay();
    }
    // 判断 delay 是否为空，或者等于 -1
    return supportedApplicationListener && (delay == null || delay == -1);
}
```

前置工作主要包含两个部分，分别是配置检查，以及 URL 装配。在导出服务之前，Dubbo 需要检查用户的配置是否合理，或者为用户补充缺省配置。配置检查完成后，接下来需要根据这些配置组装 URL。在 Dubbo 中，URL 的作用十分重要。Dubbo 使用 URL 作为配置载体，所有的拓展点都是通过 URL 获取配置。

1. export 是 synchronized 修饰的方法。也就是说暴露的过程是原子操作，正常情况下不会出现锁竞争的问题，毕竟初始化过程大多数情况下都是单一线程操作，这里联想到了spring的初始化流程，也进行了加锁操作，这里也给我们平时设计一个不错的启示：初始化流程的性能调优优先级应该放的比较低，但是安全的优先级应该放的比较高！

2. 继续看 doExport() 方法。同样是一堆初始化代码

## export的过程

```java
public synchronized void export() {
    if (provider != null) {
        // 获取 export 和 delay 配置
        if (export == null) {
            export = provider.getExport();
        }
        if (delay == null) {
            delay = provider.getDelay();
        }
    }
    // 如果 export 为 false，则不导出服务
    if (export != null && !export) {
        return;
    }

    // delay > 0，延时导出服务
    if (delay != null && delay > 0) {
        delayExportExecutor.schedule(new Runnable() {
            @Override
            public void run() {
                doExport();
            }
        }, delay, TimeUnit.MILLISECONDS);
        
    // 立即导出服务
    } else {
        doExport();
    }
}


//-----------------
/**
检测 <dubbo:service> 标签的 interface 属性合法性，不合法则抛出异常
检测 ProviderConfig、ApplicationConfig 等核心配置类对象是否为空，若为空，则尝试从其他配置类对象中获取相应的实例。
检测并处理泛化服务和普通服务类
检测本地存根配置，并进行相应的处理
对 ApplicationConfig、RegistryConfig 等配置类进行检测，为空则尝试创建，若无法创建则抛出异常
*/
protected synchronized void doExport() {
    if (unexported) {
        throw new IllegalStateException("Already unexported!");
    }
    if (exported) {
        return;
    }
    exported = true;
    // 检测 interfaceName 是否合法
    if (interfaceName == null || interfaceName.length() == 0) {
        throw new IllegalStateException("interface not allow null!");
    }
    // 检测 provider 是否为空，为空则新建一个，并通过系统变量为其初始化
    checkDefault();

    // 下面几个 if 语句用于检测 provider、application 等核心配置类对象是否为空，
    // 若为空，则尝试从其他配置类对象中获取相应的实例。
    if (provider != null) {
        if (application == null) {
            application = provider.getApplication();
        }
        if (module == null) {
            module = provider.getModule();
        }
        if (registries == null) {...}
        if (monitor == null) {...}
        if (protocols == null) {...}
    }
    if (module != null) {
        if (registries == null) {
            registries = module.getRegistries();
        }
        if (monitor == null) {...}
    }
    if (application != null) {
        if (registries == null) {
            registries = application.getRegistries();
        }
        if (monitor == null) {...}
    }

    // 检测 ref 是否为泛化服务类型
    if (ref instanceof GenericService) {
        // 设置 interfaceClass 为 GenericService.class
        interfaceClass = GenericService.class;
        if (StringUtils.isEmpty(generic)) {
            // 设置 generic = "true"
            generic = Boolean.TRUE.toString();
        }
        
    // ref 非 GenericService 类型
    } else {
        try {
            interfaceClass = Class.forName(interfaceName, true, Thread.currentThread()
                    .getContextClassLoader());
        } catch (ClassNotFoundException e) {
            throw new IllegalStateException(e.getMessage(), e);
        }
        // 对 interfaceClass，以及 <dubbo:method> 标签中的必要字段进行检查
        checkInterfaceAndMethods(interfaceClass, methods);
        // 对 ref 合法性进行检测
        checkRef();
        // 设置 generic = "false"
        generic = Boolean.FALSE.toString();
    }

    // local 和 stub 在功能应该是一致的，用于配置本地存根
    if (local != null) {
        if ("true".equals(local)) {
            local = interfaceName + "Local";
        }
        Class<?> localClass;
        try {
            // 获取本地存根类
            localClass = ClassHelper.forNameWithThreadContextClassLoader(local);
        } catch (ClassNotFoundException e) {
            throw new IllegalStateException(e.getMessage(), e);
        }
        // 检测本地存根类是否可赋值给接口类，若不可赋值则会抛出异常，提醒使用者本地存根类类型不合法
        if (!interfaceClass.isAssignableFrom(localClass)) {
            throw new IllegalStateException("The local implementation class " + localClass.getName() + " not implement interface " + interfaceName);
        }
    }

    if (stub != null) {
        // 此处的代码和上一个 if 分支的代码基本一致，这里省略
    }

    // 检测各种对象是否为空，为空则新建，或者抛出异常
    checkApplication();
    checkRegistry();
    checkProtocol();
    appendProperties(this);
    checkStubAndMock(interfaceClass);
    if (path == null || path.length() == 0) {
        path = interfaceName;
    }

    // 导出服务
    doExportUrls();

    // ProviderModel 表示服务提供者模型，此对象中存储了与服务提供者相关的信息。
    // 比如服务的配置信息，服务实例等。每个被导出的服务对应一个 ProviderModel。
    // ApplicationModel 持有所有的 ProviderModel。
    ProviderModel providerModel = new ProviderModel(getUniqueServiceName(), this, ref);
    ApplicationModel.initProviderModel(getUniqueServiceName(), providerModel);
}
```

Dubbo 允许我们使用不同的协议导出服务，也允许我们向多个注册中心注册服务。Dubbo 在 doExportUrls 方法中对多协议，多注册中心进行了支持。

```java
private void doExportUrls() {
    // 加载注册中心链接
    List<URL> registryURLs = loadRegistries(true);
    // 遍历 protocols，并在每个协议下导出服务
    for (ProtocolConfig protocolConfig : protocols) {
        doExportUrlsFor1Protocol(protocolConfig, registryURLs);
    }
}

protected List<URL> loadRegistries(boolean provider) {
    // 检测是否存在注册中心配置类，不存在则抛出异常
    checkRegistry();
    List<URL> registryList = new ArrayList<URL>();
    if (registries != null && !registries.isEmpty()) {
        for (RegistryConfig config : registries) {
            String address = config.getAddress();
            if (address == null || address.length() == 0) {
                // 若 address 为空，则将其设为 0.0.0.0
                address = Constants.ANYHOST_VALUE;
            }

            // 从系统属性中加载注册中心地址
            String sysaddress = System.getProperty("dubbo.registry.address");
            if (sysaddress != null && sysaddress.length() > 0) {
                address = sysaddress;
            }
            // 检测 address 是否合法
            if (address.length() > 0 && !RegistryConfig.NO_AVAILABLE.equalsIgnoreCase(address)) {
                Map<String, String> map = new HashMap<String, String>();
                // 添加 ApplicationConfig 中的字段信息到 map 中
                appendParameters(map, application);
                // 添加 RegistryConfig 字段信息到 map 中
                appendParameters(map, config);
                
                // 添加 path、pid，protocol 等信息到 map 中
                map.put("path", RegistryService.class.getName());
                map.put("dubbo", Version.getProtocolVersion());
                map.put(Constants.TIMESTAMP_KEY, String.valueOf(System.currentTimeMillis()));
                if (ConfigUtils.getPid() > 0) {
                    map.put(Constants.PID_KEY, String.valueOf(ConfigUtils.getPid()));
                }
                if (!map.containsKey("protocol")) {
                    if (ExtensionLoader.getExtensionLoader(RegistryFactory.class).hasExtension("remote")) {
                        map.put("protocol", "remote");
                    } else {
                        map.put("protocol", "dubbo");
                    }
                }

                // 解析得到 URL 列表，address 可能包含多个注册中心 ip，
                // 因此解析得到的是一个 URL 列表
                List<URL> urls = UrlUtils.parseURLs(address, map);
                for (URL url : urls) {
                    url = url.addParameter(Constants.REGISTRY_KEY, url.getProtocol());
                    // 将 URL 协议头设置为 registry
                    url = url.setProtocol(Constants.REGISTRY_PROTOCOL);
                    // 通过判断条件，决定是否添加 url 到 registryList 中，条件如下：
                    // (服务提供者 && register = true 或 null) 
                    //    || (非服务提供者 && subscribe = true 或 null)
                    if ((provider && url.getParameter(Constants.REGISTER_KEY, true))
                            || (!provider && url.getParameter(Constants.SUBSCRIBE_KEY, true))) {
                        registryList.add(url);
                    }
                }
            }
        }
    }
    return registryList;
}

/** 
检测是否存在注册中心配置类，不存在则抛出异常
构建参数映射集合，也就是 map
构建注册中心链接列表
遍历链接列表，并根据条件决定是否将其添加到 registryList 中
*/
```

#### doExportUrlsFor1Protocol

```java
/**
将一些信息，比如版本、时间戳、方法名以及各种配置对象的字段信息放入到 map 中，map 中的内容将作为 URL 的查询字符串。构建好 map 后，紧接着是获取上下文路径、主机名以及端口号等信息。最后将 map 和主机名等数据传给 URL 构造方法创建 URL 对象。需要注意的是，这里出现的 URL 并非 java.net.URL，而是 com.alibaba.dubbo.common.URL。
*/
private void doExportUrlsFor1Protocol(ProtocolConfig protocolConfig, List<URL> registryURLs) {
    String name = protocolConfig.getName();
    // 如果协议名为空，或空串，则将协议名变量设置为 dubbo
    if (name == null || name.length() == 0) {
        name = "dubbo";
    }

    Map<String, String> map = new HashMap<String, String>();
    // 添加 side、版本、时间戳以及进程号等信息到 map 中
    map.put(Constants.SIDE_KEY, Constants.PROVIDER_SIDE);
    map.put(Constants.DUBBO_VERSION_KEY, Version.getProtocolVersion());
    map.put(Constants.TIMESTAMP_KEY, String.valueOf(System.currentTimeMillis()));
    if (ConfigUtils.getPid() > 0) {
        map.put(Constants.PID_KEY, String.valueOf(ConfigUtils.getPid()));
    }

    // 通过反射将对象的字段信息添加到 map 中
    appendParameters(map, application);
    appendParameters(map, module);
    appendParameters(map, provider, Constants.DEFAULT_KEY);
    appendParameters(map, protocolConfig);
    appendParameters(map, this);

    // methods 为 MethodConfig 集合，MethodConfig 中存储了 <dubbo:method> 标签的配置信息
    if (methods != null && !methods.isEmpty()) {
        // 这段代码用于添加 Callback 配置到 map 中，代码太长，待会单独分析
    }

    // 检测 generic 是否为 "true"，并根据检测结果向 map 中添加不同的信息
    if (ProtocolUtils.isGeneric(generic)) {
        map.put(Constants.GENERIC_KEY, generic);
        map.put(Constants.METHODS_KEY, Constants.ANY_VALUE);
    } else {
        String revision = Version.getVersion(interfaceClass, version);
        if (revision != null && revision.length() > 0) {
            map.put("revision", revision);
        }

        // 为接口生成包裹类 Wrapper，Wrapper 中包含了接口的详细信息，比如接口方法名数组，字段信息等
        String[] methods = Wrapper.getWrapper(interfaceClass).getMethodNames();
        // 添加方法名到 map 中，如果包含多个方法名，则用逗号隔开，比如 method = init,destroy
        if (methods.length == 0) {
            logger.warn("NO method found in service interface ...");
            map.put(Constants.METHODS_KEY, Constants.ANY_VALUE);
        } else {
            // 将逗号作为分隔符连接方法名，并将连接后的字符串放入 map 中
            map.put(Constants.METHODS_KEY, StringUtils.join(new HashSet<String>(Arrays.asList(methods)), ","));
        }
    }

    // 添加 token 到 map 中
    if (!ConfigUtils.isEmpty(token)) {
        if (ConfigUtils.isDefault(token)) {
            // 随机生成 token
            map.put(Constants.TOKEN_KEY, UUID.randomUUID().toString());
        } else {
            map.put(Constants.TOKEN_KEY, token);
        }
    }
    // 判断协议名是否为 injvm
    if (Constants.LOCAL_PROTOCOL.equals(protocolConfig.getName())) {
        protocolConfig.setRegister(false);
        map.put("notify", "false");
    }

    // 获取上下文路径
    String contextPath = protocolConfig.getContextpath();
    if ((contextPath == null || contextPath.length() == 0) && provider != null) {
        contextPath = provider.getContextpath();
    }

    // 获取 host 和 port
    String host = this.findConfigedHosts(protocolConfig, registryURLs, map);
    Integer port = this.findConfigedPorts(protocolConfig, name, map);
    // 组装 URL
    URL url = new URL(name, host, port, (contextPath == null || contextPath.length() == 0 ? "" : contextPath + "/") + path, map);
    
    // 省略无关代码
}

// <dubbo:method>
/**
// 获取 ArgumentConfig 列表
for (遍历 ArgumentConfig 列表) {
    if (type 不为 null，也不为空串) {    // 分支1
        1. 通过反射获取 interfaceClass 的方法列表
        for (遍历方法列表) {
            1. 比对方法名，查找目标方法
        	2. 通过反射获取目标方法的参数类型数组 argtypes
            if (index != -1) {    // 分支2
                1. 从 argtypes 数组中获取下标 index 处的元素 argType
                2. 检测 argType 的名称与 ArgumentConfig 中的 type 属性是否一致
                3. 添加 ArgumentConfig 字段信息到 map 中，或抛出异常
            } else {    // 分支3
                1. 遍历参数类型数组 argtypes，查找 argument.type 类型的参数
                2. 添加 ArgumentConfig 字段信息到 map 中
            }
        }
    } else if (index != -1) {    // 分支4
		1. 添加 ArgumentConfig 字段信息到 map 中
    }
}
*/
private void doExportUrlsFor1Protocol(ProtocolConfig protocolConfig, List<URL> registryURLs) {
    // ...

    // methods 为 MethodConfig 集合，MethodConfig 中存储了 <dubbo:method> 标签的配置信息
    if (methods != null && !methods.isEmpty()) {
        for (MethodConfig method : methods) {
            // 添加 MethodConfig 对象的字段信息到 map 中，键 = 方法名.属性名。
            // 比如存储 <dubbo:method name="sayHello" retries="2"> 对应的 MethodConfig，
            // 键 = sayHello.retries，map = {"sayHello.retries": 2, "xxx": "yyy"}
            appendParameters(map, method, method.getName());

            String retryKey = method.getName() + ".retry";
            if (map.containsKey(retryKey)) {
                String retryValue = map.remove(retryKey);
                // 检测 MethodConfig retry 是否为 false，若是，则设置重试次数为0
                if ("false".equals(retryValue)) {
                    map.put(method.getName() + ".retries", "0");
                }
            }
            
            // 获取 ArgumentConfig 列表
            List<ArgumentConfig> arguments = method.getArguments();
            if (arguments != null && !arguments.isEmpty()) {
                for (ArgumentConfig argument : arguments) {
                    // 检测 type 属性是否为空，或者空串（分支1 ⭐️）
                    if (argument.getType() != null && argument.getType().length() > 0) {
                        Method[] methods = interfaceClass.getMethods();
                        if (methods != null && methods.length > 0) {
                            for (int i = 0; i < methods.length; i++) {
                                String methodName = methods[i].getName();
                                // 比对方法名，查找目标方法
                                if (methodName.equals(method.getName())) {
                                    Class<?>[] argtypes = methods[i].getParameterTypes();
                                    if (argument.getIndex() != -1) {
                                        // 检测 ArgumentConfig 中的 type 属性与方法参数列表
                                        // 中的参数名称是否一致，不一致则抛出异常(分支2 ⭐️)
                                        if (argtypes[argument.getIndex()].getName().equals(argument.getType())) {
                                            // 添加 ArgumentConfig 字段信息到 map 中，
                                            // 键前缀 = 方法名.index，比如:
                                            // map = {"sayHello.3": true}
                                            appendParameters(map, argument, method.getName() + "." + argument.getIndex());
                                        } else {
                                            throw new IllegalArgumentException("argument config error: ...");
                                        }
                                    } else {    // 分支3 ⭐️
                                        for (int j = 0; j < argtypes.length; j++) {
                                            Class<?> argclazz = argtypes[j];
                                            // 从参数类型列表中查找类型名称为 argument.type 的参数
                                            if (argclazz.getName().equals(argument.getType())) {
                                                appendParameters(map, argument, method.getName() + "." + j);
                                                if (argument.getIndex() != -1 && argument.getIndex() != j) {
                                                    throw new IllegalArgumentException("argument config error: ...");
                                                }
                                            }
                                        }
                                    }
                                }
                            }
                        }

                    // 用户未配置 type 属性，但配置了 index 属性，且 index != -1
                    } else if (argument.getIndex() != -1) {    // 分支4 ⭐️
                        // 添加 ArgumentConfig 字段信息到 map 中
                        appendParameters(map, argument, method.getName() + "." + argument.getIndex());
                    } else {
                        throw new IllegalArgumentException("argument config must set index or type");
                    }
                }
            }
        }
    }

    // ...
}
```

#### 导出服务

```java
/**
scope = none，不导出服务
scope != remote，导出到本地
scope != local，导出到远程
*/
private void doExportUrlsFor1Protocol(ProtocolConfig protocolConfig, List<URL> registryURLs) {
    
    // 省略无关代码
    
    if (ExtensionLoader.getExtensionLoader(ConfiguratorFactory.class)
            .hasExtension(url.getProtocol())) {
        // 加载 ConfiguratorFactory，并生成 Configurator 实例，然后通过实例配置 url
        url = ExtensionLoader.getExtensionLoader(ConfiguratorFactory.class)
                .getExtension(url.getProtocol()).getConfigurator(url).configure(url);
    }

    String scope = url.getParameter(Constants.SCOPE_KEY);
    // 如果 scope = none，则什么都不做
    if (!Constants.SCOPE_NONE.toString().equalsIgnoreCase(scope)) {
        // scope != remote，导出到本地
        if (!Constants.SCOPE_REMOTE.toString().equalsIgnoreCase(scope)) {
            exportLocal(url);
        }

        // scope != local，导出到远程
        if (!Constants.SCOPE_LOCAL.toString().equalsIgnoreCase(scope)) {
            if (registryURLs != null && !registryURLs.isEmpty()) {
                for (URL registryURL : registryURLs) {
                    url = url.addParameterIfAbsent(Constants.DYNAMIC_KEY, registryURL.getParameter(Constants.DYNAMIC_KEY));
                    // 加载监视器链接
                    URL monitorUrl = loadMonitor(registryURL);
                    if (monitorUrl != null) {
                        // 将监视器链接作为参数添加到 url 中
                        url = url.addParameterAndEncoded(Constants.MONITOR_KEY, monitorUrl.toFullString());
                    }

                    String proxy = url.getParameter(Constants.PROXY_KEY);
                    if (StringUtils.isNotEmpty(proxy)) {
                        registryURL = registryURL.addParameter(Constants.PROXY_KEY, proxy);
                    }

                    // 为服务提供类(ref)生成 Invoker
                    Invoker<?> invoker = proxyFactory.getInvoker(ref, (Class) interfaceClass, registryURL.addParameterAndEncoded(Constants.EXPORT_KEY, url.toFullString()));
                    // DelegateProviderMetaDataInvoker 用于持有 Invoker 和 ServiceConfig
                    DelegateProviderMetaDataInvoker wrapperInvoker = new DelegateProviderMetaDataInvoker(invoker, this);

                    // 导出服务，并生成 Exporter
                    Exporter<?> exporter = protocol.export(wrapperInvoker);
                    exporters.add(exporter);
                }
                
            // 不存在注册中心，仅导出服务
            } else {
                Invoker<?> invoker = proxyFactory.getInvoker(ref, (Class) interfaceClass, url);
                DelegateProviderMetaDataInvoker wrapperInvoker = new DelegateProviderMetaDataInvoker(invoker, this);

                Exporter<?> exporter = protocol.export(wrapperInvoker);
                exporters.add(exporter);
            }
        }
    }
    this.urls.add(url);
}
```



#### Invoker创建

Dubbo 默认的 ProxyFactory 实现类是 JavassistProxyFactory

```java
public <T> Invoker<T> getInvoker(T proxy, Class<T> type, URL url) {
	// 为目标类创建 Wrapper
    final Wrapper wrapper = Wrapper.getWrapper(proxy.getClass().getName().indexOf('$') < 0 ? proxy.getClass() : type);
    // 创建匿名 Invoker 类对象，并实现 doInvoke 方法。
    return new AbstractProxyInvoker<T>(proxy, type, url) {
        @Override
        protected Object doInvoke(T proxy, String methodName,
                                  Class<?>[] parameterTypes,
                                  Object[] arguments) throws Throwable {
			// 调用 Wrapper 的 invokeMethod 方法，invokeMethod 最终会调用目标方法
            return wrapper.invokeMethod(proxy, methodName, parameterTypes, arguments);
        }
    };
}
```

Wrapper 用于“包裹”目标类，Wrapper 是一个抽象类，仅可通过 getWrapper(Class) 方法创建子类。在创建 Wrapper 子类的过程中，子类代码生成逻辑会对 getWrapper 方法传入的 Class 对象进行解析，拿到诸如类方法，类成员变量等信息。以及生成 invokeMethod 方法代码和其他一些方法代码。代码生成完毕后，通过 Javassist 生成 Class 对象，最后再通过反射创建 Wrapper 实例。

```java
public static Wrapper getWrapper(Class<?> c) {	
    while (ClassGenerator.isDynamicClass(c))
        c = c.getSuperclass();

    if (c == Object.class)
        return OBJECT_WRAPPER;

    // 从缓存中获取 Wrapper 实例
    Wrapper ret = WRAPPER_MAP.get(c);
    if (ret == null) {
        // 缓存未命中，创建 Wrapper
        ret = makeWrapper(c);
        // 写入缓存
        WRAPPER_MAP.put(c, ret);
    }
    return ret;
}

//--------------------------------------------
private static Wrapper makeWrapper(Class<?> c) {
    // 检测 c 是否为基本类型，若是则抛出异常
    if (c.isPrimitive())
        throw new IllegalArgumentException("Can not create wrapper for primitive type: " + c);

    String name = c.getName();
    ClassLoader cl = ClassHelper.getClassLoader(c);

    // c1 用于存储 setPropertyValue 方法代码
    StringBuilder c1 = new StringBuilder("public void setPropertyValue(Object o, String n, Object v){ ");
    // c2 用于存储 getPropertyValue 方法代码
    StringBuilder c2 = new StringBuilder("public Object getPropertyValue(Object o, String n){ ");
    // c3 用于存储 invokeMethod 方法代码
    StringBuilder c3 = new StringBuilder("public Object invokeMethod(Object o, String n, Class[] p, Object[] v) throws " + InvocationTargetException.class.getName() + "{ ");

    // 生成类型转换代码及异常捕捉代码，比如：
    //   DemoService w; try { w = ((DemoServcie) $1); }}catch(Throwable e){ throw new IllegalArgumentException(e); }
    c1.append(name).append(" w; try{ w = ((").append(name).append(")$1); }catch(Throwable e){ throw new IllegalArgumentException(e); }");
    c2.append(name).append(" w; try{ w = ((").append(name).append(")$1); }catch(Throwable e){ throw new IllegalArgumentException(e); }");
    c3.append(name).append(" w; try{ w = ((").append(name).append(")$1); }catch(Throwable e){ throw new IllegalArgumentException(e); }");

    // pts 用于存储成员变量名和类型
    Map<String, Class<?>> pts = new HashMap<String, Class<?>>();
    // ms 用于存储方法描述信息（可理解为方法签名）及 Method 实例
    Map<String, Method> ms = new LinkedHashMap<String, Method>();
    // mns 为方法名列表
    List<String> mns = new ArrayList<String>();
    // dmns 用于存储“定义在当前类中的方法”的名称
    List<String> dmns = new ArrayList<String>();

    // --------------------------------✨ 分割线1 ✨-------------------------------------

    // 获取 public 访问级别的字段，并为所有字段生成条件判断语句
    for (Field f : c.getFields()) {
        String fn = f.getName();
        Class<?> ft = f.getType();
        if (Modifier.isStatic(f.getModifiers()) || Modifier.isTransient(f.getModifiers()))
            // 忽略关键字 static 或 transient 修饰的变量
            continue;

        // 生成条件判断及赋值语句，比如：
        // if( $2.equals("name") ) { w.name = (java.lang.String) $3; return;}
        // if( $2.equals("age") ) { w.age = ((Number) $3).intValue(); return;}
        c1.append(" if( $2.equals(\"").append(fn).append("\") ){ w.").append(fn).append("=").append(arg(ft, "$3")).append("; return; }");

        // 生成条件判断及返回语句，比如：
        // if( $2.equals("name") ) { return ($w)w.name; }
        c2.append(" if( $2.equals(\"").append(fn).append("\") ){ return ($w)w.").append(fn).append("; }");

        // 存储 <字段名, 字段类型> 键值对到 pts 中
        pts.put(fn, ft);
    }

    // --------------------------------✨ 分割线2 ✨-------------------------------------

    Method[] methods = c.getMethods();
    // 检测 c 中是否包含在当前类中声明的方法
    boolean hasMethod = hasMethods(methods);
    if (hasMethod) {
        c3.append(" try{");
    }
    for (Method m : methods) {
        if (m.getDeclaringClass() == Object.class)
            // 忽略 Object 中定义的方法
            continue;

        String mn = m.getName();
        // 生成方法名判断语句，比如：
        // if ( "sayHello".equals( $2 )
        c3.append(" if( \"").append(mn).append("\".equals( $2 ) ");
        int len = m.getParameterTypes().length;
        // 生成“运行时传入的参数数量与方法参数列表长度”判断语句，比如：
        // && $3.length == 2
        c3.append(" && ").append(" $3.length == ").append(len);

        boolean override = false;
        for (Method m2 : methods) {
            // 检测方法是否存在重载情况，条件为：方法对象不同 && 方法名相同
            if (m != m2 && m.getName().equals(m2.getName())) {
                override = true;
                break;
            }
        }
        // 对重载方法进行处理，考虑下面的方法：
        //    1. void sayHello(Integer, String)
        //    2. void sayHello(Integer, Integer)
        // 方法名相同，参数列表长度也相同，因此不能仅通过这两项判断两个方法是否相等。
        // 需要进一步判断方法的参数类型
        if (override) {
            if (len > 0) {
                for (int l = 0; l < len; l++) {
                    // 生成参数类型进行检测代码，比如：
                    // && $3[0].getName().equals("java.lang.Integer") 
                    //    && $3[1].getName().equals("java.lang.String")
                    c3.append(" && ").append(" $3[").append(l).append("].getName().equals(\"")
                            .append(m.getParameterTypes()[l].getName()).append("\")");
                }
            }
        }

        // 添加 ) {，完成方法判断语句，此时生成的代码可能如下（已格式化）：
        // if ("sayHello".equals($2) 
        //     && $3.length == 2
        //     && $3[0].getName().equals("java.lang.Integer") 
        //     && $3[1].getName().equals("java.lang.String")) {
        c3.append(" ) { ");

        // 根据返回值类型生成目标方法调用语句
        if (m.getReturnType() == Void.TYPE)
            // w.sayHello((java.lang.Integer)$4[0], (java.lang.String)$4[1]); return null;
            c3.append(" w.").append(mn).append('(').append(args(m.getParameterTypes(), "$4")).append(");").append(" return null;");
        else
            // return w.sayHello((java.lang.Integer)$4[0], (java.lang.String)$4[1]);
            c3.append(" return ($w)w.").append(mn).append('(').append(args(m.getParameterTypes(), "$4")).append(");");

        // 添加 }, 生成的代码形如（已格式化）：
        // if ("sayHello".equals($2) 
        //     && $3.length == 2
        //     && $3[0].getName().equals("java.lang.Integer") 
        //     && $3[1].getName().equals("java.lang.String")) {
        //
        //     w.sayHello((java.lang.Integer)$4[0], (java.lang.String)$4[1]); 
        //     return null;
        // }
        c3.append(" }");

        // 添加方法名到 mns 集合中
        mns.add(mn);
        // 检测当前方法是否在 c 中被声明的
        if (m.getDeclaringClass() == c)
            // 若是，则将当前方法名添加到 dmns 中
            dmns.add(mn);
        ms.put(ReflectUtils.getDesc(m), m);
    }
    if (hasMethod) {
        // 添加异常捕捉语句
        c3.append(" } catch(Throwable e) { ");
        c3.append("     throw new java.lang.reflect.InvocationTargetException(e); ");
        c3.append(" }");
    }

    // 添加 NoSuchMethodException 异常抛出代码
    c3.append(" throw new " + NoSuchMethodException.class.getName() + "(\"Not found method \\\"\"+$2+\"\\\" in class " + c.getName() + ".\"); }");

    // --------------------------------✨ 分割线3 ✨-------------------------------------

    Matcher matcher;
    // 处理 get/set 方法
    for (Map.Entry<String, Method> entry : ms.entrySet()) {
        String md = entry.getKey();
        Method method = (Method) entry.getValue();
        // 匹配以 get 开头的方法
        if ((matcher = ReflectUtils.GETTER_METHOD_DESC_PATTERN.matcher(md)).matches()) {
            // 获取属性名
            String pn = propertyName(matcher.group(1));
            // 生成属性判断以及返回语句，示例如下：
            // if( $2.equals("name") ) { return ($w).w.getName(); }
            c2.append(" if( $2.equals(\"").append(pn).append("\") ){ return ($w)w.").append(method.getName()).append("(); }");
            pts.put(pn, method.getReturnType());

        // 匹配以 is/has/can 开头的方法
        } else if ((matcher = ReflectUtils.IS_HAS_CAN_METHOD_DESC_PATTERN.matcher(md)).matches()) {
            String pn = propertyName(matcher.group(1));
            // 生成属性判断以及返回语句，示例如下：
            // if( $2.equals("dream") ) { return ($w).w.hasDream(); }
            c2.append(" if( $2.equals(\"").append(pn).append("\") ){ return ($w)w.").append(method.getName()).append("(); }");
            pts.put(pn, method.getReturnType());

        // 匹配以 set 开头的方法
        } else if ((matcher = ReflectUtils.SETTER_METHOD_DESC_PATTERN.matcher(md)).matches()) {
            Class<?> pt = method.getParameterTypes()[0];
            String pn = propertyName(matcher.group(1));
            // 生成属性判断以及 setter 调用语句，示例如下：
            // if( $2.equals("name") ) { w.setName((java.lang.String)$3); return; }
            c1.append(" if( $2.equals(\"").append(pn).append("\") ){ w.").append(method.getName()).append("(").append(arg(pt, "$3")).append("); return; }");
            pts.put(pn, pt);
        }
    }

    // 添加 NoSuchPropertyException 异常抛出代码
    c1.append(" throw new " + NoSuchPropertyException.class.getName() + "(\"Not found property \\\"\"+$2+\"\\\" filed or setter method in class " + c.getName() + ".\"); }");
    c2.append(" throw new " + NoSuchPropertyException.class.getName() + "(\"Not found property \\\"\"+$2+\"\\\" filed or setter method in class " + c.getName() + ".\"); }");

    // --------------------------------✨ 分割线4 ✨-------------------------------------

    long id = WRAPPER_CLASS_COUNTER.getAndIncrement();
    // 创建类生成器
    ClassGenerator cc = ClassGenerator.newInstance(cl);
    // 设置类名及超类
    cc.setClassName((Modifier.isPublic(c.getModifiers()) ? Wrapper.class.getName() : c.getName() + "$sw") + id);
    cc.setSuperClass(Wrapper.class);

    // 添加默认构造方法
    cc.addDefaultConstructor();

    // 添加字段
    cc.addField("public static String[] pns;");
    cc.addField("public static " + Map.class.getName() + " pts;");
    cc.addField("public static String[] mns;");
    cc.addField("public static String[] dmns;");
    for (int i = 0, len = ms.size(); i < len; i++)
        cc.addField("public static Class[] mts" + i + ";");

    // 添加方法代码
    cc.addMethod("public String[] getPropertyNames(){ return pns; }");
    cc.addMethod("public boolean hasProperty(String n){ return pts.containsKey($1); }");
    cc.addMethod("public Class getPropertyType(String n){ return (Class)pts.get($1); }");
    cc.addMethod("public String[] getMethodNames(){ return mns; }");
    cc.addMethod("public String[] getDeclaredMethodNames(){ return dmns; }");
    cc.addMethod(c1.toString());
    cc.addMethod(c2.toString());
    cc.addMethod(c3.toString());

    try {
        // 生成类
        Class<?> wc = cc.toClass();
        
        // 设置字段值
        wc.getField("pts").set(null, pts);
        wc.getField("pns").set(null, pts.keySet().toArray(new String[0]));
        wc.getField("mns").set(null, mns.toArray(new String[0]));
        wc.getField("dmns").set(null, dmns.toArray(new String[0]));
        int ix = 0;
        for (Method m : ms.values())
            wc.getField("mts" + ix++).set(null, m.getParameterTypes());

        // 创建 Wrapper 实例
        return (Wrapper) wc.newInstance();
    } catch (RuntimeException e) {
        throw e;
    } catch (Throwable e) {
        throw new RuntimeException(e.getMessage(), e);
    } finally {
        cc.release();
        ms.clear();
        mns.clear();
        dmns.clear();
    }
}
```



#### 导出服务到本地

```java
/**
根据 URL 协议头决定是否导出服务。若需导出，则创建一个新的 URL 并将协议头、主机名以及端口设置成新的值。然后创建 Invoker，并调用 InjvmProtocol 的 export 方法导出服务
*/
private void exportLocal(URL url) {
    // 如果 URL 的协议头等于 injvm，说明已经导出到本地了，无需再次导出
    if (!Constants.LOCAL_PROTOCOL.equalsIgnoreCase(url.getProtocol())) {
        URL local = URL.valueOf(url.toFullString())
            .setProtocol(Constants.LOCAL_PROTOCOL)    // 设置协议头为 injvm
            .setHost(LOCALHOST)
            .setPort(0);
        ServiceClassHolder.getInstance().pushServiceClass(getServiceClass(ref));
        // 创建 Invoker，并导出服务，这里的 protocol 会在运行时调用 InjvmProtocol 的 export 方法
        Exporter<?> exporter = protocol.export(
            proxyFactory.getInvoker(ref, (Class) interfaceClass, local));
        exporters.add(exporter);
    }
}

public <T> Exporter<T> export(Invoker<T> invoker) throws RpcException {
    // 创建 InjvmExporter
    return new InjvmExporter<T>(invoker, invoker.getUrl().getServiceKey(), exporterMap);
}
```



#### 导出服务到远程

##### RegistryProtocol

```java
// RegistryProtocol
/**
1. 调用 doLocalExport 导出服务
2. 向注册中心注册服务
3. 向注册中心进行订阅 override 数据
4. 创建并返回 DestroyableExporter
*/
public <T> Exporter<T> export(final Invoker<T> originInvoker) throws RpcException {
    // 导出服务
    final ExporterChangeableWrapper<T> exporter = doLocalExport(originInvoker);

    // 获取注册中心 URL，以 zookeeper 注册中心为例，得到的示例 URL 如下：
    // zookeeper://127.0.0.1:2181/com.alibaba.dubbo.registry.RegistryService?application=demo-provider&dubbo=2.0.2&export=dubbo%3A%2F%2F172.17.48.52%3A20880%2Fcom.alibaba.dubbo.demo.DemoService%3Fanyhost%3Dtrue%26application%3Ddemo-provider
    URL registryUrl = getRegistryUrl(originInvoker);

    // 根据 URL 加载 Registry 实现类，比如 ZookeeperRegistry
    final Registry registry = getRegistry(originInvoker);
    
    // 获取已注册的服务提供者 URL，比如：
    // dubbo://172.17.48.52:20880/com.alibaba.dubbo.demo.DemoService?anyhost=true&application=demo-provider&dubbo=2.0.2&generic=false&interface=com.alibaba.dubbo.demo.DemoService&methods=sayHello
    final URL registeredProviderUrl = getRegisteredProviderUrl(originInvoker);

    // 获取 register 参数
    boolean register = registeredProviderUrl.getParameter("register", true);

    // 向服务提供者与消费者注册表中注册服务提供者
    ProviderConsumerRegTable.registerProvider(originInvoker, registryUrl, registeredProviderUrl);

    // 根据 register 的值决定是否注册服务
    if (register) {
        // 向注册中心注册服务
        register(registryUrl, registeredProviderUrl);
        ProviderConsumerRegTable.getProviderWrapper(originInvoker).setReg(true);
    }

    // 获取订阅 URL，比如：
    // provider://172.17.48.52:20880/com.alibaba.dubbo.demo.DemoService?category=configurators&check=false&anyhost=true&application=demo-provider&dubbo=2.0.2&generic=false&interface=com.alibaba.dubbo.demo.DemoService&methods=sayHello
    final URL overrideSubscribeUrl = getSubscribedOverrideUrl(registeredProviderUrl);
    // 创建监听器
    final OverrideListener overrideSubscribeListener = new OverrideListener(overrideSubscribeUrl, originInvoker);
    overrideListeners.put(overrideSubscribeUrl, overrideSubscribeListener);
    // 向注册中心进行订阅 override 数据
    registry.subscribe(overrideSubscribeUrl, overrideSubscribeListener);
    // 创建并返回 DestroyableExporter
    return new DestroyableExporter<T>(exporter, originInvoker, overrideSubscribeUrl, registeredProviderUrl);
}
```

##### doLocalExport

```java
private <T> ExporterChangeableWrapper<T> doLocalExport(final Invoker<T> originInvoker) {
    String key = getCacheKey(originInvoker);
    // 访问缓存
    ExporterChangeableWrapper<T> exporter = (ExporterChangeableWrapper<T>) bounds.get(key);
    if (exporter == null) {
        synchronized (bounds) {
            exporter = (ExporterChangeableWrapper<T>) bounds.get(key);
            if (exporter == null) {
                // 创建 Invoker 为委托类对象
                final Invoker<?> invokerDelegete = new InvokerDelegete<T>(originInvoker, getProviderUrl(originInvoker));
                // 调用 protocol 的 export 方法导出服务
                exporter = new ExporterChangeableWrapper<T>((Exporter<T>) protocol.export(invokerDelegete), originInvoker);
                
                // 写缓存
                bounds.put(key, exporter);
            }
        }
    }
    return exporter;
}
```



##### DubboProtocol 

```java
public <T> Exporter<T> export(Invoker<T> invoker) throws RpcException {
    URL url = invoker.getUrl();

    // 获取服务标识，理解成服务坐标也行。由服务组名，服务名，服务版本号以及端口组成。比如：
    // demoGroup/com.alibaba.dubbo.demo.DemoService:1.0.1:20880
    String key = serviceKey(url);
    // 创建 DubboExporter
    DubboExporter<T> exporter = new DubboExporter<T>(invoker, key, exporterMap);
    // 将 <key, exporter> 键值对放入缓存中
    exporterMap.put(key, exporter);

    // 本地存根相关代码
    Boolean isStubSupportEvent = url.getParameter(Constants.STUB_EVENT_KEY, Constants.DEFAULT_STUB_EVENT);
    Boolean isCallbackservice = url.getParameter(Constants.IS_CALLBACK_SERVICE, false);
    if (isStubSupportEvent && !isCallbackservice) {
        String stubServiceMethods = url.getParameter(Constants.STUB_EVENT_METHODS_KEY);
        if (stubServiceMethods == null || stubServiceMethods.length() == 0) {
            // 省略日志打印代码
        } else {
            stubServiceMethodsMap.put(url.getServiceKey(), stubServiceMethods);
        }
    }

    // 启动服务器
    openServer(url);
    // 优化序列化
    optimizeSerialization(url);
    return exporter;
}

```

###### openServer

```java
private void openServer(URL url) {
    // 获取 host:port，并将其作为服务器实例的 key，用于标识当前的服务器实例
    String key = url.getAddress();
    boolean isServer = url.getParameter(Constants.IS_SERVER_KEY, true);
    if (isServer) {
        // 访问缓存
        ExchangeServer server = serverMap.get(key);
        if (server == null) {
            // 创建服务器实例
            serverMap.put(key, createServer(url));
        } else {
            // 服务器已创建，则根据 url 中的配置重置服务器
            server.reset(url);
        }
    }
}
```



###### createServer

```java
/**
createServer 包含三个核心的逻辑。第一是检测是否存在 server 参数所代表的 Transporter 拓展，不存在则抛出异常。第二是创建服务器实例。第三是检测是否支持 client 参数所表示的 Transporter 拓展，不存在也是抛出异常。
*/
private ExchangeServer createServer(URL url) {
    url = url.addParameterIfAbsent(Constants.CHANNEL_READONLYEVENT_SENT_KEY,
    // 添加心跳检测配置到 url 中
    url = url.addParameterIfAbsent(Constants.HEARTBEAT_KEY, String.valueOf(Constants.DEFAULT_HEARTBEAT));
	// 获取 server 参数，默认为 netty
    String str = url.getParameter(Constants.SERVER_KEY, Constants.DEFAULT_REMOTING_SERVER);

	// 通过 SPI 检测是否存在 server 参数所代表的 Transporter 拓展，不存在则抛出异常
    if (str != null && str.length() > 0 && !ExtensionLoader.getExtensionLoader(Transporter.class).hasExtension(str))
        throw new RpcException("Unsupported server type: " + str + ", url: " + url);

    // 添加编码解码器参数
    url = url.addParameter(Constants.CODEC_KEY, DubboCodec.NAME);
    ExchangeServer server;
    try {
        // 创建 ExchangeServer
        server = Exchangers.bind(url, requestHandler);
    } catch (RemotingException e) {
        throw new RpcException("Fail to start server...");
    }
                                   
	// 获取 client 参数，可指定 netty，mina
    str = url.getParameter(Constants.CLIENT_KEY);
    if (str != null && str.length() > 0) {
        // 获取所有的 Transporter 实现类名称集合，比如 supportedTypes = [netty, mina]
        Set<String> supportedTypes = ExtensionLoader.getExtensionLoader(Transporter.class).getSupportedExtensions();
        // 检测当前 Dubbo 所支持的 Transporter 实现类名称列表中，
        // 是否包含 client 所表示的 Transporter，若不包含，则抛出异常
        if (!supportedTypes.contains(str)) {
            throw new RpcException("Unsupported client type...");
        }
    }
    return server;
}
```



```java
public static ExchangeServer bind(URL url, ExchangeHandler handler) throws RemotingException {
    if (url == null) {
        throw new IllegalArgumentException("url == null");
    }
    if (handler == null) {
        throw new IllegalArgumentException("handler == null");
    }
    url = url.addParameterIfAbsent(Constants.CODEC_KEY, "exchange");
    // 获取 Exchanger，默认为 HeaderExchanger。
    // 紧接着调用 HeaderExchanger 的 bind 方法创建 ExchangeServer 实例
    return getExchanger(url).bind(url, handler);
}

//-------HeaderExchanger#bind---------
public ExchangeServer bind(URL url, ExchangeHandler handler) throws RemotingException {
	// 创建 HeaderExchangeServer 实例，该方法包含了多个逻辑，分别如下：
	//   1. new HeaderExchangeHandler(handler)
	//	 2. new DecodeHandler(new HeaderExchangeHandler(handler))
	//   3. Transporters.bind(url, new DecodeHandler(new HeaderExchangeHandler(handler)))
    return new HeaderExchangeServer(Transporters.bind(url, new DecodeHandler(new HeaderExchangeHandler(handler))));
}


// ---Transporters#bind----
/**
getTransporter() 方法获取的 Transporter 是在运行时动态创建的，类名为 TransporterAdaptive，也就是自适应拓展类。TransporterAdaptive 会在运行时根据传入的 URL 参数决定加载什么类型的 Transporter，默认为 NettyTransporter。
*/
public static Server bind(URL url, ChannelHandler... handlers) throws RemotingException {
    if (url == null) {
        throw new IllegalArgumentException("url == null");
    }
    if (handlers == null || handlers.length == 0) {
        throw new IllegalArgumentException("handlers == null");
    }
    ChannelHandler handler;
    if (handlers.length == 1) {
        handler = handlers[0];
    } else {
    	// 如果 handlers 元素数量大于1，则创建 ChannelHandler 分发器
        handler = new ChannelHandlerDispatcher(handlers);
    }
    // 获取自适应 Transporter 实例，并调用实例方法
    return getTransporter().bind(url, handler);
}

public Server bind(URL url, ChannelHandler listener) throws RemotingException {
	// 创建 NettyServer
	return new NettyServer(url, listener);
}
```



###### NettyServer 

```java
public class NettyServer extends AbstractServer implements Server {
    public NettyServer(URL url, ChannelHandler handler) throws RemotingException {
        // 调用父类构造方法
        super(url, ChannelHandlers.wrap(handler, ExecutorUtil.setThreadName(url, SERVER_THREAD_POOL_NAME)));
    }
}


public abstract class AbstractServer extends AbstractEndpoint implements Server {
    public AbstractServer(URL url, ChannelHandler handler) throws RemotingException {
        // 调用父类构造方法，这里就不用跟进去了，没什么复杂逻辑
        super(url, handler);
        localAddress = getUrl().toInetSocketAddress();

        // 获取 ip 和端口
        String bindIp = getUrl().getParameter(Constants.BIND_IP_KEY, getUrl().getHost());
        int bindPort = getUrl().getParameter(Constants.BIND_PORT_KEY, getUrl().getPort());
        if (url.getParameter(Constants.ANYHOST_KEY, false) || NetUtils.isInvalidLocalHost(bindIp)) {
            // 设置 ip 为 0.0.0.0
            bindIp = NetUtils.ANYHOST;
        }
        bindAddress = new InetSocketAddress(bindIp, bindPort);
        // 获取最大可接受连接数
        this.accepts = url.getParameter(Constants.ACCEPTS_KEY, Constants.DEFAULT_ACCEPTS);
        this.idleTimeout = url.getParameter(Constants.IDLE_TIMEOUT_KEY, Constants.DEFAULT_IDLE_TIMEOUT);
        try {
            // 调用模板方法 doOpen 启动服务器
            doOpen();
        } catch (Throwable t) {
            throw new RemotingException("Failed to bind ");
        }

        DataStore dataStore = ExtensionLoader.getExtensionLoader(DataStore.class).getDefaultExtension();
        executor = (ExecutorService) dataStore.get(Constants.EXECUTOR_SERVICE_COMPONENT_KEY, Integer.toString(url.getPort()));
    }
    
    protected abstract void doOpen() throws Throwable;

    protected abstract void doClose() throws Throwable;
}
```

###### doOpen

```java
protected void doOpen() throws Throwable {
    NettyHelper.setNettyLoggerFactory();
    // 创建 boss 和 worker 线程池
    ExecutorService boss = Executors.newCachedThreadPool(new NamedThreadFactory("NettyServerBoss", true));
    ExecutorService worker = Executors.newCachedThreadPool(new NamedThreadFactory("NettyServerWorker", true));
    ChannelFactory channelFactory = new NioServerSocketChannelFactory(boss, worker, getUrl().getPositiveParameter(Constants.IO_THREADS_KEY, Constants.DEFAULT_IO_THREADS));
    
    // 创建 ServerBootstrap
    bootstrap = new ServerBootstrap(channelFactory);

    final NettyHandler nettyHandler = new NettyHandler(getUrl(), this);
    channels = nettyHandler.getChannels();
    bootstrap.setOption("child.tcpNoDelay", true);
    // 设置 PipelineFactory
    bootstrap.setPipelineFactory(new ChannelPipelineFactory() {
        @Override
        public ChannelPipeline getPipeline() {
            NettyCodecAdapter adapter = new NettyCodecAdapter(getCodec(), getUrl(), NettyServer.this);
            ChannelPipeline pipeline = Channels.pipeline();
            pipeline.addLast("decoder", adapter.getDecoder());
            pipeline.addLast("encoder", adapter.getEncoder());
            pipeline.addLast("handler", nettyHandler);
            return pipeline;
        }
    });
    // 绑定到指定的 ip 和端口上
    channel = bootstrap.bind(getBindAddress());
}
```



### 服务注册

RegistryProtocol#export

```java
public <T> Exporter<T> export(final Invoker<T> originInvoker) throws RpcException {
    
    // ${导出服务}
    
    // 省略其他代码
    
    boolean register = registeredProviderUrl.getParameter("register", true);
    if (register) {
        // 注册服务
        register(registryUrl, registeredProviderUrl);
        ProviderConsumerRegTable.getProviderWrapper(originInvoker).setReg(true);
    }
    
    final URL overrideSubscribeUrl = getSubscribedOverrideUrl(registeredProviderUrl);
    final OverrideListener overrideSubscribeListener = new OverrideListener(overrideSubscribeUrl, originInvoker);
    overrideListeners.put(overrideSubscribeUrl, overrideSubscribeListener);
    // 订阅 override 数据
    registry.subscribe(overrideSubscribeUrl, overrideSubscribeListener);

    // 省略部分代码
}

/**
register 方法包含两步操作，第一步是获取注册中心实例，第二步是向注册中心注册服务。
*/
public void register(URL registryUrl, URL registedProviderUrl) {
    // 获取 Registry
    Registry registry = registryFactory.getRegistry(registryUrl);
    // 注册服务
    registry.register(registedProviderUrl);
}
```



###### AbstractRegistryFactory #getRegistry

```java
/**
getRegistry 方法先访问缓存，缓存未命中则调用 createRegistry 创建 Registry，然后写入缓存。这里的 createRegistry 是一个模板方法，由具体的子类实现。
*/
public Registry getRegistry(URL url) {
    url = url.setPath(RegistryService.class.getName())
            .addParameter(Constants.INTERFACE_KEY, RegistryService.class.getName())
            .removeParameters(Constants.EXPORT_KEY, Constants.REFER_KEY);
    String key = url.toServiceString();
    LOCK.lock();
    try {
    	// 访问缓存
        Registry registry = REGISTRIES.get(key);
        if (registry != null) {
            return registry;
        }
        
        // 缓存未命中，创建 Registry 实例
        registry = createRegistry(url);
        if (registry == null) {
            throw new IllegalStateException("Can not create registry...");
        }
        
        // 写入缓存
        REGISTRIES.put(key, registry);
        return registry;
    } finally {
        LOCK.unlock();
    }
}

protected abstract Registry createRegistry(URL url);
```



###### ZookeeperRegistryFactory 

```java
public class ZookeeperRegistryFactory extends AbstractRegistryFactory {

    // zookeeperTransporter 由 SPI 在运行时注入，类型为 ZookeeperTransporter$Adaptive
    private ZookeeperTransporter zookeeperTransporter;

    public void setZookeeperTransporter(ZookeeperTransporter zookeeperTransporter) {
        this.zookeeperTransporter = zookeeperTransporter;
    }

    @Override
    public Registry createRegistry(URL url) {
        // 创建 ZookeeperRegistry
        return new ZookeeperRegistry(url, zookeeperTransporter);
    }
}

public ZookeeperRegistry(URL url, ZookeeperTransporter zookeeperTransporter) {
    super(url);
    if (url.isAnyHost()) {
        throw new IllegalStateException("registry address == null");
    }
    
    // 获取组名，默认为 dubbo
    String group = url.getParameter(Constants.GROUP_KEY, DEFAULT_ROOT);
    if (!group.startsWith(Constants.PATH_SEPARATOR)) {
        // group = "/" + group
        group = Constants.PATH_SEPARATOR + group;
    }
    this.root = group;
    // 创建 Zookeeper 客户端，默认为 CuratorZookeeperTransporter
    zkClient = zookeeperTransporter.connect(url);
    // 添加状态监听器
    zkClient.addStateListener(new StateListener() {
        @Override
        public void stateChanged(int state) {
            if (state == RECONNECTED) {
                try {
                    recover();
                } catch (Exception e) {
                    logger.error(e.getMessage(), e);
                }
            }
        }
    });
}
```

ZookeeperTransporter 的 connect 方法调用，这个方法用于创建 Zookeeper 客户端。创建好 Zookeeper 客户端，意味着注册中心的创建过程就结束了.connect 方法会在被调用时决定加载什么类型的 ZookeeperTransporter 拓展，默认为 CuratorZookeeperTransporter。

```java
public ZookeeperClient connect(URL url) {
    // 创建 CuratorZookeeperClient
    return new CuratorZookeeperClient(url);
}

public class CuratorZookeeperClient extends AbstractZookeeperClient<CuratorWatcher> {

    private final CuratorFramework client;
    
    public CuratorZookeeperClient(URL url) {
        super(url);
        try {
            // 创建 CuratorFramework 构造器
            CuratorFrameworkFactory.Builder builder = CuratorFrameworkFactory.builder()
                    .connectString(url.getBackupAddress())
                    .retryPolicy(new RetryNTimes(1, 1000))
                    .connectionTimeoutMs(5000);
            String authority = url.getAuthority();
            if (authority != null && authority.length() > 0) {
                builder = builder.authorization("digest", authority.getBytes());
            }
            // 构建 CuratorFramework 实例
            client = builder.build();
            // 添加监听器
            client.getConnectionStateListenable().addListener(new ConnectionStateListener() {
                @Override
                public void stateChanged(CuratorFramework client, ConnectionState state) {
                    if (state == ConnectionState.LOST) {
                        CuratorZookeeperClient.this.stateChanged(StateListener.DISCONNECTED);
                    } else if (state == ConnectionState.CONNECTED) {
                        CuratorZookeeperClient.this.stateChanged(StateListener.CONNECTED);
                    } else if (state == ConnectionState.RECONNECTED) {
                        CuratorZookeeperClient.this.stateChanged(StateListener.RECONNECTED);
                    }
                }
            });
            
            // 启动客户端
            client.start();
        } catch (Exception e) {
            throw new IllegalStateException(e.getMessage(), e);
        }
    }
}
```

CuratorZookeeperClient 构造方法主要用于创建和启动 CuratorFramework 实例。



###### FailbackRegistry 

服务注册的接口为 register(URL)，这个方法定义在 FailbackRegistry 抽象类中。

```java
public void register(URL url) {
    super.register(url);
    failedRegistered.remove(url);
    failedUnregistered.remove(url);
    try {
        // 模板方法，由子类实现
        doRegister(url);
    } catch (Exception e) {
        Throwable t = e;

        // 获取 check 参数，若 check = true 将会直接抛出异常
        boolean check = getUrl().getParameter(Constants.CHECK_KEY, true)
                && url.getParameter(Constants.CHECK_KEY, true)
                && !Constants.CONSUMER_PROTOCOL.equals(url.getProtocol());
        boolean skipFailback = t instanceof SkipFailbackWrapperException;
        if (check || skipFailback) {
            if (skipFailback) {
                t = t.getCause();
            }
            throw new IllegalStateException("Failed to register");
        } else {
            logger.error("Failed to register");
        }

        // 记录注册失败的链接
        failedRegistered.add(url);
    }
}

protected abstract void doRegister(URL url);
```



```java
protected void doRegister(URL url) {
    try {
        // 通过 Zookeeper 客户端创建节点，节点路径由 toUrlPath 方法生成，路径格式如下:
        //   /${group}/${serviceInterface}/providers/${url}
        // 比如
        //   /dubbo/org.apache.dubbo.DemoService/providers/dubbo%3A%2F%2F127.0.0.1......
        zkClient.create(toUrlPath(url), url.getParameter(Constants.DYNAMIC_KEY, true));
    } catch (Throwable e) {
        throw new RpcException("Failed to register...");
    }
}

public void create(String path, boolean ephemeral) {
    if (!ephemeral) {
        // 如果要创建的节点类型非临时节点，那么这里要检测节点是否存在
        if (checkExists(path)) {
            return;
        }
    }
    int i = path.lastIndexOf('/');
    if (i > 0) {
        // 递归创建上一级路径
        create(path.substring(0, i), false);
    }
    
    // 根据 ephemeral 的值创建临时或持久节点
    if (ephemeral) {
        createEphemeral(path);
    } else {
        createPersistent(path);
    }
}
```

通过递归创建当前节点的上一级路径，然后再根据 ephemeral 的值决定创建临时还是持久节点。







Dubbo的比较核心的抽象：Invoker， Invoker是一个代理类，从ProxyFactory中生成。

1. Invoker -执行具体的远程调用
2. Protocol – 服务地址的发布和订阅
3. Exporter – 暴露服务或取消暴露



## protocol发布服务



我们看一下dubboProtocol的 export方法：openServer(url）

接着调用 openServer， 继续 createServer 创建服务

继续看其中的 createServer方法：

发现 ExchangeServer 是通过 Exchangers 创建的，直接看 Exchanger#bind 方法

getExchanger方法实际上调用的是 ExtensionLoader 的相关方法，这里的 ExtensionLoader 是dubbo插件化的核心，我们会在后面的插件化讲解中详细讲解，这里我们只需要知道Exchanger的默认实现只有一个：HeaderExchanger。上面一段代码最终调用的是：

可以看到Server与Client实例均是在这里创建的，HeaderExchangeServer需要一个Server类型的参数，来自` Transporters#bind()`：

getTransporter()获取的实例来源于配置，默认返回一个 `NettyTransporter`：

NettyServer





# 服务引用

Dubbo 服务引用的时机有两个，第一个是在 Spring 容器调用 ReferenceBean 的 afterPropertiesSet 方法时引用服务，第二个是在 ReferenceBean 对应的服务被注入到其他类中时引用。这两个引用服务的时机区别在于，第一个是饿汉式的，第二个是懒汉式的。默认情况下，Dubbo 使用懒汉式引用服务。如果需要使用饿汉式，可通过配置 \<dubbo:reference> 的 init 属性开启。下面我们按照 Dubbo 默认配置进行分析，整个分析过程从 ReferenceBean 的 getObject 方法开始。当我们的服务被注入到其他类中时，Spring 会第一时间调用 getObject 方法，并由该方法执行服务引用逻辑。按照惯例，在进行具体工作之前，需先进行配置检查与收集工作。接着根据收集到的信息决定服务用的方式，有三种，第一种是引用本地 (JVM) 服务，第二是通过直连方式引用远程服务，第三是通过注册中心引用远程服务。不管是哪种引用方式，最后都会得到一个 Invoker 实例。如果有多个注册中心，多个服务提供者，这个时候会得到一组 Invoker 实例，此时需要通过集群管理类 Cluster 将多个 Invoker 合并成一个实例。合并后的 Invoker 实例已经具备调用本地或远程服务的能力了，但并不能将此实例暴露给用户使用，这会对用户业务代码造成侵入。此时框架还需要通过代理工厂类 (ProxyFactory) 为服务接口生成代理类，并让代理类去调用 Invoker 逻辑。避免了 Dubbo 框架代码对业务代码的侵入，同时也让框架更容易使用。

## ReferenceBean

和serviceBean发布一样，也是使用NamespaceHandler作为切入点，调用ReferenceBean里面的afterPropertiesSet方法

**方法调用顺序 afterPropertiesSet() ---- getObject() -> get() -> init() -> ReferenceConfig#createProxy()**

afterPropertiesSet方法中都是确认所有的组件是否都初始化好了，都准备好后我们进入生成Invoker的部分。这里的 getObject 会调用父类 ReferenceConfig 的 init 方法完成组装：

### ReferenceConfig#init()

请求信息的构建

关注内容：Map 信息构建到底有哪些？哪些是需要自己配置的？用于位置？

服务于 url 构建

```java
private void init() {
    // 避免重复初始化
    if (initialized) {
        return;
    }
    initialized = true;
    // 检测接口名合法性
    if (interfaceName == null || interfaceName.length() == 0) {
        throw new IllegalStateException("interface not allow null!");
    }

    // 检测 consumer 变量是否为空，为空则创建
    checkDefault();
    appendProperties(this);
    if (getGeneric() == null && getConsumer() != null) {
        // 设置 generic
        setGeneric(getConsumer().getGeneric());
    }

    // 检测是否为泛化接口
    if (ProtocolUtils.isGeneric(getGeneric())) {
        interfaceClass = GenericService.class;
    } else {
        try {
            // 加载类
            interfaceClass = Class.forName(interfaceName, true, Thread.currentThread()
                    .getContextClassLoader());
        } catch (ClassNotFoundException e) {
            throw new IllegalStateException(e.getMessage(), e);
        }
        checkInterfaceAndMethods(interfaceClass, methods);
    }
    
    // -------------------------------✨ 分割线1 ✨------------------------------

    // 从系统变量中获取与接口名对应的属性值
    String resolve = System.getProperty(interfaceName);
    String resolveFile = null;
    if (resolve == null || resolve.length() == 0) {
        // 从系统属性中获取解析文件路径
        resolveFile = System.getProperty("dubbo.resolve.file");
        if (resolveFile == null || resolveFile.length() == 0) {
            // 从指定位置加载配置文件
            File userResolveFile = new File(new File(System.getProperty("user.home")), "dubbo-resolve.properties");
            if (userResolveFile.exists()) {
                // 获取文件绝对路径
                resolveFile = userResolveFile.getAbsolutePath();
            }
        }
        if (resolveFile != null && resolveFile.length() > 0) {
            Properties properties = new Properties();
            FileInputStream fis = null;
            try {
                fis = new FileInputStream(new File(resolveFile));
                // 从文件中加载配置
                properties.load(fis);
            } catch (IOException e) {
                throw new IllegalStateException("Unload ..., cause:...");
            } finally {
                try {
                    if (null != fis) fis.close();
                } catch (IOException e) {
                    logger.warn(e.getMessage(), e);
                }
            }
            // 获取与接口名对应的配置
            resolve = properties.getProperty(interfaceName);
        }
    }
    if (resolve != null && resolve.length() > 0) {
        // 将 resolve 赋值给 url
        url = resolve;
    }
    
    // -------------------------------✨ 分割线2 ✨------------------------------
    if (consumer != null) {
        if (application == null) {
            // 从 consumer 中获取 Application 实例，下同
            application = consumer.getApplication();
        }
        if (module == null) {
            module = consumer.getModule();
        }
        if (registries == null) {
            registries = consumer.getRegistries();
        }
        if (monitor == null) {
            monitor = consumer.getMonitor();
        }
    }
    if (module != null) {
        if (registries == null) {
            registries = module.getRegistries();
        }
        if (monitor == null) {
            monitor = module.getMonitor();
        }
    }
    if (application != null) {
        if (registries == null) {
            registries = application.getRegistries();
        }
        if (monitor == null) {
            monitor = application.getMonitor();
        }
    }
    
    // 检测 Application 合法性
    checkApplication();
    // 检测本地存根配置合法性
    checkStubAndMock(interfaceClass);
    
	// -------------------------------✨ 分割线3 ✨------------------------------
    
    Map<String, String> map = new HashMap<String, String>();
    Map<Object, Object> attributes = new HashMap<Object, Object>();

    // 添加 side、协议版本信息、时间戳和进程号等信息到 map 中
    map.put(Constants.SIDE_KEY, Constants.CONSUMER_SIDE);
    map.put(Constants.DUBBO_VERSION_KEY, Version.getProtocolVersion());
    map.put(Constants.TIMESTAMP_KEY, String.valueOf(System.currentTimeMillis()));
    if (ConfigUtils.getPid() > 0) {
        map.put(Constants.PID_KEY, String.valueOf(ConfigUtils.getPid()));
    }

    // 非泛化服务
    if (!isGeneric()) {
        // 获取版本
        String revision = Version.getVersion(interfaceClass, version);
        if (revision != null && revision.length() > 0) {
            map.put("revision", revision);
        }

        // 获取接口方法列表，并添加到 map 中
        String[] methods = Wrapper.getWrapper(interfaceClass).getMethodNames();
        if (methods.length == 0) {
            map.put("methods", Constants.ANY_VALUE);
        } else {
            map.put("methods", StringUtils.join(new HashSet<String>(Arrays.asList(methods)), ","));
        }
    }
    map.put(Constants.INTERFACE_KEY, interfaceName);
    // 将 ApplicationConfig、ConsumerConfig、ReferenceConfig 等对象的字段信息添加到 map 中
    appendParameters(map, application);
    appendParameters(map, module);
    appendParameters(map, consumer, Constants.DEFAULT_KEY);
    appendParameters(map, this);
    
	// -------------------------------✨ 分割线4 ✨------------------------------
    
    String prefix = StringUtils.getServiceKey(map);
    if (methods != null && !methods.isEmpty()) {
        // 遍历 MethodConfig 列表
        for (MethodConfig method : methods) {
            appendParameters(map, method, method.getName());
            String retryKey = method.getName() + ".retry";
            // 检测 map 是否包含 methodName.retry
            if (map.containsKey(retryKey)) {
                String retryValue = map.remove(retryKey);
                if ("false".equals(retryValue)) {
                    // 添加重试次数配置 methodName.retries
                    map.put(method.getName() + ".retries", "0");
                }
            }
 
            // 添加 MethodConfig 中的“属性”字段到 attributes
            // 比如 onreturn、onthrow、oninvoke 等
            appendAttributes(attributes, method, prefix + "." + method.getName());
            checkAndConvertImplicitConfig(method, map, attributes);
        }
    }
    
	// -------------------------------✨ 分割线5 ✨------------------------------

    // 获取服务消费者 ip 地址
    String hostToRegistry = ConfigUtils.getSystemProperty(Constants.DUBBO_IP_TO_REGISTRY);
    if (hostToRegistry == null || hostToRegistry.length() == 0) {
        hostToRegistry = NetUtils.getLocalHost();
    } else if (isInvalidLocalHost(hostToRegistry)) {
        throw new IllegalArgumentException("Specified invalid registry ip from property..." );
    }
    map.put(Constants.REGISTER_IP_KEY, hostToRegistry);

    // 存储 attributes 到系统上下文中
    StaticContext.getSystemContext().putAll(attributes);

    // 创建代理类
    ref = createProxy(map);

    // 根据服务名，ReferenceConfig，代理类构建 ConsumerModel，
    // 并将 ConsumerModel 存入到 ApplicationModel 中
    ConsumerModel consumerModel = new ConsumerModel(getUniqueServiceName(), this, ref, interfaceClass.getMethods());
    ApplicationModel.initConsumerModel(getUniqueServiceName(), consumerModel);
}
```







### ReferenceConfig#createProxy

请求 url 的参数抉择，构建

```java
private T createProxy(Map<String, String> map) {
    URL tmpUrl = new URL("temp", "localhost", 0, map);
    final boolean isJvmRefer;
    if (isInjvm() == null) {
        // url 配置被指定，则不做本地引用
        if (url != null && url.length() > 0) {
            isJvmRefer = false;
        // 根据 url 的协议、scope 以及 injvm 等参数检测是否需要本地引用
        // 比如如果用户显式配置了 scope=local，此时 isInjvmRefer 返回 true
        } else if (InjvmProtocol.getInjvmProtocol().isInjvmRefer(tmpUrl)) {
            isJvmRefer = true;
        } else {
            isJvmRefer = false;
        }
    } else {
        // 获取 injvm 配置值
        isJvmRefer = isInjvm().booleanValue();
    }

    // 本地引用
    if (isJvmRefer) {
        // 生成本地引用 URL，协议为 injvm
        URL url = new URL(Constants.LOCAL_PROTOCOL, NetUtils.LOCALHOST, 0, interfaceClass.getName()).addParameters(map);
        // 调用 refer 方法构建 InjvmInvoker 实例
        invoker = refprotocol.refer(interfaceClass, url);
        
    // 远程引用
    } else {
        // url 不为空，表明用户可能想进行点对点调用
        if (url != null && url.length() > 0) {
            // 当需要配置多个 url 时，可用分号进行分割，这里会进行切分
            String[] us = Constants.SEMICOLON_SPLIT_PATTERN.split(url);
            if (us != null && us.length > 0) {
                for (String u : us) {
                    URL url = URL.valueOf(u);
                    if (url.getPath() == null || url.getPath().length() == 0) {
                        // 设置接口全限定名为 url 路径
                        url = url.setPath(interfaceName);
                    }
                    
                    // 检测 url 协议是否为 registry，若是，表明用户想使用指定的注册中心
                    if (Constants.REGISTRY_PROTOCOL.equals(url.getProtocol())) {
                        // 将 map 转换为查询字符串，并作为 refer 参数的值添加到 url 中
                        urls.add(url.addParameterAndEncoded(Constants.REFER_KEY, StringUtils.toQueryString(map)));
                    } else {
                        // 合并 url，移除服务提供者的一些配置（这些配置来源于用户配置的 url 属性），
                        // 比如线程池相关配置。并保留服务提供者的部分配置，比如版本，group，时间戳等
                        // 最后将合并后的配置设置为 url 查询字符串中。
                        urls.add(ClusterUtils.mergeUrl(url, map));
                    }
                }
            }
        } else {
            // 加载注册中心 url
            List<URL> us = loadRegistries(false);
            if (us != null && !us.isEmpty()) {
                for (URL u : us) {
                    URL monitorUrl = loadMonitor(u);
                    if (monitorUrl != null) {
                        map.put(Constants.MONITOR_KEY, URL.encode(monitorUrl.toFullString()));
                    }
                    // 添加 refer 参数到 url 中，并将 url 添加到 urls 中
                    urls.add(u.addParameterAndEncoded(Constants.REFER_KEY, StringUtils.toQueryString(map)));
                }
            }

            // 未配置注册中心，抛出异常
            if (urls.isEmpty()) {
                throw new IllegalStateException("No such any registry to reference...");
            }
        }

        // 单个注册中心或服务提供者(服务直连，下同)
        if (urls.size() == 1) {
            // 调用 RegistryProtocol 的 refer 构建 Invoker 实例
            invoker = refprotocol.refer(interfaceClass, urls.get(0));
            
        // 多个注册中心或多个服务提供者，或者两者混合
        } else {
            List<Invoker<?>> invokers = new ArrayList<Invoker<?>>();
            URL registryURL = null;

            // 获取所有的 Invoker
            for (URL url : urls) {
                // 通过 refprotocol 调用 refer 构建 Invoker，refprotocol 会在运行时
                // 根据 url 协议头加载指定的 Protocol 实例，并调用实例的 refer 方法
                invokers.add(refprotocol.refer(interfaceClass, url));
                if (Constants.REGISTRY_PROTOCOL.equals(url.getProtocol())) {
                    registryURL = url;
                }
            }
            if (registryURL != null) {
                // 如果注册中心链接不为空，则将使用 AvailableCluster
                URL u = registryURL.addParameter(Constants.CLUSTER_KEY, AvailableCluster.NAME);
                // 创建 StaticDirectory 实例，并由 Cluster 对多个 Invoker 进行合并
                invoker = cluster.join(new StaticDirectory(u, invokers));
            } else {
                invoker = cluster.join(new StaticDirectory(invokers));
            }
        }
    }

    Boolean c = check;
    if (c == null && consumer != null) {
        c = consumer.isCheck();
    }
    if (c == null) {
        c = true;
    }
    
    // invoker 可用性检查
    if (c && !invoker.isAvailable()) {
        throw new IllegalStateException("No provider available for the service...");
    }

    // 生成代理类
    return (T) proxyFactory.getProxy(invoker);
}
```

先根据配置检查是否为本地调用，若是，则调用 InjvmProtocol 的 refer 方法生成 InjvmInvoker 实例。若不是，则读取直连配置项，或注册中心 url，并将读取到的 url 存储到 urls 中。然后根据 urls 元素数量进行后续操作。若 urls 元素数量为1，则直接通过 Protocol 自适应拓展类构建 Invoker 实例接口。若 urls 元素数量大于1，即存在多个注册中心或服务直连 url，此时先根据 url 构建 Invoker。然后再通过 Cluster 合并多个 Invoker，最后调用 ProxyFactory 生成代理类。Invoker 的构建过程以及代理类的过程比较重要，

## 创建 Invoker

Invoker 是 Dubbo 的核心模型，代表一个可执行体。在服务提供方，Invoker 用于调用服务提供类。在服务消费方，Invoker 用于执行远程调用。Invoker 是由 Protocol 实现类构建而来。

### REF_PROTOCOL.refer

```
invokers.add(REF_PROTOCOL.refer(interfaceClass, url));
```



##### DubboProtocol#refer

DubboProtocol#requestHandler#received,reply

```java
public <T> Invoker<T> refer(Class<T> serviceType, URL url) throws RpcException {
    optimizeSerialization(url);
    // 创建 DubboInvoker
    DubboInvoker<T> invoker = new DubboInvoker<T>(serviceType, url, getClients(url), invokers);
    invokers.add(invoker);
    return invoker;
}
```

ExchangeClient 封装到 DubboInvoker



##### DubboProtocol#getClients

```java
private ExchangeClient[] getClients(URL url) {
    // 是否共享连接
    boolean service_share_connect = false;
  	// 获取连接数，默认为0，表示未配置
    int connections = url.getParameter(Constants.CONNECTIONS_KEY, 0);
    // 如果未配置 connections，则共享连接
    if (connections == 0) {
        service_share_connect = true;
        connections = 1;
    }

    ExchangeClient[] clients = new ExchangeClient[connections];
    for (int i = 0; i < clients.length; i++) {
        if (service_share_connect) {
            // 获取共享客户端
            clients[i] = getSharedClient(url);
        } else {
            // 初始化新的客户端
            clients[i] = initClient(url);
        }
    }
    return clients;
}

//-------------不同版本有改动，原理没变-------------
private ExchangeClient getSharedClient(URL url) {
    String key = url.getAddress();
    // 获取带有“引用计数”功能的 ExchangeClient
    ReferenceCountExchangeClient client = referenceClientMap.get(key);
    if (client != null) {
        if (!client.isClosed()) {
            // 增加引用计数
            client.incrementAndGetCount();
            return client;
        } else {
            referenceClientMap.remove(key);
        }
    }

    locks.putIfAbsent(key, new Object());
    synchronized (locks.get(key)) {
        if (referenceClientMap.containsKey(key)) {
            return referenceClientMap.get(key);
        }

        // 创建 ExchangeClient 客户端
        ExchangeClient exchangeClient = initClient(url);
        // 将 ExchangeClient 实例传给 ReferenceCountExchangeClient，这里使用了装饰模式
        client = new ReferenceCountExchangeClient(exchangeClient, ghostClientMap);
        referenceClientMap.put(key, client);
        ghostClientMap.remove(key);
        locks.remove(key);
        return client;
    }
}

//------- buildReferenceCountExchangeClient ---------
private ExchangeClient initClient(URL url) {

    // 获取客户端类型，默认为 netty
    String str = url.getParameter(Constants.CLIENT_KEY, url.getParameter(Constants.SERVER_KEY, Constants.DEFAULT_REMOTING_CLIENT));

    // 添加编解码和心跳包参数到 url 中
    url = url.addParameter(Constants.CODEC_KEY, DubboCodec.NAME);
    url = url.addParameterIfAbsent(Constants.HEARTBEAT_KEY, String.valueOf(Constants.DEFAULT_HEARTBEAT));

    // 检测客户端类型是否存在，不存在则抛出异常
    if (str != null && str.length() > 0 && !ExtensionLoader.getExtensionLoader(Transporter.class).hasExtension(str)) {
        throw new RpcException("Unsupported client type: ...");
    }

    ExchangeClient client;
    try {
        // 获取 lazy 配置，并根据配置值决定创建的客户端类型
        if (url.getParameter(Constants.LAZY_CONNECT_KEY, false)) {
            // 创建懒加载 ExchangeClient 实例
            client = new LazyConnectExchangeClient(url, requestHandler);
        } else {
            // 创建普通 ExchangeClient 实例
            client = Exchangers.connect(url, requestHandler);
        }
    } catch (RemotingException e) {
        throw new RpcException("Fail to create remoting client for service...");
    }
    return client;
}

//---Exchangers#connect----
public static ExchangeClient connect(URL url, ExchangeHandler handler) throws RemotingException {
    if (url == null) {
        throw new IllegalArgumentException("url == null");
    }
    if (handler == null) {
        throw new IllegalArgumentException("handler == null");
    }
    url = url.addParameterIfAbsent(Constants.CODEC_KEY, "exchange");
    // 获取 Exchanger 实例，默认为 HeaderExchangeClient
    return getExchanger(url).connect(url, handler);
}
// getExchanger 会通过 SPI 加载 HeaderExchangeClient 实例
// ----HeaderExchanger------
// ----HeaderExchangeClient----- 

// NettyClient
```

:LazyConnectExchangeClient 代码并不是很复杂，该类会在 request 方法被调用时通过 Exchangers 的 connect 方法创建 ExchangeClient 客户端

```
// Transporters#connect
public static Client connect(URL url, ChannelHandler... handlers) throws RemotingException {
    if (url == null) {
        throw new IllegalArgumentException("url == null");
    }
    ChannelHandler handler;
    if (handlers == null || handlers.length == 0) {
        handler = new ChannelHandlerAdapter();
    } else if (handlers.length == 1) {
        handler = handlers[0];
    } else {
        // 如果 handler 数量大于1，则创建一个 ChannelHandler 分发器
        handler = new ChannelHandlerDispatcher(handlers);
    }
    
    // 获取 Transporter 自适应拓展类，并调用 connect 方法生成 Client 实例
    return getTransporter().connect(url, handler);
}

// getTransporter 方法返回的是自适应拓展类，该类会在运行时根据客户端类型加载指定的
// Transporter 实现类。若用户未配置客户端类型，则默认加载 NettyTransporter，并调用该类的
// connect 方法。

public Client connect(URL url, ChannelHandler listener) throws RemotingException {
    // 创建 NettyClient 对象
    return new NettyClient(url, listener);
}
```

##### AbstractClient

```java
    public NettyClient(final URL url, final ChannelHandler handler) throws RemotingException {
    	// you can customize name and type of client thread pool by THREAD_NAME_KEY and THREADPOOL_KEY in CommonConstants.
    	// the handler will be warped: MultiMessageHandler->HeartbeatHandler->handler
    	super(url, wrapChannelHandler(url, handler));
    }

    public AbstractClient(URL url, ChannelHandler handler) throws RemotingException {
        super(url, handler);

        needReconnect = url.getParameter(Constants.SEND_RECONNECT_KEY, false);

        initExecutor(url);

        try {
            doOpen();
        } catch (Throwable t) {
            close();
            throw new RemotingException(url.toInetSocketAddress(), null,
                    "Failed to start " + getClass().getSimpleName() + " " + NetUtils.getLocalAddress()
                            + " connect to the server " + getRemoteAddress() + ", cause: " + t.getMessage(), t);
        }
        try {
            // connect.
            connect();
            if (logger.isInfoEnabled()) {
                logger.info("Start " + getClass().getSimpleName() + " " + NetUtils.getLocalAddress() + " connect to the server " + getRemoteAddress());
            }
        } catch (RemotingException t) {
            if (url.getParameter(Constants.CHECK_KEY, true)) {
                close();
                throw t;
            } else {
                logger.warn("Failed to start " + getClass().getSimpleName() + " " + NetUtils.getLocalAddress()
                        + " connect to the server " + getRemoteAddress() + " (check == false, ignore and retry later!), cause: " + t.getMessage(), t);
            }
        } catch (Throwable t) {
            close();
            throw new RemotingException(url.toInetSocketAddress(), null,
                    "Failed to start " + getClass().getSimpleName() + " " + NetUtils.getLocalAddress()
                            + " connect to the server " + getRemoteAddress() + ", cause: " + t.getMessage(), t);
        }
    }
```

##### NettyClient#doOpen

```java
    @Override
    protected void doOpen() throws Throwable {
        final NettyClientHandler nettyClientHandler = new NettyClientHandler(getUrl(), this);
        bootstrap = new Bootstrap();
        bootstrap.group(nioEventLoopGroup)
                .option(ChannelOption.SO_KEEPALIVE, true)
                .option(ChannelOption.TCP_NODELAY, true)
                .option(ChannelOption.ALLOCATOR, PooledByteBufAllocator.DEFAULT)
                //.option(ChannelOption.CONNECT_TIMEOUT_MILLIS, getTimeout())
                .channel(NioSocketChannel.class);

        bootstrap.option(ChannelOption.CONNECT_TIMEOUT_MILLIS, Math.max(3000, getConnectTimeout()));
        bootstrap.handler(new ChannelInitializer() {

            @Override
            protected void initChannel(Channel ch) throws Exception {
                int heartbeatInterval = UrlUtils.getHeartbeat(getUrl());

                if (getUrl().getParameter(SSL_ENABLED_KEY, false)) {
                    ch.pipeline().addLast("negotiation", SslHandlerInitializer.sslClientHandler(getUrl(), nettyClientHandler));
                }

                NettyCodecAdapter adapter = new NettyCodecAdapter(getCodec(), getUrl(), NettyClient.this);
                ch.pipeline()//.addLast("logging",new LoggingHandler(LogLevel.INFO))//for debug
                        .addLast("decoder", adapter.getDecoder())
                        .addLast("encoder", adapter.getEncoder())
                        .addLast("client-idle-handler", new IdleStateHandler(heartbeatInterval, 0, 0, MILLISECONDS))
                        .addLast("handler", nettyClientHandler);

                String socksProxyHost = ConfigUtils.getProperty(SOCKS_PROXY_HOST);
                if(socksProxyHost != null) {
                    int socksProxyPort = Integer.parseInt(ConfigUtils.getProperty(SOCKS_PROXY_PORT, DEFAULT_SOCKS_PROXY_PORT));
                    Socks5ProxyHandler socks5ProxyHandler = new Socks5ProxyHandler(new InetSocketAddress(socksProxyHost, socksProxyPort));
                    ch.pipeline().addFirst(socks5ProxyHandler);
                }
            }
        });
    }
```

###### NettyClient#doConnect

```java
@Override
    protected void doConnect() throws Throwable {
        long start = System.currentTimeMillis();
        ChannelFuture future = bootstrap.connect(getConnectAddress());
        try {
            boolean ret = future.awaitUninterruptibly(getConnectTimeout(), MILLISECONDS);

            if (ret && future.isSuccess()) {
                Channel newChannel = future.channel();
                try {
                    // Close old channel
                    // copy reference
                    Channel oldChannel = NettyClient.this.channel;
                    if (oldChannel != null) {
                        try {
                            if (logger.isInfoEnabled()) {
                                logger.info("Close old netty channel " + oldChannel + " on create new netty channel " + newChannel);
                            }
                            oldChannel.close();
                        } finally {
                            NettyChannel.removeChannelIfDisconnected(oldChannel);
                        }
                    }
                } finally {
                    if (NettyClient.this.isClosed()) {
                        try {
                            if (logger.isInfoEnabled()) {
                                logger.info("Close new netty channel " + newChannel + ", because the client closed.");
                            }
                            newChannel.close();
                        } finally {
                            NettyClient.this.channel = null;
                            NettyChannel.removeChannelIfDisconnected(newChannel);
                        }
                    } else {
                        NettyClient.this.channel = newChannel;
                    }
                }
            } else if (future.cause() != null) {
                throw new RemotingException(this, "client(url: " + getUrl() + ") failed to connect to server "
                        + getRemoteAddress() + ", error message is:" + future.cause().getMessage(), future.cause());
            } else {
                throw new RemotingException(this, "client(url: " + getUrl() + ") failed to connect to server "
                        + getRemoteAddress() + " client-side timeout "
                        + getConnectTimeout() + "ms (elapsed: " + (System.currentTimeMillis() - start) + "ms) from netty client "
                        + NetUtils.getLocalHost() + " using dubbo version " + Version.getVersion());
            }
        } finally {
            // just add new valid channel to NettyChannel's cache
            if (!isConnected()) {
                //future.cancel(true);
            }
        }
    }
```



##### RegistryProtocol#refer

```java
private <T> Invoker<T> doRefer(Cluster cluster, Registry registry, Class<T> type, URL url) {
    // 创建 RegistryDirectory 实例
    RegistryDirectory<T> directory = new RegistryDirectory<T>(type, url);
    // 设置注册中心和协议
    directory.setRegistry(registry);
    directory.setProtocol(protocol);
    Map<String, String> parameters = new HashMap<String, String>(directory.getUrl().getParameters());
    // 生成服务消费者链接
    URL subscribeUrl = new URL(Constants.CONSUMER_PROTOCOL, parameters.remove(Constants.REGISTER_IP_KEY), 0, type.getName(), parameters);

    // 注册服务消费者，在 consumers 目录下新节点
    if (!Constants.ANY_VALUE.equals(url.getServiceInterface())
            && url.getParameter(Constants.REGISTER_KEY, true)) {
        registry.register(subscribeUrl.addParameters(Constants.CATEGORY_KEY, Constants.CONSUMERS_CATEGORY,
                Constants.CHECK_KEY, String.valueOf(false)));
    }

    // 订阅 providers、configurators、routers 等节点数据
    directory.subscribe(subscribeUrl.addParameter(Constants.CATEGORY_KEY,
            Constants.PROVIDERS_CATEGORY
                    + "," + Constants.CONFIGURATORS_CATEGORY
                    + "," + Constants.ROUTERS_CATEGORY));

    // 一个注册中心可能有多个服务提供者，因此这里需要将多个服务提供者合并为一个
    Invoker invoker = cluster.join(directory);
    ProviderConsumerRegTable.registerConsumer(invoker, url, subscribeUrl, directory);
    return invoker;
}
```





至此 Reference 在关联了所有application、module、consumer、registry、monitor、service、protocol后调用对应 Protocol 类的 refer 方法生成 InvokerProxy。当用户调用 service 时dubbo会通过InvokerProxy 调用 Invoker 的 invoke 的方法向服务端发起请求。客户端就这样完成了自己的初始化。

 

这个代理实例中仅仅包含一个handler对象（InvokerInvocationHandler类的实例），handler中则包含了RPC调用中非常核心的一个接口Invoker<T>的实现，Invoker接口的的的定义如下：  

```java
public interface Invoker<T> extends Node {     
    Class<T> getInterface();  //调用过程的具体表示形式      
    Result invoke(Invocation invocation) throws RpcException;
} 
```



Invoker<T>接口的核心方法是invoke(Invocation invocation)，方法的参数Invocation是一个调用过程的抽象，也是Dubbo框架的核心接口，该接口中包含如何获取调用方法的名称、参数类型列表、参数列表以及绑定的数据，定义代码如下：  



代理中的 handler 实例中包含的 Invoker<T> 接口实现者是 MockClusterInvoker，其中MockClusterInvoker仅仅是一个 Invoker 的包装，并且也实现了接口Invoker<T>，其只是用于实现Dubbo框架中的mock功能，我们可以从他的 invoke 方法的实现中看出



 DubboInvoker#doInvoke

NettyChannel#send

#### getProxy

```java
public <T> T getProxy(Invoker<T> invoker) throws RpcException {
    // 调用重载方法
    return getProxy(invoker, false);
}

public <T> T getProxy(Invoker<T> invoker, boolean generic) throws RpcException {
    Class<?>[] interfaces = null;
    // 获取接口列表
    String config = invoker.getUrl().getParameter("interfaces");
    if (config != null && config.length() > 0) {
        // 切分接口列表
        String[] types = Constants.COMMA_SPLIT_PATTERN.split(config);
        if (types != null && types.length > 0) {
            interfaces = new Class<?>[types.length + 2];
            // 设置服务接口类和 EchoService.class 到 interfaces 中
            interfaces[0] = invoker.getInterface();
            interfaces[1] = EchoService.class;
            for (int i = 0; i < types.length; i++) {
                // 加载接口类
                interfaces[i + 1] = ReflectUtils.forName(types[i]);
            }
        }
    }
    if (interfaces == null) {
        interfaces = new Class<?>[]{invoker.getInterface(), EchoService.class};
    }

    // 为 http 和 hessian 协议提供泛化调用支持，参考 pull request #1827
    if (!invoker.getInterface().equals(GenericService.class) && generic) {
        int len = interfaces.length;
        Class<?>[] temp = interfaces;
        // 创建新的 interfaces 数组
        interfaces = new Class<?>[len + 1];
        System.arraycopy(temp, 0, interfaces, 0, len);
        // 设置 GenericService.class 到数组中
        interfaces[len] = GenericService.class;
    }

    // 调用重载方法
    return getProxy(invoker, interfaces);
}

public abstract <T> T getProxy(Invoker<T> invoker, Class<?>[] types);


public <T> T getProxy(Invoker<T> invoker, Class<?>[] interfaces) {
    // 生成 Proxy 子类（Proxy 是抽象类）。并调用 Proxy 子类的 newInstance 方法创建 Proxy 实例
    return (T) Proxy.getProxy(interfaces).newInstance(new InvokerInvocationHandler(invoker));
}
```

首先是通过 Proxy 的 getProxy 方法获取 Proxy 子类，然后创建 InvokerInvocationHandler 对象，并将该对象传给 newInstance 生成 Proxy 实例。InvokerInvocationHandler 实现 JDK 的 InvocationHandler 接口，具体的用途是拦截接口类调用。

##### Proxy#getProxy

```java
public static Proxy getProxy(Class<?>... ics) {
    // 调用重载方法
    return getProxy(ClassHelper.getClassLoader(Proxy.class), ics);
}

public static Proxy getProxy(ClassLoader cl, Class<?>... ics) {
    if (ics.length > 65535)
        throw new IllegalArgumentException("interface limit exceeded");

    StringBuilder sb = new StringBuilder();
    // 遍历接口列表
    for (int i = 0; i < ics.length; i++) {
        String itf = ics[i].getName();
        // 检测类型是否为接口
        if (!ics[i].isInterface())
            throw new RuntimeException(itf + " is not a interface.");

        Class<?> tmp = null;
        try {
            // 重新加载接口类
            tmp = Class.forName(itf, false, cl);
        } catch (ClassNotFoundException e) {
        }

        // 检测接口是否相同，这里 tmp 有可能为空
        if (tmp != ics[i])
            throw new IllegalArgumentException(ics[i] + " is not visible from class loader");

        // 拼接接口全限定名，分隔符为 ;
        sb.append(itf).append(';');
    }

    // 使用拼接后的接口名作为 key
    String key = sb.toString();

    Map<String, Object> cache;
    synchronized (ProxyCacheMap) {
        cache = ProxyCacheMap.get(cl);
        if (cache == null) {
            cache = new HashMap<String, Object>();
            ProxyCacheMap.put(cl, cache);
        }
    }

    Proxy proxy = null;
    synchronized (cache) {
        do {
            // 从缓存中获取 Reference<Proxy> 实例
            Object value = cache.get(key);
            if (value instanceof Reference<?>) {
                proxy = (Proxy) ((Reference<?>) value).get();
                if (proxy != null) {
                    return proxy;
                }
            }

            // 并发控制，保证只有一个线程可以进行后续操作
            if (value == PendingGenerationMarker) {
                try {
                    // 其他线程在此处进行等待
                    cache.wait();
                } catch (InterruptedException e) {
                }
            } else {
                // 放置标志位到缓存中，并跳出 while 循环进行后续操作
                cache.put(key, PendingGenerationMarker);
                break;
            }
        }
        while (true);
    }

    long id = PROXY_CLASS_COUNTER.getAndIncrement();
    String pkg = null;
    ClassGenerator ccp = null, ccm = null;
    try {
        // 创建 ClassGenerator 对象
        ccp = ClassGenerator.newInstance(cl);

        Set<String> worked = new HashSet<String>();
        List<Method> methods = new ArrayList<Method>();

        for (int i = 0; i < ics.length; i++) {
            // 检测接口访问级别是否为 protected 或 privete
            if (!Modifier.isPublic(ics[i].getModifiers())) {
                // 获取接口包名
                String npkg = ics[i].getPackage().getName();
                if (pkg == null) {
                    pkg = npkg;
                } else {
                    if (!pkg.equals(npkg))
                        // 非 public 级别的接口必须在同一个包下，否者抛出异常
                        throw new IllegalArgumentException("non-public interfaces from different packages");
                }
            }
            
            // 添加接口到 ClassGenerator 中
            ccp.addInterface(ics[i]);

            // 遍历接口方法
            for (Method method : ics[i].getMethods()) {
                // 获取方法描述，可理解为方法签名
                String desc = ReflectUtils.getDesc(method);
                // 如果方法描述字符串已在 worked 中，则忽略。考虑这种情况，
                // A 接口和 B 接口中包含一个完全相同的方法
                if (worked.contains(desc))
                    continue;
                worked.add(desc);

                int ix = methods.size();
                // 获取方法返回值类型
                Class<?> rt = method.getReturnType();
                // 获取参数列表
                Class<?>[] pts = method.getParameterTypes();

                // 生成 Object[] args = new Object[1...N]
                StringBuilder code = new StringBuilder("Object[] args = new Object[").append(pts.length).append("];");
                for (int j = 0; j < pts.length; j++)
                    // 生成 args[1...N] = ($w)$1...N;
                    code.append(" args[").append(j).append("] = ($w)$").append(j + 1).append(";");
                // 生成 InvokerHandler 接口的 invoker 方法调用语句，如下：
                // Object ret = handler.invoke(this, methods[1...N], args);
                code.append(" Object ret = handler.invoke(this, methods[" + ix + "], args);");

                // 返回值不为 void
                if (!Void.TYPE.equals(rt))
                    // 生成返回语句，形如 return (java.lang.String) ret;
                    code.append(" return ").append(asArgument(rt, "ret")).append(";");

                methods.add(method);
                // 添加方法名、访问控制符、参数列表、方法代码等信息到 ClassGenerator 中 
                ccp.addMethod(method.getName(), method.getModifiers(), rt, pts, method.getExceptionTypes(), code.toString());
            }
        }

        if (pkg == null)
            pkg = PACKAGE_NAME;

        // 构建接口代理类名称：pkg + ".proxy" + id，比如 org.apache.dubbo.proxy0
        String pcn = pkg + ".proxy" + id;
        ccp.setClassName(pcn);
        ccp.addField("public static java.lang.reflect.Method[] methods;");
        // 生成 private java.lang.reflect.InvocationHandler handler;
        ccp.addField("private " + InvocationHandler.class.getName() + " handler;");

        // 为接口代理类添加带有 InvocationHandler 参数的构造方法，比如：
        // porxy0(java.lang.reflect.InvocationHandler arg0) {
        //     handler=$1;
    	// }
        ccp.addConstructor(Modifier.PUBLIC, new Class<?>[]{InvocationHandler.class}, new Class<?>[0], "handler=$1;");
        // 为接口代理类添加默认构造方法
        ccp.addDefaultConstructor();
        
        // 生成接口代理类
        Class<?> clazz = ccp.toClass();
        clazz.getField("methods").set(null, methods.toArray(new Method[0]));

        // 构建 Proxy 子类名称，比如 Proxy1，Proxy2 等
        String fcn = Proxy.class.getName() + id;
        ccm = ClassGenerator.newInstance(cl);
        ccm.setClassName(fcn);
        ccm.addDefaultConstructor();
        ccm.setSuperClass(Proxy.class);
        // 为 Proxy 的抽象方法 newInstance 生成实现代码，形如：
        // public Object newInstance(java.lang.reflect.InvocationHandler h) { 
        //     return new org.apache.dubbo.proxy0($1);
        // }
        ccm.addMethod("public Object newInstance(" + InvocationHandler.class.getName() + " h){ return new " + pcn + "($1); }");
        // 生成 Proxy 实现类
        Class<?> pc = ccm.toClass();
        // 通过反射创建 Proxy 实例
        proxy = (Proxy) pc.newInstance();
    } catch (RuntimeException e) {
        throw e;
    } catch (Exception e) {
        throw new RuntimeException(e.getMessage(), e);
    } finally {
        if (ccp != null)
            // 释放资源
            ccp.release();
        if (ccm != null)
            ccm.release();
        synchronized (cache) {
            if (proxy == null)
                cache.remove(key);
            else
                // 写缓存
                cache.put(key, new WeakReference<Proxy>(proxy));
            // 唤醒其他等待线程
            cache.notifyAll();
        }
    }
    return proxy;
}
```



##### ClassGenerator



## 服务调用过程

### Invoker#invoke

InvokerInvocationHandler  内置 MockClusterInvoker

```java
public class MockClusterInvoker<T> implements Invoker<T> {
    
    private final Invoker<T> invoker;
    
    public Result invoke(Invocation invocation) throws RpcException {
        Result result = null;

        // 获取 mock 配置值
        String value = directory.getUrl().getMethodParameter(invocation.getMethodName(), Constants.MOCK_KEY, Boolean.FALSE.toString()).trim();
        if (value.length() == 0 || value.equalsIgnoreCase("false")) {
            // 无 mock 逻辑，直接调用其他 Invoker 对象的 invoke 方法，
            // 比如 FailoverClusterInvoker
            result = this.invoker.invoke(invocation);
        } else if (value.startsWith("force")) {
            // force:xxx 直接执行 mock 逻辑，不发起远程调用
            result = doMockInvoke(invocation, null);
        } else {
            // fail:xxx 表示消费方对调用服务失败后，再执行 mock 逻辑，不抛出异常
            try {
                // 调用其他 Invoker 对象的 invoke 方法
                result = this.invoker.invoke(invocation);
            } catch (RpcException e) {
                if (e.isBiz()) {
                    throw e;
                } else {
                    // 调用失败，执行 mock 逻辑
                    result = doMockInvoke(invocation, e);
                }
            }
        }
        return result;
    }
    
    // 省略其他方法
}
```

#### DubboInvoker#doInvoke

##### request(invocation，timeout)

```java
public abstract class AbstractInvoker<T> implements Invoker<T> {
    
    public Result invoke(Invocation inv) throws RpcException {
        if (destroyed.get()) {
            throw new RpcException("Rpc invoker for service ...");
        }
        RpcInvocation invocation = (RpcInvocation) inv;
        // 设置 Invoker
        invocation.setInvoker(this);
        if (attachment != null && attachment.size() > 0) {
            // 设置 attachment
            invocation.addAttachmentsIfAbsent(attachment);
        }
        Map<String, String> contextAttachments = RpcContext.getContext().getAttachments();
        if (contextAttachments != null && contextAttachments.size() != 0) {
            // 添加 contextAttachments 到 RpcInvocation#attachment 变量中
            invocation.addAttachments(contextAttachments);
        }
        if (getUrl().getMethodParameter(invocation.getMethodName(), Constants.ASYNC_KEY, false)) {
            // 设置异步信息到 RpcInvocation#attachment 中
            invocation.setAttachment(Constants.ASYNC_KEY, Boolean.TRUE.toString());
        }
        RpcUtils.attachInvocationIdIfAsync(getUrl(), invocation);

        try {
            // 抽象方法，由子类实现
            return doInvoke(invocation);
        } catch (InvocationTargetException e) {
            // ...
        } catch (RpcException e) {
            // ...
        } catch (Throwable e) {
            return new RpcResult(e);
        }
    }

    protected abstract Result doInvoke(Invocation invocation) throws Throwable;
    
    // 省略其他方法
}

public class DubboInvoker<T> extends AbstractInvoker<T> {
    
    private final ExchangeClient[] clients;
    
    protected Result doInvoke(final Invocation invocation) throws Throwable {
        RpcInvocation inv = (RpcInvocation) invocation;
        final String methodName = RpcUtils.getMethodName(invocation);
        // 设置 path 和 version 到 attachment 中
        inv.setAttachment(Constants.PATH_KEY, getUrl().getPath());
        inv.setAttachment(Constants.VERSION_KEY, version);

        ExchangeClient currentClient;
        if (clients.length == 1) {
            // 从 clients 数组中获取 ExchangeClient
            currentClient = clients[0];
        } else {
            currentClient = clients[index.getAndIncrement() % clients.length];
        }
        try {
            // 获取异步配置
            boolean isAsync = RpcUtils.isAsync(getUrl(), invocation);
            // isOneway 为 true，表示“单向”通信
            boolean isOneway = RpcUtils.isOneway(getUrl(), invocation);
            int timeout = getUrl().getMethodParameter(methodName, Constants.TIMEOUT_KEY, Constants.DEFAULT_TIMEOUT);

            // 异步无返回值
            if (isOneway) {
                boolean isSent = getUrl().getMethodParameter(methodName, Constants.SENT_KEY, false);
                // 发送请求
                currentClient.send(inv, isSent);
                // 设置上下文中的 future 字段为 null
                RpcContext.getContext().setFuture(null);
                // 返回一个空的 RpcResult
                return new RpcResult();
            } 

            // 异步有返回值
            else if (isAsync) {
                // 发送请求，并得到一个 ResponseFuture 实例
                ResponseFuture future = currentClient.request(inv, timeout);
                // 设置 future 到上下文中
                RpcContext.getContext().setFuture(new FutureAdapter<Object>(future));
                // 暂时返回一个空结果
                return new RpcResult();
            } 

            // 同步调用
            else {
                RpcContext.getContext().setFuture(null);
                // 发送请求，得到一个 ResponseFuture 实例，并调用该实例的 get 方法进行等待
                return (Result) currentClient.request(inv, timeout).get();
            }
        } catch (TimeoutException e) {
            throw new RpcException(..., "Invoke remote method timeout....");
        } catch (RemotingException e) {
            throw new RpcException(..., "Failed to invoke remote method: ...");
        }
    }
    
    // 省略其他方法
}
```



#### ReferenceCountExchangeClient#request

```java
final class ReferenceCountExchangeClient implements ExchangeClient {

    private final URL url;
    private final AtomicInteger referenceCount = new AtomicInteger(0);

    public ReferenceCountExchangeClient(ExchangeClient client, ConcurrentMap<String, LazyConnectExchangeClient> ghostClientMap) {
        this.client = client;
        // 引用计数自增
        referenceCount.incrementAndGet();
        this.url = client.getUrl();
        
        // ...
    }

    @Override
    public ResponseFuture request(Object request) throws RemotingException {
        // 直接调用被装饰对象的同签名方法
        return client.request(request);
    }

    @Override
    public ResponseFuture request(Object request, int timeout) throws RemotingException {
        // 直接调用被装饰对象的同签名方法
        return client.request(request, timeout);
    }

    /** 引用计数自增，该方法由外部调用 */
    public void incrementAndGetCount() {
        // referenceCount 自增
        referenceCount.incrementAndGet();
    }
    
        @Override
    public void close(int timeout) {
        // referenceCount 自减
        if (referenceCount.decrementAndGet() <= 0) {
            if (timeout == 0) {
                client.close();
            } else {
                client.close(timeout);
            }
            client = replaceWithLazyClient();
        }
    }
    
    // 省略部分方法
}
```



#### HeaderExchangeClient#request

```java
// 封装了一些关于心跳检测的逻辑
public class HeaderExchangeClient implements ExchangeClient {

    private static final ScheduledThreadPoolExecutor scheduled = new ScheduledThreadPoolExecutor(2, new NamedThreadFactory("dubbo-remoting-client-heartbeat", true));
    private final Client client;
    private final ExchangeChannel channel;
    private ScheduledFuture<?> heartbeatTimer;
    private int heartbeat;
    private int heartbeatTimeout;

    public HeaderExchangeClient(Client client, boolean needHeartbeat) {
        if (client == null) {
            throw new IllegalArgumentException("client == null");
        }
        this.client = client;
        
        // 创建 HeaderExchangeChannel 对象
        this.channel = new HeaderExchangeChannel(client);
        
        // 以下代码均与心跳检测逻辑有关
        String dubbo = client.getUrl().getParameter(Constants.DUBBO_VERSION_KEY);
        this.heartbeat = client.getUrl().getParameter(Constants.HEARTBEAT_KEY, dubbo != null && dubbo.startsWith("1.0.") ? Constants.DEFAULT_HEARTBEAT : 0);
        this.heartbeatTimeout = client.getUrl().getParameter(Constants.HEARTBEAT_TIMEOUT_KEY, heartbeat * 3);
        if (heartbeatTimeout < heartbeat * 2) {
            throw new IllegalStateException("heartbeatTimeout < heartbeatInterval * 2");
        }
        if (needHeartbeat) {
            // 开启心跳检测定时器
            startHeartbeatTimer();
        }
    }

    @Override
    public ResponseFuture request(Object request) throws RemotingException {
        // 直接 HeaderExchangeChannel 对象的同签名方法
        return channel.request(request);
    }

    @Override
    public ResponseFuture request(Object request, int timeout) throws RemotingException {
        // 直接 HeaderExchangeChannel 对象的同签名方法
        return channel.request(request, timeout);
    }

    @Override
    public void close() {
        doClose();
        channel.close();
    }
    
    private void doClose() {
        // 停止心跳检测定时器
        stopHeartbeatTimer();
    }

    private void startHeartbeatTimer() {
        stopHeartbeatTimer();
        if (heartbeat > 0) {
            heartbeatTimer = scheduled.scheduleWithFixedDelay(
                    new HeartBeatTask(new HeartBeatTask.ChannelProvider() {
                        @Override
                        public Collection<Channel> getChannels() {
                            return Collections.<Channel>singletonList(HeaderExchangeClient.this);
                        }
                    }, heartbeat, heartbeatTimeout),
                    heartbeat, heartbeat, TimeUnit.MILLISECONDS);
        }
    }

    private void stopHeartbeatTimer() {
        if (heartbeatTimer != null && !heartbeatTimer.isCancelled()) {
            try {
                heartbeatTimer.cancel(true);
                scheduled.purge();
            } catch (Throwable e) {
                if (logger.isWarnEnabled()) {
                    logger.warn(e.getMessage(), e);
                }
            }
        }
        heartbeatTimer = null;
    }
    
    // 省略部分方法
}
```



#### HeaderExchangeChannel#request

```java
// 首先定义了一个 Request 对象，然后再将该对象传给 NettyClient 的 send 方法，进行后续的调用。需要说明的是，NettyClient 中并未实现 send 方法，该方法继承自父类 AbstractPeer
final class HeaderExchangeChannel implements ExchangeChannel {
    
    private final Channel channel;
    
    HeaderExchangeChannel(Channel channel) {
        if (channel == null) {
            throw new IllegalArgumentException("channel == null");
        }
        
        // 这里的 channel 指向的是 NettyClient
        this.channel = channel;
    }
    
    @Override
    public ResponseFuture request(Object request) throws RemotingException {
        return request(request, channel.getUrl().getPositiveParameter(Constants.TIMEOUT_KEY, Constants.DEFAULT_TIMEOUT));
    }

    @Override
    public ResponseFuture request(Object request, int timeout) throws RemotingException {
        if (closed) {
            throw new RemotingException(..., "Failed to send request ...);
        }
        // 创建 Request 对象
        Request req = new Request();
        req.setVersion(Version.getProtocolVersion());
        // 设置双向通信标志为 true
        req.setTwoWay(true);
        // 这里的 request 变量类型为 RpcInvocation
        req.setData(request);
                                        
        // 创建 DefaultFuture 对象
        DefaultFuture future = new DefaultFuture(channel, req, timeout);
        try {
            // 调用 NettyClient 的 send 方法发送请求
            channel.send(req);
        } catch (RemotingException e) {
            future.cancel();
            throw e;
        }
        // 返回 DefaultFuture 对象
        return future;
    }
}
```



#### DefaultFuture

绑定 request（Invoker）对象 

```java
public class DefaultFuture implements ResponseFuture {
    
    private static final Map<Long, Channel> CHANNELS = 
        new ConcurrentHashMap<Long, Channel>();

    private static final Map<Long, DefaultFuture> FUTURES = 
        new ConcurrentHashMap<Long, DefaultFuture>();
    
    private final long id;
    private final Channel channel;
    private final Request request;
    private final int timeout;
    private final Lock lock = new ReentrantLock();
    private final Condition done = lock.newCondition();
    private volatile Response response;
    
    public DefaultFuture(Channel channel, Request request, int timeout) {
        this.channel = channel;
        this.request = request;
        
        // 获取请求 id，这个 id 很重要，后面还会见到
        this.id = request.getId();
        this.timeout = timeout > 0 ? timeout : channel.getUrl().getPositiveParameter(Constants.TIMEOUT_KEY, Constants.DEFAULT_TIMEOUT);
        // 存储 <requestId, DefaultFuture> 映射关系到 FUTURES 中
        FUTURES.put(id, this);
        CHANNELS.put(id, channel);
    }
    
    @Override
    public Object get() throws RemotingException {
        return get(timeout);
    }

    @Override
    public Object get(int timeout) throws RemotingException {
        if (timeout <= 0) {
            timeout = Constants.DEFAULT_TIMEOUT;
        }
        
        // 检测服务提供方是否成功返回了调用结果
        if (!isDone()) {
            long start = System.currentTimeMillis();
            lock.lock();
            try {
                // 循环检测服务提供方是否成功返回了调用结果
                while (!isDone()) {
                    // 如果调用结果尚未返回，这里等待一段时间
                    done.await(timeout, TimeUnit.MILLISECONDS);
                    // 如果调用结果成功返回，或等待超时，此时跳出 while 循环，执行后续的逻辑
                    if (isDone() || System.currentTimeMillis() - start > timeout) {
                        break;
                    }
                }
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            } finally {
                lock.unlock();
            }
            
            // 如果调用结果仍未返回，则抛出超时异常
            if (!isDone()) {
                throw new TimeoutException(sent > 0, channel, getTimeoutMessage(false));
            }
        }
        
        // 返回调用结果
        return returnFromResponse();
    }
    
    @Override
    public boolean isDone() {
        // 通过检测 response 字段为空与否，判断是否收到了调用结果
        return response != null;
    }
    
    private Object returnFromResponse() throws RemotingException {
        Response res = response;
        if (res == null) {
            throw new IllegalStateException("response cannot be null");
        }
        
        // 如果调用结果的状态为 Response.OK，则表示调用过程正常，服务提供方成功返回了调用结果
        if (res.getStatus() == Response.OK) {
            return res.getResult();
        }
        
        // 抛出异常
        if (res.getStatus() == Response.CLIENT_TIMEOUT || res.getStatus() == Response.SERVER_TIMEOUT) {
            throw new TimeoutException(res.getStatus() == Response.SERVER_TIMEOUT, channel, res.getErrorMessage());
        }
        throw new RemotingException(channel, res.getErrorMessage());
    }
    
    // 省略其他方法
}
```



#### AbstractPeer#send

```java
public abstract class AbstractPeer implements Endpoint, ChannelHandler {
    
    @Override
    public void send(Object message) throws RemotingException {
        // 该方法由 AbstractClient 类实现
        send(message, url.getParameter(Constants.SENT_KEY, false));
    }
    
    // 省略其他方法
}

public abstract class AbstractClient extends AbstractEndpoint implements Client {
    
    @Override
    public void send(Object message, boolean sent) throws RemotingException {
        if (send_reconnect && !isConnected()) {
            connect();
        }
        
        // 获取 Channel，getChannel 是一个抽象方法，具体由子类实现
        Channel channel = getChannel();
        if (channel == null || !channel.isConnected()) {
            throw new RemotingException(this, "message can not send ...");
        }
        
        // 继续向下调用
        channel.send(message, sent);
    }
    
    protected abstract Channel getChannel();
    
    // 省略其他方法
}
```



#### NettyClient

##### NettyChannel#send

```java
public class NettyClient extends AbstractClient {
    
    // 这里的 Channel 全限定名称为 org.jboss.netty.channel.Channel
    private volatile Channel channel;

    @Override
    protected com.alibaba.dubbo.remoting.Channel getChannel() {
        Channel c = channel;
        if (c == null || !c.isConnected())
            return null;
        // 获取一个 NettyChannel 类型对象
        return NettyChannel.getOrAddChannel(c, getUrl(), this);
    }
}

final class NettyChannel extends AbstractChannel {

    private static final ConcurrentMap<org.jboss.netty.channel.Channel, NettyChannel> channelMap = 
        new ConcurrentHashMap<org.jboss.netty.channel.Channel, NettyChannel>();

    private final org.jboss.netty.channel.Channel channel;
    
    /** 私有构造方法 */
    private NettyChannel(org.jboss.netty.channel.Channel channel, URL url, ChannelHandler handler) {
        super(url, handler);
        if (channel == null) {
            throw new IllegalArgumentException("netty channel == null;");
        }
        this.channel = channel;
    }

    static NettyChannel getOrAddChannel(org.jboss.netty.channel.Channel ch, URL url, ChannelHandler handler) {
        if (ch == null) {
            return null;
        }
        
        // 尝试从集合中获取 NettyChannel 实例
        NettyChannel ret = channelMap.get(ch);
        if (ret == null) {
            // 如果 ret = null，则创建一个新的 NettyChannel 实例
            NettyChannel nc = new NettyChannel(ch, url, handler);
            if (ch.isConnected()) {
                // 将 <Channel, NettyChannel> 键值对存入 channelMap 集合中
                ret = channelMap.putIfAbsent(ch, nc);
            }
            if (ret == null) {
                ret = nc;
            }
        }
        return ret;
    }
}


public void send(Object message, boolean sent) throws RemotingException {
    super.send(message, sent);

    boolean success = true;
    int timeout = 0;
    try {
        // 发送消息(包含请求和响应消息)
        ChannelFuture future = channel.write(message);
        
        // sent 的值源于 <dubbo:method sent="true/false" /> 中 sent 的配置值，有两种配置值：
        //   1. true: 等待消息发出，消息发送失败将抛出异常
        //   2. false: 不等待消息发出，将消息放入 IO 队列，即刻返回
        // 默认情况下 sent = false；
        if (sent) {
            timeout = getUrl().getPositiveParameter(Constants.TIMEOUT_KEY, Constants.DEFAULT_TIMEOUT);
            // 等待消息发出，若在规定时间没能发出，success 会被置为 false
            success = future.await(timeout);
        }
        Throwable cause = future.getCause();
        if (cause != null) {
            throw cause;
        }
    } catch (Throwable e) {
        throw new RemotingException(this, "Failed to send message ...");
    }

    // 若 success 为 false，这里抛出异常
    if (!success) {
        throw new RemotingException(this, "Failed to send message ...");
    }
}
```



### 调用链

```
proxy0#sayHello(String)
  —> InvokerInvocationHandler#invoke(Object, Method, Object[])
    —> MockClusterInvoker#invoke(Invocation)
      —> AbstractClusterInvoker#invoke(Invocation)
        —> FailoverClusterInvoker#doInvoke(Invocation, List<Invoker<T>>, LoadBalance)
          —> Filter#invoke(Invoker, Invocation)  // 包含多个 Filter 调用
            —> ListenerInvokerWrapper#invoke(Invocation) 
              —> AbstractInvoker#invoke(Invocation) 
                —> DubboInvoker#doInvoke(Invocation)
                  —> ReferenceCountExchangeClient#request(Object, int)
                    —> HeaderExchangeClient#request(Object, int)
                      —> HeaderExchangeChannel#request(Object, int)
                        —> AbstractPeer#send(Object)
                          —> AbstractClient#send(Object, boolean)
                            —> NettyChannel#send(Object, boolean)
                              —> NioClientSocketChannel#write(Object)
```



#### 请求编解码

首先会通过位运算将消息头写入到 header 数组中。然后对 Request 对象的 data 字段执行序列化操作，序列化后的数据最终会存储到 ChannelBuffer 中。序列化操作执行完后，可得到数据序列化后的长度 len，紧接着将 len 写入到 header 指定位置处。最后再将消息头字节数组 header 写入到 ChannelBuffer 中，整个编码过程就结束了。

```java
public class ExchangeCodec extends TelnetCodec {

    // 消息头长度
    protected static final int HEADER_LENGTH = 16;
    // 魔数内容
    protected static final short MAGIC = (short) 0xdabb;
    protected static final byte MAGIC_HIGH = Bytes.short2bytes(MAGIC)[0];
    protected static final byte MAGIC_LOW = Bytes.short2bytes(MAGIC)[1];
    protected static final byte FLAG_REQUEST = (byte) 0x80;
    protected static final byte FLAG_TWOWAY = (byte) 0x40;
    protected static final byte FLAG_EVENT = (byte) 0x20;
    protected static final int SERIALIZATION_MASK = 0x1f;
    private static final Logger logger = LoggerFactory.getLogger(ExchangeCodec.class);

    public Short getMagicCode() {
        return MAGIC;
    }

    @Override
    public void encode(Channel channel, ChannelBuffer buffer, Object msg) throws IOException {
        if (msg instanceof Request) {
            // 对 Request 对象进行编码
            encodeRequest(channel, buffer, (Request) msg);
        } else if (msg instanceof Response) {
            // 对 Response 对象进行编码，后面分析
            encodeResponse(channel, buffer, (Response) msg);
        } else {
            super.encode(channel, buffer, msg);
        }
    }

    protected void encodeRequest(Channel channel, ChannelBuffer buffer, Request req) throws IOException {
        Serialization serialization = getSerialization(channel);

        // 创建消息头字节数组，长度为 16
        byte[] header = new byte[HEADER_LENGTH];

        // 设置魔数
        Bytes.short2bytes(MAGIC, header);

        // 设置数据包类型（Request/Response）和序列化器编号
        header[2] = (byte) (FLAG_REQUEST | serialization.getContentTypeId());

        // 设置通信方式(单向/双向)
        if (req.isTwoWay()) {
            header[2] |= FLAG_TWOWAY;
        }
        
        // 设置事件标识
        if (req.isEvent()) {
            header[2] |= FLAG_EVENT;
        }

        // 设置请求编号，8个字节，从第4个字节开始设置
        Bytes.long2bytes(req.getId(), header, 4);

        // 获取 buffer 当前的写位置
        int savedWriteIndex = buffer.writerIndex();
        // 更新 writerIndex，为消息头预留 16 个字节的空间
        buffer.writerIndex(savedWriteIndex + HEADER_LENGTH);
        ChannelBufferOutputStream bos = new ChannelBufferOutputStream(buffer);
        // 创建序列化器，比如 Hessian2ObjectOutput
        ObjectOutput out = serialization.serialize(channel.getUrl(), bos);
        if (req.isEvent()) {
            // 对事件数据进行序列化操作
            encodeEventData(channel, out, req.getData());
        } else {
            // 对请求数据进行序列化操作
            encodeRequestData(channel, out, req.getData(), req.getVersion());
        }
        out.flushBuffer();
        if (out instanceof Cleanable) {
            ((Cleanable) out).cleanup();
        }
        bos.flush();
        bos.close();
        
        // 获取写入的字节数，也就是消息体长度
        int len = bos.writtenBytes();
        checkPayload(channel, len);

        // 将消息体长度写入到消息头中
        Bytes.int2bytes(len, header, 12);

        // 将 buffer 指针移动到 savedWriteIndex，为写消息头做准备
        buffer.writerIndex(savedWriteIndex);
        // 从 savedWriteIndex 下标处写入消息头
        buffer.writeBytes(header);
        // 设置新的 writerIndex，writerIndex = 原写下标 + 消息头长度 + 消息体长度
        buffer.writerIndex(savedWriteIndex + HEADER_LENGTH + len);
    }
    
    // 省略其他方法
}

public class DubboCodec extends ExchangeCodec implements Codec2 {
    
	protected void encodeRequestData(Channel channel, ObjectOutput out, Object data, String version) throws IOException {
        RpcInvocation inv = (RpcInvocation) data;

        // 依次序列化 dubbo version、path、version
        out.writeUTF(version);
        out.writeUTF(inv.getAttachment(Constants.PATH_KEY));
        out.writeUTF(inv.getAttachment(Constants.VERSION_KEY));

        // 序列化调用方法名
        out.writeUTF(inv.getMethodName());
        // 将参数类型转换为字符串，并进行序列化
        out.writeUTF(ReflectUtils.getDesc(inv.getParameterTypes()));
        Object[] args = inv.getArguments();
        if (args != null)
            for (int i = 0; i < args.length; i++) {
                // 对运行时参数进行序列化
                out.writeObject(encodeInvocationArgument(channel, inv, i));
            }
        
        // 序列化 attachments
        out.writeObject(inv.getAttachments());
    }
}
```



###### 解

```java
public class ExchangeCodec extends TelnetCodec {
    
    @Override
    public Object decode(Channel channel, ChannelBuffer buffer) throws IOException {
        int readable = buffer.readableBytes();
        // 创建消息头字节数组
        byte[] header = new byte[Math.min(readable, HEADER_LENGTH)];
        // 读取消息头数据
        buffer.readBytes(header);
        // 调用重载方法进行后续解码工作
        return decode(channel, buffer, readable, header);
    }

    @Override
    protected Object decode(Channel channel, ChannelBuffer buffer, int readable, byte[] header) throws IOException {
        // 检查魔数是否相等
        if (readable > 0 && header[0] != MAGIC_HIGH
                || readable > 1 && header[1] != MAGIC_LOW) {
            int length = header.length;
            if (header.length < readable) {
                header = Bytes.copyOf(header, readable);
                buffer.readBytes(header, length, readable - length);
            }
            for (int i = 1; i < header.length - 1; i++) {
                if (header[i] == MAGIC_HIGH && header[i + 1] == MAGIC_LOW) {
                    buffer.readerIndex(buffer.readerIndex() - header.length + i);
                    header = Bytes.copyOf(header, i);
                    break;
                }
            }
            // 通过 telnet 命令行发送的数据包不包含消息头，所以这里
            // 调用 TelnetCodec 的 decode 方法对数据包进行解码
            return super.decode(channel, buffer, readable, header);
        }
        
        // 检测可读数据量是否少于消息头长度，若小于则立即返回 DecodeResult.NEED_MORE_INPUT
        if (readable < HEADER_LENGTH) {
            return DecodeResult.NEED_MORE_INPUT;
        }

        // 从消息头中获取消息体长度
        int len = Bytes.bytes2int(header, 12);
        // 检测消息体长度是否超出限制，超出则抛出异常
        checkPayload(channel, len);

        int tt = len + HEADER_LENGTH;
        // 检测可读的字节数是否小于实际的字节数
        if (readable < tt) {
            return DecodeResult.NEED_MORE_INPUT;
        }
        
        ChannelBufferInputStream is = new ChannelBufferInputStream(buffer, len);

        try {
            // 继续进行解码工作
            return decodeBody(channel, is, header);
        } finally {
            if (is.available() > 0) {
                try {
                    StreamUtils.skipUnusedStream(is);
                } catch (IOException e) {
                    logger.warn(e.getMessage(), e);
                }
            }
        }
    }
}
```

通过检测消息头中的魔数是否与规定的魔数相等，提前拦截掉非常规数据包，比如通过 telnet 命令行发出的数据包。接着再对消息体长度，以及可读字节数进行检测。最后调用 decodeBody 方法进行后续的解码工作，ExchangeCodec 中实现了 decodeBody 方法，但因其子类 DubboCodec 覆写了该方法，所以在运行时 DubboCodec 中的 decodeBody 方法会被调用。

```java
public class DubboCodec extends ExchangeCodec implements Codec2 {

    @Override
    protected Object decodeBody(Channel channel, InputStream is, byte[] header) throws IOException {
        // 获取消息头中的第三个字节，并通过逻辑与运算得到序列化器编号
        byte flag = header[2], proto = (byte) (flag & SERIALIZATION_MASK);
        Serialization s = CodecSupport.getSerialization(channel.getUrl(), proto);
        // 获取调用编号
        long id = Bytes.bytes2long(header, 4);
        // 通过逻辑与运算得到调用类型，0 - Response，1 - Request
        if ((flag & FLAG_REQUEST) == 0) {
            // 对响应结果进行解码，得到 Response 对象。这个非本节内容，后面再分析
            // ...
        } else {
            // 创建 Request 对象
            Request req = new Request(id);
            req.setVersion(Version.getProtocolVersion());
            // 通过逻辑与运算得到通信方式，并设置到 Request 对象中
            req.setTwoWay((flag & FLAG_TWOWAY) != 0);
            
            // 通过位运算检测数据包是否为事件类型
            if ((flag & FLAG_EVENT) != 0) {
                // 设置心跳事件到 Request 对象中
                req.setEvent(Request.HEARTBEAT_EVENT);
            }
            try {
                Object data;
                if (req.isHeartbeat()) {
                    // 对心跳包进行解码，该方法已被标注为废弃
                    data = decodeHeartbeatData(channel, deserialize(s, channel.getUrl(), is));
                } else if (req.isEvent()) {
                    // 对事件数据进行解码
                    data = decodeEventData(channel, deserialize(s, channel.getUrl(), is));
                } else {
                    DecodeableRpcInvocation inv;
                    // 根据 url 参数判断是否在 IO 线程上对消息体进行解码
                    if (channel.getUrl().getParameter(
                            Constants.DECODE_IN_IO_THREAD_KEY,
                            Constants.DEFAULT_DECODE_IN_IO_THREAD)) {
                        inv = new DecodeableRpcInvocation(channel, req, is, proto);
                        // 在当前线程，也就是 IO 线程上进行后续的解码工作。此工作完成后，可将
                        // 调用方法名、attachment、以及调用参数解析出来
                        inv.decode();
                    } else {
                        // 仅创建 DecodeableRpcInvocation 对象，但不在当前线程上执行解码逻辑
                        inv = new DecodeableRpcInvocation(channel, req,
                                new UnsafeByteArrayInputStream(readMessageData(is)), proto);
                    }
                    data = inv;
                }
                
                // 设置 data 到 Request 对象中
                req.setData(data);
            } catch (Throwable t) {
                // 若解码过程中出现异常，则将 broken 字段设为 true，
                // 并将异常对象设置到 Reqeust 对象中
                req.setBroken(true);
                req.setData(t);
            }
            return req;
        }
    }
}
```



###### DecodeableRpcInvocation 

```java
public class DecodeableRpcInvocation extends RpcInvocation implements Codec, Decodeable {
    
	@Override
    public Object decode(Channel channel, InputStream input) throws IOException {
        ObjectInput in = CodecSupport.getSerialization(channel.getUrl(), serializationType)
                .deserialize(channel.getUrl(), input);

        // 通过反序列化得到 dubbo version，并保存到 attachments 变量中
        String dubboVersion = in.readUTF();
        request.setVersion(dubboVersion);
        setAttachment(Constants.DUBBO_VERSION_KEY, dubboVersion);

        // 通过反序列化得到 path，version，并保存到 attachments 变量中
        setAttachment(Constants.PATH_KEY, in.readUTF());
        setAttachment(Constants.VERSION_KEY, in.readUTF());

        // 通过反序列化得到调用方法名
        setMethodName(in.readUTF());
        try {
            Object[] args;
            Class<?>[] pts;
            // 通过反序列化得到参数类型字符串，比如 Ljava/lang/String;
            String desc = in.readUTF();
            if (desc.length() == 0) {
                pts = DubboCodec.EMPTY_CLASS_ARRAY;
                args = DubboCodec.EMPTY_OBJECT_ARRAY;
            } else {
                // 将 desc 解析为参数类型数组
                pts = ReflectUtils.desc2classArray(desc);
                args = new Object[pts.length];
                for (int i = 0; i < args.length; i++) {
                    try {
                        // 解析运行时参数
                        args[i] = in.readObject(pts[i]);
                    } catch (Exception e) {
                        if (log.isWarnEnabled()) {
                            log.warn("Decode argument failed: " + e.getMessage(), e);
                        }
                    }
                }
            }
            
            // 设置参数类型数组
            setParameterTypes(pts);

            // 通过反序列化得到原 attachment 的内容
            Map<String, String> map = (Map<String, String>) in.readObject(Map.class);
            if (map != null && map.size() > 0) {
                Map<String, String> attachment = getAttachments();
                if (attachment == null) {
                    attachment = new HashMap<String, String>();
                }
                // 将 map 与当前对象中的 attachment 集合进行融合
                attachment.putAll(map);
                setAttachments(attachment);
            }
            
            // 对 callback 类型的参数进行处理
            for (int i = 0; i < args.length; i++) {
                args[i] = decodeInvocationArgument(channel, this, pts, i, args[i]);
            }

            // 设置参数列表
            setArguments(args);

        } catch (ClassNotFoundException e) {
            throw new IOException(StringUtils.toString("Read invocation data failed.", e));
        } finally {
            if (in instanceof Cleanable) {
                ((Cleanable) in).cleanup();
            }
        }
        return this;
    }
}
```



### 调用服务

解码器将数据包解析成 Request 对象后，NettyHandler 的 messageReceived 方法紧接着会收到这个对象，并将这个对象继续向下传递。这期间该对象会被依次传递给 NettyServer、MultiMessageHandler、HeartbeatHandler 以及 AllChannelHandler。最后由 AllChannelHandler 将该对象封装到 Runnable 实现类对象中，并将 Runnable 放入线程池中执行后续的调用逻辑。

```java
NettyHandler#messageReceived(ChannelHandlerContext, MessageEvent)
  —> AbstractPeer#received(Channel, Object)
    —> MultiMessageHandler#received(Channel, Object)
      —> HeartbeatHandler#received(Channel, Object)
        —> AllChannelHandler#received(Channel, Object)
          —> ExecutorService#execute(Runnable)    // 由线程池执行后续的调用逻辑
    
ChannelEventRunnable#run()
  —> DecodeHandler#received(Channel, Object)
    —> HeaderExchangeHandler#received(Channel, Object)
      —> HeaderExchangeHandler#handleRequest(ExchangeChannel, Request)
        —> DubboProtocol.requestHandler#reply(ExchangeChannel, Object)
          —> Filter#invoke(Invoker, Invocation)
            —> AbstractProxyInvoker#invoke(Invocation)
              —> Wrapper0#invokeMethod(Object, String, Class[], Object[])
                —> DemoServiceImpl#sayHello(String)    
```

#### NettyHandler

```java
@Sharable
public class NettyHandler extends SimpleChannelHandler {
    
    private final Map<String, Channel> channels = new ConcurrentHashMap<String, Channel>();

    private final URL url;

    private final ChannelHandler handler;
    
    public NettyHandler(URL url, ChannelHandler handler) {
        if (url == null) {
            throw new IllegalArgumentException("url == null");
        }
        if (handler == null) {
            throw new IllegalArgumentException("handler == null");
        }
        this.url = url;
        
        // 这里的 handler 类型为 NettyServer
        this.handler = handler;
    }
    
	public void messageReceived(ChannelHandlerContext ctx, MessageEvent e) throws Exception {
        // 获取 NettyChannel
        NettyChannel channel = NettyChannel.getOrAddChannel(ctx.getChannel(), url, handler);
        try {
            // 继续向下调用
            handler.received(channel, e.getMessage());
        } finally {
            NettyChannel.removeChannelIfDisconnected(ctx.getChannel());
        }
    }
}
```



#### AllChannelHandler

```java
public class AllChannelHandler extends WrappedChannelHandler {

    public AllChannelHandler(ChannelHandler handler, URL url) {
        super(handler, url);
    }

    /** 处理连接事件 */
    @Override
    public void connected(Channel channel) throws RemotingException {
        // 获取线程池
        ExecutorService cexecutor = getExecutorService();
        try {
            // 将连接事件派发到线程池中处理
            cexecutor.execute(new ChannelEventRunnable(channel, handler, ChannelState.CONNECTED));
        } catch (Throwable t) {
            throw new ExecutionException(..., " error when process connected event .", t);
        }
    }

    /** 处理断开事件 */
    @Override
    public void disconnected(Channel channel) throws RemotingException {
        ExecutorService cexecutor = getExecutorService();
        try {
            cexecutor.execute(new ChannelEventRunnable(channel, handler, ChannelState.DISCONNECTED));
        } catch (Throwable t) {
            throw new ExecutionException(..., "error when process disconnected event .", t);
        }
    }

    /** 处理请求和响应消息，这里的 message 变量类型可能是 Request，也可能是 Response */
    @Override
    public void received(Channel channel, Object message) throws RemotingException {
        ExecutorService cexecutor = getExecutorService();
        try {
            // 将请求和响应消息派发到线程池中处理
            cexecutor.execute(new ChannelEventRunnable(channel, handler, ChannelState.RECEIVED, message));
        } catch (Throwable t) {
            if(message instanceof Request && t instanceof RejectedExecutionException){
                Request request = (Request)message;
                // 如果通信方式为双向通信，此时将 Server side ... threadpool is exhausted 
                // 错误信息封装到 Response 中，并返回给服务消费方。
                if(request.isTwoWay()){
                    String msg = "Server side(" + url.getIp() + "," + url.getPort() 
                        + ") threadpool is exhausted ,detail msg:" + t.getMessage();
                    Response response = new Response(request.getId(), request.getVersion());
                    response.setStatus(Response.SERVER_THREADPOOL_EXHAUSTED_ERROR);
                    response.setErrorMessage(msg);
                    // 返回包含错误信息的 Response 对象
                    channel.send(response);
                    return;
                }
            }
            throw new ExecutionException(..., " error when process received event .", t);
        }
    }

    /** 处理异常信息 */
    @Override
    public void caught(Channel channel, Throwable exception) throws RemotingException {
        ExecutorService cexecutor = getExecutorService();
        try {
            cexecutor.execute(new ChannelEventRunnable(channel, handler, ChannelState.CAUGHT, exception));
        } catch (Throwable t) {
            throw new ExecutionException(..., "error when process caught event ...");
        }
    }
}
```

#### ChannelEventRunnable

```java
public class ChannelEventRunnable implements Runnable {
    
    private final ChannelHandler handler;
    private final Channel channel;
    private final ChannelState state;
    private final Throwable exception;
    private final Object message;
    
    @Override
    public void run() {
        // 检测通道状态，对于请求或响应消息，此时 state = RECEIVED
        if (state == ChannelState.RECEIVED) {
            try {
                // 将 channel 和 message 传给 ChannelHandler 对象，进行后续的调用
                handler.received(channel, message);
            } catch (Exception e) {
                logger.warn("... operation error, channel is ... message is ...");
            }
        } 
        
        // 其他消息类型通过 switch 进行处理
        else {
            switch (state) {
            case CONNECTED:
                try {
                    handler.connected(channel);
                } catch (Exception e) {
                    logger.warn("... operation error, channel is ...");
                }
                break;
            case DISCONNECTED:
                // ...
            case SENT:
                // ...
            case CAUGHT:
                // ...
            default:
                logger.warn("unknown state: " + state + ", message is " + message);
            }
        }

    }
}
```

#### DecodeHandler

```java
public class DecodeHandler extends AbstractChannelHandlerDelegate {

    public DecodeHandler(ChannelHandler handler) {
        super(handler);
    }

    @Override
    public void received(Channel channel, Object message) throws RemotingException {
        if (message instanceof Decodeable) {
            // 对 Decodeable 接口实现类对象进行解码
            decode(message);
        }

        if (message instanceof Request) {
            // 对 Request 的 data 字段进行解码
            decode(((Request) message).getData());
        }

        if (message instanceof Response) {
            // 对 Request 的 result 字段进行解码
            decode(((Response) message).getResult());
        }

        // 执行后续逻辑
        handler.received(channel, message);
    }

    private void decode(Object message) {
        // Decodeable 接口目前有两个实现类，
        // 分别为 DecodeableRpcInvocation 和 DecodeableRpcResult
        if (message != null && message instanceof Decodeable) {
            try {
                // 执行解码逻辑
                ((Decodeable) message).decode();
            } catch (Throwable e) {
                if (log.isWarnEnabled()) {
                    log.warn("Call Decodeable.decode failed: " + e.getMessage(), e);
                }
            }
        }
    }
}
```

#### HeaderExchangeHandler

对于双向通信，HeaderExchangeHandler 首先向后进行调用，得到调用结果。然后将调用结果封装到 Response 对象中，最后再将该对象返回给服务消费方。如果请求不合法，或者调用失败，则将错误信息封装到 Response 对象中，并返回给服务消费方。

```java
public class HeaderExchangeHandler implements ChannelHandlerDelegate {

    private final ExchangeHandler handler;

    public HeaderExchangeHandler(ExchangeHandler handler) {
        if (handler == null) {
            throw new IllegalArgumentException("handler == null");
        }
        this.handler = handler;
    }

    @Override
    public void received(Channel channel, Object message) throws RemotingException {
        channel.setAttribute(KEY_READ_TIMESTAMP, System.currentTimeMillis());
        ExchangeChannel exchangeChannel = HeaderExchangeChannel.getOrAddChannel(channel);
        try {
            // 处理请求对象
            if (message instanceof Request) {
                Request request = (Request) message;
                if (request.isEvent()) {
                    // 处理事件
                    handlerEvent(channel, request);
                } 
                // 处理普通的请求
                else {
                    // 双向通信
                    if (request.isTwoWay()) {
                        // 向后调用服务，并得到调用结果
                        Response response = handleRequest(exchangeChannel, request);
                        // 将调用结果返回给服务消费端
                        channel.send(response);
                    } 
                    // 如果是单向通信，仅向后调用指定服务即可，无需返回调用结果
                    else {
                        handler.received(exchangeChannel, request.getData());
                    }
                }
            }      
            // 处理响应对象，服务消费方会执行此处逻辑，后面分析
            else if (message instanceof Response) {
                handleResponse(channel, (Response) message);
            } else if (message instanceof String) {
                // telnet 相关，忽略
            } else {
                handler.received(exchangeChannel, message);
            }
        } finally {
            HeaderExchangeChannel.removeChannelIfDisconnected(channel);
        }
    }

    Response handleRequest(ExchangeChannel channel, Request req) throws RemotingException {
        Response res = new Response(req.getId(), req.getVersion());
        // 检测请求是否合法，不合法则返回状态码为 BAD_REQUEST 的响应
        if (req.isBroken()) {
            Object data = req.getData();

            String msg;
            if (data == null)
                msg = null;
            else if
                (data instanceof Throwable) msg = StringUtils.toString((Throwable) data);
            else
                msg = data.toString();
            res.setErrorMessage("Fail to decode request due to: " + msg);
            // 设置 BAD_REQUEST 状态
            res.setStatus(Response.BAD_REQUEST);

            return res;
        }
        
        // 获取 data 字段值，也就是 RpcInvocation 对象
        Object msg = req.getData();
        try {
            // 继续向下调用
            Object result = handler.reply(channel, msg);
            // 设置 OK 状态码
            res.setStatus(Response.OK);
            // 设置调用结果
            res.setResult(result);
        } catch (Throwable e) {
            // 若调用过程出现异常，则设置 SERVICE_ERROR，表示服务端异常
            res.setStatus(Response.SERVICE_ERROR);
            res.setErrorMessage(StringUtils.toString(e));
        }
        return res;
    }
}
```

#### DubboProtocol#requestHandler

```java
public class DubboProtocol extends AbstractProtocol {

    public static final String NAME = "dubbo";
    
    private ExchangeHandler requestHandler = new ExchangeHandlerAdapter() {

        @Override
        public Object reply(ExchangeChannel channel, Object message) throws RemotingException {
            if (message instanceof Invocation) {
                Invocation inv = (Invocation) message;
                // 获取 Invoker 实例
                Invoker<?> invoker = getInvoker(channel, inv);
                if (Boolean.TRUE.toString().equals(inv.getAttachments().get(IS_CALLBACK_SERVICE_INVOKE))) {
                    // 回调相关，忽略
                }
                RpcContext.getContext().setRemoteAddress(channel.getRemoteAddress());
                // 通过 Invoker 调用具体的服务
                return invoker.invoke(inv);
            }
            throw new RemotingException(channel, "Unsupported request: ...");
        }
        
        // 忽略其他方法
    }
//--------------param 客户端传来的 invoker ----------------------------------    
    Invoker<?> getInvoker(Channel channel, Invocation inv) throws RemotingException {
        // 忽略回调和本地存根相关逻辑
        // ...
        
        int port = channel.getLocalAddress().getPort();
        
        // 计算 service key，格式为 groupName/serviceName:serviceVersion:port。比如：
        //   dubbo/com.alibaba.dubbo.demo.DemoService:1.0.0:20880
        String serviceKey = serviceKey(port, path, inv.getAttachments().get(Constants.VERSION_KEY), inv.getAttachments().get(Constants.GROUP_KEY));

        // 从 exporterMap 查找与 serviceKey 相对应的 DubboExporter 对象，
        // 服务导出过程中会将 <serviceKey, DubboExporter> 映射关系存储到 exporterMap 集合中
        DubboExporter<?> exporter = (DubboExporter<?>) exporterMap.get(serviceKey);

        if (exporter == null)
            throw new RemotingException(channel, "Not found exported service ...");

        // 获取 Invoker 对象，并返回
        return exporter.getInvoker();
    }
    
    // 忽略其他方法
}

public abstract class AbstractProxyInvoker<T> implements Invoker<T> {

    @Override
    public Result invoke(Invocation invocation) throws RpcException {
        try {
            // 调用 doInvoke 执行后续的调用，并将调用结果封装到 RpcResult 中，并
            return new RpcResult(doInvoke(proxy, invocation.getMethodName(), invocation.getParameterTypes(), invocation.getArguments()));
        } catch (InvocationTargetException e) {
            return new RpcResult(e.getTargetException());
        } catch (Throwable e) {
            throw new RpcException("Failed to invoke remote proxy method ...");
        }
    }
    
    protected abstract Object doInvoke(T proxy, String methodName, Class<?>[] parameterTypes, Object[] arguments) throws Throwable;
}
```



 返回调用结果

响应编解码

# Dubbo插件化

Dubbo的插件化实现非常类似于原生的 JAVA 的 SPI ：它只是提供一种协议，并没有提供相关插件化实施的接口。用过的同学都知道，它有一种java原生的支持类：ServiceLoader，通过声明接口的实现类，在META-INF/services中注册一个实现类，然后通过ServiceLoader去生成一个接口实例，当更换插件的时候只需要把自己实现的插件替换到META-INF/services中即可。

 

## Dubbo的“SPI”

Dubbo 的 SPI 并非原生的 SPI，Dubbo 的规则是在`META-INF/dubbo`、`META-INF/dubbo/internal`或者`META-INF/services`下面以需要实现的接口去创建一个文件，并且在文件中以properties规则一样配置实现类的全面以及分配实现的一个名称。我们看一下dubbo-cluster模块的META-INF.dubbo.internal：

## 实现自己的扩展点

假如我们使用自己定义的协议 MyDefineProtocol

1. 在resources目录下新建META-INF/dubbo/com.alibaba.dubbo.rpc.Protocol文件，文件内容为com.***.MyDefineProtocol

2. 实现类的内容

3. 最后在main方法中调用

4. 通过结果可以看到我们已经找到

 

## 源码分析(自适应扩展)

dubbo的扩展点框架主要位于这个包下：

​    com.alibaba.dubbo.common.extension

大概结构如下：

```
com.alibaba.dubbo.common.extension  
|  
|--factory  
|     |--AdaptiveExtensionFactory   #稍后解释  
|     |--SpiExtensionFactory        #稍后解释  
|  
|--support  
|     |--ActivateComparator  
|  
|--Activate  #自动激活加载扩展的注解  
|--Adaptive  #自适应扩展点的注解  
|--ExtensionFactory  #扩展点对象生成工厂接口  
|--ExtensionLoader   #扩展点加载器，扩展点的查找，校验，加载等核心逻辑的实现类  
|--@SPI   #扩展点注解 
```



### ExtensionLoader

#### getExtensionLoader

其中最核心的类就是`ExtensionLoader`，几乎所有特性都在这个类中实现。

ExtensionLoader没有提供public的构造方法，但是提供了一个public static的 getExtensionLoader，这个方法就是获取ExtensionLoader实例的工厂方法。其public成员方法中有三个比较重要的方法：

getActivateExtension ：根据条件获取当前扩展可自动激活的实现

getExtension ： 根据名称获取当前扩展的指定实现

getAdaptiveExtension : 获取当前扩展的自适应实现

> 该方法需要一个Class类型的参数，该参数表示希望加载的扩展点类型，该参数必须是接口，且该接口必须被@SPI注解注释，否则拒绝处理。检查通过之后首先会检查ExtensionLoader缓存中是否已经存在该扩展对应的 ExtensionLoader，如果有则直接返回，否则创建一个新的ExtensionLoader负责加载该扩展实现，同时将其缓存起来。可以看到对于每一个扩展，dubbo中只会有一个对应的ExtensionLoader实例。



####ExtensionLoader的私有构造函数

接下来看下ExtensionLoader的私有构造函数：

> 这里保存了对应的扩展类型，并且设置了一个额外的objectFactory属性，他是一个ExtensionFactory类型，ExtensionFactory主要用于加载扩展的实现：

### ExtensionFactory

ExtensionFactory主要用于加载扩展的实现：

> ExtensionFactory有@SPI注解，说明当前这个接口是一个扩展点。从extension包的结构图可以看到。Dubbo内部提供了两个实现类：SpiExtensionFactory和AdaptiveExtensionFactory。不同的实现可以以不同的方式来完成扩展点实现的加载。

### AdpativeExtensionFactory

默认的 ExtensionFactory 实现中，AdaptiveExtensionFactotry 被 @Adaptive注解 注释，也就是它就是ExtensionFactory对应的自适应扩展实现(每个扩展点最多只能有一个自适应实现，如果所有实现中没有被@Adaptive注释的，那么dubbo会动态生成一个自适应实现类)，也就是说，所有对ExtensionFactory调用的地方，实际上调用的都是AdpativeExtensionFactory，那么我们看下他的实现代码：

> 这段代码，其实就相当于一个代理入口，它会遍历当前系统中所有的ExtensionFactory实现来获取指定的扩展实现，获取到扩展实现，遍历完所有ExtensionFactory实现，调用ExtensionLoader的getSupportedExtensions方法来获取ExtensionFactory的所有实现

#### getAdaptiveExtension

从前面ExtensionLoader的私有构造函数中可以看出，在选择ExtensionFactory的时候，并不是调用getExtension(name)来获取某个具体的实现类，而是调用getAdaptiveExtension来获取一个自适应的实现。那么首先我们就来分析一下getAdaptiveExtension这个方法的实现吧：.

> 首先检查缓存的adaptiveInstance是否存在，如果存在则直接使用，否则的话调用createAdaptiveExtension方法来创建新的adaptiveInstance并且缓存起来。也就是说对于某个扩展点，每次调用ExtensionLoader.getAdaptiveExtension获取到的都是同一个实例。



#### createAdaptiveExtension

> 在createAdaptiveExtension方法中，首先通过getAdaptiveExtensionClass方法获取到最终的自适应实现类型，然后实例化一个自适应扩展实现的实例，最后进行扩展点注入操作

#### getAdaptiveExtensionClass

> 他只是简单的调用了getExtensionClasses方法，然后在判adaptiveCalss缓存是否被设置，如果被设置那么直接返回，否则调用createAdaptiveExntesionClass方法动态生成一个自适应实现，关于动态生成自适应实现类然后编译加载并且实例化

#### getExtensionClasses

> 在getExtensionClasses方法中，首先检查缓存的cachedClasses，如果没有再调用loadExtensionClasses 方法来加载，加载完成之后就会进行缓存。也就是说对于每个扩展点，其实现的加载只会执行一次。我们看下loadExtensionClasses方法：

#### loadExtensionClasses

> 从代码里面可以看到，在loadExtensionClasses中首先会检测扩展点在@SPI注解中配置的默认扩展实现的名称，并将其赋值给cachedDefaultName属性进行缓存，后面想要获取该扩展点的默认实现名称就可以直接通过访问cachedDefaultName字段来完成，比如getDefaultExtensionName方法就是这么实现的。从这里的代码中又可以看到，具体的扩展实现类型，是通过调用loadFile方法来加载，分别从一下三个地方加载：META-INF/dubbo/internal/、META-INF/dubbo/   META-INF/services/



* loadFile

调用loadFile方法，代码比较长，主要做了几个事情，有几个变量会赋值

cachedAdaptiveClass : 当前Extension类型对应的AdaptiveExtension类型(只能一个)

cachedWrapperClasses : 当前Extension类型对应的所有Wrapper实现类型(无顺序)

cachedActivates : 当前Extension实现自动激活实现缓存(map,无序)

cachedNames : 扩展点实现类对应的名称(如配置多个名称则值为第一个)

 

当 loadExtensionClasses 方法执行完成之后，还有以下变量被赋值：

cachedDefaultName : 当前扩展点的默认实现名称

 

当getExtensionClasses方法执行完成之后，除了上述变量被赋值之外，还有以下变量被赋值：

cachedClasses : 扩展点实现名称对应的实现类(一个实现类可能有多个名称)

其实也就是说，在调用了getExtensionClasses方法之后，当前扩展点对应的实现类的一些信息就已经加载进来了并且被缓存了。后面的许多操作都可以直接通过这些缓存数据来进行处理了。

 

回到createAdaptiveExtension方法，他调用了getExtesionClasses方法加载扩展点实现信息完成之后，就可以直接通过判断cachedAdaptiveClass缓存字段是否被赋值盘确定当前扩展点是否有默认的AdaptiveExtension实现。如果没有，那么就调用createAdaptiveExtensionClass方法来动态生成一个。在dubbo的扩展点框架中大量的使用了缓存技术。

 

创建自适应扩展点实现类型和实例化就已经完成了，下面就来看下扩展点自动注入的实现injectExtension

### injectExtension

> 这里可以看到，扩展点自动注入的一句就是根据　setter　方法对应的参数类型和　property　名称从ExtensionFactory中查询，如果有返回扩展点实例，那么就进行注入操作。到这里getAdaptiveExtension 方法就分析完毕了。 

### getExtension

这个方法的主要作用是用来获取 ExtensionLoader 实例代表的扩展的指定实现。已扩展实现的名字作为参数，结合前面学习 getAdaptiveExtension 的代码

# 总结

在整个过程中，最重要的两个方法 getExtensionClasses 和 createAdaptiveExtensionClass

 

**getExtensionClasses**

这个方法主要是读取 META-INF/services 、META-INF/dubbo、META-INF/internal 目录下的文件内容

分析每一行，如果发现其中有哪个类的annotation是@Adaptive，就找到对应的AdaptiveClass。如果没有的话，就动态创建一个

**createAdaptiveExtensionClass**

该方法是在getExtensionClasses方法找不到AdaptiveClass的情况下被调用，该方法主要是通过字节码的方式在内存中新生成一个类，它具有AdaptiveClass的功能，Protocol就是通过这种方式获得AdaptiveClass类的。