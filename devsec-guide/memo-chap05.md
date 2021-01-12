### 5. Kubernetesクラスタのセキュリティ

・Cloud Nativeのセキュリティ戦略  
多層防御の戦略が推奨  
https://kubernetes.io/docs/concepts/security/overview/#the-4c-s-of-cloud-native-security

|  防御層名  |  対応すべき脆弱性の例  |
| ---- | ---- |
|  コード(Code)  |  設計上の脆弱性やソースコードのバグ |
|  コンテナ(Container)  |  イメージやコンテナランタイムの設計上の脆弱性や設定ミス  |
|  クラスタ(Cluster)  |  クラスタを構築するコンポーネントの設計上の脆弱性や設定ミス  |
|  クラウド(Cloud)  |  ハイパーバイザやハードウェアの故障や設計上の脆弱性  |

↑の4つの層のうち、Kubernetesの機能で対策できるのはコンテナとクラスタの2層となる

・Kubernetesのセキュリティ階層

|  k8s上の層名  |  対策  |
| ---- | ---- |
|  コンテナ  |  ResourceRequirements, SecurityContext, PodSecurityPolicy, LimitRanger  |
|  Pod  |  コンテナ層と同じ + Volumeによる資源の制限, NetworkPolicy, RBACによるAPIアクセス制限  |
|  Node  |  Taint, Toleration, NodeAffinityなどPodを配置するマシンの分離, DynamicAdmissionWebhookによるスケージュリングポリシの強制  |
|  Namespace |  RBACによるAPIアクセス制限, ResourceQuotaによるリソース使用量の制限  |
|  Cluster |  RBACによるAPIアクセス制限, etcdの暗号化, システムコンポーネント間通信の暗号化  |
