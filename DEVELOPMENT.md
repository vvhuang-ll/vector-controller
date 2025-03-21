# Vector-Controller 开发指南

本文档提供了开发 Vector-Controller 所需的详细信息和指导。

## 1. 开发环境准备

### 1.1 前置依赖

开发 Vector-Controller 需要以下工具和环境：

- **Go 1.18+**：主要开发语言
- **Docker**：用于构建和测试容器镜像
- **Kubernetes 集群**：用于测试部署（可使用 kind、minikube、k3d 等）
- **kubectl**：Kubernetes 命令行工具
- **Rancher Wrangler**：Kubernetes 控制器开发框架
- **Vector CLI**：用于验证配置文件
- **golangci-lint**：Go 代码静态分析工具

### 1.2 开发环境设置

按照以下步骤设置开发环境：

1. **安装 Go**

```bash
# 安装 Go 1.18 或更高版本
wget https://go.dev/dl/go1.18.linux-amd64.tar.gz
sudo tar -C /usr/local -xzf go1.18.linux-amd64.tar.gz
export PATH=$PATH:/usr/local/go/bin
```

2. **克隆代码仓库**

```bash
git clone https://github.com/your-org/vector-controller.git
cd vector-controller
```

3. **安装开发依赖**

```bash
# 安装 Vector CLI
curl --proto '=https' --tlsv1.2 -sSf https://sh.vector.dev | bash

# 安装 golangci-lint
curl -sSfL https://raw.githubusercontent.com/golangci/golangci-lint/master/install.sh | sh -s -- -b $(go env GOPATH)/bin v1.49.0

# 设置本地开发环境变量
export GO111MODULE=on
```

4. **设置本地开发集群**

```bash
# 使用 k3d 创建本地开发集群
k3d cluster create vector-dev

# 或使用 minikube
minikube start --driver=docker --memory=4g --cpus=2
```

## 2. 项目结构

Vector-Controller 项目使用以下目录结构：

```
vector-controller/
├── cmd/                       # 可执行程序入口
│   └── controller/            # 控制器命令
├── pkg/                       # 内部包
│   ├── apis/                  # API 定义
│   │   └── logging/           # logging API 组
│   │       └── v1alpha1/      # v1alpha1 版本API
│   ├── controllers/           # 控制器实现
│   │   ├── logflow/           # LogFlow 控制器
│   │   └── logoutput/         # LogOutput 控制器
│   ├── util/                  # 通用工具函数
│   └── vector/                # Vector 配置相关代码
├── manifests/                 # Kubernetes 部署清单
│   ├── crds/                  # CRD 定义
│   └── deployment/            # 控制器部署配置
├── hack/                      # 开发脚本
├── tests/                     # 测试资源和代码
│   ├── e2e/                   # 端到端测试
│   └── integration/           # 集成测试
├── Dockerfile                 # 容器镜像构建文件
├── go.mod                     # Go 模块定义
└── go.sum                     # Go 依赖哈希
```

## 3. 代码生成

Vector-Controller 使用 Wrangler 代码生成器创建 Kubernetes CRD 和客户端代码。

### 3.1 注释格式

API 定义使用特殊注释来指导代码生成：

```go
// +wrangler:crd:resource=logflows,kind=LogFlow,singular=logflow,plural=logflows,shortName=lf,scope=Namespaced
// +kubebuilder:printcolumn:name="Age",type="date",JSONPath=".metadata.creationTimestamp"
// +kubebuilder:printcolumn:name="Status",type="string",JSONPath=".status.conditions[?(@.type=='Ready')].status"
```

### 3.2 生成代码

在修改 API 定义后，运行以下命令更新生成的代码：

```bash
go generate ./...
```

生成的代码包括：

- CRD YAML 定义（放在 manifests/crds/ 目录）
- Go 类型的客户端代码（放在 pkg/generated/ 目录）

## 4. 控制器开发

### 4.1 控制器基础架构

