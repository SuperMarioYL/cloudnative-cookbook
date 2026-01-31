---
title: "Kubernetes 网络模型"
weight: 01
---


## 概述

Kubernetes 定义了一套简洁而强大的网络模型，为容器提供扁平的网络空间。本章介绍 Kubernetes 网络的四大基本要求、Pod 网络模型，以及各种通信场景的实现方式。

## 前置知识

- Kubernetes 基础概念（Pod、Node、Service）
- Linux 网络基础
- 容器基础知识

## Kubernetes 网络四大要求

Kubernetes 网络模型基于以下四个核心要求：

```mermaid
graph TB
    subgraph requirements["Kubernetes 网络要求"]
        R1["1. Pod 内容器通信<br/>共享 Network Namespace"]
        R2["2. Pod 到 Pod 通信<br/>无 NAT，直接通信"]
        R3["3. Pod 到 Service 通信<br/>虚拟 IP 负载均衡"]
        R4["4. 外部到 Service 通信<br/>NodePort/LoadBalancer"]
    end

    R1 --> |"localhost"| C1["同 Pod 容器"]
    R2 --> |"Pod IP"| C2["任意 Pod"]
    R3 --> |"ClusterIP"| C3["Service 后端"]
    R4 --> |"External IP"| C4["集群服务"]
```

### 1. 同 Pod 内容器通信

同一 Pod 内的容器共享 Network Namespace：

```mermaid
graph TB
    subgraph pod["Pod"]
        subgraph netns["共享 Network Namespace"]
            ETH["eth0<br/>10.244.1.2"]
            LO["lo<br/>127.0.0.1"]
        end

        C1["Container 1<br/>:8080"]
        C2["Container 2<br/>:9090"]
        C3["Container 3<br/>:3306"]

        C1 --> ETH
        C2 --> ETH
        C3 --> ETH
        C1 --> LO
        C2 --> LO
        C3 --> LO
    end
```

- 容器间通过 `localhost` 通信
- 端口不能冲突
- 共享 IP 地址

### 2. Pod 到 Pod 通信

所有 Pod 都可以不经 NAT 直接通信：

```mermaid
graph LR
    subgraph node1["Node 1"]
        POD1["Pod A<br/>10.244.1.2"]
        POD2["Pod B<br/>10.244.1.3"]
    end

    subgraph node2["Node 2"]
        POD3["Pod C<br/>10.244.2.2"]
        POD4["Pod D<br/>10.244.2.3"]
    end

    POD1 <--> |"直接通信"| POD2
    POD1 <--> |"跨节点"| POD3
    POD2 <--> |"跨节点"| POD4
```

**关键特征**：
- 每个 Pod 有独立的 IP 地址
- Pod 看到的源 IP 是真实 Pod IP（无 NAT）
- 同节点和跨节点通信方式对 Pod 透明

### 3. Pod 到 Service 通信

Service 提供稳定的虚拟 IP 和负载均衡：

```mermaid
graph TB
    CLIENT["Client Pod<br/>10.244.1.5"]

    SVC["Service<br/>my-svc<br/>ClusterIP: 10.96.1.100"]

    subgraph backends["后端 Pod"]
        POD1["Pod 1<br/>10.244.1.2:8080"]
        POD2["Pod 2<br/>10.244.2.3:8080"]
        POD3["Pod 3<br/>10.244.3.4:8080"]
    end

    CLIENT --> |"10.96.1.100:80"| SVC
    SVC --> |"负载均衡"| POD1
    SVC --> |"负载均衡"| POD2
    SVC --> |"负载均衡"| POD3
```

### 4. 外部到 Service 通信

```mermaid
graph TB
    EXT["外部客户端"]

    subgraph cluster["Kubernetes 集群"]
        NP["NodePort<br/>:30080"]
        LB["LoadBalancer<br/>External IP"]
        ING["Ingress<br/>域名路由"]

        SVC["Service"]
        PODS["Pods"]
    end

    EXT --> |"NodeIP:30080"| NP
    EXT --> |"External IP"| LB
    EXT --> |"域名"| ING

    NP --> SVC
    LB --> SVC
    ING --> SVC
    SVC --> PODS
```

