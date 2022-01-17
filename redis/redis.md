# 1. 介绍

Redis 是一个开源的（BSD licensed）内存数据结构存储，可以作为数据库、缓存、小弟代理使用。

Redis 提供的数据结构包括，字符串、哈希表、列表、集合、支持范围查询的有序集合、位图、Hyperloglogs、空间索引（geospatial indexes）、流（streams）。

Redis 内置了副本（replication）、Lua 脚本、LRU 驱逐、事务、以及不同级别的持久性（on disk），还提提供了高可用性（Redis Sentinel）和自动分区（Redis Cluster）。

Redis 数据结构都支持原子操作。

为了达到最佳性能，Redis 基于内存数据集工作。根据您的用例，您可以通过定期将数据集转储到磁盘或将每个命令附加到基于磁盘的日志来持久化数据。如果您只是需要一个功能丰富的、联网的、内存中的缓存（cache），还可以禁用持久性。

Redis 还支持异步复制，具有非常快的非阻塞首次同步，在网络分裂上自动重连接与部分重同步。

其它特性包括：

- Transactions 事务

- pub/sub

- Lua scripting

- keys with limited time-to-live

- LRU eviction of keys

- Automatic failover 自动故障转移

# 2. 数据结构

Redis 不是普通 key-value 存储，它实际上是一个数据结构服务器，它支持不同种类的值。这意味着，在传统的 key-value 存储中，字符串键和字符串值关联，而在 Redis 中，值不仅可以是字符串，还可以是更复杂的数据结构。

下面列出了 Redis 所支持的所有数据结构：

- 二进制安全的字符串（Binary-safe strings）

- 列表（Lists）：根据插入顺序排序的字符串元素集合。基本上是链表

- 集合（Sets）：唯一的、未排序的字符串元素集合

- 有序集合（Sorted sets）：类似于集合，但每个元素都关联一个被称作 score 的浮点数字值。元素总是基于其 score 排序，所以可以检索一个范围内的元素，例如 top 10

- 哈希表（Hashes）：键值映射表（maps）。键值都是字符串。

- 位数组/简单位图（Bit arrays or simple bitmaps）：可以使用特殊的命令来像操作位数组一样操作字符串值，例如，set or clear individual bits、计算所有被设置为1的位（count all the bits set to 1）、查找第一个 set/unset bit，等等。

- HyperLogLogs：这是一个概率数据结构，用来估计一个集合的基数。***不要被吓到，其实它比看起来容易。***

- 流（Streams）：一个 make-like entries 仅追加（append-only）集合（collections），用于提供抽象日志数据类型（abstract log data type）

## 2.1 keys

Redis keys 是二进制安全的，这意味着可以直接使用二进制序列作为 key，like “foo” 字符串

and JPEG 文件的内容。

***空字符串（empty string）也可以作为有效的 key。***

key 的相关规则：

- 不推荐非常长的 keys （not good idea）。不仅仅是对内存不友好的，而且，在数据集中查找这样的 key 可能需要几个昂贵的（costly）key-comparisons。即使手头的任务是匹配一个大值（large value）的存在，哈希 key （for example with SHA1）是更好的主意，尤其是从内存和贷款的角度考虑。

- 非常短的 key 通常也不是好主意。例如，如果能用“user:1000:followers”作为 key，使用“u1000flw”就没什么意义了。前者可读性更强，而且添加的空间与键对象本身和值对象使用的空间相比较小。然而短键显然会消耗更少的内存，所以，你的的工作是找到正确的平衡。

- 试着坚持一种模式（stick with a schema）。例如“object-type:id”，“user:1000” for instacne。点（Dots）和破折号（Dashes）通常用于多单词字段，例如"comment:1234:reply.to"和 "comment:1234:reply-to"。

- key 允许的最大大小为 512MB。

# 1. 基础

Redis 中的所有数据结构，都不需要先创建 key，然后再添加值，可以直接使用命令来添加键值对。例如：

```go
del test // 保证 test 不存在
incr test // => 1, test不存在的情况下，相当于初始值
```
