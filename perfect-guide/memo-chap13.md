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
↑このService Accountで起動するPodはTokenをVolumeマウントしなくなる
トークンを利用する場合は、明示的に指定する
```
spec:
  serviceAccountName: sample-serviceaccount
  automountServiceAccountToken: true
  containers:
  - image: nginx:1.18.0-alpine
```
