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

#### 6.2 Istioを使ってPod間の通信を守る
#### 6.2.1 アプリケーションを相互認証する

Istioが提供する機能

- Traffic Management<br>
  ロードバランシング、カナリアリリース、トラフィックのミラーリング、通信レート制限、サーキットブレーカ
- Observability<br>
  ログやメトリクスの収集、分散トレーシング
- Security<br>
  mTLS認証、RBACによるリクエストの認可
 

default namespace に mTLSを有効にする
```
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: default
spec:
  mtls:
    mode: STRICT
```

appをデプロイ
```
$ kubectl apply -f samples/httpbin/httpbin.yaml
$ kubectl apply -f samples/sleep/sleep.yaml
```
