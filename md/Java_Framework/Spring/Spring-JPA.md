[参考](https://segmentfault.com/a/1190000019014245)

##　mapping pojo结构详述

* 多对多
```
@Entity
@Table(name = "t_role")
public class Role {

    @Id
    @GeneratedValue(strategy = GenerationType.AUTO)
    private Long id;

    @Column(name = "role_name")
    private String roleName;

    @Column(name = "role_description")
    private String roleDescription;

    @Column
    private boolean enabled;

    //角色下的所用用户
    @JsonIgnore
    @ManyToMany(mappedBy = "roles")
    private Set<User> users = new HashSet<>();
```

```
@Entity
@Table(name = "t_user")
public class User implements Serializable, Cloneable {

    /**
     * 静态long类型常量serialVersionUID的作用：
     * <p>
     * 显示的设置serialVersionUID值就可以保证版本的兼容性，如果你在类中写上了这个值，就算类变动了，
     * 它反序列化的时候也能和文件中的原值匹配上。而新增的值则会设置成零值，删除的值则不会显示。
     */
    private static final long serialVersionUID = -8220100956296048447L;

    @Id
    @GeneratedValue
    private Long id;

    @Column(unique = true)
    private String name;

    @Column(length = 32)
    private String password;

    @Column(length = 32)
    private String salt;

    @Column(precision = 3)
    private int age;

    /**
     * 关系维护端，负责多对多关系的维护
     *
     * @JoinTable 表示关联表的信息，其中：
     * 1.name 表示关联表的名字
     * 2.joinColumns 指定外键的名字，关联到关系维护端Role
     * 3.inverseJoinColumns 指定外键的名字，要关联的关系被维护端
     * 以上完全可以默认，默认情况下：
     * 1.name 主表名_从表名
     * 2.joinColumns 主表_id
     * 3.inverseJoinColumns 从表_id
     */
    @ManyToMany
    @JoinTable(name = "t_user_role", joinColumns = @JoinColumn(name = "user_id"),inverseJoinColumns = @JoinColumn(name = "role_id"))
    private Set<Role> roles = new HashSet<>();
```
### 注解说明

* mappedBy 

mappedBy和inverse的作用是相同的，只不过inverse用在xml文件中，表示的意思是在关系双方都有关系维护任务时，以哪一方为主导，即没有被mappedBy修饰的一方维护.即外键将储存在没有被mappedBy修饰一方.

* @Temporal  

@Temporal是将java.sql.*下的Data等转成java.util.*包下的Date.

* @Transient  

@Transient标记该字段不记录到数据结构


初始化数据结构和数据

可以通过设置hibernate.hbm2ddl.auto的值来达到初始化数据结构的目的，有以下几个值.

create ： 每次加载JPA，重新创建数据库表结构，这就是导致数据库表数据丢失的原因
create-drop ：加载JPA时创建，退出是删除表结构
update ：加载JPA自动更新数据库结构
none/validate :加载JPA时，验证创建数据库表结构,当然none时不做任何操作
在本机开发调试初始化数据的时候可以选择create、update等。

但是应用发布正式版本的时候，对数据库现有的数据或表结构进行自动的更新是很危险的。此时此刻应该由DBA同志通过手工的方式进行后台的数据库操作。

hibernate.hbm2ddl.auto的值建议是none或validate。validate应该是最好的选择：这样 spring在加载之初，如果model层和数据库表结构不同，就会报错，这样有助于技术运维预先发现问题。

当然我们可以通过自定义初始化脚本的方式来实现初始化数据：

通过data: classpath:/database/import.sql的方式来实现--Spring boot亲测通过


#### 遇到的问题

Spring MVC转换JPA多对多对象的json时，无限循环问题
在使用JPA处理多对多关系时，发生了无限循环的问题，代码如下：
```
　@Entity
   @Table(name = "t_menu")
   public class Menu implements Comparable<Menu> {
       @Id
       @GeneratedValue(strategy = GenerationType.AUTO)
       private long id;
       private String menuName;
       private String menuDesc;
       private int priority;
       private String staticIndex;
       private int parantId;
       private boolean enabled;
       @Transient
       private List<Menu> children;

       //菜单所属role
       @ManyToMany(mappedBy = "roleMenus", fetch = FetchType.LAZY)
       private Set<Role> roles = new HashSet<>();

       //菜单所属role
       @ManyToMany(mappedBy = "userMenus", fetch = FetchType.LAZY)
       private Set<User> users = new HashSet<>();
       }
```

上面转换成json数据时，出现无限循环以致栈溢出．针对以上问题可以通过以下注解实现：

@JsonIgnore json转换时忽略某个属性，以断开无限递归，序列化和反序列化均忽略，可以用在字段或get(序列化),set(反序列化)方法上
@JsonBackReference json转换时忽略某个属性，以断开无限递归，序列化时忽略，可以用在字段或get(序列化),set(反序列化)方法上，序列化时,相当于@JsonIgnore
@JsonManagedReference json转换时会被序列化，反序列化时，如果没有该注解，则不会自动注入@JsonBackReference标注的属性

```
  //菜单所属role
    @JsonIgnore
    @ManyToMany(mappedBy = "roleMenus", fetch = FetchType.LAZY)
    private Set<Role> roles = new HashSet<>();

    //菜单所属role
    @JsonBackReference
    @ManyToMany(mappedBy = "userMenus", fetch = FetchType.LAZY)
    private Set<User> users = new HashSet<>();
```





## Properties

```
MySQL5InnoDBDialect(deprecated)

# datasource configuration
spring.datasource.url=jdbc:mysql://localhost:3306/seckill?useSSL=false&serverTimezone=CTT
spring.datasource.username=root
spring.datasource.password=123456



# jpa configuration
# 是否打印 sql
spring.jpa.show-sql=true
# 自动建表方式
spring.jpa.hibernate.ddl-auto=create-drop

spring.jpa.open-in-view=false
# MySQL5InnoDBDialect(deprecated)
# 不加这句则默认为myisam引擎
# 涉及参数 hibernate.dialect.storage_engine   innodb
spring.jpa.database-platform=org.hibernate.dialect.MySQL57Dialect



```

