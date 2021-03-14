# Data Access

`org.springframework.transaction.PlatformTransactionManager`

```
public interface PlatformTransactionManager extends TransactionManager {

    TransactionStatus getTransaction(TransactionDefinition definition) throws TransactionException;

    void commit(TransactionStatus status) throws TransactionException;

    void rollback(TransactionStatus status) throws TransactionException;
}

<bean id="dataSource" class="org.apache.commons.dbcp.BasicDataSource" destroy-method="close">
    <property name="driverClassName" value="${jdbc.driverClassName}" />
    <property name="url" value="${jdbc.url}" />
    <property name="username" value="${jdbc.username}" />
    <property name="password" value="${jdbc.password}" />
</bean>


<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:jee="http://www.springframework.org/schema/jee"
    xsi:schemaLocation="
        http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/jee
        https://www.springframework.org/schema/jee/spring-jee.xsd">

    <jee:jndi-lookup id="dataSource" jndi-name="jdbc/jpetstore"/>

    <bean id="txManager" class="org.springframework.transaction.jta.JtaTransactionManager" />

    <!-- other <bean/> definitions here -->

</beans>
```





`org.springframework.transaction.ReactiveTransactionManager`

Resultset  获取元数据



## 手写ORM框架

Template设计模式



只定义RowMapper接口（resultset，rownum），mapping方法    ORM由自己实现

JDBCTemplate  NamedJDBCTemplate（具名参数  :variableName）

1. 加载驱动类
2. 获取连接
3. 创建语句集（标准，预处理）
4. 执行语句集
5. 获取结果集（增删改拿到int值，影响行数；查询 Resultset）

单表操作实现NoSql
ORM手动=》自动


ORM框架：
1. 自动生成SQL
2. 自动ORM映射

DAO只负责单表操作
多表service里