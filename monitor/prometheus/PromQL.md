PromQL是Prometheus内置的数据查询语言，其提供对时间序列数据丰富的查询，聚合以及逻辑运算能力的支持。并且被广泛应用在Prometheus的日常应用当中，包括对数据查询、可视化、告警处理当中。可以这么说，PromQL是Prometheus所有应用场景的基础，理解和掌握PromQL是Prometheus入门的第一课。

# 1. 基础
## 1.1 查询时间序列
当Prometheus通过Exporter采集到相应的监控指标样本数据后，我们就可以通过PromQL对监控样本数据进行查询。

当我们直接使用监控指标名称查询时，可以查询该指标下的所有时间序列:
```
http_requests_total

or

http_requests_total{}
```

PromQL还支持用户根据时间序列的标签匹配模式来对时间序列进行过滤，目前主要支持两种匹配模式：完全匹配和正则匹配。

##### 完全匹配
PromQL支持使用=和!=两种完全匹配模式：
- 通过使用`label=value`可以选择那些标签满足表达式定义的时间序列
- 反之使用`label!=value`则可以根据标签匹配排除时间序列

例如，如果我们只需要查询所有http_requests_total时间序列中满足标签instance为localhost:9090的时间序列，则可以使用如下表达式：
```
http_requests_total{instance="localhost:9090"}
```

##### 正则匹配
除了使用完全匹配的方式对时间序列进行过滤以外，PromQL还可以支持使用正则表达式作为匹配条件，多个表达式之间使用|进行分离：
- 使用`label=~regx`表示选择那些标签符合正则表达式定义的时间序列
- 反之使用`label!~regx`进行排除

例如，如果想查询多个环节下的时间序列序列可以使用如下表达式：
```
http_requests_total{environment=~"staging|testing|development",method!="GET"}
```

## 1.2 范围查询
直接通过类似于PromQL表达式，例如`http_requests_total`查询时间序列时，返回值中只会包含该时间序列中的最新的一个样本值，这样的返回结果我们称之为**瞬时向量**。而相应的这样的表达式称之为——**瞬时向量表达式**。

而如果我们想过去一段时间范围内的样本数据时，我们则需要使用区间向量表达式。区间向量表达式和瞬时向量表达式之间的差异在于在区间向量表达式中我们需要定义时间选择的范围，时间范围通过时间范围选择器`[]`进行定义。例如，通过以下表达式可以选择最近5分钟内的所有样本数据：
```
http_request_total{}[5m]
```

通过区间向量表达式查询到的结果我们称为**区间向量**。

除了使用m表示分钟以外，PromQL的时间范围选择器支持其它时间单位：
- s - 秒
- m - 分钟
- h - 小时
- d - 天
- w - 周
- y - 年

## 1.3 时间位移操作
在瞬时向量表达式或者区间向量表达式中，都是以当前时间为基准。

而如果我们想查询，5分钟前的瞬时样本数据，或昨天一天的区间内的样本数据呢? 这个时候我们就可以使用位移操作，位移操作的关键字为`offset`。
```
http_request_total{} offset 5m
http_request_total{}[1d] offset 1d
```

## 1.4 使用聚合操作
般来说，如果描述样本特征的标签(label)在并非唯一的情况下，通过PromQL查询数据，会返回多条满足这些特征维度的时间序列。而PromQL提供的聚合操作可以用来对这些时间序列进行处理，形成一条新的时间序列:
```
# 查询系统所有http请求的总量
sum(http_request_total)

# 按照mode计算主机CPU的平均使用时间
avg(node_cpu) by (mode)

# 按照主机查询各个主机的CPU使用率
sum(sum(irate(node_cpu{mode!='idle'}[5m]))  / sum(irate(node_cpu[5m]))) by (instance)
```

## 1.5 标量和字符串

##### 标量（Scalar）：一个浮点型的数字值
标量只有一个数字，没有时序。
```
10
```

##### 字符串（String）：一个简单的字符串值
直接使用字符串，作为PromQL表达式，则会直接返回字符串。
```
"this is a string"
'these are unescaped: \n \\ \t'
`these are not unescaped: \n ' " \t`
```

## 1.6 合法的PromQL表达式
所有的PromQL表达式都必须至少包含一个指标名称(例如http_request_total)，或者一个不会匹配到空字符串的标签过滤器(例如{code="200"})。
```
http_request_total # 合法
http_request_total{} # 合法
{method="get"} # 合法

