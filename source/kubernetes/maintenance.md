# Maintenance

## worker ノードの停止

worker ノードを停止する前に、`kubectl drain <node name>` を使用して対象ノードで動いている Pod を他のノードに移動させる必要がある。

```bash
# worker ノードの確認
$ kubectl get nodes -o wide
NAME                          STATUS   ROLES    AGE   VERSION   INTERNAL-IP    EXTERNAL-IP   OS-IMAGE                KERNEL-VERSION                CONTAINER-RUNTIME
kubeadm-master1.nnstt1.work   Ready    master   58d   v1.19.0   192.168.2.20   <none>        CentOS Linux 7 (Core)   3.10.0-1127.19.1.el7.x86_64   docker://19.3.12
kubeadm-worker1.nnstt1.work   Ready    <none>   58d   v1.19.0   192.168.2.22   <none>        CentOS Linux 7 (Core)   3.10.0-1127.19.1.el7.x86_64   docker://19.3.12
kubeadm-worker2.nnstt1.work   Ready    <none>   58d   v1.19.0   192.168.2.23   <none>        CentOS Linux 7 (Core)   3.10.0-1127.19.1.el7.x86_64   docker://19.3.12
kubeadm-worker3.nnstt1.work   Ready    <none>   57d   v1.19.0   192.168.2.24   <none>        CentOS Linux 7 (Core)   3.10.0-1127.19.1.el7.x86_64   docker://19.3.12

# drain
$ kubectl drain <node name> --delete-local-data --ignore-daemonsets
```

- `--delete-local-data`

    対象ノードで動いている Pod が local storage を使っている場合は以下のエラーで drain が失敗する。

        cannot delete Pods with local storage (use --delete-local-data to override): arc/logsdb-0, (以下 Pod 名)

    当該オプションをつけることで、local storage のデータも含めて Pod を削除することが可能となる。

    !!! attention
        local storage に永続データを残すことは運用的に不適切なので、Storage Class を使用するようにマニフェストを修正したほうがよい

- `--ignore-daemonsets`

    daemonsets リソースが存在する場合にエラーとなる。

        error: cannot delete DaemonSet-managed Pods (use --ignore-daemonsets to ignore): arc/metricsdc-rmv74, (以下 Pod 名)

    当該オプションをつけることで、daemonsets の Pod を削除することが可能となる。

`kubectl drain` が成功すればノードを再起動しても安全な状態となる。

ノード再起動後は Pod がスケジュールされない状態となっているため、`kubectl uncordon <node name>` でスケジュールを有効化する。

```bash
$ kubectl uncordon kubeadm-worker1.nnstt1.work
node/kubeadm-worker1.nnstt1.work uncordoned
```