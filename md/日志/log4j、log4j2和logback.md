# log4j
```log4j.properties
private static Logger logger = Logger.getLogger(Test.class);  

### 设置###
log4j.rootLogger = debug,stdout,D,E

### 输出信息到控制台 ###
log4j.appender.stdout = org.apache.log4j.ConsoleAppender
log4j.appender.stdout.Target = System.out
log4j.appender.stdout.layout = org.apache.log4j.PatternLayout
log4j.appender.stdout.layout.ConversionPattern = [%-5p] %d{yyyy-MM-dd HH:mm:ss,SSS} method:%l%n%m%n

### 输出DEBUG 级别以上的日志到=E://logs/error.log ###
log4j.appender.D = org.apache.log4j.DailyRollingFileAppender
log4j.appender.D.File = E://logs/log.log
log4j.appender.D.Append = true
log4j.appender.D.Threshold = DEBUG 
log4j.appender.D.layout = org.apache.log4j.PatternLayout
log4j.appender.D.layout.ConversionPattern = %-d{yyyy-MM-dd HH:mm:ss}  [ %t:%r ] - [ %p ]  %m%n

### 输出ERROR 级别以上的日志到=E://logs/error.log ###
log4j.appender.E = org.apache.log4j.DailyRollingFileAppender
log4j.appender.E.File =E://logs/error.log 
log4j.appender.E.Append = true
log4j.appender.E.Threshold = ERROR 
log4j.appender.E.layout = org.apache.log4j.PatternLayout
log4j.appender.E.layout.ConversionPattern = %-d{yyyy-MM-dd HH:mm:ss}  [ %t:%r ] - [ %p ]  %m%n
```

```java
//配置根Logger
log4j.rootLogger = [ level ] , appenderName, appenderName, …

//配置日志信息输出目的地Appender
log4j.appender.appenderName = fully.qualified.name.of.appender.class  
log4j.appender.appenderName.option1 = value1  
…  
log4j.appender.appenderName.option = valueN


org.apache.log4j.ConsoleAppender（控制台），  
org.apache.log4j.FileAppender（文件），  
org.apache.log4j.DailyRollingFileAppender（每天产生一个日志文件），  
org.apache.log4j.RollingFileAppender（文件大小到达指定尺寸的时候产生一个新的文件），  
org.apache.log4j.WriterAppender（将日志信息以流格式发送到任意指定的地方）

//配置日志信息的格式（布局）
log4j.appender.appenderName.layout = fully.qualified.name.of.layout.class  
log4j.appender.appenderName.layout.option1 = value1  
…  
log4j.appender.appenderName.layout.option = valueN


org.apache.log4j.HTMLLayout（以HTML表格形式布局），  
org.apache.log4j.PatternLayout（可以灵活地指定布局模式），  
org.apache.log4j.SimpleLayout（包含日志信息的级别和信息字符串），  
org.apache.log4j.TTCCLayout（包含日志产生的时间、线程、类别等等信息）

```

## 日志级别

每个Logger都被了一个日志级别（log level），用来控制日志信息的输出。日志级别从高到低分为：  
A：off 最高等级，用于关闭所有日志记录。  
B：fatal 指出每个严重的错误事件将会导致应用程序的退出。  
C：error 指出虽然发生错误事件，但仍然不影响系统的继续运行。  
D：warm 表明会出现潜在的错误情形。  
E：info 一般和在粗粒度级别上，强调应用程序的运行全程。  
F：debug 一般用于细粒度级别上，对调试应用程序非常有帮助。  
G：all 最低等级，用于打开所有日志记录。

