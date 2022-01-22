# Redis pipeline

redis server 基于TCP协议，并使用client-server模型和request/response协议。这意味着一个redis请求是经过如下步骤完成的：

- client发送查询请求到server，并从socket读取server的response，通常是阻塞的。

- server收到client的request，处理后，发送response。

每一条查询请求都会通过网络连接发送，随之而来的就是网络延迟（latency），也就是RTT。例如一条查询收到结构的RTT是250ms，这样，即便server每秒能支持100k条查询，在这样的环境下，每秒也只能处理4条请求。

redis实现了pipeline来解决这样的问题。

# 实现

pipeline是一种被广泛使用了几十年的技术。request/response服务器可以实现为，在client没有read之前发送的reponse的情况下，server仍然可以处理新的request。这样，client可以一次性发送多条查询命令到服务器，并在最后一次性读取responses。

IMPORTANCE NOTE：当client使用pipeline发送命令，server会使用额外的内存来保证response的排队。所以，当你需要发送大批量的命令，最好是按照一个合理的数量分批次发送。这样，对查询效率影响不大的同时，也能降低server的内存压力。

# VS Scripting

在pipeline中使用scripting可以获得更高的性能提升。通过scripting，可以在server方一次性处理大量的工作，以最小的延迟读写数据，例如read，compute，write可以很快完成。如果只使用pipeline，在前一次read收到response之前，是不能write的。
