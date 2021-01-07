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
https://docs.docker.com/develop/develop-images/build_enhancements/

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

#### 4.2 コンテナ内で安全にイメージをbuildする
#### 4.2.1 buildkit

後でやる

#### 4.2.2 Kaniko

https://github.com/GoogleContainerTools/kaniko

後でやる

#### 4.3 イメージの脆弱性を検査する
#### 4.3.1 パッケージの脆弱性を検査する(Clair)

https://github.com/quay/clair
https://github.com/optiopay/klar

```
[root@centos8 ~]# export CLAIR_ADDR=http://127.0.0.1:6060
[root@centos8 ~]# ./klar nginx:1.17.3-alpine
clair timeout 1m0s
docker timeout: 1m0s
no whitelist file
Analysing 2 layers
Got results from Clair API v1
Found 2 vulnerabilities
Unknown: 2

CVE-2019-13627: [Unknown]
Found in: libgcrypt [1.8.4-r2]
Fixed By: 1.8.5-r0

https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2019-13627
-----------------------------------------
CVE-2019-18197: [Unknown]
Found in: libxslt [1.1.33-r1]
Fixed By: 1.1.33-r2

https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2019-18197
-----------------------------------------
```

#### 4.3.2 パッケージの脆弱性を検査する(Trivy)

https://github.com/aquasecurity/trivy

Trivyでnginx:1.17.3-alpineを検査する

