## 网络编程

#### Http



##### Http1.0 Http1.1 Http2.0有什么区别 +2

```java
Http1.0与Http1.1区别
长连接：HTTP1.1默认开启keep-alive，在一个TCP连接上传送多个HTTP请求和响应，减少建立和关闭连接的消耗和延迟。
  
节约带宽：HTTP1.1支持只发送header信息（不带任何body信息），如果服务器认为客户端有权限请求服务器，则返回100，客户端接收到100才开始把请求body发送到服务器；如果返回401，客户端就可以不用发送请求body了节约了带宽。
  
Host域：
HTTP1.0中认为每台服务器都绑定一个唯一的IP地址，因此请求消息中的URL并没有传递主机名（hostname），HTTP1.0没有host域。
随着虚拟主机技术的发展，在一台物理服务器上可以存在多个虚拟主机，并且它们共享一个IP地址。HTTP1.1的请求消息和响应消息都支持host域，且请求消息中如果没有host域会报告一个错误（400 Bad Request）。

缓存处理：
HTTP1.0中主要使用header里的If-Modified-Since,Expires来做为缓存判断的标准。
HTTP1.1则引入了更多的缓存控制策略例如Entity tag，If-Unmodified-Since, If-Match, If-None-Match等更多可供选择的缓存头来控制缓存策略。

错误通知的管理：
HTTP1.1中新增了24个错误状态响应码，如409（Conflict）表示请求的资源与资源的当前状态发生冲突；410（Gone）表示服务器上的某个资源被永久性的删除。
  

Http2.0
多路复用：HTTP2.0使用了多路复用的技术，做到同一个连接并发处理多个请求。

头部数据压缩：
HTTP1.x，HTTP请求和响应都是由状态行、请求/响应头部、消息主体三部分组成。消息主体都会经过gzip压缩，或者本身传输的就是压缩过后的二进制文件，但状态行和头部却没有经过任何压缩，直接以纯文本传输。
HTTP2.0，使用HPACK算法对header的数据进行压缩，这样数据体积小了，在网络上传输就会更快。
  
服务器推送：
服务端推送是一种在客户端请求之前发送数据的机制。
网页使用了许多资源：HTML、样式表、脚本、图片等等。在HTTP1.1中这些资源每一个都必须明确地请求。这是一个很慢的过程。浏览器从获取HTML开始，然后在它解析和评估页面的时候，增量地获取更多的资源。因为服务器必须等待浏览器做每一个请求，网络经常是空闲的和未充分使用的。

为了改善延迟，HTTP2.0引入了server push，它允许服务端推送资源给浏览器，在浏览器明确地请求之前，免得客户端再次创建连接发送请求到服务器端获取。这样客户端可以直接从本地加载这些资源，不用再通过网络。
```





##### Http与Https的区别

```java
Http存在着这些缺点：
通信使用明文，内容可能被窃听(重要密码泄露)
不验证通信方身份，有可能遭遇伪装(跨站点请求伪造)
无法证明报文的完整性，有可能已遭篡改(运营商劫持)

Https是在http协议基础上加入 加密处理、认证机制、完整性保护。（http+加密+认证+完整性保护=https）
```



##### Http如何下载大文件？

```java
使用断点续传：
一个最简单的断点续传实现大概如下：
1.客户端下载一个1024K的文件，已经下载了其中512K。
2. 网络中断，客户端请求续传，因此需要在HTTP头中申明本次需要续传的片段：Range:bytes=512000-
这个头通知服务端从文件的512K位置开始传输文件。
3. 服务端收到断点续传请求，从文件的512K位置开始传输，并且在HTTP头中增加：Content-Range:bytes 512000-/1024000
并且此时服务端返回的HTTP状态码应该是206，而不是200。

但是在实际场景中，会出现一种情况，即在终端发起续传请求时，URL对应的文件内容在服务端已经发生变化，此时续传的数据肯定是错误的。如何解决这个问题了？显然此时我们需要有一个标识文件唯一性的方法。在RFC2616中也有相应的定义，比如实现Last-Modified来标识文件的最后修改时间，这样即可判断出续传文件时是否已经发生过改动。同时RFC2616中还定义有一个ETag的头，可以使用ETag头来放置文件的唯一标识，比如文件的MD5值。

终端在发起续传请求时应该在HTTP头中申明If-Match 或者If-Modified-Since 字段，帮助服务端判别文件变化。

另外RFC2616中同时定义有一个If-Range头，终端如果在续传是使用If-Range。If-Range中的内容可以为最初收到的ETag头或者是Last-Modfied中的最后修改时候。服务端在收到续传请求时，通过If-Range中的内容进行校验，校验一致时返回206的续传回应，不一致时服务端则返回200回应，回应的内容为新的文件的全部数据。


```



