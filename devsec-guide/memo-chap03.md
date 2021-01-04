### 第3章 ランタイムのセキュリティTips

#### 3.1 Docker APIエンドポイントを保護する

TLSクライアント認証による保護

```
[root@centos8 cert]# CAROOT=. mkcert dockerd.example.com
Created a new local CA 💥
Note: the local CA is not installed in the system trust store.
Run "mkcert -install" for certificates to be trusted automatically ⚠️

Created a new certificate valid for the following names 📜
 - "dockerd.example.com"

The certificate is at "./dockerd.example.com.pem" and the key at "./dockerd.example.com-key.pem" ✅

It will expire on 29 March 2023 🗓

[root@centos8 cert]# CAROOT=. mkcert -client client
Note: the local CA is not installed in the system trust store.
Run "mkcert -install" for certificates to be trusted automatically ⚠️

Created a new certificate valid for the following names 📜
 - "client"

The certificate is at "./client-client.pem" and the key at "./client-client-key.pem" ✅

It will expire on 29 March 2023 🗓

[root@centos8 cert]# ls
client-client-key.pem  client-client.pem  dockerd.example.com-key.pem  dockerd.example.com.pem  rootCA-key.pem  rootCA.pem
[root@centos8 cert]# cp rootCA.pem dockerd.example.com.pem dockerd.example.com-key.pem /etc/docker/
```

/etc/docker/daemon.jsonを作成する

```
[root@centos8 ~]# cat /etc/docker/daemon.json 
{
    "tlsverify": true,
    "tlscacert": "/etc/docker/rootCA.pem",
    "tlscert": "/etc/docker/dockerd.example.com.pem",
    "tlskey": "/etc/docker/dockerd.example.com-key.pem"
}
```

/etc/systemd/system/docker-tcp.socketを作成する

```
[root@centos8 ~]# cat /etc/systemd/system/docker-tcp.socket 
[Unit]
Description=Docker (TCP)

[Socket]
ListenStream=2376
Service=docker.service

[Install]
WantedBy=sockets.target
```

dockerを起動する

```
[root@centos8 ~]# systemctl start docker-tcp.socket
[root@centos8 ~]# systemctl start docker
```

クライアント証明書を.dockerフォルダに配置する

```
[root@centos8 ~]# export DOCKER_HOST=tcp://dockerd.example.com:2376
[root@centos8 ~]# export DOCKER_TLS_VERIFY=1
[root@centos8 cert]# cp rootCA.pem /root/.docker/ca.pem
[root@centos8 cert]# cp client-client.pem /root/.docker/cert.pem
[root@centos8 cert]# cp client-client-key.pem /root/.docker/key.pem
```

TLSクライアント認証をしてdockerデーモンに接続できることを確認する

```
[root@centos8 ~]# docker info
Client:
 Context:    default
 Debug Mode: false
 Plugins:
  app: Docker App (Docker Inc., v0.9.1-beta3)
  buildx: Build with BuildKit (Docker Inc., v0.5.0-docker)

Server:
 Containers: 0
  Running: 0
  Paused: 0
  Stopped: 0
 Images: 9
 Server Version: 18.09.1
 Storage Driver: overlay2
  Backing Filesystem: xfs
  Supports d_type: true
  Native Overlay Diff: true
 Logging Driver: json-file
 Cgroup Driver: cgroupfs
 Plugins:
  Volume: local
  Network: bridge host macvlan null overlay
  Log: awslogs fluentd gcplogs gelf journald json-file local logentries splunk syslog
 Swarm: inactive
 Runtimes: runc
 Default Runtime: runc
 Init Binary: docker-init
 containerd version: c4446665cb9c30056f4998ed953e6d4ff22c7c39
 runc version: 4fc53a81fb7c994640722ac585fa9ca548971871
 init version: fec3683
 Security Options:
  seccomp
   Profile: default
 Kernel Version: 4.18.0-193.14.2.el8_2.x86_64
 Operating System: CentOS Linux 8 (Core)
 OSType: linux
 Architecture: x86_64
 CPUs: 2
 Total Memory: 978.3MiB
 Name: centos8
 ID: J2YF:DOFE:WITX:YJWR:TNUD:ZSEN:ZRJD:GKLF:ZDKS:YYAB:CGO4:E6KH
 Docker Root Dir: /var/lib/docker
 Debug Mode: false
 Registry: https://index.docker.io/v1/
 Labels:
 Experimental: false
 Insecure Registries:
  127.0.0.0/8
 Live Restore Enabled: false
 Product License: Community Engine
```

