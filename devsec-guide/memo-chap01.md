## Docker/Kubernetes 開発・運用のためのセキュリティ実践ガイド

- 第1章 Docker/Kubernetesのおさらい
- 第2章 コンテナ運用における脅威の事例
- 第3章 ランタイムのセキュリティTips
- 第4章 イメージのセキュリティTips
- 第5章 KubernetesクラスタのセキュリティTips
- 第6章 アプリケーション間通信を守る

### 第1章 1.3 TLSの復習

・証明書の操作

cfsslとcfssljsonのインストール

```
vagrant@:~$ wget https://github.com/cloudflare/cfssl/releases/download/v1.5.0/cfssl_1.5.0_linux_amd64
vagrant@:~$ wget https://github.com/cloudflare/cfssl/releases/download/v1.5.0/cfssljson_1.5.0_linux_amd64
vagrant@:~$ mv cfssljson_1.5.0_linux_amd64 cfssljson
vagrant@:~$ mv cfssl_1.5.0_linux_amd64 cfssl
vagrant@:~$ chmod +x cfssl*
vagrant@:~$ sudo mv cfssl* /usr/local/bin/
```

CA証明書の作成

```
vagrant@:pem$ cat ca.json
{
    "CN": "my-ca.example.com",
    "key": {
        "algo": "ecdsa",
        "size": 256
    },
    "names": [
        {
            "C": "JP",
            "ST": "Kanagawa",
            "L": "Kawasaki-city",
            "O":  "my-group"
        }
    ]
}
vagrant@:pem$ cfssl genkey -initca ca.json | cfssljson -bare ca
2020/12/21 02:47:17 [INFO] generate received request
2020/12/21 02:47:17 [INFO] received CSR
2020/12/21 02:47:17 [INFO] generating key: ecdsa-256
2020/12/21 02:47:17 [INFO] encoded CSR
2020/12/21 02:47:17 [INFO] signed certificate with serial number 558792046248720381156523558473102373691656812260]]]]]
vagrant@:pem$ ls
ca-key.pem  ca.csr  ca.json  ca.pem
```

サーバ証明書の作成

証明書発行に使うコンフィグファイル

```
vagrant@:pem$ cat ca-config.json
{
    "signing":{
        "default":{
            "expiry":"8760h"
        },
        "profiles":{
            "server":{
                "expiry":"8760h",
                "usages":[
                    "signing",
                    "key encipherment",
                    "server auth"
                ]
            },
            "client":{
                "expiry":"8760h",
                "usages":[
                    "signing",
                    "key encipherment",
                    "server auth"
                ]
            },
            "client-server":{
                "expiry":"8760h",
                "usages":[
                    "signing",
                    "key encipherment",
                    "server auth",
                    "client auth"
                ]
            }
        }
    }
}
```

サーバ証明書のメタデータ

```
vagrant@:pem$ cat server.json
{
  "CN": "my-server",
  "hosts": [
    "127.0.0.1",
    "192.168.0.1",
    "my-server.example.com"
  ],
  "key": {
    "algo": "ecdsa",
    "size": 256
  },
  "names": [
    {
      "C": "JP",
      "ST": "Kanagawa",
      "L": "Kawasaki-city",
      "O": "my-group"
    }
  ]
}
```

サーバ証明書の発行

```
vagrant@:pem$ cfssl gencert -ca ca.pem -ca-key ca-key.pem -config ca-config.json -profile server server.json | cfssljson -bare server
2020/12/21 03:34:35 [INFO] generate received request
2020/12/21 03:34:35 [INFO] received CSR
2020/12/21 03:34:35 [INFO] generating key: ecdsa-256
2020/12/21 03:34:35 [INFO] encoded CSR
2020/12/21 03:34:35 [INFO] signed certificate with serial number 661800445712760233080772937934852566472848262973
vagrant@:pem$ ls -1 | grep server
server-key.pem
server.csr
server.json
server.pem
```