##### Http 怎么支持断点续传的？

```java
Http1.1协议中默认支持获取文件的部分内容，这其中主要是通过头部的两个参数：Range 和 Content Range 来实现的。客户端发请求时对应的是 Range ，服务器端响应时对应的是 Content-Range。
  
Range：
客户端想要获取文件的部分内容，那么它就需要请求头部中的 Range 参数中指定获取内容的起始字节的位置和终止字节的位置，它的格式一般为：
Range:(unit=first byte pos)-[last byte pos]
例如：
Range: bytes=0-499      表示第 0-499 字节范围的内容 
Range: bytes=500-999    表示第 500-999 字节范围的内容 
Range: bytes=-500       表示最后 500 字节的内容 
Range: bytes=500-       表示从第 500 字节开始到文件结束部分的内容 
Range: bytes=0-0,-1     表示第一个和最后一个字节 
Range: bytes=500-600,601-999 同时指定几个范围

Content Range：
在收到客户端中携带 Range 的请求后，服务器会在响应的头部中添加 Content Range 参数，返回可接受的文件字节范围及其文件的总大小。它的格式如下：
Content-Range: bytes (unit first byte pos) - [last byte pos]/[entity legth]
例如：
Content-Range: bytes 0-499/22400    // 0－499 是指当前发送的数据的范围，而 22400 则是文件的总大小。
  
使用断点续传和不使用断点续传的响应内容区别：
不使用断点续传：HTTP/1.1 200 Ok
使用断点续传：HTTP/1.1 206 Partial Content
  
处理请求资源发生改变的问题：
在现实的场景中，服务器中的文件是会有发生变化的情况的，那么我们发起续传的请求肯定是失败的，那么为了处理这种服务器文件资源发生改变的问题，在 RFC2616 中定义了 Last-Modified 和 Etag 来判断续传文件资源是否发生改变。
  Last-Modified & If-Modified-Since（文件最后修改时间）
  Last-Modified：记录 Http 页面最后修改时间的 Http 头部参数，Last-Modified 是由服务端发送给客户端的
  If-Modified-Since：记录 Http 页面最后修改时间的 Http 头部参数，If-Modified-Since 是有客户端发送给客户端的
```



##### http2的功能有哪些？

```java
HTTP/2通过以下方法减少延迟，用来改进页面加载的速度，
  HTTP Header的压缩，采用的是HPack算法。
  HTTP/2的Server Push，非常重要的一个特性。
  请求的pipeline。
  修复在HTTP 1.x的队头阻塞问题。
  在单个TCP连接里多工复用请求。
```



#### Https

##### https是如何加密的？+5

