某些PromQL较为复杂且计算量较大时，直接使用PromQL可能会导致Prometheus响应超时的情况。这时需要一种能够类似于后台批处理的机制能够在后台完成这些复杂运算的计算，对于使用者而言只需要查询这些运算结果即可。Prometheus通过Recoding Rule规则支持这种后台计算的方式，可以实现对复杂查询的性能优化，提高查询效率。

# 定义 recording rules

在Prometheus配置文件中，通过rule_files定义recoding rule规则文件的访问路径。

```yaml
rule_files:
  [ - <filepath_glob> ... ]
```

每一个规则文件通过以下格式进行定义：

```yaml
groups:
  [ - <rule_group> ]
```

一个简单的规则文件可能是这个样子的：

```yaml
groups:
  - name: example
    rules:
    - record: job:http_inprogress_requests:sum
      expr: sum(http_inprogress_requests) by (job)
```

rule_group的具体配置项如下所示：

```yaml
# The name of the group. Must be unique within a file.
name: <string>

# How often rules in the group are evaluated.
[ interval: <duration> | default = global.evaluation_interval ]

rules:
  [ - <rule> ... ]
```

与告警规则一致，一个group下可以包含多条规则rule。

```yaml
# The name of the time series to output to. Must be a valid metric name.
record: <string>

# The PromQL expression to evaluate. Every evaluation cycle this is
# evaluated at the current time, and the result recorded as a new set of
# time series with the metric name as given by 'record'.
expr: <string>

# Labels to add or overwrite before storing the result.
labels:
  [ <labelname>: <labelvalue> ]
```

根据规则中的定义，Prometheus会在后台完成expr中定义的PromQL表达式计算，并且将计算结果保存到新的时间序列`record`中。同时还可以通过labels为这些样本添加额外的标签。

