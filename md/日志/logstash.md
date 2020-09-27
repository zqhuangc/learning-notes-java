

### [logstash文档](https://doc.yonyoucloud.com/doc/logstash-best-practice-cn/input/file.html)


```
Logstash 通过管道进行运作，管道有两个必需的元素，输入和输出，还有一个可选的元素，过滤器。

输入插件从数据源获取数据，过滤器插件根据用户指定的数据格式修改数据，输出插件则将数据写入到目的地。
```


Logstash是一个接收，处理，转发日志的工具。支持系统日志，webserver日志，错误日志，应用日志，总之包括所有可以抛出来的日志类型。怎么样听起来挺厉害的吧？
在一个典型的使用场景下(ELK)：用Elasticsearch作为后台数据的存储，kibana用来前端的报表展示。Logstash在其过程中担任搬运工的角色，它为数据存储，报表查询和日志解析创建了一个功能强大的管道链。Logstash提供了多种多样的 input,filters,codecs和output组件，让使用者轻松实现强大的功能。好了让我们开始吧
```bash
bin/logstash -e 'input { stdin { } } output { elasticsearch { host => localhost } }'

curl 'http://localhost:9200/_search?pretty'
```

Elasticsearch-kopf插件
多重输出
默认配置 - 按照每日日期建立索引


## logstash一些核心的特性和如何和logstash引擎交互

### 事件的生命周期
Inputs,Outputs,Codecs,Filters构成了Logstash的核心配置项。Logstash通过建立一条事件处理的管道，从你的日志提取出数据保存到Elasticsearch中，为高效的查询数据提供基础。

### Inputs

input 及输入是指日志数据传输到Logstash中。其中常见的配置如下：
* file：从文件系统中读取一个文件，很像UNIX命令 "tail -0a"
* syslog：监听514端口，按照RFC3164标准解析日志数据
* redis：从redis服务器读取数据，支持channel(发布订阅)和list模式。redis一般在Logstash消费集群中作为"broker"角色，保存events队列共Logstash消费。
* lumberjack：使用lumberjack协议来接收数据，目前已经改为 logstash-forwarder。.



*type* 和 *tags* 是 logstash 事件中两个特殊的字段。通常来说我们会在*输入区段*中通过 *type* 来标记事件类型 —— 我们肯定是提前能知道这个事件属于什么类型的。而 *tags* 则是在数据处理过程中，由具体的插件来添加或者删除的。

```
codec => "plain"
codec => "json"
```

```conf
file {
    path => "E:/developer/bat/ELK/logstash-tutorial-dataset.log"
    start_position => "beginning"
}
path: 读取文件路径
discover_interval:logstash 每隔多久去检查一次被监听的 path 下是否有新文件。默认值是 15 秒。
exclude:不想被监听的文件可以排除出去，这里跟 path 一样支持 glob 展开。
sincedb_path:如果你不想用默认的 $HOME/.sincedb
    (Windows 平台上在 C:\Windows\System32\config\systemprofile\.sincedb)，
    可以通过这个配置定义 sincedb 文件到其他位置。
sincedb_write_interval:logstash 每隔多久写一次 sincedb 文件，默认是 15 秒。
stat_interval:logstash 每隔多久检查一次被监听文件状态（是否有更新），默认是 1 秒。
start_position:logstash 从什么位置开始读取文件数据，默认是结束位置，也就是说 logstash 进程会以类似 tail -F 的形式运行。如果你是要导入原有数据，把这个设定改成 "beginning"，logstash 进程就从头开始读取，有点类似 cat，但是读到最后一行不会终止，而是继续变成 tail -F。




tcp {
   port => 8888
   mode => "server"
   ssl_enable => false
}
generator {
    count => 10000000
    message => '{"key1":"value1","key2":[1,2],"key3":{"subkey1":"subvalue1"}}'
   codec => json
}
syslog {
   port => "514"
}
redis {
    data_type => "pattern_channel"
    key => "logstash-*"
    host => "192.168.0.2"
    port => 6379
    threads => 5
}


```





### Filters

Fillters 在Logstash处理链中担任中间处理组件。他们经常被组合起来实现一些特定的行为来，处理匹配特定规则的事件流。常见的filters如下：  
* grok：解析无规则的文字并转化为有结构的格式。Grok 是目前最好的方式来将无结构的数据转换为有结构可查询的数据。有120多种匹配规则，会有一种满足你的需要。
* mutate：mutate filter 允许改变输入的文档，你可以从命名，删除，移动或者修改字段在处理事件的过程中。
* drop：丢弃一部分events不进行处理，例如：debug events。
* clone：拷贝 event，这个过程中也可以添加或移除字段。
* geoip：添加地理信息(为前台kibana图形化展示使用)

