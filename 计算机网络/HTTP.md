# HTTP

![计算机网络](https://qraffa-1304595678.cos.ap-guangzhou.myqcloud.com/img/%E8%AE%A1%E7%AE%97%E6%9C%BA%E7%BD%91%E7%BB%9C.png)

## http请求方法

方法GET、HEAD、POST、PUT、DELETE、CONNECT、OPTIONS、TRACE

| 方法    | 安全 | 幂等 |
| ------- | ---- | ---- |
| GET     | √    | √    |
| HEAD    | √    | √    |
| POST    | x    | x    |
| PUT     | x    | √    |
| DELETE  | x    | √    |
| CONNECT | x    | x    |
| OPTIONS | √    | √    |
| TRACE   | √    | √    |

**安全：**这里的安全指的是，这个请求方法不会修改服务器的数据，即这是一个只读的方法。所有的安全方法同时也是幂等的。虽然这些方法是只读的，但服务器仍然能更改其数据，比如写入日志。

**幂等：**指同样的请求被执行一次和执行多次效果是一样的，服务器状态也是一样的。幂等性只与服务器的实际状态相关，而相同的请求被执行多次得到的状态码可能是不相同的。

### GET方法与POST方法

GET方法通常用于请求服务器发送某个资源，POST方法用来向服务器输入数据的。

安全性和幂等性：GET方法是安全幂等的，POST方法则不是

缓存：GET方法是可以被缓存的，POST方法一般不会，除非手动设置

参数：往往GET方法的参数都是在url的，POST参数都是在body中，但这并不是规范，GET方法的参数也可以在body中

参数长度：http协议对于url和body并没有长度限制，对url长度的限制大多是因为浏览器或服务器

## http状态码

### 101 Switching Protocols

协议升级

常见场景：http升级到websocket协议

### 204 No Content

服务器成功响应了请求，但没有返回内容

### 206 Partial Content

客户端发送的请求带有Range字段，请求资源的一部分，服务器成功执行并返回

常见场景：断点续传，大文件加载

### 301 Moved Permanently

永久重定向。客户端请求的资源在其他地方，需要使用返回的URL重新请求。该响应是可以被缓存的，因此将来的请求会直接使用返回的新的URL请求资源。

### 302 Move temporarily

临时重定向。由于是临时的，因此将来的请求客户端仍然需要向原地址继续请求资源。

### 303 See Other

该重定向行为需要改变请求方法，如果原地址的请求方法为POST，那么返回303表示重定向需要使用GET方法。

### 304 Not Modified

未改变。客户端验证缓存有效性时，返回304说明缓存没过期，客户端可以使用缓存中的数据。

### 307 Temporary Redirect

临时重定向。与302具有相同语义，但不能变更请求的方法和消息主体。

### 308 Permanent Redirect

永久重定向。与301具有相同语义，但不能变更请求的方法和消息主体。

### 3xx状态码区分总结

// TODO

### 400 Bad Request

请求报文中存在语法错误。当错误发生时，需要重新修改请求内容，并重新发起请求。

### 401 Unauthorized

未授权。

### 403 Forbidden

禁止。资源不可用，服务器理解了客户端的请求，但拒绝处理。通常是因为服务器中目录和文件的权限问题。

### 404 Not Found

无法找到指定的资源。

### 500 Internal Server Error

服务器内部错误，无法处理请求。通常由服务代码问题引起。

### 502 Bad Gateway

网关错误。服务器作为网关或代理，为了处理请求，访问了下一个服务器，但得到了错误的响应。

https://kebingzao.com/2018/10/05/http-status-code/

https://github.com/febobo/web-interview/issues/144

## http头字段

| 字段名            | 含义                                                        | 示例                                                         | 备注                 |
| ----------------- | ----------------------------------------------------------- | ------------------------------------------------------------ | -------------------- |
| Accept            | 请求头用于告知客户端可以处理的内容类型                      | application/json                                             |                      |
| Accept-Encoding   | 客户端能够处理的内容编码方式                                | gzip, deflate                                                |                      |
| Accept-Language   | 客户端能够处理的自然语言类型                                | zh-CN,zh;q=0.8                                               | q表示优先顺序        |
| Connection        | 优先使用的连接类型                                          | keep-alive                                                   |                      |
| Cookie            | 含有先前由服务器下发并存储到客户端的cookie                  | name=value                                                   | 由一系列的键值对组成 |
| Host              | 请求发送到的服务器的域名                                    | zh.wikipedia.org                                             |                      |
| Referer           | 访问的前一个页面                                            | http://zh.wikipedia.org/wiki/Main_Page                       |                      |
| User-Agent        | 浏览器的浏览器身份标识字符串                                | Mozilla/5.0 (X11; Linux x86_64; rv:12.0) Gecko/20100101 Firefox/21.0 |                      |
| Origin            | 表示请求的来源                                              |                                                              |                      |
| Content-Length    | 请求体/响应体的长度                                         |                                                              |                      |
| Content-Type      | 请求体/响应体的内容类型                                     | application/json                                             |                      |
| Date              | 报文被创建的日期时间                                        | Tue, 15 Nov 1994 08:12:31 GMT                                |                      |
| Server            | 服务器信息                                                  | nginx                                                        |                      |
| Transfer-Encoding | 将实体安全传递给客户端的编码格式                            | chunked                                                      |                      |
| Expires           | 指定一个日期/时间，超过该时间则认为此回应已经过期           | Thu, 01 Dec 1994 16:00:00 GMT                                |                      |
| Cache-Control     | 指定缓存机制                                                | no-cache                                                     |                      |
| Last-Modified     | 请求对象的最后修改日期时间                                  | Tue, 15 Nov 1994 12:45:26 GMT                                |                      |
| If-Modified-Since | 请求校验缓存有效性，参数值为上一次该资源请求的Last-Modified | Tue, 15 Nov 1994 12:45:26 GMT                                |                      |
| ETag              | 对于某个资源某个版本的标识符                                | "737060cd8c284d8af7ad3082f209582d"                           |                      |
| If-None-Match     | 请求校验缓存有效性，参数值为上一次该资源请求的ETag          | "737060cd8c284d8af7ad3082f209582d"                           |                      |

https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/

https://zh.wikipedia.org/wiki/HTTP%E5%A4%B4%E5%AD%97%E6%AE%B5

## http缓存

缓存分为强缓存和协商缓存两种。

强缓存由`cache-control`和`expires`字段控制。协商缓存由`last-modified`和`etag`字段控制。其中协商缓存即使命中缓存也仍然需要向服务器发起请求，来验证缓存的有效性。

### 缓存控制

**expires**

该字段中包含具体的日期时间，在该时间之后，则缓存过期。

**cache-control**

- public：表示该响应可以被客户端、代理服务器等各种对象缓存
- private：表示该响应只能被用户本地浏览器缓存，而不能被代理服务器等缓存
- no-store：不存储请求响应中的任何内容，即不使用缓存
- no-cache：可以缓存内容，但在使用缓存内容前需要请求服务器进行缓存验证
- max-age：设置缓存的有效时间，单位为秒。设置改字段后会覆盖expires字段。

**last-modified**

`last-modified`表示文件的最后修改日期时间。服务器在响应时会返回该字段，在客户端下一次请求时，在请求字段if-modified-since`填上该值，用于验证缓存是否可用。如果缓存未过期，服务器返回304，并且没有新内容，浏览器直接使用缓存即可；如果缓存过期，则服务器返回200和新的内容。

**etag**

etag是资源的特定版本的标识符。服务器在响应时会返回该字段，在客户端下一次请求时，在请求字段`if-none-match`填上该值，用于验证缓存是否可用。如果缓存未过期，服务器返回304，并且没有新内容，浏览器直接使用缓存即可；如果缓存过期，则服务器返回200和新的内容。

>为何需要使用etag，而不是last-modified？
>
>- 有些文档可能会被周期性的重写，但内容数据是一样的，尽管内容没有变化，但修改时间会更新
>- 有些文档可能发生了变更，但所做的修改并不重要，不期望让所有的缓存都失效
>- 有些服务器无法准确的判定其页面的最后修改时间
>- 有些服务器提供的文档会在亚秒间隙发生变化(比如1秒内发生多次变化)，那么以秒为单位的粒度就不够用了

https://www.infoq.cn/article/aiwqlgtlk2eft5yi7doy

https://github.com/amandakelake/blog/issues/13

## http特点

- **简单：**http报文设计的简单易读，能被人读懂

- **可扩展：**通过http中的headers，可以方便的扩展新功能，只需要客户端和服务端就header达成语义一致

- **无状态：**在同一个连接中，两次请求之间是没有关系的，可以通过cookie来创建多个请求共享的上下文，以创建有状态的会话。

## http状态

### cookie

cookie是服务器发送到用户浏览器并保存在本地的一小块数据，它会在浏览器下次向同一服务器再发起请求时被携带并发送到服务器上。通常，它用于告知服务端两个请求是否来自同一浏览器，如保持用户的登录状态。

服务器响应中会设置`set-cookie`字段，浏览器收到响应后将该字段存储在本地，之后对该服务器的请求都会在`cookie`字段中携带上该值。

### session

是一种会话状态的概念.主要的实现是将sessionId保存在服务端,以此来验证客户端发来的session会话.用户在登录后服务器为其生成一个sessionId,并保存在服务端,再将该sessionId发送回客户端.以后客户端的请求需要带上该sessionId,服务端就可以根据该id来验证用户身份.

客户端通常是使用cookie来保存用户的sessionId

### token

是一种登录凭证.服务端根据客户端发来的信息,对其进行加密,然后把该字符串作为凭证发回给用户,以后用户请求时只要带上这个凭证,服务端就根据该凭证来解密,获得里面的相关信息.

### 区别

| 方式    | 存储                                    |      |      |
| ------- | --------------------------------------- | ---- | ---- |
| cookie  | 本地                                    |      |      |
| session | session存储在服务器,sessionid存储在本地 |      |      |
| token   | 本地                                    |      |      |

https://ssshooter.com/2021-02-21-auth/

https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Cookies

https://www.cnblogs.com/cxuanBlog/p/12635842.html

https://juejin.cn/post/6844904034181070861

## http版本

以http/1.0为标准

### http/1.1

- 连接复用：在http1.0中，每一个http请求都需要建立一个tcp连接。因此在http1.1中可以设置`keep-alive`来复用之前的tcp连接，以减小三次握手等开销。
- 支持pipeline管道传输：在http1.0中，每个http请求是串行化的。因此在http1.1中使用管道传输，当第一个请求发出后，不必等待该请求响应就可以发送下一个请求了，可以减少整体的响应时间。
- 增加了缓存控制(cache-control)
- 增加了`Host`首字段：多个域名可以解析到同一个ip上，通过host字段来区分请求的是哪个域名

### http/2

- 二进制协议：在http1.1中，传输的数据是文本，因此需要借助客户端和服务端的CPU做压缩以减少网络带宽。在http2中使用二进制协议，提高了数据传输的效率。
- 首部压缩：同时发出多个请求，并且他们首部是一样或者相似的，那么会对首部使用HPACK算法进行压缩
- 多路复用：可以在一个tcp连接中并发请求多个http请求
- 服务器推送：在客户端请求后，服务端可以主动将客户端需要的一些其他资源一并推送给客户端，缓存到客户端本地

### http/3

http/2存在的问题：多个http请求复用同一个tcp连接，由于tcp的特性，当出现一个丢包时，需要等待该数据包的重传，那么其他的http都需要等待该数据包的重传完成。

- 协议变更UDP：底层的传输层协议使用UDP，UDP之上则依赖QUIC协议来保证可靠性等
- 更快的建立连接：在https建立连接过程时，需要先经历tcp的三次握手，然后是tls的三次握手，在QUIC中只需要三次握手即可建立连接

https://coolshell.cn/articles/19840.html

## https

**对称加密：**加密解密使用同一个密钥。常见的有：AES、ChaCha20

**非对称加密：**分为公钥和私钥两个不同的秘药，公钥加密的数据需要用私钥解密，私钥加密的数据需要用公钥解密。常见的有：RSA、ECC

**摘要算法：**对任意长度的数据生成固定长度的摘要字符串，用于做数据完整性校验。常见的算法有：MD5、SHA-1、SHA-2。通过摘要算法得出的摘要字符串，无法逆推出原文。

**数字签名：**对原文生成摘要，然后发送方使用私钥加密该摘要，生成数字签名。接收方收到数据后，对原文生成摘要，然后用发送方的公钥解数字签名得到发送方发的摘要，将两个摘要比较，以验证消息的真实性。

**数字证书和CA：**为了保证公钥的可信，通过CA来为各个公钥签名生成数字证书。CA使用自己的私钥为服务器的公加密生成数字证书，发送方将该数字证书发送给接收方，接收方收到后使用CA的公钥解密数字证书，得到发送方的公钥。低级的CA可以让更高级的CA为其签名认证，最终直到Root CA，我们必须信任Root CA，才能完成CA的整个信任链路。

### RSA算法握手过程

![](https://qraffa-1304595678.cos.ap-guangzhou.myqcloud.com/img/tls%25E6%258F%25A1%25E6%2589%258B.png)

**第一次握手：**

客户端向服务器发送client hello数据包。其中携带了TLS版本号，客户端此次选择的随机数，客户端支持的密码套件等信息。

![微信图片_20220711235807](https://qraffa-1304595678.cos.ap-guangzhou.myqcloud.com/img/%E5%BE%AE%E4%BF%A1%E5%9B%BE%E7%89%87_20220711235807.png)

**第二次握手：**

![image-20220708111714124](https://qraffa-1304595678.cos.ap-guangzhou.myqcloud.com/img/image-20220708111714124.png)服务端会通过一个TCP包发送三条消息，如图所示。

- Server Hello：与客户端的client hello类似，其中携带了TLS版本号，服务端此次选择的随机数，服务端最终选择的密码套件。

  ![image-20220708112118961](https://qraffa-1304595678.cos.ap-guangzhou.myqcloud.com/img/image-20220708112118961.png)

- Certificate：为了验证服务端的公钥的可信，服务端还将自己的证书一并发送给客户端

  ![image-20220708112230148](https://qraffa-1304595678.cos.ap-guangzhou.myqcloud.com/img/image-20220708112230148.png)

- Server Hello Done：表示服务端hello的信息都已经发送完毕了

  ![image-20220708112625767](https://qraffa-1304595678.cos.ap-guangzhou.myqcloud.com/img/image-20220708112625767.png)

**第三次握手：**

![image-20220708112802295](https://qraffa-1304595678.cos.ap-guangzhou.myqcloud.com/img/image-20220708112802295.png)

客户端会通过一个TCP包发送三条消息，如图所示。

- Client Key Exchange：客户端选择一个随机数 「PreMaster」，然后通过上一步服务端发来的证书中的服务器公钥进行加密，然后发送给服务端。服务端收到后通过自己的私钥揭秘得到「PreMaster」

  现在，服务端和客户端都知道了「Client Random」、「Server Random」、「PreMaster」，因此可以通过这三个数来生成此次会话的「Master Secret」

  > 主密钥生成以及使用：

  ![image-20220708112929039](https://qraffa-1304595678.cos.ap-guangzhou.myqcloud.com/img/image-20220708112929039.png)

- Change Cipher Spec：该消息表示之后的消息开始使用加密的方式

  ![image-20220708113308624](https://qraffa-1304595678.cos.ap-guangzhou.myqcloud.com/img/image-20220708113308624.png)

- Encrypted Handshake Message：为了防止中间的消息出现攻击篡改，客户端还会将自己之前收到和发送过的所有消息做个摘要，然后使用生成的主密钥加密，发送服务端做检查。

  ![image-20220708113342005](https://qraffa-1304595678.cos.ap-guangzhou.myqcloud.com/img/image-20220708113342005.png)

**第四次握手：**

第四次握手和第三次握手类似，由服务端来发送「Change Cipher Spec」、「Encrypted Handshake Message」，内容和作用原理同第三次握手。

![image-20220708115822626](https://qraffa-1304595678.cos.ap-guangzhou.myqcloud.com/img/image-20220708115822626.png)

至此，整个TLS握手连接完成，之后的通信都采用对称加密的方式，「Application Data」部分

![image-20220708120106224](https://qraffa-1304595678.cos.ap-guangzhou.myqcloud.com/img/image-20220708120106224.png)

### ECDHE算法握手过程

![](https://zq99299.github.io/note-book2/assets/img/69493b53f1b1d540acf886ebf021a26c.69493b53.png)

**第一次握手：**

客户端向服务器发送client hello数据包。其中携带了TLS版本号，客户端此次选择的随机数，客户端支持的密码套件等信息。

![image-20220708141913455](https://qraffa-1304595678.cos.ap-guangzhou.myqcloud.com/img/image-20220708141913455.png)

**第二次握手：**

服务端会通过一个TCP包发送三条消息，如图所示。

![image-20220708142051466](https://qraffa-1304595678.cos.ap-guangzhou.myqcloud.com/img/image-20220708142051466.png)

- Server Hello：与客户端的client hello类似，其中携带了TLS版本号，服务端此次选择的随机数，服务端最终选择的密码套件。

  ![image-20220708142223615](https://qraffa-1304595678.cos.ap-guangzhou.myqcloud.com/img/image-20220708142223615.png)

- Certificate：为了验证服务端的公钥的可信，服务端还将自己的证书一并发送给客户端

  ![image-20220708142324909](https://qraffa-1304595678.cos.ap-guangzhou.myqcloud.com/img/image-20220708142324909.png)

- Server Key Exchange：由于服务端选择的是ECDHE，所以这里还需要告诉客户端，选择的椭圆曲线比如这里是「x25519」，以及服务端生成随机数作为自己的椭圆曲线私钥，并用该私钥计算出椭圆曲线的公钥「Server Params」，并对这个公钥做个签名防止篡改

  ![image-20220708142555159](https://qraffa-1304595678.cos.ap-guangzhou.myqcloud.com/img/image-20220708142555159.png)

- Server Hello Done：表示服务端hello的信息都已经发送完毕了

  ![image-20220708142836859](https://qraffa-1304595678.cos.ap-guangzhou.myqcloud.com/img/image-20220708142836859.png)

**第三次握手：**

![image-20220708144222095](https://qraffa-1304595678.cos.ap-guangzhou.myqcloud.com/img/image-20220708144222095.png)

客户端会通过一个TCP包发送三条消息，如图所示。

- Client Key Exchange：客户端生成随机数作为自己的椭圆曲线私钥，再根据上一步服务端发来的相关参数，计算出椭圆曲线公钥「Client Params」，并将该公钥发送给服务端。

  此时服务端和客户端都有了「Server Params」和「Client Parmas」，椭圆曲线的公共参数，以及各自拥有的椭圆曲线私钥，因此可以算出「PreMaster」。并且，服务端和客户端都知道了「Client Random」、「Server Random」、「PreMaster」，因此可以通过这三个数来生成此次会话的「Master Secret」

  ![image-20220708144351073](https://qraffa-1304595678.cos.ap-guangzhou.myqcloud.com/img/image-20220708144351073.png)Change Cipher Spec：该消息表示之后的消息开始使用加密的方式

  ![image-20220708144408464](https://qraffa-1304595678.cos.ap-guangzhou.myqcloud.com/img/image-20220708144408464.png)

- Encrypted HandShake Message：为了防止中间的消息出现攻击篡改，客户端还会将自己之前收到和发送过的所有消息做个摘要，然后使用生成的主密钥加密，发送服务端做检查。

  ![image-20220708144432019](https://qraffa-1304595678.cos.ap-guangzhou.myqcloud.com/img/image-20220708144432019.png)

**第四次握手：**

第四次握手和第三次握手类似，由服务端来发送「Change Cipher Spec」、「Encrypted Handshake Message」，内容和作用原理同第三次握手。

![image-20220708155554172](https://qraffa-1304595678.cos.ap-guangzhou.myqcloud.com/img/image-20220708155554172.png)

至此，整个TLS握手连接完成，之后的通信都采用对称加密的方式，「Application Data」部分

![image-20220708155620546](https://qraffa-1304595678.cos.ap-guangzhou.myqcloud.com/img/image-20220708155620546.png)

### RSA与ECDHE

- RSA不具有前向安全，因为使用服务端的证书做「PreMaster」的加密解密，因此一旦服务端的私钥被破解或泄漏，那么就可以根据该私钥解出之前所有的密文。而ECDHE每次握手使用的公钥私钥都是临时生成的，因此保证了前向安全。

  > 前向保密，指的是长期使用的主[密钥](https://zh.m.wikipedia.org/wiki/密钥)泄漏不会导致过去的[会话密钥](https://zh.m.wikipedia.org/wiki/會話密鑰)泄漏。[[2\]](https://zh.m.wikipedia.org/zh-hans/前向保密#cite_note-Handbook-2)前向保密能够保护过去进行的通讯不受[密码](https://zh.m.wikipedia.org/wiki/密码)或[密钥](https://zh.m.wikipedia.org/wiki/密钥)在未来暴露的威胁。

### 会话复用

**SessionID**

客户端和服务端在建立TLS连接后，会保存一个会话ID，在服务端中保存这个会话ID对应的主密钥等相关信息。因此当下一次该客户端继续请求连接时，在「Client Hello」中会携带SessionID，服务端根据这个SessionID查找对应的主密钥等信息，如果能找到，则直接使用该信息恢复会话状态，省去了之前证书，密钥等过程。

SessionID存在的问题：

- 服务端需要为每个客户端保存密钥等信息，占用大量资源
- 使用负载均衡后，下一次相同的客户端连接，可能不是在同一台服务器上，仍然需要完整的TLS握手过程

![](https://qraffa-1304595678.cos.ap-guangzhou.myqcloud.com/img/session_resumption_with_session_id.jpg)

**Session Ticket**

与SessionID不同，Session Ticket是将密钥信息存储放到各个客户端上。当客户端和服务端建立TLS连接后，服务端将密钥等信息加密生成ticket发送给客户端，当下一次客户端继续请求连接时，会携带上该ticket，服务端对该ticket解密并验证有效性，如果有效则使用该信息恢复会话状态。

服务端需要使用固定的密钥文件来加密ticket，为了保证安全，需要定时更换密钥文件。

![](https://qraffa-1304595678.cos.ap-guangzhou.myqcloud.com/img/session_resumption_with_session_ticket.jpg)

 https://www.cnblogs.com/xiaolincoding/p/16129282.html
