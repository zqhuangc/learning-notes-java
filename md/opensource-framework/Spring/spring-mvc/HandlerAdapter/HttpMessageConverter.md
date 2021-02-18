

# HttpMessageConverter

```java
// @since 3.0  Spring3.0后推出的   是个泛型接口
// 策略接口，指定可以从HTTP请求和响应转换为HTTP请求和响应的转换器
public interface HttpMessageConverter<T> {

	// 指定转换器可以读取的对象类型，即转换器可将请求信息转换为clazz类型的对象
	// 同时支持指定的MIME类型(text/html、application/json等)
	boolean canRead(Class<?> clazz, @Nullable MediaType mediaType);
	// 指定转换器可以将clazz类型的对象写到响应流当中，响应流支持的媒体类型在mediaType中定义
	boolean canWrite(Class<?> clazz, @Nullable MediaType mediaType);
	// 返回当前转换器支持的媒体类型~~
	List<MediaType> getSupportedMediaTypes();

	// 将请求信息转换为T类型的对象 流对象为：HttpInputMessage
	T read(Class<? extends T> clazz, HttpInputMessage inputMessage) throws IOException, HttpMessageNotReadableException;
	// 将T类型的对象写到响应流当中，同事指定响应的媒体类型为contentType 输出流为：HttpOutputMessage 
	void write(T t, @Nullable MediaType contentType, HttpOutputMessage outputMessage) throws IOException, HttpMessageNotWritableException;
}
```



## HttpMessage

```java
public interface HttpMessage {
	// Return the headers of this message
	HttpHeaders getHeaders();
}

public interface HttpInputMessage extends HttpMessage {
	InputStream getBody() throws IOException;
}
public interface HttpOutputMessage extends HttpMessage {
	OutputStream getBody() throws IOException;
}
```



### FormHttpMessageConverter：form表单提交/文件下载

### AbstractHttpMessageConverter

```java
public abstract class AbstractHttpMessageConverter<T> implements HttpMessageConverter<T> {

	// 它主要内部维护了这两个属性，可议构造器赋值，也可以set方法赋值~~
	private List<MediaType> supportedMediaTypes = Collections.emptyList();
	@Nullable
	private Charset defaultCharset;

	// supports是个抽象方法，交给子类自己去决定自己支持的转换类型~~~~
	// 而canRead(mediaType)表示MediaType也得在我支持的范畴了才行（入参MediaType若没有指定，就返回true的）
	@Override
	public boolean canRead(Class<?> clazz, @Nullable MediaType mediaType) {
		return supports(clazz) && canRead(mediaType);
	}

	// 原理基本同上，supports和上面是同一个抽象方法  所以我们发现并不能入参处理Map，出餐处理List等等
	@Override
	public boolean canWrite(Class<?> clazz, @Nullable MediaType mediaType) {
		return supports(clazz) && canWrite(mediaType);
	}


	// 这是Spring的惯用套路:readInternal  虽然什么都没做，但我觉得还是挺有意义的。Spring后期也非常的好扩展了~~~~
	@Override
	public final T read(Class<? extends T> clazz, HttpInputMessage inputMessage)
			throws IOException, HttpMessageNotReadableException {
		return readInternal(clazz, inputMessage);
	}


	// 整体上就write方法做了一些事~~
	@Override
	public final void write(final T t, @Nullable MediaType contentType, HttpOutputMessage outputMessage)
			throws IOException, HttpMessageNotWritableException {

		final HttpHeaders headers = outputMessage.getHeaders();
		// 设置一个headers.setContentType 和 headers.setContentLength
		addDefaultHeaders(headers, t, contentType);

		if (outputMessage instanceof StreamingHttpOutputMessage) {
			StreamingHttpOutputMessage streamingOutputMessage = (StreamingHttpOutputMessage) outputMessage;
			// StreamingHttpOutputMessage增加的setBody()方法，关于它下面会给一个使用案例~~~~
			streamingOutputMessage.setBody(outputStream -> writeInternal(t, new HttpOutputMessage() {
				// 注意此处复写：返回的是outputStream ，它也是靠我们的writeInternal对它进行写入的~~~~
				@Override
				public OutputStream getBody() {
					return outputStream;
				}
				@Override
				public HttpHeaders getHeaders() {
					return headers;
				}
			}));
		}
		// 最后它执行了flush，这也就是为何我们自己一般不需要flush的原因
		else {
			writeInternal(t, outputMessage);
			outputMessage.getBody().flush();
		}
	}
	
	// 三个抽象方法
	protected abstract boolean supports(Class<?> clazz);
	protected abstract T readInternal(Class<? extends T> clazz, HttpInputMessage inputMessage)
			throws IOException, HttpMessageNotReadableException;
	protected abstract void writeInternal(T t, HttpOutputMessage outputMessage)
			throws IOException, HttpMessageNotWritableException;
			
}
```



##### StringHttpMessageConverter

