# SNMP Exporter

SNMP Exporter の設定方法です。

## generator

https://github.com/prometheus/snmp_exporter/tree/master/generator

Config Generator を使って SNMP Exporter 用の設定ファイル `snmp.yml` を作成します。

generator の利用方法は2通りあります。

1. ソースコードからビルド
1. Docker イメージ

ここでは Docker イメージを利用する方法を記載します。

### Docker Build

公式リポジトリにある Dockerfile からイメージをビルドします。

```Dockerfile
# Dockerfile
FROM golang:latest

RUN apt-get update && \
    apt-get install -y libsnmp-dev p7zip-full && \
    go get github.com/prometheus/snmp_exporter/generator && \
    cd /go/src/github.com/prometheus/snmp_exporter/generator && \
    go get -v . && \
    go install

WORKDIR "/opt"

ENTRYPOINT ["/go/bin/generator"]

ENV MIBDIRS mibs

CMD ["generate"]
```

```sh
git clone https://github.com/prometheus/snmp_exporter.git
cd snmp_exporter/generator
docker build -t snmp-generator .
```

### Docker Run

```sh
cd <work-dir>
docker run -ti \
  -v "${PWD}:/opt/" \
  snmp-generator generate
```