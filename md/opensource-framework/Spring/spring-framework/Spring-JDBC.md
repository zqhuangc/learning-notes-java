手写ORM框架
Template设计模式
只定义RowMapper接口，mapping方法    ORM由自己实现
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