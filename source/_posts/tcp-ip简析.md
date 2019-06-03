---
title: tcp简析
date: 2019-05-03 12:43:02
tags: 
- tcp
categories: 计算机网络
---



### OSI七层模型与TCP/IP五层模型
  OSI（Open System Interconnect），即开放式系统互联模型是ISO（国际标准化组织）组织在1985年发布的网络模型,其结构如下:

![image-20190503131546100](https://gitee.com/neusnail/img/raw/master/blogimg/image-20190503131546100.png)

其中的表示层负责提供各种用于应用层数据的编码和转换功能, 会话层负责建立、管理和终止表示层实体之间的通信会话. 这两层在TCP/IP模型中整合到了应用层.所以我们常用的TCP/IP模型一般只讨论5层:

![image-20190503131508854](https://gitee.com/neusnail/img/raw/master/blogimg/image-20190503131508854.png)

可以看到从应用层发出的数据,每经过一层都会添加上该层特有的header,终端收到数据包之后又会一层层解除头部拿到原始数据:
> 图片摘自阮一峰的博客http://www.ruanyifeng.com/blog/2017/06/tcp-protocol.html

![net协议](https://gitee.com/neusnail/img/raw/master/blogimg/net协议.png)



#### 物理层

物理层， 为数据端设备提供传送数据的通路。可以双向通信，为数据传输提供可靠的环境。对应我们的网线、光纤(物理设备)。

这一层不详细展开了,只说下物理层通信原理的一个重要概念:

* 单工：单向通道，只能单向通信。比如广播。

- 半双工：双向通道,但同一时间点只能一方通信。比如对讲机。
- 全双工：双向通道，相互通信。互不干扰。比如：电话。

####  数据链路层

数据链路层在两个网络实体之间提供数据链路连接的创建、维持和释放管理。

数据链路层只负责数据的封装成祯和透明传输,不保证祯的可靠性,可靠性是tcp协议实现的

数据链路层职责:

* 封装成帧

  将网络层交付下来的ip数据报的前后添加首部和尾部，封装成帧。首部和尾部的重要作用是进行帧定界。为了提高传输效率，需要尽量增大数据部分的长度，但是每个链路层协议都规定了帧数据部分的长度上限，最大传输单元MTU(Maximum Transfer Unit)。当数据是可打印的ASCII码，帧定界可以用特殊的帧定界符SOH(Start of Header)(00000001)和EOT(End of Transmission)(00000100)

  ![image-20190503134026958](https://gitee.com/neusnail/img/raw/master/blogimg/image-20190503134026958.png)
  帧的最大长度为1522字节,其中1500字节是payload:

  >  图片摘自阮一峰的博客http://www.ruanyifeng.com/blog/2017/06/tcp-protocol.html
  
  ![net_package](https://gitee.com/neusnail/img/raw/master/blogimg/net_package.png)


* 透明传输

* 传输的数据中任何8比特的组合不允许和帧定界的控制字符比特一样，否则会出现帧定界错误。文本文件不会产生这样的问题，可以实现透明传输。但是如果是二进制文件，可能会找到错误的帧边界。解决方法是采用字节填充。在SOH和EOT的前面加上转义字符ESC(1B)，如果数据中存在转义字符，就在转义字符前插入个转义字符(11011)

* 差错控制
 使用循环冗余检验CRC(Cyclic Redundancy Check)，添加帧检验序列FCS(Frame Check Sequence)

 差错校验保证传递过来的帧无差错，但是数据链路层不提供可靠服务，还是存在帧丢失、帧重复、帧失序的问题

#### 网络层

网络层提供路由和寻址的功能，使两终端系统能够互连且决定最佳路径，并具有一定的拥塞控制和流量控制的能力。

网络层常见协议有IP，ICMP，OSPF，EIGRP，IGMP等

#### 传输层

常见的协议有TCP,UDP等

TCP 是一种**面向连接的**协议，它给用户进程提供**可靠的全双工的**字节流。确保数据包的可靠，有序，以及支持流量控制。

一个TCP包最长1480字节,payload大概1460左右.因此，一条1500字节的信息需要两个 TCP 数据包。HTTP/2 协议的一大改进， 就是压缩 HTTP 协议的头信息，使得一个 HTTP 请求可以放在一个 TCP 数据包里面，而不是分成多个，这样就提高了速度。

**tcp和udp的区别**

* tcp是面向连接的,也就是说，在收发数据前，必须和对方建立可靠的连接。upd是无连接的,传输数据之前源端和终端不建立连接， 当它想传送时就简单地去抓取来自应用程序的数据，并尽可能快地把它扔到网络上。 
* tcp是可靠的,而udp是不可靠的,upd不保证数据不丢失,也不保证package顺序,可靠性只能通过上层应用把控
* tcp因为是面向连接的,所以只能点对点传输,而upd因为不建立链接不需要维护状态,可以广播
* udp信息包的标题很短，只有8个字节，相对于TCP的20个字节信息包的额外开销很小。

可见udp更适合对实时性要求高的广播而tcp适合可靠传输

#### 应用层

是OSI参考模型的最高层，它是计算机用户，以及各种应用程序和网络之间的接口，其功能是直接向用户提供服务，完成用户希望在网络上完成的各种工作。

常见协议HTTP,FTP,SNMP等

### TCP协议

#### SEQ

tcp包在发送时会为每一个包编号,称为sequence number(SEQ),以便接收的一方按照顺序还原。万一发生丢包，也可以知道丢失的是哪一个包。
包的下一个seq即next seq=seq+payload 比如当前seq=1,payload长度为1400,那下一个seq就应该是1401.
#### ack
ack是acknowledgement的缩写,代表确认的意思.需要注意的是ack=n 代表着确认seq<=n-1的包正常接收.
#### 三次握手



TCP是面向连接的，无论哪一方向另一方发送数据之前，都必须先在双方之间建立一条连接。在TCP/IP协议中，TCP协议提供可靠的连接服务，连接是通过三次握手进行初始化的。三次握手的目的是同步连接双方的序列号和确认号并交换 TCP窗口大小信息。

三次握手的图示如下:

![image-20190503222544384](https://gitee.com/neusnail/img/raw/master/blogimg/image-20190503222544384.png)

wireshark抓包演示:

![image-20190503214011085](https://gitee.com/neusnail/img/raw/master/blogimg/image-20190503214011085.png)

* 第一次client发起连接,发送一个SYN包表示建立连接,可以看到此时seq=0,payload长度也是0
* 第二次server对client应答,并发送连接请求. 发送一个包flags标志位里既包含syn又包含ack此时ack=x+1=0+1=1 ,seq=0

* 第三次client接到server的应答后,对server的连接请求做应答,此时ack=y+1=0+1=1



##### 为什么需要三次握手?

握手为什么是三次?不是两次或者四次?

第一次握手是client发起连接请求,接到后**server确认client能正常发送**.

第二次握手是server对client的应答,接收到后**client确认server能正常接收,且能正常发送**

第三次握手是client对server的应答,接收到后**server确认client能正常接收**

至此,双方就都能确保对方可以正常接收和发送数据,tcp通过三次握手保证了连接的可靠性

#### 四次挥手

tcp连接断开的时候需要四次包发送操作,称为四次挥手:

![image-20190503224512958](https://gitee.com/neusnail/img/raw/master/blogimg/image-20190503224512958.png)

wireshark抓包示例:
![image-20190503223327645](https://gitee.com/neusnail/img/raw/master/blogimg/image-20190503223327645.png)

* 第一次client送一个FIN包和一个seq612,之后client进入FIN_WAIT_1阶段

* server接收到之后回复一个ack613,和seq945,表示收到了client的关闭请求,之后进入CLOSE_WAIT状态,client进入FIN_WAIT_2状态

* 之后server处理完自己其他的package之后发送一个FIN(实际用wireshark抓包没看到FIN单独出现的情况,都是伴随着一次ack)和seq945,此时server进入LAST_ACK状态,不再回复消息.

* client接收到server的FIN包后回复一个ack946之后进入TIME_WAIT状态,server接收到这个包之后直接进入CLOSED状态,client等待了两个msl(Maximum Segment Lifetime最大报文生存时间)之后没有收到应答,代表server正常关闭,便也进入CLOSED状态,关闭连接.

  

##### 为什么要四次挥手?

为什么连接时候只需要三次握手,而断开时需要四次?

因为连接的时候当server端收到client端的SYN连接请求报文后，可以直接发送syn+ack报文,ack用来应答,syn用来请求连接. 但是在关闭的时候server收到client的fin请求不能立即回复fin包,因为server可能还有其他的包没有接收或发送完毕. 所以server端只能先回一个ack,表示收到了client的关闭请求. 然后当server端处理好自己的包之后再发送一个fin报文来请求关闭连接.

#### 滑动窗口

**滑动窗口协议**（Sliding Window Protocol），属于TCP协议的一种应用，用于网络数据传输时的流量控制，以避免拥塞的发生。 该协议允许发送方在停止并等待确认前发送多个数据分组。 由于发送方不必每发一个分组就停下来等待确认，因此该协议可以加速数据的传输，提高网络吞吐量。

在上面的例子中,我们都是发送一个包,ack,再发另一个包.可以看出来,这样的吞吐量并不大,发送端需要等接收端ack一个包之后才能发下一个包.滑动窗口就是为了提高发送效率(一次发送一组package),并控制流量.

发送端的缓存结构如下图所示,下图的缓存分为`已发送已ack` `已发送未ack` `待发送` `未发送`四块,其中**已发送未ack和待发送**称为发送窗口.
而接收端缓存(这里没有画)分为`已接收` `未接收准备接收` `未接收不准备接收`三块,其中**未接收准备接收**称为接收窗口

> 滑动窗口的图懒得画了,直接偷的别人的图:<https://juejin.im/post/5c9f1dd651882567b4339bce>

![image-20190504135716557](https://gitee.com/neusnail/img/raw/master/blogimg/image-20190504135716557.png)

![image-20190504135751244](https://gitee.com/neusnail/img/raw/master/blogimg/image-20190504135751244.png)

可以看到已发送的package得到了ack确认窗口左端才会向右移一位.

TCP是双工的协议，会话的双方都可以同时接收、发送数据。TCP会话的双方都各自维护一个“发送窗口”和一个“接收窗口”。其中各自的“接收窗口”大小取决于应用、系统、硬件的限制（TCP传输速率不能大于应用的数据处理速率）。各自的“发送窗口”则要和对方的“接收窗口”大小相同

tcp在三次建立连接时就会交换双方的接收窗口大小,借此来确定发送窗口大小.

在收发过程中可以随时调整窗口的大小来达到流量控制的目的.

![image-20190504144450307](https://gitee.com/neusnail/img/raw/master/blogimg/image-20190504144450307.png)

#### TCP重传

tcp要保证数据可靠性,所以要有丢失重传机制.

tcp的ack机制只会确认连续的最后一个包,比如A发送了1,2,3三个连续的包,B接收到之后会ack4,A拿到这个ack之后就知道B确认接收到了seq=4之前的所有包.

现在假如B没接收到3号包,这时4号包来了,要怎么应答呢? 此时不能应答4号包,否则A就以为B收到了3

##### 超时重传

一种解决方案是B接收不到3,即使接收到了4也不ack,就在那死等3.而A发现B应答timeout之后会重传3或3及其之后的包.超时重传的缺点是需要等timeout,浪费时间.

##### 快速重传

TCP引入了一种叫**Fast Retransmit** 的算法，**不以时间驱动，而以数据驱动重传**。

如果包有丢失,接收方就连续ack丢失的那个包,发送方连续三次接收到相同的ack,就重传这个包.

比如上面的例子,B没拿到3号包就会ack3,此时A发来4号包,B还是ack3.A发来5号包,B继续ack3.

此时A拿到三个ack3就知道3号包丢失,于是马上重传3号包.B就收到3号包之后直接ack6.代表5号包之前都接收成功.
