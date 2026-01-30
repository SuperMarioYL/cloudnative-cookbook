---
title: "开发环境搭建"
weight: 1
---
本文详细介绍 Kubernetes 开发环境的搭建过程，包括系统要求、代码编译、IDE 配置和测试环境准备。

## 1. 系统要求与依赖

### 1.1 硬件要求

| 资源 | 最低配置 | 推荐配置 |
|-----|---------|---------|
| CPU | 4 核 | 8 核以上 |
| 内存 | 8 GB | 16 GB 以上 |
| 磁盘 | 50 GB SSD | 100 GB SSD |
| 网络 | 稳定连接 | 可访问 GitHub |

### 1.2 Go 版本要求

Kubernetes 1.32 要求 Go 1.25+：

```bash
# 检查 Go 版本
go version
# 预期输出: go version go1.25.x darwin/arm64 (或其他平台)

# 查看项目指定的 Go 版本
cat .go-version
# 输出: 1.25.6
```

### 1.3 必需工具

```bash
# macOS 安装依赖
brew install go git make docker rsync jq coreutils

# Linux (Ubuntu/Debian) 安装依赖
sudo apt-get update
sudo apt-get install -y \
    build-essential \
    git \
    rsync \
    jq \
    docker.io

# 安装 Go (如果未安装)
# 下载: https://golang.org/dl/
wget https://go.dev/dl/go1.25.6.linux-amd64.tar.gz
sudo tar -C /usr/local -xzf go1.25.6.linux-amd64.tar.gz
export PATH=$PATH:/usr/local/go/bin
```

### 1.4 可选工具

```bash
# etcd (用于本地测试)
ETCD_VER=v3.5.12
curl -L https://github.com/etcd-io/etcd/releases/download/${ETCD_VER}/etcd-${ETCD_VER}-linux-amd64.tar.gz -o etcd.tar.gz
tar xzf etcd.tar.gz
sudo mv etcd-${ETCD_VER}-linux-amd64/etcd* /usr/local/bin/

# kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# kind (Kubernetes in Docker)
go install sigs.k8s.io/kind@latest

# delve 调试器
go install github.com/go-delve/delve/cmd/dlv@latest
```

## 2. 代码获取与编译

### 2.1 获取源码

```bash
# 克隆仓库
git clone https://github.com/kubernetes/kubernetes.git
cd kubernetes

# 查看分支
git branch -a

# 切换到指定版本 (例如 v1.32.0)
git checkout v1.32.0

# 或创建开发分支
git checkout -b my-feature origin/master
```

### 2.2 项目结构

```
kubernetes/
├── cmd/                    # 各组件入口
│   ├── kube-apiserver/     # API Server 入口
│   ├── kube-controller-manager/
│   ├── kube-scheduler/
│   ├── kubelet/
│   └── kube-proxy/
├── pkg/                    # 核心实现
│   ├── api/                # API 类型
│   ├── controller/         # 控制器
│   ├── kubelet/            # Kubelet 实现
│   ├── proxy/              # kube-proxy 实现
│   ├── scheduler/          # 调度器
│   └── volume/             # 存储
├── staging/src/k8s.io/     # 独立发布的库
│   ├── client-go/          # Go 客户端
│   ├── apimachinery/       # API 机制
│   ├── apiserver/          # API Server 框架
│   └── ...
├── hack/                   # 构建和开发脚本
├── test/                   # 测试代码
│   ├── e2e/                # E2E 测试
│   └── integration/        # 集成测试
├── go.mod                  # Go 模块定义
├── go.work                 # Go 工作区
└── Makefile                # 构建目标
```

### 2.3 Makefile 构建系统

```bash
# 查看可用的 make 目标
make help

# 常用构建目标
make                        # 构建所有二进制文件
make all                    # 同上
make WHAT=cmd/kube-apiserver  # 只构建 API Server
make WHAT="cmd/kubelet cmd/kubectl"  # 构建多个组件

# 快速构建 (跳过代码生成)
make quick-release

# 构建特定平台
make KUBE_BUILD_PLATFORMS=linux/amd64

# 清理构建产物
make clean
```

### 2.4 构建示例

```bash
# 完整构建 (首次)
make

# 构建产物位置
ls _output/bin/
# kube-apiserver  kube-controller-manager  kubectl  kubelet  kube-proxy  kube-scheduler

# 增量构建单个组件
make WHAT=cmd/kube-apiserver

# 带调试符号的构建 (用于 delve 调试)
make DBG=1 WHAT=cmd/kube-apiserver
```

### 2.5 代码生成

Kubernetes 使用代码生成来创建样板代码：

```bash
# 更新所有生成的代码
make update

# 只更新代码生成
make generated_files

# 更新 API 相关代码
./hack/update-codegen.sh

# 更新 vendor 依赖
./hack/update-vendor.sh

# 验证生成的代码是否最新
make verify

# 验证特定内容
./hack/verify-codegen.sh
./hack/verify-vendor.sh
```

