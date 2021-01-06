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


3.7.3 コンテナの挙動を監視する(Falco)

https://falco.org/docs/

CentOS8にFalcoをインストールする
https://falco.org/docs/getting-started/installation/

```
# yum install -y epel-release
# yum install -y make dkms
# yum -y install kernel-devel-$(uname -r)
# rpm --import https://falco.org/repo/falcosecurity-3672BA8F.asc
# curl -s -o /etc/yum.repos.d/falcosecurity.repo https://falco.org/repo/falcosecurity-rpm.repo
# yum -y install falco
```

コンテナを実行する

```
[root@centos8 ~]# docker run -it --rm alpine ash
/ # apk add nmap
fetch http://dl-cdn.alpinelinux.org/alpine/v3.12/main/x86_64/APKINDEX.tar.gz
fetch http://dl-cdn.alpinelinux.org/alpine/v3.12/community/x86_64/APKINDEX.tar.gz
(1/7) Installing libgcc (9.3.0-r2)
(2/7) Installing lua5.3-libs (5.3.5-r6)
(3/7) Installing libpcap (1.9.1-r2)
(4/7) Installing pcre (8.44-r0)
(5/7) Installing libssh2 (1.9.0-r1)
(6/7) Installing libstdc++ (9.3.0-r2)
(7/7) Installing nmap (7.80-r2)
Executing busybox-1.31.1-r16.trigger
OK: 20 MiB in 21 packages
/ # nmap 172.17.0.1
Starting Nmap 7.80 ( https://nmap.org ) at 2021-01-06 03:29 UTC
Nmap scan report for 172.17.0.1
Host is up (0.0000060s latency).
Not shown: 999 closed ports
PORT   STATE SERVICE
22/tcp open  ssh
MAC Address: 02:42:DD:DA:02:DC (Unknown)

Nmap done: 1 IP address (1 host up) scanned in 0.12 second
```

- ↑でターミナルが接続されたコンテナが起動したときのNotice
- パッケージマネージャが実行されたときのError
- nmapが実行されたときのNotice
- --privileged でコンテナが起動したときのNotice
- ホスト上のファイルシステムをbind-mountしたコンテナが起動したときのNotice
- コンテナ内の/usr/binに書き込みフラグつきのopenがは発生したときのError

```
[root@centos8 ~]# journalctl -f -u falco.service
...
Jan 06 12:24:57 centos8 falco[1119]: 12:24:57.027922903: Notice A shell was spawned in a container with an attached terminal (user=root user_loginuid=-1 <NA> (id=d67c857e03a1) shell=ash parent=<NA> cmdline=ash terminal=34816 container_id=d67c857e03a1 image=<NA>)
Jan 06 12:26:50 centos8 falco[1119]: 12:26:50.531988988: Error Package management process launched in container (user=root user_loginuid=-1 command=apk add nmap container_id=d67c857e03a1 container_name=distracted_leavitt image=alpine:latest)
Jan 06 12:29:21 centos8 falco[1119]: 12:29:21.273035150: Notice Network tool launched in container (user=root user_loginuid=-1 command=nmap 172.17.0.1 parent_process=ash container_id=d67c857e03a1 container_name=distracted_leavitt image=alpine:latest)
...
Jan 06 12:32:42 centos8 falco[1119]: 12:32:42.724587617: Notice Privileged container started (user=root user_loginuid=0 command=container:b52a01491014 hungry_hugle (id=b52a01491014) image=alpine:latest)
...
Jan 06 12:34:55 centos8 falco[1119]: 12:34:55.402934796: Notice Container with sensitive mount started (user=root user_loginuid=0 command=container:0be4616ea92a zealous_cartwright (id=0be4616ea92a) image=alpine:latest mounts=/etc:/host-etc::true:rprivate)
...
Jan 06 12:36:32 centos8 falco[1119]: 12:36:32.922479690: Error File below a known binary directory opened for writing (user=root user_loginuid=-1 command=touch /usr/bin/foo file=/usr/bin/foo parent=ash pcmdline=ash gparent=<NA> container_id=0be4616ea92a image=alpine)
```

