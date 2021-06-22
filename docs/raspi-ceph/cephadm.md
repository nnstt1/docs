# Ceph クラスタ構築

[Cephadm](https://docs.ceph.com/en/pacific/cephadm/){target=_blank} を使って Ceph クラスタを構築していきます。

## クラスタ構築

### パッケージインストール

Ceph クラスタ構築に必要なパッケージをインストールします。

```bash
$ yum install -y python3 gdisk
$ sudo yum remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-engine
$ yum install -y yum-utils
$ yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo
$ yum install -y docker-ce docker-ce-cli containerd.io
$ systemctl start docker
$ docker run hello-world
$ systemctl enable docker
```

### Cephadm インストール

GitHub から cephadm をインストールします。

```bash
$ curl --silent --remote-name --location https://github.com/ceph/ceph/raw/octopus/src/cephadm/cephadm
$ curl --silent --remote-name --location https://github.com/ceph/ceph/raw/pacific/src/cephadm/cephadm
$ chmod +x cephadm
$ ./cephadm add-repo --release octopus
Writing repo to /etc/yum.repos.d/ceph.repo...
Enabling EPEL...

# エラー出た場合は再実行すればいける
$ ./cephadm install

# ceph cli
$ cephadm install ceph-common
```

### Bootstrap

`cephadm bootstrap` コマンドで Ceph クラスタを構築します。

```bash
$ cephadm bootstrap --mon-ip 192.168.2.11 --skip-monitoring-stack
```

- `--skip-monitoring-stack`

    Prometheus などの監視用モジュールを追加しない

### ホスト追加

構築した Ceph クラスタにノードを追加します。

```bash
# 現状の確認
$ ceph orch host ls

# Ceph 公開鍵の登録
$ ssh-copy-id -f -i /etc/ceph/ceph.pub root@ceph2
$ ssh-copy-id -f -i /etc/ceph/ceph.pub root@ceph3

# ホスト追加
$ ceph orch host add ceph2
$ ceph orch host add ceph3

# 追加後の確認
$ ceph orch host ls
```

### OSD 登録

OSD を追加します。

```bash
# デバイスの状態を確認
$ ceph orch device ls --refresh
Hostname  Path      Type  Serial         Size   Health   Ident  Fault  Available  
ceph1     /dev/sda  hdd   DB9876543214E   256G  Unknown  N/A    N/A    Yes        
ceph2     /dev/sda  hdd   DB9876543214E   256G  Unknown  N/A    N/A    Yes        
ceph3     /dev/sda  hdd   DB9876543214E   256G  Unknown  N/A    N/A    Yes    

# Available: Yes の全デバイスを対象に OSD を追加
$ ceph orch apply osd --all-available-devices
Scheduled osd.all-available-devices update...

# OSD Tree で状態確認
$ ceph osd tree
```

```yaml
service_type: osd
service_id: osd_spec_default
placement:
  host_pattern: '*'
data_devices:
  all: true
encrypted: false
objectstore: bluestore
```


## ceph.conf の配布

https://docs.ceph.com/en/latest/cephadm/operations/#etc-ceph-ceph-conf

クラスタ構築後に設定ファイルをサブノードに配布するためには、以下のコマンドを実行する。

```bash
ceph config set mgr mgr/cephadm/manage_etc_ceph_ceph_conf true
```


## RADOSGW

https://docs.ceph.com/en/latest/cephadm/rgw/

```bash
$ radosgw-admin realm create --rgw-realm=myorg --default
$ radosgw-admin zonegroup create --rgw-zonegroup=japan --endpoints http://ceph2:80 --rgw-realm=myorg --master --default
$ radosgw-admin zone create --rgw-zonegroup=japan --rgw-zone=japan-east-1 --master --default
$ ceph orch apply rgw myorg japan-east-1 --placement="2 ceph2 ceph3"
$ radosgw-admin period update --rgw-realm=myorg --commit
```

### HA-RADOSGW

```bash
$ cat <<EOF > /etc/sysctl.d/radosgw_ipv4.conf
net.ipv4.ip_forward = 1
net.ipv4.ip_nonlocal_bind = 1
EOF
$ sysctl -p /etc/sysctl.d/radosgw_ipv4.conf
$ ceph orch apply -i radosgw.yaml
```


## ダッシュボードのオブジェクトストレージ画面有効化

```bash
$ radosgw-admin user create --uid=admin --display-name=admin --system
$ radosgw-admin user info --uid=admin
$ ceph dashboard set-rgw-api-user-id admin
$ ceph dashboard set-rgw-api-access-key A6D1B1JL68472ZH972ZL
$ ceph dashboard set-rgw-api-secret-key zb3RBRkLJKBLlzdz7hJNfY6VOY5ZK8vjF5EUabrs
```


## やり直し手順

Cephadm で構築したクラスタとディスクの初期化をおこなって、クラスタ構築をやり直します。

```bash
# サブノードで実行
$ systemctl stop docker.socket

# メインノードで実行
$ cephadm rm-cluster --fsid <fsid> --force
$ rm -i /etc/systemd/system/ceph.target 

# サブノートで実行
$ rm -rf /etc/systemd/system/ceph*

# 各ノードで実行
$ DISK="/dev/sda"
$ sgdisk --zap-all $DISK
$ dd if=/dev/zero of="$DISK" bs=1M count=100 oflag=direct,dsync
$ ls /dev/mapper/ceph-* | xargs -I% -- dmsetup remove %
$ rm -rf /dev/ceph-*
```


## トラブルシュート

OSD コンテナが activate → deactivate → activate の無限ループで起動しないので調査します。

docker のロギングドライバーを `json-file` から `journald` に変更。

```bash
$ cat <<EOF > etc/docker/daemon.json 
{
  "log-driver": "journald"
}
EOF
$ systemctl restart docker
$ docker info
$ journalctl -f
(snip)
 3月 02 23:19:52 ceph2 containerd[8412]: time="2021-03-02T23:19:52.094657155+09:00" level=error msg="add cg to OOM monitor" error="cgroups: memory cgroup not supported on this system"
(snip)
```

- cgroup memory

`/boot/cmdline.txt` に `cgroup_memory=1 cgroup_enable=memory` を追記

効果なしっぽい

- SELinux

/etc/selinux/config


OSD 用のデバイスで bluestore/firestore どっち使うか判定できていない。

https://github.com/ceph/ceph/blob/v15.2.9/src/ceph_osd.cc#L276
https://github.com/ceph/ceph/blob/master/src/os/bluestore/BlueStore.cc#L5026
https://github.com/ceph/ceph/blob/master/src/os/bluestore/BlueStore.cc#L5093

```bash
$ FSID=<fsid>
$ OSD=2
$ cat <<EOF > /var/lib/ceph/${FSID}/osd.${OSD}/type
bluestore
EOF

# 以下のコマンドを /var/lib/ceph/${FSID}/osd.${OSD}/unit.run の中の osd.activate コンテナで実行する
$ /usr/bin/mount -t tmpfs tmpfs /var/lib/ceph/osd/ceph-1
$ /usr/bin/chown -R ceph:ceph /var/lib/ceph/osd/ceph-1
$ /usr/bin/ceph-bluestore-tool --cluster=ceph prime-osd-dir --dev /dev/ceph-959f5584-a212-4e2a-a0d0-71531ab24727/osd-block-40f0f67b-f973-41e2-8390-ed6116bbbd84 --path /var/lib/ceph/osd/ceph-1 --no-mon-config
$ /usr/bin/ln -snf /dev/ceph-959f5584-a212-4e2a-a0d0-71531ab24727/osd-block-40f0f67b-f973-41e2-8390-ed6116bbbd84 /var/lib/ceph/osd/ceph-1/block
$ /usr/bin/chown -h ceph:ceph /var/lib/ceph/osd/ceph-1/block
$ /usr/bin/chown -R ceph:ceph /dev/mapper/ceph--959f5584--a212--4e2a--a0d0--71531ab24727-osd--block--40f0f67b--f973--41e2--8390--ed6116bbbd84
$ /usr/bin/chown -R ceph:ceph /var/lib/ceph/osd/ceph-1

```text
3月 23 06:56:38 ceph1 bash[12317]: debug 2021-03-22T21:56:38.372+0000 7fae96f200  1 mon.ceph1@0(leader).osd e6 _set_new_cache_sizes cache_size:1020054731 inc_alloc: 369098752 full_alloc: 369098752 kv_alloc: 272629760
3月 23 06:56:38 ceph1 bash[23335]: Running command: /usr/bin/mount -t tmpfs tmpfs /var/lib/ceph/osd/ceph-2
3月 23 06:56:38 ceph1 bash[23335]: Running command: /usr/bin/chown -R ceph:ceph /var/lib/ceph/osd/ceph-2
3月 23 06:56:38 ceph1 bash[23335]: Running command: /usr/bin/ceph-bluestore-tool --cluster=ceph prime-osd-dir --dev /dev/ceph-e3a34c4d-3fde-4e66-88a3-edc14c59262b/osd-block-0dbc31f0-333a-47cc-a0b5-53f85c92f2e7 --path /var/lib/ceph/osd/ceph-2 --no-mon-config
3月 23 06:56:38 ceph1 bash[23335]: Running command: /usr/bin/ln -snf /dev/ceph-e3a34c4d-3fde-4e66-88a3-edc14c59262b/osd-block-0dbc31f0-333a-47cc-a0b5-53f85c92f2e7 /var/lib/ceph/osd/ceph-2/block
3月 23 06:56:38 ceph1 bash[23335]: Running command: /usr/bin/chown -h ceph:ceph /var/lib/ceph/osd/ceph-2/block
3月 23 06:56:38 ceph1 bash[23335]: Running command: /usr/bin/chown -R ceph:ceph /dev/mapper/ceph--e3a34c4d--3fde--4e66--88a3--edc14c59262b-osd--block--0dbc31f0--333a--47cc--a0b5--53f85c92f2e7
3月 23 06:56:38 ceph1 bash[23335]: Running command: /usr/bin/chown -R ceph:ceph /var/lib/ceph/osd/ceph-2
3月 23 06:56:38 ceph1 bash[23335]: --> ceph-volume lvm activate successful for osd ID: 2
```