{job=~".*"} # 不合法
```

同时，除了使用`<metric name>{label=value}`的形式以外，我们还可以使用内置的__name__标签来指定监控指标名称：
```
{__name__=~"http_request_total"} # 合法
{__name__=~"node_disk_bytes_read|node_disk_bytes_written"} # 合法
```

# 2. 操作符
使用PromQL除了能够方便的按照查询和过滤时间序列以外，PromQL还支持丰富的操作符，用户可以使用这些操作符进一步地对时间序列进行二次加工。这些操作符包括：数学运算符，逻辑运算符，布尔运算符等等。

## 2.1 数学运算
例如，我们可以通过指标`node_memory_free_bytes_total`获取当前主机可用的内存空间大小，其样本单位为Bytes。如果客户端要求使用`MB`作为单位响应数据，那只需要将查询到的时间序列的样本值进行单位换算即可：
```
node_memory_free_bytes_total / (1024 * 1024)
```

当瞬时向量与标量之间进行数学运算时，数学运算符会依次作用域瞬时向量中的每一个样本值，从而得到一组新的时间序列。

而如果是瞬时向量与瞬时向量之间进行数学运算时，过程会相对复杂一点，例如：
```
node_disk_bytes_written + node_disk_bytes_read
```
那这个表达式是如何工作的呢？依次找到与左边向量元素匹配（标签完全一致）的右边向量元素进行运算，如果没找到匹配元素，则直接丢弃。同时新的时间序列将不会包含指标名称。

PromQL支持的所有数学运算符如下所示：
- \+    (加法)
- \-    (减法)
- \*    (乘法)
- /     (除法)
- %     (求余)
- ^     (幂运算)

## 2.2 使用布尔运算过滤时间序列
在PromQL通过标签匹配模式，用户可以根据时间序列的特征维度对其进行查询。而布尔运算则支持用户根据时间序列中样本的值，对时间序列进行过滤。

例如，系统管理员在排查问题的时候可能只想知道当前内存使用率超过95%的主机呢？通过使用布尔运算符可以方便的获取到该结果：
```
(node_memory_bytes_total - node_memory_free_bytes_total) / node_memory_bytes_total > 0.95
```

瞬时向量与标量进行布尔运算时，PromQL依次比较向量中的所有时间序列样本的值，如果比较结果为true则保留，反之丢弃。

瞬时向量与瞬时向量直接进行布尔运算时，同样遵循默认的匹配模式：依次找到与左边向量元素匹配（标签完全一致）的右边向量元素进行相应的操作，如果没找到匹配元素，则直接丢弃。

目前，Prometheus支持以下布尔运算符如下：
- == (相等)
- != (不相等)
- \> (大于)
- < (小于)
- \>= (大于等于)
- <= (小于等于)

## 2.3 使用bool修饰符改变布尔运算符的行为
布尔运算符的默认行为是对时序数据进行过滤。而在其它的情况下我们可能需要的是真正的布尔结果。这时可以使用`bool`修饰符改变布尔运算的默认行为。 例如：
```
http_requests_total > bool 1000
```

同时需要注意的是，如果是在两个标量之间使用布尔运算，则必须使用bool修饰符:
```
2 == bool 2 # 结果为1
```

## 2.4 使用集合运算符
使用瞬时向量表达式能够获取到一个包含多个时间序列的集合，我们称为瞬时向量。 通过集合运算，可以在两个瞬时向量与瞬时向量之间进行相应的集合操作。目前，Prometheus支持以下集合运算符：
- and (并且)
- or (或者)
- unless (排除)

`vector1 and vector2` 会产生一个由vector1的元素组成的新的向量。该向量包含vector1中完全匹配vector2中的元素组成。

`vector1 or vector2` 会产生一个新的向量，该向量包含vector1中所有的样本数据，以及vector2中没有与vector1匹配到的样本数据。

`vector1 unless vector2` 会产生一个新的向量，新向量中的元素由vector1中没有与vector2匹配的元素组成。

## 2.5 操作符优先级
在PromQL操作符中优先级由高到低依次为：
- ^
- *, /, %
- +, -
- ==, !=, <=, <, >=, >
- and, unless
- or

## 2.6 匹配模式详解
向量与向量之间进行运算操作时会基于默认的匹配规则：依次找到与左边向量元素匹配（**标签完全一致**）的右边向量元素进行运算，如果没找到匹配元素，则直接丢弃。

PromQL中有两种典型的匹配模式：一对一（one-to-one）,多对一（many-to-one）或一对多（one-to-many）。

### 2.6.1 一对一匹配
一对一匹配模式会从操作符两边表达式获取的瞬时向量依次比较并找到唯一匹配(标签完全一致)的样本值。默认情况下，使用表达式：
```
vector1 <operator> vector2
```

在操作符两边表达式标签不一致的情况下，可以使用`on(label list)`或者`ignoring(label list)`来修改便签的匹配行为。使用ignoreing可以在匹配时忽略某些便签，而on则用于将匹配行为限定在某些便签之内。
```
<vector expr> <bin-op> ignoring(<label list>) <vector expr>
<vector expr> <bin-op> on(<label list>) <vector expr>
```

### 2.6.2 多对一和一对多
多对一和一对多两种匹配模式指的是“一”侧的每一个向量元素可以与"多"侧的多个元素匹配的情况。在这种情况下，必须使用group修饰符：`group_left`或者`group_right`来确定哪一个向量具有更高的基数（充当“多”的角色）。
```
<vector expr> <bin-op> ignoring(<label list>) group_left(<label list>) <vector expr>
<vector expr> <bin-op> ignoring(<label list>) group_right(<label list>) <vector expr>
<vector expr> <bin-op> on(<label list>) group_left(<label list>) <vector expr>
<vector expr> <bin-op> on(<label list>) group_right(<label list>) <vector expr>
```

多对一和一对多两种模式一定是出现在操作符两侧表达式返回的向量标签不一致的情况。因此需要使用ignoring和on修饰符来排除或者限定匹配的标签列表。

> 提醒：group修饰符只能在比较和数学运算符中使用。在逻辑运算and,unless和or中默认与右向量中的所有元素进行匹配。

# 3. 聚合操作
Prometheus还提供了下列内置的聚合操作符，**这些操作符作用于瞬时向量**。可以将瞬时表达式返回的样本数据进行聚合，形成一个新的时间序列。
- `sum` (求和)
- `min` (最小值)
- `max` (最大值)
- `avg` (平均值)
- `stddev` (标准差)
- `stdvar` (标准差异)
- `count` (计数)
- `count_values` (对value进行计数)
- `bottomk` (后n条时序，升序)
- `topk` (前n条时序，降序)
- `quantile` (分布统计)

使用聚合操作的语法如下：
```
<aggr-op>([parameter,] <vector expression>) [without|by (<label list>)]
or
<aggr-op> [without|by (<label list>)] ([parameter,] <vector expression>) 
```
其中只有`count_values`, `quantile`, `topk`, `bottomk`支持参数(parameter)。

`without`用于从计算结果中移除列举的标签，而保留其它标签。`by`则正好相反，结果向量中只保留列出的标签，其余标签则移除。通过`without`和`by`可以按照样本的问题对数据进行聚合。

`label list`是一组未加引号的标签，它们的末尾可能包含逗号，即`(label1, label2)`和`(label1, label2)`都是有效的语法。

`count_values`用于时间序列中每一个样本值出现的次数。count_values会为每一个唯一的样本值输出一个时间序列，并且每一个时间序列包含一个额外的标签。
```
count_values("count", http_requests_total)
```

`quantile`用于计算当前样本数据值的分布情况`quantile(φ, express)`其中`0 ≤ φ ≤ 1`。例如，当φ为0.5时，即表示找到当前样本数据中的中位数：
```
quantile(0.5, http_requests_total)
```

# 4. 内置函数
Prometheus提供了大量的内置函数，可以对时序数据进行丰富的处理。下面介绍一下一些常用的内置函数以及相关的使用场景和用法。

## 4.1 计算Counter指标增长率
`increase(v range-vector)`函数是PromQL中提供的众多内置函数之一。其中参数v是一个区间向量，increase函数获取区间向量中的第一个和最后一个样本并返回其增长量。因此，可以通过以下表达式Counter类型指标的增长率:

```
increase(node_cpu[2m]) / 120
```

除了使用increase函数以外，PromQL中还直接内置了`rate(v range-vector)`函数，rate函数可以直接计算区间向量v在时间窗口内**平均增长速率**。因此，通过以下表达式可以得到与前面例子相同的结果：
```
rate(node_cpu[2m])
```

需要注意的是使用rate或者increase函数去计算样本的平均增长速率，容易陷入“长尾问题”当中，其无法反应在时间窗口内样本数据的突发变化。 例如，对于主机而言在2分钟的时间窗口内，可能在某一个由于访问量或者其它问题导致CPU占用100%的情况，但是通过计算在时间窗口内的平均增长率却无法反应出该问题。

为了解决该问题，PromQL提供了另外一个灵敏度更高的函数`irate(v range-vector)`。irate同样用于计算区间向量的增长率，但是其反应出的是**瞬时增长率**。irate函数是通过区间向量中最后两个样本数据来计算区间向量的增长速率。这种方式可以避免在时间窗口范围内的“长尾问题”，并且体现出更好的灵敏度，通过irate函数绘制的图表能够更好的反应样本数据的瞬时变化状态。
```
irate(node_cpu[2m])
```

irate函数相比于rate函数提供了更高的灵敏度，不过当需要分析长期趋势或者在告警规则中，irate的这种灵敏度反而容易造成干扰。**因此在长期趋势分析或者告警中更推荐使用rate函数**。

## 4.2 预测Gauge指标变化趋势
在一般情况下，系统管理员为了确保业务的持续可用运行，会针对服务器的资源设置相应的告警阈值。例如，当磁盘空间只剩512MB时向相关人员发送告警通知。 这种基于阈值的告警模式对于当资源用量是**平滑增长**的情况下是能够有效的工作的。 但是如果资源不是平滑变化的呢？ 比如有些某些业务增长，存储空间的增长速率提升了高几倍。这时，如果基于原有阈值去触发告警，当系统管理员接收到告警以后可能还没来得及去处理问题，系统就已经不可用了。 因此阈值通常来说不是固定的，需要定期进行调整才能保证该告警阈值能够发挥去作用。 那么还有没有更好的方法吗？

PromQL中内置的`predict_linear(v range-vector, t scalar)` 函数可以帮助系统管理员更好的处理此类情况，predict_linear函数可以预测时间序列v在t秒后的值。它基于简单线性回归的方式，对时间窗口内的样本数据进行统计，从而可以对时间序列的变化趋势做出预测。例如，基于2小时的样本数据，来预测主机可用磁盘空间的是否在4个小时候被占满，可以使用如下表达式：
```
predict_linear(node_filesystem_free{job="node"}[2h], 4 * 3600) < 0
```

## 4.3 统计Histogram指标的分位数
`Histogram`和`Summary`都可以用于统计和分析数据的分布情况。区别在于Summary是直接在客户端计算了数据分布的分位数情况。而Histogram的分位数计算需要通过`histogram_quantile(φ float, b instant-vector)`函数进行计算。其中`φ（0<=φ<=1）`表示需要计算的分位数，如果需要计算中位数φ取值为0.5，以此类推即可。

通过对Histogram类型的监控指标，用户可以轻松获取样本数据的分布情况。同时分位数的计算，也可以非常方便的用于评判当前监控指标的服务水平。

需要注意的是通过histogram_quantile计算的分位数，并非为精确值，而是通过http_request_duration_seconds_bucket和http_request_duration_seconds_sum近似计算的结果。

## 4.4 动态标签替换
可以通过`label_replace`函数为时间序列添加额外的标签。label_replace的具体参数如下：
```
label_replace(v instant-vector, dst_label string, replacement string, src_label string, regex string)
```
该函数会依次对`v`中的每一条时间序列进行处理，通过`regex`匹配`src_label`的值，并将匹配部分`relacement`写入到`dst_label`标签中。例如：
```
up{instance="localhost:8080",job="cadvisor"}    1
up{instance="localhost:9090",job="prometheus"}    1
up{instance="localhost:9100",job="node"}    1