```java
对称加密：双方用同一个密钥进行加密解密。（效率高，但是涉及密钥传递）
非对称加密：公钥加密，私钥解密。公钥可以公开，私钥要保密。（不涉及密钥传递，但是效率不高）
数字证书认证：防止公钥被篡改，需要第三方机构来担保你的公钥的正确性。
	你需要把公钥给CA机构认证（CA机构会用它的公钥签名你的公钥），然后将认证的公钥返回给你，你就可以发布你的公钥了。
  浏览器或系统会内置CA机构的公钥，对你的公钥进行认证真伪，是真的就提取你的公钥。


TLS（SSL3.1就是TLS）是更为安全的升级版 SSL，ssl在应用层跟传输层之间）
tcp完成连接之后
客户端发送client hello （random_c + 一堆密码套件（假设x,y,z））
服务端发送server hello （random_s + 选了其中某个密码套件(假设选了x)）
服务端又发Server Key Exchange （服务端下发CA认证公钥）
服务端最后发送hello done

客户端验证发过来的服务端公钥
客户端根据 (random_c + random_s)x 生成 pre-master
客户端发送 (pre-master)(服务端公钥加密)

服务端收到(pre-master)(服务端公钥加密)后，用私钥解密得到 pre-master
服务端根据 (random_c + random_s)x 生成my_pre-master，pre-master 与 my_pre-master比对。

服务端发送 用pre-master加密的“Finished”消息。（服务端：我要确认客户端的密钥是不是对的）
客户端发送 也用pre-master加密的“Finished”消息。（客户端：我解出来了，服务端的密钥是对的。我要把这消息告诉服务端一声）
完成tls握手
```



#### TCP

##### 说说TCP和UDP特点与区别

```java
UDP
无连接的。尽最大可能交付，没有拥塞控制，面向报文（对于应用程序传下来的报文不合并也不拆分，只是添加 UDP 首部）。支持一对一、一对多、多对一和多对多的交互通信。
  
UDP 首部字段只有 8 个字节，包括源端口、目的端口、长度、检验和。12 字节的伪首部是为了计算检验和临时添加的。

  
TCP
面向连接的。提供可靠交付，有流量控制，拥塞控制，提供全双工通信，面向字节流（把应用层传下来的报文看成字节流，把字节流组织成大小不等的数据块）。
每一条 TCP 连接只能是点对点的（一对一）

TCP 首部格式比 UDP 复杂。
	序号：用于对字节流进行编号，例如序号为 301，表示第一个字节的编号为 301，如果携带的数据长度为 100 字节，那么下一个报文段的序号应为 401。
	确认号：期望收到的下一个报文段的序号。例如 B 正确收到 A 发送来的一个报文段，序号为 501，携带的数据长度为 200 字节，因此 B 期望下一个报文段的序号为 701，B 发送给 A 的确认报文段中确认号就为 701。
	数据偏移：指的是数据部分距离报文段起始处的偏移量，实际上指的是首部的长度。
	控制位：八位从左到右分别是 CWR，ECE，URG，ACK，PSH，RST，SYN，FIN。
	CWR：CWR 标志与后面的 ECE 标志都用于 IP 首部的 ECN 字段，ECE 标志为 1 时，则通知对方已将拥塞窗口缩小；
	ECE：若其值为 1 则会通知对方，从对方到这边的网络有阻塞。在收到数据包的 IP 首部中 ECN 为 1 时将 TCP 首部中的 ECE 设为 1；
	URG：该位设为 1，表示包中有需要紧急处理的数据，对于需要紧急处理的数据，与后面的紧急指针有关；
	ACK：该位设为 1，确认应答的字段有效，TCP规定除了最初建立连接时的 SYN 包之外该位必须设为 1；
	PSH：该位设为 1，表示需要将收到的数据立刻传给上层应用协议，若设为 0，则先将数据进行缓存；
	RST：该位设为 1，表示 TCP 连接出现异常必须强制断开连接；
	SYN：用于建立连接，该位设为 1，表示希望建立连接，并在其序列号的字段进行序列号初值设定；
	FIN：该位设为 1，表示今后不再有数据发送，希望断开连接。当通信结束希望断开连接时，通信双方的主机之间就可以相互交换 FIN 位置为 1 的 TCP 段。
	每个主机又对对方的 FIN 包进行确认应答之后可以断开连接。不过，主机收到 FIN 设置为 1 的 TCP 段之后不必马上回复一个 FIN 包，而是可以等到缓冲区中的所有数据都因为已成功发送而被自动删除之后再发 FIN 包；
	窗口：窗口值作为接收方让发送方设置其发送窗口的依据。之所以要有这个限制，是因为接收方的数据缓存空间是有限的。
```



##### TCP建立连接需要三次握手？