クライアント証明書のメタデータと発行

```
vagrant@:pem$ cat client.json
{
  "CN": "my-client",
  "hosts": [
    "my-client.example.com"
  ],
  "key": {
    "algo": "ecdsa",
    "size": 256
  },
  "names": [
    {
      "C": "JP",
      "ST": "Kanagawa",
      "L": "Kawasaki-city",
      "O": "my-group"
    }
  ]
}
vagrant@:pem$ cfssl gencert -ca ca.pem -ca-key ca-key.pem -config ca-config.json -profile client client.json | cfssljson -bare client
2020/12/21 03:40:53 [INFO] generate received request
2020/12/21 03:40:53 [INFO] received CSR
2020/12/21 03:40:53 [INFO] generating key: ecdsa-256
2020/12/21 03:40:53 [INFO] encoded CSR
2020/12/21 03:40:53 [INFO] signed certificate with serial number 254539650614603788110103808899064585478000621842
vagrant@:pem$ ls -1 | grep client
client-key.pem
client.csr
client.json
client.pem
```

mkcertで証明書を作成

mkcertインストール

```
vagrant@:~$ wget https://github.com/FiloSottile/mkcert/releases/download/v1.4.3/mkcert-v1.4.3-linux-amd64
vagrant@:~$ mv mkcert-v1.4.3-linux-amd64 mkcert
vagrant@:~$ chmod +x mkcert
vagrant@:~$ sudo mv mkcert /usr/local/bin
```

証明書作成

```
vagrant@:mkcert$ CAROOT=. mkcert my-server.example.com
Created a new local CA 💥
Note: the local CA is not installed in the system trust store.
Run "mkcert -install" for certificates to be trusted automatically ⚠️

Created a new certificate valid for the following names 📜
 - "my-server.example.com"

The certificate is at "./my-server.example.com.pem" and the key at "./my-server.example.com-key.pem" ✅

It will expire on 22 March 2023 🗓

vagrant@:mkcert$ CAROOT=. mkcert -client client
Note: the local CA is not installed in the system trust store.
Run "mkcert -install" for certificates to be trusted automatically ⚠️

Created a new certificate valid for the following names 📜
 - "client"

The certificate is at "./client-client.pem" and the key at "./client-client-key.pem" ✅

It will expire on 22 March 2023 🗓

vagrant@:mkcert$ ls
client-client-key.pem  my-server.example.com-key.pem  rootCA-key.pem
client-client.pem      my-server.example.com.pem      rootCA.pem
```

mkcertで作成したのをk8sにデプロイして確認

```
vagrant@:mkcert$ k create configmap ca --from-file=rootCA.pem
configmap/ca created
vagrant@:mkcert$ kubectl create secret tls nginx-tls --cert=my-server.example.com.pem --key=my-server.example.com-key.pe
m
secret/nginx-tls created
vagrant@:mkcert$ cd ../
vagrant@:yaml$ kubectl create configmap nginx --from-file=nginx.conf
configmap/nginx created
vagrant@:yaml$ kubectl apply -f nginx.yaml
pod/nginx created
vagrant@:yaml$ kubectl port-forward pod/nginx 8443:443 &
[1] 135646
vagrant@:yaml$ kubectl get pod
NAME    READY   STATUS    RESTARTS   AGE
nginx   1/1     Running   0          45
```

curlで確認

```
vagrant@:mkcert$ curl --head --cacert ./rootCA.pem --cert ./client-client.pem --key ./client-client-key.pem https://my-s
erver.example.com:8443/
Handling connection for 8443
HTTP/1.1 200 OK
Server: nginx/1.16.1
Date: Tue, 22 Dec 2020 03:24:04 GMT
Content-Type: text/html
Content-Length: 612
Last-Modified: Tue, 13 Aug 2019 10:05:00 GMT
Connection: keep-alive
ETag: "5d528b4c-264"
Accept-Ranges: bytes
```