Vector-Controller 使用 Wrangler 框架的 OnChange 和 OnRemove 事件机制实现控制器。控制器主要组件包括：

```go
// 控制器实现示例
type Controller struct {
    // 客户端依赖
    loggingClient   logging.Interface
    coreClient      corev1.Interface
    
    // 缓存和状态
    loggingCache    logging.LogFlowCache
    podCache        corev1.PodCache
    
    // 其他依赖
    vectorConfigGen *vector.ConfigGenerator
}

// 初始化控制器
func Register(ctx context.Context, clients *wrangler.Context) error {
    controller := &Controller{
        loggingClient:   clients.Logging,
        coreClient:      clients.Core,
        loggingCache:    clients.Logging.Cache(),
        podCache:        clients.Core.Cache(),
        vectorConfigGen: vector.NewConfigGenerator(),
    }
    
    // 注册事件处理程序
    clients.Logging.LogFlow().OnChange(ctx, "logflow-controller", controller.OnLogFlowChange)
    clients.Logging.LogFlow().OnRemove(ctx, "logflow-controller", controller.OnLogFlowRemove)
    
    return nil
}
```

### 4.2 事件处理

使用 Wrangler 的事件处理机制处理资源变更：

```go
// LogFlow 变更处理程序
func (c *Controller) OnLogFlowChange(key string, logFlow *v1alpha1.LogFlow) (*v1alpha1.LogFlow, error) {
    if logFlow == nil || logFlow.DeletionTimestamp != nil {
        return nil, nil
    }
    
    // 1. 获取相关资源
    outputs, err := c.getReferencedOutputs(logFlow)
    if err != nil {
        return c.updateLogFlowStatus(logFlow, err)
    }
    
    // 2. 生成Vector配置
    config, err := c.vectorConfigGen.GenerateConfig(logFlow, outputs)
    if err != nil {
        return c.updateLogFlowStatus(logFlow, err)
    }
    
    // 3. 应用配置
    if err := c.applyVectorConfig(config); err != nil {
        return c.updateLogFlowStatus(logFlow, err)
    }
    
    // 4. 更新状态
    return c.updateLogFlowStatus(logFlow, nil)
}

// LogFlow 删除处理程序
func (c *Controller) OnLogFlowRemove(key string, logFlow *v1alpha1.LogFlow) (*v1alpha1.LogFlow, error) {
    // 清理资源和配置
    return nil, c.cleanupVectorConfig(key)
}
```

### 4.3 状态更新

控制器负责维护 CRD 资源的状态：

```go
// 更新 LogFlow 状态
func (c *Controller) updateLogFlowStatus(logFlow *v1alpha1.LogFlow, opErr error) (*v1alpha1.LogFlow, error) {
    // 创建状态的副本
    logFlowCopy := logFlow.DeepCopy()
    
    // 根据操作结果设置状态
    if opErr != nil {
        meta.SetStatusCondition(&logFlowCopy.Status.Conditions, metav1.Condition{
            Type:    "Ready",
            Status:  metav1.ConditionFalse,
            Reason:  "ConfigurationFailed",
            Message: opErr.Error(),
        })
    } else {
        meta.SetStatusCondition(&logFlowCopy.Status.Conditions, metav1.Condition{
            Type:    "Ready",
            Status:  metav1.ConditionTrue,
            Reason:  "ConfigurationApplied",
            Message: "Vector configuration successfully applied",
        })
    }
    
    // 更新资源状态
    return c.loggingClient.LogFlow().UpdateStatus(logFlowCopy)
}
```

## 5. 测试

### 5.1 单元测试

为控制器和核心逻辑编写单元测试：

```bash
# 运行所有单元测试
go test ./pkg/...

# 运行特定包的测试
go test ./pkg/controllers/logflow/...

# 带覆盖率的测试
go test -coverprofile=coverage.out ./pkg/...
go tool cover -html=coverage.out
```

### 5.2 集成测试

