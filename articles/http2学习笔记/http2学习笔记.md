# HTTP2学习笔记
## 一、简介
HTTP1.1已经广泛应用多年。然而，HTTP1.1本身简单的机制却导致了网络资源利用率较低的问题，在实际应用中性能受限。   

因此，大佬们引入了HTTP2改善性能方面的问题。

HTTP2不会改动HTTP方法、状态码、URI及首部字段等核心概念。只会修改数据实际传输的方式，例如增加了分帧、流等概念。而这些底层的传输方式对于应用来说都是隐藏的。也就是说我们使用HTTP1.1的应用无需修改过多就可以改用HTTP2。

## 二、帧
HTTP2使用了新的报文结构，如下图所示（图来自RFC 7540）：
![](~/Desktop/学习笔记/http2学习笔记/frame structure.png)

在HTTP2中，一个TCP连接上可有多个并行流，一个流中可传输多个帧。这与HTTP1.1有明显的区别。   

相比于HTTP1.1的报文，帧是更小的粒度单位，更加的灵活，便于更好的利用网络资源。

### 首部压缩
帧中也引入了首部压缩算法，减少数据传输量，以提升传输效率。   

在传输时，使用头部压缩算法将头部列表串行化为一个头部块。该块之后被分为一个或多个字节序列，被称为头部块分组。然后再将该分组作为HEADERS、PUSH_PROMISE、CONTINUATION的payload。   

接收者连接所有分组，然后解密获取头部列表。

在HTTP2中首部压缩使用HPACK算法，该算法可显著减少头部流量。

#### HPACK算法
压缩算法主要分为以下三个部分：   
+ 静态字典：存储多个常用的头部字段。
+ 动态字典：在传输过程中动态增加头部，有数量限制。
+ 哈夫曼编码：对字符串进行编码，可缩短长度。

HPACK通过对头部进行编码来达到压缩的效果。   

例如HPACK需要为某头部编码，他首先查找静态与动态字典，若该头部完整存在，则直接从字典去除该项的编码即可，只需要一个字节即可存储该头部。   

若该头部不完整存在，则HPACK尝试寻找拥有相同键的头部并获取其编码，于是一个键就被压缩成了一个字节。在这种情况下由于值并不重复，因此没有进行压缩。   

若键未找到，则直接使用哈夫曼算法进行压缩。

#### 分组
+ 单个分组：单个HEADERS与PUSH_PROMISE帧，含有END_HEADERS flag。
+ 多个分组：单个HEADERS与PUSH_PROMISE帧，无END_HEADERS flag。并且有多个CONTINUATION帧，最后一个CONTINUATION帧拥有END_HEADERS flag。

首部压缩是有状态的，加密与解密使用同一个上下文。

## 三、流
### 多路复用
帧的Stream Identifier字段标识了该帧属于哪个流，并且奇数指客户端初始化的流，偶数指服务器初始化的流。0x0标识控制信息的流。   

一个TCP连接中可有多个流，流是互相独立的，每个流中有一系列帧在发送者和接收者间传输。并行独立的流即可实现多路复用。   

同时，流的引入可以很好的解决队首阻塞问题。   
  
在http1.1中，对于同一个tcp连接、允许一次发送多个http请求，也就是说，不必等待前一个请求的响应收到，就可以发送下一个请求。  

但是，http1.1规定，服务器端的响应的发送要根据请求被接收的顺序排队，先接收到的请求的响应也要先发送。   

这样造成的问题是，如果最先收到的请求的处理时间长的话，响应生成也慢，就会阻塞已经生成了的响应的发送。于是便造成队首阻塞。   

而在HTTP2中，由于流是并行的，因此不会造成队首阻塞问题。

### 流控制
HTTP2使用WINDOW_UPDATE帧来进行流控制，动态调节window大小，限制传输帧的数量。接收者可使用该帧设定某个流的window和整个连接的window。   

只有DATA帧会占用window，其他帧不会占用。这样可防止重要数据被流控制妨碍，如控制信息等。
   
流控制基于每一跳进行，而不是端到端（指代理的情况，可参见参考资料）。  
     
### 流优先级
流的优先级决定了资源分配的顺序。   

在实际应用中，我们往往需要优先获取html和css文件，减少页面白屏时间。流的优先级能够很好的解决这个问题。   

http2采用了基于依赖的优先级算法。

#### 流依赖性
一个流可以依赖另一个流，根据他们的依赖关系我们可以生成一颗依赖树，如下图所示：   
![](~/Desktop/学习笔记/http2学习笔记/dependency.png)

其中B、C流依赖D流，而D流依赖A流。

在依赖树中，流只能在其依赖的所有流关闭或阻塞时才能获取网络资源进行传输。   

如在上例中D流只能在A流关闭或阻塞时才能进行传输。若上例中的A流传输html文件、D流传输css文件、B与C流传输JavaScript文件则可以满足优先获取html与css的需求。    

此外，依赖统一流的各个流可设置依赖权重（1-256的证书），按比例分配网络资源。

#### 优先级设置
可通过HEADERS帧初始化流的优先权，PRIORITY帧可在任何时候修改流的优先权。 


## 四、常见帧列表
### DATA
用于传输数据
### HEADERS
用于开启一个新的流，同时传输请求的header
### PRIORITY
用于更改流的优先权，可在任何状态下发送
### RST_STREAM
在发生错误时发送，能够立刻中止流
### SETTINGS
用于终端间交流的配置参数。   
将在流初始化时发送，同时也可能在其他时候发送。
### PUSH_PROMISE
用于在流初始化前告知终端
### PING
用于探测最小RTT
### GOAWAY
用于报错与中止流
### WINDOW_UPDATE
用于修改流控制中的window
### CONTINUATION
用于发送更多的HEADERS，最后一个该帧需要END_HEADERS flag

## 五、实例分析

## 参考资料
+ RFC 2616 (HTTP1.1):   
https://tools.ietf.org/html/rfc2616
+ RFC 7540、RFC 7541 (HTTP2):   
https://httpwg.org/specs/rfc7540.html    
https://httpwg.org/specs/rfc7541.html
+ HTTP2抓包：  
https://imququ.com/post/http2-traffic-in-wireshark.html
+ 测试网站：   
https://www.imydl.tech/
+ 关于流控制跳的解释：   
https://stackoverflow.com/questions/40747040/how-is-http-2-hop-by-hop-flow-control-accomplished
+ 优先级依赖工作：  
https://nghttp2.org/blog/2014/04/27/how-dependency-based-prioritization-works/
+ HPACK详解：  
https://www.zcfy.cc/article/hpack-the-silent-killer-feature-of-http-2-1969.html