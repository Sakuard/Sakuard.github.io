---
title: 用 CRD 解決 GKE 上資源不同步
date: 2025-5-20 08:00:00 +0800
categories: [Blog, TechLog]
tags: [CRD, K8s]
---


# TL;DR
- 用 CRD 解決 CI/CD pipeline 可能導致的資源不匹配問題

## 背景
服務的部署拆成 `project CI/CD` 與 `helmfile`
而 `project CI/CD` 部署是透過 `kubectl set image` 的方式更新 container image
而當 `project` 的 GAR 若有更新，會因為 `set image` 是繞過 `helm release` 直接針對 `K8s component` 做調整，導致 `helm diff` 無法及時察覺差異

### 三個方案
- 用純腳本
- 用 `helm --reuse-value`
- 寫 `CRDs` 配合 `shell script`

#### 純腳本
現有服務的部署多樣性
- 可能一個 helm release 產生包含 2 個以上的 `Deployment` 的服務
- 可能一個 `Deployment` 同時包含多個 container
- 部署類型包含 `Deployment`, `Statefulset`, `Daemonset`

#### 用 `--reuse-value`
- 對現有服務需要統一 `helm chart` 版本，並打包成 oci
- 在 `project CICD` 利用 `--reuse-value` 同時替換 `GAR` 參數，讓 `helm release` 的資源達到同步更新

#### 用 `CRD` 配合 `shell script`
寫一個 `CRD` 紀錄服務 `release` 相關 `image` 資訊，並透過 `shell script` 針對指定資源做匹配比對

- 在全環境部署 `CRDs`
- 在需要版控的服務額外加上 `CR`
- `Project CICD` 用 `kubectl patch` 的方式，對服務 `CR` 做資源調整
- `Helm CI/CD` 內利用 `shell script` 針對 `CR` 做資源比對
---
:point_right:  純腳本
要在一個腳本同時符合上述條件的便是，其邏輯會過於複雜，對後續維護可能會有一定負擔
:point_right: 用 `--reuse-value`
目前現行的服務 `helm Chart` 多數還是在 local 且有複數個版本，若要使用 `--reuse-value` 則需要統一版本後打包成 oci 讓 `project CICD` 使用，版本統一的 effert 會有一定負擔
:point_right: 用 `CRD` 配合 `shell script`
需要多了解 CRD 以及 CR 的使用
但 `Shell script` 的邏輯相對單純
也不需太多異動

CRD
```yaml=
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: sreimageverifier.pd.sre  # 用於 kubectl get crd 時查詢用的命名
spec:
  group: pd.sre
  names:
    kind: SREImageVerifier       # define yaml 時候的 kind 使用
    plural: sreimageverifier     # 用於 kubectl get 時候的資源命名
    singular: sreimageverifier
    listKind: SREImageVerifierList
  scope: Namespaced
  versions:
  - additionalPrinterColumns:
    - jsonPath: .spec.imageRegistry
      name: image registry
      type: string
    - jsonPath: .spec.imageTag
      name: image tag
      type: string
    name: v1beta1
    schema:
      openAPIV3Schema:
        description: SRE image tag management
        type: object
        properties:
          apiVersion:
            description: 'APIVersion defines the versioned schema of this representation'
            type: string
          kind:
            description: 'Kind is a string value representing the REST resource this object represents. Servers may infer this from the endpoint the client submits requests to. Cannot be updated. In CamelCase. More info: https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#types-kinds'
            type: string
          metadata:
            type: object
          spec:
            description: SREImageVerifierSpec defines the desired state of SREImageVerifier
            type: object
            properties:
              imageRegistry:
                description: Service's current image registry.
                type: string
              imageTag:
                description: Service's current image tag.
                type: string
    served: true      # 該 CR 是否仍提供服務
    storage: true     # 該 CR 是否為 etcd 中儲存使用的 main version
    subresources:
      status: {}
```

CR
```yaml=
apiVersion: pd.sre/v1beta1
kind: SREImageVerifier
metadata:
  name: {{ .Values.app.name }}-imageverifier
spec:
  imageRegistry: {{ .Values.image.repository }}
  imageTag: {{ .Values.image.tag }}
```