# 日志接口(slf4j)
slf4j是对所有日志框架制定的一种规范、标准、接口，并不是一个框架的具体的实现，因为接口并不能独立使用，需要和具体的日志框架实现配合使用（如log4j、logback）   
**使用日志接口便于更换为其他日志框架**。
log4j、logback、log4j2都是一种日志具体实现框架，所以既可以单独使用也可以结合slf4j一起搭配使用
# log4j2
log4j2.xml、log4j.json、log4j.js
```xml
Web工程方式：

<context-param>  
    <param-name>log4jConfiguration</param-name>  
    <param-value>/WEB-INF/conf/log4j2.xml</param-value>  
</context-param>  

<listener>   
    <listener-class>org.apache.logging.log4j.web.Log4jServletContextListener</listener-class>  
</listener> 
```
```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="WARN">  
    <Appenders>  
        <Console name="Console" target="SYSTEM_OUT">  
            <PatternLayout pattern="%d{HH:mm:ss.SSS} [%t] %-5level %logger{36} - %msg%n" />  
        </Console>  
    </Appenders>  

    <Loggers>  
        <Root level="info">  
            <AppenderRef ref="Console" />  
        </Root>  
    </Loggers>  
</Configuration>


<?xml version="1.0" encoding="UTF-8"?>
<!-- Configuration后面的status，这个用于设置log4j2自身内部的信息输出，可以不设置，当设置成trace时，
你会看到log4j2内部各种详细输出。可以设置成OFF(关闭)或Error(只输出错误信息) -->
<!--日志级别以及优先级排序: OFF > FATAL > ERROR > WARN > INFO > DEBUG > TRACE > ALL -->
<Configuration status="OFF">

    <Appenders>
        <!-- 输出控制台日志的配置 -->
        <Console name="console" target="SYSTEM_OUT">
            <!--控制台只输出level及以上级别的信息（onMatch），其他的直接拒绝（onMismatch） -->
            <ThresholdFilter level="DEBUG" onMatch="ACCEPT" onMismatch="DENY"/>
            <!-- 输出日志的格式 -->
            <PatternLayout pattern="%d{yyyy-MM-dd HH:mm:ss.SSS} [%t] %-5level %clr{${sys:PID}} [%l] - %msg%n"/>
        </Console>
        
        <Kafka name="Kafka" topic="serverlogs">
            <PatternLayout pattern="%d{yyyy-MM-dd HH:mm:ss.SSS}[%t] %-5level [%l] - %msg"/>
            <Property name="bootstrap.servers">118.31.11.163:9092</Property>
        </Kafka>
    </Appenders>

    <Loggers>
        <Root level="ALL">
            <AppenderRef ref="Kafka"/>
            <AppenderRef ref="console"/>
        </Root>
        <Logger name="org.apache.kafka" level="INFO"/>
        <logger name="org.springframework" level="INFO"/>
    </Loggers>
</Configuration>
```

RollingRandomAccessFile 会根据命名规则当文件满足一定大小时就会另起一个新的文件

## log4j2配置文件详解

log4j2.xml文件的配置大致如下：

Configuration 
properties
Appenders 
Console 
PatternLayout
File
RollingRandomAccessFile
Async
Loggers 
Logger
Root 
AppenderRef
Configuration：为根节点，有status和monitorInterval等多个属性

status的值有 “trace”, “debug”, “info”, “warn”, “error” and “fatal”，用于控制log4j2日志框架本身的日志级别，如果将stratus设置为较低的级别就会看到很多关于log4j2本身的日志，如加载log4j2配置文件的路径等信息
monitorInterval，含义是每隔多少秒重新读取配置文件，可以不重启应用的情况下修改配置
Appenders：输出源，用于定义日志输出的地方 
log4j2支持的输出源有很多，有控制台Console、文件File、RollingRandomAccessFile、MongoDB、Flume 等

Console：控制台输出源是将日志打印到控制台上，开发的时候一般都会配置，以便调试

File：文件输出源，用于将日志写入到指定的文件，需要配置输入到哪个位置（例如：D:/logs/mylog.log）

RollingRandomAccessFile: 该输出源也是写入到文件，不同的是比File更加强大，可以指定当文件达到一定大小（如20MB）时，另起一个文件继续写入日志，另起一个文件就涉及到新文件的名字命名规则，因此需要配置文件命名规则 
这种方式更加实用，因为你不可能一直往一个文件中写，如果一直写，文件过大，打开就会卡死，也不便于查找日志。

