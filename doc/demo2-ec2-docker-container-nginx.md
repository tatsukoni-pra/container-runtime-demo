# EC2内で、dockerコマンドを用いてコンテナを起動し、nginxを動かす

## Dockerfile

https://github.com/tatsukoni-pra/docker-nginx/tree/main

## コンテナイメージをECRにPush

```bash
$ aws ecr get-login-password --region ap-northeast-1 --profile hogehoge | docker login --username AWS --password-stdin 000000000000.dkr.ecr.ap-northeast-1.amazonaws.com
Login Succeeded

$ docker image build --platform=linux/amd64 -t my-nginx:v1 .

$ docker image ls -a
REPOSITORY   TAG       IMAGE ID       CREATED         SIZE
my-nginx     v1        3ab258c23c41   4 minutes ago   13.2MB

$ docker tag my-nginx:v1 000000000000.dkr.ecr.ap-northeast-1.amazonaws.com/my-nginx:v1

$ docker image ls -a
REPOSITORY                                                   TAG       IMAGE ID       CREATED         SIZE
000000000000.dkr.ecr.ap-northeast-1.amazonaws.com/my-nginx   v1        3ab258c23c41   5 minutes ago   13.2MB
my-nginx                                                     v1        3ab258c23c41   5 minutes ago   13.2MB

$ docker push 000000000000.dkr.ecr.ap-northeast-1.amazonaws.com/my-nginx:v1
```

## EC2インスタンス内で、コンテナイメージを動かす

※ EC2インスタンスに、`AmazonEC2ContainerRegistryReadOnly` ポリシーのアタッチが必要。

※ EC2インスタンスのセキュリティグループで、ポート番号8080からのアクセスを許可しておくことが必要。

まず、AWS CLI・dockerをインストール

```bash
$ sudo snap install aws-cli --classic
aws-cli (v2/stable) 2.22.17 from Amazon Web Services (aws✓) installed
$ aws --version
aws-cli/2.22.17 Python/3.12.6 Linux/6.8.0-1018-aws exe/x86_64.ubuntu.24

$ sudo snap install docker
$ docker --version
Docker version 27.2.0, build 3ab4256
```

ECRからコンテナイメージをpull

```bash
$ sudo su

$ aws ecr get-login-password --region ap-northeast-1 | docker login --username AWS --password-stdin 000000000000.dkr.ecr.ap-northeast-1.amazonaws.com
Login Succeeded

$ sudo docker image pull 000000000000.dkr.ecr.ap-northeast-1.amazonaws.com/my-nginx:v1
v1: Pulling from my-nginx
38a8310d387e: Pull complete
b1104e0b2c96: Pull complete
90ab1777dfd9: Pull complete
466df74973c0: Pull complete
Digest: sha256:89f5f19823c44c94a972876a25728f3efa7847aac9a49c3b673916d39d625282
Status: Downloaded newer image for 000000000000.dkr.ecr.ap-northeast-1.amazonaws.com/my-nginx:v1
000000000000.dkr.ecr.ap-northeast-1.amazonaws.com/my-nginx:v1

$ sudo docker image ls -a
REPOSITORY                                                   TAG       IMAGE ID       CREATED              SIZE
000000000000.dkr.ecr.ap-northeast-1.amazonaws.com/my-nginx   v1        3b2b43c88cf8   About a minute ago   12.6MB
```

## コンテナを起動

```bash
$ sudo docker container run -it \
--name my-nginx-container \
-p 8080:80 \
000000000000.dkr.ecr.ap-northeast-1.amazonaws.com/my-nginx:v1
```

## ホスト側からプロセス確認

ホスト側からは、プロセスを確認することができる。

```bash
$ sudo ps aux

root        1491  0.1  5.0 2358124 98560 ?       Ssl  06:21   0:08 dockerd --group docker --exec-root=/run/snap.docker --data-root=/var/snap/docker/common/var-lib-docker --pidfile=/run/snap.docker/docker.pid --c
root        1603  0.1  2.4 1801076 48136 ?       Ssl  06:21   0:06 containerd --config /run/snap.docker/containerd/containerd.toml --log-level error
root        5895  0.0  0.3  17132  6912 pts/2    S+   08:01   0:00 sudo docker container run -it --name my-nginx-container -p 8080:80 000000000000.dkr.ecr.ap-northeast-1.amazonaws.com/my-nginx:v1
root        5896  0.0  0.1  17132  2484 pts/0    Ss   08:01   0:00 sudo docker container run -it --name my-nginx-container -p 8080:80 000000000000.dkr.ecr.ap-northeast-1.amazonaws.com/my-nginx:v1
root        5897  0.1  1.2 1769184 24064 pts/0   Sl+  08:01   0:00 /snap/docker/2963/bin/docker container run -it --name my-nginx-container -p 8080:80 000000000000.dkr.ecr.ap-northeast-1.amazonaws.com/my-nginx:v
root        5922  0.0  0.1 1671880 3584 ?        Sl   08:01   0:00 /snap/docker/2963/bin/docker-proxy -proto tcp -host-ip 0.0.0.0 -host-port 8080 -container-ip 172.17.0.2 -container-port 80
root        5931  0.0  0.1 1745868 3584 ?        Sl   08:01   0:00 /snap/docker/2963/bin/docker-proxy -proto tcp -host-ip :: -host-port 8080 -container-ip 172.17.0.2 -container-port 80
root        5968  0.0  0.6 1238720 13540 ?       Sl   08:01   0:00 /snap/docker/2963/bin/containerd-shim-runc-v2 -namespace moby -id e13f778356e61f4e2f33adc9fd6d6e7ea6275f1e32bbf5dd812f9dffdd3f148d -address /run
root        5988  0.1  0.2   9980  4792 pts/0    Ss+  08:01   0:00 nginx: master process nginx -g daemon off;
message+    6012  0.0  0.1  10436  2224 pts/0    S+   08:01   0:00 nginx: worker process
message+    6013  0.0  0.1  10436  2096 pts/0    S+   08:01   0:00 nginx: worker process
```

## コンテナ内からプロセス確認

ホスト側のプロセスは隠蔽されている

```bash
$ sudo docker container exec -it e13f778356e6 ps aux
PID   USER     TIME  COMMAND
1 root      0:00 nginx: master process nginx -g daemon off;
7 nginx     0:00 nginx: worker process
8 nginx     0:00 nginx: worker process
```
