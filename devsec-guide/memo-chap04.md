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