fileName 指定当前日志文件的位置和文件名称
filePattern 指定当发生Rolling时，文件的转移和重命名规则
SizeBasedTriggeringPolicy 指定当文件体积大于size指定的值时，触发Rolling
DefaultRolloverStrategy 指定最多保存的文件个数
TimeBasedTriggeringPolicy 这个配置需要和filePattern结合使用，注意filePattern中配置的文件重命名规则是${FILE_NAME}-%d{yyyy-MM-dd HH-mm}-%i，最小的时间粒度是mm，即分钟
TimeBasedTriggeringPolicy指定的size是1，结合起来就是每1分钟生成一个新文件。如果改成%d{yyyy-MM-dd HH}，最小粒度为小时，则每一个小时生成一个文件
NoSql：MongoDb, 输出到MongDb数据库中

Flume：输出到Apache Flume（Flume是Cloudera提供的一个高可用的，高可靠的，分布式的海量日志采集、聚合和传输的系统，Flume支持在日志系统中定制各类数据发送方，用于收集数据；同时，Flume提供对数据进行简单处理，并写到各种数据接受方（可定制）的能力。）

Async：异步，需要通过AppenderRef来指定要对哪种输出源进行异步（一般用于配置RollingRandomAccessFile）

PatternLayout：控制台或文件输出源（Console、File、RollingRandomAccessFile）都必须包含一个PatternLayout节点，用于指定输出文件的格式（如 日志输出的时间 文件 方法 行数 等格式），例如 pattern=”%d{HH:mm:ss.SSS} [%t] %-5level %logger{36} - %msg%n”

%d{HH:mm:ss.SSS} 表示输出到毫秒的时间
%t 输出当前线程名称
%-5level 输出日志级别，-5表示左对齐并且固定输出5个字符，如果不足在右边补0
%logger 输出logger名称，因为Root Logger没有名称，所以没有输出
%msg 日志文本
%n 换行

其他常用的占位符有：
%F 输出所在的类文件名，如Log4j2Test.java
%L 输出行号
%M 输出所在方法名
%l 输出语句所在的行数, 包括类名、方法名、文件名、行数

Loggers：日志器 
日志器分根日志器Root和自定义日志器，当根据日志名字获取不到指定的日志器时就使用Root作为默认的日志器，自定义时需要指定每个Logger的名称name（对于命名可以以包名作为日志的名字，不同的包配置不同的级别等），日志级别level，相加性additivity（是否继承下面配置的日志器）， 对于一般的日志器（如Console、File、RollingRandomAccessFile）一般需要配置一个或多个输出源AppenderRef；

每个logger可以指定一个level（TRACE, DEBUG, INFO, WARN, ERROR, ALL or OFF），不指定时level默认为ERROR

additivity指定是否同时输出log到父类的appender，缺省为true。

<Logger name="rollingRandomAccessFileLogger" level="trace" additivity="true">  
​    <AppenderRef ref="RollingRandomAccessFile" />  
</Logger>

properties: 属性 
使用来定义常量，以便在其他配置的时候引用，该配置是可选的，例如定义日志的存放位置 
···

# logback
https://blog.csdn.net/rulon147/article/details/52620541

Logback是由log4j创始人设计的又一个开源日志组件。logback当前分成三个模块：logback-core,logback- classic和logback-access。logback-core是其它两个模块的基础模块。logback-classic是log4j的一个 改良版本。此外logback-classic完整实现SLF4J API使你可以很方便地更换成其它日志系统如log4j或JDK14 Logging。logback-access访问模块与Servlet容器集成提供通过Http来访问日志的功能

## 根节点<configuration>
## 设置上下文名称：<contextName>

每个logger都关联到logger上下文，默认上下文名称为default。但可以使用<contextName>设置成其他名字，用于区分不同应用程序的记录。一旦设置，不能修改。

