# kube-prometheus

`kube-prometheus` という Prometheus Operator を使った Prometheus Stack を Kubernetes クラスタにデプロイする手順です。

## Install

Golang と Jsonnet と Jsonnet-bundler をインストールします。

```bash
sudo yum install epel-release
sudo yum install golang

export GOPATH=/home/nnstt1/go
export PATH=$PATH:$GOPATH/bin

GO111MODULE="on" go get github.com/jsonnet-bundler/jsonnet-bundler/cmd/jb
go get github.com/google/go-jsonnet/cmd/jsonnet
go get github.com/brancz/gojsontoyaml
```

インストールが完了したら公式のサンプル Jsonnet ファイルを使ってマニフェストを生成します。

```bash
$ mkdir my-kube-prometheus; cd my-kube-prometheus
$ RELEASE_VER=release-0.8
$ jb init
$ jb install github.com/prometheus-operator/kube-prometheus/jsonnet/kube-prometheus@$RELEASE_VER
$ wget https://raw.githubusercontent.com/prometheus-operator/kube-prometheus/$RELEASE_VER/example.jsonnet -O example.jsonnet
$ wget https://raw.githubusercontent.com/prometheus-operator/kube-prometheus/$RELEASE_VER/build.sh -O build.sh

$ ./build.sh example.jsonnet
```

ビルドが完了したら manifests/ ディレクトリ配下にマニフェストが作成されます。
Namespace や CRD といった先にクラスタへデプロイすべきマニフェストが manifests/setup/ ディレクトリに含まれているので、先にデプロイします。

```bash
$ kubectl apply -f manifests/setup
$ kubectl apply -f manifests/
```


## カスタム

Prometheus や Grafana のサービスにクラスタ外からアクセスできるようにするために Ingress を使います。
Ingress 用のマニフェストも Jsonnet から生成するため、`example.jsonnet` を編集します。

ここでは、カスタムした Jsonnet と分かるようにファイル名を変更します。

```bash
$ cp example.jsonnet home-cluster.jsonnet
```

### Ingress

サンプル Jsonnet では Prometheus や Grafana にクラスタ外からアクセスするためのマニフェストは出力されません。
各サービスにアクセスするための Ingress リソースを出力するように Jsonnet を編集します。

事前に Nginx Ingress Controller をクラスタにデプロイしていることが条件です。

```jsonnet
    ingress+:: {
      'prometheus-k8s': {
        apiVersion: 'networking.k8s.io/v1',
        kind: 'Ingress',
        metadata: {
          name: $.prometheus.prometheus.metadata.name,
          namespace: $.prometheus.prometheus.metadata.namespace,
          annotations: {
            'kubernetes.io/ingress.class': 'nginx',
          },
        },
        spec: {
          rules: [{
            host: 'prometheus.nnstt1.work',
            http: {
              paths: [{
                backend: {
                  service: {
                    name: $.prometheus.service.metadata.name,
                    port: {
                      number: 9090
                    },
                  },
                },
                path: '/',
                pathType: 'Prefix',
              }],
            },
          }],
        },
      },
      'grafana': {
        apiVersion: 'networking.k8s.io/v1',
        kind: 'Ingress',
        metadata: {
          name: $.grafana.service.metadata.name,
          namespace: $.grafana.service.metadata.namespace,
          annotations: {
            'kubernetes.io/ingress.class': 'nginx',
          },
        },
        spec: {
          rules: [{
            host: 'grafana.nnstt1.work',
            http: {
              paths: [{
                backend: {
                  service: {
                    name: $.grafana.service.metadata.name,
                    port: {
                      number: 3000
                    },
                  },
                },
                path: '/',
                pathType: 'Prefix',
              }],
            },
          }],
        },
      },
    },
```

上記を追加してビルドすると、`ingress-prometheus-k8s.yaml` と `ingress-grafana.yaml` が生成されます。

### PVC

Prometheus で取得したメトリクスを永続化するため、Rook Ceph で構築している StorageClass を使った volumeClaimTemplates を Prometheus マニフェストに追加します。

なお、Grafana のデータ永続化は kube-prometheus のポリシー外のようで ([参考 :fa-external-link:](https://github.com/prometheus-operator/kube-prometheus/issues/442){target=_blank})、それに倣って Grafana の volumeClaimTemplates は設定しません。

```jsonnet
    prometheus+:: {
      prometheus+: {
        spec+: {
          externalUrl: 'http://prometheus.nnstt1.work',
          retention: '30d',
          storage: {
            volumeClaimTemplate: {
              apiVersion: 'v1',
              kind: 'PersistentVolumeClaim',
              spec: {
                accessModes: ['ReadWriteOnce'],
                resources: { 
                  requests: { 
                    storage: '10Gi'
                  },
                },
                storageClassName: 'rook-ceph-block',
              },
            },
          },
        },
      },
    },
```


### ServiceMonitor

新規に作成した ServiceMonitor を Prometheus が監視対象とするように Jsonnet を変更します。

具体的には、Prometheus Operator による監視対象を全 Namespace にするよう Jsonnet で定義します。

https://github.com/prometheus-operator/kube-prometheus/blob/main/examples/all-namespaces.jsonnet

```jsonnet
local kp = (import 'kube-prometheus/main.libsonnet') +
           (import 'kube-prometheus/addons/all-namespaces.libsonnet') + {    # 追加
  values+:: {
    common+: {
      namespace: 'monitoring',
    },
    prometheus+: {    # 追加
      namespaces: [],
    },
  },
};

{ ['00namespace-' + name]: kp.kubePrometheus[name] for name in std.objectFields(kp.kubePrometheus) } +
{ ['0prometheus-operator-' + name]: kp.prometheusOperator[name] for name in std.objectFields(kp.prometheusOperator) } +
{ ['node-exporter-' + name]: kp.nodeExporter[name] for name in std.objectFields(kp.nodeExporter) } +
{ ['kube-state-metrics-' + name]: kp.kubeStateMetrics[name] for name in std.objectFields(kp.kubeStateMetrics) } +
{ ['alertmanager-' + name]: kp.alertmanager[name] for name in std.objectFields(kp.alertmanager) } +
{ ['prometheus-' + name]: kp.prometheus[name] for name in std.objectFields(kp.prometheus) } +
{ ['grafana-' + name]: kp.grafana[name] for name in std.objectFields(kp.grafana) }
```