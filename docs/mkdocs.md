# MkDocs

静的サイトジェネレータ `MkDocs` の使い方です。

## インストール

```sh
pip install mkdocs mkdocs-material mkdocs-minify-plugin fontawesome-markdown
```

## ビルド

デフォルトでは入力元は `docs` ディレクトリ、出力先は `site` ディレクトリとしてビルドされます。
しかし、GitHub Pages では `docs` ディレクトリのコンテンツを参照してページが生成されるため、
`mkdocs.yml` で入力元と出力先のディレクトリを変更します。

```yaml
# mkdocs.yml
docs_dir: source
site_dir: docs
```

```sh
mkdocs build
```

