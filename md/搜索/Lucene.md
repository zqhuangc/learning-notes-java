基于Java的全文索引/检索引擎——Lucene
类比字典理解
## 基本了解
Lucene是apache软件基金会4 jakarta项目组的一个子项目，是一个开放源代码的全文检索引擎工具包，但它不是一个完整的全文检索引擎，而是一个全文检索引擎的架构，提供了完整的查询引擎和索引引擎，部分文本分析引擎。Lucene的目的是为软件开发人员提供一个简单易用的工具包.  
Lucene的API接口设计的比较通用，输入输出结构都很像数据库的表==>记录==>字段

Lucene和solr的区别,solr是一个搜索系统,打个比方,就如servlet和struts2的区别   Lucene就是servlet,solr就好比struts2,solr封装了Lucene.



提取资源中关键信息，建立索引（目录）
搜索时，根据关键字（目录），找到资源的位置

document  表
field（域） 类似字段   列名-value


创建Document文档对象
添加field
创建分析器（分词器）
创建IndexWriter写入对象
把Document写入到索引库中



```java
package com.itheima.lucene;

import java.io.File;
import java.util.ArrayList;
import java.util.List;

import org.apache.lucene.analysis.Analyzer;
import org.apache.lucene.analysis.standard.StandardAnalyzer;
import org.apache.lucene.document.Document;
import org.apache.lucene.document.Field.Store;
import org.apache.lucene.document.TextField;
import org.apache.lucene.index.DirectoryReader;
import org.apache.lucene.index.IndexReader;
import org.apache.lucene.index.IndexWriter;
import org.apache.lucene.index.IndexWriterConfig;
import org.apache.lucene.queryparser.classic.QueryParser;
import org.apache.lucene.search.IndexSearcher;
import org.apache.lucene.search.Query;
import org.apache.lucene.search.ScoreDoc;
import org.apache.lucene.search.TopDocs;
import org.apache.lucene.store.Directory;
import org.apache.lucene.store.FSDirectory;
import org.apache.lucene.util.Version;
import org.junit.Test;
import org.wltea.analyzer.lucene.IKAnalyzer;

import com.itheima.dao.BookDao;
import com.itheima.dao.impl.BookDaoImpl;
import com.itheima.pojo.Book;

public class CreateIndexTest {
    //分词
    @Test
    public void testCreateIndex() throws Exception{
    //    1. 采集数据
        BookDao bookDao = new BookDaoImpl();
        List<Book> listBook = bookDao.queryBookList();
        
    //    2. 创建Document文档对象
        List<Document> documents = new ArrayList<>();
        for (Book bk : listBook) {
            
            Document doc = new Document();
            doc.add(new TextField("id", String.valueOf(bk.getId()), Store.YES));// Store.YES:表示存储到文档域中
            doc.add(new TextField("name", bk.getName(), Store.YES));
            doc.add(new TextField("price", String.valueOf(bk.getPrice()), Store.YES));
            doc.add(new TextField("pic", bk.getPic(), Store.YES));
            doc.add(new TextField("desc", bk.getDesc(), Store.YES));
            
            // 把Document放到list中
            documents.add(doc);
        }
        
    //    3. 创建分析器（分词器）
        //Analyzer analyzer = new StandardAnalyzer();
        //中文  IK
        Analyzer analyzer = new IKAnalyzer();
        
    //    4. 创建IndexWriterConfig配置信息类
        IndexWriterConfig config = new IndexWriterConfig(Version.LUCENE_4_10_3, analyzer);
        
    //    5. 创建Directory对象，声明索引库存储位置
        Directory directory = FSDirectory.open(new File("H:\\temp"));
        
    //    6. 创建IndexWriter写入对象
        IndexWriter writer = new IndexWriter(directory, config);
        
    //    7. 把Document写入到索引库中
        for (Document doc : documents) {
            writer.addDocument(doc);
        }
        
    //    8. 释放资源
        writer.close();
    }


         //查
    @Test
    public void serachIndex() throws Exception{
        //创建分词器   必须和检索时的分析器一致
        Analyzer analyzer = new StandardAnalyzer();
        // 创建搜索解析器，第一个参数：默认Field域，第二个参数：分词器
        QueryParser queryParser = new QueryParser("desc", analyzer);
        
        // 1. 创建Query搜索对象
        Query query = queryParser.parse("desc:java AND lucene");
            
        // 2. 创建Directory流对象,声明索引库位置
        Directory directory = FSDirectory.open(new File("H:\\temp"));
        
        // 3. 创建索引读取对象IndexReader
        IndexReader indexReader = DirectoryReader.open(directory);
        
        // 4. 创建索引搜索对象IndexSearcher
        IndexSearcher indexSearcher = new IndexSearcher(indexReader);
        
        // 5. 使用索引搜索对象，执行搜索，返回结果集TopDocs
        // 第一个参数：搜索对象，第二个参数：返回的数据条数，指定查询结果最顶部的n条数据返回
        TopDocs topDocs  = indexSearcher.search(query, 10);
        System.out.println("查询到的数据总条数是：" + topDocs.totalHits);
        //获得结果集
        ScoreDoc[] docs = topDocs.scoreDocs;
        
        // 6. 解析结果集
        for (ScoreDoc scoreDoc : docs) {
            //获得文档
            int docID = scoreDoc.doc;
            Document doc = indexSearcher.doc(docID);
            
            System.out.println("docID:"+docID);
            System.out.println("bookid:"+doc.get("id"));
            System.out.println("pic:"+doc.get("pic"));
            System.out.println("name:"+doc.get("name"));
            System.out.println("desc:"+doc.get("desc"));
            System.out.println("price:"+doc.get("price"));
        }
        
        // 7. 释放资源
        indexReader.close();
    }
}
```