label_replace(up, "host", "$1", "instance",  "(.*):.*")

up{host="localhost",instance="localhost:8080",job="cadvisor"}    1
up{host="localhost",instance="localhost:9090",job="prometheus"}    1
up{host="localhost",instance="localhost:9100",job="node"} 1
```

Prometheus还提供了`label_join`函数，该函数可以将时间序列中多个标签`src_label`的值，通过`separator`作为连接符写入到一个新的标签`dst_label`中:
```
label_join(v instant-vector, dst_label string, separator string, src_label_1 string, src_label_2 string, ...)
```

label_replace和label_join函数提供了对时间序列标签的自定义能力，从而能够更好的与客户端或者可视化工具配合。

## 4.5 其它内置函数
[PromQL Functions](https://prometheus.io/docs/prometheus/latest/querying/functions)

# 5. 在HTTP API中使用PromQL
Prometheus当前稳定的HTTP API可以通过`/api/v1`访问。

通用占位符的定义如下：
- `<rfc3339 | unix_timestamp>`: 输入时间戳可以以[RFC3339](https://www.ietf.org/rfc/rfc3339.txt)格式提供，也可以以Unix时间戳(以秒为单位)提供，可选的小数位数用于亚秒级精度。输出时间戳总是以Unix时间戳表示。
- `<series_selector>`: Prometheus[时间序列选择器](https://prometheus.io/docs/prometheus/latest/querying/basics/#time-series-selectors), 如http_requests_total或http_requests_total{method=~"(GET|POST)}，需要url编码。
- `<duration>`: Prometheus[时间字符串](https://prometheus.io/docs/prometheus/latest/querying/basics/#time_durations)。例如，5m表示5分钟的持续时间。
- `<bool>`: 布尔值(字符串`true`和`false`)。

> 可以重复的查询参数名称，以[]结尾

## 5.1 API 响应格式
Prometheus API使用了JSON格式的响应内容。当API调用成功后将会返回2xx的HTTP状态码。

反之，当API调用失败时可能返回以下几种不同的HTTP状态码：
- 404 Bad Request：当参数错误或者缺失时。
- 422 Unprocessable Entity 当表达式无法执行时。
- 503 Service Unavailiable 当请求超时或者被中断时。
- 在到达API端点之前发生的错误可能会返回其他非2xx代码。

所有的API响应均使用以下的JSON格式：
```json
{
  "status": "success" | "error",
  "data": <data>,

  // Only set if status is "error". The data field may still hold
  // additional data.
  "errorType": "<string>",
  "error": "<string>"
  
  // if there are errors that do not inhibit the request execution.
  // Only if there were warnings while executing the request.
  // There will still be data in the data field.
  "warnings": ["<string>"]
}
```

## 5.2 瞬时数据查询
通过使用`/api/v1/query`我们可以查询PromQL在特定时间点下的计算结果。
```
GET /api/v1/query

