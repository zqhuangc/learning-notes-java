从一开始，MyBatis就是一个XML驱动的框架。配置基于XML，映射语句以XML定义。使用MyBatis 3，可以使用新选项。MyBatis 3 构建于全面且功能强大的基于Java的Configuration API之上。此Configuration API是基于XML的MyBatis配置的基础，以及基于Annotation的新配置。注释提供了一种实现简单映射语句的简单方法，而不会引入大量开销。

注解	目标	XML等价物	描述
@Param	参数	N / A	   如果mapper方法采用多个参数，则可以将此注释应用于mapper方法参数，以便为每个参数指定名称。否则，多个参数将以其前缀为“param”的位置命名（不包括任何RowBounds参数）。例如＃{param1}，＃{param2}等是默认值。使用@Param（“person”），参数将命名为＃{person}。

@One  
@Many

@Property	        N/A	    <property>
@CacheNamespaceRef	Class	<cacheRef>
@Results	Method	<resultMap>









