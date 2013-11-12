MQTT 3.1 protocol specific
===================================
MQ(Message Queue) Telemetry Transport(MQTT)是一个轻量级的基于经纪的发布/订阅消息协议，  
设计原则是开放、简单，并且容易实现。这些特点使MQTT协议成为有限的环境下的比较完美的解决方案，  
有限的环境可能有：
>1.在网络费用比较高的地方，只有较低的带宽或者网络不稳定  
>2.在嵌入式设备中，性能有限的处理器和内存资源  

##协议特点包括：
>1.发布/订阅的消息模式提供了一种消息一对多的分发和与应用程序解耦的机制  
>2.对传递的内容是不可知的一种消息传输方式  
>3.使用TCP/IP协议去提供基础的网络连接  
>4.三个层次的消息分发服务：

>>最多一次，消息发布完全依赖底层 TCP/IP 网络。会发生消息丢失或重复。这一级别可用于如下情况，环境传感器数据，丢失一次读记录无所谓，因为不久后还会有第二次发送。  
>>"至少一次"，确保消息到达，但消息重复可能会发生。  
>>"只有一次"，确保消息到达一次。这一级别可用于如下情况，在计费系统中，消息重复或丢失会导致不正确的结果。

>5.小型传输，开销很小（固定长度的头部是 2 字节），协议交换最小化，以降低网络流量。  
>6.使用 Last Will 和 Testament 特性通知有关各方客户端异常中断的机制。  

#目录
##1.引言
包括三个小节：
>1.对所有类型的数据包通用的消息格式  
>2.每种数据包的消息内容的具体信息  
>3.数据包是如何在客户端和服务器之间流转  
在附录中提供了主题是如何使用通配符的介绍。

###1.1 与V3版本的改变
下面是MQTT V3 和MQTT V3.1之间的变化：
>1.用户名和密码现在可以用一个连接数据包来发送  
>2.新的返回代码在"CONNACK"数据包，提高安全性  
>3.客户端将不会收到没有经过授权的发布/订阅命令，并且MQTT流程必须完成不管命令有没有被执行  
>4.消息中的字符串新增UTF-8的支持，不仅仅支持US-ASCII  

协议的版本号通过“CONNECT”数据包，仍然停留在3.x版本。MQTT V3的服务器可以接受V3.1的客户端的请求，只要请求中正确处理"剩余长度"字段，并且因此忽略了额外的安全问题。

##2.消息格式
每条MQTT命令消息都包含一个固定的头。有些消息也需要可变的头和有效载荷。下面的章节描述了消息头的每个部分：

###2.1固定头
每条MQTT命令消息都包含一个固定的头。下面表格展示了固定头的格式：  
![fixed header](http://ww3.sinaimg.cn/large/92540662jw1eahg7z7vz8j20gz02ijrj.jpg)

###第一个字节
    包括消息类型和标识（DUP，QoS level， RETAIN）字段
###第二个字节
    （至少一个字节）包括剩余的信息字段

所有的字段在下面的章节描述。所有的数据排序规则是顺序值越大越优先。1个16bit的字母出现在最重要的字节，后面是最不重要的字节。

###1.消息类型
位置：第一个字节，7-4bits  
是1个4bit的无符号值。  
![message types](http://ww4.sinaimg.cn/large/92540662jw1eahg810ro9j20ek0g9gnz.jpg)

###2.标识
第一个字节剩下的位数包括DUP，QoS 和 保留字。这几个位数的位置设置：  
![flags](http://ww4.sinaimg.cn/large/92540662jw1eahg80a0ibj208c0370su.jpg)

####DUP
位置：字节1的第三位  
当客户端或服务器尝试重新分发一个PUBLISH,PUBREL,SUBSCRIBE或者  
UNSUBSCRIBE消息的时候设置这个标识。这可以让消息的QoS>0，并且需要认同。  
如果设置了DUP位，变量头就包括一个消息的ID。

消息的接收者可以把这个标识作为前一个消息是否已经接收到的标记。但是不应该被依赖于检测重复。

####QoS
位置：字节1的1-2位  
这个标识表示消息发布的服务质量。值和服务质量的对应关系：  
![QoS](http://ww2.sinaimg.cn/large/92540662jw1eaiib4h5s5j20fq04it97.jpg)

####RETAIN
位置：字节1的第0位  
这个标识只有发布消息时使用。当客户端发送一个PUBLISH消息到服务器，如果这个标识的值是1，  
服务器就会一直持有这个消息，直到消息被分发到当前所有的用户。

当一个主题上创建了一个新的订阅，这个主题最后遗留的消息将设置RETAIN标识并被发送到用户。  
如果没有遗留的消息，什么都不会被发送。  

这个标识在发布者发布基于“异常通知”的消息时是很有用的，也可能用于消息之间。  
这让新用户及时的收到遗留的数据或者上一个正确的数据。

当服务器发送一个“PUBLISH”消息到客户端作为一个在原始的"PUBLISH"消息到达时就已经存在的  
“订阅”的响应，这时不应该设置“RETAIN”标识，不管原始的"PUBLISH"消息的“RETAIN”标识。  
这就使得客户端可以区分将要接收的消息是遗留的还是那些“活着”的。

服务器重启需要保存好遗留的消息。  

当服务器收到一个0字节的命令或者RETAIN标识被设置为相同的主题时，可以删除遗留的消息。

###第二个字节
这个字节包含当前消息的剩余部分，包括变量头部和负载的数据。

可变长度的编码方式使用一个单独的字节使消息可以达到127字节的长度上限。更长的消息被处理：  
每个字节的7位编码剩余的数据长度，第8位显示的是出现在后面的字节。每个字节编码了128个值    
和一个“继续位”。比如，10进制的数字64编码为hex 0x40，10进制的数字321（=65 + 2 * 128）  
编码为2个字节，第一个字节更重要：65 + 128=193。注意第一位用来标识至少还有一个字节。  
第二个字节是2。  

协议限制最多4个字节，这样程序可以发送最大256Ｍ的消息。消息传输时数字的展现是：  
0xFF,0xFF,0xFF,0x7F。

下面表格展示剩余长度和增长的字节数：
![remaining_length](http://ww1.sinaimg.cn/large/92540662jw1eaik06fmpyj20j604ljs3.jpg)

把十进制的数字编码为可变长的编码方式是这样的：
```
do
    digit = X MOD 128
    X = X DIV 128
    // if there are more digits to encode, set the top bit of this digit
    if(X > 0)
        digit = digit OR 0x80
    endif
    'output' digit
while (X>0)
```

MOD 是求模的操作符(就像Ｃ语言的%)，  
DIV是除法取整（就像Ｃ语言的/），
OR 是按位或（就像Ｃ语言的|）。

编码剩余长度字段的算法是：  
```
multiplier = 1
value = 0
do
    digit = 'next digit from stream'
    value += (digit AND 127) * multiplier
    multiplier *= 128
while((digit AND 128) != 0)
```

AND 是按位与（就想Ｃ语言的&）。

这个算法执行完成后，值 就包含了剩余的长度。

剩余长度的编码不是可变头部的一部分。用于编码剩余长度的字节不会改变剩余长度的值。  
“extension bytes”的变量长度是固定头的一部分，不是可变头的。
