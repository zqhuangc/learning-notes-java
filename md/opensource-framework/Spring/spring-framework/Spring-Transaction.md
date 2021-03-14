约定由于配置

> TransactionDefinition  
> ||
> PlatformTransactionManager => hibernate,jpa,jta,jdbc(datasource)
> ||
> TransactionStatus

TxNamespaceHandler
init()
parse()
parseInternal()

TransactionInterceptor  
invoke
doGetTransaction    ThreadLocal


doBegin  开启事务

数据清空千万别用delete from无where条件会锁表


数据不允许修改，final

注解与配置，注解优先





本**类内调用**不触发事务代理



>  PlatformTransactionManager   中的方法
>
> getTransaction    调用了 TransactionSynchronizationManager 类的 getResource()
>
> 从 ThreadLocal 里面取值，Map\<Key:DataSource,Value:ConnectionHolder\>（相当于获取一个连接对象 Connection）；
>
> conn.setAutoCommit(false);
>
> Commit  conn.commit()
> Rollback  conn.rollback()

























