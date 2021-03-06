---
title: 访问 TiDB Dashboard
summary: 介绍如何在 Kubernetes 环境下访问 TiDB Dashboard
category: how-to
---

# TiDB Dashboard 指南

TiDB Dashboard 是 TiDB 4.0 专门用来帮助观察与诊断整个 TiDB 集群的可视化面板，你可以在 [TiDB Dashboard](https://github.com/pingcap-incubator/tidb-dashboard) 了解详情。本篇文章将介绍如何在 Kubernetes 环境下访问 TiDB Dashboard。

## 快速上手

> **注意：**
>
> 以下教程仅为演示如何快速访问 TiDB Dashboard，请勿在生产环境中直接使用以下方法。 

`TiDB Dashboard` 目前在 4.0 版本中已经内嵌在了 PD 组件中，你可以通过以下的例子在 Kubernetes 环境下快速部署一个 4.0.0-rc 版本的 TiDB 集群。运行 `kubectl apply -f` 命令，将以下 yaml 文件部署到 Kubernetes 集群中。

```yaml
apiVersion: pingcap.com/v1alpha1
kind: TidbCluster
metadata:
  name: basic
spec:
  version: v4.0.0-rc
  timezone: UTC
  pvReclaimPolicy: Delete
  pd:
    baseImage: pingcap/pd
    replicas: 1
    requests:
      storage: "1Gi"
    config: {}
  tikv:
    baseImage: pingcap/tikv
    replicas: 1
    requests:
      storage: "1Gi"
    config: {}
  tidb:
    baseImage: pingcap/tidb
    replicas: 1
    service:
      type: ClusterIP
    config: {}

```

当集群创建完毕时，你可以通过以下指令将 `TiDB Dashboard` 暴露在本地机器:

{{< copyable "shell-regular" >}}

```shell
kubectl port-forward svc/tidb-pd -n ${namespace} 2379:2379
```

然后在浏览器中访问 <http://localhost:2379/dashboard> 即可访问到 TiDB Dashboard。

## 通过 Ingress 访问 TiDB Dashboard

> **注意：**
>
> 我们推荐在生产环境、关键环境内使用 `Ingress` 来暴露 `TiDB Dashboard` 服务。由于内嵌式 Dashboard 的端口与 PD API 的端口是同一个端口，如果采用其他自治方案在生产环境、关键环境内暴露 `TiDB Dashboard` 服务，需要注意不应该暴露 PD API 的相关接口。

### 环境准备

使用 `Ingress` 前需要 Kubernetes 集群安装有 `Ingress` 控制器，仅创建 `Ingress` 资源无效。您可能需要部署 `Ingress` 控制器，例如 [ingress-nginx](https://kubernetes.github.io/ingress-nginx/deploy/)。您可以从许多 [Ingress 控制器](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/) 中进行选择。

### 使用 Ingress

你可以通过 `Ingress` 来将 TiDB Dashboard 服务暴露到 Kubernetes 集群外，从而在 Kubernetes 集群外通过 http/https 的方式访问服务。你可以通过 [Ingress](https://kubernetes.io/zh/docs/concepts/services-networking/ingress/) 了解更多关于 `Ingress` 的信息。以下是一个使用 `Ingress` 访问 `TiDB Dashboard` 的 yaml 文件例子。运行 `kubectl apply -f` 命令，将以下 yaml 文件部署到 Kubernetes 集群中。

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: access-dashboard
  namespace: ${namespace}
spec:
  rules:
    - host: ${host}
      http:
        paths:
          - backend:
              serviceName: ${cluster_name}-pd
              servicePort: 2379
            path: /dashboard
```

当部署了 Ingress 后，你可以在 Kubernetes 集群外通过 <http://${host}/dashboard> 访问 TiDB Dashboard。

## 开启 Ingress TLS

> **注意：**
>
> 由于 Ingress 假定了 TLS 终止，所以当目前 TiDB 集群开启了 [TLS 验证](enable-tls-between-components.md)时，你将无法通过 Ingress 访问 Dashboard。

Ingress 提供了 TLS 支持，你可以通过 [Ingress TLS](https://kubernetes.io/zh/docs/concepts/services-networking/ingress/#tls) 了解更多。以下是一个使用 Ingress TLS 的例子，其中 `testsecret-tls` 包含了 `exmaple.com` 所需要的 `tls.crt` 与 `tls.key`：

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: access-dashboard
  namespace: ${namespace}
spec:
  tls:
  - hosts:
    - ${host}
    secretName: testsecret-tls
  rules:
    - host: ${host}
      http:
        paths:
          - backend:
              serviceName: ${cluster_name}-pd
              servicePort: 2379
            path: /dashboard
```

以下是 `testsecret-tls` 的一个例子:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: testsecret-tls
  namespace: default
data:
  tls.crt: base64 encoded cert
  tls.key: base64 encoded key
type: kubernetes.io/tls
```

当 Ingress 部署完成以后，你就可以通过 <https://{host}/dashboard> 访问 TiDB Dashboard。
