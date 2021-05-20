`node_exporter`揭露各种各样硬件和内核的指标。

# 安装和运行
`node_exporter`是一个单一的二进制文件，[下载](https://prometheus.io/download/)后直接解压运行：

```sh
wget https://github.com/prometheus/node_exporter/releases/download/v*/node_exporter-*.*-amd64.tar.gz
tar xvfz node_exporter-*.*-amd64.tar.gz
cd node_exporter-*.*-amd64
./node_exporter
```

`node_exporter`暴露的指标以`node`为前缀。

# 配置 Prometheus
需要正确配置本地运行的 **Prometheus** 实例，以便访问 Node 导出的指标。

```
scrape_configs:
- job_name: node
  static_configs:
  - targets: ['localhost:9100']
```
