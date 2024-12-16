# EC2内で、containerdを用いてコンテナを起動し、nginxを動かす

## インストール

```bash
$ sudo apt update && apt install containerd
$ containerd -v
containerd github.com/containerd/containerd 1.7.12

# containerdは、ctrコマンドで操作する
$ ctr --help
NAME:
   ctr -
        __
  _____/ /______
 / ___/ __/ ___/
/ /__/ /_/ /
\___/\__/_/

containerd CLI

USAGE:
   ctr [global options] command [command options] [arguments...]

VERSION:
   1.7.12

DESCRIPTION:

ctr is an unsupported debug and administrative client for interacting
with the containerd daemon. Because it is unsupported, the commands,
options, and operations are not guaranteed to be backward compatible or
stable from release to release of the containerd project.

COMMANDS:
   plugins, plugin            Provides information about containerd plugins
   version                    Print the client and server versions
   containers, c, container   Manage containers
   content                    Manage content
   events, event              Display containerd events
   images, image, i           Manage images
   leases                     Manage leases
   namespaces, namespace, ns  Manage namespaces
   pprof                      Provide golang pprof outputs for containerd
   run                        Run a container
   snapshots, snapshot        Manage snapshots
   tasks, t, task             Manage tasks
   install                    Install a new package
   oci                        OCI tools
   sandboxes, sandbox, sb, s  Manage sandboxes
   info                       Print the server info
   deprecations
   shim                       Interact with a shim directly
   help, h                    Shows a list of commands or help for one command

GLOBAL OPTIONS:
   --debug                      Enable debug output in logs
   --address value, -a value    Address for containerd's GRPC server (default: "/run/containerd/containerd.sock") [$CONTAINERD_ADDRESS]
   --timeout value              Total timeout for ctr commands (default: 0s)
   --connect-timeout value      Timeout for connecting to containerd (default: 0s)
   --namespace value, -n value  Namespace to use with commands (default: "default") [$CONTAINERD_NAMESPACE]
   --help, -h                   show help
   --version, -v                print the version

# runc もインストールされる
$ runc -v
runc version 1.1.12-0ubuntu3.1
spec: 1.0.2-dev
go: go1.22.2
libseccomp: 2.5.5
```

## containerdを用いて、コンテナイメージをpullする

```bash
# AWS CLIをインストール
$ sudo snap install aws-cli --classic

# ECR Private Registoryからコンテナイメージをpullする
$ ECR_REGION=ap-northeast-1
$ ECR_PASSWORD=$(aws ecr get-login-password --region $ECR_REGION)
$ ctr image pull -u "AWS:$ECR_PASSWORD" 000000000000.dkr.ecr.ap-northeast-1.amazonaws.com/my-nginx:v1
000000000000.dkr.ecr.ap-northeast-1.amazonaws.com/my-nginx:v1:                    resolved       |++++++++++++++++++++++++++++++++++++++|
manifest-sha256:89f5f19823c44c94a972876a25728f3efa7847aac9a49c3b673916d39d625282: done           |++++++++++++++++++++++++++++++++++++++|
layer-sha256:38a8310d387e375e0ec6fabe047a9149e8eb214073db9f461fee6251fd936a75:    done           |++++++++++++++++++++++++++++++++++++++|
layer-sha256:466df74973c0ed2b990977b68ed994551b803f5b80369297417b71b99f8e27b5:    done           |++++++++++++++++++++++++++++++++++++++|
config-sha256:3b2b43c88cf875a2d732ef6ff3e1d1735bee88da302cf3243404ed3ec95c7cd1:   done           |++++++++++++++++++++++++++++++++++++++|
layer-sha256:b1104e0b2c96bd3f0ca4e78bfeb5e45ae6e9513265c1aaf866b16e7d190b5129:    done           |++++++++++++++++++++++++++++++++++++++|
layer-sha256:90ab1777dfd9e29f4991820bca4613bedc3b9baf48913ebbe4c34d649d39f85e:    done           |++++++++++++++++++++++++++++++++++++++|
elapsed: 0.7 s                                                                    total:  6.0 Mi (8.6 MiB/s)
unpacking linux/amd64 sha256:89f5f19823c44c94a972876a25728f3efa7847aac9a49c3b673916d39d625282...
done: 487.330622ms

$ ctr images list
REF                                                           TYPE                                                 DIGEST                                                                  SIZE    PLATFORMS   LABELS
000000000000.dkr.ecr.ap-northeast-1.amazonaws.com/my-nginx:v1 application/vnd.docker.distribution.manifest.v2+json sha256:89f5f19823c44c94a972876a25728f3efa7847aac9a49c3b673916d39d625282 6.9 MiB linux/amd64 -
```

