# EC2内で、runcを用いてコンテナを起動し、nginxコンテナを動かす

- https://github.com/opencontainers/runc

## runCのバージョン

```bash
$ apt install runc

$ runc -v
runc version 1.1.12-0ubuntu3.1
spec: 1.0.2-dev
go: go1.22.2
libseccomp: 2.5.5
```

## コンテナのルートファイルシステムを用意する

runCはcontainerdとは異なり、コンテナイメージの管理は行わないため、dockerを用いてコンテナイメージからファイルシステムをエクスポートする。

```bash
$ sudo snap install aws-cli --classic
$ sudo snap install docker

$ mkdir -p /tmp/runc/rootfs
$ aws ecr get-login-password --region ap-northeast-1 | docker login --username AWS --password-stdin 000000000000.dkr.ecr.ap-northeast-1.amazonaws.com
$ sudo docker image pull 000000000000.dkr.ecr.ap-northeast-1.amazonaws.com/my-nginx:v1
$ CID=$(sudo docker container create 000000000000.dkr.ecr.ap-northeast-1.amazonaws.com/my-nginx:v1)
$ sudo docker container export $CID | tar -x -C /tmp/runc/rootfs
$ sudo docker container rm $CID
$ sudo ip link delete docker0
$ ls /tmp/runc/rootfs
bin  dev  etc  home  lib  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  var
```

## runc specでコンテナランタイムが動作する際の仕様ファイルを作成

```bash
$ cd /tmp/runc/
$ runc spec
# config.json が作成される
$ ls
config.json  rootfs

$ cat config.json
{
        "ociVersion": "1.0.2-dev",
        "process": {
                "terminal": true,
                "user": {
                        "uid": 0,
                        "gid": 0
                },
                "args": [
                        "sh"
                ],
                "env": [
                        "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin",
                        "TERM=xterm"
                ],
                "cwd": "/",
                "capabilities": {
                        "bounding": [
                                "CAP_AUDIT_WRITE",
                                "CAP_KILL",
                                "CAP_NET_BIND_SERVICE"
                        ],
                        "effective": [
                                "CAP_AUDIT_WRITE",
                                "CAP_KILL",
                                "CAP_NET_BIND_SERVICE"
                        ],
                        "permitted": [
                                "CAP_AUDIT_WRITE",
                                "CAP_KILL",
                                "CAP_NET_BIND_SERVICE"
                        ],
                        "ambient": [
                                "CAP_AUDIT_WRITE",
                                "CAP_KILL",
                                "CAP_NET_BIND_SERVICE"
                        ]
                },
                "rlimits": [
                        {
                                "type": "RLIMIT_NOFILE",
                                "hard": 1024,
                                "soft": 1024
                        }
                ],
                "noNewPrivileges": true
        },
        "root": {
                "path": "rootfs",
                "readonly": true
        },
        "hostname": "runc",
        "mounts": [
                {
                        "destination": "/proc",
                        "type": "proc",
                        "source": "proc"
                },
                {
                        "destination": "/dev",
                        "type": "tmpfs",
                        "source": "tmpfs",
                        "options": [
                                "nosuid",
                                "strictatime",
                                "mode=755",
                                "size=65536k"
                        ]
                },
                {
                        "destination": "/dev/pts",
                        "type": "devpts",
                        "source": "devpts",
                        "options": [
                                "nosuid",
                                "noexec",
                                "newinstance",
                                "ptmxmode=0666",
                                "mode=0620",
                                "gid=5"
                        ]
                },
                {
                        "destination": "/dev/shm",
                        "type": "tmpfs",
                        "source": "shm",
                        "options": [
                                "nosuid",
                                "noexec",
                                "nodev",
                                "mode=1777",
                                "size=65536k"
                        ]
                },
                {
                        "destination": "/dev/mqueue",
                        "type": "mqueue",
                        "source": "mqueue",
                        "options": [
                                "nosuid",
                                "noexec",
                                "nodev"
                        ]
                },
                {
                        "destination": "/sys",
                        "type": "sysfs",
                        "source": "sysfs",
                        "options": [
                                "nosuid",
                                "noexec",
                                "nodev",
                                "ro"
                        ]
                },
                {
                        "destination": "/sys/fs/cgroup",
                        "type": "cgroup",
                        "source": "cgroup",
                        "options": [
                                "nosuid",
                                "noexec",
                                "nodev",
                                "relatime",
                                "ro"
                        ]
                }
        ],
        "linux": {
                "resources": {
                        "devices": [
                                {
                                        "allow": false,
                                        "access": "rwm"
                                }
                        ]
                },
                "namespaces": [
                        {
                                "type": "pid"
                        },
                        {
                                "type": "network"
                        },
                        {
                                "type": "ipc"
                        },
                        {
                                "type": "uts"
                        },
                        {
                                "type": "mount"
                        },
                        {
                                "type": "cgroup"
                        }
                ],
                "maskedPaths": [
                        "/proc/acpi",
                        "/proc/asound",
                        "/proc/kcore",
                        "/proc/keys",
                        "/proc/latency_stats",
                        "/proc/timer_list",
                        "/proc/timer_stats",
                        "/proc/sched_debug",
                        "/sys/firmware",
                        "/proc/scsi"
                ],
                "readonlyPaths": [
                        "/proc/bus",
                        "/proc/fs",
                        "/proc/irq",
                        "/proc/sys",
                        "/proc/sysrq-trigger"
                ]
        }
}
```

