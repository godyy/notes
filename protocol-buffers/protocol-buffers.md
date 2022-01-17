# Enumeration

- 必须以0作为一个第一个枚举值。

- 可以定义别名: option allow_alias

  ```
  message MyMessage1 {
    enum EnumAllowingAlias {
      option allow_alias = true; // must set.
      UNKNOWN = 0;
      STARTED = 1;
      RUNNING = 1;
    }
  }
  ```

- 枚举值必须在32位整数的取值范围内

- 枚举值采用 varint 编码，负数值是低效且不推荐的

- Reserved Values，容错

  ```
  enm Foo {
  	reserved 2, 15, 9 to 11, 40 to max;
  	reserved "FOO", "BAR";
  }
  ```

  

# Importing Definitions 导入定义

可以 import 其它`.proto`文件来使用其中的定义。

```
import "myproject/other_protos.proto";
```

默认情况下，只能使用直接导入的`.proto`文件中的定义。然而有时可能需要移动`.proto`文件到一个新的位置。避免直接移动`.proto`文件并且更新所有的import路径，你可在旧的`.proto`文件中放置一个占位符，将所有的导入指向新的位置，通过使用`import public`概念。（not available in Java）

任何`import public`依赖都可以直接传递到`import`了包含`import public`语句的proto文件的代码中。举例：

```
// new.proto
// All definitions are moved here
```

```
// old.proto
// This is the proto that all clients are importing.
import public "new.proto";
import "other.proto";
```

```
// client.proto
import "old.proto";
// You use definitions from old.proto and new.proto, but not other.proto
```

protocol 编译器使用`-I/——proto_path`标志在命令行指定的一组用以搜索导入文件的目录。如果没有给出标志，则在调用编译器的目录中查找。一般情况下，应该将`——proto_path`标志设置为项目的根目录，并对所有导入使用完全限定的名称。

# 使用 proto2 Message type

可以在 proto3 message 中导入 proto2 message，反之亦然。然而 proto2 枚举不能直接在 proto3 语法中使用。

# 更新 Message Type（兼容）

如果已有的 Message Type 无法满足你的所有需求，例如，你需要 message 具有一个额外的字段，但你希望继续使用用旧的 message 格式创建的代码。更新 message 并且不破坏已有的代码，谨记下列规则：

- 不要变更已存在字段的 field number
- 如果增加了新的字段，使用旧的 message format 创建的任意消息，都可以被新生成的代码（generated code）解析。旧的 message 被新的代码解析，新的字段会被设置默认值。新的 message 被旧代码解析，新的字段会被忽略。
- 字段可以被移除，只要 field number 不再被更新后的 message type 使用。你可能需要重命名该字段来代替，可以加上“OBSOLETE”前缀，或者使 field number reserved，这样未来的使用者就不会不小心地使用这个 number。
- `int32`,`uint32`,`int64`,`uint64`,`bool`都是可兼容的，这就意味着，你可以将一个字段从其中一个类型替换为其它类型，而不会破坏向前/向后的兼容性。如果解析出的数字不符合相应的类型，将得到如在C++中做同样转换的相同效果。如果一个64位的数是以`int32`来读取的，结果会被`truncated`。
- `sint32`,`sint64`是相互兼容的，但不与其它整数类型兼容。
- `string`和`bytes`是相互兼容的，只要`bytes`是有效的`UTF-8`。
- 嵌入的 message 可与 bytes 兼容，只要 bytes 是 message 的编码形式。
- `fixed32`与`sfixed32`兼容，`fixed64`与`sfixed64`兼容。
- 对于`string`,`bytes`和`message`字段，`optional`与`repeated`兼容。给定一个重复字段的序列化数据作为输入，期望该字段为`optional`的客户端将接受最后一个输入值(如果它是基本类型字段)，或者合并所有的输入元素(如果它是消息类型字段)。请注意，这对于数字类型(包括bool和enum)通常是不安全的。数字类型的重复字段可以以`packed`格式序列化，但当需要一个`optional`字段时，将无法正确解析该格式。
- `enum`与`int32`,`uint32`,`int64`,`uint64`兼容。但是要注意，在反序列化时，客户端代码可能会以不同的方式对待他们：例如，未被识别的 proto3 `enum`类型会被保留在 message 中，但是当消息被反序列化时，它是如何表示的取决于语言。Int 字段总是保留他们的值。
- 将一个单值（single value）调整为新建的`oneof`的成员是安全的并且二进制兼容的。将多个字段移动到一个新建的`oneof`内部，可能是安全的，如果你能确保没有代码会在某一时刻设置多个值。将任意字段移动到一个已经存在`oneof`中是不安全的。

