solr是一个应用程序solr.war
## Linux配置solr
第一步：安装jdk、安装tomcat  
第二步：解压solr压缩包。  
第三步：把dist/solr-4.10.3.war部署到tomcat下。  
第四步：解压缩war包。启动tomcat解压。   tail –f logs/catalina.out 查看信息  
第五步：需要把/root/solr-4.10.3/example/lib/ext目录下的所有的jar包添加到solr工程中。  
第六步：创建solrhome。把/root/solr-4.10.3/example/solr文件夹复制一份作为solrhome。  
第七步：告诉solr服务solrhome的位置。需要修改web.xml

在web.xml配置
```xml
<env-entry>
<env-entry-name></env-entry-name>
<env-entry-value></env-entry-value>
<env-entry-type></env-entry-type>
</env-entry>
```

## 配置中文分析器、自定义业务域
分析器使用IKAnalyzer。
使用方法：
第一步：把IKAnalyzer依赖的jar包添加到solr工程中。把分析器使用的扩展词典添加到classpath中。
第二步：需要自定义一个FieldType。Schema.xml中定义。可以在FieldType中指定中文分析器。
```xml
<fieldType name="text_ik" class="solr.TextField">
<analyzer class="org.wltea.analyzer.lucene.IKAnalyzer"/>
</fieldType>
```

第三步：自定义域。指定域的类型为自定义的FieldType。
```sql
Sql语句：
SELECT
	a.id,
	a.title,
	a.sell_point,
	a.price,
	a.image,
	b.`name` category_name,
	c.item_desc
FROM
	tb_item a
LEFT JOIN tb_item_cat b ON a.cid = b.id
LEFT JOIN tb_item_desc c ON a.id = c.item_id
WHERE
	a.`status` = 1
```
```xml
<field name="item_title" type="text_ik" indexed="true" stored="true"/>
<field name="item_sell_point" type="text_ik" indexed="true" stored="true"/>
<field name="item_price"  type="long" indexed="true" stored="true"/>
<field name="item_image" type="string" indexed="false" stored="true" />
<field name="item_category_name" type="string" indexed="true" stored="true" />
<field name="item_desc" type="text_ik" indexed="true" stored="false" />

<field name="item_keywords" type="text_ik" indexed="true" stored="false" multiValued="true"/>
<copyField source="item_title" dest="item_keywords"/>
<copyField source="item_sell_point" dest="item_keywords"/>
<copyField source="item_category_name" dest="item_keywords"/>
<copyField source="item_desc" dest="item_keywords"/>
```
第四步：重新启动tomcat


pom中依赖
```xml
<!-- solr客户端 -->
<dependency>
    <groupId>org.apache.solr</groupId>
    <artifactId>solr-solrj</artifactId>
</dependency>

```

```java
public class SolrJTest {
	@Test
	publicvoid testSolrJ() throws Exception {
		//创建连接
		SolrServer solrServer = new HttpSolrServer("http://192.168.25.154:8080/solr");
		//创建一个文档对象
		SolrInputDocument document = new SolrInputDocument();
		//添加域
		document.addField("id", "solrtest01");
		document.addField("item_title", "测试商品");
		document.addField("item_sell_point", "卖点");
		//添加到索引库
		solrServer.add(document);
		//提交
		solrServer.commit();
	}
	
	@Test
	publicvoid testQuery() throws Exception {
		//创建连接
		SolrServer solrServer = new HttpSolrServer("http://192.168.25.154:8080/solr");
		//创建一个查询对象
		SolrQuery query = new SolrQuery();
		query.setQuery("*:*");
		//执行查询
		QueryResponse response = solrServer.query(query);
		//取查询结果
		SolrDocumentList solrDocumentList = response.getResults();
		for (SolrDocument solrDocument : solrDocumentList) {
			System.out.println(solrDocument.get("id"));
			System.out.println(solrDocument.get("item_title"));
			System.out.println(solrDocument.get("item_sell_point"));
		}
	}
}


```

## 适用场景：
* 对于存储在文件中的日志 可以导入到solr中做分析，
* 对于关系型数据库里需要做全文搜索的字段 可以导入到slor 中。