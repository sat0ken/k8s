### イメージのセキュリティTips
#### 4.1 DockerfileからプライベートなGitやS3にアクセスする

- SSHの秘密鍵やAWSのシークレットキーなどはイメージに含めないように注意する必要がある
- Dokckerfileでrmコマンドで消してもイメージレイヤ内にクレデンシャルが残ってしまう
- imageをアーカイブとして出力すると見えてしまう


RUN命令でrmコマンドを実行してもファイルは最初のイメージレイヤに残っており削除されない

```
[root@centos8 alpine]# echo "some secret data" > .some-secret
[root@centos8 alpine]# cat Dockerfile
FROM alpine:latest
COPY .some-secret /root/.some-secret
RUN echo "bad example!"; rm -f /root/.some-secret
[root@centos8 alpine]# docker build -t foo .
Sending build context to Docker daemon  3.072kB
Step 1/3 : FROM alpine:latest
 ---> 389fef711851
Step 2/3 : COPY .some-secret /root/.some-secret
 ---> a53e7d87e978
Step 3/3 : RUN echo "bad example!"; rm -f /root/.some-secret
 ---> Running in f7d04217bae1
bad example!
Removing intermediate container f7d04217bae1
 ---> 1a6c17f896e2
Successfully built 1a6c17f896e2
Successfully tagged foo:latest
[root@centos8 alpine]# docker save foo | tar xv
1a6c17f896e21335846c49eca11a65bd595ea695160211a7a3ea6f3e3a729997.json
391b706c609789fcdc6960afe75543a1e3928f9651ed9be0f8a9f65511e190bb/
391b706c609789fcdc6960afe75543a1e3928f9651ed9be0f8a9f65511e190bb/VERSION
391b706c609789fcdc6960afe75543a1e3928f9651ed9be0f8a9f65511e190bb/json
391b706c609789fcdc6960afe75543a1e3928f9651ed9be0f8a9f65511e190bb/layer.tar
5481ac9a0febdf4562f887b92a5b35d7fcc420dd0a26d364e2be2a5c2d844e79/
5481ac9a0febdf4562f887b92a5b35d7fcc420dd0a26d364e2be2a5c2d844e79/VERSION
5481ac9a0febdf4562f887b92a5b35d7fcc420dd0a26d364e2be2a5c2d844e79/json
5481ac9a0febdf4562f887b92a5b35d7fcc420dd0a26d364e2be2a5c2d844e79/layer.tar
55efeb458a0e95ebbf71840d8ba58be6ae4cfc764c271e5a421c940ddf78323a/
55efeb458a0e95ebbf71840d8ba58be6ae4cfc764c271e5a421c940ddf78323a/VERSION
55efeb458a0e95ebbf71840d8ba58be6ae4cfc764c271e5a421c940ddf78323a/json
55efeb458a0e95ebbf71840d8ba58be6ae4cfc764c271e5a421c940ddf78323a/layer.tar
manifest.json
repositories
[root@centos8 alpine]# tar xf 391b706c609789fcdc6960afe75543a1e3928f9651ed9be0f8a9f65511e190bb/layer.tar
[root@centos8 alpine]# cat root/.some-secret
some secret data
```

#### 4.1.1 シークレットファイルを扱う(docker build --secret)

docker build --secret を有効化する

```
[root@centos8 ~]# cat /etc/docker/daemon.json | grep buildkit
    "features": {"buildkit": true},
[root@centos8 alpine]# systemctl restart docker
```

SSHにアクセス可能なDockerfileとdocker build --secret コマンドの使用

```
[root@centos8 ubuntu]# cat Dockerfile
# syntax=docker/dockerfile:1.1-experimental

FROM ubuntu:18.04
RUN apt-get update && apt-get install -y openssh-client && \
 useradd -m -u 1000 testusr
USER testusr
RUN mkdir -p -m 0700 /home/testusr/.ssh
RUN --mount=type=secret,id=ssh,target=/home/testusr/.ssh/id_rsa,uid=1000
[root@centos8 ubuntu]# docker build -t foo --secret id=ssh,src=/root/.ssh/id_rsa .
[+] Building 3.8s (12/12) FINISHED
 => [internal] load .dockerignore                                                                                  0.0s
 => => transferring context: 2B                                                                                    0.0s
 => [internal] load build definition from Dockerfile                                                               0.0s
 => => transferring dockerfile: 377B                                                                               0.0s
 => resolve image config for docker.io/docker/dockerfile:1.1-experimental                                          1.9s
 => CACHED docker-image://docker.io/docker/dockerfile:1.1-experimental@sha256:de85b2f3a3e8a2f7fe48e8e84a65f6fdd5c  0.0s
 => [internal] load build definition from Dockerfile                                                               0.0s
 => => transferring dockerfile: 377B                                                                               0.0s
 => [internal] load .dockerignore                                                                                  0.0s
 => [internal] load metadata for docker.io/library/ubuntu:18.04                                                    0.7s
 => [1/4] FROM docker.io/library/ubuntu:18.04@sha256:fd25e706f3dea2a5ff705dbc3353cf37f08307798f3e360a13e9385840f7  0.0s
 => CACHED [2/4] RUN apt-get update && apt-get install -y openssh-client &&  useradd -m -u 1000 testusr            0.0s
 => [3/4] RUN mkdir -p -m 0700 /home/testusr/.ssh                                                                  0.3s
 => [4/4] RUN --mount=type=secret,id=ssh,target=/home/testusr/.ssh/id_rsa,uid=1000                                 0.5s
 => exporting to image                                                                                             0.1s
 => => exporting layers                                                                                            0.0s
 => => writing image sha256:fc93953551dd45adf4191f44a7e8be28f71d1c1ce35db02e97cd6d6c4dd49dde                       0.0s
 => => naming to docker.io/library/foo
```