设置变量： <property>  
**使用“${}”来变量。**  
设置loger：
<loger>
用来设置某一个包或者具体的某一个类的日志打印级别、以及指定<appender>。<loger>仅有一个name属性，一个可选的level和一个可选的addtivity属性。
name: 用来指定受此loger约束的某一个包或者具体的某一个类。
level: 用来设置打印级别，大小写无关：TRACE, DEBUG, INFO,WARN,ERROR,ALL和OFF，还有一个特俗值INHERITED或者同义词NULL`，代表强制执行上级的级别。 如果未设置此属性，那么当前loger将会继承上级的级别。
addtivity: 是否向上级loger传递打印信息。默认是true。
<root>
也是<loger>元素，但是它是根loger。只有一个level属性，应为已经被命名为"root".
level: 用来设置打印级别，大小写无关：TRACE, DEBUG, INFO, WARN, ERROR, ALL 和 OFF，不能设置为INHERITED或者同义词NULL。 默认是DEBUG。
<loger>和<root>可以包含零个或多个<appender-ref>元素，标识这个appender将会添加到这个loger。  

第1种：只配置root  
第2种：带有loger的配置，不指定级别，不指定appender：  
第3种：带有多个loger的配置，指定级别，指定appender:


## 一般配置

```
<?xml version="1.0" encoding="UTF-8" ?>
<!-- 
    scan:当此属性设置为true时，配置文件如果发生改变，将会被重新加载，默认值为true
    scanPeriod:设置监测配置文件是否有修改的时间间隔，如果没有给出时间单位默认单位是毫秒，当scan为true时此属性生效，默认时间间隔为1分钟
    debug:当此属性设置为true时，将打印出logback内部日志信息，实时查看logback运行状态，默认值为false
 -->
