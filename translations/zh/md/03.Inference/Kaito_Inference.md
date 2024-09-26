## 使用 Kaito 进行推理

[Kaito](https://github.com/Azure/kaito) 是一个操作器，用于在 Kubernetes 集群中自动化 AI/ML 推理模型的部署。

与大多数基于虚拟机基础设施的主流模型部署方法相比，Kaito 具有以下关键区别：

- 使用容器镜像管理模型文件。提供一个 HTTP 服务器，通过模型库执行推理调用。
- 通过提供预设配置，避免为适配 GPU 硬件而调整部署参数。
- 根据模型需求自动配置 GPU 节点。
- 如果许可允许，将大型模型镜像托管在公共的 Microsoft 容器注册表 (MCR) 中。

使用 Kaito，可以大大简化在 Kubernetes 中引入大型 AI 推理模型的工作流程。

## 架构

Kaito 遵循经典的 Kubernetes 自定义资源定义 (CRD)/控制器设计模式。用户管理一个描述 GPU 需求和推理规范的 `workspace` 自定义资源。Kaito 控制器将通过协调 `workspace` 自定义资源来自动化部署。
<div align="left">
  <img src="https://github.com/Azure/kaito/blob/main/docs/img/arch.png" width=80% title="Kaito 架构" alt="Kaito 架构">
</div>

上图展示了 Kaito 的架构概述。其主要组件包括：

- **Workspace 控制器**：它协调 `workspace` 自定义资源，创建 `machine`（如下所述）自定义资源以触发节点自动配置，并根据模型预设配置创建推理工作负载（`deployment` 或 `statefulset`）。
- **节点配置控制器**：控制器的名称是 [gpu-provisioner helm chart](https://github.com/Azure/gpu-provisioner/tree/main/charts/gpu-provisioner) 中的 *gpu-provisioner*。它使用来自 [Karpenter](https://sigs.k8s.io/karpenter) 的 `machine` CRD 与 workspace 控制器交互。它集成了 Azure Kubernetes Service (AKS) API，以向 AKS 集群添加新的 GPU 节点。
> 注意：[*gpu-provisioner*](https://github.com/Azure/gpu-provisioner) 是一个开源组件。如果支持 [Karpenter-core](https://sigs.k8s.io/karpenter) API，也可以用其他控制器替换。

## 安装

请查看 [这里](https://github.com/Azure/kaito/blob/main/docs/installation.md) 的安装指南。

## 快速开始推理 Phi-3
[示例代码推理 Phi-3](https://github.com/Azure/kaito/tree/main/examples/inference)

```
apiVersion: kaito.sh/v1alpha1
kind: Workspace
metadata:
  name: workspace-phi-3-mini
resource:
  instanceType: "Standard_NC6s_v3"
  labelSelector:
    matchLabels:
      apps: phi-3
inference:
  preset:
    name: phi-3-mini-4k-instruct
    # 注意：此配置也适用于 phi-3-mini-128k-instruct 预设
```

```sh
$ cat examples/inference/kaito_workspace_phi_3.yaml

apiVersion: kaito.sh/v1alpha1
kind: Workspace
metadata:
  name: workspace-phi-3-mini
resource:
  instanceType: "Standard_NC6s_v3"
  labelSelector:
    matchLabels:
      app: phi-3-adapter
tuning:
  preset:
    name: phi-3-mini-4k-instruct
  method: qlora
  input:
    urls:
      - "https://huggingface.co/datasets/philschmid/dolly-15k-oai-style/resolve/main/data/train-00000-of-00001-54e3756291ca09c6.parquet?download=true"
  output:
    image: "ACR_REPO_HERE.azurecr.io/IMAGE_NAME_HERE:0.0.1" # 调优输出 ACR 路径
    imagePushSecret: ACR_REGISTRY_SECRET_HERE
    

$ kubectl apply -f examples/inference/kaito_workspace_phi_3.yaml
```

可以通过运行以下命令来跟踪 workspace 状态。当 WORKSPACEREADY 列变为 `True` 时，表示模型已成功部署。

```sh
$ kubectl get workspace kaito_workspace_phi_3.yaml
NAME                  INSTANCE            RESOURCEREADY   INFERENCEREADY   WORKSPACEREADY   AGE
workspace-phi-3-mini   Standard_NC6s_v3   True            True             True             10m
```

接下来，可以找到推理服务的集群 IP，并使用临时 `curl` pod 测试集群中的服务端点。

```sh
$ kubectl get svc workspace-phi-3-mini
NAME                  TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)            AGE
workspace-phi-3-mini-adapter  ClusterIP   <CLUSTERIP>  <none>        80/TCP,29500/TCP   10m

export CLUSTERIP=$(kubectl get svc workspace-phi-3-mini-adapter -o jsonpath="{.spec.clusterIPs[0]}") 
$ kubectl run -it --rm --restart=Never curl --image=curlimages/curl -- curl -X POST http://$CLUSTERIP/chat -H "accept: application/json" -H "Content-Type: application/json" -d "{\"prompt\":\"YOUR QUESTION HERE\"}"
```

## 快速开始使用适配器推理 Phi-3

安装 Kaito 后，可以尝试以下命令启动推理服务。

[示例代码使用适配器推理 Phi-3](https://github.com/Azure/kaito/blob/main/examples/inference/kaito_workspace_phi_3_with_adapters.yaml)

```
apiVersion: kaito.sh/v1alpha1
kind: Workspace
metadata:
  name: workspace-phi-3-mini-adapter
resource:
  instanceType: "Standard_NC6s_v3"
  labelSelector:
    matchLabels:
      apps: phi-3-adapter
inference:
  preset:
    name: phi-3-mini-128k-instruct
  adapters:
    - source:
        name: "phi-3-adapter"
        image: "ACR_REPO_HERE.azurecr.io/ADAPTER_HERE:0.0.1"
      strength: "1.0"
```

```sh
$ cat examples/inference/kaito_workspace_phi_3_with_adapters.yaml

apiVersion: kaito.sh/v1alpha1
kind: Workspace
metadata:
  name: workspace-phi-3-mini-adapter
resource:
  instanceType: "Standard_NC6s_v3"
  labelSelector:
    matchLabels:
      app: phi-3-adapter
tuning:
  preset:
    name: phi-3-mini-128k-instruct
  method: qlora
  input:
    urls:
      - "https://huggingface.co/datasets/philschmid/dolly-15k-oai-style/resolve/main/data/train-00000-of-00001-54e3756291ca09c6.parquet?download=true"
  output:
    image: "ACR_REPO_HERE.azurecr.io/IMAGE_NAME_HERE:0.0.1" # 调优输出 ACR 路径
    imagePushSecret: ACR_REGISTRY_SECRET_HERE
    

$ kubectl apply -f examples/inference/kaito_workspace_phi_3_with_adapters.yaml
```

可以通过运行以下命令来跟踪 workspace 状态。当 WORKSPACEREADY 列变为 `True` 时，表示模型已成功部署。

```sh
$ kubectl get workspace kaito_workspace_phi_3_with_adapters.yaml
NAME                  INSTANCE            RESOURCEREADY   INFERENCEREADY   WORKSPACEREADY   AGE
workspace-phi-3-mini-adapter   Standard_NC6s_v3   True            True             True             10m
```

接下来，可以找到推理服务的集群 IP，并使用临时 `curl` pod 测试集群中的服务端点。

```sh
$ kubectl get svc workspace-phi-3-mini-adapter
NAME                  TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)            AGE
workspace-phi-3-mini-adapter  ClusterIP   <CLUSTERIP>  <none>        80/TCP,29500/TCP   10m

export CLUSTERIP=$(kubectl get svc workspace-phi-3-mini-adapter -o jsonpath="{.spec.clusterIPs[0]}") 
$ kubectl run -it --rm --restart=Never curl --image=curlimages/curl -- curl -X POST http://$CLUSTERIP/chat -H "accept: application/json" -H "Content-Type: application/json" -d "{\"prompt\":\"YOUR QUESTION HERE\"}"
```

免责声明：本翻译由人工智能模型从原文翻译而来，可能并不完美。
请审阅翻译结果并进行必要的修改。