# Any Message Type

Any message type 使你能够在不导入.proto文件定义的情况下，嵌入 message type。

一个 Any 包含了一个任意的序列化 message 作为 bytes，伴随一个用作确定 message type 的 URL（全局标识符）。

使用 Any 需要导入 `google/protobuf/any.proto`。

```
import "google/protobuf/any.proto";

message ErrorStatus {
	string message = 1;
	repeated google.protobuf.Any details = 2;
}
```

默认的 message type URL 的格式为`type.googleapis.com/_packagename_._messagename_`。

不同的 language 实现的不同的运行库来支持打包和解包 Any message。例如，在 go 中：

```
// Storing an arbitrary message type in Any.
var details = &NetworkErrorDetails{...}
var status = &ErrorStatus{...}
var any = &anypb.Any{}
any.MarshalFrom(details)
status.Details = append(status.Details, any)
```

**<u>*目前，用于 Any 的运行时库仍在开发中。*</u>**

# Oneof （share memory）

Oneof 字段类似于常规字段，除了在同一个 Oneof 中的所有字段共享内存，并且同一时间只可以设置一个字段。设置 Oneof 中的任意字段都会自动清楚其它所有字段。你可以使用特殊的`case()`或`WhichOneof`方法来检查设置了哪个字段。

## 使用

使用`oneof`关键字紧跟一个名称来定义一个 oneof:

```
message SampleMessage {
	oneof test_oneof {
		string name = 4;
		SubMessage sub_message = 9;
	}
}
```

**<u>*使用 oneof 就像是直接在 parent message 里直接定义字段，所以 field number range 和 field name 是共享的。*</u>**

**<u>*你可以在 oneof 中添加除了`map`和`repeated`的所有类型字段。*</u>**

在你的生成代码中，oneof 字段痛常规字段一样具有 getters 和 setters。你同样会得到一个用来检查哪个字段被设置了的特殊方法。

## 特性

- 设置一个 oneof 字段会自动清除所有其它 oneof 中的成员。如果设置多个 oneof 字段，只有最后一个字段会保留值：

  ```
  SampleMessage message;
  message.set_name("name");
  CHECK(message.has_name());
  message.mutable_sub_message();   // Will clear name field.
  CHECK(!message.has_name());
  ```

- 如果解析器遇到了多个属于同一个 oneof 的成员，仅最后一个在解析出的消息中可见。

- 一个 oneof 不能是 repeated。

- 反射 APIS 为 oneof 字段工作。

- 如果为 oneof 字段设置默认值（例如 int32 to 0），这个 oneof 字段的“case”会被设置，值也会被序列化到报文中。

- 如果你在使用c++,确保你的代码不会产生内存崩溃。

  ```
  SampleMessage message;
  SubMessage* sub_message = message.mutable_sub_message();
  message.set_name("name");      // Will delete sub_message
  sub_message->set_...            // Crashes here
  ```

- 还是C++，如果你`swap()`两个message中的oneof，两个message中的oneof字段会互换。

  ```
  SampleMessage msg1;
  msg1.set_name("name");
  SampleMessage msg2;
  msg2.mutable_sub_message();
  msg1.swap(&msg2);	// msg1 have sub_message, msg2 have name
  CHECK(msg1.has_sub_message());
  CHECK(msg2.has_name());
  ```

  ## 向后兼容性问题

  添加或删除 oneof 字段时要小心。如果检查 oneof 的值返回None/NOT_SET，这可能意味着 oneof 的值没有被设置，或者它被设置为 oneof 的不同版本的字段。没有办法区分它们的区别，因为没有办法知道报文中的未知字段是否属于 oneof。

  ### Tag Reuse 问题

  - 将字段移入或移除 oneof：在序列化和解析后，你可能会丢失某些信息（某些字段会被清理）。你可以安全的移动字段到新的 oneof，也可以移动多个字段，只要你确保只会设置其中一个。
  - 删除一个 oneof 字段并添加回来：这样可能会清除你的当前设置。
  - 分割和合并 oneof：同移动常规字段具有同样的问题。



# Map

```
map<key_type, value_type> map_field = N;
```

- key_type 可以是任意整数或字符串类型，即除了浮点数之外的任何标量和 bytes
- enum 不是有效的 key_type
- value_type 可以是除了 map 外的所有类型

例如，你想创建一个 Projects map，每一个 Project 都关联一个 string 关键字：

```
map<string, Project> projects = 3;
```

