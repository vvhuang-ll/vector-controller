# Vector-Controller 使用示例

本目录包含了Vector-Controller的使用示例，演示如何配置和使用LogFlow和LogOutput CRD来管理Kubernetes集群中的日志采集。

## 目录结构

```
examples/
├── logflow/               # LogFlow示例
│   ├── basic.yaml         # 基础LogFlow配置
│   ├── file-logs.yaml     # 文件日志采集
│   ├── host-metrics.yaml  # 主机指标采集
│   └── multi-source.yaml  # 多源采集
├── logoutput/             # LogOutput示例
│   ├── kafka.yaml         # Kafka输出配置
│   ├── elasticsearch.yaml # Elasticsearch输出配置
│   ├── loki.yaml          # Loki输出配置
│   └── s3.yaml            # S3输出配置
└── README.md              # 本文档
```

## 快速开始

### 1. 部署Vector-Controller

在使用示例前，请确保已经部署了Vector-Controller：

```bash
# 安装CRD
kubectl apply -f ../config/crd/bases/

# 部署Controller
kubectl apply -f ../config/manager/manager.yaml

# 部署Vector Agent
kubectl apply -f ../config/agent/vector-agent.yaml
```

### 2. 创建LogOutput资源

首先创建一个LogOutput资源，定义日志的存储目标：

```bash
# 创建Kafka输出
kubectl apply -f logoutput/kafka.yaml

# 或创建Elasticsearch输出
kubectl apply -f logoutput/elasticsearch.yaml
```

### 3. 创建LogFlow资源

然后创建LogFlow资源，定义日志采集规则：

```bash
# 创建基础LogFlow，采集所有Pod日志
kubectl apply -f logflow/basic.yaml

# 或创建特定的文件日志采集配置
kubectl apply -f logflow/file-logs.yaml
```

### 4. 验证配置

检查LogFlow和LogOutput状态：

```bash
# 查看LogFlow状态
kubectl get logflows -A

# 查看LogOutput状态
kubectl get logoutputs -A
```

检查生成的Vector配置：

```bash
# 查看Vector ConfigMap
kubectl get configmap -n vector-system vector-config -o yaml
```

检查Vector Agent日志：

```bash
# 查看Vector Agent日志
kubectl logs -n vector-system -l app=vector-agent
```

## 示例详解

### LogFlow示例

#### 基础日志采集 (basic.yaml)

此示例展示了最基本的LogFlow配置，采集所有命名空间中的Pod日志：

```yaml
apiVersion: logging.vector.io/v1
kind: LogFlow
metadata:
  name: basic-logflow
  namespace: default
spec:
  sources:
    kubernetes_logs:
      type: kubernetes_logs  # 使用Vector内置的kubernetes_logs源
  
  metadata:
    inject:
      - podName
      - namespace
      - podLabels
      - nodeName
  
  outputRef:
    name: kafka-output
    namespace: logging-system
```

#### 文件日志采集 (file-logs.yaml)

此示例展示如何采集特定路径的文件日志：

```yaml
apiVersion: logging.vector.io/v1
kind: LogFlow
metadata:
  name: file-logflow
  namespace: default
spec:
  sources:
    nginx_logs:
      type: file
      config:
        include: ["/var/log/nginx/*.log"]
        read_from: "beginning"
  
  metadata:
    inject:
      - nodeName
  
  outputRef:
    name: elasticsearch-output
    namespace: logging-system
```

#### 主机指标采集 (host-metrics.yaml)

此示例展示如何采集主机指标：

```yaml
apiVersion: logging.vector.io/v1
kind: LogFlow
metadata:
  name: host-metrics-logflow
  namespace: monitoring
spec:
  sources:
    host_metrics:
      type: host_metrics
      config:
        scrape_interval_secs: 60
        filesystem:
          devices:
            excludes: ["binfmt_misc"]
  
  outputRef:
    name: prometheus-output
    namespace: monitoring
```

#### 多源采集 (multi-source.yaml)

此示例展示如何在一个LogFlow中配置多个数据源：

```yaml
apiVersion: logging.vector.io/v1
kind: LogFlow
metadata:
  name: multi-source-logflow
  namespace: default
spec:
  sources:
    # Kubernetes容器日志
    kubernetes_logs:
      type: kubernetes_logs
      config:
        extra_namespace_label_selector: "app=webapp"
    
    # 系统日志文件
    system_logs:
      type: file
      config:
        include: ["/var/log/syslog"]
        read_from: "beginning"
    
    # 主机指标
    host_metrics:
      type: host_metrics
      config:
        scrape_interval_secs: 30
  
  metadata:
    inject:
      - podName
      - namespace
      - nodeName
  
  outputRef:
    name: loki-output
    namespace: logging-system
```

