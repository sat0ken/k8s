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

#### 5.2 ミスや攻撃から守るAPIのアクセス制御

・k8s APIのアクセス制御は3段階に分かれる

1. 認証
   アクセスしてきたアカウントが誰であるか資格情報をもとに判定する
2. 認可
   認証されたユーザが該当リソースの操作権限を有しているか判定する
3. 受付制御
   リクエスト内容の検証やポリシに応じたリクエスト内容の書き換えが可能

#### 5.3 認証モジュールの選び方と使い方
#### 5.3.1 アカウント種別と対応する認証モジュールの違い

APIサーバの認証対象

- ユーザ   : 管理者など人間
- サービス : CIや監視ツールなどプログラム

k8sにユーザアカウントの管理機能はないので外部の認証システムなどで実施、サービスは提供されている管理用APIリソースを使う

#### 5.3.3 OpenID Connectによるユーザ認証

OpenID ConnectのID tokenをAPIサーバの認証に利用するモジュール、別途IDプロバイダが必要

#### 5.3.4 Webhook Token Authenticationを利用した外部認証基盤との連携

Webhook Token Authenticationを利用すると外部認証基盤でTokenの検証が実施できる
