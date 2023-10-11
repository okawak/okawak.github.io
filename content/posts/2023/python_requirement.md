--- 
title: "pipやrequirements.txtについての備忘録"
date: 2023-10-11T13:48:38+09:00
draft: false
disableComments: false
description: ""
slug: ""
tags: ["python", "venv", "pip", "requirements.txt"]
categories: []
externalLink: ""
series: []
---

python環境をvenvを使って作成する際、必要なパッケージなどをrequirements.txtで管理するようにしている。

githubにソースを置いておくと時々セキュリティ面からパッケージをアップグレードするようにと連絡がくるため、
必要なコマンド等を忘れないようにメモしておく。

随時メモしておくべきことがあれば更新します。

## 基本的な使い方

### pipでパッケージを一括でインストールする
```shell
pip install -r requirements.txt
```

### 現在の環境をrequirements.txtに書き出す
```shell
pip freeze > requirements.txt
```

リダイレクトの設定が上書き禁止になっている場合
（「noclobber」がonになっている場合。この設定は``set -o | grep noclobber``などで確認できる）、
このコマンドでは新たなファイルを作成することはできない。
強制的に上書きするには次のようにする。
```shell
pip freeze >| requirements.txt
```

## パッケージのアップグレード

### pipで一つずつアップグレード
```shell
pip install -U PackageName
# or
pip install --upgrade PackageName
```

### awkを用いて一括アップグレード

まず、pipで更新可能なパッケージを確認するには、
```shell
pip list -o
```
で確認できる。この出力を用いて一括アップデートするコマンドは以下の通りである。

```shell
pip list -o | tail -n +3 | awk '{ print $1 }' | xargs pip install -U
```
これは、``pip list -o``で出力されるパッケージについて、``tail -n +3``でパッケージ部分のみを取り出して（3行目から表示させるという意味）、
``awk '{print $1}'``で一列目を取り出し（パッケージの名前）、これらを``pip install -U``するという処理になっている。


### pip-reviewを用いて一括アップグレード

まず、pip-reviewをインストールする。
```shell
pip install pip-review
```
一括でアップグレードするコマンドが用意されており、以下のコマンドでできてしまう。
```shell
pip-review --auto
```
``pip-review``単体のコマンドを打てば、更新があるパッケージを一覧表示してくれる。

また、インタラクティブに更新を行いたい場合は次のようにする。
```shell
pip-review --interactive
```


## まとめ
pip-reviewを使えば簡単に一括アップグレードできる。
ただし、依存関係が重要な場合、一斉にアップグレードすると動かなくなる可能性もあるので注意。

最後にpip freezeでパッケージをまとめておく。
```shell
# python環境に入る
pip-review --auto
pip freeze >| requirements.txt
```
