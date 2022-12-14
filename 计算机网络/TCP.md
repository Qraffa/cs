## TCP

![计算机网络 (1)](https://qraffa-1304595678.cos.ap-guangzhou.myqcloud.com/img/%E8%AE%A1%E7%AE%97%E6%9C%BA%E7%BD%91%E7%BB%9C%20(1).png)

> TCP/IP卷一读书笔记

### 第12章 TCP

处理通信中信息差错的方法：

- 差错校验码
- 自动重复请求

通信时可能遇到的差错类型：

- 分组重排
- 分组重复
- 分组丢失

**分组丢失**

解决方法：重发分组直到它被正确接收。

> 如何判断已经被正确接收？

基于以下两个判断：

1. 接收方是否已经收到分组？
2. 接收方接收到的分组是否和发送方发送的是一致的？

接收方通过给发送方发送信号以确定自己已经收到正确的分组。发送方发送一个分组，然后等待接收方回复一个ACK，当接收方接收到该分组后，发送对应的ACK。

> 使用ACK方法需要回答以下问题：
>
> 1. 发送方对一个ACK应该等待多久？
> 2. 接收方发送的ACK丢失怎么办？
> 3. 接收到分组，但存在错误怎么办？

ACK等待多久：// TODO

ACK丢失：当接收方的ACK丢失后，发送方由于收不到ACK，因此再次重发该分组即可。那么接收方就可能收到重复的分组。

分组错误：通过使用校验和的方法来检测分组的差错。当接收方收到含有差错的分组时，不发送ACK，等待发送方重发。

**分组重复**

发送方为每个唯一的分组确定一个序列号，该序列号由分组自身一直携带。接收方可以根据该序列号来判断是否已经正确收到该分组，如果已经收到则可以直接丢弃。

#### TCP

TCP提供一种**面向连接**的、**可靠**的**字节流**服务。

面向连接：使用TCP传输数据之前，需要先建立一个TCP连接

可靠：TCP通过校验和、序列号、超时重传、滑动窗口、流量窗口等机制来保证可靠性

字节流：UDP是面向报文的，即应用层交付多长报文，UDP照样发送。TCP面向字节流，即TCP将应用层交付的数据看成无结构的字节流，当发送时，会维护缓冲区，如果交付的数据太长，则划分为多段多次发送；如果交付数据太短，则等待累计到足够的字节再发送。由于TCP会拆分或累计，则会导致数据是没有边界的，当缓冲区够大，可能一次性收到发送方发送的多段数据，这也是TCP拆包和粘包出现的原因

> TCP格式

TCP在IP数据报中的封装：

![image-20210323005125703](https://qraffa-1304595678.cos.ap-guangzhou.myqcloud.com/img/image-20210323005125703.png)

TCP头部格式：

![image-20210323005255380](https://qraffa-1304595678.cos.ap-guangzhou.myqcloud.com/img/image-20210323005255380.png)

### 第13章 TCP连接管理

#### 连接建立

连接建立三次握手

![tcp-1](https://qraffa-1304595678.cos.ap-guangzhou.myqcloud.com/img/tcp-1.jpeg)

##### 三次握手的原因

- 确认自己和对方的发送和接收能力正常
- 阻止重复历史连接的初始化
- 同步双方的序列号
- 避免资源浪费

**确认自己和对方的发送和接收能力正常**

使用三次握手的目的是为了建立可靠的通信连接,需要确认自己和对方的发送和接受能力正常

第一次握手:客户端无法确认信息,服务端接受到报文,可以确认自己的接受能力和对方的发送能力正常

第二次握手:客户端收到确认报文,因此确认自己的发送和接受能力正常,以及对方的发送和接受能力正常

第三次握手:服务端收到确认报文,确认对方的接受能力和自己的发送能力正常

|      | 客户端   |          |          |          | 服务端   |          |          |          |
| ---- | -------- | -------- | -------- | -------- | -------- | -------- | -------- | -------- |
|      | 自己接受 | 自己发送 | 对方接受 | 对方发送 | 自己接受 | 自己发送 | 对方接受 | 对方发送 |
| 一次 | ×        | ×        | ×        | ×        | √        | ×        | ×        | √        |
| 二次 | √        | √        | √        | √        | √        | ×        | ×        | √        |
| 三次 | √        | √        | √        | √        | √        | √        | √        | √        |

**阻止重复历史连接的初始化**

```haskell
	TCP A                                                TCP B

  1.  CLOSED                                               LISTEN

  2.  SYN-SENT    --> <SEQ=100><CTL=SYN>               ...

  3.  (duplicate) ... <SEQ=90><CTL=SYN>               --> SYN-RECEIVED

  4.  SYN-SENT    <-- <SEQ=300><ACK=91><CTL=SYN,ACK>  <-- SYN-RECEIVED

  5.  SYN-SENT    --> <SEQ=91><CTL=RST>               --> LISTEN


  6.              ... <SEQ=100><CTL=SYN>               --> SYN-RECEIVED

  7.  SYN-SENT    <-- <SEQ=400><ACK=101><CTL=SYN,ACK>  <-- SYN-RECEIVED

  8.  ESTABLISHED --> <SEQ=101><ACK=401><CTL=ACK>      --> ESTABLISHED
```

<SEQ=90><CTL=SYN>是一个重复历史连接请求，由于TCP B是无法分辨这是重复历史连接请求，因此还是正常返回。当TCP A收到<SEQ=300><ACK=91><CTL=SYN,ACK>后发现ACK字段不正确，因为自己要的是101，因此发送RST，将该历史重复连接给重置掉了。后面<SEQ=100><CTL=SYN>的连接请求正常。

如果使用二次握手，那么TCP B收到重复历史连接请求后，直接就进入ESTABLISHED状态，开始发送数据了，TCP A等待接收到TCP B发送的数据或者ACK，才会发送RST重置，那么这里就浪费了连接资源和数据资源的无效传输。

**避免资源浪费**

假设客户端发出的第一个连接请求在网络中延迟了.客户端选择**重发**连接请求,该连接请求被确认,双方建立连接,然后完成数据传输后释放连接.此时第一个延迟的连接请求到达服务端,服务端又建立一次连接,向客户端发出确认报文,客户端丢弃该报文,服务端一直在等待客户端发送数据,浪费资源.

而在三次握手的情况下,第一个延迟的报文在服务端发送确认报文,等待客户端报文时,因为一直等不到会认为这次连接请求取消.

##### 初始序列号随机选择

- 防止出现与其他连接序列号重叠的情况
- 防止历史报文被下一个相同四元组建立的连接接收处理

由于TCP由包含2个IP地址和2个端口号的四元组组成，因此如果前一个连接由于某个报文长时间延迟导致连接被重置了，那么接下来以相同的四元组重新建立连接，该历史报文可能被正常接收处理。因此使用随机的初始序列号后，并且随机选择的序列号是递增的，那么历史报文将不会被接收处理。

#### 连接断开

连接断开四次挥手

![img](http://mianbaoban-assets.oss-cn-shenzhen.aliyuncs.com/xinyu-images/MBXY-CR-3a354b2083e45e590f4de1f00a0b9b39.png)

##### 四次挥手的原因

主要在于服务端接受到客户端的断开连接请求后,先发送一个ACK确认报文,因为服务端可能还有数据需要发送,当书服务没有数据需要发送时,才发送FIN报文来断开服务端到客户端的连接

##### TIME_WAIT状态

处于TIME_WAIT状态时，TCP会等待2*MSL(最大报文段存活时间)，当该计时器超时后进入CLOSED状态，连接关闭。

原因：

- 保证被动关闭一方能收到最后的ACK正确关闭

- 有足够的时间来保证本次连接中的所有报文都在网络中消失,不会影响后面新的连接

主要为了避免四次挥手中最后一个ACK报文丢失的情况，当最后一个ACK报文丢失时，被动关闭一方由于未收到其发送的FIN的ACK，因此其会选择重发FIN，那么处于TIME_WAIT状态下的端收到后会对该FIN再次ACK，并重置计时器。因此一个ACK报文和一个FIN报文的最大存活时间就是等待的时间。

### 第14章 TCP超时与重传

当数据段或确认信息丢失后，TCP启动重传操作，重传尚未确认的数据。通过以下两种方式：

- 重传计时器：当发送一个数据段后启动重传计时器，当计时器超时仍未收到ACK，则认为超时，重传该数据段
- 快速重传：TCP累积确认没有收到新的ACK或SACK中的选择确认信息表明出现了失序的报文段，则推断出需要重传的报文。

#### 超时重传

TCP需要根据网络环境推断RTT(round-trip time，报文往返时延)来动态设置RTO(重传超时)。

**超时重传计算**

参考[RFC6298](https://datatracker.ietf.org/doc/html/rfc6298)

```haskell
1. 首次测量计算，R为首次测量RTT值
SRTT=R
RTTVAR=R/2
RTO=SRTT+K*RTTVAR

2. 后续测量计算
SRTT=(1-α)*SRTT+α*R'
RTTVAR=(1-β)*RTTVAR+β*|SRTT-R'|
RTO=SRTT+K*RTTVAR

其中K=4,α=1/8,β=1/4
```

#### 重传二义性

重传二义性问题：假定一个分组被发送，在重传时间内该分组的确认未收到，因此发送方重传，然后收到一个该分组的确认，那么如何判断该确认报文是第一个分组还是第二个分组的确认？

karn算法：

- 当出现超时重传时，接收到重传数据的ACK信息不能用于计算RTO
- 当再次出现超时重传时，将RTO翻倍

#### 快速重传

当接收端收到失序的报文段时，需要立即生成重复的ACK确认报文；当接收端收到一定数量（通常是3个）的重复ACK后，即重传需要重传的数据，而不必等待重传超时器的超时。

当第一次快重传发生时，发送端已经发送了更多的数据，因此记录之前发送过的最大的序列号作为恢复点；当发送端收到大于或等于恢复点的ACK时，则可以认为快重传完成，因而从快重传恢复。

### 第15章 TCP数据流与窗口管理

#### Nagle算法

```haskell
if there is new data to send then
    if the window size ≥ MSS and available data is ≥ MSS then
        send complete MSS segment now
    else
        if there is unconfirmed data still in the pipe then
            enqueue data in the buffer until an acknowledge is received
        else
            send data immediately
        end if
    end if
end if
```

- 如果窗口大小>=MSS或者数据大小>=MSS，则直接发送MSS大小的数据
- 如果TCP中存在已经发送但未收到确认的数据，则小的报文段需要等待

#### 滑动窗口

TCP协议中通过头部的`窗口大小`字段，表示发送该报文的一端的目前接收窗口可用大小，单位为字节。

由于TCP是全双工通信，两端都可以发送和接收数据，因此两端都需要为各自的发送窗口和接收窗口。

![img](https://qraffa-1304595678.cos.ap-guangzhou.myqcloud.com/img/tcpswflow.png)

##### 零窗口探测

当TCP接收端应用层处理不及时，可能导致TCP接收端的窗口降到0，从而阻止发送端发送数据，直到有新的可用窗口空间。当有新的空间时，接收端会向发送端发送ACK，表示窗口更新，期望继续发送数据，但是这样不带数据的ACK包没有重传机制，即可能在网络中丢失。因此双端都会陷入等待状态：发送方等待接收方通知非0的窗口大小，接收方等待发送方发送数据。

针对上述问题，发送方会使用持续计时器，当计时器超时，触发窗口探测的传输，要求接收端返回ACK，并通告当前的窗口大小。在窗口探测包中携带一个字节的数据，因此采用可靠传输，避免更新窗口的ACK数据包丢失导致的死锁等待。

##### 糊涂窗口综合征

使用滑动窗口时，可能出现窗口比较小的情况，此时双方交换的数据段大小是一些比较小的，这样将使得网络带宽利用率降低。

产生的原因：

- 接收端通告的窗口较小
- 发送端发送的数据段较小

解决办法：

- 对于接收方，当窗口小于MSS或者缓存空间的1/2时，直接返回0窗口大小
- 对于发送方，使用Nagle算法

### 第16章 TCP拥塞控制

拥塞控制：用于防止网络因为大规模的通信负载造成网络瘫痪。基本方法是当认为网络将出现拥塞或者已经因为网络拥塞出现丢包时，降低TCP的传输速率。

拥塞检测：基本方法是看有没有丢包发生。

拥塞窗口(cwnd)用于反映实际网络的传输能力，则发送端的可用窗口W=min(awnd, cwnd)

#### 拥塞控制基本算法

##### 慢开始

描述：初始阶段选择初始窗口(一般为1)，每收到一个发送端回传的ACK，那么窗口大小增加1，因此k轮之后，窗口大小为2^k，窗口大小呈指数增长。

目的：在传输初始阶段，由于不知道网络的传输能力，因此缓慢的增长传输速率，避免突然大量的数据使得网络瘫痪

执行时机：

- 一个新的TCP连接建立
- 出现超时重传丢包

##### 拥塞避免

描述：每收到一个发送端回传的ACK，那么窗口大小增加1/cwnd，窗口大小呈线性增长。

目的：当cwnd持续指数增长时，大量数据包的发送将导致网络瘫痪，因此降低传输速率，。

执行时机：当cwnd>=ssthresh(slow start threshold)，执行拥塞避免

##### 拥塞发生

分两种情况：

一：超时重传

ssthresh=cwnd/2

cwnd=1

使用慢启动算法

二：快重传

ssthresh=cwnd/2

cwnd=ssthresh+3

使用快恢复算法

##### 快恢复

描述：每收到一个重复ACK，则cwnd+1。当收到正常的ACK时，cwnd=ssthresh，进入拥塞避免算法。