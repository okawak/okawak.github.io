--- 
title: "hugoで新しい記事を作成する"
date: 2023-10-08T12:32:08+09:00
draft: false
disableComments: false
description: "hugoで新しい記事を作成する方法について"
slug: ""
tags: ["hugo"]
categories: []
externalLink: ""
series: []
---

初めて記事を投稿させていただきます。
今回は、hugoを用いて新規記事を作成したい場合の手順についてメモしたいと思います。
今回の内容は以下のバージョンを用いております。
```shell
> hugo version
hugo v0.117.0+extended linux/amd64 BuildDate=unknown
```

## カスタマイズ前
hugoをインストールし、新しいサイトを作るには以下のコマンドを使用します。
```shell
hugo new site SiteName
```

テーマを導入したり種々の設定(hugo.toml)の設定を終えた後、そのまま新しい記事を作成する際は、例えば以下のようにして作成できます。
```shell
hugo new posts/hoge.md
```

このとき作成された./content/posts/hoge.mdの内容は例えば以下の通りです。
```markdown
---
title: "Hoge"
date: 2023-10-08T12:55:16+09:00
draft: true
---
```

この三つの「-」で囲まれた部分は`Front matter`と呼ばれる部分で、この変数を用いてページをカスタマイズすることができるようになっています。
ここで定義される変数の意味については、使っているテーマなどにもよって異なると思いますので、使用するテーマに応じて確認して下さい。
Front matter部分は記事本文には反映されず、この次に書かれることが反映されるようになります。

このデフォルトで作成されるファイルは、./archetypes/default.mdによって作成されています。
```markdown
---
title: "{{ replace .Name "-" " " | title }}"
date: {{ .Date }}
draft: true
---
```

ここに現れている.Nameや.Dateなどはhugoでアクセスできる変数で、詳細は[公式ページ](https://gohugo.io/variables/)を確認して下さい。
つまり、このテンプレートファイルをカスタマイズすることで、自動で記事に必要な情報を加えることができるということになります。



## テンプレートファイルの作成

./archetypes/default.mdを変更すれば、全ての記事の初期設定を決めることができますが、作成するページに応じて設定したい内容が異なることがほとんどだと思います。
例えば、ディレクトリを分けて記事(./content/)を書き、posts/以下にブログ、about/以下にプロファイルを記載するといった用途が考えられます。(こちらのサイトもそうなっています。)

そこで、postsディレクトリ以下に作成する記事には、ある特定のテンプレートファイルを使いたいと考えるようになるはずです。
hugoにはデフォルトでそのような機能がついており、`./archetypes/ディレクトリ名.md`を作成することで、このテンプレートファイルが初期設定として使われるようになります。

例として、./archetypes/posts.mdを作成して、新たな記事./content/posts/huga.mdを作成する例を見てみたいと思います。
テンプレートファイルposts.mdを以下のように作成します。
```markdown
---
title: "{{ replace .Name "-" " " | title }}"
date: {{ .Date }}
draft: true
tags: []
---

## 初めに

## 内容

## まとめ
```

これを作成した後に、今までと同じようにhugoコマンドで記事を作成すると、
```shell
hugo new posts/huga.md
```
```markdown
---
title: "Huga"
date: 2023-10-08T13:25:13+09:00
draft: true
tags: []
---

## 初めに

## 内容

## まとめ
```
という内容になっているはずです。
使っているテーマやhugoの変数を用いて、色々テンプレートをカスタマイズすることで、より記事が書きやすくなると思います。


## まとめ
以上のようにテンプレートファイルを作成することで、必要なFront matterの設定を忘れることなく記事を書くことができるようになります。

一度テンプレートファイルを作ってしまえば、
```shell
hugo new posts/hoge.md
```
で大まかな骨格が作られたファイルが作成できます。
この記事で採用しているcoderのテーマでのカスタマイズに関しては後日書こうと思っております。


## 参考文献

- [Hugoで新規記事を作成するときのTips的なメモ](https://qiita.com/n0bisuke/items/4701481c3bca4df81b0b)



