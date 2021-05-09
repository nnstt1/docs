# Ceph-Grafana コンテナ


## ビルド

ARM アーキテクチャ用の ceph-grafana コンテナイメージをビルドします。

Docker か Podman がインストールされている前提です。

```bash
$ yum install -y jq make buildah

$ echo <<EOF > /etc/sysctl.d/42-rootless.conf
user.max_user_namespaces=10000
EOF
$ sysctl --system

$ vim /etc/default/grub
# systemd.unified_cgroup_hierarchy=0 を追加
# GRUB_CMDLINE_LINUX="crashkernel=auto rd.lvm.lv=centos/root rd.lvm.lv=centos/swap rhgb quiet systemd.unified_cgroup_hierarchy=0"
$ grub2-mkconfig
$ shutdown -r now

$ sudo make ceph_version=octopus
```


## 使用方法

カスタムイメージを Ceph で使う設定です。

```bash
$ ceph config set mgr mgr/cephadm/container_image_grafana nnstt1/ceph-grafana:6.7.4
```

初期構築時に監視スタックを構築しなかった場合のマニュアル構築です。

```bash
$ ceph mgr module enable prometheus
$ ceph orch apply node-exporter '*'
$ ceph orch apply alertmanager 1
$ ceph orch apply prometheus 1
$ ceph orch apply grafana 1
```