## **config.jsonを編集**

```json
{
        "ociVersion": "1.0.2-dev",
        "process": {
                "terminal": true,
                "user": {
                        "uid": 0,
                        "gid": 0
                },
                "args": [
                        "sh"
                ],
                "env": [
                        "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin",
                        "TERM=xterm"
                ],
                "cwd": "/",
                "capabilities": {
                        "bounding": [
                                "CAP_AUDIT_WRITE",
                                "CAP_KILL",
                                "CAP_NET_BIND_SERVICE",
                                "CAP_SETUID",
		                            "CAP_SETGID",
                                "CAP_CHOWN",
                                "CAP_FOWNER",
                                "CAP_DAC_OVERRIDE"
                        ],
                        "effective": [
                                "CAP_AUDIT_WRITE",
                                "CAP_KILL",
                                "CAP_NET_BIND_SERVICE",
                                "CAP_SETUID",
		                            "CAP_SETGID",
                                "CAP_CHOWN",
                                "CAP_FOWNER",
                                "CAP_DAC_OVERRIDE"
                        ],
                        "inheritable": [
                                "CAP_AUDIT_WRITE",
                                "CAP_KILL",
                                "CAP_NET_BIND_SERVICE",
		                            "CAP_SETUID",
		                            "CAP_SETGID",
                                "CAP_CHOWN",
                                "CAP_FOWNER",
                                "CAP_DAC_OVERRIDE"
                        ],
                        "permitted": [
                                "CAP_AUDIT_WRITE",
                                "CAP_KILL",
                                "CAP_NET_BIND_SERVICE",
                                "CAP_SETUID",
		                            "CAP_SETGID",
                                "CAP_CHOWN",
                                "CAP_FOWNER",
                                "CAP_DAC_OVERRIDE"
                        ],
                        "ambient": [
                                "CAP_AUDIT_WRITE",
                                "CAP_KILL",
                                "CAP_NET_BIND_SERVICE",
                                "CAP_SETUID",
		                            "CAP_SETGID",
                                "CAP_CHOWN",
                                "CAP_FOWNER",
                                "CAP_DAC_OVERRIDE"
                        ]
                },
                "rlimits": [
                        {
                                "type": "RLIMIT_NOFILE",
                                "hard": 1024,
                                "soft": 1024
                        }
                ],
                "noNewPrivileges": true
        },
        "root": {
                "path": "rootfs",
                "readonly": false
        },
        "hostname": "runc",
        "mounts": [
                {
                        "destination": "/proc",
                        "type": "proc",
                        "source": "proc"
                },
                {
                        "destination": "/dev",
                        "type": "tmpfs",
                        "source": "tmpfs",
                        "options": [
                                "nosuid",
                                "strictatime",
                                "mode=755",
                                "size=65536k"
                        ]
                },
                {
                        "destination": "/dev/pts",
                        "type": "devpts",
                        "source": "devpts",
                        "options": [
                                "nosuid",
                                "noexec",
                                "newinstance",
                                "ptmxmode=0666",
                                "mode=0620",
                                "gid=5"
                        ]
                },
                {
                        "destination": "/dev/shm",
                        "type": "tmpfs",
                        "source": "shm",
                        "options": [
                                "nosuid",
                                "noexec",
                                "nodev",
                                "mode=1777",
                                "size=65536k"
                        ]
                },
                {
                        "destination": "/dev/mqueue",
                        "type": "mqueue",
                        "source": "mqueue",
                        "options": [
                                "nosuid",
                                "noexec",
                                "nodev"
                        ]
                },
                {
                        "destination": "/sys",
                        "type": "sysfs",
                        "source": "sysfs",
                        "options": [
                                "nosuid",
                                "noexec",
                                "nodev",
                                "ro"
                        ]
                },
                {
                        "destination": "/sys/fs/cgroup",
                        "type": "cgroup",
                        "source": "cgroup",
                        "options": [
                                "nosuid",
                                "noexec",
                                "nodev",
                                "relatime",
                                "ro"
                        ]
                }
        ],
        "linux": {
                "resources": {
                        "devices": [
                                {
                                        "allow": false,
                                        "access": "rwm"
                                }
                        ]
                },
                "namespaces": [
                        {
                                "type": "pid"
                        },
                        {
                                "type": "network"
                        },
                        {
                                "type": "ipc"
                        },
                        {
                                "type": "uts"
                        },
                        {
                                "type": "mount"
                        },
                        {
                                "type": "cgroup"
                        }
                ],
                "maskedPaths": [
                        "/proc/acpi",
                        "/proc/asound",
                        "/proc/kcore",
                        "/proc/keys",
                        "/proc/latency_stats",
                        "/proc/timer_list",
                        "/proc/timer_stats",
                        "/proc/sched_debug",
                        "/sys/firmware",
                        "/proc/scsi"
                ],
                "readonlyPaths": [
                        "/proc/bus",
                        "/proc/fs",
                        "/proc/irq",
                        "/proc/sys",
                        "/proc/sysrq-trigger"
                ]
        }
}
```

