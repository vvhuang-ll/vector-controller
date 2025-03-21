# Vector-Controller: Kubernetes原生日志采集系统

## 项目概述

Vector-Controller是一个基于Kubernetes的原生日志采集系统，它利用Vector作为日志采集Agent，通过CRD（自定义资源定义）实现声明式的日志管理。该系统能够实现日志的全自动发现、处理和分发。

### 核心功能

- **全自动发现**：动态感知Pod生命周期和存储配置，自动调整采集规则
- **CRD驱动管理**：通过CRD进行声明式管理，实现日志采集、变换和分发的一体化配置
- **元数据自动注入**：采集Pod级别日志时，自动补充Kubernetes相关元数据（Namespace、PodName、Labels等）
- **多后端支持**：支持Kafka、Elasticsearch、Loki、S3等多种日志存储后端
- **高可靠性**：配置校验、热加载、动态变更、异常处理，确保日志采集稳定运行

## 系统架构

### 整体架构

```
┌─────────────────────────┐
│      用户配置层         │
│  LogFlow/LogOutput CRD  │
└───────────┬─────────────┘
            ↓
┌─────────────────────────┐
│      控制平面          │
│    Vector Controller    │
└───────────┬─────────────┘
            ↓
┌─────────────────────────┐
│      数据平面          │
│     Vector Agent        │
└───────────┬─────────────┘
            ↓
┌─────────────────────────┐
│    日志存储后端         │
│ Kafka/ES/Loki/S3等      │
└─────────────────────────┘
```

### 关键组件

| 组件名称              | 作用                                            |
| --------------------- | ----------------------------------------------- |
| **LogFlow CRD**       | 定义日志采集规则（Source + Transform + Output） |
| **LogOutput CRD**     | 定义日志存储目标（Kafka、Elasticsearch等）      |
| **Vector Controller** | 监听CRD变更，解析Pod信息，生成Vector配置        |
| **Vector Agent**      | 运行在DaemonSet模式，采集并转发日志             |
| **日志后端**          | Kafka、Elasticsearch、S3、Loki等                |

## 使用指南

### 安装部署

```bash
# 克隆仓库
git clone https://github.com/yourusername/vector-controller.git

# 安装CRD
kubectl apply -f manifests/crds/

# 部署Controller
kubectl apply -f manifests/deployment/controller.yaml

# 部署Vector Agent
kubectl apply -f manifests/agent/vector-agent.yaml
```

### 创建LogFlow示例

```yaml
apiVersion: logging.vector.io/v1
kind: LogFlow
metadata:
  name: app-log-flow
  namespace: production
spec:
  # 1. 日志源配置
  sources:
    kubernetes_logs:
      type: kubernetes_logs
      extra_namespace_label_selector: "app=log"
  
  # 2. 元数据注入配置
  metadata:
    inject:
      - podName
      - namespace
      - podLabels
      - deployment
  
  # 3. 输出配置
  outputRef:
    name: kafka-output
    namespace: logging-system
```

### 创建LogOutput示例

```yaml
apiVersion: logging.vector.io/v1
kind: LogOutput
metadata:
  name: kafka-output
  namespace: logging-system
spec:
  sinks:
    kafka_output:
      type: kafka
      inputs: ["kubernetes_logs", "extract_k8s_metadata"]
      bootstrap_servers: "kafka-service:9092"
      topic: "k8s-logs-{{ kubernetes.pod_namespace }}"
      encoding:
        codec: json
      compression: "gzip"
      key_field: ".kubernetes.pod_name"
```

## 开发指南

### 环境准备

- Go 1.18+
- Kubernetes 1.19+
- Rancher Wrangler 工具集
- Vector CLI

### 代码结构

```
vector-controller/
├── pkg/
│   ├── apis/                       # API定义
│   │   └── logging.vector.io/      # API组
│   │       └── v1/                 # API版本
│   ├── generated/                  # 生成的客户端代码
│   ├── controllers/                # 控制器实现
│   ├── renderer/                   # 配置渲染
│   ├── metadata/                   # 元数据处理
│   └── validator/                  # 配置验证
├── cmd/
│   └── controller/                 # 主程序入口
├── manifests/                      # Kubernetes资源清单
│   ├── crds/                       # CRD定义
│   ├── deployment/                 # 控制器部署
│   └── agent/                      # Vector Agent配置
├── scripts/                        # 脚本文件
├── Makefile                        # 构建脚本
└── go.mod                         # Go模块定义
```

### 构建指南

```bash
# 生成代码和CRD
make generate

# 构建Controller镜像
make docker-build docker-push IMG=your-registry/vector-controller:tag

# 部署Controller
make deploy IMG=your-registry/vector-controller:tag
```

## 贡献指南

1. Fork本仓库
2. 创建特性分支 (`git checkout -b feature/amazing-feature`)
3. 提交更改 (`git commit -m 'Add some amazing feature'`)
4. 推送到分支 (`git push origin feature/amazing-feature`)
5. 打开Pull Request

## 许可证

本项目采用MIT许可证 - 详情见[LICENSE](LICENSE)文件 