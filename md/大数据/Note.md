Job = Map + Reduce(可有可无)
Hadoop自己实现了一套数据类型

键值对

* 大数据设计
技术能力
robots.txt  
dataV、Flink、storm、spark、MaxCompute、Quick BI
1、awesome + xx
2、功能 + 网站
3、接口

jdk 持久版本
![kuayugongji](../image/CSRF.jpg)

Mapper<keyin,valuein,keyout,valueout>

Mapper<LongWritable,Text,Text,NullWritable>


LongWritable key,key1单词偏移量
Text value, 输入的数据
Context context, 上下文
//对于Map ，上文  HDFS 下文 Reduce
高德- canvas
svg  webGl



不符规则，脏数据

包装类
比较数字更好 string  
equals用处


空格 \t

底层java

标识文件--字节校验
crc32  adler32

监听文件变化
inotify  轮询3-4s

合并日志
  捕获字段 游走在上下文

实现高吞吐
批量化读取处理
预加载 buffer
批量化发送
    提高压缩比
    降低请求头的大小
异步化处理
    文件监听  日志处理   日志发送  解耦
如何实现资源控制
    句柄
       惰性持有  不自动释放  定时器  0释放句柄
    内存控制  OOM

    io密集  计算密集

流量控制--- cpu控制   正比
固定窗口 滑动窗口  令牌桶  漏桶  限流


非阻塞流  NIO
Guava  Retemiter
蚁群算法


paddlepaddle




HDFS
元信息：

缓存方式：内存 磁盘 日志  

上传流程：


水平复制是在datanode之间进行的

数据展示（可视化）
Tableau：PC/服务器
Infogram：实时数据与可视化信息图表
Plotly：人性化
chartBLOCKs：