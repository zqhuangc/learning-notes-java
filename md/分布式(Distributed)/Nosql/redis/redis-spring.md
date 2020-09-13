### 客户端连接

RedisTemplate 和 StringRedisTemplate  序列化方式不同

[RedisTemplate  api](https://www.cnblogs.com/EasonJim/p/7803067.html)

RedisConnectionFactory：jedis，lettuce

xxxRedisSerializer

JedisClientConfiguration 多机

RedisStandaloneConfiguration 单机



```
JedisConnectionConfiguration
RedisAutoConfiguration
```



Pipeline 单向， 关闭连接前不获取结果

multi与exec 事务

### 操作

XXX：Value(String)、List、Set、Hash、ZSet

opsForXXX

XXXOperations