## コンテナを起動

```bash
$ runc run my-nginx-container
/ # ls
bin    dev    etc    home   lib    media  mnt    opt    proc   root   run    sbin   srv    sys    tmp    usr    var

/ # ps aux
PID   USER     TIME  COMMAND
1 root      0:00 sh
7 root      0:00 ps aux

/ # ip link show
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
```

## コンテナ <> ホスト間のネットワークを設定する

```bash
# コンテナのPIDを確認
$ sudo runc list
ID          PID         STATUS      BUNDLE      CREATED                          OWNER
my-nginx    1908        running     /tmp/runc   2024-12-15T04:38:26.103284752Z   root

# コンテナのnetnsを、/var/run/netns以下にシンボリックを貼る.
$ PID=1908
$ sudo ln -s /proc/$PID/ns/net /var/run/netns/container_netns
$ ip netns show
container_netns

# vethペアを作る
sudo ip link add name veth0 type veth peer name container_veth0

# container_veth0をcontainer_netnsにアタッチ
sudo ip link set dev container_veth0 netns container_netns

# IPアドレスを付与
sudo ip netns exec container_netns ip addr add 192.168.10.1/24 dev container_veth0
sudo ip addr add 192.168.10.2/24 dev veth0

# リンクup
sudo ip netns exec container_netns ip link set dev container_veth0 up  
sudo ip link set dev veth0 up

# コンテナ内から確認
/ # ip link show
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
4: container_veth0@if5: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue state UP qlen 1000
    link/ether 52:9e:d3:97:29:7b brd ff:ff:ff:ff:ff:ff
```

## コンテナ内でNginxを起動

```bash
/ # nginx -v
nginx version: nginx/1.26.2

/ # nginx -g "daemon off;"
```

この時点で、ホスト側からコンテナNginxへのアクセスが可能になる。

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