集成测试验证控制器与其他组件的交互：

```bash
# 运行集成测试
go test ./tests/integration/...
```

### 5.3 端到端测试

端到端测试在实际 Kubernetes 集群中验证功能：

```bash
# 启动测试集群（如果尚未运行）
k3d cluster create test-cluster

# 运行端到端测试
go test ./tests/e2e/...
```

## 6. Vector 配置管理

### 6.1 配置生成

Vector 配置生成器负责将 CRD 规范转换为 Vector 配置：

```go
// 配置生成器示例
func (g *ConfigGenerator) GenerateConfig(logFlow *v1alpha1.LogFlow, outputs []*v1alpha1.LogOutput) (*VectorConfig, error) {
    config := NewVectorConfig()
    
    // 1. 为LogFlow添加Sources
    if err := g.addSources(config, logFlow); err != nil {
        return nil, err
    }
    
    // 2. 为LogFlow添加Transforms
    if err := g.addTransforms(config, logFlow); err != nil {
        return nil, err
    }
    
    // 3. 为LogOutputs添加Sinks
    if err := g.addSinks(config, logFlow, outputs); err != nil {
        return nil, err
    }
    
    return config, nil
}
```

### 6.2 配置验证

使用 Vector CLI 验证生成的配置：

```go
// 验证Vector配置
func (v *VectorManager) ValidateConfig(config *VectorConfig) error {
    // 将配置转换为TOML
    configData, err := config.ToTOML()
    if err != nil {
        return err
    }
    
    // 创建临时文件
    tmpFile, err := ioutil.TempFile("", "vector-config-*.toml")
    if err != nil {
        return err
    }
    defer os.Remove(tmpFile.Name())
    
    // 写入配置
    if _, err := tmpFile.Write(configData); err != nil {
        return err
    }
    tmpFile.Close()
    
    // 使用Vector CLI验证
    cmd := exec.Command("vector", "validate", "--no-environment", tmpFile.Name())
    output, err := cmd.CombinedOutput()
    if err != nil {
        return fmt.Errorf("invalid config: %s\n%s", err, output)
    }
    
    return nil
}
```

## 7. 构建和部署

### 7.1 构建控制器

构建 Vector-Controller 镜像：

```bash
# 构建Docker镜像
docker build -t vector-controller:latest .

# 推送到镜像仓库
docker tag vector-controller:latest your-registry/vector-controller:latest
docker push your-registry/vector-controller:latest
```

### 7.2 安装到集群

部署 Vector-Controller 到 Kubernetes 集群：

```bash
# 应用CRD
kubectl apply -f manifests/crds/

# 创建命名空间
kubectl create namespace vector-system

# 部署控制器
kubectl apply -f manifests/deployment/controller.yaml
```

### 7.3 部署示例资源

部署示例 LogFlow 和 LogOutput 资源：

```bash
# 部署示例LogOutput
kubectl apply -f examples/logoutput-elasticsearch.yaml

# 部署示例LogFlow
kubectl apply -f examples/logflow-nginx.yaml
```

## 8. 调试技巧

### 8.1 远程调试

使用 Delve 调试器进行远程调试：

```bash
# 在本地构建调试版本
go build -gcflags="all=-N -l" -o vector-controller ./cmd/controller

# 运行带调试器的二进制文件
dlv --listen=:2345 --headless=true --api-version=2 --accept-multiclient exec ./vector-controller
```

### 8.2 日志调试

提高日志级别获取更详细的信息：

```bash
# 通过环境变量设置日志级别
export LOG_LEVEL=debug
./vector-controller

# 或在部署配置中设置
env:
- name: LOG_LEVEL
  value: "debug"
```

### 8.3 使用VS Code进行远程调试

在 `.vscode/launch.json` 中配置远程调试：

```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Remote Debug",
            "type": "go",
            "request": "attach",
            "mode": "remote",
            "remotePath": "/app",
            "port": 2345,
            "host": "127.0.0.1"
        }
    ]
}
```