Falcoはホストのプロセスが/usr/bin以下に書き込みしたことを検知することはできるが、コンテナが/usr/binをbind-mountして/usr/binに書き込んだ場合は検知されない  
→openシステムコールの文字列を比較しているため


```
[root@centos8 ~]# docker run -it --rm -v /usr/bin:/host-usr-bin alpine ash
/ # cd host-usr-bin/
/host-usr-bin # touch hoge
...

↑を検知されない
[root@centos8 ~]# journalctl -f -u falco.service
```

3.7.4 ファイルアクセスを監視する(auditd)  
auditdを利用するとFalcoでは検知できないコンテナのプロセスがホストのファイルシステムにアクセスしたことを検知できる

auditdの設定

```
[root@centos8 ~]# cat /etc/audit/rules.d/audit.rules | tail -n 1
-w /usr/bin -p wa -k usrbin
```

bind-mountして起動したコンテナから/usr/binに書き込み

```
[root@centos8 ~]# docker run -it --rm -v /usr/bin:/host-usr-bin alpine ash
/ # cd /host-usr-bin/
/host-usr-bin # touch hoge
```

audit.logで検知

```
type=SYSCALL msg=audit(1609911221.106:168): arch=c000003e syscall=2 success=yes exit=3 a0=7ffdfc691f57 a1=42 a2=1b6 a3=0 items=2 ppid=2554 pid=2592 auid=4294967295 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=pts0 ses=4294967295 comm="touch" exe="/bin/busybox" subj=system_u:system_r:spc_t:s0 key="usrbin"ARCH=x86_64 SYSCALL=open AUID="unset" UID="root" GID="root" EUID="root" SUID="root" FSUID="root" EGID="root" SGID="root" FSGID="root"
type=CWD msg=audit(1609911221.106:168): cwd="/host-usr-bin"
type=PATH msg=audit(1609911221.106:168): item=0 name="/host-usr-bin" inode=16797827 dev=fd:00 mode=040555 ouid=0 ogid=0 rdev=00:00 obj=system_u:object_r:bin_t:s0 nametype=PARENT cap_fp=0 cap_fi=0 cap_fe=0 cap_fver=0 cap_frootid=0OUID="root" OGID="root"
type=PATH msg=audit(1609911221.106:168): item=1 name="hoge" inode=16799827 dev=fd:00 mode=0100644 ouid=0 ogid=0 rdev=00:00 obj=system_u:object_r:bin_t:s0 nametype=CREATE cap_fp=0 cap_fi=0 cap_fe=0 cap_fver=0 cap_frootid=0OUID="root" OGID="root"
```

3.7.5 コンテナのリソース使用状況を監視する
Skip

#### 3.8 設定を検証する
3.8.1 Docker Bench for Security

Docker Bench for Securityを利用すると、Dockerの設定がCIS Docker Benchmarkに沿っているか検証できる  
https://github.com/docker/docker-bench-security  
https://www.cisecurity.org/benchmark/docker/

実行する

```
[root@centos8 ~]# git clone https://github.com/docker/docker-bench-security
Cloning into 'docker-bench-security'...
remote: Enumerating objects: 23, done.
remote: Counting objects: 100% (23/23), done.
remote: Compressing objects: 100% (21/21), done.
remote: Total 2085 (delta 8), reused 6 (delta 2), pack-reused 2062
Receiving objects: 100% (2085/2085), 2.95 MiB | 1.44 MiB/s, done.
Resolving deltas: 100% (1456/1456), done.
[root@centos8 ~]# cd docker-bench-security/
[root@centos8 docker-bench-security]# ./docker-bench-security.sh
# ------------------------------------------------------------------------------
# Docker Bench for Security v1.3.5
#
# Docker, Inc. (c) 2015-
#
# Checks for dozens of common best-practices around deploying Docker containers in production.
# Inspired by the CIS Docker Benchmark v1.2.0.
# ------------------------------------------------------------------------------

Initializing Wed Jan  6 15:37:27 JST 2021


[INFO] 1 - Host Configuration

.....

[INFO] 8 - Docker Enterprise Configuration
[INFO]   * Community Engine license, skipping section 8

[INFO] Checks: 76
[INFO] Score: 38
```