### LogOutput示例

#### Kafka输出 (kafka.yaml)

将日志发送到Kafka：

```yaml
apiVersion: logging.vector.io/v1
kind: LogOutput
metadata:
  name: kafka-output
  namespace: logging-system
spec:
  sinks:
    kafka_sink:
      type: kafka
      inputs: ["*"]  # 接收所有输入
      config:
        bootstrap_servers: "kafka-broker:9092"
        topic: "k8s-logs-{{ kubernetes.namespace }}"
        encoding:
          codec: json
        compression: "gzip"
        key_field: ".kubernetes.pod_name"
```

#### Elasticsearch输出 (elasticsearch.yaml)

将日志发送到Elasticsearch：

```yaml
apiVersion: logging.vector.io/v1
kind: LogOutput
metadata:
  name: elasticsearch-output
  namespace: logging-system
spec:
  sinks:
    es_sink:
      type: elasticsearch
      inputs: ["*"]
      config:
        endpoint: "http://elasticsearch-master:9200"
        index: "logs-%Y-%m-%d"
        bulk:
          action: "create"
```

#### Loki输出 (loki.yaml)

将日志发送到Loki：

```yaml
apiVersion: logging.vector.io/v1
kind: LogOutput
metadata:
  name: loki-output
  namespace: logging-system
spec:
  sinks:
    loki_sink:
      type: loki
      inputs: ["*"]
      config:
        endpoint: "http://loki-gateway:80"
        encoding:
          codec: json
        labels:
          namespace: "{{ kubernetes.namespace }}"
          pod: "{{ kubernetes.pod_name }}"
          app: "{{ kubernetes.pod_labels.app }}"
```

#### S3输出 (s3.yaml)

将日志归档到S3：

```yaml
apiVersion: logging.vector.io/v1
kind: LogOutput
metadata:
  name: s3-output
  namespace: logging-system
spec:
  sinks:
    s3_sink:
      type: aws_s3
      inputs: ["*"]
      config:
        bucket: "logs-archive"
        region: "us-west-2"
        key_prefix: "k8s-logs/{{ kubernetes.namespace }}/{{ kubernetes.pod_name }}"
        compression: gzip
        encoding:
          codec: json
```

## 高级配置

### 使用变换器处理日志

以下示例展示如何使用变换器处理日志内容：

```yaml
apiVersion: logging.vector.io/v1
kind: LogFlow
metadata:
  name: transform-logflow
  namespace: default
spec:
  sources:
    app_logs:
      type: kubernetes_logs
      config:
        extra_label_selector: "app=myapp"
  
  # 这部分通常由控制器自动生成，但也可以手动配置
  transforms:
    parse_json:
      type: remap
      inputs: ["app_logs"]
      source: |
        . = parse_json!(.message)
    
    filter_errors:
      type: filter
      inputs: ["parse_json"]
      condition: '.level == "error"'
  
  outputRef:
    name: alerting-output
    namespace: monitoring
```

### 配置多个输出

以下示例展示如何配置多个输出目标：

```yaml
apiVersion: logging.vector.io/v1
kind: LogOutput
metadata:
  name: multi-output
  namespace: logging-system
spec:
  sinks:
    # 所有日志存储到S3
    s3_all:
      type: aws_s3
      inputs: ["*"]
      config:
        bucket: "logs-archive"
        region: "us-west-2"
    
    # 错误日志发送到Kafka告警主题
    kafka_errors:
      type: kafka
      inputs: ["filter_errors"]  # 引用上游的过滤变换器
      config:
        bootstrap_servers: "kafka-broker:9092"
        topic: "error-alerts"
    
    # 指标数据发送到Prometheus
    prometheus:
      type: prometheus_exporter
      inputs: ["host_metrics"]
      config:
        address: "0.0.0.0:9090"
```

## 故障排除

如果遇到问题，请检查以下内容：

1. 确认CRD已成功安装：
   ```bash
   kubectl api-resources | grep vector.io
   ```

2. 检查LogFlow和LogOutput的状态：
   ```bash
   kubectl describe logflow <name> -n <namespace>
   kubectl describe logoutput <name> -n <namespace>
   ```

3. 检查Vector Controller日志：
   ```bash
   kubectl logs -n vector-system deployment/vector-controller-manager
   ```

4. 检查Vector Agent日志：
   ```bash
   kubectl logs -n vector-system daemonset/vector-agent
   ```

5. 确认生成的Vector配置是否正确：
   ```bash
   kubectl get configmap -n vector-system vector-config -o yaml
   ``` 