<configuration scan="true" scanPeriod="3 seconds" debug="false">
    <!-- 这一句的意思是打印所有进入的信息 -->
    <statusListener class="ch.qos.logback.core.status.OnConsoleStatusListener" />

    <!-- 
        appender是<configuration>的子节点，是负责写日志的组件
        两个必要属性name和class:name指定appender名称，class指定appender的全限定名
        定义控制台appender 作用:把日志输出到控制台 class="ch.qos.logback.core.ConsoleAppender" 
    -->
    <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
        <!-- 对日志进行格式化 -->
        <layout class="ch.qos.logback.classic.PatternLayout">
            <pattern>
                {"timestamp":"%d{yyyy-MM-dd'T'HH:mm:ss.SSS}","thread":"%t","line":"%line","log_level":"%p","class_name":"%C;","msg":"%m"}\n
            </pattern>
        </layout>
    </appender>

    <!--
        定义滚动记录文件appender 作用:滚动记录文件,先将日志记录到指定文件,当符合某个条件时,将日志记录到其他文件 
        RollingFileAppender class="ch.qos.logback.core.rolling.RollingFileAppender"
        参数： 
            <append>:如果是true日志被追加到文件结尾，如果是false清空现存文件,默认是true
            <file>:被写入的文件名，可以是相对目录也可以是绝对目录，如果上级目录不存在会自动创建，没有默认值
            <rollingPolicy>:当发生滚动时，决定RollingFileAppender的行为，涉及文件移动和重命名
            <triggeringPolicy>:告知RollingFileAppender合适激活滚动
            <prudent>:当为true时不支持FixedWindowRollingPolicy支持TimeBasedRollingPolicy，但是有两个限制:1不支持也不允许文件压缩,2不能设置file属性必须留空
     -->
    <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <!-- 如果是true日志被追加到文件结尾，如果是false清空现存文件，默认是true -->
        <append>true</append>
        <File>logs/logback-test.log</File>
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <!-- 每天滚动一次的日志 只保留30天内的日志文件 -->
            <FileNamePattern>
                logs/logback-test.%d{yyyy-MM-dd}.log.zip
                <maxHistory>30</maxHistory> 
            </FileNamePattern>

            <!-- 每分钟滚动一次日志 -->
            <!-- <FileNamePattern>
                logs/logback-test.%d{yyyy-MM-dd_HH-mm}.log.zip
            </FileNamePattern> -->
        </rollingPolicy>
        <!-- 对日志进行格式化 -->
        <layout class="ch.qos.logback.classic.PatternLayout">
            <Pattern>
                {"timestamp":"%d{yyyy-MM-dd'T'HH:mm:ss.SSS}","thread":"%t","line":"%line","log_level":"%p","class_name":"%C;","msg":"%m"}\n
            </Pattern>
        </layout>
    </appender>

    <!--
        定义滚动记录文件appender 作用:滚动记录文件,先将日志记录到指定文件,当符合某个条件时,将日志记录到其他文件 
        RollingFileAppender class="ch.qos.logback.core.rolling.RollingFileAppender"
        参数： 
            <append>:如果是true日志被追加到文件结尾，如果是false清空现存文件,默认是true
            <file>:被写入的文件名，可以是相对目录也可以是绝对目录，如果上级目录不存在会自动创建，没有默认值
            <rollingPolicy>:当发生滚动时，决定RollingFileAppender的行为，涉及文件移动和重命名
            <triggeringPolicy>:告知RollingFileAppender合适激活滚动
            <prudent>:当为true时不支持FixedWindowRollingPolicy支持TimeBasedRollingPolicy，但是有两个限制:1不支持也不允许文件压缩,2不能设置file属性必须留空
     -->
    <appender name="LOGBACK_ERROR_LOG" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <append>true</append>
        <file>logs/logback-test-error.log</file>

        <!--
            定义滚动策略 作用:根据固定窗口算法重命名文件的滚动策略 class="ch.qos.logback.core.rolling.FixedWindowRollingPolicy
            <fileNamePattern>:表示当触发了回滚策略后，按这个文件命名规则生成归档文件，命名规则中的%i表示在maxIndex和minIndex之间的一个整数值
            <minIndex>:最小索引值
            <maxIndex>:最大索引值
         -->
        <rollingPolicy class="ch.qos.logback.core.rolling.FixedWindowRollingPolicy">
            <fileNamePattern>logs/logback-test-error.log.%i</fileNamePattern>
            <minIndex>1</minIndex>
            <maxIndex>20</maxIndex>
        </rollingPolicy>

        <!-- 
            定义按文件大小触发滚动策略triggeringPolicy 作用:查看当前活动文件的大小，如果超过指定大小会告知RollingFileAppender触发当前活动文件滚动
            只有一个参数 maxSize 这是活动文件的大小，默认值是10MB
        -->
        <triggeringPolicy class="ch.qos.logback.core.rolling.SizeBasedTriggeringPolicy">
            <maxFileSize>50MB</maxFileSize>
        </triggeringPolicy>

        <!-- 
            配置日志级别过滤器 作用:根据日志级别进行过滤，如果日志级别等于配置级别过滤器会根据onMath和onMismatch接收或拒绝日志
            参数:
                <level>:设置过滤级别
                <onMatch>:用于配置符合过滤条件的操作
                <onMismatch>:用于配置不符合过滤条件的操作
                此处配置为只接收ERROR日志级别信息
        -->
        <filter class="ch.qos.logback.classic.filter.LevelFilter">
            <level>ERROR</level>
            <onMatch>ACCEPT</onMatch>
            <onMismatch>DENY</onMismatch>
        </filter>

        <encoder>
            <pattern>
                {"timestamp":"%d{yyyy-MM-dd'T'HH:mm:ss.SSS}","thread":"%t","line":"%line","log_level":"%p","class_name":"%C;","msg":"%m%n", "caller":"%caller{1}"}
            </pattern>
        </encoder>
    </appender>

    <!--
        定义滚动记录文件appender 作用:滚动记录文件,先将日志记录到指定文件,当符合某个条件时,将日志记录到其他文件 
        RollingFileAppender class="ch.qos.logback.core.rolling.RollingFileAppender"
        参数： 
            <append>:如果是true日志被追加到文件结尾，如果是false清空现存文件,默认是true
            <file>:被写入的文件名，可以是相对目录也可以是绝对目录，如果上级目录不存在会自动创建，没有默认值
            <rollingPolicy>:当发生滚动时，决定RollingFileAppender的行为，涉及文件移动和重命名
            <triggeringPolicy>:告知RollingFileAppender合适激活滚动
            <prudent>:当为true时不支持FixedWindowRollingPolicy支持TimeBasedRollingPolicy，但是有两个限制:1不支持也不允许文件压缩,2不能设置file属性必须留空
    -->
    <appender name="LOGBACK_ALL_LOG" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <append>true</append>
        <file>logs/logback-test-all.log</file>

        <!--
            定义滚动策略 作用:根据固定窗口算法重命名文件的滚动策略 class="ch.qos.logback.core.rolling.FixedWindowRollingPolicy
            <fileNamePattern>:表示当触发了回滚策略后，按这个文件命名规则生成归档文件，命名规则中的%i表示在maxIndex和minIndex之间的一个整数值
            <minIndex>:最小索引值
            <maxIndex>:最大索引值
         -->
        <rollingPolicy class="ch.qos.logback.core.rolling.FixedWindowRollingPolicy">
            <fileNamePattern>logs/logback-test-all.log.%i</fileNamePattern>
            <minIndex>1</minIndex>
            <maxIndex>20</maxIndex>
        </rollingPolicy>

        <!-- 
            定义按文件大小触发滚动策略triggeringPolicy 作用:查看当前活动文件的大小，如果超过指定大小会告知RollingFileAppender触发当前活动文件滚动
            只有一个参数 maxSize 这是活动文件的大小，默认值是10MB
        -->
        <triggeringPolicy class="ch.qos.logback.core.rolling.SizeBasedTriggeringPolicy">
            <maxFileSize>50MB</maxFileSize>
        </triggeringPolicy>

        <!-- 
            配置临界值过滤器 作用:过滤掉低于指定临界值的日志，当日志级别等于或高于临界值时过滤器返回NEUTRAL，当日志级别低于临界值时，日志会被拒绝
            此处配置为INFO 即过滤掉日志级别小于INFO的日志信息
        -->
        <filter class="ch.qos.logback.classic.filter.ThresholdFilter">
            <level>INFO</level>
        </filter>

        <encoder>
            <pattern>
                {"timestamp":"%d{yyyy-MM-dd'T'HH:mm:ss.SSS}","thread":"%t","line":"%line","log_level":"%p","class_name":"%C;","msg":"%m%n", "caller":"%caller{1}"}
            </pattern>
        </encoder>
    </appender>

    <!--                    
        logger用来设置某一个包的日志打印级别,以及指定<appender>
        <loger> 仅有一个name属性,一个可选的level和一个可选的addtivity属性
                name:用来指定受此loger约束的某一个包或者具体的某一个类
                level:用来设置打印级别,大小写无关:TRACE, DEBUG, INFO, WARN, ERROR, ALL 和 OFF
                addtivity:是否向上级loger传递打印信息。默认是true,会将信息输入到root配置指定的地方,可以包含多个appender-ref，标识这个appender会添加到这个logger
    -->
    <logger name="com.roberto" level="DEBUG" additivity="false">
        <appender-ref ref="STDOUT" />
        <appender-ref ref="LOGBACK_ALL_LOG" />
        <appender-ref ref="LOGBACK_ERROR_LOG" />
    </logger>

    <!-- 将root的打印级别设置为"INFO",指定了名字为"FILE","STDOUT"的appender -->
    <root>
        <level value="INFO" />
        <appender-ref red="STDOUT" />
        <appender-ref ref="FILE" />
    </root>
