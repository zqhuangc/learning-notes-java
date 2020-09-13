
datasource->connectiondatasource->connection->createStatement


DbUtils
commons-dbutils

queryrunner

ResultSetHandler 接口


### hibernate
configure->sessionfactory->session
创建Configuration 的两种方式
属性文件（hibernate.properties）:
    Configuration cfg = new Configuration();
Xml文件（hibernate.cfg.xml）
    Configuration cfg = new Configuration().configure();
Configuration对象根据当前的配置信息生成 SessionFactory 对象。SessionFactory 对象一旦构造完毕，即被赋予特定的配置信息(SessionFactory 对象中保存了当前的数据库配置信息和所有映射关系以及预定义的SQL语句。同时，SessionFactory还负责维护Hibernate的二级缓存)。 
   Configuration cfg = new Configuration().configure();
   SessionFactory sf = cfg.buildSessionFactory();
是线程安全的。 
SessionFactory是生成Session的工厂：
   Session session = sf.openSession();

Transaction tx = session.beginTransaction();
常用方法:
commit():提交相关联的session实例
rollback():撤销事务操作
wasCommitted():检查事务是否提交

```xml
<hibernate-mapping>
    <class name="cn.itcast.hibernate.datatype.DataType" table="datatype">
     <id name="id" column="id" type="long">
       <generator class="increment"></generator>
     </id>
     <property name="tag" column="tag" type="boolean"></property>
     <property name="createDate" column="createDate" type="date"></property>
     <property name="vip" column="vip" type="character"></property>
     <property name="logTime" column="logTime" type="timestamp"></property>
     <property name="description" column="description" type="binary"></property>
  </class>
</hibernate-mapping>

```

<composite-id>           
<key-property name="name" column="student_name" type="string“/>
<key-property name="cardID" column="card_id" type="string“/>        </composite-id> 


对象映射关系
1:1
1:N
N:1
Map、Set和List的映射

组件映射
HQL

SessionFactory缓存【该类的对象是静态的】
       Hibernate中提供的第二级缓存指的是SessionFactory缓存，是支持可插拔式
   的缓存。该缓存主要是由SessionFactory进行管理。

