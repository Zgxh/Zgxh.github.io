---
title: HTTP 超文本传输协议
date: 2021-03-01 11:29:33
tags: 计算机网络
categories: 专业课
---

# HTTP 整理

## 1. 概念整理

### 1.1 URI 和 URL

- URI：Uniform Resource Indentifier 统一资源标识符，用于唯一**标识**一个网络资源
- URL：Uniform Resource Locater 统一资源定位符，用于唯一**定位**一个网络资源
    - URL 是 URI 的子集

### 1.2 虚拟主机

虚拟主机出现在一台服务器搭建多个 web 站点的情况。
- 此时多个 web 站点域名由 DNS 解析后对应同一个 IP，所以在发送 HTTP 请求时，须在 Host 首部完整指定主机名或 URI。

### 1.3 代理

代理位于 Server 与 Client 之间，代理服务器负责接收 Client 的请求转发给 Server，同时接收 Server 的响应转发给 Client。

![代理服务器示意](http://note.youdao.com/yws/public/resource/f31b92f56b7fced3dd47a48b6ac36c7b/xmlnote/6A84B177853441A48E4AB43D8159A9DE/17788)

代理分为缓存代理和透明代理：
- **缓存代理**：又称为缓存服务器，代理转发服务器响应时保存缓存，再次收到对相同资源的请求时直接从代理缓存返回响应。
    - 当 Client 有特殊要求、缓存有效期等因素，会使缓存服务器强制向向源服务器获取新的资源。
    - 另一种缓存是**客户端浏览器缓存**，若过期则重新请求资源。
- **透明代理**：转发请求或响应时，不对报文加工处理，并会传送真实的 IP。
    - 反之，非透明代理。

#### 1.3.1 正向代理与反向代理

**正向代理**需要客户端自己来配置，配置了代理后，浏览器在发送请求时会对报文进行相应的修改。

**反向代理**是对服务器的代理，客户端检测不到代理服务器的存在。通过反向代理后，客户端的请求先到达反向代理服务器，然后由反向代理服务器转发请求到真正的后台服务器。
- 反向代理服务器一般用来实现负载均衡和高可用，并起到保护服务器的效果。

### 1.3.2 HTTP 网关

网关是转发其他服务器通信数据的服务器。
- 网关能使服务器提供非 HTTP 协议服务。网关会把来自客户端的 HTTP 请求转换为其他协议，从而与后端服务器通信。
- 利用网关能提高通信的安全性。（可以在 Client 与网关之间通过 SSL 加密来确保连接安全）

![网关](http://note.youdao.com/yws/public/resource/f31b92f56b7fced3dd47a48b6ac36c7b/xmlnote/DC3F47ECAC334121A32970F1D64453C0/17974)

### 1.3.3 Web 隧道 Web Tunnel

隧道可按要求建立一条 Client 到 Server 的通信线路，并使用 SSL 等手段进行加密，这种机制让使用 HTTP 代理的客户端可以访问 TLS 网站（HTTPS）。

Web 隧道允许用户通过 HTTP 连接发送非 HTTP 流量，这样就可以在 HTTP 上捎带其他协议数据了。使用 Web 隧道最常见的原因就是要在 HTTP 连接中嵌入非 HTTP 流量，这样，这类流量就可以穿过只允许 Web 流量通过的防火墙了。

**HTTP CONNECT 方式**：客户端要求 HTTP 代理服务器将 TCP 连接转发到所需的目的地，然后代理服务器继续代理客户端进行连接。服务器建立连接后，代理服务器将继续代理与客户端之间的 TCP 流。只有初始连接请求是 HTTP，之后服务器将仅代理建立的 TCP 连接。

![HTTP CONNECT方式建立隧道](http://note.youdao.com/yws/public/resource/f31b92f56b7fced3dd47a48b6ac36c7b/xmlnote/A3D1C39923C5481FB01BB686B700B065/17986)

## 2. HTTP 简介

HTTP协议是**Hyper Text Transfer Protocol 超文本传输协议**的缩写，他基于TCP/IP协议来传递数据。

### HTTP 工作原理

HTTP 协议工作与 C-S 架构上。浏览器作为HTTP客户端向WEB服务器发送请求。

#### 特点

1. **无连接**：每次连接只处理一个请求，处理完请求即断开。后来 `keep-alive` 可以保持连接，避免连接的频繁断开与建立。
2. **媒体独立**：任何类型的数据都可以通过HTTP发送
3. **无状态**：对于事务处理没有记忆能力。比如：不会记录你的登录信息。

##### 长短连接

http 1.0 默认短连接，http 1.1 默认长连接。
```
Connection: keep-alive
```

http 长短连接的实质是 tcp 的长短连接。
- 短连接：管理简单，每个连接都是有用的连接；
- 长连接：长连接建立后会有一个定时器，如果两小时内连接中没有动作，则服务器主动发送探测报文，如果有响应，则重置定时器；如果因为客户端崩溃而没有响应，则间隔75秒发送10个这样的探测报文，如果还没响应，就关闭连接；如果客户端没有崩溃但已经关闭了从客户端到服务端的连接，则服务端的探测报文会收到一个复位的响应，提示服务端关闭连接。

tomcat 中默认支持 keep-alive ，默认参数：
- maxKeepAliveRequest：一个长连接接受的最大请求数 100
- keepAliveTimeout：一个长连接的最长空闲时间 60s

具体是长短连接依赖于客户端的请求，看Connection字段。

### 2.1 HTTP/1.1支持的请求方法

1. GET ：请求获取某个资源
2. POST ：向服务器提交数据处理请求，请求体中包含实体主体，如表单等。
3. PUT ： 向服务器提交数据来取代原有数据。
4. HEAD ：与 GET 类似，但只会返回报文首部。
5. DELETE ： 请求服务器删除指定内容。
6. OPTIONS ：查询指定 URI 支持的 HTTP 请求方法。   
7. TRACE : 追踪路径
8. CONNECT : 用隧道协议连接代理
9. PATCH ：对某资源发起局部修改请求

### http 1.1 

1. 持久连接
2. 增加了 host 字段
3. 实现了范围请求与断点续传
4. 管线化 pipelining：多个http请求放在同一个tcp连接中依次发送，不用等待服务器的响应，但客户端要按照请求的顺序来依次接收响应。

### http 2.0

1. http 1.1 采用的是基于文本的解析，http 2 采用了基于二进制的解析。
2. 多路复用。多个请求可在同一个连接上并行执行，每个请求对应一个 request id，接受方可以根据 request id 来把 request归属到各自不同的服务端请求中。
3. header 压缩。
4. 服务端推送。

#### get post 区别

1. get一般直接在url拼接参数，不适合传递敏感信息；post一般把参数放到请求体中，更安全；
2. get参数长度有限制，受浏览器影响；post没有。
3. GET在浏览器回退时是无害的，而POST会再次提交请求。
4. get会把header和data同时发出，是一个tcp数据包；post则先发header，响应100 continue，再发body，是两个tcp数据包。这也与浏览器有关。
    - 原因：网络好的时候，发一个包和两个包几乎没有时间差别；网络差的时候，发两次包在验证数据包完整性上有优势。

## 2.2 HTTP 报文

报文是 HTTP 通信中的**基本单位**，由 8 bit 为单位的**字节流**组成，通过 HTTP 通信传输。

HTTP 报文结构：
- 报文首部
- 空行 `CR + LF`， 回车并换行
- 报文主体

<!--![](http://note.youdao.com/yws/public/resource/f31b92f56b7fced3dd47a48b6ac36c7b/xmlnote/DCF7CE7D19CB448EBA91949F1487D8DB/16364)-->

<!--![](http://note.youdao.com/yws/public/resource/f31b92f56b7fced3dd47a48b6ac36c7b/xmlnote/3186376859F54C25B7A2C928F8BDE5F2/16367)-->

### 2.2.1 请求报文

请求报文首部：
- 请求行：请求方法 + URI + HTTP 版本
- 首部字段：请求首部、通用首部、实体首部
- 其他：Cookie 等

### 2.2.2 响应报文

响应报文首部：
- 状态行：响应状态码 + 描述 + HTTP 版本
- 首部字段：响应首部、通用首部、实体首部
- 其他：Cookie 等

### 2.2.3 报文主体和实体主体的差异

- 报文 message：报文是 HTTP 通信中的基本单位，由 8 bit组字节流 组成，通过HTTP通信传输。
- 实体 entity：实体是传输中的有效载荷数据，包括实体首部和实体主体。

**HTTP 报文主体**用于传输请求或响应的**实体主体**。
- 通常二者相等；
- 当编码后，实体主体内容变化，二者则产生差异。

当请求为 `GET` 时，实体主体为空，当请求为 `POST` 时，则会产生实体主体。

### 2.2.4 分块传输编码

在传输大数据时，把原始数据分块，让浏览器逐步显示页面。
- 每个块用**十六进制**标记大小，最后一块用 `0 (CR+LF)` 标记

### 2.3 HTTP 报文首部

#### 2.3.1 HTTP/1.1 首部字段 47种

##### 1. 通用首部字段

<!--![通用首部字段](http://note.youdao.com/yws/public/resource/f31b92f56b7fced3dd47a48b6ac36c7b/xmlnote/40894F40481A4CB6B361B83D9174A1AE/17619)-->

###### Cache-Control

用于控制缓存行为。

no-cache, no-store, max-age=xx

<!--![缓存请求指令](http://note.youdao.com/yws/public/resource/f31b92f56b7fced3dd47a48b6ac36c7b/xmlnote/D78D3146ED364169BD40DBFDD01DB8A3/17623)-->

<!--![缓存响应指令](http://note.youdao.com/yws/public/resource/f31b92f56b7fced3dd47a48b6ac36c7b/xmlnote/327B66EC4DD44D9ABD6DD328BC618531/17625)-->

###### Connection

1. 控制不再转发给代理的首部字段：`Connection: 不再转发的首部字段名`
2. 管理持久连接: HTTP/1.1 默认连接为持久连接 `Connnection: keep-alive`，
3. 当 Server 想断开连接时，则指定 `Connection: close` 即可。

###### Date

表明创建 HTTP 报文的日期时间。

<!--###### Pragma-->

<!--用于控制缓存行为，为与 HTTP/1.0 兼容而保留。-->

<!--```java-->
<!--Cache-Control: no-cache-->
<!--Pragma: no-cache        // 兼容 HTTP/1.1 之前的版本-->
<!--```-->

###### Upgrade

检测 HTTP 协议或其他协议是否可用更高版本进行通信。

###### Transfer-Encoding

规定传输报文主体时采用的编码方式，HTTP/1.1 的传输编码方式只对分块传输编码有效。

```java
Upgrade: TLS/1.0        // Upgrade字段仅限于Client与邻接服务器之间
Connection: Upgrade     // 所以使用Upgrade时需要额外指定这句话
```

###### Via

- 追踪报文的传输路径，每经过代理或网关，先在 `Via` 字段中附加该服务器信息，然后再转发出去
    - 可以避免请求回环的发生

##### 2. 请求首部字段

![请求首部字段举例](http://note.youdao.com/yws/public/resource/f31b92f56b7fced3dd47a48b6ac36c7b/xmlnote/4BCCD67591B64B879E0BF78C1D4F6BE4/17700)

###### Accept

通知 Server 用户代理能够处理的媒体类型、及优先级。
- 用 `q=xx` 来指定优先级权重。
- 优先返回**权重高**的媒体类型。

###### Accept-Charset

通知服务器用户代理支持的字符集及字符集的相对优先顺序.

###### Accept-Encoding

用来告知服务器用户代理支持的内容编码及内容编码的优先级顺序。
- 可一次性指定多种内容编码。

###### Accept-Language

用来告知服务器用户代理能够处理的自然语言集（指中文或英文等）.

###### Authorization

告知服务器，用户代理的认证信息。
- 一般，想要通过 Server 认证的用户代理会在接收到服务器的 `401` 状态码响应后，把 `Authorization` 字段加入到请求中。

###### Host

告知服务器，请求的资源所处的互联网主机名和端口号。
- `Host` 字段与虚拟主机的工作机制有关。

###### If-Range

- 若指定的 `If-Range` 字段值（ETag 值或时间,etag相当于资源的版本号）和请求资源的 ETag 值或时间相一致时，则作为范围请求处理。
- 反之，返回全部资源。

##### 3. 响应首部字段

![响应首部字段举例](http://note.youdao.com/yws/public/resource/f31b92f56b7fced3dd47a48b6ac36c7b/xmlnote/C30CD0032ACF440D97BF458A4A9BF3C4/17702)

###### Age

告知 Client，源服务器在多久前创建了响应。
- 若创建该响应的服务器是**缓存服务器**，则 `Age` 值是指缓存后的响应再次发起认证到认证完成的时间。
    <!--- 缓存服务器是指存放缓存资源的服务器，缓存资源是经常被访问的资源。-->
- 创建代理响应时必须加上首部字段 `Age`。

##### 4. 实体首部字段

![实体首部字段举例](http://note.youdao.com/yws/public/resource/f31b92f56b7fced3dd47a48b6ac36c7b/xmlnote/A934855072F24ADA9AA62A5CE0676FB2/17728)

<!--###### Content-Location-->

<!--当返回资源与实际请求的对象内容不同时，`Content-Location` 会注明报文主体返回资源对应的 URI。-->

###### content-type

text/html, application/json

###### Content-MD5

是一串由 MD5 算法生成的值，目的在于检查报文主体在传输过程中是否保持完整，以及确认传输到达。
<!--- 但其对内容的改变无从查证，也无法检测传输过程中的恶意篡改。-->

##### 5. 其他高频首部字段

`Cookie`：请求首部字段。Client 想获得 HTTP 状态管理支持时，就会在请求中注明从 Server 接收到的 Cookie。

`Set-Cookie`：响应首部字段。当服务器开始管理 Client 时，会告知 Client 其需要携带的 Cookie 信息。

##### 6. HTTP 首部字段在 Spring MVC 中的体现

- HTTP 首部的 `Content-Type` 对应 Spring MVC 中 `RequestMapping` 注解中的 `consumes` 参数，表示 HTTP 请求中的媒体类型，二者需对应。表示http请求中的内容类型可以被后台消费。
- HTTP 首部的 `Accept` 字段对应 Spring MVC 中 `RequestMapping` 的 `produces` 参数，当 `Accept` 字段包含 `produces` 指定的返回内容类型时才可返回。

#### 2.3.2 客户端与服务器的内容协商

当一项资源被访问时，资源的展现形式是通过**内容协商**来确定的。
- 判断基准包括：语言、字符集、编码方式等。
- 包括首部字段：`Accept`, `Accept-Charset`, `Accept-Encoding`, `Accept-Language`, `Content-Language`

##### 1. 协商类型

- **服务器驱动协商** Server-driven Negotiation：客户端来设置特定的 HTTP 首部，服务器自动参考请求首部来返回指定类型的资源。
    - 这是内容协商的**标准形式**。
    - 客户端请求头：`Accept`, `Accept-Charset`, `Accept-Encoding`, `Accept-Language`。
- 客户端驱动协商 Client-driven Negotiation：用户自选浏览器选项，或者 js 脚本自动选择，如按 os 类型或 web 浏览器类型选择。
- ~~透明协商 Transparent Negotiation：Server 和 Client 各自进行内容协商。~~
    - 已被遗弃。

## 3. HTTP 状态码

- 2XX：正常
- 3XX：重定向
- 4XX：客户端错误
- 5XX：服务器错误

![](http://note.youdao.com/yws/public/resource/f31b92f56b7fced3dd47a48b6ac36c7b/xmlnote/77B24D65014749FA8B39466FDCCF385B/16391)

##### 200 OK

请求被成功处理，内容正常返回。

##### 204 No Content

请求被成功处理，但响应报文不含实体主体，即**没有资源返回**。同时浏览器页面不更新。

##### 206 Partial Content

表示客户端进行了**范围请求**，服务器成功执行了该范围 `GET` 请求。
- 响应报文中包含 `Content-Range` 指定的范围内容。

###### HTTP 范围请求

范围请求会指定请求的范围，只获取部分资源。
- 请求单一范围： `Range: bytes=0-1023`
    - 响应 `206 Partial Content`：`Content-Range: bytes 0-1023/146515 Content-Length: 1024`
- 请求多重范围： `Range: bytes=0-50, 100-150`
    - 响应 `206 Partial Content`: `Content-Type: multipart/byteranges; boundary=3d6b6a416f9b5` 以及各分段的范围响应
- 条件式范围请求： `If-Range: xxxx`
    - 若条件满足，则返回 206 及对应的范围主体
    - 若条件不满足，则返回 200 OK，同时返回整个资源

参考：[检测服务器端是否支持范围请求](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Range_requests)

##### 301 Moved Permanently

**永久性重定向**。表示请求的资源已被分配了新的 URI。

##### 302 Found

**临时性重定向**。表示请求的资源被临时分配了新的 URI，希望客户端本次能用新的 URI 访问。

##### 303 See Other

与 302 类似，不过 303 明确提示客户端应该用 `GET` 方法去另一 URI 获取资源。

**当 301、302、303 响应状态码返回时，几乎所有的浏览器都会自动把 `POST` 改成 `GET`，并删除请求报文内的主体，之后请求会自动再次发送。**

##### 304 Not Modified

客户端发送了**带条件的请求**，服务器允许访问，但没找到符合条件的资源。
- 虽然在 3XX 类别中，但与重定向无关。
- 附带的条件指 `GET` 请求包含：`If-Match`, `If-Modified-Since`, `If-None-Match`, `If-Range`, `If-Unmodified-Since`

##### 307 Temporary Redirect

临时重定向，与 302 含义相同。
- 但 307 会遵守标准，不会把 POST 变成 GET

##### 400 Bad Request

表示请求报文中存在语法错误。

##### 401 Unauthorized

表示未认证，提示客户端发送的请求中需要有认证信息。
- HTTP 认证、BASIC 认证、DIGEST 认证

##### 403 Forbidden

请求的资源被服务器拒绝。
- 原因：如没有访问权限等。

##### 404 Not Found

表明在服务器上无法找到请求的资源。

##### 405 Method Not Allowed 

表明服务器禁用该请求方法。

##### 500 Internet Server Error

表示服务器端在执行请求时发生了错误。

##### 501 Not Implemented 

服务器无法识别该请求，或服务器不具备完成该请求的功能。

##### 502 Bad GateWay

网关错误，响应超时。

##### 503 Service Unavailable

表示服务器暂时超负载，或正在进行停机维护，无法处理请求。

##### 504 GateWay Timeout 

代理网关等待另一服务器时超时：因为服务器执行操作时间过长，导致响应超时。

## 4. Cookie 与 Session

HTTP 本身是无状态协议，一旦数据交换完成连接就会关闭。服务器无法跟踪每个用户的会话，因此无法验证用户身份和获取用户状态。

于是利用 Cookie 和 Session 机制来验证用户身份、用户状态。

Cookie 通过在**客户端**记录信息确定用户身份，Session 通过在**服务端**记录信息确定用户身份。

### 4.1 Cookie 

cookie 是一小段记录用户信息的文本。
- 如果服务器想让浏览器保存 Cookie，则会在响应报文中添加 `Set-Cookie` 字段，通知浏览器保存 Cookie。当下次 Client 再往 Server 发送 HTTP 请求时，Client 会自动在请求报文中加入 `Cookie` 值。
    - Cookie 附加在 HTTP 请求头内；
- Server 端收到 Client 发送的 `Cookie` 后，解析得到用户身份和用户状态。

#### 4.1.0 cookie 的管理

cookie 需要浏览器的支持，如果想使用 cookie，必须浏览器支持。

Cookie 是由**浏览器**来实现和管理的。分为内存型 cookie 和硬盘型 cookie。
- 内存型 cookie 在浏览器关闭后就会丢失；
- 硬盘型 cookie 可以持久保存在硬盘上，除非到期或手动清除。
    - ==如果 cookie 定义了有效期，则会在硬盘进行保存；否则只在浏览器被关闭前有效。==

#### 4.1.1 Cookie 的颁发与发送

服务端响应中包含 `Set-Cookie` 字段：

```
Connection: keep-alive
Content-Length: 49
Content-Type: application/json; charset=utf-8
Date: Thu, 02 Jan 2020 08:48:08 GMT
Set-Cookie: userName=admin; path=/
Set-Cookie: password=123456; path=/
Set-Cookie: sessionId=9527; path=/; expires=Thu, 02 Jan 2020 08:48:18 GMT; httponly
```

浏览器向服务器发起 HTTP 请求时携带 `Cookie` 的请求头：

```
Accept: application/json, text/plain, */*
Content-Length: 40
Content-Type: application/json;charset=UTF-8
Cookie: userName=admin; password=123456
```

#### 4.1.2 cookie 的特性

1. **不能跨域**。同一个父域名下的两个子域名之间也不行。如网站 images.google.com 与网站 www.google.com。
2. Cookie 由**键值对**构成。
    - `name`：名字，创建后不可修改；
    - `value`：cookile的值；
    - `Max-Age`：cookie 的有效时间，秒为单位。
    - `Domain`：定义可访问该 cookie 的域名。
    - `Path`：定义网站上可以访问 cookie 的页面的路径。以"/"结尾。
    - `Comment`：对该 cookie 的描述。
    - `Secure`：定义cookie的安全性，当该值为true时必须是HTTPS状态下cookie才从客户端附加在HTTP消息中发送到服务端。
    - `Version`：定义cookie的版本，由cookie的创建者定义。
3. cookie 是有大小限制的，大多数浏览器支持最大为 **4096 字节**的 Cookie。cookie不能过大，影响传输速度。
4. Cookie 除了可以在服务器端创建外，也可以在浏览器中用客户端脚本(如 javascript)创建.
5. cookie 不支持修改和删除，修改可以通过覆盖来实现。

#### 4.1.2 Cookie 安全

1. Cookie 是以文本文件的形式保存在本地的，所以不要存放重要信息，或者先加密再保存。
2. 为避免跨域脚本 (XSS) 攻击，应该为 `Set-Cookie` 设置 `HttpOnly` 标记，这样，js 就无法访问到该 Cookie。

### 4.2 Session 会话

Session 即会话，由服务器端维护，每个用户在服务端对应一个独立的 Session，相当于一张客户明细表，一般保存在服务端内存中。

服务器利用 session 在网站的上下文不同页面间传递变量、用户身份认证、记录客户端程序状态记录等。
- session 一般在用户第一次访问服务器时创建，由服务器来维护。记录的是从浏览器连接上服务器，到用户关闭浏览器这一期间的会话信息。
- Session 一般保存在服务器内存中，当面对用户请求时，服务器会去查找用户对应的 Session 信息。

#### 4.2.1 sessionID

服务器通过 HTTP request 中的 sessionID 来识别客户端用户。（Cookie 中的 `JSESSIONID`。）
- 如果没有设置 Session 的生成周期，则 sessionID 储存在用户的内存中，关闭浏览器后 sessionID 丢失；重启浏览器发起请求时，会重新注册 sessionID。
- 如果没有禁用 cookie，则 sessionID 和 session 生存期储存在 cookie 中。
- 如果禁用了 cookie，可以通过 URL 中携带 sessionID （称为 **URL 重写**） 或者把 sessionID 通过隐藏表单传送给服务器。
    - 大部分的手机浏览器都不支持 cookie，所以一般都采用 URL 重写的方式来进行。

#### session 认证流程

1. 用户向服务器发送用户名和密码。
2. 服务器验证通过后，在当前对话（session）里面保存相关数据，比如用户角色、登录时间等等。
3. 服务器向用户返回一个 session_id，写入用户的 Cookie。
4. 用户随后的每一次请求，都会通过 Cookie，将 session_id 传回服务器。
5. 服务器收到 session_id，找到前期保存的数据，由此得知用户的身份。

#### 4.2.2 Sticky Sessions 粘滞会话

如果服务器是分布式的，当使用反向代理做负载均衡时，用户对同一内容的多次请求，可能被转发到了不同的后端服务器，而session在不同服务器之间可能是不共享的，所以这样可能导致问题。

**粘滞会话**：指让用户在一次会话周期内的所有请求始终转发到同一台特定的后端服务器上。
- 对于 Nginx，只需要在 upstream 中声明 ip_hash 即可，每个请求都访问 IP 的 hash 结果分配，这样每个访客固定访问一个后端服务器。

参考：

[HTTP & HTTPS, Session & Cookie](https://chenjiayang.me/2017/07/29/https-cookie-session/)

[如何区分不同用户——Cookie/Session机制详解](https://www.cnblogs.com/zhouhbing/p/4204132.html)

## 5. HTTP/1.1 新特性

1. **默认持久连接**：只要 C/S 任一端没有明确提出断开 TCP 连接，则会一直保持连接，可以发送多次 HTTP 请求。
2. **管线化**：客户端可以同时发送多个 HTTP 请求，不用等待上一个请求响应。
3. **断点续传**：利用 HTTP header，使用分块传输编码，把实体主体进行分块传输。

## 6. HTTPS

### 6.0 加密方式

#### 6.1 对称加密

又称**共享密钥加密**。加密和解密使用同一个密钥。

#### 6.2 非对称加密

又称**公开密钥加密**。非对称加密的密钥是成对的（公钥和私钥）。私钥由自己安全保管不外泄，而公钥则可以发给网络中的任何人。通信的另一方用自己的公钥进行加密，然后自己用私钥进行解密。

### 6.1 TLS 和 SSL

HTTPS 通信使用 SSL 和 TLS 两个协议。TLS 是以 SSL 3.0 为原型开发的协议，可与 SSL 统称为 SSL。
- **SSL**： Secure Socket Layer， 安全套接层协议
- **TLS**： Transport Layer Security， 传输层安全协议

SSL 是独立于 HTTP 的协议，是当今世界上应用最为广泛的网络安全技术。与 SSL 组合使用的 HTTP 称为**HTTPS** （HTTP Secure，超文本安全传输协议）。即 HTTP over SSL。

SSL 采用**非对称加密**的方式来进行加密通信。
- 通过数字证书来确认对方身份。证书信息包括发布机构，所属人，公钥，过期时间等信息。
    - 证书由值得信任的**第三方认证机构** （Certification Authority， CA） 颁发，来证明服务器和客户端的身份，从而证明对方公钥的正确性，防止黑客对公钥进行了修改。
    - 通信时一方携带自己的证书发送消息，对方接收到之后，先拿着收到的证书信息去 CA 验证证书的有效性和正确性，验证成功后再回复消息。

### 6.2 HTTPS = HTTP + 加密 + 认证 + 完整性保护

#### 6.2.1 HTTP 与 HTTPS 的区别

1. 安全性不同。
    - HTTP 报文是明文传输，信息传输不安全。
    - HTTPS 是利用了 SSL 进行加密通信，并具有身份认证功能，可以保证数据传输的安全性。
2. 通信方式不同。
    - HTTP 直接和 TCP 进行通信。
    - HTTPS 则使用了 SSL，HTTP 先和 SSL 通信，SSL 再和 TCP 通信。
3. 占用的端口不用。
    - HTTP 占用 80 口。
    - HTTPS 占用 443 口。
4. 认证身份的方式不同。
    - HTTP 不会去验证对方身份。
    - HTTPS 利用 SSL，通过数字证书来验证身份。
5. 数据完整性验证。
    - HTTP 不会对数据完整性进行验证，因此检测不到是否信息被篡改。
    - HTTPS 采用**数字签名**来实解决数据完整性验证，此外，数字签名还能标识发送者信息。
5. 速度不同：HTTPS 比 HTTP 慢 2~100 倍：
    - HTTPS 需要额外进行 SSL 通信，消耗网络资源大。
    - HTTPS 需要做服务器、客户端双方加密、解密，消耗 CPU、内存等硬件资源。

所以，一般只在包含个人信息等敏感数据时，才进行 HTTPS 加密通信。

![SSL的位置](http://note.youdao.com/yws/public/resource/f31b92f56b7fced3dd47a48b6ac36c7b/xmlnote/9781904B72E24D4A9444E6825C1D72F6/17823)

#### 6.2.2 HTTPS 加密方式

HTTPS 采用混合加密机制来传送消息：**共享密钥加密** + **公开密钥加密**。

因为公开密钥方式比共享密钥加密方式更加复杂，所以应该更少地使用公开密钥加密的方式。因此：
1. 先通过**公开密钥加密**的方式来安全地交换到**共享密钥**。
    - 公开密钥的正确性是通过**数字证书**来保证的。
2. 再利用共享密钥来进行**共享密钥加密**通信。

![HTTPS加密通信方式](http://note.youdao.com/yws/public/resource/f31b92f56b7fced3dd47a48b6ac36c7b/xmlnote/EDB8D5BADBFF44ECADF09A9E7692935E/17849)

<!--![](http://note.youdao.com/yws/public/resource/f31b92f56b7fced3dd47a48b6ac36c7b/xmlnote/0E60C5BB9CEF423CA39A64A7A738960B/17838)-->

#### 6.2.3 HTTPS 通信步骤

![HTTPS 通信步骤](http://note.youdao.com/yws/public/resource/f31b92f56b7fced3dd47a48b6ac36c7b/xmlnote/456C6A9B39354A929D6A23BFED6DBEA1/18078)

##### 1. 客户端向服务端发起请求

1. 客户端生成随机数 R1 发送给服务端；
2. 告诉服务端自己支持哪些加密算法；

##### 2. 服务器向客户端发送数字证书

1. 服务端生成随机数R2;
2. 从客户端支持的加密算法中选择一种双方都支持的加密算法（此算法用于利用 R1，R2，R3 生成会话密钥）;
3. 服务端生成把证书、随机数 R2、会话密钥生成算法，一同发给客户端;

##### 3. 客户端验证数字证书

1. 验证证书的可靠性，先用 CA 的公钥解密被加密过后的证书,能解密则说明证书没有问题，然后通过证书里提供的摘要算法进行对数据进行摘要，然后通过自己生成的摘要与服务端发送的摘要比对；
2. 验证证书合法性，包括证书是否吊销、是否到期、域名是否匹配，通过后则进行后面的流程；
3. 获得证书的公钥、会话密钥生成算法、随机数 R2；
4. 生成一个随机数R3；
5. 根据会话秘钥算法使用 R1、R2、R3 共同生成会话秘钥（**共享对称密钥**）；
6. 用服务端证书的公钥加密随机数 R3 并发送给服务端。

##### 4. 服务器得到会话密钥

1. 服务器用私钥解密客户端发过来的随机数 R3
2. 根据会话秘钥算法使用 R1、R2、R3 生成会话秘钥

##### 5. 客户端与服务端进行加密会话

1. 客户端利用会话密钥进行加密，并发送加密后的数据给服务端；
2. 服务端收到消息，用会话密钥进行解密，然后再用会话密钥加密响应数据，回复给客户端；
3. 客户端用会话密钥解密响应数据，这样就完成了一次请求和响应。

##### 具体

![HTTPS通信过程](http://note.youdao.com/yws/public/resource/f31b92f56b7fced3dd47a48b6ac36c7b/xmlnote/A415F2FB205841CB9C9CA43F7787A845/17847)

1. Client 发送 Client Hello 报文来开始 SSL 通信。（报文中包括Client支持的SSL版本、加密组件列表(加密算法、密钥长度等)）
2. Server 回复Server Hello报文。（内容同样包括SSL版本、**筛选的**加密组件）
3. Server 发送Certificate报文。（包含**公钥**，之后Client发送的报文会以该公钥进行加密）
4. Server 发送Server Hello Done报文，代表最初阶段的SSL第一次握手协商结束。
5. Client 发送Client Key Exchange报文。（包含Pre-master secret随机密码串）
6. Client 发送Change Cipher Spec报文，提示Server之后的通信采用Pre-master secret加密。
7. Client 发送Finished报文。（包含从连接开始的全部报文的整体校验值）
8. Server 发送Change Cipher Spec报文。
9. Server 发送Finished报文。
10. 服务器与客户端 Finished 报文交换完毕，**SSL连接成功建立**。开始在SSL的保护下进行应用层协议通信，发送HTTP请求。
11. 应用层协议通信，回复HTTP响应。
12. HTTP通信结束时由客户端断开连接，通过发送 close_notify 报文。之后发送TCP FIN报文来关闭与TCP的通信。

## Reference

[HTTP就是这么简单](https://mp.weixin.qq.com/s?__biz=MzI4Njg5MDA5NA==&mid=2247483733&idx=2&sn=93b359af4397cd4afa791fdb5f51f0b5&chksm=ebd74054dca0c942be5180cdf0f460ed7f534ca51230147fe0081df15adac76dac9e61d97761&scene=21#wechat_redirect)

[HTTP2和HTTPS来不来了解一下？](https://mp.weixin.qq.com/s?__biz=MzI4Njg5MDA5NA==&mid=2247484302&idx=1&sn=5fafcb988463b5b2df9120552b6dc3f8&chksm=ebd7428fdca0cb99d5ed60296100b315c4ecaefc901fb5bb5448f902c6f41b0fa0dc18d5ee06&scene=21#wechat_redirect)

[网关、隧道和中继](https://www.cnblogs.com/xiaohuochai/p/6180941.html)
