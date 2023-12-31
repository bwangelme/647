---
title: "Thrift 协议学习笔记"
date: 2023-01-11T11:03:19+08:00
lastmod: 2023-01-11T11:03:19+08:00
draft: false
tags: [thrift, blog]
author: "bwangel"
comment: true

---

本文主要讲了 thrift 的协议格式

<!--more-->
---

Thrift 是一个集序列化和服务通信功能于一身的 RPC 框架，主要包含三大部分，代码生成，序列化框架，RPC 框架。

Thrift 支持多种序列化协议，主要有 Binary, Compact, JSON, 本文主要讲解了 Binary 和 Compact 序列化协议。

# Thrift 请求响应模型

Binary 协议和 Compact 协议都假定底层的 transport 层会提供一个允许双向通信的字节流信道(例如 TCP 套接字)。它们都使用了如下的通信模式:

![](https://passage-1253400711.cos.ap-beijing.myqcloud.com/2023-01-11-113200.png)

1. Client 发送 Message 给 Server, Message 主要包括方法名，参数以及一些元信息
2. Client 将方法参数当作一个 Struct 发给 Server
3. Server 处理完后，发送 Message 给 Client。Message 主要包括方法名，消息类型和一些元信息
4. Server 发送 Struct 给 Client, 这个 Struct 包括响应或异常 body

## Binary 协议

### Message 格式

#### 编码格式

Binary 协议中, Message 的格式如下 (strict encoding)

![](https://passage-1253400711.cos.ap-beijing.myqcloud.com/2023-01-11-121420.png)

字节索引|功能说明
---|---
1|第1字节的第一位固定是1, 表示使用 strict encoding
1-2|`vvv` 前两个字节的后15位表示版本号, 固定是 1 (二进制格式是 `000 0000 0000 0001`)
3|`unused` 第三个字节 unused 表示它不使用
4|`mmm` 第四个字节格式为 00000mmm，后三位表示消息的类型
4-8|`name length` 4-8 字节存储方法名的长度，它是一个32位有符号整数，使用大端序存储，这意味着方法名最长是 2^31-1
8..|`name` 8字节后存储变长的方法名字符串，它使用 utf-8 编码
...4-end|`seq id` 最后四个字节存储序列号

#### 消息类型

Value|Binary|说明
---|---|---
1|001|`Call`: thrift 调用
2|010|`Reply`: thrift 响应
3|011|`Exception`: thrift server 返回了异常
4|100|`Oneway`: thrift oneway 调用

#### 序列号

序列号是有符号的四字节整数，在一个传输层的连接上所有未完成的请求必须有唯一的序列号，客户端用序列号来处理响应的失序到达，实现请求和响应的匹配。服务端不需要检查序列号，也不应该在实现中对序列号有任何的依赖，只需要在响应的时候将其原样返回。

#### old encoding

上述讲的 binary 协议中的 strict encoding, 在 thrift 早期版本中，使用的是 old encoding 编码方式。old encoding 的编码方式如下

![](https://passage-1253400711.cos.ap-beijing.myqcloud.com/2023-01-11-123231.png)

它没有版本号等信息，直接传输了方法名，类型以及 seq id

因为 old encoding 中，`name length` 是使用32位有符号数存储的，因为 `name length` 不能为负数，所以 old encoding 的第一位始终是0 (`name length` 在最前面)。strict encoding 中，将第一位设置成了1, 以此来区分 old encoding 和 strict encoding。

### struct 格式

Struct 类型由两部分组成: Field + StopField

```
Struct := Field + StopField
Field := Field Header + Field Value
Field Header := field id + field value
```

![](https://passage-1253400711.cos.ap-beijing.myqcloud.com/2023-01-11-130322.png)

StopField 长度一个字节，全都置0，表示 Struct 或 Request/Response Body 的结束

![](https://passage-1253400711.cos.ap-beijing.myqcloud.com/2023-01-11-130230.png)

Normal Field 的格式如上图所示，字段说明如下

字段|说明
---|---
tttttttt|field 类型，一个8位的有符号整数
field id|16位的有符号整数，使用大端字节序存储
field value|编码后的 field value

field 类型的定义表

类型|field type number
---|---
bool|2
byte|3
double|4
i16|6
i32|8
i64|10
string|11, used for binary and string fields
struct|12, used for struct and union fields
map|13
set|14
list|15

![](https://passage-1253400711.cos.ap-beijing.myqcloud.com/2023-01-11-130956.png)

上图是一个 thrift 请求的抓包，可以看到此请求的 struct 中，共有三个 Field，分别存储了 1, 0, 4 这三个 I32 整数。

### list 和 set 的格式

list 和 set 类型使用的是相同的格式，如下图所示:

![](https://passage-1253400711.cos.ap-beijing.myqcloud.com/2023-01-11-145100.png)

- tttttttt 表示集合元素类型，被编码成了 i8
- size 表示集合中元素的格式，被编码成了 i32 (32位有符号整数)，且只允许是正数
- elements 表示元素

集合中元素的类型和 struct 中 field-types 使用相同的定义。

可以配置 list/set 最大的长度，默认没有设置，即是 i32 的最大值，(2^31-1, 2147483647)

### binary/string 格式

![](https://passage-1253400711.cos.ap-beijing.myqcloud.com/2023-01-11-170954.png)

binary 类型的格式如上图所示，4个字节的长度 + values。长度是 32位的有符号整数，这意味着 binary 数据的最大长度是 2^32-1, 2147483647

string 类型被 encode 成 utf-8 编码，然后使用 binary 的格式传输。

### map 格式

map 的格式如下:

![](https://passage-1253400711.cos.ap-beijing.myqcloud.com/2023-01-11-145643.png)

- kkkkkkkk 表示 map 中 key 的元素类型，被编码成了 i8
- vvvvvvvv 表示 map 中 value 的元素类型，被编码成了 i8
- size 表示 map 的大小，被编码成了 i32，只允许是正数
- key value pairs 表示被编码后的 key-value 对。

可以配置 map 最大的长度，默认没有设置，即是 i32 的最大值，(2^31-1, 2147483647)

### request 格式

thrift Request 由两部分组成, Message + Data

1. Message 的格式参考上文
2. Data 使用 struct 格式序列化的方法的参数

![](https://passage-1253400711.cos.ap-beijing.myqcloud.com/2023-01-11-150009.png)

上图是用 wireshark 抓到的 thrift 请求:

1. 它是一个 Call 消息，方法名是 `calculate`, seq id 是 0
2. 它有两个参数，第一个参数是 i32 整数，第二个参数是一个结构体，这两个参数被用 struct 的方式序列化起来，变成了 Data

### response 格式

thrift Response 由两部分组成，Message + Data

1. Message 的格式参考上文
2. Data 是函数的返回值，它以 struct 的方式序列化，
   - 返回值的 field id == 0 时，表示这是一个正常的响应
   - 返回值的 field id == 1 时，表示这个响应是 server 抛出了一个 thrift idl 文件中定义的异常

![](https://passage-1253400711.cos.ap-beijing.myqcloud.com/2023-01-11-151113.png)

上图抓到的 thrift 响应的包:

- Message 部分表示它是一个 Reply 类型的消息。
- Data 部分，由于 field id == 1, 表示它返回了一个异常。
- 整个 Data 部分是一个 struct 格式的响应体，异常又是一个 struct 格式的数据
- 响应的异常包含了 i32 数字和 string 两个字段。

![](https://passage-1253400711.cos.ap-beijing.myqcloud.com/2023-01-11-151113.png)

### exception 格式

thrift exception 响应表示 thrift server 出现了某种错误，无法正确地处理请求

它由两部分组成: Message + Exception

1. Message 的格式参考上文
2. Exception 的 Data 也是按照 struct 格式来编码的

下图是 thrift exception 响应抓取的包:

![](https://passage-1253400711.cos.ap-beijing.myqcloud.com/2023-01-11-173446.png)

- `00 00 00 2e` 是 frame 的长度
- `80 01 ... 6b 00 00 00 00` 这是 Message 的定义
- `0b` 表示第一个 field 是一个 string
- `00 01` 表示 field id == 1
- `00 00 00 0e` 表示 string 的长度是 14
- `49 6e 74 65 72 6e 61 6c 20 65 72 72 6f 72` 表示 `Internal error` 这个字符串
- `08` 表示第二个 field 是一个 i32
- `00 02` 表示 field id == 2
- `00 00 00 06` 表示 i32 的 值是6
- `00` 表示 StopField，代表这个 Struct 结束了

thrift exception 中数字的含义如下:

![](https://passage-1253400711.cos.ap-beijing.myqcloud.com/2023-01-11-173956.png)

## Compact 协议

### var int

Compact 协议中，使用 var int + zigzag int 的方式来将整数进行编码

1. i32, i64 格式的整数被转换成 zigzag 格式的整数，这样所有整数都使用非负整数的格式存储
2. zigzag 整数被编码成了 var int 变长整数。i32 整数消耗 1-5 个字节，i64 整数消耗 1-10 个字节。

- i16 首先被转换成 i32 之后，再进行 zigzag + var int 编码, i8 数字不会进行编码，直接进行传输。

关于 zigzag 以及变长整数编码，可以参考我的另一篇文章: [ZigZag 变长整数编码](/2022/03/01/variant_zigzag/)

### Message 如何编码

![](https://passage-1253400711.cos.ap-beijing.myqcloud.com/2023-01-11-125326.png)

Compact 协议中，Message 的编码格式如上图

字段|说明
---|---
pppppppp|协议 ID, 固定是 0x82, 1000 0010
mmm|消息类型，和 binary 协议中的消息类型定义相同
vvvvv|协议版本，无符号5位整数，固定是 00001
seq id|序列号 ID, 一个32位有符号整数被编码成了 var int
name length|方法名长度，一个有符号32位整数被编码成了 var int (name length >= 0)
name|方法名, utf-8 编码的字符串

## framed vs unframed

- unframed 会将数据直接写入到 socket

- 使用 framed 格式，client/server 先将 request/response 写入到一个 buffer 中。最后向 socket 写入时，先写入一个四字节的 request/response 长度，再写入 request/response。

- framed 格式下，请求的最大长度是 16384000 (16M)
- thrift 引入 framed 格式的目的是为了方便异步处理器的实现。

## 参考链接

- [Thrift序列化协议浅析](https://andrewpqc.github.io/2019/02/24/thrift/)
- [Thrift specification - Remote Procedure Call](https://erikvanoosten.github.io/thrift-missing-specification/)
- [ZigZag 变长整数编码](https://www.bwangel.me/2022/03/01/variant_zigzag/)