```java
目的：确保双方的收发能力

1、客户端 向 服务端 发送连接请求报文，SYN=1，ACK=0，选择一个初始的序号x。
2、服务端 收到连接请求报文，如果同意建立连接，则向 客户端 发送连接确认报文，SYN=1，ACK=1，确认号为 x+1，同时也选择一个初始的序号 y。
3、客户端 收到 服务端 的连接确认报文后，还要向 服务端 发出确认，确认号为 y+1，序号为 x+1。
服务端 收到 客户端 的确认后，连接建立。

客户端发，服务端收。-> 服务端结论：客户端发送功能正常、自己接收功能正常。
服务端发，客户端收。-> 客户端结论：自己发送功能正常、接收功能正常，服务端发送功能正常、接收功能正常。
客户端发，服务端收。-> 服务端结论：自己发送功能正常。
```



##### TCP关闭连接为什么是四次挥手？

```java
目的：确保双方都断开连接

每个方向都需要一个 FIN 和一个 ACK，因此通常被称为四次挥手
1、客户端打算关闭连接，此时会发送 FIN 报文，之后客户端进入 FIN_WAIT_1 状态。
2、服务端收到该报文后，就向客户端发送 ACK 应答报文，接着服务端进入 CLOSED_WAIT 状态。
客户端收到服务端的 ACK 应答报文后，之后进入 FIN_WAIT_2 状态。
3、等待服务端处理完数据后，也向客户端发送 FIN 报文，之后服务端进入 LAST_ACK 状态。
4、客户端收到服务端的 FIN 报文后，回一个 ACK 应答报文，之后进入 TIME_WAIT 状态。
服务器收到了 ACK 应答报文后，就进入了 CLOSE 状态，至此服务端已经完成连接的关闭。
客户端在经过 2MSL 时间后，自动进入 CLOSE 状态，至此客户端也完成连接的关闭。

客户端发，服务端收。-> 服务端知道客户端的关闭请求了。
服务端发，客户端收。-> 客户端知道发送的请求已经送达了。
但是服务端可能还有数据没发送完成，还不同意关闭。
服务端发，客户端收。-> 服务端处理完了，同意关闭了。
客户端发，服务端收。-> 通知服务端一声，自己收到服务端的同意指令了。
最后服务端关闭连接，客户端还会等待一段时间再执行关闭。
```



##### TCP在四次挥手之后为什么还要一个TIME_WAIT（等待2MSL的时间）？

```java
MSL是 Maximum Segment Lifetime，报文最大生存时间。（Linux里一个MSL是30秒。Linux系统停留在TIME_WAIT的时间为固定的60秒）

① 保证连接正确关闭。
TIME-WAIT作用是等待足够的时间以确保最后的ACK报文能让服务端接收到，从而帮助其正常关闭。
如果服务端等待一段时候没收到ACK，就会再发送一个FIN给客户端。

② 防止旧连接的数据包。
服务端有可能在关闭之前，给客户端发送数据。但是由于网络延迟，延时到达了。
这时有相同端口的tcp连接被复用后，被延迟的数据抵达了客户端，那么客户端是有可能接收这个过期数据的，这就会产生数据错乱等严重的问题。

  
过多的TIME_WAIT会照成什么影响？怎么优化？
① 内存资源占用；
② 对端口资源的占用，一个tcp连接消耗一个本地端口（端口资源是有限的）。

优化：
1、打开 net.ipv4.tcp_tw_reuse 和 net.ipv4.tcp_timestamps 选项；（复用处于 TIME_WAIT 的 socket 为新的连接所用。）
2、net.ipv4.tcp_max_tw_buckets（一旦超过这个值时，系统就会将所有的 TIME_WAIT 连接状态重置。好危险的！）
3、程序中使用 SO_LINGER ，应用强制使用 RST 关闭。（设置 socket 选项，来设置调用 close 关闭连接行为。如果l_onoff为非 0， 且l_linger值为 0，那么调用close后，会立该发送一个RST标志给对端，该 TCP 连接将跳过四次挥手，也就跳过了TIME_WAIT状态，直接关闭。）
```



##### TCP建立连接以后客户端突然故障了怎么办？

