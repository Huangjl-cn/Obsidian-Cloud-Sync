最近在学校做直播点播系统，顺带把 WebSocket、WebRTC、HLS 这些概念捋了一遍。这篇笔记就按我自己的理解顺序来记，从 HTTP 和 WebSocket 的区别开始，再到 React + Spring 里怎么写，最后是一些底层和实际选型的问题。

## 一、HTTP 和 WebSocket 协议基础

### 1.1 都是应用层协议

我一开始就确认了一下：WebSocket 和 HTTP 一样，都是应用层协议。它们通常都跑在 TCP（或 TLS over TCP）之上，在 TCP/IP 四层模型里属于同一层。

WebSocket 有自己的标准（RFC 6455），握手之后双方发的是 WebSocket 数据帧，不再是普通 HTTP 请求/响应。

### 1.2 长连接的问题

我原来以为“HTTP 不建立长连接”，后来发现这个说法不准确。

WebSocket 确实会建立一个持久、全双工的连接。握手成功后，这条连接一直保持，双方随时可以互相发消息。

但 HTTP/1.1 默认也使用 `keep-alive`，TCP 连接可以复用，不必每次请求都重新三次握手。所以 HTTP 也可以有长连接。只是它的通信模型仍然是“客户端发起请求 → 服务器返回响应”，服务器通常不能在这条 HTTP 连接上随意主动给客户端发消息。

更准确地说：WebSocket 是持久、全双工的长连接；HTTP 可以复用 TCP 长连接，但通信模式不同。

### 1.3 握手头是标准 HTTP 格式

WebSocket 握手时，客户端会发一个看起来完全像 HTTP 的请求：

```http
GET /chat HTTP/1.1
Host: example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13
```

这是标准 HTTP/1.1 格式，里面的 `Upgrade`、`Connection`、`Sec-WebSocket-*` 都是 RFC 6455 规定的标准头，不是像 RPC 那样随意自定义的字段。

服务器同意后返回：

```http
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```

`101 Switching Protocols` 也是标准 HTTP 状态码。握手成功后，这条连接就从 HTTP 升级为 WebSocket 协议。

### 1.4 信息交换格式：WebSocket 帧

握手之后，双方不再发 HTTP 报文，而是发 WebSocket 帧。帧是二进制格式，由 RFC 6455 规定。

每个帧头最少 2 字节，包含：

- FIN：是否是消息的最后一帧
- opcode：帧类型（text、binary、close、ping、pong 等）
- MASK：是否使用掩码
- Payload length：数据长度
- Masking-key：掩码密钥
- Payload Data：实际数据

一条消息可以由一个或多个帧组成。第一帧 opcode 是 text 或 binary，后面的分片帧 opcode 是 continuation，FIN=1 表示最后一帧。

应用层 payload 里放什么由双方约定，可以是 JSON、Protobuf、纯文本、二进制文件等。WebSocket 本身只负责传文本或二进制。

## 二、WebSocket 在 React + Java Spring 中的实战

### 2.1 一开始看到的复杂写法

我最初看到的示例用了 STOMP 和 SockJS，配置了消息代理、`@MessageMapping`、`@SendTo` 这些。那套确实重，适合快速做订阅/广播，但理解成本高。

后来发现最基础的 WebSocket 其实很简单：前端 `new WebSocket()`，后端继承 `TextWebSocketHandler`，双方直接发文本。

### 2.2 最简版：前端 React + 后端 Spring

后端依赖：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-websocket</artifactId>
</dependency>
```

配置类：

```java
@Configuration
@EnableWebSocket
public class WebSocketConfig implements WebSocketConfigurer {
    @Override
    public void registerWebSocketHandlers(WebSocketHandlerRegistry registry) {
        registry.addHandler(new MyHandler(), "/ws")
                .setAllowedOrigins("*");
    }
}
```

Handler：

```java
public class MyHandler extends TextWebSocketHandler {
    @Override
    protected void handleTextMessage(WebSocketSession session, TextMessage message) throws Exception {
        String received = message.getPayload();
        session.sendMessage(new TextMessage("服务器回显: " + received));
    }
}
```

前端 React：

```jsx
import { useEffect, useRef, useState } from 'react';

