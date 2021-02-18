对于`IDEA`系列编辑器，XML 文件是不能放在 java 文件夹中的，IDEA 默认不会编译源码文件夹中的 XML 文件，可以参照以下方式解决：

- 将配置文件放在 resource 文件夹中
- 对于 Maven 项目，可指定 POM 文件的 resource

```xml
<build>
  <resources>
      <resource>
          <!-- xml放在java目录下-->
          <directory>src/main/java</directory>
          <includes>
              <include>**/*.xml</include>
          </includes>
      </resource>
      <!--指定资源的位置（xml放在resources下，可以不用指定）-->
      <resource>
          <directory>src/main/resources</directory>
      </resource>
  </resources>
</build>
```

> TIP
>
> 注意！Maven 多模块项目的扫描路径需以 `classpath*:` 开头 （即加载多个 jar 包下的 XML 文件）