```java
TCP有一个心跳机制。
定义一个时间段，在这个时间段内，如果没有任何连接相关的活动，TCP保活机制会开始作用，每隔一段时间发送一个心跳报文，如果连续几个心跳报文都没有得到响应，则认为当前的tcp连接已经死亡，系统内核将错误信息通知给上层应用程序。
```



##### TCP短连接和长连接的区别？

```java
短连接：Client 向 Server 发送消息，Server 回应 Client，然后一次读写就完成了。这时候双方任何一个都可以发起 close 操作，不过一般都是 Client 先发起 close 操作。短连接一般只会在 Client/Server 间传递一次读写操作。
短连接的优点：管理起来比较简单，建立存在的连接都是有用的连接，不需要额外的控制手段。

长连接：Client 与 Server 完成一次读写之后，它们之间的连接并不会主动关闭，后续的读写操作会继续使用这个连接。
在长连接的应用场景下，Client 端一般不会主动关闭它们之间的连接，Client 与 Server 之间的连接如果一直不关闭的话，随着客户端连接越来越多，Server 压力也越来越大，这时候 Server 端需要采取一些策略，如关闭一些长时间没有读写事件发生的连接，这样可以避免一些恶意连接导致 Server 端服务受损；如果条件再允许可以以客户端为颗粒度，限制每个客户端的最大长连接数，从而避免某个客户端连累后端的服务。

长连接和短连接的产生在于 Client 和 Server 采取的关闭策略，具体的应用场景采用具体的策略。
```



##### TCP为什么会出现粘包、拆包？怎么解决？

```java
粘包：发送端发送了两个数据包，接收方一次性接收了两个包，这样两个包就粘在一起了，导致难以解析。
拆包：发送端发送了两个数据包，接收方分两次接收，第一次接收的是完成数据包1跟部分数据包2，第二次接收的是部分数据包2，导致难以解析。

出现原因：
1、要发送的数据大于 TCP 发送缓冲区剩余空间大小，将会发生拆包。
2、待发送数据大于 MSS（最大报文长度），TCP 在传输前将进行拆包。
3、要发送的数据小于 TCP 发送缓冲区的大小，TCP 将多次写入缓冲区的数据一次发送出去，将会发生粘包。
4、接收数据端的应用层没有及时读取接收缓冲区中的数据，将发生粘包。

解决方法：
由于TCP本身是面向字节流的，无法理解上层的业务数据，所以在底层是无法保证数据包不被拆分和重组的，这个问题只能通过上层的应用协议栈设计来解决。
1、消息定长：发送端将每个数据包封装为固定长度（不够的可以通过补 0填充），这样接收端每次接收缓冲区中读取固定长度的数据就自然而然的把每个数据包拆分开来。
2、设置消息边界：服务端从网络流中按消息边界分离出消息内容。在包尾增加回车换行符进行分割，例如 FTP 协议。
3、将消息分为消息头和消息体：消息头中包含表示消息总长度（或者消息体长度）的字段。
```



##### 说说TCP的流量控制

##### 说说TCP滑动窗口机制（问流量控制都是想问滑动窗口的实现）

