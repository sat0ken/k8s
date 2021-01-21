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

RoleBindingとClusterRoleBinding
ClusterRoleBindingで作成するとNamespaceをまたいでも同じ権限を付与することができる

RoleBindingの作成
```
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: sample-rolebinding
  namespace: default
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: sample-role
subjects:
- kind: ServiceAccount
  name: sample-serviceaccount
  namespace: default
```

ClusterRoleBindingの作成
```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: sample-clusterrolebinding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: sample-clusterrole
subjects:
- kind: ServiceAccount
  name: sample-serviceaccount
  namespace: default
```

RBACのテスト
検証に利用するPod
```
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: sample-kubectl
  name: sample-kubectl
spec:
  serviceAccountName: sample-serviceaccount
  containers:
  - command:
    - sleep
    - "3600"
    image: lachlanevenson/k8s-kubectl:v1.20.2
    name: sample-kubectl
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Never
status: {}
```

Deploymentの作成→権限付与しているのでOK
```
$ k exec sample-kubectl -- kubectl create deployment nginx --image=nginx:1.16
deployment.apps/nginx created
```

ReplicaSetのスケーリング→ReplicaSetの権限付与していないので×
```
$ k exec sample-kubectl -- kubectl scale replicaset nginx-6d4cf56db6 --replicas 2
Error from server (Forbidden): replicasets.apps "nginx-6d4cf56db6" is forbidden: User "system:serviceaccount:default:sample-serviceaccount" cannot patch resource "replicasets/scale" in API group "apps" in the namespace "default"
```

Deploymentのスケーリング→権限付与しているのでOK
```
$ k exec sample-kubectl -- kubectl scale deployment nginx --replicas 2
deployment.apps/nginx scaled
```

別のNamespaceを作成してClusterRoleBindの動作確認
ClusterRoleではDeploymentの作成権限を付与していないので×
```
$ kubectl create namespace sample-rbac
namespace/sample-rbac created
$ k exec sample-kubectl -- kubectl create deployment nginx --image=nginx:1.16 -n sample-rbac
error: failed to create deployment: deployments.apps is forbidden: User "system:serviceaccount:default:sample-serviceaccount" cannot create resource "deployments" in API group "apps" in the namespace "sample-rbac"
$ k exec sample-kubectl -- kubectl get deploy -n sample-rbac
No resources found in sample-rbac namespace.
```

`kubectl auth can-i` コマンドを利用して確認
```
$ k exec sample-kubectl -- kubectl auth can-i create deployment
yes
$ k exec sample-kubectl -- kubectl auth can-i create deployment -n sample-rbac
no
$ kubectl auth can-i create deployment -n sample-rbac
yes
```

`--as` オプションの利用
```
$ kubectl -n sample-rbac create deployment nginx --image=nginx:1.16 --as sample-serviceaccount
error: failed to create deployment: deployments.apps is forbidden: User "sample-serviceaccount" cannot create resource "deployments" in API group "apps" in the namespace "sample-rbac"
```

#### 13.3 SecurityContext
SecurityContextに設定可能な項目

| 設定項目   | 内容       | 
| ------ | ---------- | 
| allowPrivilegeEscalation | コンテナ実行時に親プロセス以上の権限を与えるかどうか       | 
| capabilities |   capabilitiesの追加と削除     | 
| privileged      | 特権コンテナとして実行 | 
| procMount   |    | 
| readOnlyRootFilesystem    | rootファイルシステムをReadOnlyにするかどうか       | 
| runAsGroup   | 実行するグループ   | 
| runAsNonRoot  | rootでの実行を拒否する   | 
| runAsUser | 実行するユーザ       | 
| seLinuxOptions | SELinuxのオプション  | 
| seccompProfile | seccompのオプション  | 

特権コンテナの作成
```
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: sample-privilegd
  name: sample-privilegd
spec:
  containers:
  - image: nginx:1.16
    name: sample-privilegd
    securityContext:
      privileged: true
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Never
status: {}
```

Capabilitiesの付与
```
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: sample-capabilities
  name: sample-capabilities
spec:
  containers:
  - image: nginx:1.16
    name: sample-capabilities
    securityContext:
      capabilities:
        add: ["SYS_ADMIN"]
        drop: ["AUDIT_WRITE"]
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Never
status: {}
```