## 3. IDE 配置

### 3.1 VSCode 配置

创建 `.vscode/settings.json`：

```json
{
    "go.useLanguageServer": true,
    "go.buildOnSave": "off",
    "go.lintOnSave": "off",
    "go.vetOnSave": "off",
    "go.testOnSave": false,
    "go.coverOnSave": false,
    "go.formatTool": "gofmt",
    "go.goroot": "/usr/local/go",
    "gopls": {
        "build.directoryFilters": [
            "-vendor",
            "-_output",
            "-third_party"
        ],
        "formatting.gofumpt": false,
        "ui.completion.usePlaceholders": true,
        "ui.semanticTokens": true
    },
    "editor.formatOnSave": true,
    "[go]": {
        "editor.codeActionsOnSave": {
            "source.organizeImports": "explicit"
        }
    },
    "files.watcherExclude": {
        "**/vendor/**": true,
        "**/_output/**": true,
        "**/third_party/**": true
    },
    "search.exclude": {
        "**/vendor/**": true,
        "**/_output/**": true
    }
}
```

创建 `.vscode/launch.json` 用于调试：

```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Debug kube-apiserver",
            "type": "go",
            "request": "launch",
            "mode": "debug",
            "program": "${workspaceFolder}/cmd/kube-apiserver",
            "args": [
                "--etcd-servers=http://127.0.0.1:2379",
                "--service-cluster-ip-range=10.0.0.0/16",
                "--secure-port=6443",
                "--authorization-mode=RBAC",
                "--v=2"
            ],
            "env": {
                "KUBERNETES_SERVICE_HOST": "",
                "KUBERNETES_SERVICE_PORT": ""
            }
        },
        {
            "name": "Debug kube-controller-manager",
            "type": "go",
            "request": "launch",
            "mode": "debug",
            "program": "${workspaceFolder}/cmd/kube-controller-manager",
            "args": [
                "--kubeconfig=${env:HOME}/.kube/config",
                "--leader-elect=false",
                "--v=2"
            ]
        },
        {
            "name": "Debug kube-scheduler",
            "type": "go",
            "request": "launch",
            "mode": "debug",
            "program": "${workspaceFolder}/cmd/kube-scheduler",
            "args": [
                "--kubeconfig=${env:HOME}/.kube/config",
                "--leader-elect=false",
                "--v=2"
            ]
        },
        {
            "name": "Debug Unit Test",
            "type": "go",
            "request": "launch",
            "mode": "test",
            "program": "${fileDirname}",
            "args": [
                "-test.v",
                "-test.run",
                "TestFunctionName"
            ]
        }
    ]
}
```

### 3.2 GoLand 配置

1. **导入项目**：
   - File → Open → 选择 kubernetes 目录
   - 等待索引完成（首次可能需要较长时间）

2. **配置 GOROOT**：
   - Preferences → Go → GOROOT
   - 设置为 Go 1.25+ 的安装路径

3. **配置排除目录**：
   - 右键 `vendor` → Mark Directory as → Excluded
   - 右键 `_output` → Mark Directory as → Excluded

4. **运行配置**：
   - Run → Edit Configurations
   - 添加 Go Build 配置
   - Package path: `k8s.io/kubernetes/cmd/kube-apiserver`
   - 设置运行参数和环境变量

### 3.3 代码导航技巧

```bash
# 使用 grep 查找代码
grep -r "func NewDeploymentController" pkg/

# 使用 ripgrep (更快)
rg "func NewDeploymentController" pkg/

# 查找接口实现
rg "implements.*Interface" --type go

# 查找结构体定义
rg "^type.*struct" staging/src/k8s.io/client-go/tools/cache/

# 查找特定函数的调用
rg "\.Reconcile\(" pkg/controller/
```

## 4. 测试环境准备

### 4.1 Kind 集群搭建

Kind (Kubernetes in Docker) 是本地开发测试的首选：

```bash
# 安装 kind
go install sigs.k8s.io/kind@latest

# 创建单节点集群
kind create cluster --name dev

# 创建多节点集群
cat > kind-config.yaml << EOF
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
- role: worker
EOF
kind create cluster --name dev --config kind-config.yaml

# 使用本地构建的镜像
kind load docker-image my-image:tag --name dev

# 获取 kubeconfig
kind get kubeconfig --name dev > ~/.kube/kind-dev-config
export KUBECONFIG=~/.kube/kind-dev-config

# 删除集群
kind delete cluster --name dev
```

### 4.2 Minikube 使用

```bash
# 安装 minikube
# macOS
brew install minikube

# Linux
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube

# 启动集群
minikube start --cpus=4 --memory=8192 --driver=docker

# 使用特定 Kubernetes 版本
minikube start --kubernetes-version=v1.32.0

# 启用插件
minikube addons enable metrics-server
minikube addons enable dashboard

# 访问 dashboard
minikube dashboard

# 停止和删除
minikube stop
minikube delete
```