## Pod 网络模型

### Pod 网络结构

```mermaid
graph TB
    subgraph pod_detail["Pod 网络详解"]
        subgraph netns["Pod Network Namespace"]
            LO["lo<br/>127.0.0.1"]
            ETH["eth0<br/>10.244.1.2/32"]

            subgraph routes["路由表"]
                R1["default via 169.254.1.1"]
                R2["169.254.1.1 dev eth0"]
            end
        end

        subgraph containers["容器"]
            PAUSE["pause 容器<br/>(持有 netns)"]
            APP1["app 容器 1"]
            APP2["app 容器 2"]
        end

        PAUSE --> netns
        APP1 --> |"共享"| netns
        APP2 --> |"共享"| netns
    end

    subgraph host["Host Network"]
        VETH["cali12345<br/>(veth peer)"]
        HOST_RT["路由: 10.244.1.2 → cali12345"]
    end

    ETH <--> |"veth pair"| VETH
```

### pause 容器的作用

每个 Pod 都有一个隐藏的 `pause` 容器：

```mermaid
sequenceDiagram
    participant Kubelet
    participant CRI as Container Runtime
    participant CNI
    participant Pause as pause 容器
    participant App as 应用容器

    Kubelet->>CRI: 创建 Pod
    CRI->>Pause: 创建 pause 容器
    Note over Pause: 持有 Network Namespace

    CRI->>CNI: ADD (pause 容器的 netns)
    CNI->>CNI: 配置网络
    CNI-->>CRI: 返回 IP

    CRI->>App: 创建应用容器
    Note over App: 加入 pause 的 netns

    Note over Pause,App: 共享网络栈
```

**pause 容器的职责**：
1. 持有 Pod 的 Network Namespace
2. 作为 PID 1 进程（init 进程）
3. 回收僵尸进程

### IP 地址分配

```mermaid
graph TB
    subgraph ipam["IPAM 层次"]
        CLUSTER["Cluster CIDR<br/>10.244.0.0/16"]
        CLUSTER --> NODE1["Node 1 CIDR<br/>10.244.1.0/24"]
        CLUSTER --> NODE2["Node 2 CIDR<br/>10.244.2.0/24"]
        CLUSTER --> NODE3["Node 3 CIDR<br/>10.244.3.0/24"]

        NODE1 --> POD1["Pod: 10.244.1.2"]
        NODE1 --> POD2["Pod: 10.244.1.3"]
        NODE2 --> POD3["Pod: 10.244.2.2"]
    end
```

## 通信场景详解

### 场景 1：同节点 Pod 通信

```mermaid
sequenceDiagram
    participant PodA as Pod A<br/>10.244.1.2
    participant VethA as veth-a
    participant Host as Host 路由
    participant VethB as veth-b
    participant PodB as Pod B<br/>10.244.1.3

    PodA->>VethA: 发送到 10.244.1.3
    VethA->>Host: 进入 host netns
    Host->>Host: 查路由表<br/>10.244.1.3 → veth-b
    Host->>VethB: 转发
    VethB->>PodB: 进入 Pod B netns
```

**特点**：
- 不经过物理网络
- 在内核中完成转发
- 延迟最低

### 场景 2：跨节点 Pod 通信（路由模式）

```mermaid
sequenceDiagram
    participant PodA as Pod A<br/>10.244.1.2
    participant Host1 as Node 1
    participant Network as 物理网络
    participant Host2 as Node 2
    participant PodB as Pod B<br/>10.244.2.2

    PodA->>Host1: 发送到 10.244.2.2
    Host1->>Host1: 路由查找<br/>10.244.2.0/24 via Node2
    Host1->>Network: 转发（源IP不变）
    Network->>Host2: 路由到 Node 2
    Host2->>Host2: 路由查找<br/>10.244.2.2 → veth-b
    Host2->>PodB: 送达
```

### 场景 3：跨节点 Pod 通信（IPIP/VXLAN 模式）

