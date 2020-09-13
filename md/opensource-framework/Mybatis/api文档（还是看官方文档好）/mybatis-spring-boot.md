@MapperScan(basePackages="com.ostrich.*.repository")这个注解是用户扫描mapper接口的也就是dao类，

mybatis.mapper-locations，而这个是用于扫描mapper.xml的，二者缺少一个都会报错



properties文件中使用${}引用配置项

