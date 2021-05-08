# コンテナランタイム

## containerd

Kubernetes のコンテナランタイムを `containerd` に変更する。

```bash
kubectl drain <k8s-node> --ignore-daemonsets --delete-local-data

# k8s-node
yum install -y yum-utils device-mapper-persistent-data lvm2
yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo
yum update -y && yum install -y containerd.io
mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
systemctl restart containerd
yum remove -y docker-ce

cat <<EOF > /var/lib/kubelet/kubeadm-flags.env
KUBELET_KUBEADM_ARGS="--network-plugin=cni --pod-infra-container-image=k8s.gcr.io/pause:3.2 --container-runtime=remote --container-runtime-endpoint=/run/containerd/containerd.sock"
EOF

systemctl restart kubelet

kubectl uncordon <k8s-node>
```