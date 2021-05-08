# docs

作業メモを記録して GitHub Pages で公開するためのリポジトリです。

## URL

https://nnstt1.github.io/

## 使用方法

### ビルド & デプロイ

本リポジトリを更新すると、Github Actions で mkdocs のビルドが実行されます。

ビルドによって作成された資材は専用のリポジトリ (https://github.com/nnstt1/nnstt1.github.io) に格納されます。


### 検証

コミット前に mkdocs のページを検証する手順です。

```bash
$ MKDOCS_PORT=8000

$ sudo firewall-cmd --add-port=$MKDOCS_PORT/tcp --permanent
$ sudo firewall-cmd --reload

$ source .venv/bin/activate
$ mkdocs serve -a 0.0.0.0:$MKDOCS_PORT
```


## GitHub Actions の設定

GitHub Actions で作成した資材を別リポジトリに格納するために、以下の設定をおこなっています。

https://github.com/peaceiris/actions-gh-pages#%EF%B8%8F-create-ssh-deploy-key


### キーペアの作成

```bash
$ ssh-keygen -t rsa -b 4096 -C "$(git config user.email)" -f gh-pages -N ""
```

### 公開鍵の登録

GitHub Pages で公開するリポジトリに公開鍵を設定します。

### 秘密鍵の登録

本リポジトリに秘密鍵を登録します。