```java
TCP是面向连接的可靠的传输协议，一个可靠的传输协议就需要对数据进行确认。
TCP也是双工的协议，会话的双方都可以同时接收和发送数据。TCP会话的双方都各自维护一个发送窗口和一个接收窗口。各自的接收窗口大小取决于应用、系统、硬件的限制（TCP传输速率不能大于应用的数据处理速率）。各自的发送窗口则要求取决于对端通告的接收窗口，要求相同。

如果发送方和接收方对数据包的处理速度不同，如果发送速度太快，接收方来不及处理就可能overflow。
如何让双方达成收发一致呢？流量控制。TCP使用窗口机制来实现流量控制。

TCP协议里窗口机制有2种：固定窗口；滑动窗口。

固定窗口：吞吐量非常的低。
发送方发送一个包1，接收方收到包1，然后给发送方回复一个包1已收到的消息。
发送包2，确认包2。
如果发送方迟迟没有收到回复消息，就会重新发送。
就这样一直下去，直到把数据完全发送完毕。

滑动窗口：高吞吐量，解决固定窗口的吞吐量低问题。
第一次发送数据这个时候的窗口大小是根据链路带宽的大小来决定的，我们假设窗口的大小是3。
发送方：发送包1，2，3
接收方：收到包1，2，丢了包3，回复[ACK 3, window size 2]，窗口右滑到3的位置，因为已经收到1，2的包了
发送方：窗口右滑动到3的位置因为1，2包已经确认收到了。发送包3，4（接收方窗口大小为2只发两条，并且再请求3再发一次）
接收方：收到包3，4，回复[ACK 5, window size 2]，窗口右滑到5的位置
发送方：那么久没回复，我重新发包3，4
接收方：怎么又是包3，4。我不要，回复[ACK 5, window size 5]（我缓存区有更多空间了，多给我点数据包）
发送方：终于收到回复了，窗口右滑动到5位置。要包5是吧，窗口还变大了。发送包5，6，7，8，9
...


生动的例子：
老师说一段话, 学生来记. （不可靠通讯）
老师说"从前有个人, 她叫马冬梅. 她喜欢他, 而他却喜欢她."
学生写道"从前有..". "老师你说的太快我跟不上"

于是他们换了一种方案.（固定窗口，吞吐量低）
老师说"从"
学生写"从". 学生说"嗯"
老师说"前"
学生写"前". 学生说"嗯"
老师说"今天我还想早点下班呢..."

于是他们又换了一种方案.（高吞吐，但会丢包）
老师说"从前有个人"
学生写"从前有个人". 学生说"嗯"
老师说"她叫马冬梅". 
学生写"她叫马...梅". 学生说"马什么梅?"
老师说"她叫马冬梅". 
学生写"她叫马冬...". 学生说"马冬什么?"
老师"....."
学生说"有的时候状态好我能把5个字都记下来, 有的时候状态不好就记不下来. 我状态不好的时候你能不能慢一点. "

他们换了最后一种方案（滑动窗口）
老师说"从前有个人"
学生写"从前有个人". 学生说"嗯, 再来5个"
老师说"她叫马冬梅"
学生写"她叫马..梅". 学生说"啥?重来, 来2个"
老师说"她叫"
学生写"她叫". 学生说"嗯,再来3个"
老师说"马冬梅". 
学生写"马冬梅". 学生说"嗯, 给我来10个"
老师说"她喜欢他,而他却喜欢她"
学生写...

所以呢
第一种模式简单粗暴, 发的只管发, 收的更不上.
第二种模式稳定却低效, 每发一个, 必须等到确认才再次发送, 等待时间很多.
第四中模式才是起到了流控的作用, 接收方认为状态好的时候, 让发送方每次多发一点. 接收方认为状态不好的时候(阻塞), 让发送方每次少发送一点.
```



##### 说说TCP的拥塞控制

```java
网络出现拥塞，将会出现丢包的情况。此时发送方会继续重传，从而导致网络拥塞程度更高。
因此当出现拥塞时，应当控制发送方的速率。这一点和流量控制很像，但是出发点不同。
流量控制是为了让接收方能来得及接收，而拥塞控制是为了降低整个网络的拥塞程度。

发送方需要维护一个叫做拥塞窗口（cwnd）的状态变量，注意拥塞窗口与发送方窗口的区别：拥塞窗口只是一个状态变量，实际决定发送方能发送多少数据的是发送方窗口。

慢开始与拥塞避免
发送的最初执行慢开始，令 cwnd = 1，发送方只能发送 1 个报文段；当收到确认后，将 cwnd 加倍，因此之后发送方能够发送的报文段数量为：2、4、8 ...
注意到慢开始每个轮次都将 cwnd 加倍，这样会让 cwnd 增长速度非常快，从而使得发送方发送的速度增长速度过快，网络拥塞的可能性也就更高。设置一个慢开始门限 ssthresh，当 cwnd >= ssthresh 时，进入拥塞避免，每个轮次只将 cwnd 加 1。
如果出现了超时，则令 ssthresh = cwnd / 2，然后重新执行慢开始。

快重传与快恢复
在接收方，要求每次接收到报文段都应该对最后一个已收到的有序报文段进行确认。例如已经接收到 M1 和 M2，此时收到 M4，应当发送对 M2 的确认。
在发送方，如果收到三个重复确认，那么可以知道下一个报文段丢失，此时执行快重传，立即重传下一个报文段。例如收到三个 M2，则 M3 丢失，立即重传 M3。
在这种情况下，只是丢失个别报文段，而不是网络拥塞。因此执行快恢复，令 ssthresh = cwnd / 2 ，cwnd = ssthresh，注意到此时直接进入拥塞避免。
慢开始和快恢复的快慢指的是 cwnd 的设定值，而不是 cwnd 的增长速率。慢开始 cwnd 设定为 1，而快恢复 cwnd 设定为 ssthresh。
```



