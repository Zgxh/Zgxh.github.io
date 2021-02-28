---
title: 《图解HTTP》读书笔记
date: 2020-03-06 22:19:11
tags: 计算机网络
categories: 专业课
---

# 第一章 Web基础

- 客户端：通过发送请求来获取服务器资源的web浏览器等
- HTTP：HyperText Transfer Protocol

## TCP/IP

TCP/IP协议族按层次分为：应用层、传输层、网络层、数据链路层
- 应用层：决定了向用户提供应用服务时通信的活动。
    - FTP(File Transfer protocol)、DNS服务(Domain Name System)、HTTP协议等
- 传输层：提供处于网络连接中的两台计算机之间的数据传输。
    - TCP协议(Transmission Control Protocol)、UDP协议(User Data Protocol)
- 网络层：选择传输路线来处理数据包，数据包是网络传输的最小数据单位。
    - IP协议(Internet Protocol)
- 数据链路层：处理连接网络的硬件部分。

## 与HTTP密切相关的协议

### IP协议

- 位于网络层，负责传输数据包。
- 传输根据IP地址和MAC地址。
- 使用ARP协议(Address Resolution Protocol)来解析地址：ARP协议可以通过IP地址反查出MAC地址。
- 网络通信的中转机制称为 路由选择。

### TCP协议

- 提供可靠的字节流服务。
    - 把大块数据分割成以报文段为单位的数据包
    - 可以保证传输的准确可靠性
- 三次握手：发送端首先发送一个带 SYN 标志的数据包给对方。接收端收到后，回传一个带有 SYN/ACK 标志的数据包以示传达确认信息。最后，发送端再回传一个带 ACK 标志的数据包，代表“握手”结束。

### DNS服务

DNS协议通过域名找IP地址，或者通过IP地址反查域名。

## URI和URL

URI：Uniform Resource Indentifier 统一资源标识符
URI用字符串标识某一互联网资源，URL标识资源在互联网上所处的位置。
- URL是URI的子集

# 第二章 简单的HTTP协议

- HTTP请求的报文包括：请求方法、请求URI、协议版本、可选的请求head字段、内容实体。
- 响应报文包括：协议版本、状态码、状态码对应的原因短语、可选的相应head字段、实体主体。
- HTTP协议是无状态协议，对发送过的请求或响应不做持久化处理。
    - 为了实现状态保持，引入了Cookie技术。
- HTTP/1.1支持的请求方法：
    - GET
    - POST
    - PUT
    - HEAD
    - DELETE
    - OPTIONS请求方法：查询指定URI支持的方法。
    - TRACE:追踪路径
    - CONNECT:用隧道协议连接代理

## 持久连接

HTTP Persistent Connections，只要任意一端没有明确提出断开连接，则保持TCP连接状态。

- 减少了TCP连接重复建立和断开的额外开销。

管线化 pipelining：并行发送HTTP请求。

## Cookie

Cookie技术通过在请求和响应报文中写入Cookie信息来控制客户端的状态。

- Cookie会根据Server端反馈的响应报文中的一个叫做Set-Cookie字段，通知Client端保存Cookie，当下次Client再往Server发送HTTP请求时，Client会自动在请求报文中加入Cookie值。
- Server端收到Client发送的Cookie后，检查是哪个Client发送的连接请求，然后对比Server上的记录，得到之前的状态信息。

# 第三章 HTTP报文

## HTTP报文

HTTP报文结构：
- 报文首部
- 空行 CR+LF， 回车并换行
- 报文主体

请求报文首部：
- 请求行：请求方法 + URI + HTTP版本
- 首部字段：请求首部、通用首部、实体首部
- 其他：Cookie等

响应报文首部：
- 状态行：响应状态码 + 原因短语 + HTTP版本
- 首部字段：响应首部、通用首部、实体首部
- 其他：Cookie等

<center>
    <img src="./HTTP报文.jpg" >
</center>

<center>
    <img src="./HTTP报文2.jpg" >
</center>

## 压缩编码传输

报文主体和实体主体的差异：
- 报文 message：HTTP通信中的基本单位，由 8 bit组字节流 组成，通过HTTP通信传输。
- 实体 entity：传输中的有效载荷数据，包括实体首部和实体主体。
- HTTP报文主体用于传输请求或响应的实体主体。通常二者相等，当编码后，实体主体内容变化，二者则产生差异。

