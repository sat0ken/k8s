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


```
[root@centos8 harbor]# ./install.sh --with-clair --with-notary --with-chartmuseum

[Step 0]: checking if docker is installed ...

Note: docker version: 20.10.2

[Step 1]: checking docker-compose is installed ...

Note: docker-compose version: 1.27.4

[Step 2]: loading Harbor images ...
72021dc640d8: Loading layer [==================================================>]  34.51MB/34.51MB
7910849b33b6: Loading layer [==================================================>]  6.779MB/6.779MB
06f8c409309f: Loading layer [==================================================>]  8.993MB/8.993MB
1184d84d88ef: Loading layer [==================================================>]  173.6kB/173.6kB
cdd9ba89482e: Loading layer [==================================================>]  152.6kB/152.6kB
26740b4a9b89: Loading layer [==================================================>]  66.56kB/66.56kB
56da0269ee28: Loading layer [==================================================>]  17.41kB/17.41kB
9788bddc3377: Loading layer [==================================================>]  15.36kB/15.36kB
Loaded image: goharbor/harbor-portal:v2.1.3
bd0187afcaea: Loading layer [==================================================>]  8.071MB/8.071MB
a560372832d6: Loading layer [==================================================>]  3.584kB/3.584kB
9b614956de45: Loading layer [==================================================>]   2.56kB/2.56kB
f90e06d4a8f0: Loading layer [==================================================>]  63.96MB/63.96MB
be27aa2f3572: Loading layer [==================================================>]  64.78MB/64.78MB
Loaded image: goharbor/harbor-jobservice:v2.1.3
8faeb4338473: Loading layer [==================================================>]  35.94MB/35.94MB
f5d31a9d7d81: Loading layer [==================================================>]  3.072kB/3.072kB
24b9420d082c: Loading layer [==================================================>]   59.9kB/59.9kB
5b2a87b87d06: Loading layer [==================================================>]  61.95kB/61.95kB
Loaded image: goharbor/redis-photon:v2.1.3
5283593b1862: Loading layer [==================================================>]  4.926MB/4.926MB
1f3837f98b68: Loading layer [==================================================>]  6.343MB/6.343MB
d275be601d07: Loading layer [==================================================>]  15.84MB/15.84MB
a28b9cd51ad8: Loading layer [==================================================>]  27.97MB/27.97MB
6ca0eed54c12: Loading layer [==================================================>]  22.02kB/22.02kB
27bf34c977d0: Loading layer [==================================================>]  15.84MB/15.84MB
Loaded image: goharbor/notary-server-photon:v2.1.3
dc5bc7aff8ca: Loading layer [==================================================>]  111.8MB/111.8MB
dc8328e19685: Loading layer [==================================================>]  12.59MB/12.59MB
d041752aa703: Loading layer [==================================================>]  3.072kB/3.072kB
e337675f6c90: Loading layer [==================================================>]  49.15kB/49.15kB
f5bc237280d3: Loading layer [==================================================>]  4.096kB/4.096kB
e755dfec8710: Loading layer [==================================================>]  13.46MB/13.46MB
Loaded image: goharbor/clair-photon:v2.1.3
a7a4db38256b: Loading layer [==================================================>]  6.237MB/6.237MB
87151122a1e7: Loading layer [==================================================>]  4.096kB/4.096kB
9423c6ccfa58: Loading layer [==================================================>]  3.072kB/3.072kB
5b33926a12b1: Loading layer [==================================================>]  23.51MB/23.51MB
8314fcee5545: Loading layer [==================================================>]  9.432MB/9.432MB
662ea98e6ca6: Loading layer [==================================================>]  33.76MB/33.76MB
Loaded image: goharbor/trivy-adapter-photon:v2.1.3
7a35644f5612: Loading layer [==================================================>]  77.47MB/77.47MB
aea5e7931b4f: Loading layer [==================================================>]  56.14MB/56.14MB
85011a40c597: Loading layer [==================================================>]   2.56kB/2.56kB
9492557f0459: Loading layer [==================================================>]  1.536kB/1.536kB
7166c618fc33: Loading layer [==================================================>]  18.43kB/18.43kB
db498ea36dc3: Loading layer [==================================================>]  4.044MB/4.044MB
fe783ba3ed96: Loading layer [==================================================>]  266.2kB/266.2kB
Loaded image: goharbor/prepare:v2.1.3
040bb6ac6bbd: Loading layer [==================================================>]  4.933MB/4.933MB
b205abac2507: Loading layer [==================================================>]  4.096kB/4.096kB
3cc94eef4661: Loading layer [==================================================>]  20.51MB/20.51MB
a20b7ec432b7: Loading layer [==================================================>]  3.072kB/3.072kB
8c93ddeb06b0: Loading layer [==================================================>]  25.91MB/25.91MB
5b820ad21060: Loading layer [==================================================>]  47.24MB/47.24MB
Loaded image: goharbor/harbor-registryctl:v2.1.3
6cfbab833c05: Loading layer [==================================================>]  74.89MB/74.89MB
5d9dae6b4be3: Loading layer [==================================================>]  3.584kB/3.584kB
274111b2154b: Loading layer [==================================================>]  3.072kB/3.072kB
24aa1969673b: Loading layer [==================================================>]   2.56kB/2.56kB
13ca9a511d7e: Loading layer [==================================================>]  3.072kB/3.072kB
12340e94a4b6: Loading layer [==================================================>]  3.584kB/3.584kB
ed1d3427850e: Loading layer [==================================================>]  12.29kB/12.29kB
f403d83ebf58: Loading layer [==================================================>]  3.584kB/3.584kB
Loaded image: goharbor/harbor-log:v2.1.3
ac275526e8b7: Loading layer [==================================================>]  63.76MB/63.76MB
7cc25b455629: Loading layer [==================================================>]  78.31MB/78.31MB
fec4c3ccd0b1: Loading layer [==================================================>]  6.144kB/6.144kB
7f202bd62270: Loading layer [==================================================>]   2.56kB/2.56kB
a468a2ead8d0: Loading layer [==================================================>]   2.56kB/2.56kB
88ab2997d705: Loading layer [==================================================>]   2.56kB/2.56kB
119f7fb77e7c: Loading layer [==================================================>]   2.56kB/2.56kB
e0eb7950afe7: Loading layer [==================================================>]  11.26kB/11.26kB
Loaded image: goharbor/harbor-db:v2.1.3
aad26a60848a: Loading layer [==================================================>]  6.779MB/6.779MB
Loaded image: goharbor/nginx-photon:v2.1.3
4146d3da77a1: Loading layer [==================================================>]  4.932MB/4.932MB
d47cdb202be9: Loading layer [==================================================>]  62.71MB/62.71MB
21eba4461bc1: Loading layer [==================================================>]  3.072kB/3.072kB
487faec285d2: Loading layer [==================================================>]  4.096kB/4.096kB
24a42bcfcda4: Loading layer [==================================================>]  63.53MB/63.53MB
Loaded image: goharbor/chartmuseum-photon:v2.1.3
3928a230d44f: Loading layer [==================================================>]  8.072MB/8.072MB
5eb1dffa1144: Loading layer [==================================================>]  3.584kB/3.584kB
a6aaade6f42d: Loading layer [==================================================>]   2.56kB/2.56kB
b1d9936cf892: Loading layer [==================================================>]  54.28MB/54.28MB
a0eae0d31be7: Loading layer [==================================================>]  5.632kB/5.632kB
2580d019fed2: Loading layer [==================================================>]  60.42kB/60.42kB
113646e55716: Loading layer [==================================================>]  11.78kB/11.78kB
b0c71cec7d35: Loading layer [==================================================>]   55.1MB/55.1MB
3f622fe858eb: Loading layer [==================================================>]   2.56kB/2.56kB
Loaded image: goharbor/harbor-core:v2.1.3
046f35736958: Loading layer [==================================================>]  4.933MB/4.933MB
bfc4a424487c: Loading layer [==================================================>]  4.096kB/4.096kB
adea95e62ff3: Loading layer [==================================================>]  3.072kB/3.072kB
a8950efbcaf0: Loading layer [==================================================>]  20.51MB/20.51MB
7dff05d49fa5: Loading layer [==================================================>]  21.33MB/21.33MB
Loaded image: goharbor/registry-photon:v2.1.3
e9222690ad3e: Loading layer [==================================================>]  4.926MB/4.926MB
c194ec78c6d4: Loading layer [==================================================>]  6.343MB/6.343MB
11ed9d4d704d: Loading layer [==================================================>]  14.43MB/14.43MB
be10dcc39732: Loading layer [==================================================>]  27.97MB/27.97MB
b3d46f5af780: Loading layer [==================================================>]  22.02kB/22.02kB
de875ccaa613: Loading layer [==================================================>]  14.43MB/14.43MB
Loaded image: goharbor/notary-signer-photon:v2.1.3
f60395b31a75: Loading layer [==================================================>]  4.933MB/4.933MB
ccc8622e314b: Loading layer [==================================================>]  4.096kB/4.096kB
333864843b25: Loading layer [==================================================>]  3.072kB/3.072kB
bd374121ff18: Loading layer [==================================================>]  9.427MB/9.427MB
1135c12b800b: Loading layer [==================================================>]  10.25MB/10.25MB
Loaded image: goharbor/clair-adapter-photon:v2.1.3


[Step 3]: preparing environment ...

[Step 4]: preparing harbor configs ...
prepare base dir is set to /root/harbor
Generated configuration file: /config/portal/nginx.conf
Generated configuration file: /config/log/logrotate.conf
Generated configuration file: /config/log/rsyslog_docker.conf
Generated configuration file: /config/nginx/nginx.conf
Generated configuration file: /config/core/env
Generated configuration file: /config/core/app.conf
Generated configuration file: /config/registry/config.yml
Generated configuration file: /config/registryctl/env
Generated configuration file: /config/registryctl/config.yml
Generated configuration file: /config/db/env
Generated configuration file: /config/jobservice/env
Generated configuration file: /config/jobservice/config.yml
Generated and saved secret to file: /data/secret/keys/secretkey
Successfully called func: create_root_cert
Successfully called func: create_root_cert
Successfully called func: create_cert
Copying certs for notary signer
Copying nginx configuration file for notary
Generated configuration file: /config/nginx/conf.d/notary.upstream.conf
Generated configuration file: /config/nginx/conf.d/notary.server.conf
Generated configuration file: /config/notary/server-config.postgres.json
Generated configuration file: /config/notary/server_env
Generated and saved secret to file: /data/secret/keys/defaultalias
Generated configuration file: /config/notary/signer_env
Generated configuration file: /config/notary/signer-config.postgres.json
Generated configuration file: /config/clair/postgres_env
Generated configuration file: /config/clair/config.yaml
Generated configuration file: /config/clair/clair_env
Generated configuration file: /config/clair-adapter/env
Generated configuration file: /config/chartserver/env
Generated configuration file: /compose_location/docker-compose.yml
Clean up the input dir



[Step 5]: starting Harbor ...
Creating network "harbor_harbor" with the default driver
Creating network "harbor_harbor-clair" with the default driver
Creating network "harbor_harbor-notary" with the default driver
Creating network "harbor_harbor-chartmuseum" with the default driver
Creating network "harbor_notary-sig" with the default driver
Creating harbor-log ... done
Creating registryctl   ... done
Creating redis         ... done
Creating harbor-db     ... done
Creating chartmuseum   ... done
Creating registry      ... done
Creating harbor-portal ... done
Creating notary-signer ... done
Creating harbor-core   ... done
Creating clair         ... done
Creating notary-server     ... done
Creating clair-adapter     ... done
Creating harbor-jobservice ... done
Creating nginx             ... done
✔ ----Harbor has been installed and started successfully.----
```

HarborにイメージをPushする

```
[root@centos8 ~]# docker tag nginx:1.17.3-alpine harbor.example.com/library/nginx:1.17.3-alpine
[root@centos8 ~]# docker push harbor.example.com/library/nginx:1.17.3-alpine
The push refers to repository [harbor.example.com/library/nginx]
3e76d2df1790: Pushed
03901b4a2ea8: Pushed
1.17.3-alpine: digest: sha256:1907fa667b160c40dcc7f99d884f4b12cf49e487408a869857e75e64838fc9b6 size: 739
```
