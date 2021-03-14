# [JDBC](https://docs.spring.io/spring-framework/docs/current/reference/html/data-access.html#jdbc)

`NamedParameterJdbcTemplate`

`MappingSqlQuery`, `SqlUpdate`, and `StoredProcedure` 

## JDBC core classes



- [Running Statements](https://docs.spring.io/spring-framework/docs/current/reference/html/data-access.html#jdbc-statements-executing)

  运行声明

  ```
  this.jdbcTemplate.execute("create table mytable (id integer, name varchar(100))")
  ```

- [Running Queries](https://docs.spring.io/spring-framework/docs/current/reference/html/data-access.html#jdbc-statements-querying)

  运行查询

- [Updating the Database](https://docs.spring.io/spring-framework/docs/current/reference/html/data-access.html#jdbc-updates)

  更新数据库



### [Using `JdbcTemplate`](https://docs.spring.io/spring-framework/docs/current/reference/html/data-access.html#jdbc-JdbcTemplate)

使用 JdbcTemplate

```
String lastName = this.jdbcTemplate.queryForObject(
        "select last_name from t_actor where id = ?",
        String.class, 1212L);


private final RowMapper<Actor> actorRowMapper = (resultSet, rowNum) -> {
    Actor actor = new Actor();
    actor.setFirstName(resultSet.getString("first_name"));
    actor.setLastName(resultSet.getString("last_name"));
    return actor;
};

public List<Actor> findAllActors() {
    return this.jdbcTemplate.query( "select first_name, last_name from t_actor", actorRowMapper);
}

// 插入
this.jdbcTemplate.update(
        "insert into t_actor (first_name, last_name) values (?, ?)",
        "Leonor", "Watling");
// 更新       
this.jdbcTemplate.update(
        "update t_actor set last_name = ? where id = ?",
        "Banjo", 5276L);
        
// 删除
this.jdbcTemplate.update(
        "delete from t_actor where id = ?",
        Long.valueOf(actorId));

;
```



### [Using `NamedParameterJdbcTemplate`](https://docs.spring.io/spring-framework/docs/current/reference/html/data-access.html#jdbc-NamedParameterJdbcTemplate)

使用 NamedParameterJdbcTemplate

`SqlParameterSource`

`MapSqlParameterSource`

`BeanPropertySqlParameterSource` 

`JdbcOperations` 

```
// some JDBC-backed DAO class...
private NamedParameterJdbcTemplate namedParameterJdbcTemplate;

public void setDataSource(DataSource dataSource) {
    this.namedParameterJdbcTemplate = new NamedParameterJdbcTemplate(dataSource);
}

public int countOfActorsByFirstName(String firstName) {

    String sql = "select count(*) from T_ACTOR where first_name = :first_name";

    SqlParameterSource namedParameters = new MapSqlParameterSource("first_name", firstName);
    // Map<String, String> namedParameters = Collections.singletonMap("first_name", firstName);
    
    return this.namedParameterJdbcTemplate.queryForObject(sql, namedParameters, Integer.class);
}

public int countOfActors(Actor exampleActor) {

    // notice how the named parameters match the properties of the above 'Actor' class
    String sql = "select count(*) from T_ACTOR where first_name = :firstName and last_name = :lastName";

    SqlParameterSource namedParameters = new BeanPropertySqlParameterSource(exampleActor);

    return this.namedParameterJdbcTemplate.queryForObject(sql, namedParameters, Integer.class);

```

### [Using `SQLExceptionTranslator`](https://docs.spring.io/spring-framework/docs/current/reference/html/data-access.html#jdbc-SQLExceptionTranslator)

使用 SQLExceptionTranslator

> SQLExceptionTranslator 是一个由类实现的接口，可以在 SQLExceptions 和 Spring 自己的 org.springframework.dao 之间进行转换。DataAccessException，在数据访问策略方面是不可知的。实现可以是通用的(例如，对 JDBC 使用 SQLState 代码)或专有的(例如，使用 Oracle 错误代码) ，以获得更高的精度。

```
this.jdbcTemplate.setExceptionTranslator(tr);

public class CustomSQLErrorCodesTranslator extends SQLErrorCodeSQLExceptionTranslator {

    protected DataAccessException customTranslate(String task, String sql, SQLException sqlEx) {
        if (sqlEx.getErrorCode() == -12345) {
            return new DeadlockLoserDataAccessException(task, sqlEx);
        }
        return null;
    }
}
```



### [Retrieving Auto-generated Keys](https://docs.spring.io/spring-framework/docs/current/reference/html/data-access.html#jdbc-auto-generated-keys)

检索自动生成的密钥

```
final String INSERT_SQL = "insert into my_test (name) values(?)";
final String name = "Rob";

KeyHolder keyHolder = new GeneratedKeyHolder();
jdbcTemplate.update(connection -> {
    PreparedStatement ps = connection.prepareStatement(INSERT_SQL, new String[] { "id" });
    ps.setString(1, name);
    return ps;
}, keyHolder);

// keyHolder.getKey() now contains the generated key
```



## DataSource

- [Using `DataSource`](https://docs.spring.io/spring-framework/docs/current/reference/html/data-access.html#jdbc-datasource)

  使用数据源

- [Using `DataSourceUtils`](https://docs.spring.io/spring-framework/docs/current/reference/html/data-access.html#jdbc-DataSourceUtils)

  使用 DataSourceUtils

- [Implementing `SmartDataSource`](https://docs.spring.io/spring-framework/docs/current/reference/html/data-access.html#jdbc-SmartDataSource)

  实现 SmartDataSource

- [Extending `AbstractDataSource`](https://docs.spring.io/spring-framework/docs/current/reference/html/data-access.html#jdbc-AbstractDataSource)

  扩展 AbstractDataSource

- [Using `SingleConnectionDataSource`](https://docs.spring.io/spring-framework/docs/current/reference/html/data-access.html#jdbc-SingleConnectionDataSource)

  使用 singlecon/nectiondatasource

- [Using `DriverManagerDataSource`](https://docs.spring.io/spring-framework/docs/current/reference/html/data-access.html#jdbc-DriverManagerDataSource)

  使用 DriverManagerDataSource

- [Using `TransactionAwareDataSourceProxy`](https://docs.spring.io/spring-framework/docs/current/reference/html/data-access.html#jdbc-TransactionAwareDataSourceProxy)

  使用 transactionawareadasourceproxy

- [Using `DataSourceTransactionManager`](https://docs.spring.io/spring-framework/docs/current/reference/html/data-access.html#jdbc-DataSourceTransactionManager)

  使用 DataSourceTransactionManager



```
<bean id="dataSource" class="com.mchange.v2.c3p0.ComboPooledDataSource" destroy-method="close">
    <property name="driverClass" value="${jdbc.driverClassName}"/>
    <property name="jdbcUrl" value="${jdbc.url}"/>
    <property name="user" value="${jdbc.username}"/>
    <property name="password" value="${jdbc.password}"/>
</bean>

<bean id="dataSource" class="org.apache.commons.dbcp.BasicDataSource" destroy-method="close">
    <property name="driverClassName" value="${jdbc.driverClassName}"/>
    <property name="url" value="${jdbc.url}"/>
    <property name="username" value="${jdbc.username}"/>
    <property name="password" value="${jdbc.password}"/>
</bean>

<context:property-placeholder location="jdbc.properties"/>



```



## 批处理

通过实现特殊接口 BatchPreparedStatementSetter 的两个方法，并将该实现作为 batchUpdate 方法调用中的第二个参数传入，可以完成 JdbcTemplate 批处理。可以使用 getBatchSize 方法提供当前批处理的大小。可以使用 setValues 方法设置已准备语句的参数的值。

ParameterMetaData.getParameterType

```
private NamedParameterTemplate namedParameterJdbcTemplate;

public void setDataSource(DataSource dataSource) {
      this.namedParameterJdbcTemplate = new NamedParameterJdbcTemplate(dataSource);
}

public int[] batchUpdate(final List<Actor> actors) {
        return this.jdbcTemplate.batchUpdate(
                "update t_actor set first_name = ?, last_name = ? where id = ?",
                new BatchPreparedStatementSetter() {
                
                    public void setValues(PreparedStatement ps, int i) throws SQLException {
                        Actor actor = actors.get(i);
                        ps.setString(1, actor.getFirstName());
                        ps.setString(2, actor.getLastName());
                        ps.setLong(3, actor.getId().longValue());
                    }
                    
                    public int getBatchSize() {
                        return actors.size();
                    }
                });
    }

// SqlParameterSource  
// 命名参数的批量更新
public int[] batchUpdate(List<Actor> actors) {
        return this.namedParameterJdbcTemplate.batchUpdate(
                "update t_actor set first_name = :firstName, last_name = :lastName where id = :id",
                SqlParameterSourceUtils.createBatch(actors));
    }

    // ... additional methods
}  


 public int[] batchUpdate(final List<Actor> actors) {
        List<Object[]> batch = new ArrayList<Object[]>();
        for (Actor actor : actors) {
            Object[] values = new Object[] {
                    actor.getFirstName(), actor.getLastName(), actor.getId()};
            batch.add(values);
        }
        return this.jdbcTemplate.batchUpdate(
                "update t_actor set first_name = ?, last_name = ? where id = ?",
                batch);
    }
    
    
// 多批处理    
public int[][] batchUpdate(final Collection<Actor> actors) {
        int[][] updateCounts = jdbcTemplate.batchUpdate(
                "update t_actor set first_name = ?, last_name = ? where id = ?",
                actors,
                100,
                (PreparedStatement ps, Actor actor) -> {
                    ps.setString(1, actor.getFirstName());
                    ps.setString(2, actor.getLastName());
                    ps.setLong(3, actor.getId().longValue());
                });
        return updateCounts;
    }
```



## `SimpleJdbcInsert` and `SimpleJdbcCall`

```
public class JdbcActorDao implements ActorDao {

    private SimpleJdbcInsert insertActor;

    public void setDataSource(DataSource dataSource) {
        // 
        this.insertActor = new SimpleJdbcInsert(dataSource).withTableName("t_actor");
        
    }

    public void add(Actor actor) {
        Map<String, Object> parameters = new HashMap<String, Object>(3);
        parameters.put("id", actor.getId());
        parameters.put("first_name", actor.getFirstName());
        parameters.put("last_name", actor.getLastName());
        insertActor.execute(parameters);
    }
    
    public void setDataSource(DataSource dataSource) {
        this.insertActor = new SimpleJdbcInsert(dataSource)
                .withTableName("t_actor")
                .usingColumns("first_name", "last_name")
                .usingGeneratedKeyColumns("id");
    }
    
    public void add(Actor actor) {
        Map<String, Object> parameters = new HashMap<String, Object>(2);
        parameters.put("first_name", actor.getFirstName());
        parameters.put("last_name", actor.getLastName());
        Number newId = insertActor.executeAndReturnKey(parameters);
        actor.setId(newId.longValue());
    }

    public void setDataSource(DataSource dataSource) {
        this.insertActor = new SimpleJdbcInsert(dataSource)
                .withTableName("t_actor")
                .usingGeneratedKeyColumns("id");
    }
    public void add(Actor actor) {
        SqlParameterSource parameters = new BeanPropertySqlParameterSource(actor);
        Number newId = insertActor.executeAndReturnKey(parameters);
        actor.setId(newId.longValue());
    }
    
    public void add(Actor actor) {
        SqlParameterSource parameters = new MapSqlParameterSource()
                .addValue("first_name", actor.getFirstName())
                .addValue("last_name", actor.getLastName());
        Number newId = insertActor.executeAndReturnKey(parameters);
        actor.setId(newId.longValue());
    }
}



```



```
public class JdbcActorDao implements ActorDao {

    private SimpleJdbcCall procReadActor;

    public void setDataSource(DataSource dataSource) {
        this.procReadActor = new SimpleJdbcCall(dataSource)
                .withProcedureName("read_actor");
    }
    
    public void setDataSource(DataSource dataSource) {
        JdbcTemplate jdbcTemplate = new JdbcTemplate(dataSource);
        jdbcTemplate.setResultsMapCaseInsensitive(true);
        this.procReadActor = new SimpleJdbcCall(jdbcTemplate)
                .withProcedureName("read_actor")
                .withoutProcedureColumnMetaDataAccess()
                .useInParameterNames("in_id")
                .declareParameters(
                        new SqlParameter("in_id", Types.NUMERIC),
                        new SqlOutParameter("out_first_name", Types.VARCHAR),
                        new SqlOutParameter("out_last_name", Types.VARCHAR),
                        new SqlOutParameter("out_birth_date", Types.DATE)
                );
    }
    
    // this.funcGetActorName = new SimpleJdbcCall(jdbcTemplate).withFunctionName("get_actor_name");

    public Actor readActor(Long id) {
        SqlParameterSource in = new MapSqlParameterSource()
                .addValue("in_id", id);
        Map out = procReadActor.execute(in);
        Actor actor = new Actor();
        actor.setId(id);
        actor.setFirstName((String) out.get("out_first_name"));
        actor.setLastName((String) out.get("out_last_name"));
        actor.setBirthDate((Date) out.get("out_birth_date"));
        return actor;
    }

    // ... additional methods
}
The code
```



## 手写ORM框架

Template设计模式

只定义RowMapper接口（resultset，rownum），mapping方法    ORM由自己实现

JDBCTemplate

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