```java
实际运用中，我们需要处理各种各样的日志文件，如果你都是在配置文件里各自写一行自己的表达式，就完全不可管理了。所以，我们建议是把所有的 grok 表达式统一写入到一个地方。然后用 filter/grok 的 patterns_dir 选项来指明。

Grok 支持把预定义的 grok 表达式 写入到文件中，官方提供的预定义 grok 表达式见：https://github.com/logstash/logstash/tree/v1.4.2/patterns。

注意：在新版本的logstash里面，pattern目录已经为空，最后一个commit提示core patterns将会由logstash-patterns-core gem来提供，该目录可供用户存放自定义patterns

如果你把 "message" 里所有的信息都 grok 到不同的字段了，数据实质上就相当于是重复存储了两份。所以你可以用 remove_field 参数来删除掉 message 字段，或者用 overwrite 参数来重写默认的 message 字段，只保留最重要的部分。

filter {
    grok {
        patterns_dir => "/path/to/your/own/patterns"
        match => {
            "message" => "%{SYSLOGBASE} %{DATA:message}"
        }
        overwrite => ["message"]
    }
}

filter {
    json {
        source => "message"
        target => "jsoncontent"
    }
}

filter {
      grok {
          patterns_dir => ["/root/logstash-6.2.3/patterns"]
          match => {
              "message" => "%{TIMESTAMP_ISO8601:timestamp}\[%{CURR_THREAD:threadname}\]\s%{LOGLEVEL:loglevel}\s\s\[%{INVOKE_METHOD:method}\]\s-\s%{LOG_MSG:information}"
          }
      }
}

自定义文件放在目录 patterns_dir：  
CURR_THREAD ([a-zA-Z]+?\-[a-zA-Z]+?\-[0-9]+?\-[a-zA-Z]+?\-[0-9])
INVOKE_METHOD (.+?\(.+?\))
LOG_MSG (.*)
```



### Outputs

outputs是logstash处理管道的最末端组件。一个event可以在处理过程中经过多重输出，但是一旦所有的outputs都执行结束，这个event也就完成生命周期。一些常用的outputs包括：
elasticsearch：如果你计划将高效的保存数据，并且能够方便和简单的进行查询

输出既可以输出到elasticsearch中，也可以指定到标准输出stdout在控制台打印。

* file：将event数据保存到文件中。
* graphite：将event数据发送到图形化组件中，一个很流行的开源存储图形化展示的组件。http://graphite.wikidot.com/。
* statsd：statsd是一个统计服务，比如技术和时间统计，通过udp通讯，聚合一个或者多个后台服务，如果你已经开始使用statsd，该选项对你应该很有用。



### Codecs

* codecs 是基于数据流的过滤器，它可以作为input，output的一部分配置。Codecs可以帮助你轻松的分割发送过来已经被序列化的数据。流行的codecs包括 json,msgpack,plain(text)。
* json：使用json格式对数据进行编码/解码
multiline：将汇多个事件中数据汇总为一个单一的行。比如：java异常信息和堆栈信息
获取完整的配置信息，请参考 Logstash文档中 "plugin configuration"部分。



## [下载](https://www.elastic.co/cn/downloads/logstash)

localhost:9600

在 logstash console 輸入  nihao console

可得到响应

### 使用配置文件

-f：通过这个命令可以指定Logstash的配置文件，根据配置文件配置logstash

-e：后面跟着字符串，该字符串可以被当做logstash的配置（如果是“” 则默认使用stdin作为输入，stdout作为输出）

-l：日志输出的地址（默认就是stdout直接在控制台中输出）

　　-t：测试配置文件是否正确，然后退出。

--config.reload.automatic （-r）选项的意思是启用自动配置加载，以至于每次你修改完配置文件以后无需停止然后重启Logstash

--config.test_and_exit（-t）选项的意思是解析配置文件并报告任何错误

 --log.level

-h --help 打印命令列表

使用-e参数在命令行中指定配置是很常用的方式
```
logstash -e "input { stdin {} } output { stdout {} }"
bin/logstash -f logstash-simple.conf

input {
      kafka {
          bootstrap_servers => ["你的kafka服务器地址:9092"]
          topics => ["serverlogs"] //数组类型，可配置多个topic
          type => "log4j-json" //所有插件通用属性,尤其在input里面配置多个数据源时很有用
          codec => json
          client_id => "test"
          group_id => "test"
          auto_offset_reset => "latest" //从最新的偏移量开始消费
          consumer_threads => 5
          decorate_events => true //此属性会将当前topic、offset、group、partition等信息也带到message中
        
           
      }
  }
  output {
      stdout {
        #"控制台"
        codec => rubydebug
      }
      elasticsearch {
          hosts => ["你的elasticsearch服务器地址:9200"]
          index => "applogstash-%{+YYYY.MM.dd.HH}"
      }
  }
  
  
  
# filebeat 
# 输入源
input {
      beats {
          port => "5044"
      }
  }

output {
      #"控制台"
      stdout {
        codec => rubydebug
      }
      elasticsearch {
          hosts => ["你的elasticsearch服务器地址:9200"]
          index => "applogstash-%{+YYYY.MM.dd.HH}"
      }
  }
```

