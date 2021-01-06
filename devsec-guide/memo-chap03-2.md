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

#### 3.7 コンテナを監視する
3.7.1 コンテナのログを収集する

コンテナのログを表示する

```
[root@centos8 ~]# docker run --name hello hello-world

Hello from Docker!
...
[root@centos8 ~]# docker logs hello

Hello from Docker!
...
```

ログファイルを確認する

```
[root@centos8 ~]# docker inspect hello | grep -i logpath
        "LogPath": "/var/lib/docker/containers/f3b94d7b4c6050a44c6669c6e0efd1dd85a9b815a6a0b0bce8d73ebcdccde816/f3b94d7b4c6050a44c6669c6e0efd1dd85a9b815a6a0b0bce8d73ebcdccde816-json.log",
[root@centos8 ~]# cat /var/lib/docker/containers/f3b94d7b4c6050a44c6669c6e0efd1dd85a9b815a6a0b0bce8d73ebcdccde816/f3b94d7b4c6050a44c6669c6e0efd1dd85a9b815a6a0b0bce8d73ebcdccde816-json.log
{"log":"\n","stream":"stdout","time":"2021-01-06T02:09:40.413121525Z"}
{"log":"Hello from Docker!\n","stream":"stdout","time":"2021-01-06T02:09:40.413156361Z"}
...
```

ログドライバにfluentdを指定する  
fluentdを起動する  
https://kazuhira-r.hatenablog.com/entry/2019/01/02/125947
https://github.com/fluent/fluentd-docker-image/blob/master/README.md

```
[root@centos8 fluentd]# cat test.conf
<source>
  @type forward
</source>


<match docker.**>
  @type stdout
  @id output_stdout
</match>
[root@centos8 fluentd]# docker run -d -p 24224:24224 -p 24224:24224/udp -u fluent -v $(pwd):/fluentd/etc -e FLUENTD_CONF=test.conf --name fluentd fluentd
```

ログドライバにfluentdを指定してコンテナを実行する

```
[root@centos8 ~]# docker run --rm --log-driver fluentd --log-opt fluentd-address=127.0.0.1:24224 --log-opt tag=docker.{{.ImageName}}.{{.Name}}.{{.ID}} --name hello hello-world
```

fluentdに送信されるメッセージ
```
[root@centos8 fluentd]# docker logs fluentd | grep hello
2021-01-06 02:48:28.000000000 +0000 docker.hello-world.hello.5780a95f0026: {"container_id":"5780a95f0026e571ce300a14d1b9f88887b1dd49fd77463fcf7850cf486fbbbc","container_name":"/hello","source":"stdout","log":""}
2021-01-06 02:48:28.000000000 +0000 docker.hello-world.hello.5780a95f0026: {"container_id":"5780a95f0026e571ce300a14d1b9f88887b1dd49fd77463fcf7850cf486fbbbc","container_name":"/hello","source":"stdout","log":"Hello from Docker!"}
2021-01-06 02:48:28.000000000 +0000 docker.hello-world.hello.5780a95f0026: {"log":"This message shows that your installation appears to be working correctly.","container_id":"5780a95f0026e571ce300a14d1b9f88887b1dd49fd77463fcf7850cf486fbbbc","container_name":"/hello","source":"stdout"}
```

3.7.2 コンテナに関するイベントを収集する

```
[root@centos8 ~]# docker events
2021-01-06T11:17:27.138033358+09:00 container create 4941291b345038e7453e13cfd45df0c92ad41e15320eb637d18bc7b991479fdf (image=alpine, name=tender_feynman)
2021-01-06T11:17:27.139492582+09:00 container attach 4941291b345038e7453e13cfd45df0c92ad41e15320eb637d18bc7b991479fdf (image=alpine, name=tender_feynman)
2021-01-06T11:17:27.158611312+09:00 network connect a2b3d6f880fc5d1f8ce99b68b7c660b97144e3d3455802844696a05d41697f62 (container=4941291b345038e7453e13cfd45df0c92ad41e15320eb637d18bc7b991479fdf, name=bridge, type=bridge)
2021-01-06T11:17:27.446711564+09:00 container start 4941291b345038e7453e13cfd45df0c92ad41e15320eb637d18bc7b991479fdf (image=alpine, name=tender_feynman)
2021-01-06T11:17:27.447355558+09:00 container resize 4941291b345038e7453e13cfd45df0c92ad41e15320eb637d18bc7b991479fdf (height=30, image=alpine, name=tender_feynman, width=120)
```