```mermaid
sequenceDiagram
    participant PodA as Pod A<br/>10.244.1.2
    participant Host1 as Node 1
    participant Tunnel as 隧道接口
    participant Network as 物理网络
    participant Host2 as Node 2
    participant PodB as Pod B<br/>10.244.2.2

    PodA->>Host1: 原始包<br/>src=10.244.1.2 dst=10.244.2.2
    Host1->>Tunnel: 封装<br/>外层: Node1→Node2<br/>内层: Pod A→Pod B
    Tunnel->>Network: 发送封装后的包
    Network->>Host2: 送达 Node 2
    Host2->>Host2: 解封装
    Host2->>PodB: 原始包送达
```

### 场景 4：Pod 访问 Service

```mermaid
sequenceDiagram
    participant Pod as Client Pod<br/>10.244.1.2
    participant IPT as iptables/eBPF
    participant Backend as Backend Pod<br/>10.244.2.3

    Pod->>IPT: dst=ClusterIP:80
    Note over IPT: DNAT 转换<br/>ClusterIP → 10.244.2.3
    IPT->>Backend: dst=10.244.2.3:8080

    Backend->>IPT: src=10.244.2.3:8080
    Note over IPT: 反向 NAT
    IPT->>Pod: src=ClusterIP:80
```

## Node 网络

### Node 网络组件

```mermaid
graph TB
    subgraph node["Kubernetes Node"]
        subgraph host_netns["Host Network Namespace"]
            ETH0["eth0<br/>物理网卡<br/>192.168.1.100"]
            LO["lo"]
            TUNL["tunl0<br/>(IPIP 隧道)"]
            VXLAN["vxlan.calico<br/>(VXLAN 隧道)"]

            subgraph veths["veth 接口"]
                V1["cali-xxx1"]
                V2["cali-xxx2"]
                V3["cali-xxx3"]
            end

            RT["路由表"]
            IPT["iptables"]
        end

        subgraph pods["Pod Namespaces"]
            P1["Pod 1<br/>10.244.1.2"]
            P2["Pod 2<br/>10.244.1.3"]
            P3["Pod 3<br/>10.244.1.4"]
        end
    end

    V1 <--> P1
    V2 <--> P2
    V3 <--> P3
    RT --> V1
    RT --> V2
    RT --> V3
    RT --> TUNL
    RT --> VXLAN
    RT --> ETH0
```

### Node 路由表示例

```bash
# Calico 节点的典型路由表
$ ip route

# 默认路由
default via 192.168.1.1 dev eth0

# 本节点 Pod 路由
10.244.1.2 dev cali12345 scope link
10.244.1.3 dev cali67890 scope link
10.244.1.4 dev caliabc12 scope link

# 其他节点 Pod 路由（BGP 学习，路由模式）
10.244.2.0/24 via 192.168.1.101 dev eth0 proto bird
10.244.3.0/24 via 192.168.1.102 dev eth0 proto bird

# 或者隧道模式
# 10.244.2.0/24 via 192.168.1.101 dev tunl0 proto bird onlink
# 10.244.3.0/24 via 192.168.1.102 dev tunl0 proto bird onlink
```

## CNI 插件的职责

CNI（Container Network Interface）负责 Pod 网络的创建和删除：

```mermaid
graph TB
    subgraph cni_workflow["CNI 工作流"]
        KUBELET["Kubelet"]
        CRI["Container Runtime"]
        CNI["CNI Plugin"]

        subgraph cni_operations["CNI 操作"]
            ADD["ADD - 配置网络"]
            DEL["DEL - 清理网络"]
            CHECK["CHECK - 检查状态"]
        end

        KUBELET --> CRI
        CRI --> CNI
        CNI --> ADD
        CNI --> DEL
        CNI --> CHECK
    end

    ADD --> |"创建"| VETH["veth pair"]
    ADD --> |"分配"| IP["IP 地址 (IPAM)"]
    ADD --> |"配置"| ROUTE["路由"]
    ADD --> |"创建"| WEP["WorkloadEndpoint (Calico)"]
```

## 实验：探索 Pod 网络

### 实验 1：查看 Pod 网络配置

```bash
# 创建测试 Pod
kubectl run test-pod --image=nginx --restart=Never

# 查看 Pod IP
kubectl get pod test-pod -o wide

# 进入 Pod 查看网络
kubectl exec -it test-pod -- sh

# 在 Pod 内执行
ip addr          # 查看接口
ip route         # 查看路由
cat /etc/resolv.conf  # 查看 DNS
```