分块传输编码：在传输大数据时，把数据分块，让浏览器逐步显示页面。
- 每个块用十六进制标记大小，最后一块用 0(CR+LF) 标记

## 范围请求

指定请求的范围，只获取部分资源。
- 对于范围请求，响应会返回状态码 206 Partial Content
- 对于多重范围的 范围请求，响应先在首部content-type标明multipart/byteranges，再返回报文。
- HTTP首部的Content-Type对应Spring MVC中RequestMapping的consumes参数，表示HTTP请求中的媒体类型，二者需对应。

## 内容协商

Client与Server交涉响应资源内容，然后Server提供给Client最合适的资源形式。
- 判断基准包括：语言、字符集、编码方式等。
- 包括首部字段：Accept, Accept-Charset, Accept-Encoding, Accept-Language, Content-Language
- Accept字段对应Spring MVC中RequestMapping的produces参数，当Accept字段包含produces指定的返回内容类型时才可返回。

协商类型：
- 服务器驱动协商 Server-driven Negotiation：服务器自动参考请求首部
- 客户端驱动协商 Client-driven Negotiation：用户自选浏览器选项，或者js脚本自动选择，如按os类型或web浏览器类型选择。
- 透明协商 Transparent Negotiation：Server和Client各自进行内容协商。

# 第四章 HTTP状态码

正常：2XX
错误：4XX，5XX

<center>
    <img src="./HTTP状态码.jpg" >
</center>

## 常使用的14种状态码

### 200 OK

- 请求成功处理，内容正常返回。

### 204 No Content

- 请求成功处理，但响应报文不含实体主体，即没有资源返回。
- 浏览器页面不更新。

### 206 Partial Content

- 客服端进行了范围请求，服务器成功执行了这部分GET请求。
- 响应报文中包含Content-Range指定的范围内容。

### 301 Moved Permanently

- 永久性重定向。
- 表示请求的资源已被分配了新的URI。

### 302 Found

- 临时性重定向。
- 表示请求的资源被临时分配了新的URI，希望用户本次能用新的URI访问。

### 303 See Other

与302类似，不过303明确提示客户端应该用 GET 方法去另一URI获取资源。

**当 301、302、303 响应状态码返回时，几乎所有的浏览器都会把POST 改成 GET，并删除请求报文内的主体，之后请求会自动再次发送。**

### 304 Not Modified

客户端发送了带条件的请求，服务器允许访问，但没找到符合条件的资源。
- 虽然在3XX类别中，但与重定向无关。
- 附带的条件指GET请求包含：If-Match, If-Modified-Since, If-None-Match, If-Range, If-Unmodiried-Since

### 307 Temporary Redirect

临时重定向，与302含义相同。
- 307会遵守标准，不会把POST变成GET

### 400 Bad Request

表示请求报文中存在语法错误。

### 401 Unauthorized

表示发送的请求需要有认证信息。
- HTTP认证、BASIC认证、DIGEST认证

### 403 Forbidden

请求的资源被服务器拒绝。
- 原因：如没有访问权限等。

### 404 Not Found

表明服务器上无法找到请求的资源。

### 500 Internet Server Error

表示Server端在执行请求时发生了错误。

### 503 Service Unavailable

表示服务器暂时超负载，或正在进行停机维护，无法处理请求。

# 第五章 Web服务器

## 虚拟主机

一台服务器搭建多个web站点。
- 此时多个web站点域名由DNS解析后对应同一个IP，所以在发送HTTP请求时，须在Host首部完整指定主机名或URI。

## 通信数据转发

### 代理

位于Server与Client之间，接收Client的请求转发给Server，同时接收Server的响应转发给Client。

<center>
    <img src="./代理.jpg" >
</center>

- **缓存代理**：代理转发服务器响应时保存缓存，再次收到对相同资源的请求时直接从代理缓存返回响应。
    - 缓存服务器是代理服务器的一种。
    - 即使存在缓存，也会因为Client要求、缓存有效期等因素向源服务器确认资源有效性。若缓存过期则重新从源服务器获取资源。
    - 另一种缓存是**客户端浏览器缓存**，若过期则重新请求资源。
- **透明代理**：转发请求或响应时，不对报文加工处理。反之，非透明代理。

### 网关

网关是转发其他服务器通信数据的服务器。
- 网关能使服务器提供非HTTP协议服务。
- 利用网关能提高通信的安全性。（在Client与网关之间加密来确保连接安全）