rootファイルシステムのReadOnly化
```
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: sample-readonly
  name: sample-readonly
spec:
  containers:
  - image: alpine
    name: sample-readonly
    args:
    - sleep
    - "3600"
    securityContext:
      readOnlyRootFilesystem: true
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Never
status: {}
```
ReadOnlyなので書き込みできない
```
$ k exec sample-readonly -- sh -c "echo hoge >> hoge.txt"
sh: can't create hoge.txt: Read-only file system
```

#### 13.4 PodSecurityContext
PodSecurityContextに設定可能な項目

| 設定項目   | 内容       | 
| ------ | ---------- | 
| fsGroup | ファイルシステムのグループを指定       | 
| fsGroupChangePolicy | 所有権とパーミッションを指定       | 
| runAsGroup      | 実行グループ | 
| runAsNonRoot   | rootでの実行を拒否する   | 
| runAsUser      | 実行ユーザ | 
| seLinuxOptions   | SELinuxのオプション   | 
| seccompProfile   | seccompのオプション   | 
| supplementalGroups      | プライマリGIDに追加で付与するGIDのリストを指定 | 
| sysctls   | 上書きするカーネルパラメータを指定   | 

実行ユーザの変更
```
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: sample-runuser
  name: sample-runuser
spec:
  securityContext:
    runAsUser: 65534
    runAsGroup: 65534
    supplementalGroups:
    - 1001
    - 1002
  containers:
  - args:
    - sleep
    - "3600"
    image: alpine
    name: sample-runuser
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
```

```
$ k exec sample-runuser -- id
uid=65534(nobody) gid=65534(nobody) groups=1001,1002
```

rootユーザでの実行制限
```
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: sample-noroot
  name: sample-noroot
spec:
  securityContext:
    runAsNonRoot: true
  containers:
  - image: nginx:1.16
    name: sample-noroot
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
```

nginxがrootでの起動に失敗するのでFail
```
$ kubectl describe pod sample-noroot | tail -n 5
  Type     Reason     Age                From               Message
  ----     ------     ----               ----               -------
  Normal   Scheduled  56s                default-scheduler  Successfully assigned default/sample-noroot to kind-worker2
  Normal   Pulled     12s (x6 over 55s)  kubelet            Container image "nginx:1.16" already present on machine
  Warning  Failed     12s (x6 over 55s)  kubelet            Error: container has runAsNonRoot and image will run as root
```

ファイルシステムのグループ指定
```
apiVersion: v1
kind: Pod
metadata:
  name: sample-fsgroup
spec:
  securityContext:
    fsGroup: 1001
  containers:
  - image: nginx:1.16
    name: sample-fsgroup
    volumeMounts:
    - mountPath: /cache
      name: cache-volume
  volumes:
  - name: cache-volume
    emptyDir: {}
```

ディレクトリのパーミッションを確認
```
$ kubectl exec sample-fsgroup -- ls -ld /cache
drwxrwsrwx 2 root 1001 4096 Jan 18 08:35 /cache
```

sysctlによるカーネルパラメータの設定
```
apiVersion: v1
kind: Pod
metadata:
  name: sample-sysctl
spec:
  securityContext:
    sysctls:
    - name: net.core.somaxconn
      value: "12345"
  containers:
  - image: nginx:1.16
    name: sample-sysctl
```

```
$ k get pod
NAME            READY   STATUS            RESTARTS   AGE
sample-sysctl   0/1     SysctlForbidden   0          45s
```

カーネルパラメータをInit Containerで変更する
```
apiVersion: v1
kind: Pod
metadata:
  name: sample-sysctl-initcontainer
spec:
  initContainers:
  - name: initialize-sysctl
    image: busybox:1.27
    command:
    - /bin/sh
    - -c
    - sysctl -w net.core.somaxconn=12345
    securityContext:
      privileged: true
  containers:
  - image: nginx:1.16
    name: sample-sysctl-initcontainer
```

#### 13.5 PodSecurityPolicy
PodSecurityPolicyはk8sクラスタに対してセキュリティポリシにによる制限を行う