```
[root@centos8 ~]# trivy nginx:1.17.3-alpine
2021-01-07T17:33:27.081+0900    INFO    Need to update DB
2021-01-07T17:33:27.082+0900    INFO    Downloading DB...
19.58 MiB / 19.58 MiB [------------------------------------------------------------------------] 100.00% 3.72 MiB p/s 5s
2021-01-07T17:33:35.545+0900    INFO    Detecting Alpine vulnerabilities...
2021-01-07T17:33:35.547+0900    INFO    Trivy skips scanning programming language libraries because no supported file was detected

nginx:1.17.3-alpine (alpine 3.10.2)
===================================
Total: 26 (UNKNOWN: 0, LOW: 2, MEDIUM: 15, HIGH: 9, CRITICAL: 0)

+---------------+------------------+----------+-------------------+---------------+---------------------------------------+
|    LIBRARY    | VULNERABILITY ID | SEVERITY | INSTALLED VERSION | FIXED VERSION |                 TITLE
  |
+---------------+------------------+----------+-------------------+---------------+---------------------------------------+
| freetype      | CVE-2020-15999   | MEDIUM   | 2.10.0-r0         | 2.10.0-r1     | freetype: Heap-based buffer
  |
|               |                  |          |                   |               | overflow due to integer
  |
|               |                  |          |                   |               | truncation in Load_SBit_Png
  |
|               |                  |          |                   |               | -->avd.aquasec.com/nvd/cve-2020-15999 |
+---------------+------------------+----------+-------------------+---------------+---------------------------------------+
| libcrypto1.1  | CVE-2020-1967    | HIGH     | 1.1.1c-r0         | 1.1.1g-r0     | openssl: Segmentation
  |
|               |                  |          |                   |               | fault in SSL_check_chain
  |
|               |                  |          |                   |               | causes denial of service
  |
|               |                  |          |                   |               | -->avd.aquasec.com/nvd/cve-2020-1967  |
+               +------------------+----------+                   +---------------+---------------------------------------+
|               | CVE-2019-1547    | MEDIUM   |                   | 1.1.1d-r0     | openssl: side-channel weak
  |
|               |                  |          |                   |               | encryption vulnerability
  |
|               |                  |          |                   |               | -->avd.aquasec.com/nvd/cve-2019-1547  |
+               +------------------+          +                   +               +---------------------------------------+
|               | CVE-2019-1549    |          |                   |               | openssl: information
  |
|               |                  |          |                   |               | disclosure in fork()
  |
|               |                  |          |                   |               | -->avd.aquasec.com/nvd/cve-2019-1549  |
+               +------------------+          +                   +---------------+---------------------------------------+
|               | CVE-2019-1551    |          |                   | 1.1.1d-r2     | openssl: Integer overflow in RSAZ     |
|               |                  |          |                   |               | modular exponentiation on x86_64      |
|               |                  |          |                   |               | -->avd.aquasec.com/nvd/cve-2019-1551  |
+               +------------------+          +                   +---------------+---------------------------------------+
|               | CVE-2020-1971    |          |                   | 1.1.1i-r0     | openssl: EDIPARTYNAME
  |
|               |                  |          |                   |               | NULL pointer de-reference
  |
|               |                  |          |                   |               | -->avd.aquasec.com/nvd/cve-2020-1971  |
+               +------------------+----------+                   +---------------+---------------------------------------+
|               | CVE-2019-1563    | LOW      |                   | 1.1.1d-r0     | openssl: information
  |
|               |                  |          |                   |               | disclosure in PKCS7_dataDecode        |
|               |                  |          |                   |               | and CMS_decrypt_set1_pkey
  |
|               |                  |          |                   |               | -->avd.aquasec.com/nvd/cve-2019-1563  |
+---------------+------------------+----------+-------------------+---------------+---------------------------------------+
| libgcrypt     | CVE-2019-13627   | MEDIUM   | 1.8.4-r2          | 1.8.5-r0      | libgcrypt: ECDSA timing attack        |
|               |                  |          |                   |               | allowing private key leak
  |
|               |                  |          |                   |               | -->avd.aquasec.com/nvd/cve-2019-13627 |
+---------------+------------------+----------+-------------------+---------------+---------------------------------------+
| libgd         | CVE-2018-14553   | HIGH     | 2.2.5-r2          | 2.2.5-r3      | gd: NULL pointer
  |
|               |                  |          |                   |               | dereference in gdImageClone
  |
|               |                  |          |                   |               | -->avd.aquasec.com/nvd/cve-2018-14553 |
+               +------------------+----------+                   +               +---------------------------------------+
|               | CVE-2019-11038   | MEDIUM   |                   |               | gd: Information disclosure
  |
|               |                  |          |                   |               | in gdImageCreateFromXbm()
  |
|               |                  |          |                   |               | -->avd.aquasec.com/nvd/cve-2019-11038 |
+---------------+------------------+----------+-------------------+---------------+---------------------------------------+
| libjpeg-turbo | CVE-2019-2201    | HIGH     | 2.0.2-r0          | 2.0.3-r0      | libjpeg-turbo: several integer        |
|               |                  |          |                   |               | overflows and subsequent
  |
|               |                  |          |                   |               | segfaults when attempting to          |
|               |                  |          |                   |               | compress/decompress gigapixel...      |
|               |                  |          |                   |               | -->avd.aquasec.com/nvd/cve-2019-2201  |
+               +------------------+          +                   +---------------+---------------------------------------+
|               | CVE-2020-13790   |          |                   | 2.0.4-r1      | libjpeg-turbo: heap-based buffer      |
|               |                  |          |                   |               | over-read in get_rgb_row() in rdppm.c |
|               |                  |          |                   |               | -->avd.aquasec.com/nvd/cve-2020-13790 |
+---------------+------------------+          +-------------------+---------------+---------------------------------------+
| libssl1.1     | CVE-2020-1967    |          | 1.1.1c-r0         | 1.1.1g-r0     | openssl: Segmentation
  |
|               |                  |          |                   |               | fault in SSL_check_chain
  |
|               |                  |          |                   |               | causes denial of service
  |
|               |                  |          |                   |               | -->avd.aquasec.com/nvd/cve-2020-1967  |
+               +------------------+----------+                   +---------------+---------------------------------------+
|               | CVE-2019-1547    | MEDIUM   |                   | 1.1.1d-r0     | openssl: side-channel weak
  |
|               |                  |          |                   |               | encryption vulnerability
  |
|               |                  |          |                   |               | -->avd.aquasec.com/nvd/cve-2019-1547  |
+               +------------------+          +                   +               +---------------------------------------+
|               | CVE-2019-1549    |          |                   |               | openssl: information
  |
|               |                  |          |                   |               | disclosure in fork()
  |
|               |                  |          |                   |               | -->avd.aquasec.com/nvd/cve-2019-1549  |
+               +------------------+          +                   +---------------+---------------------------------------+
|               | CVE-2019-1551    |          |                   | 1.1.1d-r2     | openssl: Integer overflow in RSAZ     |
|               |                  |          |                   |               | modular exponentiation on x86_64      |
|               |                  |          |                   |               | -->avd.aquasec.com/nvd/cve-2019-1551  |
+               +------------------+          +                   +---------------+---------------------------------------+
|               | CVE-2020-1971    |          |                   | 1.1.1i-r0     | openssl: EDIPARTYNAME
  |
|               |                  |          |                   |               | NULL pointer de-reference
  |
|               |                  |          |                   |               | -->avd.aquasec.com/nvd/cve-2020-1971  |
+               +------------------+----------+                   +---------------+---------------------------------------+
|               | CVE-2019-1563    | LOW      |                   | 1.1.1d-r0     | openssl: information
  |
|               |                  |          |                   |               | disclosure in PKCS7_dataDecode        |
|               |                  |          |                   |               | and CMS_decrypt_set1_pkey
  |
|               |                  |          |                   |               | -->avd.aquasec.com/nvd/cve-2019-1563  |
+---------------+------------------+----------+-------------------+---------------+---------------------------------------+
| libxml2       | CVE-2019-19956   | HIGH     | 2.9.9-r2          | 2.9.9-r3      | libxml2: memory leak in
  |
|               |                  |          |                   |               | xmlParseBalancedChunkMemoryRecover    |
|               |                  |          |                   |               | in parser.c
  |
|               |                  |          |                   |               | -->avd.aquasec.com/nvd/cve-2019-19956 |
+               +------------------+----------+                   +---------------+---------------------------------------+
|               | CVE-2020-24977   | MEDIUM   |                   | 2.9.9-r4      | libxml2: Buffer Overflow
  |
|               |                  |          |                   |               | vulnerability in
  |
|               |                  |          |                   |               | xmlEncodeEntitiesInternal
  |
|               |                  |          |                   |               | at libxml2/entities.c
  |
|               |                  |          |                   |               | -->avd.aquasec.com/nvd/cve-2020-24977 |
+---------------+------------------+----------+-------------------+---------------+---------------------------------------+
| libxslt       | CVE-2019-13117   | HIGH     | 1.1.33-r1         | 1.1.33-r3     | libxslt: an xsl number with certain   |
|               |                  |          |                   |               | format strings could lead to a...     |
|               |                  |          |                   |               | -->avd.aquasec.com/nvd/cve-2019-13117 |
+               +------------------+          +                   +               +---------------------------------------+
|               | CVE-2019-13118   |          |                   |               | libxslt: read of uninitialized        |
|               |                  |          |                   |               | stack data due to too narrow          |
|               |                  |          |                   |               | xsl:number instruction...
  |
|               |                  |          |                   |               | -->avd.aquasec.com/nvd/cve-2019-13118 |
+               +------------------+          +                   +---------------+---------------------------------------+
|               | CVE-2019-18197   |          |                   | 1.1.33-r2     | libxslt: use after free in
  |
|               |                  |          |                   |               | xsltCopyText in transform.c
  |
|               |                  |          |                   |               | could lead to information...          |
|               |                  |          |                   |               | -->avd.aquasec.com/nvd/cve-2019-18197 |
+---------------+------------------+----------+-------------------+---------------+---------------------------------------+
| musl          | CVE-2020-28928   | MEDIUM   | 1.1.22-r3         | 1.1.22-r4     | In musl libc through 1.2.1,
  |
|               |                  |          |                   |               | wcsnrtombs mishandles particular      |
|               |                  |          |                   |               | combinations of destination buffer... |
|               |                  |          |                   |               | -->avd.aquasec.com/nvd/cve-2020-28928 |
+---------------+                  +          +                   +               +
  +
| musl-utils    |                  |          |                   |               |
  |
|               |                  |          |                   |               |
  |
|               |                  |          |                   |               |
  |
|               |                  |          |                   |               |
  |
+---------------+------------------+          +-------------------+---------------+---------------------------------------+
| pcre          | CVE-2020-14155   |          | 8.43-r0           | 8.43-r1       | pcre: integer overflow in libpcre     |
|               |                  |          |                   |               | -->avd.aquasec.com/nvd/cve-2020-14155 |
+---------------+------------------+----------+-------------------+---------------+---------------------------------------+
```

#### 4.3.3 パッケージ以外の脆弱性を検査する(Dockle)

```
[root@centos8 ~]# dockle nginx:1.17.3-alpine
WARN    - CIS-DI-0001: Create a user for the container
        * Last user should not be root
INFO    - CIS-DI-0005: Enable Content trust for Docker
        * export DOCKER_CONTENT_TRUST=1 before docker pull/build
INFO    - CIS-DI-0006: Add HEALTHCHECK instruction to the container image
        * not found HEALTHCHECK statement
```

#### 4.4 改ざんされたイメージのデプロイを防ぐ
#### 4.4.1 イメージのダイジェスト(ハッシュ値)を指定する

Skip

#### 4.4.2 Docker Content Trust(Notary)で署名する

Skip

#### 4.5 プライベートレジストリを構築する(Harbor)
#### 4.5.1 Harborをインストールする

https://github.com/goharbor/harbor