</configuration>
```





```xml
<?xml version="1.0" encoding="UTF-8"?>
<!--日志级别以及优先级排序: OFF > FATAL > ERROR > WARN > INFO > DEBUG > TRACE > ALL -->
<!--Configuration后面的status,这个用于设置log4j2自身内部的信息输出,可以不设置,当设置成trace时,你会看到log4j2内部各种详细输出-->
<!--monitorInterval：Log4j能够自动检测修改配置 文件和重新配置本身,设置间隔秒数-->
<configuration status="WARN" monitorInterval="1800">
    <Properties>
        <!-- 日志默认存放的位置,这里设置为项目根路径下,也可指定绝对路径 -->
        <!-- ${web:rootDir}是web项目根路径,java项目没有这个变量,需要删掉,否则会报异常 -->
        <!--<property name="basePath">D://log4j2Logs</property>-->
        <property name="basePath">logs</property>
​
        <!-- 控制台默认输出格式,"%-5level":日志级别,"%l":输出完整的错误位置,是小写的L,因为有行号显示,所以影响日志输出的性能 -->
        <property name="console_log_pattern">%d{yyyy-MM-dd HH:mm:ss.SSS} [%-5level] %l - %m%n</property>
        <!-- 日志文件默认输出格式,不带行号输出(行号显示会影响日志输出性能);%C:大写,类名;%M:方法名;%m:错误信息;%n:换行 -->
        <property name="log_pattern">%d{yyyy-MM-dd HH:mm:ss.SSS} [%-5level] %C.%M - %m%n</property>