function App() {
  const [messages, setMessages] = useState([]);
  const [input, setInput] = useState('');
  const wsRef = useRef(null);

  useEffect(() => {
    const ws = new WebSocket('ws://localhost:8080/ws');
    wsRef.current = ws;

    ws.onopen = () => console.log('已连接');
    ws.onmessage = (event) => setMessages(prev => [...prev, event.data]);
    ws.onclose = () => console.log('已断开');

    return () => ws.close();
  }, []);

  const send = () => {
    if (wsRef.current && wsRef.current.readyState === WebSocket.OPEN) {
      wsRef.current.send(input);
      setInput('');
    }
  };

  return (
    <div>
      {messages.map((msg, i) => <div key={i}>{msg}</div>)}
      <input value={input} onChange={e => setInput(e.target.value)} />
      <button onClick={send}>发送</button>
    </div>
  );
}
```

前端取值是 `ws.onmessage`，存值是 `ws.send()`；后端取值是 `message.getPayload()`，存值是 `session.sendMessage()`。

### 2.3 和 HTTP Controller 写法的对比

HTTP 的写法：

```java
@RestController
@RequestMapping("/api")
public class UserController {
    @GetMapping("/hello")
    public String hello(@RequestParam String name) {
        return "Hello " + name;
    }
}
```

- `@RestController` 声明 HTTP 控制器
- `@GetMapping` 把路径映射到方法
- 请求来了调一次，返回响应就结束
- 返回值自动变成响应体
- 无状态

WebSocket 基础版：

- `@Configuration` + `@EnableWebSocket` 开启功能
- 在配置类里注册 Handler 到路径
- Handler 继承 `TextWebSocketHandler`，重写 `handleTextMessage`
- 没有 `@GetMapping` 这类注解
- 方法通常返回 void，手动 `session.sendMessage()`
- 连接建立后一直活着，每次收到消息触发回调
- 有状态，`WebSocketSession` 代表连接

感觉上，HTTP 是“你问我答，答完就走”；WebSocket 是“先连上，之后随时互相喊话”。

### 2.4 连接建立和关闭

连接过程：

1. 前端 `new WebSocket('ws://.../ws')` 发起 HTTP 握手
2. 后端 Spring 根据 `/ws` 路由到 Config 里注册的 Handler
3. Spring 升级连接，创建 `WebSocketSession`
4. 触发 `afterConnectionEstablished`
5. 前端 `onopen` 触发

关闭过程：

- 前端可以 `ws.close()`
- 后端可以 `session.close(CloseStatus.NORMAL)`
- 双方都会发送 Close 帧，然后触发各自的关闭回调，最后断开 TCP

后端关闭回调：

```java
@Override
public void afterConnectionClosed(WebSocketSession session, CloseStatus status) {
    sessions.remove(session);
}
```

### 2.5 Config 和 Handler 的更多细节

`TextWebSocketHandler` 只是最常用的一个。Spring 还提供了：

- `BinaryWebSocketHandler`：只处理二进制消息
- `AbstractWebSocketHandler`：可以同时处理文本和二进制
- `PerConnectionWebSocketHandler`：为每个连接创建独立的 Handler 实例

注册时可以绑定多个路径到多个 Handler：

```java
registry.addHandler(new ChatHandler(), "/chat").setAllowedOrigins("*");
registry.addHandler(new DataHandler(), "/data").setAllowedOrigins("*");
```

前端连 `/chat` 就走 `ChatHandler`，连 `/data` 就走 `DataHandler`，互不干扰。

### 2.6 广播：维护 Session 集合

原生 WebSocket 的广播，就是后端自己维护所有在线 `WebSocketSession` 的集合，需要时遍历发送。

```java
@Component
public class MyHandler extends TextWebSocketHandler {
    private static final Set<WebSocketSession> sessions = ConcurrentHashMap.newKeySet();

    @Override
    public void afterConnectionEstablished(WebSocketSession session) {
        sessions.add(session);
    }

    @Override
    public void afterConnectionClosed(WebSocketSession session, CloseStatus status) {
        sessions.remove(session);
    }

