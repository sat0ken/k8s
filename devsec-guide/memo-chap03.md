### 第3章 ランタイムのセキュリティTips

#### Docker APIエンドポイントを保護する

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