​
        <!-- 日志默认切割的最小单位 -->
        <property name="every_file_size">20MB</property>
        <!-- 日志默认输出级别 -->
        <property name="output_log_level">DEBUG</property>
​
        <!-- 日志默认存放路径(所有级别日志) -->
        <property name="rolling_fileName">${basePath}/all.log</property>
        <!-- 日志默认压缩路径,将超过指定文件大小的日志,自动存入按"年月"建立的文件夹下面并进行压缩,作为存档 -->
        <property name="rolling_filePattern">${basePath}/%d{yyyy-MM}/all-%d{yyyy-MM-dd}-%i.log.gz</property>
        <!-- 日志默认同类型日志,同一文件夹下可以存放的数量,不设置此属性则默认为7个 -->
        <property name="rolling_max">50</property>
​
        <!-- Info日志默认存放路径(Info级别日志) -->
        <property name="info_fileName">${basePath}/info.log</property>
        <!-- Info日志默认压缩路径,将超过指定文件大小的日志,自动存入按"年月"建立的文件夹下面并进行压缩,作为存档 -->
        <property name="info_filePattern">${basePath}/%d{yyyy-MM}/info-%d{yyyy-MM-dd}-%i.log.gz</property>
        <!-- Info日志默认同一文件夹下可以存放的数量,不设置此属性则默认为7个 -->
        <property name="info_max">10</property>
​
        <!-- Warn日志默认存放路径(Warn级别日志) -->
        <property name="warn_fileName">${basePath}/warn.log</property>
        <!-- Warn日志默认压缩路径,将超过指定文件大小的日志,自动存入按"年月"建立的文件夹下面并进行压缩,作为存档 -->
        <property name="warn_filePattern">${basePath}/%d{yyyy-MM}/warn-%d{yyyy-MM-dd}-%i.log.gz</property>
        <!-- Warn日志默认同一文件夹下可以存放的数量,不设置此属性则默认为7个 -->
        <property name="warn_max">10</property>
​
        <!-- Error日志默认存放路径(Error级别日志) -->
        <property name="error_fileName">${basePath}/error.log</property>
        <!-- Error日志默认压缩路径,将超过指定文件大小的日志,自动存入按"年月"建立的文件夹下面并进行压缩,作为存档 -->
        <property name="error_filePattern">${basePath}/%d{yyyy-MM}/error-%d{yyyy-MM-dd}-%i.log.gz</property>
        <!-- Error日志默认同一文件夹下可以存放的数量,不设置此属性则默认为7个 -->
        <property name="error_max">10</property>
​
        <!-- 控制台显示的日志最低级别 -->
        <property name="console_print_level">DEBUG</property>
​
    </Properties>
​
    <!--定义appender -->
    <appenders>
        <!-- 用来定义输出到控制台的配置 -->
        <Console name="Console" target="SYSTEM_OUT">
            <!-- 设置控制台只输出level及以上级别的信息(onMatch),其他的直接拒绝(onMismatch)-->
            <ThresholdFilter level="${console_print_level}" onMatch="ACCEPT" onMismatch="DENY"/>
            <!-- 设置输出格式,不设置默认为:%m%n -->
            <PatternLayout pattern="${console_log_pattern}"/>
        </Console>
​
        <!-- 打印root中指定的level级别以上的日志到文件 -->
        <RollingFile name="RollingFile" fileName="${rolling_fileName}" filePattern="${rolling_filePattern}">
            <PatternLayout pattern="${log_pattern}"/>
            <SizeBasedTriggeringPolicy size="${every_file_size}"/>
            <!-- 设置同类型日志,同一文件夹下可以存放的数量,如果不设置此属性则默认存放7个文件 -->
            <DefaultRolloverStrategy max="${rolling_max}" />
            <!-- 匹配INFO以及以上级别 -->
            <Filters>
                <ThresholdFilter level="INFO" onMatch="ACCEPT" onMismatch="DENY"/>
            </Filters>
        </RollingFile>