## 9. 常见开发任务

### 9.1 添加新的 CRD 字段

1. 在API类型定义中添加字段：

```go
// LogFlowSpec 定义了LogFlow的预期状态
type LogFlowSpec struct {
    // 新增字段示例
    BufferSize int `json:"bufferSize,omitempty"`
}
```

2. 更新 CRD 定义（通过运行代码生成）

3. 在控制器中处理新字段：

```go
// 在配置生成代码中处理新字段
if logFlow.Spec.BufferSize > 0 {
    config.Sources["my_source"].BufferSize = logFlow.Spec.BufferSize
}
```

### 9.2 添加新的输出类型

1. 添加新的输出类型常量：

```go
const (
    // 现有输出类型
    OutputTypeElasticsearch = "elasticsearch"
    OutputTypeKafka         = "kafka"
    
    // 新输出类型
    OutputTypeLoki          = "loki"
)
```

2. 实现新输出类型的配置生成：

```go
// 为Loki添加Sink配置
func (g *ConfigGenerator) addLokiSink(config *VectorConfig, output *v1alpha1.LogOutput) error {
    if output.Spec.Type != OutputTypeLoki {
        return nil
    }
    
    // 获取Loki特定配置
    lokiConfig := output.Spec.LokiConfig
    if lokiConfig == nil {
        return fmt.Errorf("loki configuration is required for loki output")
    }
    
    // 添加Sink配置
    sinkName := fmt.Sprintf("loki_%s", output.Name)
    config.Sinks[sinkName] = map[string]interface{}{
        "type":     "loki",
        "inputs":   []string{},  // 将在调用方填充
        "endpoint": lokiConfig.Endpoint,
        "encoding": map[string]string{
            "codec": "json",
        },
        "labels": lokiConfig.Labels,
    }
    
    return nil
}
```

### 9.3 调试Vector Agent配置

1. 获取Vector Agent Pod：

```bash
kubectl get pods -n vector-system -l app=vector-agent
```

2. 查看当前配置：

```bash
kubectl exec -n vector-system vector-agent-abc123 -- cat /etc/vector/vector.toml
```

3. 验证配置：

```bash
kubectl exec -n vector-system vector-agent-abc123 -- vector validate --no-environment /etc/vector/vector.toml
```

## 10. 持续集成

Vector-Controller 使用以下 CI 流程：

1. **提交前检查**：
   - 代码格式化：`go fmt ./...`
   - 代码静态分析：`golangci-lint run`
   - 单元测试：`go test ./pkg/...`

2. **PR 检查**：
   - 集成测试：`go test ./tests/integration/...`
   - 构建检查：`docker build -t vector-controller:test .`

3. **合并后流程**：
   - 构建和推送镜像
   - 运行端到端测试
   - 生成文档

## 11. 贡献指南

### 11.1 代码风格

- 遵循标准 Go 代码风格
- 使用 `golangci-lint` 进行代码检查
- 所有导出的函数和类型必须有文档注释

### 11.2 提交规范

使用语义化提交消息：

```
<type>(<scope>): <描述>

[正文]

[页脚]
```

类型包括：
- `feat`: 新功能
- `fix`: 错误修复
- `docs`: 文档更改
- `style`: 格式调整
- `refactor`: 代码重构
- `test`: 添加测试
- `chore`: 构建过程或辅助工具变动

### 11.3 PR流程

1. 创建功能分支
2. 实现更改
3. 编写/更新测试
4. 提交PR
5. 代码审查
6. 合并

## 12. 性能优化

### 12.1 缓存使用

利用 Wrangler 提供的缓存减少 API 服务器负载：

```go
// 使用缓存获取Pod
pod, err := c.podCache.Get(namespace, podName)
if err != nil {
    return err
}

// 批量获取LogOutputs
outputs, err := c.logOutputCache.List(namespace, labels.Everything())
```

### 12.2 批量处理

