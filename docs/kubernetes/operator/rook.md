# Rook

[Rook](https://rook.github.io/){target=_blank} は Kubernetes に各種ストレージソリューション (Ceph, Cassandra, etc...) を提供してくれるストレージオーケストレータです。

ここでは、Rook を使って Ceph クラスタを構築し、K8s クラスタにブロックストレージとオブジェクトストレージの StorageClass を提供する手順を説明します。

!!! note
    本手順は v1.5 をベースにしています。

## Install

[Ceph Storage Quickstart](https://rook.github.io/docs/rook/v1.5/ceph-quickstart.html){target=_blank}

### Rook Ceph

最新版の Rook をリポジトリから clone して、Rook を K8s クラスタにインストールします。

```bash
$ git clone --single-branch --branch v1.5.7 https://github.com/rook/rook.git
$ cd rook/cluster/examples/kubernetes/ceph
$ kubectl apply -f crds.yaml
$ kubectl apply -f common.yaml
$ kubectl apply -f operator.yaml
$ kubectl apply -f cluster.yaml
```

Rook による OSD のデプロイ方法は2通りあります。

- Host-based OSDs/MONs
- PVC-based OSDs/MONs (OSD on PVC)

デプロイ方法の違いについては、こちらのスライドが参考になります。

- [Rook v1.1なら出来るPVC-basedな分散ストレージ](https://speakerdeck.com/tzkoba/rook-v1-dot-1narachu-lai-rupvc-basednafen-san-sutorezi){target=_blank}
- [Rook/Ceph OSD on PVC in practice](https://speakerdeck.com/sat/ceph-osd-on-pvc-in-practice){target=_blank}


### Block Storage

RBD 用のマニフェストをデプロイします。
このマニフェストには `StorageClass` リソースの他に、`CephBlockPool` というカスタムリソースが使用されています。

```bash
$ kubectl apply -f csi/rbd/storageclass.yaml

# ceph-tools コンテナにログインして Ceph の構築状況を確認
$ kubectl apply -f toolbox.yaml
$ kubectl -n rook-ceph exec -it $(kubectl -n rook-ceph get pod -l "app=rook-ceph-tools" -o jsonpath='{.items[0].metadata.name}') -- bash
$ ceph status
```

構築完了後には StorageClass が利用可能となる。

マニフェストの PersistentVolume で以下の StorageClass を指定することで、Ceph Block Storage を利用できる。

```bash
$ kubectl get sc
NAME              PROVISIONER                  RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
rook-ceph-block   rook-ceph.rbd.csi.ceph.com   Delete          Immediate           true                   57d
```

### Object Storage

## Dashboard

### Deploy

```bash
$ kubectl apply -f rook/cluster/examples/kubernetes/ceph/dashboard-loadbalancer.yaml
```

### Login

```bash
# パスワード確認
$ kubectl -n rook-ceph get secret rook-ceph-dashboard-password -o jsonpath="{['data']['password']}" | base64 --decode && echo
```

https://192.168.2.244:8443

管理者ユーザは admin


## Cleanup

Rook をクリーンアップする場合の手順です。
各 Worker Node で実施します。

### Rook 関連データ削除

```bash
$ rm -rf /var/lib/rook
```

### ブロックデバイス初期化

初期化するブロックデバイス毎に下記コマンドを実行します。

```sh
#!/usr/bin/env bash
# https://github.com/rook/rook/blob/master/Documentation/ceph-teardown.md
$ DISK="/dev/sdb"
# Zap the disk to a fresh, usable state (zap-all is important, b/c MBR has to be clean)
# You will have to run this step for all disks.
$ sgdisk --zap-all $DISK
# Clean hdds with dd
$ dd if=/dev/zero of="$DISK" bs=1M count=100 oflag=direct,dsync
# Clean disks such as ssd with blkdiscard instead of dd
$ blkdiscard $DISK

# These steps only have to be run once on each node
# If rook sets up osds using ceph-volume, teardown leaves some devices mapped that lock the disks.
$ ls /dev/mapper/ceph-* | xargs -I% -- dmsetup remove %
# ceph-volume setup can leave ceph-<UUID> directories in /dev (unnecessary clutter)
$ rm -rf /dev/ceph-*
```


## Kubernetes v1.20 で動かない場合

Kubernetes v1.20 から Feature Gate `CSIVolumeFSGroupPolicy` が Beta に昇格しており、機能が有効状態となりました。

その影響で Rook が動作しない場合があり、暫定対処として旧バージョンと同じ状態にする `--feature-gates=CSIVolumeFSGroupPolicy=false` を kubelet に設定しました。

```ini
# /var/lib/kubelet/kubeadm-flags.env
KUBELET_KUBEADM_ARGS="--network-plugin=cni --pod-infra-container-image=k8s.gcr.io/pause:3.2 --container-runtime=remote --container-runtime-endpoint=/run/containerd/containerd.sock --feature-gates=CSIVolumeFSGroupPolicy=false"
```