### 4.3 多节点测试集群

使用 local-up-cluster.sh 脚本：

```bash
# 设置环境变量
export CONTAINER_RUNTIME_ENDPOINT=unix:///run/containerd/containerd.sock
export LOG_LEVEL=2

# 启动本地集群 (单节点)
./hack/local-up-cluster.sh

# 在另一个终端使用集群
export KUBECONFIG=/var/run/kubernetes/admin.kubeconfig
kubectl get nodes
```

## 5. 常用开发命令

### 5.1 单元测试

```bash
# 运行所有单元测试
make test

# 运行特定包的测试
go test -v ./pkg/controller/deployment/...

# 运行特定测试函数
go test -v -run TestSyncDeployment ./pkg/controller/deployment/

# 带覆盖率的测试
go test -cover -coverprofile=coverage.out ./pkg/controller/deployment/
go tool cover -html=coverage.out

# 运行短测试 (跳过长时间测试)
go test -short ./...

# 并行测试
go test -parallel 4 ./pkg/controller/...
```

### 5.2 集成测试

```bash
# 运行所有集成测试
make test-integration

# 运行特定的集成测试
go test -v ./test/integration/scheduler/...

# 设置超时
go test -timeout 30m ./test/integration/...

# 带日志级别
go test -v -args -v=4 ./test/integration/apiserver/...
```

### 5.3 E2E 测试

```bash
# 构建 E2E 测试二进制
make WHAT=test/e2e/e2e.test

# 运行 E2E 测试 (需要运行中的集群)
./_output/bin/e2e.test --provider=local --kubeconfig=$HOME/.kube/config

# 运行特定测试
./_output/bin/e2e.test --provider=local \
    --kubeconfig=$HOME/.kube/config \
    --ginkgo.focus="Deployment"

# 跳过某些测试
./_output/bin/e2e.test --provider=local \
    --kubeconfig=$HOME/.kube/config \
    --ginkgo.skip="Slow|Serial"
```

### 5.4 代码检查

```bash
# 运行所有验证
make verify

# 验证代码格式
./hack/verify-gofmt.sh

# 验证导入顺序
./hack/verify-import-aliases.sh

# 验证生成代码
./hack/verify-codegen.sh

# 静态分析
./hack/verify-golint.sh
./hack/verify-govet.sh

# 运行所有 lint 检查
make lint
```

### 5.5 调试技巧

```bash
# 启用详细日志
./cmd/kube-apiserver --v=6

# 启用特定组件的日志
./cmd/kube-controller-manager --v=4 --vmodule=deployment_controller=6

# 使用 pprof 性能分析
# API Server 默认在 /debug/pprof 提供 pprof 端点
go tool pprof http://localhost:6443/debug/pprof/heap
go tool pprof http://localhost:6443/debug/pprof/profile?seconds=30

# CPU 分析
go tool pprof -http=:8080 http://localhost:6443/debug/pprof/profile?seconds=30

# 内存分析
go tool pprof -http=:8080 http://localhost:6443/debug/pprof/heap
```

## 6. 环境变量

### 6.1 常用环境变量

```bash
# Go 相关
export GOPATH=$HOME/go
export PATH=$PATH:$GOPATH/bin
export GO111MODULE=on
export GOPROXY=https://proxy.golang.org,direct

# Kubernetes 开发
export KUBE_EDITOR=vim
export KUBECONFIG=$HOME/.kube/config

# 构建加速
export KUBE_FASTBUILD=true
export KUBE_VERBOSE=0

# 测试
export KUBE_TEST_ARGS="-v"
export KUBE_TIMEOUT=300s

# 调试
export KUBE_VERBOSE_LEVEL=4
```

### 6.2 构建相关环境变量

| 变量 | 描述 | 示例 |
|-----|-----|-----|
| `WHAT` | 指定构建目标 | `cmd/kube-apiserver` |
| `DBG` | 启用调试符号 | `1` |
| `KUBE_BUILD_PLATFORMS` | 目标平台 | `linux/amd64` |
| `KUBE_FASTBUILD` | 快速构建模式 | `true` |
| `GOGC` | GC 触发比例 | `200` |
| `GOMAXPROCS` | 最大并行数 | `8` |

## 小结

本文介绍了 Kubernetes 开发环境的完整搭建流程：

1. **系统要求**：Go 1.25+、足够的内存和磁盘空间
2. **代码获取**：从 GitHub 克隆并选择适当的版本分支
3. **编译构建**：使用 Makefile 进行增量或完整构建
4. **IDE 配置**：VSCode 和 GoLand 的最佳实践配置
5. **测试环境**：Kind 或 Minikube 搭建本地集群
6. **开发命令**：单元测试、集成测试、E2E 测试和代码检查

下一篇文章将介绍如何使用 local-up-cluster.sh 进行本地集群调试。
