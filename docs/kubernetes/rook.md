# Rook

[Rook](https://rook.github.io/){target=_blank} は Kubernetes に各種ストレージソリューション (Ceph, Cassandra, etc...) を提供してくれるストレージオーケストレータです。

ここでは、Rook を使って Ceph クラスタを構築し、K8s クラスタにブロックストレージとオブジェクトストレージの StorageClass を提供する手順を説明します。

!!! note
    本手順は v1.5 をベースにしています。

## Install

[Ceph Storage Quickstart](https://rook.github.io/docs/rook/v1.5/ceph-quickstart.html){target=_blank}

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

構築完了後には以下の StorageClass が利用可能となります。

```bash
$ kubectl get sc
NAME              PROVISIONER                  RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
rook-ceph-block   rook-ceph.rbd.csi.ceph.com   Delete          Immediate           true                   57d
```


## Dashboard

```bash
$ kubectl apply -f rook/cluster/examples/kubernetes/ceph/dashboard-loadbalancer.yaml

# パスワード確認
$ kubectl -n rook-ceph get secret rook-ceph-dashboard-password -o jsonpath="{['data']['password']}" | base64 --decode && echo
```


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


## Extenal Cluster

```bash
$ kubectl apply -f crds.yaml
$ kubectl apply -f common.yaml
$ kubectl apply -f operator.yaml
$ kubectl apply -f common-external.yaml
```

### 環境変数

- `NAMESPACE`
- `ROOK_EXTERNAL_FSID`: Ceph クラスタで `ceph fsid` コマンドを実行して fsid を確認して設定します。
- `ROOK_EXTERNAL_CEPH_MON_DATA`: MON の情報を設定します。
- `ROOK_EXTERNAL_ADMIN_SECRET`: Ceph クラスタで `ceph auth get-key client.admin` コマンドを実行して、Ceph クラスタの admin シークレットキーを取得します。

Ceph ノードで External Cluster 環境変数として使う情報を取得します。

```bash
$ bash create-external-cluster-resources.sh
export ROOK_EXTERNAL_USER_SECRET=AQBMvFBgH++mJBAAJKv/9LAWvc4I09WfycMLUg==
export ROOK_EXTERNAL_USERNAME=client.healthchecker
export CSI_RBD_NODE_SECRET_SECRET=AQB6aU1gcIRuHRAAM20ehPo8QwTrPUy2Eea4DA==
export CSI_RBD_PROVISIONER_SECRET=AQB5aU1gSl1dOxAAV81b2Puiih2VBajLlHq7Cw==
export CSI_CEPHFS_NODE_SECRET=AQB7aU1gcfuQHBAAqliipwmEdNo6jdFNJaI5oA==
export CSI_CEPHFS_PROVISIONER_SECRET=AQB6aU1gW3RzOhAAp39Bq0a0sMBnyWd7Bs9bbg==
successfully created users and keys, execute the above commands and run import-external-cluster.sh to inject them in your Kubernetes cluster.
```

```bash
$ export NAMESPACE=rook-ceph-external
$ export ROOK_EXTERNAL_FSID=c70278a6-8275-11eb-ab03-dca6329a21a7
$ export ROOK_EXTERNAL_CEPH_MON_DATA=a=192.168.1.21:6789,b=192.168.1.22:6789,c=192.168.1.23:6789
$ export ROOK_EXTERNAL_ADMIN_SECRET=AQCdKEpglwQYNhAAd0rszoHkeCD+JJrz2YsSzA==
$ bash import-external-cluster.sh
```

### CephCluster デプロイ


https://github.com/rook/rook/issues/5732#issuecomment-756042524

```bash
$ kubectl apply -f cluster-external.yaml
```


## Remove an OSD

https://rook.io/docs/rook/v1.5/ceph-osd-mgmt.html#remove-an-osd

Host-based クラスタから OSD を除外します。
[osd-purge.yaml](https://github.com/rook/rook/blob/release-1.5/cluster/examples/kubernetes/ceph/osd-purge.yaml){target=_blank} 内の `<OSD-IDs>` に除外する OSD の ID を指定して、以下のコマンドを実行します。

```bash
kubectl scale deploy rook-ceph-operator --replicas=0
kubectl apply -f osd-purge.yml
kubectl scale deploy rook-ceph-operator --replicas=1
```