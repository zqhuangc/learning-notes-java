https://github.com/swagger-api

https://zhuanlan.zhihu.com/p/21353795

```java
<dependency>
    <groupId>io.springfox</groupId>
    <artifactId>springfox-swagger-ui</artifactId>
    <version>3.0.0</version>
</dependency>

<dependency>
    <groupId>io.springfox</groupId>
    <artifactId>springfox-swagger2</artifactId>
    <version>3.0.0</version>
</dependency>

```



```java
@Configuration
@EnableSwagger2
public class Swagger2Config {

    @Bean
    public Docket createRestApi() {
        return new Docket(DocumentationType.SWAGGER_2)
                .apiInfo(apiInfo())
                .select()
                .apis(RequestHandlerSelectors.basePackage("info.xiaomo.website"))
                .paths(PathSelectors.any())
                .build();
    }

    private ApiInfo apiInfo() {
        return new ApiInfoBuilder()
                .title("Spring Boot中使用Swagger2构建RESTful APIs")
                .description("api根地址：http://api.xiaomo.info:8080/")
                .termsOfServiceUrl("https://xiaomo.info/")
                .contact("小莫")
                .version("1.0")
                .build();
    }
}
```





相关注解解读

1. @Api

用在类上，说明该类的作用
@Api(value = "UserController", description = "用户相关api")

2. @ApiOperation

用在方法上，说明方法的作用
@ApiOperation(value = "查找用户", notes = "查找用户", httpMethod = "GET", produces = MediaType.APPLICATION_JSON_UTF8_VALUE)

3. @ApiImplicitParams

用在方法上包含一组参数说明

4. @ApiImplicitParam

用在@ApiImplicitParams注解中，指定一个请求参数的各个方面
paramType：参数放在哪个地方
header–>请求参数的获取：@RequestHeader
query–>请求参数的获取：@RequestParam
path（用于restful接口）–>请求参数的获取：@PathVariable
body（不常用）
form（不常用）
name：参数名
dataType：参数类型
required：参数是否必须传
value：参数的意思
defaultValue：参数的默认值

```text
@ApiImplicitParams({
        @ApiImplicitParam(name = "id", value = "唯一id", required = true, dataType = "Long", paramType = "path"),
})
```

5. @ApiResponses

用于表示一组响应

6. @ApiResponse

用在@ApiResponses中，一般用于表达一个错误的响应信息
code：数字，例如400
message：信息，例如”请求参数没填好”
response：抛出异常的类

```text
@ApiResponses(value = {  
          @ApiResponse(code = 400, message = "No Name Provided")  
  })
```

7. @ApiModel

描述一个Model的信息（这种一般用在post创建的时候，使用@RequestBody这样的场景，请求参数无法使用@ApiImplicitParam注解进行描述的时候）
@ApiModel(value = "用户实体类")

8. @ApiModelProperty

描述一个model的属性
@ApiModelProperty(value = "登录用户")



和Swagger UI的集成

首先，从github [swagger-ui](https://link.zhihu.com/?target=https%3A//github.com/swagger-api/swagger-ui) 上下载Swagger-UI, 把该项目dist目录下的内容拷贝到项目的resources的目录public下。



访问

[http://localhost:8080/swagger-ui.html](https://link.zhihu.com/?target=http%3A//localhost%3A8080/swagger-ui.html) 就可以看到效果如下