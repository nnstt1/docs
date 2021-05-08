# Istio

```bash
curl -L https://istio.io/downloadIstio | sh -
cd istio-1.7.0
export PATH="$PATH:/home/nnstt1/istio/istio-1.7.0/bin"
istioctl x precheck
istioctl install --set profile=demo
kubectl label namespace default istio-injection=enabled

## sample
kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml
kubectl exec "$(kubectl get pod -l app=ratings -o jsonpath='{.items[0].metadata.name}')" -c ratings -- curl -s productpage:9080/productpage | grep -o "<title>.*</title>"

kubectl apply -f samples/bookinfo/networking/bookinfo-gateway.yaml
istioctl analyze
```