```java
// @since 3.0  出生非常早
public class StringHttpMessageConverter extends AbstractHttpMessageConverter<String> {
	// 这就是为何你return中文的时候会乱码的原因（若你不设置它的编码的话~）
	public static final Charset DEFAULT_CHARSET = StandardCharsets.ISO_8859_1;
	@Nullable
	private volatile List<Charset> availableCharsets;
	// 标识是否输出 Response Headers:Accept-Charset(默认true表示输出)
	private boolean writeAcceptCharset = true;

	public StringHttpMessageConverter() {
		this(DEFAULT_CHARSET);
	}
	public StringHttpMessageConverter(Charset defaultCharset) {
		super(defaultCharset, MediaType.TEXT_PLAIN, MediaType.ALL);
	}

	//Indicates whether the {@code Accept-Charset} should be written to any outgoing request.
	// Default is {@code true}.
	public void setWriteAcceptCharset(boolean writeAcceptCharset) {
		this.writeAcceptCharset = writeAcceptCharset;
	}

	// 只处理String类型~
	@Override
	public boolean supports(Class<?> clazz) {
		return String.class == clazz;
	}

	@Override
	protected String readInternal(Class<? extends String> clazz, HttpInputMessage inputMessage) throws IOException {
		// 哪编码的原则为：
		// 1、contentType自己指定了编码就以指定的为准
		// 2、没指定，但是类型是`application/json`，统一按照UTF_8处理
		// 3、否则使用默认编码：getDefaultCharset  ISO_8859_1
		Charset charset = getContentTypeCharset(inputMessage.getHeaders().getContentType());
		// 按照此编码，转换为字符串~~~
		return StreamUtils.copyToString(inputMessage.getBody(), charset);
	}


	// 显然，ContentLength和编码也是有关的~~~
	@Override
	protected Long getContentLength(String str, @Nullable MediaType contentType) {
		Charset charset = getContentTypeCharset(contentType);
		return (long) str.getBytes(charset).length;
	}


	@Override
	protected void writeInternal(String str, HttpOutputMessage outputMessage) throws IOException {
		// 默认会给请求设置一个接收的编码格式~~~（若用户不指定，是所有的编码都支持的）
		if (this.writeAcceptCharset) {
			outputMessage.getHeaders().setAcceptCharset(getAcceptedCharsets());
		}
		
		// 根据编码把字符串写进去~
		Charset charset = getContentTypeCharset(outputMessage.getHeaders().getContentType());
		StreamUtils.copy(str, charset, outputMessage.getBody());
	}
	...
}
```



### GenericHttpMessageConverter 子接口









## Spring 注解启用 @EnableWebMvc

###  常用`HttpMessageConverter`

| 名称                                    | 作用                                                         | 读支持MediaType                                              | 写支持MediaType                                        | 备注                                                         |
| --------------------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------ | ------------------------------------------------------------ |
| FormHttpMessageConverter                | 表单与MultiValueMap的相互转换                                | application/x-www-form-urlencoded                            | application/x-www-form-urlencoded和multipart/form-data | 可用于处理下载                                               |
| XmlAwareFormHttpMessageConverter        | Spring3.2后已过期，使用下面AllEnc…代替                       | 略                                                           | 略                                                     |                                                              |
| AllEncompassingFormHttpMessageConverter | 对FormHttp…的扩展，提供了对xml和json的支持                   | 略                                                           | 略                                                     |                                                              |
| SourceHttpMessageConverter              | 数据与javax.xml.transform.Source的相互转换                   | application/xml和text/xml和application/*+xml                 | 同read                                                 | 和Sax/Dom等有关                                              |
| ResourceHttpMessageConverter            | 数据与org.springframework.core.io.Resource                   | \*/*                                                         | \*/*                                                   |                                                              |
| ByteArrayHttpMessageConverter           | 数据与字节数组的相互转换                                     | \*/*                                                         | application/octet-stream                               |                                                              |
| ObjectToStringHttpMessageConverter      | 内部持有一个StringHttpMessageConverter和ConversionService    | 他俩的&&                                                     | 他俩的&&                                               |                                                              |
| RssChannelHttpMessageConverter          | 处理RSS <channel> 元素                                       | application/rss+xml                                          | application/rss+xml                                    |                                                              |
| MappingJackson2HttpMessageConverter     | 使用Jackson的ObjectMapper转换Json数据                        | application/json和application/*+json                         | application/json和application/*+json                   | 默认编码UTF-8                                                |
| MappingJackson2XmlHttpMessageConverter  | 使用Jackson的XmlMapper转换XML数据                            | application/xml和text/xml                                    | application/xml和text/xml                              | 需要额外导包Jackson-dataformat-XML才能生效。从Spring4.1后才有 |
| GsonHttpMessageConverter                | 使用Gson处理Json数据                                         | application/json                                             | application/json                                       | 默认编码UTF-8                                                |
| ResourceRegionHttpMessageConverter      | 数据和org.springframework.core.io.support.ResourceRegion的转换 | application/octet-stream                                     | application/octet-stream                               | Spring4.3才提供此类                                          |
| ProtobufHttpMessageConverter            | 转换com.google.protobuf.Message数据                          | application/x-protobuf和text/plain和application/json和application/xml | 同read                                                 | @since 4.1                                                   |
| StringHttpMessageConverter              | 数据与String类型的相互转换                                   | \*/*                                                         | \*/*                                                   | 转成字符串的默认编码为ISO-8859-1                             |
| BufferedImageHttpMessageConverter       | 数据与java.awt.image.BufferedImage的相互转换                 | Java I/O API支持的所有类型                                   | Java I/O API支持的所有类型                             |                                                              |
| FastJsonHttpMessageConverter            | 使用FastJson处理Json数据                                     |                                                              |                                                        |                                                              |

​		



#### HttpRequestHandler



