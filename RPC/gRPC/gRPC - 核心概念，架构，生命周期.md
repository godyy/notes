# 概述
## Service Definition
像其它很多 RPC 系统一样，gRPC 基于定义服务的构想，指定可以通过其参数和返回类型来远程调用的方法。gRPC 默认使用 protocol buffers 作为 Interface Definition Language（IDL），用于描述服务接口和负载结构（message）。

```
service HelloService {
  rpc SayHello (HelloRequest) returns (HelloResponse);
}

message HelloRequest {
  string greeting = 1;
}

message HelloResponse {
  string reply = 1;
}
```
gRPC 允许定义4种 service 方法：

- 一元方法，客户端发送单个请求，服务器返回单个响应。

```
rpc SayHello(HelloRequest) returns (HelloResponse);
```
- 服务端流式方法，客户端发送单个请求，服务端返回一个流来读取一系列的消息。gRPC 保证单个调用中的消息顺序。

```
rpc LotsOfReplies(HelloRequest) returns (stream HelloResponse);
```
- 客户端流式方法，客户端写入一系列消息并将它们发送到服务器。一旦客户端完成了消息的写入，它就会等待服务器读取消息并返回响应。同样，gRPC保证了单个RPC调用中的消息顺序。

```
rpc LotsOfGreetings(stream HelloRequest) returns (HelloResponse);
```
- 双向流式方法，双方使用读写流发送一系列消息。两个流独立运作,因此客户端和服务器可以读和写在他们喜欢的任何顺序:例如,服务器可以等待收到所有客户端消息后再响应,也可以交替阅读一条消息然后写一个消息,或其他一些读写的结合。每个流中的消息顺序被保留。

```
rpc BidiHello(stream HelloRequest) returns (stream HelloResponse);
```

## 使用 API
gRPC 提供了 protocol buffer 编译器插件来生成在.proto文件中定义的服务的 client and server side code。

- 服务端实现了 service 中声明的方法，并且运行 gRPC 服务器来处理来自客户端的调用。 gRPC infrastructure 对传入的请求进行解码，执行服务方法，并对服务响应进行编码。

- 在客户端有一个称为存根(对于某些语言，首选术语是客户机)的本地对象，它实现了与 service 相同的方法。然后客户端就可以在本地对象上调用这些方法，将调用的参数包装在适当的 protocol buffer 消息类型中—— gRPC 负责向服务器发送请求并返回服务器的 protocol buffer 响应。

## 同步与异步
同步调用，在服务器的响应到达之前一直处于阻塞状态，是 RPC 所期望的过程调用的最接近的抽象。另一方面，网络天生就是异步的，在很多情况下，能够在不阻塞当前线程的情况下调用 RPC 是很有用的。

大多数语言实现的 gRPC 编程 API 都提供了同步和异步的方法。

# RPC 生命周期
来看一看，在一个 gRPC 客户端调用一个 gRPC 服务端方法时发生了什么。

## 一元 RPC
1. 一旦客户机调用存根方法，服务器就会被通知，RPC已经被调用了，该调用使用客户机的元数据、方法名以及指定的 deadline (如果适用的话)。
2. 然后，服务器可以直接返回它自己的初始元数据(必须在任何响应之前发送)，或者等待客户机的请求消息。首先发生的是特定于应用程序的。
3. 一旦服务器获得了客户机的请求消息，它就会执行创建和填充响应所需的任何工作。然后将响应(如果成功)连同状态详细信息(status code 和可选的 status message)和可选的（trailing metadata）返回给客户端。
4. 如果 response status 为OK，则客户端将获得响应，从而完成客户端上的调用。

## 服务端流式 RPC
与一元 RPC 类似，除了服务端返回消息流。在发送完所有消息后，服务端向客户端发送状态详情和可选的 trailing metadata，这样服务端的处理即完成。客户端在收到所有消息后完成。

## 客户端流式 RPC
与一元 RPC 类似，除了客户端发送的是消息流。服务端返回单个消息（伴随状态详情和可选的 trailing metadata），通常但不一定是在收到所有客户端消息之后。

## 双向流式 RPC
调用由调用方法的客户机和接收客户机元数据、方法名和 deadline 的服务器发起。服务器可以选择发回其初始元数据或等待客户端开始消息流。

客户端和服务器端流处理是特定于应用程序的。由于这两个流是独立的，客户端和服务器可以以任何顺序读写消息。例如,一个服务器可以等到它已经收到了客户端的所有消息之后响应,或者，服务器和客户端可以玩“乒乓球”——服务器收到一个请求,然后发回一个响应,然后根据响应客户端发送另一个请求,等等。

## Deadlines/Timeouts
gRPC 允许客户端指定在使用 DEADLINE_EXCEEDED 错误终止 RPC 之前，他们愿意等待RPC 的完成多长时间。在服务端，服务器可以查询特定的 RPC 是否超时，或者还剩下多少时间来完成 RPC。

指定 deadline 或 timeout 是特定于语言的: 一些语言 API 使用 timeout (持续时间)，而一些语言使用 deadline (一个固定的时间点)，可能有也可能没有默认的 deadline。

## RPC 终止
在 gRPC 中，客户端和服务端都对调用的成功与否做出独立的本地判断，并且他们的结论可能不匹配。这意味着，例如，您可以有一个 RPC 调用，它在服务器端成功完成(“我已经发送了我所有的响应!”)，但在客户端失败(“响应在我的 deadline 之后到达!”)。服务器也可以在客户端发送完所有请求之前决定是否完成。

## 取消一个 RPC
客户端和服务端都可以在任意时候取消一个 RPC 调用。“取消”会立即终止 RPC 调用，不会完成进一步的工作。
> Warning
> 
> 在“取消”之前完成的更改不会回滚

## Metadata
元数据是以键-值对列表的形式表示的关于特定 RPC 调用的信息(比如身份验证细节)，其中键是字符串，值通常是字符串，但也可以是二进制数据。元数据对于 gRPC本身 是不透明的——它让客户机提供与服务器调用相关联的信息，反之亦然。

访问 metadata 独立于语言。

## Channels
gRPC 通道提供到指定 host 和 port 的 gRPC 服务器的连接。它在创建客户端存根时使用。客户端可以指定通道参数来修改 gRPC 的默认行为，比如打开或关闭消息压缩。通道有状态，包括 `connected` 和 `idle`。

gRPC 如何处理通道的关闭是语言独立的。某些语言还允许查询通道的状态。