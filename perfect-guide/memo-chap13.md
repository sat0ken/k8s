### 13章 セキュリティ
#### 13.1 Service Account

Service Accountの作成
```
$ kubectl create serviceaccount sample-serviceaccount
serviceaccount/sample-serviceaccount created
```

Service AccountとToken
```
$ kubectl get serviceaccounts sample-serviceaccount -o yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  creationTimestamp: "2021-01-15T05:21:44Z"
  name: sample-serviceaccount
  namespace: default
  resourceVersion: "648550"
  selfLink: /api/v1/namespaces/default/serviceaccounts/sample-serviceaccount
  uid: acd50d3b-4960-4946-94d0-44ac19682104
secrets:
- name: sample-serviceaccount-token-59vcf
```

ServiceAccountを指定してPodを作成し、TokenがVolumeとしてマウントされていることを確認
```
$ kubectl get pod sample-serviceaccount-pod -o yaml | yq .spec.volumes -y
- name: sample-serviceaccount-token-59vcf
  secret:
    defaultMode: 420
    secretName: sample-serviceaccount-token-59vcf
```

Tokenの自動マウント
```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: sample-serviceaccount
  namespace: default
#自動マウント機能を無効化する
automountServiceAccountToken: false
```
↑このService Accountで起動するPodはTokenをVolumeマウントしなくなる<br>
トークンを利用する場合は、明示的に指定する
```
spec:
  serviceAccountName: sample-serviceaccount
  automountServiceAccountToken: true
  containers:
  - image: nginx:1.18.0-alpine
```

#### 13.2 RBAC

Role = どういった操作を許可するか<br>
RoleBinding = Service AccountなどのUserに対してRoleを紐づけることで権限を付与する

リソースのレベルに応じて2種類ある
- NamespaceレベルのリソースへのRoleとRoleBinding
- CluserレベルのリソースへのClusterRoleとClusterRoleBinding

RoleとClusterRole
Roleに指定可能な実行できる操作(verb)<br>
| 種別   | 内容       | 
| ------ | ---------- | 
| *      | 全ての処理 | 
| create | 作成       | 
| delete | 削除       | 
| get    | 取得       | 
| list   | 一覧取得   | 
| patch  | 一部変更   | 
| update | 更新       | 
| watch | 変更の追従  | 

Roleの作成
```
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: sample-role
  namespace: default
rules:
- apiGroups:
  - 'apps'
  - 'extensions'
  resources:
  - replicasets
  - deployments
  - deployments/scale
  verbs:
  - "*"
```

ClusterRoleの作成
```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: sample-clusterrole
rules:
- apiGroups:
  - apps
  - extensions
  resources:
  - replicasets
  - deployments
  verbs:
  - get
  - list
  - watch
- nonResourceURLs:
  - /healthz
  - /healthz/*
  - /version
  verbs:
  - get
```

ClusterRoleのAggregation
labelがついたClusterRoleを集約して合体させる
```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: sub-clusterrole1
  labels:
    app: sample-rbac
rules:
- apiGroups:
  - apps
  resources:
  - deployments
  verbs:
  - get
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: sub-clusterrole2
  labels:
    app: sample-rbac
rules:
- apiGroups:
  - ""
  resources:
  - services
  verbs:
  - get
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: sample-aggregated-clusterrole
aggregationRule:
  clusterRoleSelectors:
  - matchLabels:
      app: sample-rbac
rules:
- apiGroups:
  - ""
  resources:
  - pods
  verbs:
  - get
```

↑の１と２のClusterRoleが集約している
```
$ k get clusterrole sample-aggregated-clusterrole -o yaml | yq -y .rules
- apiGroups:
    - apps
  resources:
    - deployments
  verbs:
    - get
- apiGroups:
    - ''
  resources:
    - services
  verbs:
    - get
```
