# Velero

Velero は、Kubernetes のリソースや PV (Persistent Volume) のバックアップ・リストアをするための OSS です。

https://velero.io/

Velero はオブジェクトストレージにバックアップデータを格納します。
オブジェクトストレージを用意する方法はいくつかあります。

## MinIO

オブジェクトストレージを MinIO で用意します。

https://velero.io/docs/v1.5/contributions/minio/

### Velero デプロイ

```bash
velero install \
    --provider aws \
    --plugins velero/velero-plugin-for-aws:v1.0.0 \
    --bucket velero \
    --secret-file ./credentials-velero \
    --use-volume-snapshots=false \
    --backup-location-config region=minio,s3ForcePathStyle="true",s3Url=http://minio.minio.svc:9000
```



## Velero & Rook Ceph

Velero が使用するオブジェクトストレージを Rook Ceph を使って用意します。

### Rook Ceph オブジェクトストレージ

Rook インストール済みであれば、サンプルマニフェストをデプロイするだけ。

```bash
kubectl apply -f rook/cluster/examples/kubernetes/ceph/object-test.yaml
kubectl apply -f rook/cluster/examples/kubernetes/ceph/storageclass-bucket-delete.yaml
kubectl apply -f rook/cluster/examples/kubernetes/ceph/object-bucket-claim-delete.yaml

export AWS_HOST=$(kubectl get cm ceph-delete-bucket -o jsonpath='{.data.BUCKET_HOST}')
export AWS_ACCESS_KEY_ID=$(kubectl get secret ceph-delete-bucket -o jsonpath='{.data.AWS_ACCESS_KEY_ID}' | base64 --decode)
export AWS_SECRET_ACCESS_KEY=$(kubectl get secret ceph-delete-bucket -o jsonpath='{.data.AWS_SECRET_ACCESS_KEY}' | base64 --decode)
```

`rook-ceph-tools` Pod を使ってオブジェクトストレージが使えるか確認をします。

```bash
kubectl exec -it $(kubectl get pods -l app=rook-ceph-tools -o=jsonpath={.items[*].metadata.name}) -- bash
export AWS_HOST=<host>
export AWS_ENDPOINT=<endpoint>
export AWS_ACCESS_KEY_ID=<accessKey>
export AWS_SECRET_ACCESS_KEY=<secretKey>

echo "Hello Rook" > /tmp/rookObj

# Upload
# <bucket-name> は ObjectBucketClame から参照可能
s3cmd put /tmp/rookObj --no-ssl --host=${AWS_HOST} --host-bucket=  s3://<bucket-name>

# Download
s3cmd get s3://<bucket-name>/rookObj /tmp/rookObj-download --no-ssl --host=${AWS_HOST} --host-bucket=
```

### Velero デプロイ

```bash
velero install \
    --provider aws \
    --plugins velero/velero-plugin-for-aws:v1.0.0 \
    --bucket ceph-bkt-c71df44b-8657-4f02-b680-fe0894debc07 \
    --secret-file ./credentials-velero \
    --use-volume-snapshots=false \
    --backup-location-config region=us-east-1,s3ForcePathStyle="true",s3Url=http://rook-ceph-rgw-my-store.rook-ceph.svc
```

## 検証

```bash
# テスト用リソースの準備
kubectl apply -f nginx-base.yaml
kubectl get deployments -l component=velero --namespace=velero
kubectl get deployments --namespace=nginx-example

# バックアップ
velero backup create nginx-backup2 --selector app=nginx

# バックアップリソースの確認
velero backup get
velero backup describe nginx-backup
velero backup logs nginx-backup

# テスト用リソースの削除
kubectl delete namespace nginx-example
kubectl get deployments --namespace=nginx-example

# リストア
velero restore create --from-backup nginx-backup2

# リストアリソースの確認
velero restore get
kubectl get deployments --namespace=nginx-example

```


## アンインストール

https://velero.io/docs/v1.5/uninstalling/

```bash
kubectl delete namespace/velero clusterrolebinding/velero
kubectl delete crds -l component=velero
```