### 隧道

隧道可按要求建立一条Client到Server的通信线路，并使用SSL等手段进行加密。
- 确保Client与Server之间安全通信。

# 第六章 HTTP首部

## HTTP/1.1 首部字段 47种

### 通用首部字段

<center>
    <img src="./通用首部字段.jpg" >
</center>

#### Cache-Control

用于控制缓存行为。

<center>
    <img src="./缓存请求指令.jpg" >
</center>

<center>
    <img src="./缓存响应指令.jpg" >
</center>

#### Connection

- 控制不再转发给代理的首部字段

```java
Connection: 不再转发的首部字段名
```

- 管理持久连接: HTTP/1.1默认连接为持久连接，当Server想断开连接时，则指定Connection字段为close。

#### Date

表明创建HTTP报文的日期和时间。

#### Pragma

为与HTTP/1.0兼容而保留。

```java
Cache-Control: no-cache
Pragma: no-cache        // 兼容HTTP/1.1之前的版本
```

#### Upgrade

检测HTTP协议或其他协议是否可用更高版本进行通信。

#### Transfer-Encoding

规定传输报文主体时采用的编码方式，HTTP/1.1的传输编码方式只对分块传输编码有效。

```java
Upgrade: TLS/1.0        // Upgrade字段仅限于Client与邻接服务器之间
Connection: Upgrade     // 所以使用Upgrade时需要额外指定这句话
```

#### Via

- 追踪报文的传输路径，每经过代理或网关，现在Via字段中附加该服务器信息，然后再转发
- 避免请求回环的发生

### 请求首部字段

<center>
    <img src="./请求首部字段.jpg" >
</center>

#### Accept

通知Server用户代理能够处理的媒体类型、及优先级。
- 用q=xx来指定优先级权重。
- 优先返回权重高的媒体类型。

#### Accept-Charset

通知服务器用户代理支持的字符集及字符集的相对优先顺序.

#### Accept-Encoding

用来告知服务器用户代理支持的内容编码及内容编码的优先级顺序。
- 可一次性指定多种内容编码。

#### Accept-Language

用来告知服务器用户代理能够处理的自然语言集（指中文或英文等）.

#### Authorization

告知服务器，用户代理的认证信息。
- 一般，想要通过Server认证的用户代理会在接收到服务器的401状态码响应后，把Authorization字段加入到请求中。

#### Host

告知服务器，请求的资源所处的互联网主机名和端口号。
- 与虚拟主机的工作机制有关。

#### If-Range

若指定的If-Range字段值（ETag或时间）和请求资源的ETag值或时间相一致时，则作为范围请求处理。反之，返回全部资源。

### 响应首部字段

<center>
    <img src="./响应首部字段.jpg" >
</center>

#### Age

告知Client，源服务器在多久前创建了响应。
- 若创建该响应的服务器是缓存服务器，则Age值是指缓存后的响应再次发起认证到认证完成的时间。
- 创建代理响应时必须加上首部字段Age。

### 实体首部字段

<center>
    <img src="./实体首部字段.jpg" >
</center>

#### Content-Location

当返回资源与实际请求的对象内容不同时，Content-Location会注明报文主体返回资源对应的URI。

#### Content-MD5

是一串由MD5算法生成的值，目的在于检查报文主体在传输过程中是否保持完整，以及确认传输到达。
- 但其对内容的改变无从查证，也无法检测传输过程中的恶意篡改。

## 其他高频首部字段

Cookie：请求首部字段。Client想获得HTTP状态管理支持时，就会在请求中包含从Server接收到的Cookie。
Set-Cookie：响应首部字段。是服务器开始管理Client时，告知Client的Cookie信息。

# 第七章 HTTPS

**SSL： Secure Socket Layer， 安全套接层
TLS： Transport Layer Security， 安全层传输协议**

SSL是独立于HTTP的协议，是当今世界上应用最为广泛的网络安全技术。

与SSL组合使用的HTTP称为**HTTPS （HTTP Secure，超文本安全传输协议）**。
HTTP over SSL

SSL提供了证书手段来确认服务器。证书由值得信任的第三方机构颁发，来证明服务器和客户端的身份。

## HTTPS=HTTP+加密+认证+完整性保护

HTTP直接和TCP通信。
HTTPS则使用了SSL，HTTP先和SSL通信，SSL再和TCP通信。

