# ラズパイ初期設定

Raspberry Pi 4 に Ceph クラスタを構築するための初期設定をおこないます。

## OS イメージ

[Raspberry Pi Imager](https://www.raspberrypi.org/software/){target=_blank} を使って、ブート用 MicroSD カードに [CentOS 7 ARM64](http://isoredirect.centos.org/altarch/7/isos/aarch64/){target=_blank} の OS イメージを焼きます。

## rootfs 拡張

```bash
$ df -h
$ /usr/bin/rootfs-expand
$ df -h
```

## ロケール

```bash
$ localectl set-keymap jp106
$ timedatectl set-timezone Asia/Tokyo
```

## ネットワーク

```bash
$ nmcli d
# $ nmcli dev wifi list
# $ nmcli --ask dev wifi connect <SSID>

$ nmcli connection modify "Wired connection 1" ipv4.method manual
$ nmcli connection modify "Wired connection 1" ipv4.addresses 192.168.1.21/24
$ nmcli connection modify "Wired connection 1" ipv4.gateway 192.168.1.1
$ nmcli connection modify "Wired connection 1" ipv4.dns 192.168.2.3
$ nmcli connection reload
$ nmcli connection down "Wired connection 1" && nmcli con up "Wired connection 1"

# IPv6 無効化
$ cat <<EOF > /etc/sysctl.d/disable_ipv6.conf
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
EOF
$ sysctl -p
```

## ホスト名

ラズパイ毎にホスト名を設定します。

```bash
$ hostnamectl set-hostname ceph1
```