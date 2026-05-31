# sre-agent-gitops
## 1. 项目背景

K8s 里组件多、状态多，Pod 异常时经常要看监控、查事件、查日志、看配置。对实习生来说，最容易卡在三个地方：

- 不知道先看哪个命令。
- 不知道告警对应什么故障。
- 手动部署和排查步骤多，容易漏。

所以这个项目把几块能力串起来：

```mermaid
flowchart LR
    A[自动化部署K8s] --> B[Prometheus发现告警]
    B --> C[Go Agent整理告警]
    C --> D[Ollama辅助分析]
    D --> E[K8s API基础处理]
```

## 2 项目整体规划

项目按五层来理解：

| 层次           | 做什么              | 对应组件                        | 面试表达                             |
| -------------- | ------------------- | ------------------------------- | ------------------------------------ |
| 环境准备层     | 先把 K8s 集群搭起来 | Ansible、Kubernetes             | 这是项目地基，不是核心业务逻辑       |
| 组件部署层     | 部署监控和 AI 组件  | Prometheus、Grafana、Ollama     | 让告警和模型能力可用                 |
| 故障模拟发现层 | 发现 Pod 异常       | 部署 Crash应用， PrometheusRule | 让系统主动发现 CrashLoopBackOff      |
| 智能分析层     | 整理告警并生成建议  | Go Agent、Ollama                | 把告警转换成可理解的处理建议         |
| 处理验证层     | 验证基础处理动作    | client-go、K8s API、Deployment  | 删除异常 Pod，让 Deployment 自动重建 |

## 3. 总体架构图

```mermaid
flowchart TD
    subgraph User[访问与运维入口]
        U1[运维人员]
        U2[浏览器访问Grafana/Prometheus/ArgoCD]
    end

    subgraph K8s[Kubernetes集群 1 Master + 2 Worker]
        A[Argo CD]
        P[Prometheus]
        G[Grafana]
        O[Ollama LLM服务]
        S[SRE Agent Go程序]
        C[crash-app测试应用]
        API[Kubernetes API Server]
        PV[PV/PVC持久化存储]
    end

    Git[Git仓库/部署配置] --> A
    A --> P
    A --> G
    A --> O
    C --> P
    P --> S
    S --> O
    S --> API
    API --> C
    O --> PV
    U1 --> U2
    U2 --> A
    U2 --> P
    U2 --> G
```


## 4. 核心执行流程图

```mermaid
sequenceDiagram
    participant App as crash-app Pod
    participant Prom as Prometheus
    participant Agent as SRE Agent
    participant LLM as Ollama
    participant K8s as Kubernetes API
    participant Deploy as Deployment控制器

    App->>Prom: 暴露Pod状态指标
    Prom->>Prom: PrometheusRule判断CrashLoopBackOff
    Agent->>Prom: GET /api/v1/alerts
    Prom-->>Agent: 返回firing告警
    Agent->>Agent: 提取namespace/pod/reason
    Agent->>LLM: POST /api/generate 发送Prompt
    LLM-->>Agent: 返回restart或ignore建议
    Agent->>K8s: 删除异常Pod
    K8s->>Deploy: Pod被删除
    Deploy->>App: 自动创建新Pod
```


项目里我先部署了一个 crash-app，它启动后会直接退出，所以 Pod 会进入 CrashLoopBackOff。Prometheus 通过 kube-prometheus-stack 采集到这个状态，PrometheusRule 触发 PodCrashLooping 告警。Go Agent 每 30 秒请求 Prometheus 的 `/api/v1/alerts`，筛选 firing 状态的告警。如果告警名是 PodCrashLooping，就提取 Pod 名和命名空间，构造 Prompt 发给 Ollama。Ollama 返回建议后，Agent 判断里面是否包含 restart，如果包含，就通过 client-go 删除这个 Pod。因为 Pod 是 Deployment 管理的，删除后 Kubernetes 会自动重建。