URL请求参数：
    query=：PromQL表达式。
    time=：用于指定用于计算PromQL的时间戳。可选参数，默认情况下使用当前系统时间。
    timeout=：超时设置。可选参数，默认情况下使用-query,timeout的全局设置。
```

## 5.3 响应数据类型
当API调用成功后，Prometheus会返回JSON格式的响应内容，并且在data节点中返回查询结果。data节点格式如下：
```
{
  "resultType": "matrix" | "vector" | "scalar" | "string",
  "result": <value>
}
```

- 瞬时向量: vector, result格式如下，value只包含一个唯一的样本：
```
[
  {
    "metric": { "<label_name>": "<label_value>", ... },
    "value": [ <unix_time>, "<sample_value>" ]
  },
  ...
]
```

- 区间向量：matrix，result格式如下，values包含当前时间序列的一组样本：
```
[
  {
    "metric": { "<label_name>": "<label_value>", ... },
    "values": [ [ <unix_time>, "<sample_value>" ], ... ]
  },
  ...
]
```

- 标量: scalar，result格式如下，由于标量不存在时间序列一说，因此result表示为当前系统时间一个标量的值：
```
[ <unix_time>, "<scalar_value>" ]
```

- 字符串：string，result格式如下，响应内容格式同标量相同：
```
[ <unix_time>, "<string_value>" ]
```

## 5.4 区间数据查询
使用`/api/v1/query_range`我们则可以直接查询PromQL表达式在一段时间返回内的计算结果。
```
GET /api/v1/query_range

URL请求参数：
    query=: PromQL表达式。
    start=: 起始时间。
    end=: 结束时间。
    step=: 查询步长，duration格式或浮点数（秒）。
    timeout=: 超时设置。可选参数，默认情况下使用-query,timeout的全局设置。
```
> 需要注意的是，在QUERY_RANGE API中PromQL只能使用瞬时向量选择器类型的表达式。

## 5.5 More
[Prometheus HTTP API](https://prometheus.io/docs/prometheus/latest/querying/api/)

# 示例
- 主机的CPU使用率：`100 * (1 - avg (irate(node_cpu{mode='idle'}[5m])) by(job) )`