コンテナ内からDocker APIを呼び出す

Docker-outside-of-Docker

```
[root@centos8 ~]# docker run --rm -v /var/run/docker.sock:/var/run/docker.sock docker:19.03.3 docker run --rm hello-world

Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
    (amd64)
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.

To try something more ambitious, you can run an Ubuntu container with:
 $ docker run -it ubuntu bash

Share images, automate workflows, and more with a free Docker ID:
 https://hub.docker.com/

For more examples and ideas, visit:
 https://docs.docker.com/get-started/
```

Docker-in-Docker

```
[root@centos8 ~]# docker run -d --name dind --privileged docker:19.03.3-dind
228285f40bc08222b1d1fc28e489c399a674660cf1f13370d5446c8a9b9a3e53
[root@centos8 ~]# docker exec dind docker run --rm hello-world
Unable to find image 'hello-world:latest' locally
latest: Pulling from library/hello-world
0e03bdcc26d7: Pulling fs layer
0e03bdcc26d7: Verifying Checksum
0e03bdcc26d7: Download complete
0e03bdcc26d7: Pull complete
Digest: sha256:1a523af650137b8accdaed439c17d684df61ee4d74feac151b5b337bd29e7eec
Status: Downloaded newer image for hello-world:latest

Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
    (amd64)
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.

To try something more ambitious, you can run an Ubuntu container with:
 $ docker run -it ubuntu bash

Share images, automate workflows, and more with a free Docker ID:
 https://hub.docker.com/

For more examples and ideas, visit:
 https://docs.docker.com/get-started/
```

Docker in Docker コンテナに他のコンテナから接続する

```
[root@centos8 ~]# docker network create dindnet
efd7386505f1ba3e3533d912e29b28ef316ce4e6a8f830bb4aef86e69e99a139
[root@centos8 ~]# docker run -d --privileged --name dind --hostname dind --network dindnet docker:19.03.3-dind
c46f337d3121bb8d4ac77598e2a3047fd6640172dadc71f4cfb3344702179276
[root@centos8 ~]# mkdir dind
[root@centos8 ~]# cd dind/
[root@centos8 dind]# docker cp dind:/certs/client .
[root@centos8 dind]# ls client/
ca.pem  cert.pem  csr.pem  key.pem  openssl.cnf
[root@centos8 dind]# docker run --rm --network dindnet -v $(pwd)/client:/certs/client:ro -e DOCKER_HOST=tcp://dind:2376 docker:19.03.3 docker run --rm hello-world

Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
    (amd64)
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.

To try something more ambitious, you can run an Ubuntu container with:
 $ docker run -it ubuntu bash

Share images, automate workflows, and more with a free Docker ID:
 https://hub.docker.com/

For more examples and ideas, visit:
 https://docs.docker.com/get-started/

[root@centos8 dind]#
```

#### 3.2 コンテナ実行ユーザを変更する

UID:GIDを指定してコンテナを実行する

```
[root@centos8 ~]# docker run -it --user 1000:1000 alpine ash
/ $ id
uid=1000 gid=1000
```

Dockerfileでユーザを指定する

