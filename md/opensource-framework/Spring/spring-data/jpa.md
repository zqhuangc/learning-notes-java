Spring JPA 支持提供了三种设置 JPA 实体管理工厂的方法，应用程序使用该工厂来获得实体管理器。

- [Using `LocalEntityManagerFactoryBean`](https://docs.spring.io/spring-framework/docs/current/reference/html/data-access.html#orm-jpa-setup-lemfb)

  使用 localentitymanagerybean

- [Obtaining an EntityManagerFactory from JNDI](https://docs.spring.io/spring-framework/docs/current/reference/html/data-access.html#orm-jpa-setup-jndi)

  从 JNDI 获取一个 EntityManagerFactory

- [Using `LocalContainerEntityManagerFactoryBean`](https://docs.spring.io/spring-framework/docs/current/reference/html/data-access.html#orm-jpa-setup-lcemfb)

  使用 localcontainerentitymanagerybean







> For applications that rely on multiple persistence units locations (stored in various JARS in the classpath, for example), Spring offers the `PersistenceUnitManager` to act as a central repository and to avoid the persistence units discovery process, which can be expensive. The default implementation lets multiple locations be specified. These locations are parsed and later retrieved through the persistence unit name. (By default, the classpath is searched for `META-INF/persistence.xml` files.) The following example configures multiple locations:
>
> 对于依赖于多个持久性单元位置的应用程序(例如，存储在类路径中的各种 jar 中) ，Spring 提供 PersistenceUnitManager 作为中央存储库，并避免持久性单元发现过程，这个过程可能代价高昂。默认实现允许指定多个位置。对这些位置进行解析，然后通过持久性单元名称进行检索。(默认情况下，在类路径中搜索 META-INF/persistence.xml 文件。)下面的例子配置了多个位置:

```
<bean id="pum" class="org.springframework.orm.jpa.persistenceunit.DefaultPersistenceUnitManager">
    <property name="persistenceXmlLocations">
        <list>
            <value>org/springframework/orm/jpa/domain/persistence-multi.xml</value>
            <value>classpath:/my/package/**/custom-persistence.xml</value>
            <value>classpath*:META-INF/persistence.xml</value>
        </list>
    </property>
    <property name="dataSources">
        <map>
            <entry key="localDataSource" value-ref="local-db"/>
            <entry key="remoteDataSource" value-ref="remote-db"/>
        </map>
    </property>
    <!-- if no datasource is specified, use this one -->
    <property name="defaultDataSource" ref="remoteDataSource"/>
</bean>

<bean id="emf" class="org.springframework.orm.jpa.LocalContainerEntityManagerFactoryBean">
    <property name="persistenceUnitManager" ref="pum"/>
    <property name="persistenceUnitName" value="myCustomUnit"/>
</bean>
```





`EntityManagerFactory` 



`@PersistenceContext` `EntityManager`

PersistenceAnnotationBeanPostProcessor