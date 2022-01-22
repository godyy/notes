Redis客户端和服务器端通信使用名为 **RESP** (REdis Serialization Protocol) 的协议。

RESP 是下面条件的折中：

- 实现起来简单。
- 解析速度快。
- 有可读性。

RESP 能序列化不同的数据类型，例如整型(integers)、字符串(strings)、数组(arrays)。额外还有特殊的错误类型。请求从客户端以字符串数组的形式发送到redis服务器，这些字符串表示要执行的命令的参数。Redis用特定于命令的数据类型回复。

RESP 是二进制安全的，并且不需要处理从一个进程发到另外一个进程的大量数据，因为它使用前缀长度来传输批量数据。

***NOTE：这里概述的协议仅用于客户机-服务器通信。Redis集群使用不同的二进制协议在节点之间交换消息。***

# 网络层

虽然RESP在技术上不特定于TCP，但是在Redis的上下文中，该协议仅用于TCP连接（或类似的面向流的连接，如unix套接字）。

# Request/Response 模型

Redis接受由不同参数组成的命令。一旦收到命令，就会对其进行处理，并将应答发送回客户端。

Redis协议遵从简单的请求/响应模型，除了两个例外情况：

- redis 支持 pipeline，客户端可以一次性发送多个command，然后等待回复。

- 当一个Redis客户端订阅一个channel，那么协议会改变语义并变成*push* protocol, 也就是说，客户端不再需要发送命令，因为服务器端会一收到新消息，就会自动发送给客户端。

# RESP 描述

RESP实际上是一个支持以下数据类型的序列化协议: 简单字符串（simple string）、错误（error）、整数（integer）、大容量字符串（bulk string）和数组（array）。

RESP在Redis中作为一个请求-响应协议以如下方式使用：

- 客户端以大容量字符串RESP数组的方式发送命令给服务器端。
- 服务器端根据命令的具体实现返回某一种RESP数据类型。

在 RESP 中，数据的类型依赖于首字节：

- **单行字符串（Simple Strings）：** 响应的首字节是 `+`
- **错误（Errors）： 响应的首字节是** `-`
- **整型（Integers）： 响应的首字节是** `:`
- **多行字符串（Bulk Strings）： 响应的首字节是`$`
- **数组（Arrays）：** 响应的首字节是 `*`

RESP协议的不同部分总是以 `\r\n` (CRLF) 结束。

空值有两种表示形式：

- 用 Null Bulk String 表示：`$-1\r\n`

- 用 Null Array 表示：`*-1\r\n`

# 向Redis Server发送命令

进一步说明客户端和服务端如何交互工作：

- client向服务器发送一个由bulk string构成的RESP Array。

- Redis Server 发送有效的RESP Data Type作为回复。 

例如，使用`LLEN mylist`命令获取`mylist`的长度：

```
req: *2\r\n$4\r\nLLEN\r\n$6\r\nmylist\r\n
    *2\r\n RESP Array，长度2
    $4\r\nLLEN\r\n Bulk String，长度4，命令
    $6\r\nmylist\r\n Bulk String，长度6，key


resp：:48293\r\n Interger, 48293
```

# 多命令和pipeline

client可以使用同一个连接来发出多个command。Redis 支持pipeline，所以，多个command可以通过一个write被发送出去，并且不用等待read前面command的回复才能发送在一个命令，所有回复可以最后一起读取。

# Inline Commands

有时你手边只能操作`telnet` 并且需要给Redis 服务器端发送命令。虽然Redis协议是容易实现的，但并不适合用在交互会话。`redis-cli` 也不是随时都能可用。因此，redis还以一种特殊的方式接受为人类设计的命令，称为内联命令（Inline Commands）格式。

基本上，您只需在telnet会话中编写空格分隔的参数。由于统一请求协议中没有以`*`开头的命令，因此Redis能够检测到这种情况并解析您的命令。

```
C: PING
S: +PONG
```

```
C: EXISTS somekey
S: :0
```




