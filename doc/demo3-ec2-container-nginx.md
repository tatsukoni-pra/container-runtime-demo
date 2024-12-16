# EC2内で、コンテナ要素技術だけを用いてコンテナを起動し、nginxを動かす

※ 権限の部分で試行錯誤中

## EC2のホストOS

```bash
$ sudo cat /etc/os-release
PRETTY_NAME="Ubuntu 24.04.1 LTS"
NAME="Ubuntu"
VERSION_ID="24.04"
VERSION="24.04.1 LTS (Noble Numbat)"
VERSION_CODENAME=noble
ID=ubuntu
ID_LIKE=debian
HOME_URL="https://www.ubuntu.com/"
SUPPORT_URL="https://help.ubuntu.com/"
BUG_REPORT_URL="https://bugs.launchpad.net/ubuntu/"
PRIVACY_POLICY_URL="https://www.ubuntu.com/legal/terms-and-policies/privacy-policy"
UBUNTU_CODENAME=noble
LOGO=ubuntu-logo
```

## コンテナのルートファイルシステムを用意する

※ ルートファイルシステム自体、dockerコマンドを用いず作成することも可能だが、時間の都合上割愛している。

```bash
$ sudo snap install aws-cli --classic
$ sudo snap install docker

$ mkdir -p /tmp/container
$ aws ecr get-login-password --region ap-northeast-1 | docker login --username AWS --password-stdin 000000000000.dkr.ecr.ap-northeast-1.amazonaws.com
$ sudo docker image pull 000000000000.dkr.ecr.ap-northeast-1.amazonaws.com/my-nginx:v1
$ CID=$(sudo docker container create 000000000000.dkr.ecr.ap-northeast-1.amazonaws.com/my-nginx:v1)
$ sudo docker container export $CID | tar -x -C /tmp/container
$ sudo docker container rm $CID
$ sudo ip link delete docker0
$ ls /tmp/container
bin  dev  etc  home  lib  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  var
```

## CPU、メモリを制限するグループ(cgroup)を作成する。

Ubuntu24.04 では、cgroup v2 が使われている。

※ 事前に、cgroup-tools をインストールしておく。

```bash
# 事前に、cgroup-tools をインストールしておく。
$ sudo apt install cgroup-tools

$ mount | grep cgroup
cgroup2 on /sys/fs/cgroup type cgroup2 (rw,nosuid,nodev,noexec,relatime,nsdelegate,memory_recursiveprot)

$ sudo cgcreate \
-t root:root \
-a root:root \
-g cpu,memory:my-nginx-container-cgroup

$ ls /sys/fs/cgroup
cgroup.controllers      cgroup.stat             cpu.stat.local         dev-mqueue.mount  io.prio.class     memory.stat                proc-sys-fs-binfmt_misc.mount  system.slice
cgroup.max.depth        cgroup.subtree_control  cpuset.cpus.effective  init.scope        io.stat           memory.zswap.writeback     sys-fs-fuse-connections.mount  user.slice
cgroup.max.descendants  cgroup.threads          cpuset.cpus.isolated   io.cost.model     memory.numa_stat  misc.capacity              sys-kernel-config.mount
cgroup.pressure         cpu.pressure            cpuset.mems.effective  io.cost.qos       memory.pressure   misc.current               sys-kernel-debug.mount
cgroup.procs            cpu.stat                dev-hugepages.mount    io.pressure       memory.reclaim    my-nginx-container-cgroup  sys-kernel-tracing.mount
```

CPUを10%、メモリを10MBに制限する。

- memory.max：cgroupとその子孫のcgroupのメモリ消費が設定値を超えた場合で、減らせない場合はcgroupに対してOOM Killerが呼ばれる。デフォルト値はmax
- cpu.max：帯域幅制限を行う単位となる期間とその単位時間内での制限値。フォーマットは"（制限値⁠）⁠ （⁠期間）"（スペース区切り⁠）⁠。単位はマイクロ秒。デフォルト値は"max 100000”

```bash
# CPUを10%に制限（100000がmax）
echo "10000 100000" | sudo tee /sys/fs/cgroup/my-nginx-container-cgroup/cpu.max

# メモリを10MBに制限
echo "10485760" | sudo tee /sys/fs/cgroup/my-nginx-container-cgroup/memory.max
```

## Namespaceでカーネルリソースを隔離したコンテナを作成

```bash
# コンテナ用の Network Namespace(container_netns)を作成
$ sudo ip netns add container_netns

$ CONTAINER_ROOTF=/tmp/container
$ cgexec -g cpu,memory:my-nginx-container-cgroup \
unshare --net=/var/run/netns/container_netns -muipfr \
/bin/sh -c "capsh --caps='cap_net_bind_service,cap_chown,cap_dac_override,cap_setuid,cap_setgid,cap_fowner+eip' -- -c '\
mount -t proc proc $CONTAINER_ROOTF/proc && \
touch $CONTAINER_ROOTF$(tty) && \
mount --bind $(tty) $CONTAINER_ROOTF$(tty) && \
touch $CONTAINER_ROOTF/dev/pts/ptmx && \
mount --bind /dev/pts/ptmx $CONTAINER_ROOTF/dev/pts/ptmx && \
ln -sf /dev/pts/ptmx $CONTAINER_ROOTF/dev/ptmx && \
touch $CONTAINER_ROOTF/dev/null && \
mount --bind /dev/null $CONTAINER_ROOTF/dev/null && \
/bin/hostname my-nginx-container && \
chroot $CONTAINER_ROOTF /bin/sh'"

/ # ps aux
PID   USER     TIME  COMMAND
1 root      0:00 /bin/sh -c mount -t proc proc /tmp/container/proc && touch /tmp/container/dev/pts/1 && mount --bind /dev/pts/1 /tmp/container/dev/pts/1 && touch /tmp/container/dev/pts/ptmx && mount --bind
11 root      0:00 /bin/sh
12 root      0:00 ps aux

/ # ip link show
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
```

## コンテナ <> ホスト間通信用のネットワーク空間を設定する

```bash
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

# コンテナ内で確認
/ # ip link show
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
4: container_veth0@if5: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue state UP qlen 1000
    link/ether 4a:92:57:34:ca:64 brd ff:ff:ff:ff:ff:ff
```

## コンテナ内でNginxを起動

```bash
/ # nginx -v
nginx version: nginx/1.26.2

/ # nginx -g "daemon off;"
nginx: [alert] could not open error log file: open() "/var/lib/nginx/logs/error.log" failed (13: Permission denied)
2024/12/16 03:49:55 [emerg] 16#16: mkdir() "/var/lib/nginx/tmp/client_body" failed (13: Permission denied)
```

以下のエラーが発生するので修正中

```bash
nginx: [alert] could not open error log file: open() "/var/lib/nginx/logs/error.log" failed (13: Permission denied)
2024/12/15 02:05:38 [emerg] 11#11: mkdir() "/var/lib/nginx/tmp/client_body" failed (13: Permission denied)
```
