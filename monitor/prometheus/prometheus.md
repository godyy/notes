[TOC]



# 介绍

**Prometheus** 是一款开源的系统监控和警报工具。

特色：
- 多维度的数据模型
- 查询多维数据的灵活语言
- 不以来分布式存储，每个结点自治
- 时间序列通过 HTTP pull model 收集
- 可通过中间网关（pushgateway）推送时间序列
- 监控目标可以通过服务探索或静态配置被发现
- 多种方法支持图形化和表板化

# 资源
- 中文学习网站：[https://www.prometheus.wang/](https://www.prometheus.wang)

# 概念
##### 任务和实例
在 Prometheus 中，每一个暴露监控样本数据的HTTP服务称为一个实例。

任务（Job）是一组用于相同采集目的的实例。

[https://prometheus.io/docs/concepts/jobs_instances/](https://prometheus.io/docs/concepts/jobs_instances/)

##### Time Series
time series（时间序列）是按时间顺序进行排列的测量序列。

[https://grafana.com/docs/grafana/latest/getting-started/timeseries/](https://grafana.com/docs/grafana/latest/getting-started/timeseries/)

##### Metric Types（指标类型）
[https://prometheus.io/docs/concepts/metric_types/](https://prometheus.io/docs/concepts/metric_types/)

##### Histogram（直方图）
[https://grafana.com/docs/grafana/latest/getting-started/intro-histograms/](https://grafana.com/docs/grafana/latest/getting-started/intro-histograms/)

# 安装
## 通过编译好的二进制文件
下载最新的 [release版](https://prometheus.io/download/)，然后解压：
```sh
tar xvfz prometheus-*.tar.gz
cd prometheus-*
```

**Prometheus server** 是一个单独的叫做`prometheus`的二进制文件，我们可以通过`--help`执行它以查看帮助信息：
```sh
./prometheus --help
```

### 配置
**Prometheus** 配置文件采用 **YAML**格式，在解压出的文件中有一个名为`prometheus.yaml`的示例配置文件。

在示例配置文件中，有3个配置区块。

`global`控制全局配置，文件中有两个选择：
- scrape_interval 控制多久从目标拉去一次数据，可以针对单个目标重写
- evaluation_interval 控制多久评估一次规则，规则用于生成新的时间序列和警报

`rule_files`指定需要读取的规则文件的位置。

`scrap_configs`控制需要监控的资源。**Prometheus** 将自己的数据作为HTTP端点公开，因此它可以获取并监视自己的运行状况。Prometheus 期望通过目标上的路径`/metrics`拉取指标。

返回的时间序列数据将详细说明 Prometheus server 的状态和性能。

完整的配置选项说明[configuration documentation](https://prometheus.io/docs/operating/configuration)。

### 启动
切换到解压出的 Prometheus 目录，然后运行：
```sh
./prometheus --config.file=prometheus.yml
```

# 使用
`/graph`访问内置表达式浏览器。

在表达式输入框中输入表达式，然后执行来查询指标。[表达式语法](https://prometheus.io/docs/querying/basics/)。

**reload configuration** 发送`SIGHUP`到 **Prometheus** 进程，或者 **HTTP POST Request** 到 Prometheus 的`/-/reload`endpoint。