- map 字段不能是 repeated
- map 值在报文中的顺序和迭代顺序是未定义的
- 在为.proto生成文本格式时，map 是按键排序的
- 从报文解析或合并 map 时，如果有重复的键，则使用最后一个键值；从文本格式解析 map 时，如果存在重复的键，解析会失败
- 如果你提供一个键未提供键值，该键如何被序列化取决于所用 language。在C++，Kotlin，Java，Python中，默认值会被序列化；其它 language 中不会序列化

## 向后兼容

map 语法在报文中的格式相当于下面代码：

```
message MapFieldEntry {
  key_type key = 1;
  value_type value = 2;
}

repeated MapFieldEntry map_field = N;
```

所以不支持 map 的 protocol buffer 实现一样可以处理这些数据。

任何支持 map 的 protocol buffer 实现都必须生成和接受可以被上述定义接受的数据。

# Packages

```
package foo.bar;
message Open { ... }
```

```
message Foo {
  ...
  foo.bar.Open open = 1;
  ...
}
```

package specifier 影响生成代码的方式，取决于你选择的语言：

- C++: package specifier 作为 namespace
- Java 和 Kotlin：package specifier 作为 Java package，除非在.proto文件中指明`option java_package`选项
- Python： package specifier 被忽略，Python 模块通过在文件系统中的位置来识别
- Go：package specifier 作为 Go package name，除非在.proto文件中明确指定`option go_package`选项
- C#: package specifier 做 PascalCase 转换后作为 namespace，除非在.proto文件中明确指定`option csharp_namespace`

## Packages 和名称解析

protocol buffer 中的类型名解析的工作原理与C++相似：首先搜索最里面的作用域，然后是第二个最里面的作用域，以此类推，每个 package 都在其父 package 的内部。以"."开头的包名（.foo.bar.Baz）意味着从最外层作用域开始搜索。

protocol buffer 编译器通过解析导入的.proto文件来解析所有类型名。每种语言的代码生成器知道如何引用该语言中的每种类型，即使它有不同的作用域规则。

# Service

要在 RPC 系统中使用 Message type，可以在.proto文件中定义 RPC Service 接口，protocol buffer 编译器将用你选择的语言生成接口代码和存根。

```
service SearchService {
  rpc Search(SearchRequest) returns (SearchResponse);
}
```