```
[root@centos8 alpine]# cat Dockerfile 
FROM alpine:latest
RUN adduser -D exampleuser
USER exampleuser
[root@centos8 alpine]# docker build -q -t foo . && docker run -it --rm foo
sha256:6c558fd88cc553cafaebe83d37e3fe764007698cf1c4a659c300fd835b387ec6
/ $ id
uid=1000(exampleuser) gid=1000(exampleuser)
/ $ env
HOSTNAME=e8941102c656
SHLVL=1
HOME=/home/exampleuser
TERM=xterm
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
PWD=/
```

非rootユーザをrootに見せかける

User Namespaceを有効にする
userns-remapにdefaultを指定すると、Dockerは自動的にホストに`dockremap`ユーザを作成してホストの/etc/subuidと/etc/subgidに設定を作成する

```
[root@centos8 ~]# cat /etc/docker/daemon.json | grep userns
        "userns-remap": "default"
[root@centos8 ~]# systemctl restart docker
[root@centos8 ~]# cat /etc/subuid | grep dock
dockremap:165536:65536
[root@centos8 ~]# cat /etc/subgid | grep dock
dockremap:165536:65536
[root@centos8 ~]#
```

↑ではコンテナ内のuid=0(root)にホストのuid=165536がマッピングされる。

```
[root@centos8 ~]# docker run -it --rm alpine ash
/ #
/ # id
uid=0(root) gid=0(root) groups=0(root),1(bin),2(daemon),3(sys),4(adm),6(disk),10(wheel),11(floppy),20(dialout),26(tape),27(video)

ホストでpsコマンドを実行
[root@centos8 ~]# ps -aux | grep -e 165536 -e USER| grep -v grep
USER         PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
165536   3066359  0.0  0.1   1648  1064 pts/0    Ss+  12:40   0:00 ash
```

ホストのファイルシステムをbind-mountしても`Permission denied`になる

```
[root@centos8 ~]# docker run -it --rm alpine ash
/ #
/ # id
uid=0(root) gid=0(root) groups=0(root),1(bin),2(daemon),3(sys),4(adm),6(disk),10(wheel),11(floppy),20(dialout),26(tape),27(video)
/ # ps -ef
PID   USER     TIME  COMMAND
    1 root      0:00 ash
    7 root      0:00 ps -ef
/ # exit
[root@centos8 ~]# docker run -it --rm -v /:/host --user 65534:65534 alpine ash
~ $ id
uid=65534(nobody) gid=65534(nobody)
~ $ ls -ln /host/etc/shadow
----------    1 65534    65534          857 Jan  4 03:30 /host/etc/shadow
~ $ cat /host/etc/shadow
cat: can't open '/host/etc/shadow': Permission denied
```

見せかけのrootでも同じく

```
[root@centos8 ~]# docker run -it --rm -v /:/host alpine ash
/ # id
uid=0(root) gid=0(root) groups=0(root),1(bin),2(daemon),3(sys),4(adm),6(disk),10(wheel),11(floppy),20(dialout),26(tape),27(video)
/ # cat /host/etc/shadow
cat: can't open '/host/etc/shadow': Permission denied
```

ランタイム自体を非rootで動かす
Rootless版のDockerをインストールする

https://docs.docker.com/engine/security/rootless/