​
        <!-- 打印INFO级别的日志到文件 -->
        <RollingFile name="InfoFile" fileName="${info_fileName}" filePattern="${info_filePattern}">
            <PatternLayout pattern="${log_pattern}"/>
            <SizeBasedTriggeringPolicy size="${every_file_size}"/>
            <DefaultRolloverStrategy max="${info_max}" />
            <!-- 匹配INFO级别 -->
            <Filters>
                <ThresholdFilter level="WARN" onMatch="DENY" onMismatch="NEUTRAL"/>
                <ThresholdFilter level="INFO" onMatch="ACCEPT" onMismatch="DENY"/>
            </Filters>
        </RollingFile>
​
        <!-- 打印WARN级别的日志到文件 -->
        <RollingFile name="WarnFile" fileName="${warn_fileName}" filePattern="${warn_filePattern}">
            <PatternLayout pattern="${log_pattern}"/>
            <SizeBasedTriggeringPolicy size="${every_file_size}"/>
            <DefaultRolloverStrategy max="${warn_max}" />
            <!-- 匹配WARN级别 -->
            <Filters>
                <ThresholdFilter level="ERROR" onMatch="DENY" onMismatch="NEUTRAL"/>
                <ThresholdFilter level="WARN" onMatch="ACCEPT" onMismatch="DENY"/>
            </Filters>
        </RollingFile>
​
        <!-- 打印ERROR级别的日志到文件 -->
        <RollingFile name="ErrorFile" fileName="${error_fileName}" filePattern="${error_filePattern}">
            <PatternLayout pattern="${log_pattern}"/>
            <SizeBasedTriggeringPolicy size="${every_file_size}"/>
            <DefaultRolloverStrategy max="${error_max}" />
            <!-- 匹配ERROR级别 -->
            <Filters>
                <ThresholdFilter level="FATAL" onMatch="DENY" onMismatch="NEUTRAL"/>
                <ThresholdFilter level="ERROR" onMatch="ACCEPT" onMismatch="DENY"/>
            </Filters>
        </RollingFile>
​
        <Kafka name="Kafka" topic="log-test" ignoreExceptions="false">
            <PatternLayout pattern="[%-4level]_|_%d{YYYY-MM-dd HH:mm:ss}_|_%m_|_${sys:ip}"/>
            <Property name="bootstrap.servers">192.168.254.202:9092</Property>
            <Property name="max.block.ms">2000</Property>
        </Kafka>
        <RollingFile name="failoverKafkaLog" 
                     fileName="${basePath}/failoverKafka/request.log"
                     filePattern="${basePath}/failoverKafka/request.%d{yyyy-MM-dd}.log">
            <ThresholdFilter level="INFO" onMatch="ACCEPT" onMismatch="DENY"/>
            <PatternLayout>
                <Pattern>[%-4level]_|_%d{YYYY-MM-dd HH:mm:ss}_|_%m_|_${sys:ip}%n</Pattern>
            </PatternLayout>
            <Policies>
                <TimeBasedTriggeringPolicy />
            </Policies>
        </RollingFile>
        <Failover name="Failover" primary="Kafka" retryIntervalSeconds="600">
            <Failovers>
                <AppenderRef ref="failoverKafkaLog"/>
            </Failovers>
        </Failover>
​
        <!--异步-->
        <Async name="kafkaLogger" level="INFO" additivity="false">
            <appender-ref ref="Failover"/>
        </Async>
        
    </appenders>
​
    <!--然后定义logger,只有定义了logger并引入的appender,appender才会生效-->
    <loggers>
        <!-- 设置对打印sql语句的支持 -->
        <logger name="java.sql" level="debug" additivity="false">
            <appender-ref ref="Console"/>
        </logger>
        <!--建立一个默认的root的logger-->
        <root level="${output_log_level}">
            <appender-ref ref="RollingFile"/>
            <appender-ref ref="Console"/>
            <appender-ref ref="InfoFile"/>
            <appender-ref ref="WarnFile"/>
            <appender-ref ref="ErrorFile"/>
            <AppenderRef ref="Kafka"/>
        </root>
        <Logger name="org.apache.kafka" level="INFO" /> <!-- avoid recursive logging -->
    </loggers>
</configuration>
```