## コンテナ <> ホスト間通信用のネットワーク空間を設定する

```bash
# コンテナ用の Network Namespace(container_netns)を作成
$ sudo ip netns add container_netns

# vethペアを作る
$ sudo ip link add name veth0 type veth peer name container_veth0

# container_veth0をcontainer_netnsにアタッチ
$ sudo ip link set dev container_veth0 netns container_netns

# IPアドレスを付与
$ sudo ip netns exec container_netns ip addr add 192.168.10.1/24 dev container_veth0
$ sudo ip addr add 192.168.10.2/24 dev veth0

# リンクup
$ sudo ip netns exec container_netns ip link set dev container_veth0 up  
$ sudo ip link set dev veth0 up
```

## コンテナを動かす

`/run/nginx` を作らないといけないらしい

```bash
$ ctr run \
  --rm \
  -t \
  --snapshotter=native \
  --with-ns network:/var/run/netns/container_netns \
  000000000000.dkr.ecr.ap-northeast-1.amazonaws.com/my-nginx:v1 \
  my-nginx-container-setup \
  /bin/sh -c 'mkdir -p /run/nginx && nginx -g "daemon off;"'
```

```bash
$ ps aux
USER         PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND

root        2132  0.0  1.5 1773912 29824 pts/1   Sl+  03:01   0:00 ctr run --rm -t --snapshotter=native 000000000000.dkr.ecr.ap-northeast-1.amazonaws.com/my-nginx:v1 my-nginx-container-setup /bin/sh -c mkdir -p
root        2148  0.0  0.7 1237844 13876 ?       Sl   03:01   0:00 /usr/bin/containerd-shim-runc-v2 -namespace default -id my-nginx-container-setup -address /run/containerd/containerd.sock
root        2169  0.0  0.2   9980  4524 pts/0    Ss+  03:01   0:00 nginx: master process nginx -g daemon off;
message+    2182  0.0  0.1  10436  1980 pts/0    S+   03:01   0:00 nginx: worker process
message+    2183  0.0  0.1  10436  2108 pts/0    S+   03:01   0:00 nginx: worker process
```

ホスト → コンテナ 方向への疎通が可能になる。

```bash
$ curl 192.168.10.1:80
<h1>Hello World!</h1>
<p>This is Nginx Demo System.</p>
```

## インターネットから、コンテナにアクセス可能にする

```bash
$ sudo apt install net-tools

# ip_forwardを有効にする
$ echo 1 > /proc/sys/net/ipv4/ip_forward

# FORWARDチェインのデフォルトポリシーを変更
$ sudo iptables -P FORWARD ACCEPT

# コンテナ内で、デフォルトゲートウェイを設定する
# 192.168.10.1/24サブネット以外の宛先IPの場合、192.168.10.2へ転送するように
$ sudo ip netns exec container_netns ip route add default via 192.168.10.2
$ ip netns exec container_netns netstat -rn
Kernel IP routing table
Destination     Gateway         Genmask         Flags   MSS Window  irtt Iface
0.0.0.0         192.168.10.2    0.0.0.0         UG        0 0          0 container_veth0
192.168.10.0    0.0.0.0         255.255.255.0   U         0 0          0 container_veth0

# 192.168.10.0/24サブネットのパケットの送信元IPを、外部ネットワークに繋がっているens5のIPへと書き換える
$ sudo iptables -t nat -A POSTROUTING \
  -s 192.168.10.0/24 \
  -o ens5 \
  -j MASQUERADE

# PREROUTINGルール（外部からのアクセスをコンテナに転送）
sudo iptables -t nat -A PREROUTING \
  -p tcp \
  --dport 8080 \
  -j DNAT \
  --to-destination 192.168.10.1:80
```

インターネットから、8080ポートでコンテナにアクセス可能になる。