| 設定項目   | 内容       | 
| ------ | ---------- | 
|allowPrivilegeEscalation       | allowPrivilegeEscalationを許可するかどうか   |
|allowedCSIDrivers      |  許可するCSIDriver  |
|allowedCapabilities    |  許可するCapability  |
|allowedFlexVolumes     |    |
|allowedHostPaths       |  hostpathで利用可能なパスのホワイトリスト  |
|allowedProcMountTypes  |    |
|allowedUnsafeSysctls   |  利用を許可するunsafeなsysctl  |
|defaultAddCapabilities |  デフォルトで追加するCapability  |
|defaultAllowPrivilegeEscalation        |  デフォルトでallowPrivilegeEscalationを許可するかどうか  |
|forbiddenSysctls       |  利用を拒否するsysctl  |
|fsGroup        |  fsGroupで利用可能なGIDの範囲  |
|hostIPC        |  hostIPCの利用を許可するかどうか  |
|hostNetwork    |  hostNetworkの利用を許可するかどうか  |
|hostPID        |  hostPIDの利用を許可するかどうか  |
|hostPorts      |  hostPortの利用を許可するかどうか  |
|privileged     |  特権コンテナの利用可否  |
|readOnlyRootFilesystem |  readOnlyRootFilesystemを強制するかどうか  |
|requiredDropCapabilities       |  Dropする必要のあるCapability  |
|runAsGroup     |  実行可能なプライマリグループのGID  |
|runAsUser      |  実行可能なユーザのUID  |
|runtimeClass   |  利用可能なRuntimeClass  |
|seLinux        |  設定可能なSELinuxのラベル  |
|supplementalGroups     |  supplementalGroupsに設定可能なGID  |
|volumes        |  利用可能なVolumeプラグイン  |

PSPの有効化
api-serverの`--enable-admission-plugins` オプションにセット
```
# cat /etc/kubernetes/manifests/kube-apiserver.yaml | grep PodSecurity
    - --enable-admission-plugins=NodeRestriction,PodSecurityPolicy
```

PodSecurityPoicyによるPod作成の権限の付与
https://kubernetes.io/docs/concepts/policy/pod-security-policy/
```
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: sample-podsecuritypolicy
spec:
  privileged: false
  runAsUser:
    rule: RunAsAny
  allowPrivilegeEscalation: true
  allowedCapabilities:
  - '*'
  allowedHostPaths:
  - pathPrefix: "/etc"
  fsGroup:
    rule: RunAsAny
  seLinux:
    rule: RunAsAny
  supplementalGroups:
    rule: RunAsAny
  volumes:
  - '*'
```

ServiceAccountの作成とPodの作成可能な権限を付与
```
$ kubectl create serviceaccount psp-test
$ kubectl create rolebinding psp-test --clusterrole=edit --serviceaccount=default:psp-test
```

編集権限が付与されたデフォルトのService Accountを指定してPodを作成すると一致するPodSecurityPolicyが存在しないためPodの作成に失敗する
```
$ kubectl --as=system:serviceaccount:default:psp-test create -f- <<EOF
> apiVersion: v1
> kind: Pod
> metadata:
>   name: pause
> spec:
>   containers:
>     - name: pause
>       image: k8s.gcr.io/pause
> EOF
Error from server (Forbidden): error when creating "STDIN": pods "pause" is forbidden: PodSecurityPolicy: unable to admit pod: []
```

PSPを作成
PSPが紐づいたRoleを作成してService Accountと紐づける
```
$ kubectl apply -f sample-podsecuritypolicy.yaml
$ kubectl create role psp-test-role --verb=use --resource=podsecuritypolicy --resource-name=sample-podsecuritypolicy
$ kubectl create rolebinding psp-test-binding --role=psp-test-role --serviceaccount=default:psp-test

```

Podを作成してみる
普通のPodは作成できるが特権コンテナはPSPで拒否される
```
$ kubectl --as=system:serviceaccount:default:psp-test create -f- <<EOF
> apiVersion: v1
> kind: Pod
> metadata:
>   name: pause
> spec:
>   containers:
>     - name: pause
>       image: k8s.gcr.io/pause
> EOF
pod/pause created
$ kubectl --as=system:serviceaccount:default:psp-test create -f- <<EOF
> apiVersion: v1
> kind: Pod
> metadata:
>   name: privileged
s> spec:
>   containers:
>     - name: pause
>       image: k8s.gcr.io/pause
>       securityContext:
>         privileged: true
> EOF
Error from server (Forbidden): error when creating "STDIN": pods "privileged" is forbidden: PodSecurityPolicy: unable to admit pod: [spec.containers[0].securityContext.privileged: Invalid value: true: Privileged containers are not allowed]
```
