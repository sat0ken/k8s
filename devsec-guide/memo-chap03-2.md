### 第3章 ランタイムのセキュリティTips
#### 3.6 代替ランタイムを利用する

3.6.1 Kata Containers

Kata Containerのインストール
※CentOS8に入れたけどサポートされてはいない

https://github.com/kata-containers/documentation/blob/master/install/centos-installation-guide.md

```
# yum -y install yum-utils
# yum-config-manager --add-repo http://download.opensuse.org/repositories/home:/katacontainers:/releases:/x86_64:/stable-1.11/CentOS_7/home:katacontainers:releases:x86_64:stable-1.11.repo
# yum -y install kata-runtime kata-proxy kata-shim
```

以下の設定をしてDockerを再起動する

```
# cat /etc/docker/daemon.json
{
    "runtimes": {
        "kata": {
            "path": "/usr/bin/kata-runtime"
        }
    }
}
# systemctl restart docker
# docker info | grep Runtimes
 Runtimes: kata runc
```

Kata Containersを使ってコンテナを起動する

```
[root@centos8 ~]# uname -r
4.18.0-193.14.2.el8_2.x86_64
[root@centos8 ~]# docker run -it --rm --runtime=kata alpine ash
/ # uname -a
Linux 9bcca8eb885b 5.4.32-11.1.container #1 SMP Thu Jan 1 00:00:00 UTC 1970 x86_64 Linux
```