<center>
    <img src="./HTTPS.jpg" >
</center>

共享密钥加密：加密和解密使用同一个密钥。

SSL采用了**公开密钥加密** （Public-Key cryptography），使用非对称密钥。
- 私有密钥 private key
- 公开密钥 public key
- **发送密文方使用对方的公开密钥进行加密，对方收到密文后，用自己的私有密钥进行解密。**

### HTTPS采用混合加密机制

共享密钥加密 + 公开密钥加密

<center>
    <img src="./HTTPS混合加密机制.jpg" >
</center>

使用**公开密钥证书**来确保公开密钥的正确性。

## HTTPS通信步骤

<center>
    <img src="./HTTPS通信步骤.jpg" >
</center>

1. Client发送Client Hello报文来开始SSL通信。（报文中包括Client支持的SSL版本、加密组件列表(加密算法、密钥长度等)）
2. Server回复Server Hello报文。（内容同样包括SSL版本、**筛选的**加密组件）
3. Server发送Certificate报文。（包含**公钥**，之后Client发送的报文会以该公钥进行加密）
4. Server发送Server Hello Done报文，代表最初阶段的SSL第一次握手协商结束。
5. Client发送Client Key Exchange报文。（包含Pre-master secret随机密码串）
6. Client发送Change Cipher Spec报文，提示Server之后的通信采用Pre-master secret加密。
7. Client发送Finished报文。（包含从连接开始的全部报文的整体校验值）
8. Server发送Change Cipher Spec报文。
9. Server发送Finished报文。
10. 服务器与客户端Finished报文交换完毕，**SSL连接成功建立**。开始在SSL的保护下进行应用层协议通信，发送HTTP请求。
11. 应用层协议通信，回复HTTP响应。
12. HTTP通信结束时由客户端断开连接，通过发送close_notify报文。之后发送TCP FIN报文来关闭与TCP的通信。

- 为了保护报文的完整性，在应用层发送数据时附加MAC （Message Authentication Code）的报文摘要来检测报文是否被篡改。

<center>
    <img src="./HTTPS通信细节.jpg" >
</center>

## TLS和SSL

HTTPS通信使用SSL和TLS两个协议。
- TLS是以SSL 3.0为原型开发的协议，可与SSL统称为SSL。

## HTTPS比HTTP慢2~100倍

1. 通信慢，需要额外进行SSL通信，消耗网络资源大。
2. HTTPS需要做服务器、客户端双方加密、解密，消耗CPU、内存等硬件资源。

所以一般只在包含个人信息等敏感数据时，才进行HTTPS加密通信。

# 第八章 认证机制

HTTP/1.1使用的认证方式：
- BASIC认证
    - 采用Base64编码，解码不需要任何附加信息，安全性差。
- DIGEST认证
- SSL客户端认证
    - 利用HTTPS的客户端证书进行认证客户端，客户端证书要钱
    - 再利用密码来确定用户本人
- FormBase认证（基于表单认证）
    - 由Web应用各自实现，没有标准

# 第九章 基于HTTP的功能追加协议

## Ajax 技术

Asynchronous JavaScript and XML， 异步JS与XML技术，从已加载完毕的Web页面上发起HTTP请求，只更新局部页面，但可能导致大量请求产生。

## Comet 技术

客户端发送内容更新请求时，Server先挂起响应，一旦Server有内容更新，直接主动给客户端返回响应，实现实时更新。
- 但为了维持TCP连接会消耗更多资源。

## SPDY 协议

为了在协议级别消除HTTP的瓶颈。
没有完全改写HTTP，而是在TCP/IP的应用层与传输层之间新加入会话层，控制数据流动；但还是采用HTTP进行通信连接。

<center>
    <img src="./SPDY.jpg" >
</center>

- 通过单一TCP连接，可无限制处理多个HTTP请求。
- 可给请求逐个分配优先级。
- 压缩HTTP请求和响应的首部。
- 支持服务器向客户端的推送。
- 服务器可主动提示客户端请求所需的资源。

## WebSocket：全双工通信标准

- 支持Server到Client的推送。
- 建立起WebSocket连接后，一直保持连接状态，通信时开销减少；WebSocket首部信息小，通信量也减少。
- 在HTTP连接建立后，进行一次握手来实现WebSocket通信。

```java
Upgrade: websocket
Connection: Upgrade
```