使用 protocol buffer 的最直接的 RPC 系统是[gRPC](https://grpc.io/)。gRPC与 protocol buffer 得特别好，它允许你使用一个特殊的 protocol buffer 编译器插件直接从你的.proto文件生成相关的RPC代码。

如果不想使用 gRPC，可以使用你自己的 RPC 实现。查阅 [Proto2 Language Guide](https://developers.google.com/protocol-buffers/docs/proto#services).

protocol buffer 有很多正在开发的第三方 RPC 实现。

# Options 选项

可以使用`options`做个人声明。选项不会改变生命的总体含义，但可能会影响特定上下文中处理声明的方式。

可用选项的完整列表定义在`google/protobuf/descriptor.proto`。

选项的种类如下：

- 文件级选项，这意味着它们应该写入顶级作用域，而不是在任何 message、enum 或 service 定义中
- 消息级别的选项，这意味着它们应该在 message 定义中编写
- 字段级选项，这意味着它们应该在字段定义中编写

下面是一些最常用的选项：

- `java_package`(file option): 指定用于生成`Jave/Kotlin`类的 package。如果没有明确指定，将使用  proto package。如果不生成`Java/Kotlin`代码，此选项无效。

- `java_outer_classname` (file option): 

- `optimize_for` (file option)：可以设置`SPEED`, `CODE_SIZE`, or `LITE_RUNTIME`。将以以下方式影响c++和Java代码生成器(可能还有第三方生成器)：

  - `SPEED` (default): protocol buffer 编译器将生成用于序列化、解析和对消息类型执行其他常见操作的代码。这段代码经过了高度优化。
  - `CODE_SIZE`: protocol buffer 编译器将生成最小的类，并依赖于共享的、基于反射的代码来实现序列化、解析和各种其他操作。因此，生成的代码将比使用`SPEED`时小得多，但操作将会更慢。类将仍然实现与`SPEED`模式下完全相同的公共API。这种模式在包含大量.proto文件且不需要把所有文件都快速处理的应用程序中最有用。
  - `LITE_RUNTIME`: 编译器将生成只依赖于“lite”运行库的类(libprotobuf-lite而不是libprotobuf)。lite运行时比完整库小得多(大约小一个数量级)，但省略了某些特性，如描述符和反射。这对于运行在受限平台(如手机)上的应用程序尤其有用。编译器仍然会像在SPEED模式下那样生成所有方法的快速实现。生成的类将只在每种语言中实现MessageLite接口，它只提供完整Message接口方法的一个子集。

- `cc_enable_arenas` (file option): 为c++生成的代码启用 [arena allocation](https://developers.google.com/protocol-buffers/docs/reference/arenas) 。

- `deprecated` (field option): 如果设置为true，表示该字段已弃用，不应被新代码使用。在大多数语言中，这没有实际效果。在Java中，这变成了`@Deprecated`注释。将来，其他特定于语言的代码生成器可能会在字段的访问器上生成弃用注释，这反过来会导致在编译试图使用该字段的代码时发出警告。如果该字段没有被任何人使用，并且您希望阻止新用户使用它，请考虑用`reserved`语句替换该字段声明。

  ```
  int32 old_field = 6 [deprecated = true];
  ```

## 自定义 Options



# 生成代码

```sh
protoc --proto_path=IMPORT_PATH --cpp_out=DST_DIR --java_out=DST_DIR --python_out=DST_DIR --go_out=DST_DIR --ruby_out=DST_DIR --objc_out=DST_DIR --csharp_out=DST_DIR path/to/file.proto
```

- `IMPORT_PATH`: 指定查找.proto（import）文件的路径。如果不指定，默认值为当前目录。可以指定多次
- 必须提供一个或多个.proto文件作为输入。尽管文件的命名相对于当前目录，但每个文件必须驻留在一个`IMPORT_PATH`中，以便编译器可以确定它的规范名称。

# 编码

## Varints 可变字节整数

Varints 是一种用一个或多个字节序列化整数的方法。

在一个 varint 中的每个字节，除了最后一个字节，每个字节的最高有效位（most significant bit）msb 都会设置为1，用于标识还有更多的字节。每个字节的低7位用于存储该整数的二进制补码形式，小端字节序。

下面是整数1的例子。只需要一个字节，并且 msb 没有设置：

```
0000 0001
```

下面是整数300的例子，更复杂一些：

```
1010 1100 0000 0010
```

要将上例二进制表示还原为300，只需按照下列步骤：

1. 丢弃 msb 位

   ```
   010 1100 000 0010
   ```

2. 反转（小端字节序）两个低7位分组，然后连接起来：

   ```
   000 0010 010 1100
   -> 100101100
   -> 256 + 32 + 8 + 4 = 300
   ```

## 编码 Message

Message的二进制形式使用`field number`做key，每个字段的名称和声明的类型只能在解码端通过引用Message类型定义（.proto文件）来确定。

在Message被编码时，keys和values会连接成一个 byte stream。在Message被解码时，parser需要能够跳过其不识别的字段。这样就不会因为在Message中新增字段而破坏旧版程序的功能。为此，Message报文中的每一个key其实都包含两个值，`.proto`文件中的`field number`，及 wire type （为紧跟的value长度提供足够的信息）。在大多数语言实现中，这个key被称为tag。

可用的 wire type 如下：

| Type | Meaning          | Used For                                                 |
| :--- | :--------------- | :------------------------------------------------------- |
| 0    | Varint           | int32, int64, uint32, uint64, sint32, sint64, bool, enum |
| 1    | 64-bit           | fixed64, sfixed64, double                                |
| 2    | Length-delimited | string, bytes, embedded messages, packed repeated fields |
| 3    | Start group      | groups (deprecated)                                      |
| 4    | End group        | groups (deprecated)                                      |
| 5    | 32-bit           | fixed32, sfixed32, float                                 |

streamed Message 中的每个 key 都是一个 varint，值为 :

```
(field_number << 3) | writetype
```

可知，在 stream 中的第一个 number 总是 varint key。下面是 `08`，舍弃了 msb:

```
000 1000
```

从最后3位得出 wire type 为0，然后右移3位得到 field number 为1。如此可确定字段的编号为1，且紧跟的 value 为 varint。如紧跟的两字节如下，可计算出值为150：

```
96 01 = 1001 0110 0000 0001
-> 000 0001 001 0110
-> 10010110
-> 128 + 16 + 4 + 2 = 150
```

# 兼容性准则

在发布你自己的使用 protocol buffer 的代码之后，你迟早会想要“改进” protocol buffer 的定义。如果您希望你的新 buffers 是向后兼容的，而你的的旧 buffers 是向前兼容的，那么你需要遵循一些规则。

在新版的 protocol buffer 中：

- 不能更改任何现在字段的 tag number
- 可以删除字段
- 新增字段时使用全新的 tag number（例如，从未使用过的 tag number，不包括已删除字段的）