#### 命令参数

>  -n, --node.name NAME          Specify the name of this logstash instance, if no value is given
> ​                                  it will default to the current hostname.
> ​                                   (default: "LAPTOP-RS4ICTB6")
>  -f, --path.config CONFIG_PATH Load the logstash config from a specific file
> ​                                  or directory.  If a directory is given, all
> ​                                  files in that directory will be concatenated
> ​                                  in lexicographical order and then parsed as a
> ​                                  single config file. You can also specify
> ​                                  wildcards (globs) and any matched files will
> ​                                  be loaded in the order described above.
>  -e, --config.string CONFIG_STRING Use the given string as the configuration
> ​                                  data. Same syntax as the config file. If no
> ​                                  input is specified, then the following is
> ​                                  used as the default input:
> ​                                  "input { stdin { type => stdin } }"
> ​                                  and if no output is specified, then the
> ​                                  following is used as the default output:
> ​                                  "output { stdout { codec => rubydebug } }"
> ​                                  If you wish to use both defaults, please use
> ​                                  the empty string for the '-e' flag.
> ​                                   (default: nil)
>
>  --path.data PATH              This should point to a writable directory. Logstash
> ​                                  will use this directory whenever it needs to store
> ​                                  data. Plugins will also have access to this path.
> ​                                   (default: "E:/developer/bat/ELK/logstash-7.9.1/data")
> ​    -p, --path.plugins PATH       A path of where to find plugins. This flag
> ​                                  can be given multiple times to include
> ​                                  multiple paths. Plugins are expected to be
> ​                                  in a specific directory hierarchy:
> ​                                  'PATH/logstash/TYPE/NAME.rb' where TYPE is
> ​                                  'inputs' 'filters', 'outputs' or 'codecs'
> ​                                  and NAME is the name of the plugin.
> ​                                   (default: [])
> ​    -l, --path.logs PATH          Write logstash internal logs to the given
> ​                                  file. Without this flag, logstash will emit
> ​                                  logs to standard output.
> ​                                   (default: "E:/developer/bat/ELK/logstash-7.9.1/logs")
> ​    --log.level LEVEL             Set the log level for logstash. Possible values are:
>
>                                     - fatal
>                                     - error
>                                     - warn
>                                     - info
>                                     - debug
>                                     - trace
>                                                                       (default: "info")
>     ​    --config.debug                Print the compiled config ruby code out as a debug log (you must also have --log.level=debug enabled).
>  ​                                 ​                                  WARNING: This will include any 'password' options passed to plugin configs as plaintext, and may result
>  ​                                 ​                                  in plaintext passwords appearing in your logs!
>  ​                                  ​                                   (default: false)
>     ​    -i, --interactive SHELL       Drop to shell instead of running as normal.
>  ​                                 ​                                  Valid shells are "irb" and "pry"
>     ​    -V, --version                 Emit the version of logstash and its friends,
>  ​                                 ​                                  then exit.
>     ​    -t, --config.test_and_exit    Check configuration for valid syntax and then exit.
>  ​                                  ​                                   (default: false)
>     ​    -r, --config.reload.automatic Monitor configuration changes and reload
>  ​                                 ​                                  whenever it is changed.
>  ​                                 ​                                  NOTE: use SIGHUP to manually reload the config
>  ​                                  ​                                   (default: false)
>     ​    --config.reload.interval RELOAD_INTERVAL How frequently to poll the configuration location
>  ​                                 ​                                  for changes, in seconds.
>  ​                                  ​                                   (default: #<LogStash::Util::TimeValue:0x123255d @duration=3, @time_unit=:second>)
>     ​    --http.enabled ENABLED        Can be used to disable the Web API, which is
>  ​                                 ​                                  enabled by default.
>  ​                                  ​                                   (default: true)
>     ​    --http.host HTTP_HOST         Web API binding host (default: "127.0.0.1")
>     ​    --http.port HTTP_PORT         Web API http port (default: 9600..9700)
>     ​    --log.format FORMAT           Specify if Logstash should write its own logs in JSON form (one
>  ​                                 ​                                  event per line) or in plain text (using Ruby's Object#inspect)
>  ​                                  ​                                   (default: "plain")
>     ​    --path.settings SETTINGS_DIR  Directory containing logstash.yml file. This can also be
>  ​                                 ​                                  set through the LS_SETTINGS_DIR environment variable.
>  ​                                  ​                                   (default: "E:/developer/bat/ELK/logstash-7.9.1/config")
>     ​    --verbose                     Set the log level to info.
>  ​                                 ​                                  DEPRECATED: use --log.level=info instead.
>     ​    --debug                       Set the log level to debug.
>  ​                                 ​                                  DEPRECATED: use --log.level=debug instead.
>     ​    --quiet                       Set the log level to info.
>  ​                                 ​                                  DEPRECATED: use --log.level=info instead.
>     ​    -h, --help                   print help







实用的例子

Apache 日志(从文件获取)
条件判断
Syslog