对多个资源进行批处理以减少更新次数：

```go
// 批量获取和处理资源
logFlows, err := c.logFlowCache.List(namespace, labels.Everything())
if err != nil {
    return err
}

// 批量生成配置
configs := make(map[string]*VectorConfig)
for _, logFlow := range logFlows {
    // 处理每个LogFlow
    config, err := c.generateConfigForLogFlow(logFlow)
    if err != nil {
        // 记录错误并继续
        log.Errorf("Error generating config for %s: %v", logFlow.Name, err)
        continue
    }
    configs[logFlow.Name] = config
}

// 合并配置并应用
mergedConfig := mergeConfigs(configs)
return c.applyConfig(mergedConfig)
```

### 12.3 并发处理

使用 Go 协程并发处理多个资源：

```go
// 并发处理多个LogFlows
var wg sync.WaitGroup
results := make(chan result, len(logFlows))

for _, logFlow := range logFlows {
    wg.Add(1)
    go func(lf *v1alpha1.LogFlow) {
        defer wg.Done()
        // 处理LogFlow
        config, err := c.generateConfigForLogFlow(lf)
        results <- result{
            name:   lf.Name,
            config: config,
            err:    err,
        }
    }(logFlow)
}

// 等待所有处理完成
wg.Wait()
close(results)

// 收集结果
configs := make(map[string]*VectorConfig)
for r := range results {
    if r.err != nil {
        log.Errorf("Error processing %s: %v", r.name, r.err)
        continue
    }
    configs[r.name] = r.config
}
```

## 13. 故障排除

### 13.1 常见问题解决

1. **CRD 创建失败**：
   - 检查 CRD 定义是否符合 Kubernetes APIServer 版本要求
   - 验证 CRD schema 是否有效

2. **控制器未响应资源变更**：
   - 检查控制器日志中的错误
   - 验证控制器服务账号权限
   - 确认事件处理程序是否正确注册

3. **Vector 配置无效**：
   - 使用 Vector 验证工具检查配置
   - 检查控制器生成的具体配置
   - 确认所有必需字段都已填充

### 13.2 诊断技巧

1. **查看控制器日志**：
   ```bash
   kubectl logs -n vector-system -l app=vector-controller -f
   ```

2. **查看事件**：
   ```bash
   kubectl get events -n vector-system
   ```

3. **检查资源状态**：
   ```bash
   kubectl describe logflow <name>
   kubectl describe logoutput <name>
   ```

## 14. API参考

### 14.1 LogFlow API

LogFlow CRD 定义日志采集配置。

**类型定义**：

```go
// LogFlow 定义日志采集规则
type LogFlow struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`
    
    Spec   LogFlowSpec   `json:"spec,omitempty"`
    Status LogFlowStatus `json:"status,omitempty"`
}

// LogFlowSpec 定义了LogFlow的预期状态
type LogFlowSpec struct {
    // 选择要收集日志的Pod
    Selector *metav1.LabelSelector `json:"selector,omitempty"`
    
    // 日志源配置
    Sources []LogSource `json:"sources"`
    
    // 日志变换配置
    Transforms []Transform `json:"transforms,omitempty"`
    
    // 输出引用
    OutputRefs []string `json:"outputRefs"`
}