    private void broadcast(String text) {
        TextMessage msg = new TextMessage(text);
        for (WebSocketSession s : sessions) {
            if (s.isOpen()) {
                try {
                    s.sendMessage(msg);
                } catch (Exception e) {
                    sessions.remove(s);
                }
            }
        }
    }
}
```

维护时机是连接建立时加入，连接关闭时移除。多个 Handler 可以各维护一份，也可以共享一个全局 `SessionManager`。

## 三、线程模型与 IO 底层

### 3.1 Tomcat 的 HTTP 和 WebSocket 并发

HTTP 的并发能力主要看 `maxThreads`，默认 200。`maxConnections` 默认 10000，但真正处理请求的线程数有限。

WebSocket 基于 NIO，连接不占固定线程，并发上限主要看 `maxConnections` 和操作系统的文件描述符限制。一个事件线程可以管成千上万个连接。

### 3.2 为什么 HTTP 是一个请求一个线程，WebSocket 不是

传统 Servlet 模型里，请求进来后 Tomcat 从线程池拿一个线程，这个线程被这个请求独占，直到响应完成才释放。所以并发能力约等于工作线程数。

WebSocket 连接建立后，注册到 NIO 的 Selector 上，不分配固定线程。有消息来了，事件线程才去处理。所以少量线程就能管理大量连接。

### 3.3 和 Java 21 虚拟线程的区别

感觉上有点像，但层次不同。

虚拟线程是 JVM 线程实现层面的东西，让“一个任务一个线程”重新可行，因为虚拟线程很便宜，阻塞时挂起，不占载体线程。

NIO 是 IO 事件模型，尽量避免阻塞，用少量线程管很多连接。

一个让阻塞变便宜，一个避免阻塞。实际系统里可以组合：HTTP 用虚拟线程，WebSocket 用 NIO。

### 3.4 NIO 是什么

NIO 有两个常见含义：Java NIO（New I/O）和非阻塞 IO（Non-blocking I/O）。在 Tomcat、Netty 这些语境里，通常指基于多路复用的事件驱动非阻塞 IO 模型。

核心是 Channel、Buffer、Selector。一个线程通过 Selector 监听多个 Channel 的事件，谁有数据就处理谁。

BIO 是一个连接一个线程，线程阻塞等数据；NIO 是一个线程管很多连接，谁举手去谁那。

Netty 是基于 Java NIO 的网络框架，封装得更好用，很多框架底层都用它。

## 四、实时音视频与直播选型

### 4.1 聊天功能都用 WebSocket 吗

实时收发用 WebSocket 很合适，但实际项目通常是 HTTP + WebSocket 混合：登录、拉历史消息、搜索用户、上传文件用 HTTP；实时收消息、在线状态、正在输入用 WebSocket。

### 4.2 WebRTC 和 HLS 是什么

WebRTC 不是单一协议，而是一套实时通信技术栈，主要跑在 UDP 上，包含 ICE、STUN、TURN、DTLS、SRTP、SCTP 等。延迟极低，适合连麦、视频会议。

HLS 是基于 HTTP 的流媒体协议，把视频切成小片段，通过 HTTP 下载播放。延迟较高，但 CDN 友好，适合大班直播和点播。

### 4.3 WebRTC 是应用层协议吗

不准确。它是一套协议族/技术栈。信令部分可以跑在 WebSocket 或 HTTP 上，但媒体传输主要跑在 UDP 上，用 SRTP，不是 HTTP 也不是 WebSocket。

### 4.4 只用 WebSocket 传视频的问题

WebSocket 基于 TCP，实时视频怕 TCP 的队头阻塞：一帧丢了，后面的帧都要等重传，画面会卡。TCP 的延迟也会累积。WebRTC 有完整的媒体优化，WebSocket 只给一条字节管道，帧边界、时间戳、序列号、丢包处理、拥塞控制都要自己搞。

服务端转发压力也大，每个观众都要保持连接，不能直接走 CDN 缓存。所以大班直播和点播更常用 HLS + CDN，小班连麦用 WebRTC。

## 五、工业级案例：微信的通信架构

### 5.1 微信网页版和客户端底层

微信网页版核心是长轮询，不是标准 WebSocket。客户端是私有 TCP 长连接 + mmtls 加密 + Protobuf，WebSocket 只是辅助。小程序原生提供 WebSocket 接口。

### 5.2 为什么自研应用层协议

移动网络特殊，弱网、断网、切换网络频繁，运营商 NAT 超时短。自研协议可以自定义心跳、重连、多路复用、二进制编码，省电省流量，安全可控，快速迭代。标准协议是通用方案，微信要的是针对自己场景的最优解。

### 5.3 微信客户端传输层用 TCP 还是 UDP

以 TCP 长连接为核心，用于聊天消息、文件传输等可靠场景。音视频通话用 UDP 或基于 UDP 的协议，比如 QUIC 也在部分场景使用。整体是按业务拆分的混合传输。

---

这篇笔记大概就是我这段时间理下来的内容。从协议基础到前后端写法，再到线程模型和实际选型，最后看了下微信这种工业级系统怎么做取舍。写下来之后，思路清晰了不少。