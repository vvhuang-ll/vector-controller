# Vector-Controller 贡献指南

感谢您对 Vector-Controller 项目的关注！本文档将指导您如何为项目做出贡献。

## 目录

- [行为准则](#行为准则)
- [开发环境设置](#开发环境设置)
- [代码风格](#代码风格)
- [提交 Pull Request](#提交-Pull-Request)
- [代码审查流程](#代码审查流程)
- [发布流程](#发布流程)
- [社区交流](#社区交流)

## 行为准则

本项目采用 [Contributor Covenant](https://www.contributor-covenant.org/) 行为准则。参与者需要遵守开放、尊重、包容的社区环境。

## 开发环境设置

1. **Fork 并克隆仓库**

   ```bash
   git clone https://github.com/yourusername/vector-controller.git
   cd vector-controller
   ```

2. **安装依赖**

   确保您已安装以下工具：
   - Go 1.18+
   - Kubernetes 集群或 kind/minikube
   - Kubebuilder
   - Vector CLI

3. **初始化开发环境**

   ```bash
   # 安装Go依赖
   go mod tidy
   
   # 生成CRD代码
   make generate
   
   # 创建本地测试集群
   kind create cluster --name vector-dev
   ```

4. **运行测试**

   ```bash
   # 运行单元测试
   make test
   
   # 运行E2E测试
   make e2e-test
   ```

## 代码风格

Vector-Controller 项目遵循以下代码风格规范：

1. **Go 代码风格**：遵循 [Effective Go](https://golang.org/doc/effective_go) 和 [Go Code Review Comments](https://github.com/golang/go/wiki/CodeReviewComments)。

2. **提交信息**：使用清晰的提交信息，格式如下：
   ```
   组件: 简短描述（不超过50个字符）
   
   详细描述问题和解决方案。
   ```

3. **代码文档**：为所有公共函数和类型添加文档注释。

4. **测试覆盖率**：为新功能添加单元测试和集成测试。

## 提交 Pull Request

1. **创建分支**：从 `main` 分支创建新的功能分支。
   ```bash
   git checkout -b feature/your-feature-name
   ```

2. **提交更改**：在分支上进行更改并提交。
   ```bash
   git add .
   git commit -m "feat: add your feature"
   ```

3. **推送到 Fork**：
   ```bash
   git push origin feature/your-feature-name
   ```

4. **创建 Pull Request**：在 GitHub 上创建从您的分支到原始仓库 `main` 分支的 Pull Request。

5. **CI 检查**：确保所有 CI 检查通过。

6. **代码审查**：等待代码审查并根据反馈进行修改。

## 代码审查流程

1. 所有代码变更必须经过至少一名项目维护者的审查。
2. 代码审查关注点包括：功能正确性、代码质量、测试覆盖率、文档完整性。
3. 创建 PR 时，请详细描述所做的更改和测试方法。

## 发布流程

1. **版本号规范**：Vector-Controller 遵循 [语义化版本控制](https://semver.org/)。

2. **发布流程**：
   - 更新 CHANGELOG.md
   - 创建发布分支（如 `release-v1.0.0`）
   - 通过 GitHub Releases 创建发布标签
   - 构建并推送 Docker 镜像

## 社区交流

- **问题反馈**：使用 GitHub Issues
- **讨论**：加入 [Slack 频道](#) 或 [邮件列表](#)
- **用户文档**：更新 docs/ 目录

---

感谢您为 Vector-Controller 做出贡献！如有任何问题，请通过 Issues 或社区渠道联系我们。 