// LogFlowStatus 定义了LogFlow的实际状态
type LogFlowStatus struct {
    // 条件显示了这个LogFlow的当前服务状态
    Conditions []metav1.Condition `json:"conditions,omitempty"`
    
    // 最后配置应用时间
    LastConfigurationTime *metav1.Time `json:"lastConfigurationTime,omitempty"`
}
```

### 14.2 LogOutput API

LogOutput CRD 定义日志输出目标。

**类型定义**：

```go
// LogOutput 定义日志输出目标
type LogOutput struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`
    
    Spec   LogOutputSpec   `json:"spec,omitempty"`
    Status LogOutputStatus `json:"status,omitempty"`
}

// LogOutputSpec 定义了LogOutput的预期状态
type LogOutputSpec struct {
    // 输出类型 (elasticsearch, kafka, loki, aws_s3, etc.)
    Type string `json:"type"`
    
    // 类型特定配置
    ElasticsearchConfig *ElasticsearchConfig `json:"elasticsearchConfig,omitempty"`
    KafkaConfig         *KafkaConfig         `json:"kafkaConfig,omitempty"`
    LokiConfig          *LokiConfig          `json:"lokiConfig,omitempty"`
    S3Config            *S3Config            `json:"s3Config,omitempty"`
}

// LogOutputStatus 定义了LogOutput的实际状态
type LogOutputStatus struct {
    // 条件显示了这个LogOutput的当前服务状态
    Conditions []metav1.Condition `json:"conditions,omitempty"`
}
```

## 15. 最佳实践

### 15.1 错误处理

使用结构化日志和详细错误信息：

```go
if err := doSomething(); err != nil {
    // 记录详细错误以便调试
    log.WithFields(log.Fields{
        "resource": resource.Name,
        "namespace": resource.Namespace,
        "error": err.Error(),
    }).Error("Failed to process resource")
    
    // 返回对用户友好的错误
    return nil, fmt.Errorf("failed to process resource: %w", err)
}
```

### 15.2 资源管理

遵循控制器最佳实践：

1. **幂等性**：确保多次运行相同操作产生相同结果
2. **失败恢复**：从临时故障中恢复的能力
3. **最小权限**：使用最小必要权限
4. **优雅终止**：捕获信号并优雅地关闭

### 15.3 CRD设计

遵循 Kubernetes API 设计最佳实践：

1. **声明式API**：描述期望状态而非操作步骤
2. **类型安全**：使用强类型而非任意映射
3. **版本控制**：计划API版本演进
4. **状态子资源**：使用状态子资源更新状态

## 16. 附录

### 16.1 常用命令参考

```bash
# 构建控制器
go build -o vector-controller ./cmd/controller

# 运行控制器（本地开发模式）
./vector-controller --kubeconfig ~/.kube/config

# 检查CRD状态
kubectl get crds | grep vector

# 查看所有LogFlows
kubectl get logflows --all-namespaces

# 查看所有LogOutputs
kubectl get logoutputs --all-namespaces
```

### 16.2 配置示例

**LogFlow 示例**：

```yaml
apiVersion: logging.vector.dev/v1alpha1
kind: LogFlow
metadata:
  name: nginx-logs
  namespace: default
spec:
  selector:
    matchLabels:
      app: nginx
  sources:
    - name: nginx_access
      type: file
      paths:
        - /var/log/nginx/access.log
  transforms:
    - name: parse_nginx
      type: regex_parser
      inputs: ["nginx_access"]
      pattern: '^(?P<remote_addr>\S+) - (?P<remote_user>\S+) \[(?P<time_local>[^\]]+)\] "(?P<request>[^"]*)" (?P<status>\d+) (?P<body_bytes_sent>\d+) "(?P<http_referer>[^"]*)" "(?P<http_user_agent>[^"]*)"$'
  outputRefs:
    - elasticsearch-logs
```

**LogOutput 示例**：

```yaml
apiVersion: logging.vector.dev/v1alpha1
kind: LogOutput
metadata:
  name: elasticsearch-logs
  namespace: default
spec:
  type: elasticsearch
  elasticsearchConfig:
    endpoint: https://elasticsearch-service:9200
    index: logs-%Y-%m-%d
    authentication:
      strategy: basic
      user: elastic
      passwordSecretRef:
        name: elasticsearch-credentials
        key: password
```

### 16.3 常用链接

- [Vector 文档](https://vector.dev/docs/)
- [Wrangler 文档](https://github.com/rancher/wrangler)
- [Kubernetes 控制器开发指南](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/)
- [Kubernetes API 约定](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api-conventions.md) 