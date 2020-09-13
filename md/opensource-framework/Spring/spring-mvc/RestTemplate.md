## ClientHttpRequestFactory

```java
// @since 3.0  RestTemplate这个体系都是3.0后才有的
@FunctionalInterface
public interface ClientHttpRequestFactory {	

	// 返回一个ClientHttpRequest，这样调用其execute()方法就可以发送rest请求了~
	ClientHttpRequest createRequest(URI uri, HttpMethod httpMethod) throws IOException;
}
```



AsyncClientHttpRequestFactory

```java
// 在Spring5.0后被标记为过时了，被org.springframework.http.client.reactive.ClientHttpConnector所取代（但还是可用的嘛）
@Deprecated
public interface AsyncClientHttpRequestFactory {

	// AsyncClientHttpRequest#executeAsync()返回的是ListenableFuture<ClientHttpResponse>
	// 可见它的异步是通过ListenableFuture实现的
	AsyncClientHttpRequest createAsyncRequest(URI uri, HttpMethod httpMethod) throws IOException;
}
```

#### SimpleClientHttpRequestFactory

```java
public class SimpleClientHttpRequestFactory implements ClientHttpRequestFactory, AsyncClientHttpRequestFactory {

	private static final int DEFAULT_CHUNK_SIZE = 4096;
	@Nullable
	private Proxy proxy; //java.net.Proxy
	private boolean bufferRequestBody = true; // 默认会缓冲body
	
	// URLConnection's connect timeout (in milliseconds).
	// 若值设置为0，表示永不超时 @see URLConnection#setConnectTimeout(int)
	private int connectTimeout = -1;
	// URLConnection#setReadTimeout(int) 
	// 超时规则同上
	private int readTimeout = -1;
	
	//Set if the underlying URLConnection can be set to 'output streaming' mode.
	private boolean outputStreaming = true;

	// 异步的时候需要
	@Nullable
	private AsyncListenableTaskExecutor taskExecutor;
	... // 省略所有的set方法
	
	@Override
	public ClientHttpRequest createRequest(URI uri, HttpMethod httpMethod) throws IOException {
		
		// 打开一个HttpURLConnection
		HttpURLConnection connection = openConnection(uri.toURL(), this.proxy);
		// 设置超时时间、请求方法等一些参数到connection
		prepareConnection(connection, httpMethod.name());

		//SimpleBufferingClientHttpRequest的excute方法最终使用的是connection.connect();
		// 然后从connection中得到响应码、响应体~~~
		if (this.bufferRequestBody) {
			return new SimpleBufferingClientHttpRequest(connection, this.outputStreaming);
		} else {
			return new SimpleStreamingClientHttpRequest(connection, this.chunkSize, this.outputStreaming);
		}
	}

	// createAsyncRequest()方法略，无非就是在线程池里异步完成请求
	...
}
```



HttpClient

#### AbstractClientHttpRequestFactoryWrapper

##### `InterceptingClientHttpRequestFactory`（重要）

ClientHttpRequestInterceptor



## RestOperations



**Form Data方式**：我们用from表单提交的方式就是它；使用ajax（注意：这里指的是jQuery的ajax，而不是源生js的）默认的提交方式也是它~

**request payload方式**：多部分方式/json方式



















