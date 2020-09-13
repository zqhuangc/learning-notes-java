* mybatis通过接口查找对应的mapper.xml以及方法执行  

获取接口，然后调用接口的方法。只要方法名和对应的mapper.xml中的id名字相同，就可以执行sql。 
那么接口是如何与mapper.xml对应的呢？ 

首先看下，在 getMapper() 方法是如何操作的。 

1. 在 DefaultSqlSession 中调用了configuration.getMapper()
2. 在 Configuration 中调用了 mapperRegistry.getMapper(type, sqlSession)；
3. MapperRegistry 中实现了动态代理
4. addMapper() 的顺序，包名
```java
public void addMappers(String packageName, Class<?> superType) {
    ResolverUtil<Class<?>> resolverUtil = new ResolverUtil<Class<?>>();
    //通过包名，查找该包下所有的接口进行遍历，放入集合中
    resolverUtil.find(new ResolverUtil.IsA(superType), packageName);
    Set<Class<? extends Class<?>>> mapperSet = resolverUtil.getClasses();
    for (Class<?> mapperClass : mapperSet) {
      addMapper(mapperClass);
    }
  }

 //解析包名下的接口
  public void addMappers(String packageName) {
    addMappers(packageName, Object.class);
  }

//该方法的调用是在SqlSessionFactory.build();时对配置文件的解析
```

通过接口的全路径来查找对应的xml。这里有两种方式解析，也就是我们平常xml文件放置位置的两种写法。 
* 第一种是不加namespace，把xml文件放在和接口相同的路径下，同时xml的名字与接口名字相同，如接口名为Student.java，xml文件为Student.xml。在相同的包下。这种当时可以不加namespace.   
* 第二种是加namespace,通过namespace来查找对应的xml. 

到这就是**接口名和xml的全部注册流程**。
### 通过动态代理获取接口名字来对应xml中的id。 
主要有两个类MapperProxyFactory.java和MapperProxy.java 
```java
public class MapperProxyFactory<T> {

  private final Class<T> mapperInterface;
  private Map<Method, MapperMethod> methodCache = new ConcurrentHashMap<Method, MapperMethod>();
    //构造函数，获取接口类
  public MapperProxyFactory(Class<T> mapperInterface) {
    this.mapperInterface = mapperInterface;
  }

  public Class<T> getMapperInterface() {
    return mapperInterface;
  }

  public Map<Method, MapperMethod> getMethodCache() {
    return methodCache;
  }

  @SuppressWarnings("unchecked")
  protected T newInstance(MapperProxy<T> mapperProxy) {
    return (T) Proxy.newProxyInstance(mapperInterface.getClassLoader(), new Class[] { mapperInterface }, mapperProxy);
  }
//供外部调用
  public T newInstance(SqlSession sqlSession) {
    final MapperProxy<T> mapperProxy = new MapperProxy<T>(sqlSession, mapperInterface, methodCache);
    return newInstance(mapperProxy);
  }

}
```
## 在MapperProxy.java中进行方法的执行
```java
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    if (Object.class.equals(method.getDeclaringClass())) {
      try {
        return method.invoke(this, args);
      } catch (Throwable t) {
        throw ExceptionUtil.unwrapThrowable(t);
      }
    }
    final MapperMethod mapperMethod = cachedMapperMethod(method);
    //方法的执行
    return mapperMethod.execute(sqlSession, args);
  }

  private MapperMethod cachedMapperMethod(Method method) {
    MapperMethod mapperMethod = methodCache.get(method);
    if (mapperMethod == null) {
      mapperMethod = new MapperMethod(mapperInterface, method, sqlSession.getConfiguration());
      methodCache.put(method, mapperMethod);
    }
    return mapperMethod;
  }
```