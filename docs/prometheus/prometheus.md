# Prometheus

## kube-prometheus

```bash
sudo yum install epel-release
sudo yum install golang

export GOPATH=/home/nnstt1/go
export PATH=$PATH:$GOPATH/bin

GO111MODULE="on" go get github.com/jsonnet-bundler/jsonnet-bundler/cmd/jb
go get github.com/google/go-jsonnet/cmd/jsonnet
go get github.com/brancz/gojsontoyaml

jb install github.com/prometheus-operator/kube-prometheus/jsonnet/kube-prometheus@release-0.6
```



```bash
kubectl apply -f manifests/setup
kubectl apply -f manifests/

kubectl get all -n monitoring
kubectl apply -f grafana-service.yaml
```

```yaml
# grafana-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: grafana
  labels:
    app: grafana
  namespace: monitoring
spec:
  ports:
  - port: 80
    targetPort: 3000
  selector:
    app: grafana
  type: LoadBalancer
```

