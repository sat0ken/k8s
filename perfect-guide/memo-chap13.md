### 13章 セキュリティ
#### 13.1 Service Account

Service Accountの作成
```
$ kubectl create serviceaccount sample-serviceaccount
serviceaccount/sample-serviceaccount created
```

Service Accountとトークン
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
