---
title: "grpc底层原理浅析"
date: 2023-02-09
tags: ["grpc"]
categories: ["grpc"]
---

之前已经写过grpc的使用，以及HTTP2的介绍，可以参考如下链接：
* [基于Go语言的gRPC 使用指北](https://zhengweikeng.github.io/seed-blog/posts/grpc-go-technology/)
* [一文说透HTTP2](https://zhengweikeng.github.io/seed-blog/posts/one-blog-to-know-http2/)

因此本文则默认为你已经了解如何使用grpc来搭建服务与完成服务间的通信，来重点介绍grpc的底层原理。

## protocol buffers的编码
我们知道grpc是使用protocol buffers协议作为它的接口定义语言（IDL），并转换成对应语言的源代码，最终由该语言的源码进行编码作为网络传输，而这个编码的规则也是协议定义好的，所以我们有必要来了解下protocol buffers是如何进行编码。

我们以下面一个protocol buffers定义为例子：

```protobuf
message User{
	string name=1;
}
```

这里我们定义了一个简单的message，其中有一个字段类型为string的字段name。

该协议最终会被编码成如下结构的字节流：

![protobuf_byte_steam](protobuf_byte_stream.png)

也就是说每个字段的定义都会被分为两部分，标签和值。

其中标签的值需要由以下两部分组成：
1. 字段索引（field index）
2. 线路类型（wire type）

字段索引，顾名思义就是我们定义字段时最右侧的那个索引值。

而线路类型则会根据字段类型来进行定义，它被用来确定值的长度。

首先，我们来看下线路类型和字段类型的映射关系：

| 线路类型 | 分类         | 字段类型                                                 |
| -------- | ------------ | -------------------------------------------------------- |
| 0        | Varint       | int32、int64、uint32、uint64、sint32、sint64、bool、enum |
| 1        | 64位         | fixed64、sfixed64、double                                |
| 2        | 基于长度分隔 | string、bytes、嵌入式消息、打包的repeated字段            |
| 3        | 起始组       | groups（已废弃）                                         |
| 4        | 结束组       | groups（已废弃）                                         |
| 5        | 32位         | fixed32、sfixed32、float                                                         |

然后会按照如下规则来计算标签的值：

tag value = (field index << 3)  | wire index

也就是说先将字段的索引值左移3位，再将其与对应的线路类型进行按位或操作。

这个也比较好理解，其实就是线路类型占3位，再拼上字段索引。

对应到上述例子，name字段的类型为string，则线路类型为2，对应的二进制为00000010。字段的索引值为1，对应的二进制为00000001。代入以上公式中，则为

tag value = (00000001 << 3) | 00000010 = 00001010

00001010对应的10进制值为10，也就是说name字段的标签值为10。

再来说下上述字节流中的值。在protocol buffers中，会根据字段的类型选择不同的编码方式。

目前protocol buffers采用的字段编码技术有：
* Varint，即可变长度整数，使用了单字节或多字节来序列化整数的方法，值越小的数字，使用的字节数越少，这样对空间的占用也得到减少。例如一个int32类型的数字，一般是需要4个字节来存储的，但是对于很小的数字，像1、10、100这些，甚至1个字节就足够存储。
* 固定字节类型
	* 64位类型，如fixed64、sfixed64、double
	* 32位类型，如fixed32、sfixed32、float
* 字符串类型，会使用UTF-8来进行编码，这个就不做过多介绍了，网上资料也有很多

好了，了解了protocol buffers采用的字段编码技术，根据之前的例子，假设name字段的值是Jack，则对应的utf-8编码值为`\x4A\x61\x63\x6B`，此时protocol buffers会用如下16进制编码来表示：

```
A 04 4A 61 63 6B
```

其中04说明了编码后的字符串值的长度。

最后将编码后的标签和值连接到之前说的字节流中即可，流结束时会以0作为结尾。

## 消息分帧
知道了消息是怎么编码的，接下来看下消息最终的数据结构。

![grpc-message.png](grpc-message.png)

第一个字节表示压缩标记，用来表示是否进行了压缩。

接下来的4个字节，用来表示消息的具体长度，需要注意的是，这里会采用大端（big-endian）的格式来表示。4个字节来记录长度，也就说明gRPC可以处理大小不超过 4GB 的消息。

注：大端是一种在系统或消息中对二进制数据进行排序的方式。在大端格式中， 序列中的最高有效位(2 的最大乘方)存储在最低的存储地址上。

之后的字节就是具体的消息数据了。

以上这种消息分帧的技术我们称为 **长度前缀分帧(length-prefix framing)** 的消息分帧技术。

## HTTP/2
grpc会采用HTTP/2作为其网络传输协议，关于HTTP2这里就不解释了，可参考开头处给出的链接。

客户端和服务端会采用HTTP/2协议建立连接，之后会在连接中采用流的方式传输数据，流中采用数据帧的形式进行发送。而被发送的消息，可能会在一个数据帧中，也可能会跨多个数据帧。

当客户端需要发起请求时，便会发起请求消息操作。

![](grpc-req-res-message.png)

在发起请求的时候，grpc会封装一些http/2的请求头，例如如下所示

```
HEADERS (flags = END_HEADERS)
:method = POST
:schema = http
:path = /user
:autoority = example.com
grpc-timeout = 0.5s
content-type = application/grpc
grpc-encoding= gzip
```

关于请求头有几点注意：
* 这里以":"开头的请求头是保留头信息，是HTTP/2要求保留头信息需要出现在其他头信息之前
* grpc传递的头信息包含：
	* 调用定义的头信息（call-definition header），其是HTTP/2预定义的头信息，需要在下述的自定义元数据之前发送
	* 自定义元数据，是由应用程序定义的任意一组键值对，这些头要确保不要使用“grpc-”开头

请求消息结束需要放置一个特殊的EOS（End of Stream）数据帧来标记。

```
DATA (flags = END_STREAM)
<Length-Prefixed Message>
```

对于响应消息也有类似的结构，只是消息中间可能并没有“以长度作为前缀的消息”

```
HEADERS (flags = END_HEADERS)
:status = 200
grpc-encoding = gzip
content-type = application/grpc
```

响应消息会发送单独的END_STREAM数据帧来说明结束响应的消息，即trailer

```
DATA
<Length-Prefixed Message>
```

这个结束的消息帧还会包含一些头信息

```
HEADERS (flags = END_STREAM, END_HEADERS)
grpc-status = 0
grpc-message = xxx
```

最后来看下grpc中不同的通信模式下，流的通信方式。

grpc中针对不同的使用场景有不同的通信模式，这个具体也可以参考开头所给的文章，这里不重复说明。


![rpc-steram.png](rpc-stream.png)

## 小结
文本对grpc的底层原理做了个基本的介绍，当然其通信细节远远不止于此，例如protocol buffers的编码细节这里也没有展开讲，对这方面感兴趣的可以查阅protocol buffers的官方文档。

而grpc支持很多扩展功能，例如服务发现、负载均衡等，这些都属于应用层的技术，也可以自行去查看其源码。