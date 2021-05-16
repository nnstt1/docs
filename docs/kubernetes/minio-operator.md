# MinIO Operator

S3 互換のオブジェクトストレージである MinIO を Kubernetes 上で運用するための [Minio Operator](https://github.com/minio/operator) について説明します。


## 前提条件

- **Kubernetes クラスタのバージョンが 1.17.0 以上であること**

    1.17.0 未満だったら Kubernetes 公式を参照してアップグレードします。

    ```bash
    $ kubectl version
    Client Version: version.Info{Major:"1", Minor:"21", GitVersion:"v1.21.0", GitCommit:"cb303e613a121a29364f75cc67d3d580833a7479", GitTreeState:"clean", BuildDate:"2021-04-08T16:31:21Z", GoVersion:"go1.16.1", Compiler:"gc", Platform:"linux/amd64"}
    Server Version: version.Info{Major:"1", Minor:"21", GitVersion:"v1.21.0", GitCommit:"cb303e613a121a29364f75cc67d3d580833a7479", GitTreeState:"clean", BuildDate:"2021-04-08T16:25:06Z", GoVersion:"go1.16.1", Compiler:"gc", Platform:"linux/amd64"}
    ```

- **Kubernetes クラスタに MinIO テナント用の namespace があること**

    テナント毎に namespace を作成します。

    ```bash
    $ kubectl create namespace minio-tenant-1
    namespace/minio-tenant-1 created
    ```

- **Kubernetes クラスタに `volumeBindingMode: WaitForFirstConsumer` の StorageClass があること**

    Rook/Ceph を使って MinIO 用の StorageClass を作成します。

    ```yaml
    apiVersion: ceph.rook.io/v1
    kind: CephBlockPool
    metadata:
      name: minio-pool
      namespace: rook-ceph
    spec:
      failureDomain: host
      replicated:
        size: 3
        requireSafeReplicaSize: true
    ---
    apiVersion: storage.k8s.io/v1
    kind: StorageClass
    metadata:
      name: minio-rook-ceph-block
    provisioner: rook-ceph.rbd.csi.ceph.com
    parameters:
        clusterID: rook-ceph
        pool: replicapool
        imageFormat: "2"
        imageFeatures: layering
        csi.storage.k8s.io/provisioner-secret-name: rook-csi-rbd-provisioner
        csi.storage.k8s.io/provisioner-secret-namespace: rook-ceph
        csi.storage.k8s.io/controller-expand-secret-name: rook-csi-rbd-provisioner
        csi.storage.k8s.io/controller-expand-secret-namespace: rook-ceph
        csi.storage.k8s.io/node-stage-secret-name: rook-csi-rbd-node
        csi.storage.k8s.io/node-stage-secret-namespace: rook-ceph
        csi.storage.k8s.io/fstype: ext4
    allowVolumeExpansion: true
    reclaimPolicy: Delete
    volumeBindingMode: WaitForFirstConsumer
    ```

- **Krew がインストールされていること**

    [こちらのページ](./krew.md) を参照して Krew をインストールします。

## インストール

Krew を使って MinIO Operator をインストールしていきます。

```bash
$ kubectl krew update
Updated the local copy of plugin index.

$ kubectl krew install minio
Updated the local copy of plugin index.
Installing plugin: minio
Installed plugin: minio
\
 | Use this plugin:
 |      kubectl minio
 | Documentation:
 |      https://github.com/minio/operator/tree/master/kubectl-minio
 | Caveats:
 | \
 |  | * For resources that are not in default namespace, currently you must
 |  |   specify -n/--namespace explicitly (the current namespace setting is not
 |  |   yet used).
 | /
/
WARNING: You installed plugin "minio" from the krew-index plugin repository.
   These plugins are not audited for security by the Krew maintainers.
   Run them at your own risk.
```

インストールできたら、Operator を初期化します。

```bash
$ kubectl minio init
namespace/minio-operator created
serviceaccount/minio-operator created
clusterrole.rbac.authorization.k8s.io/minio-operator-role created
clusterrolebinding.rbac.authorization.k8s.io/minio-operator-binding created
customresourcedefinition.apiextensions.k8s.io/tenants.minio.min.io created
service/operator created
deployment.apps/minio-operator created
serviceaccount/console-sa created
clusterrole.rbac.authorization.k8s.io/console-sa-role created
clusterrolebinding.rbac.authorization.k8s.io/console-sa-binding created
configmap/console-env created
service/console created
deployment.apps/console created
-----------------

To open Operator UI, start a port forward using this command:

kubectl minio proxy -n minio-operator 

-----------------
```

上記出力の通り、proxy 経由で MinIO Operator の管理画面にアクセスできるようになります。

```bash
$ kubectl minio proxy -n minio-operator
Starting port forward of the Console UI.

To connect open a browser and go to http://localhost:9090

Current JWT to login: eyJhbGciOiJSUzI1NiIsImtpZCI6Ii0zbFc4cm51LXNPTWliSUVOYXdTNmI3ejRWeWRuVEF0dU9aenpKY0xaNGMifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJtaW5pby1vcGVyYXRvciIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJjb25zb2xlLXNhLXRva2VuLWJmNDZyIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6ImNvbnNvbGUtc2EiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiI3ZDgwZDNjOS0wMzUyLTRjNDItOWQ5Yy0xM2I3ZDg2ZjA4OWIiLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6bWluaW8tb3BlcmF0b3I6Y29uc29sZS1zYSJ9.F8XsI2buT71U4QUi7tfShaDYAOs0rcgjPmZO5HiJYyWYZ0ZYN5LDZbIRf4il23CGRjGVUWksKsfeUqRFD0iG2lP21WxvTgIbUov3DbZtItB3Sj1dO4xI5WH7PLjyYAyE5eYGRQwd3K0qSpaexh4cKsG8vj5pxzd8g-f6vwWjE1Co7Ax83PzP_ATmNnOWVQE-6lI287QijtH-PfdK3UEHM1d1WU67yvei0YGLTyMgC3t6F8xKwqay0B2OLEUGirDcq74xZA5qB_Ipuvys5K05nOCKbQiWujv0J_gyyaEAwszRpGc4Gy8Mrc-bhAYe7EGLh8_BoQiwJ2d4UVcBEuEmmw

Forwarding from 0.0.0.0:9090 -> 9090
```

ブラウザで `kubectl minio proxy` コマンドを実行しているサーバにアクセスするとログイン画面が表示されるので、コマンド実行時に出力された文字列を入力してログインします。

![GUI Login](../images/minio-operator-gui-login.png)

![GUI](../images/minio-operator-gui.png)

## テナント作成

MinIO Operator を使ってテナントを作成します。

```bash
$ kubectl minio tenant create minio-tenant-1 \
    --servers 1 \
    --volumes 4 \
    --capacity 4Gi \
    --namespace minio-tenant-1 \
    --storage-class minio-rook-ceph-block

Tenant 'minio-tenant-1' created in 'minio-tenant-1' Namespace

  Username: admin 
  Password: ac8a676c-d4a3-4cb0-98b9-c432f3c9cd5c 
  Note: Copy the credentials to a secure location. MinIO will not display these again.

+-------------+------------------------+----------------+--------------+--------------+
| APPLICATION | SERVICE NAME           | NAMESPACE      | SERVICE TYPE | SERVICE PORT |
+-------------+------------------------+----------------+--------------+--------------+
| MinIO       | minio                  | minio-tenant-1 | ClusterIP    | 443          |
| Console     | minio-tenant-1-console | minio-tenant-1 | ClusterIP    | 9443         |
+-------------+------------------------+----------------+--------------+--------------+
```

`--volumes` が 4 未満だった場合、エラーとなります。

```bash
$ kubectl minio tenant create minio-tenant-1 \
    --servers 1 \
    --volumes 3 \
    --capacity 3Gi \
    --namespace minio-tenant-1 \
    --storage-class minio-rook-ceph-block
Error: pool #0 setup must have a minimum of 4 volumes per server
```

### トラブル

テナント作成コマンド実行後、MinIO Operator によって Pod が作成されますが、`minio-tenant-1-tls` Secret をマウントできずに Pod が起動してきませんでした。

```bash
$ kubectl describe pods minio-tenant-1-ss-0-0 -n minio-tenant-1
(snip)
Events:
  Type     Reason                  Age                 From                     Message
  ----     ------                  ----                ----                     -------
  Normal   Scheduled               2m17s               default-scheduler        Successfully assigned minio-tenant-1/minio-tenant-1-ss-0-0 to kubeadm-worker3.nnstt1.work
  Normal   SuccessfulAttachVolume  2m17s               attachdetach-controller  AttachVolume.Attach succeeded for volume "pvc-894acc42-ca13-4a25-87eb-d396acab50cc"
  Normal   SuccessfulAttachVolume  2m17s               attachdetach-controller  AttachVolume.Attach succeeded for volume "pvc-1af6963b-0658-49f1-a4a0-18488ad5d6e7"
  Normal   SuccessfulAttachVolume  2m17s               attachdetach-controller  AttachVolume.Attach succeeded for volume "pvc-6875cba8-5c7b-4bfe-95bb-ad491c2940ab"
  Normal   SuccessfulAttachVolume  2m17s               attachdetach-controller  AttachVolume.Attach succeeded for volume "pvc-153554c7-b5e8-4e4a-95dd-28f625af163f"
  Warning  FailedMount             14s                 kubelet                  Unable to attach or mount volumes: unmounted volumes=[minio-tenant-1-tls], unattached volumes=[3 minio-tenant-1-tls kube-api-access-7s52p 0 1 2]: timed out waiting for the condition
  Warning  FailedMount             8s (x9 over 2m16s)  kubelet                  MountVolume.SetUp failed for volume "minio-tenant-1-tls" : secret "minio-tenant-1-tls" not found
```

### 2021/5/13

Issue を参考にデバッグ用コンテナで証明書コピーしてみました。
https://github.com/minio/operator/issues/459#issuecomment-774319087

```bash
$ kubectl run my-shell  -i --tty --image miniodev/debugger -- bash
root@my-shell:/\# cp /var/run/secrets/kubernetes.io/serviceaccount/ca.crt /etc/ssl/certs/
root@my-shell:/\# mc config host add myminio https://minio.minio-tenant-1.svc.cluster.local admin b4e71593-29b4-4f93-8673-cf43009f74bf
mc: <ERROR> Unable to initialize new alias from the provided credentials. Get "https://minio.minio-tenant-1.svc.cluster.local/probe-bucket-sign-ezqrq20bdldg/?location=": dial tcp 10.97.131.201:443: connect: connection refused.
```

パスワードがあっていなかったようなので、テナント再作成したところ、新しいテナントでは `minio-tenant-1-tls` Sercret が作成されて Pod も正常に起動しました。

### 2021/5/14

Operator を削除してからやり直したところ、`kubectl minio tenant create` を実行して 4min ほど放置したら `minio-tenant-1-tls` も作成されました。

```bash
$ kubectl get secret
NAME                            TYPE                                  DATA   AGE
default-token-ntlzm             kubernetes.io/service-account-token   3      2m24s
minio-tenant-1-console-secret   Opaque                                4      2m22s
minio-tenant-1-console-tls      Opaque                                2      62s
minio-tenant-1-creds-secret     Opaque                                2      2m22s
minio-tenant-1-tls              Opaque                                2      2m2s
operator-tls                    Opaque                                1      2m17s
operator-webhook-secret         Opaque                                3      2m17s

$ kubectl get pods
NAME                    READY   STATUS    RESTARTS   AGE
minio-tenant-1-console-6b7488946f-2mvkf   1/1     Running   0          2m43s
minio-tenant-1-console-6b7488946f-6djht   1/1     Running   0          2m43s
minio-tenant-1-ss-0-0                     1/1     Running   0          3m43s
```

再度試したところ、今度は作られませんでした。
CSR リソースを確認したところ、前回作られたと思われるものが残っていました。

```bash
$ kubectl get csr
NAME                                        AGE   SIGNERNAME                     REQUESTOR                                             CONDITION
minio-tenant-1-console-minio-tenant-1-csr   44m   kubernetes.io/legacy-unknown   system:serviceaccount:minio-operator:minio-operator   Approved,Issued
minio-tenant-1-minio-tenant-1-csr           45m   kubernetes.io/legacy-unknown   system:serviceaccount:minio-operator:minio-operator   Approved,Issued
operator-minio-operator-csr                 46m   kubernetes.io/legacy-unknown   system:serviceaccount:minio-operator:minio-operator   Approved,Issued
```

CSR リソースを削除して再々度チャレンジ、`kubectl minio init` 後の CSR リソース。

```bash
$ kubectl get csr
NAME                          AGE   SIGNERNAME                     REQUESTOR                                             CONDITION
operator-minio-operator-csr   96s   kubernetes.io/legacy-unknown   system:serviceaccount:minio-operator:minio-operator   Approved,Issued
```

`kubectl minio tenant create` 後。

```bash
$ kubectl get csr
NAME                                        AGE    SIGNERNAME                     REQUESTOR                                             CONDITION
minio-tenant-1-console-minio-tenant-1-csr   17s    kubernetes.io/legacy-unknown   system:serviceaccount:minio-operator:minio-operator   Approved,Issued
minio-tenant-1-minio-tenant-1-csr           77s    kubernetes.io/legacy-unknown   system:serviceaccount:minio-operator:minio-operator   Approved,Issued
operator-minio-operator-csr                 4m5s   kubernetes.io/legacy-unknown   system:serviceaccount:minio-operator:minio-operator   Approved,Issued
```

新しく CSR リソースが作られ、`minio-tenant-1-tls` も作成されました。


## 外部公開用サービス

Kubernetes クラスタ外のアプリケーションから MinIO テナントに接続するためには Ingress や Load Balancer が必要です。

今回は MinIO 用の Load Balancer を作成します。

### MinIO

```yaml
apiVersion: v1
kind: Service
metadata:
  namespace: minio-tenant-1
  name: minio-loadbalancer
  labels:
    component: minio-tenant-1
  annotations:
    external-dns.alpha.kubernetes.io/hostname: minio-tenant-1.nnstt1.work.    # ExternalDNS 用アノテーション
spec:
  type: LoadBalancer
  ports:
    - port: 9000
      targetPort: 9000
      protocol: TCP
  selector:
    v1.min.io/tenant: minio-tenant-1
```

### Console

```yaml
apiVersion: v1
kind: Service
metadata:
  namespace: minio-tenant-1
  name: console-loadbalancer
  annotations:
    external-dns.alpha.kubernetes.io/hostname: minio-console.nnstt1.work.    # ExternalDNS 用アノテーション
spec:
  type: LoadBalancer
  ports:
    - name: https-console
      port: 9443
      protocol: TCP
      targetPort: 9443
  selector:
    v1.min.io/console: minio-tenant-1-console
```

## アンインストール

### テナント削除

MinIO Operator で作成したテナントを削除する方法です。

tenant リソースは namespace 毎に作成されるため、`-n` オプションで対象の namespace を指定してください。

```bash
# テナント削除
$ kubectl minio tenant delete minio-tenant-1 -n minio-tenant-1

This will delete the Tenant minio-tenant-1 and ALL its data. Do you want to proceed?: y
Deleting MinIO Tenant minio-tenant-1
Deleting MinIO Tenant Credentials Secret minio-tenant-1-creds-secret
Deleting MinIO Tenant Console Secret minio-tenant-1-console-secret

# namespace 確認
$ kubectl get pods -n minio-tenant-1
No resources found in minio-tenant-1 namespace.
$ kubectl get secret -n minio-tenant-1
NAME                  TYPE                                  DATA   AGE
default-token-swx2m   kubernetes.io/service-account-token   3      2d
```

### Operator 削除

Kubernetes クラスタから MinIO Operator を削除する方法です。

```bash
$ kubectl minio delete

Are you sure you want to delete ALL the MinIO Tenants and MinIO Operator?: y
namespace "minio-operator" deleted
serviceaccount "minio-operator" deleted
clusterrole.rbac.authorization.k8s.io "minio-operator-role" deleted
clusterrolebinding.rbac.authorization.k8s.io "minio-operator-binding" deleted
customresourcedefinition.apiextensions.k8s.io "tenants.minio.min.io" deleted
service "operator" deleted
deployment.apps "minio-operator" deleted
serviceaccount "console-sa" deleted
clusterrole.rbac.authorization.k8s.io "console-sa-role" deleted
clusterrolebinding.rbac.authorization.k8s.io "console-sa-binding" deleted
configmap "console-env" deleted
service "console" deleted
deployment.apps "console" deleted
```