```
[root@centos8 ~]# su - user01
Last login: Mon Jan  4 14:53:46 JST 2021 on pts/0
[user01@centos8 ~]$ id -u
1001
[user01@centos8 ~]$ grep ^$(whoami): /etc/subuid
user01:231072:65536
[user01@centos8 ~]$ grep ^$(whoami): /etc/subgid
user01:231072:65536
[user01@centos8 ~]$ curl -fsSL https://get.docker.com/rootless | sh
# Installing stable version 20.10.1
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 65.7M  100 65.7M    0     0  4398k      0  0:00:15  0:00:15 --:--:-- 5034k
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 19.0M  100 19.0M    0     0  4597k      0  0:00:04  0:00:04 --:--:-- 4597k
+ PATH=/home/user01/bin:/home/user01/.local/bin:/home/user01/bin:/usr/local/bin:/usr/bin:/usr/local/sbin:/usr/sbin
+ /home/user01/bin/dockerd-rootless-setuptool.sh install
[INFO] systemd not detected, dockerd-rootless.sh needs to be started manually:

PATH=/home/user01/bin:/sbin:/usr/sbin:$PATH dockerd-rootless.sh

[INFO] Make sure the following environment variables are set (or add them to ~/.bashrc):

# WARNING: systemd not found. You have to remove XDG_RUNTIME_DIR manually on every logout.
export XDG_RUNTIME_DIR=/home/user01/.docker/run
export PATH=/home/user01/bin:$PATH
export DOCKER_HOST=unix:///home/user01/.docker/run/docker.sock
```

dockerdが非rootユーザ権限で動作していることを確認する

```
[user01@centos8 ~]$ ./bin/dockerd-rootless.sh &
[user01@centos8 ~]$ ps xf -o user,pid,comm
USER         PID COMMAND
user01   3067200 bash
user01   3067479  \_ rootlesskit
user01   3067487  |   \_ exe
user01   3067515  |   |   \_ dockerd
user01   3067531  |   |       \_ containerd
user01   3067497  |   \_ vpnkit
user01   3067691  \_ ps
[user01@centos8 ~]$ docker run  --rm hello-world
Unable to find image 'hello-world:latest' locally
latest: Pulling from library/hello-world
0e03bdcc26d7: Pull complete
Digest: sha256:1a523af650137b8accdaed439c17d684df61ee4d74feac151b5b337bd29e7eec
Status: Downloaded newer image for hello-world:latest
time="2021-01-04T15:04:30.554337887+09:00" level=info msg="starting signal loop" namespace=moby path=/home/user01/.docker/run/docker/containerd/daemon/io.containerd.runtime.v2.task/moby/6d5ceaf866571f147052d08b590de37d9fefb72529c28343acded64d2f91ba52 pid=3067731

Hello from Docker!
This message shows that your installation appears to be working correctly.
```

#### 3.3 ケーパビリティやシステムコールを制限する

特定のケーパビリティを削除 = `--cap-drop`
必要なケーパビリティを追加 = `--cap-add` 

必要なケーパビリティのみ明示的に追加する

```
[root@centos8 ~]# docker run -d -p 80:80 --cap-drop all --cap-add chown --cap-add setuid --cap-add setgid --cap-add net_bind_service nginx:1.17.3-alpine
Unable to find image 'nginx:1.17.3-alpine' locally
1.17.3-alpine: Pulling from library/nginx
9d48c3bd43c5: Pull complete
b6dac14ba0a9: Pull complete
Digest: sha256:99be6ae8d32943b676031b3513782ad55c8540c1d040b1f7b8c335c67a241b06
Status: Downloaded newer image for nginx:1.17.3-alpine
d263e46a90fb790acab02b0194b4e579e5f51dfc126af0788706273884156715
[root@centos8 ~]# curl -s localhost:80 | grep Welcome
<title>Welcome to nginx!</title>
```

CAP_NET_RAWがないとpingができない

```
[root@centos8 ~]# docker run --rm --cap-drop all alpine ping 8.8.8.8
PING 8.8.8.8 (8.8.8.8): 56 data bytes
ping: permission denied (are you root?)
```

GID 0~1000 にCAP_NET_RAW無しでのpingを許可する

```
[root@centos8 ~]# docker run --rm --cap-drop all --sysctl "net.ipv4.ping_group_range=0 1000" alpine ping 8.8.8.8
PING 8.8.8.8 (8.8.8.8): 56 data bytes
64 bytes from 8.8.8.8: seq=0 ttl=42 time=11.480 ms
64 bytes from 8.8.8.8: seq=1 ttl=42 time=10.045 ms
```

不要なシステムコールの呼び出しを禁止する