### 实验 2：追踪 Pod 网络接口

```bash
# 获取 Pod 所在节点
NODE=$(kubectl get pod test-pod -o jsonpath='{.spec.nodeName}')

# SSH 到节点
ssh $NODE

# 找到对应的 veth
# 方法1：通过接口索引
kubectl exec test-pod -- cat /sys/class/net/eth0/iflink
# 假设输出 15

ip link | grep "^15:"
# 输出：15: cali12345@if4: ...

# 方法2：通过 IP
ip route | grep <pod-ip>
# 输出：10.244.1.2 dev cali12345 scope link
```

### 实验 3：测试 Pod 间通信

```bash
# 创建两个 Pod
kubectl run pod-a --image=nginx --restart=Never
kubectl run pod-b --image=busybox --restart=Never --command -- sleep 3600

# 获取 pod-a 的 IP
POD_A_IP=$(kubectl get pod pod-a -o jsonpath='{.status.podIP}')

# 从 pod-b ping pod-a
kubectl exec pod-b -- ping -c 3 $POD_A_IP

# 从 pod-b curl pod-a
kubectl exec pod-b -- wget -qO- http://$POD_A_IP

# 清理
kubectl delete pod pod-a pod-b
```

### 实验 4：观察跨节点通信

```bash
# 创建 DaemonSet 确保 Pod 分布在不同节点
cat << 'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: network-test
spec:
  selector:
    matchLabels:
      app: network-test
  template:
    metadata:
      labels:
        app: network-test
    spec:
      containers:
      - name: test
        image: busybox
        command: ["sleep", "3600"]
EOF

# 等待 Pod 运行
kubectl wait --for=condition=Ready pod -l app=network-test

# 获取 Pod 列表
kubectl get pods -l app=network-test -o wide

# 从一个 Pod ping 另一个节点的 Pod
kubectl exec <pod-on-node1> -- ping -c 3 <ip-of-pod-on-node2>

# 在节点上抓包观察
tcpdump -i eth0 icmp

# 清理
kubectl delete ds network-test
```

## Kubernetes 网络与 Calico

### Calico 如何满足 K8s 网络要求

| K8s 要求 | Calico 实现 |
|----------|------------|
| Pod 内通信 | 共享 netns（K8s 原生） |
| Pod 到 Pod | 纯路由/IPIP/VXLAN |
| Pod 到 Service | iptables/eBPF kube-proxy 替代 |
| 外部到 Service | NodePort/LoadBalancer 支持 |

### Calico 网络架构

```mermaid
graph TB
    subgraph calico_network["Calico 网络架构"]
        subgraph control["控制平面"]
            FELIX["Felix<br/>路由 + 策略"]
            BIRD["BIRD<br/>BGP 路由"]
            TYPHA["Typha<br/>数据分发"]
        end

        subgraph data["数据平面"]
            IPT["iptables/eBPF"]
            ROUTE["Linux 路由"]
            TUNNEL["隧道(可选)"]
        end

        FELIX --> IPT
        FELIX --> ROUTE
        BIRD --> ROUTE
        TYPHA --> FELIX
    end
```

## 总结

本章介绍了 Kubernetes 网络模型的核心概念：

1. **四大要求** - Pod 内、Pod 间、Pod-Service、外部-Service 通信
2. **Pod 网络** - 独立 IP、共享 netns、pause 容器
3. **通信场景** - 同节点、跨节点、访问 Service
4. **Node 网络** - veth、路由表、隧道接口

理解这些概念是掌握 Calico 的基础，后续章节将深入 CNI 规范和 Calico 的具体实现。

## 参考资料

- [Kubernetes 网络模型](https://kubernetes.io/docs/concepts/cluster-administration/networking/)
- [Pod 网络](https://kubernetes.io/docs/concepts/workloads/pods/#pod-networking)
- [Service 网络](https://kubernetes.io/docs/concepts/services-networking/service/)
- [Calico 网络架构](https://docs.tigera.io/calico/latest/about/)
