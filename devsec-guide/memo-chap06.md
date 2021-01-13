### 6章 アプリケーション間の通信を守る
#### 6.1 Network Policyを使ってPodの通信を制御する

- Network Policyを有効にするにはアクセス制御をしたいPodと同じNamespaceにNetworkPolicyリソースを作成し、<br>アクセス許可をするルールを定義する
- アクセスルールにはIngressとEgressトラフィックを指定できる
- Namespace内にNetworkPolicyリソースが存在しない場合はデフォルト許可になるのでデフォルト拒否にしたい場合は、<br>空のルールのリソースをNamespaceに作成する

デフォルト拒否
```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: all-deny
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

メタデータAPIへのアクセスを禁止する
```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-egress-metadata
  namespace: default
spec:
  podSelector: {}
  ingress:
  - {}
  egress:
  - to:
    - ipBlock:
        cider: 0.0.0.0/0
        except:
        - 169.254.169.254/32
```
