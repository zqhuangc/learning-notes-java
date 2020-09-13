# WebSocket

Comet
​	一种长期持有的 HTTP 请求Web 应用模型，允许Web 服务器向浏览器推送数据。



Comet 是一个整体的术语（Umbrella term），包括多种技术完成交互。包括 Ajax Push、Reverse Ajax、 Two-way-web、HTTP Streaming and HTTP server push

## WebSocket 协议

WebSocket 是一种通讯协议，通过单个TCP连接提供完全多工（full-duplex）通讯管道。WebScoket 协议于2011年被IETF标准化，作为 RFC 6455，同时 Web IDL中的 WebSocket API 正在被W3C 标准化。



  WebSocket 被 Web浏览器与Web服务器实现，然而它能用于任何客户端或服务器应用。WebSocket 协议是基于TCP的独立协议。

IETF ：Internet Engineering Task Force，一个开放的组织，致力于开发和提升因特网标准。

RFC：Request for Comments，一种IETF发行类型。



- 协议握手

为建立一个WebSocket 连接，客户端发送WebSocket 握手请求，服务器返回握手的响应。

## Java WebSocket API（JSR-356）

开发 WebSocket 的Java API 集合

版本：Java WebSocket API 1.1（JSR-356）

- 术语

  > 端点（Endpoint）
  > 连接（Connection）
  > 对点（Peer）
  > 会话（Session）
  > 客户端端点、服务器端点



### 端点生命周期（Endpoint Lifecycle）

- 打开连接
  Endpoint#onOpen(Session,EndpointConfig)
  @OnOpen
- 关闭连接
  Endpoint#onClose(Session,CloseReason)
  @OnClose
- 错误
  Endpoint#onError(Session,Throwable)
  @OnError

### 会话（Sessions）

API：javax.websocket.Session

- 接受消息：javax.websocket.MessageHandler
  部分
  整体
- 发送消息：javax.websocket.RemoteEndpoint.Basic

### 配置（Configuration）

- 服务端配置（javax.websocket.ServerEndpointConfig）

  > @ServerEndpoint
  >
  > URI 映射
  > 子协议协商
  > 扩展点修改
  > Origin检测
  > 握手修改
  > 自定义端点创建

- 客户端配置（javax.websocket.ClientEndpointConfig）

  > 子协议
  > 扩展点
  > 客户端配置修改

### 部署（Deployment）

- 应用部署到Web 容器
  WEB-INF/classes
  WEB-INF/lib
- 应用部署到独立WebSocket 服务器
  javax.websocket.server.ServerApplicationConfig
- 编程方式
  **javax.websocket.server.ServerContainer**

## Spring WebSocket 抽象

## WebSocket Spring Boot 整合