##### 如果提高TCP的网络利用率

```java
1、Nagle 算法
发送端即使还有应该发送的数据，但如果这部分数据很少的话，则进行延迟发送的一种处理机制。具体来说，就是仅在下列任意一种条件下才能发送数据。如果两个条件都不满足，那么暂时等待一段时间以后再进行数据发送。

已发送的数据都已经收到确认应答。
可以发送最大段长度的数据时。

2、延迟确认应答
接收方收到数据之后可以并不立即返回确认应答，而是延迟一段时间的机制。

在没有收到 2*最大段长度的数据为止不做确认应答。
其他情况下，最大延迟 0.5秒 发送确认应答。
TCP 文件传输中，大多数是每两个数据段返回一次确认应答。

3、捎带应答
在一个 TCP 包中既发送数据又发送确认应答的一种机制，由此，网络利用率会提高，计算机的负荷也会减轻，但是这种应答必须等到应用处理完数据并将作为回执的数据返回为止。
```



#### 其他

##### 域名遭到劫持的解决方案。

##### 如何防抓包。



## 加密

##### 什么是对称加密

```java
通信双⽅使⽤同⼀个密钥，使⽤加密算法配合上密钥来加密，解密时使⽤加密过程的完全逆过程配合密钥来进⾏解密。

应用：
  DES（56 位密钥，密钥太短⽽逐渐被弃⽤）、AES（128 位、192 位、256 位密钥，现在最流⾏）
  
优点：
  使用简单、效率高。
缺点：
  不能在不安全⽹络上传输密钥，⼀旦密钥泄露则加密通信失败。
  
破解思路：
  拿到⼀组或多组原⽂-密⽂对，设法找到⼀个密钥，这个密钥可以将这些原⽂-密⽂对中的原⽂加密为密⽂，以
  及将密⽂解密为原⽂的组合，即为成功破解。
```



##### 什么是非对称加密

```java
使⽤公钥对数据进⾏加密得到密⽂，使⽤私钥对数据进⾏解密得到原数据。

应用：
  数字签名技术：原数据进行hash后将hash值加密，原数据+加密hash传输。
  					  对方拿到数据后，解密hash并且自己也对原数据进行hash，两个hash比对，完成签名校验。
  经典的数据签名有：RSA（可⽤于加密和签名）、DSA（仅⽤于签名，但速度更快）
  
优点：
  可以在不安全⽹络上传输密钥。
缺点：
  计算复杂，因此性能相⽐对称加密差很多。
```



##### 什么是hash，hash有什么用途？

```java
定义：把任意数据转换成指定⼤⼩范围（通常很⼩，例如 256 字节以内）的数据。
作用：相当于从数据中提出摘要信息，因此最主要⽤途是数字指纹。

用途：
  唯⼀性验证：例如 Java 中的 hashCode() ⽅法。
  数据完整性验证：通过⽐对⽂件的 Hash 值（例如 MD5、SHA1）
  快速查找：HashMap
  隐私保护：当重要数据必须暴露的时候，例如⽹站登录时。可以只保存⽤户密码的Hash值，在每次登录验
					证时只需要将输⼊的密码的Hash值和数据库中保存的Hash值作⽐对就好
  
注意：
  hash不是编码。
  hash不是加密。
 	hash是单向过程往往是不可逆的，⽆法进⾏逆向恢复操作，不属于编码也不属于加密。
 （记住，MD5 不是加密！）
```



