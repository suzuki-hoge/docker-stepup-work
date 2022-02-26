---
title: "２部: ストレージとは"
---

コンテナ内のファイルをホストマシンと共有するために、ストレージについて理解します。

# このページで初登場するコマンドとオプション
## ボリュームを作成する
```:コマンド
$ docker volume create [option]
```

### オプション
オプション | 意味 | 用途  
:-- | :-- | :--
`--name`   | ボリューム名を指定 | ID ではなく名前で扱えるようにする

## ボリュームを調べる
```:コマンド
$ docker volume inspect [option]
```

### オプション
このページで新たに使うオプションはなし

## コンテナを起動する
```:新コマンド
$ docker container run [option] <image> [command]
```

```:旧コマンド
$ docker run [option] <image> [command]
```

### オプション
オプション | 意味 | 用途  
:-- | :-- | :--
`-v, --volume`   | マウントする | 短く指定する
`--mount`   | マウントする   | 丁寧に指定する

# ストレージとは
todo で確認した通り、コンテナ内で作成・変更されたファイルはコンテナの終了とともに全てリセットされます。またホストマシンのファイルシステムとコンテナのファイルシステムは隔離されています。
このままではコンテナを一度終了するとコンテナ内にあったログやデータを次起動するコンテナに引き継げませんし、コンテナ内のプログラムをホストマシンで直接編集することもできません。

それらの課題を解決するために、ファイルをホストマシン常に保存するいくつかの方法のうち 2 つを説明します。

# ボリューム
## ボリュームとは
ボリュームはコンテナ内のファイルをホストマシン上で Docker が管理してくれる仕組みです。
「どこにどういう形式で保存されているかにあまり関心はなく、とにかくデータを永続化・共有したい」という場合に有用で、たとえば DB のデータの永続化やコンテナ間のファイル連携などに活用できます。

todo e

## ボリュームの作成とコンテナへのマウント
ボリュームを扱うには、まずボリュームを作成します。

```:コマンド
$ docker volume create [option]
```

`[option]` は未指定でも良いですが、人間が扱いやすいように `--name` で名前をつけると良いでしょう。今回は `sample-volume` とします。

以上を踏まえて、次のようにボリュームを作成します。

```:Host Machine
$ docker volume create --name sample-volume
```

次にコンテナ起動時に作成したボリュームをマウントします。

マウントするには `--volume` オプションと `--mount` オプションがありますが、細かな点を除いてできることにほぼ違いはありません。
この Book では両方のコマンドを掲載しますが、不慣れな場合は長いけどわかりやすくなる `--mount` の方で理解を深めることをおすすめします。

まずは先に全ての設定を定められた順番で `:` 区切りで列挙する `--volume` オプションを使ってコンテナを起動します。

`--volume` の指定方法は 1 つめがボリューム名で、2 つめがマウント先で、3 つめがオプションです。
今回は先ほど作成した `sample-volume` を `/my-volume` にオプションなしでマウントするので、コンテナ起動のコマンドは次のようになります。

```:Host Machine
$ docker container run              \
  --name ubuntu                     \
  --rm                              \
  --interactive                     \
  --tty                             \
  --volume sample-volume:/my-volume \
  ubuntu:20.04                      \
  bash

# cd /my-volume
```

`/my-volume` というディレクトリがコンテナ内に存在することを確認できます。

ディレクトリ内にファイルを書き残し、コンテナを ( `bash` の終了によって ) 終了します。

```:Container
# cd /my-volume

# echo 'Hello Volume.' > hello.txt

# exit
```

次は `key=val` 形式で指定する `--mount` オプションを使い、同じボリュームを同じディレクトリにマウントしてコンテナを起動します。

`--mount` の指定方法は `type` と `source` と `destination` のほか、`readonly` などのオプションを任意の順番で指定します。
`source` は `src` など、`destination` は `dst` や `target` などの略記も存在します。
今回はボリュームをマウントするので `type=volume` で、残りは `src` と `dst` を使って指定します。

```:Host Machine
$ docker container run                                 \
  --name ubuntu                                        \
  --rm                                                 \
  --interactive                                        \
  --tty                                                \
  --mount type=volume,src=sample-volume,dst=/my-volume \
  ubuntu:20.04                                         \
  bash

# cat /my-volume/hello.txt
Hello Volume.
```

コンテナを跨いで `hello.txt` が存在することを確認できます。

## ボリュームの実態
作成したボリュームは `docker volume inspect` で詳細を把握することができます。

```:Host Machine
$ docker volume inspect sample-volume
[
    {
        "CreatedAt": "2022-02-06T23:07:52Z",
        "Driver": "local",
        "Labels": {},
        "Mountpoint": "/var/lib/docker/volumes/sample-volume/_data",
        "Name": "sample-volume",
        "Options": {},
        "Scope": "local"
    }
]
```

ここの `Mountpoint` がデータのある場所の実体なのですが、これは Docker Engine の VM 上のパスのことなのでホストマシンで `ls` しても何も確認できません。

ボリュームは Docker が管理しているので単純に読み書きすることはできない、ということだけ理解しておきましょう。

## ボリュームの活用例

# バインドマウントとは
バインドマウントはホストマシンの任意のディレクトリをコンテナにマウントする仕組みです。
「ホストマシンとコンテナ双方がファイルの変更に関心がある」という場合に有用で、たとえばソースコードの共有などに活用できます。

## コンテナへのマウント
バインドマウントはホストマシンのファイルシステムをそのままマウントするので、事前にボリュームの作成のような作業は必要ありません。

バインドマウントも `--volume` オプションと `--mount` オプションで可能で、できることやメリットデメリットも同じです。

まずは `--volume` オプションを使ってコンテナを起動します。
1 つめをボリューム名ではなく絶対パスにすることでバインドマウントが行われるため、`pwd` を使ってカレントディレクトリをマウントします。

```:Host Machine
$ docker container run     \
  --name ubuntu            \
  --rm                     \
  --interactive            \
  --tty                    \
  --volume $(pwd):/my-bind \
  ubuntu:20.04             \
  bash

# cd /my-bind
```

コンテナ内でファイルを作成します。

```:Container
# echo 'Hello Bind.' > bind.txt
```

ホストマシン上でマウントしたディレクトリを確認すると、`bind.txt` があることを確認できます。

```:Host Machine
$ ls

bind.txt
```

ホストマシンで `bind.txt` に追記をします。

```:Host Machine
$ echo 'from Host Machine.' >> bind.txt
```

コンテナで確認できます。

```:Container
# cat bind.txt
Hello Bind.
from Host Machine.
```

コンテナを ( `bash` の終了によって ) 終了します。

```:Container
# exit
```

最後に `--mount` オプションを使ってコンテナ起動をする方法を確認します。
今回はバインドマウントをするので `type=bind` で、残りはボリュームマウントの時と同じく `src` と `dst` を使って指定します。

```:Host Machine
$ docker container run                      \
  --name ubuntu                             \
  --rm                                      \
  --interactive                             \
  --tty                                     \
  --mount type=bind,src=$(pwd),dst=/my-bind \
  ubuntu:20.04                              \
  bash

# cat /my-bind/bind.txt
Hello Bind.
from Host Machine.
```

## バインドマウントの活用例

# まとめ
## ボリュームとバインドマウントの比較
ボリュームは次のような特徴があります。

- `docker volume create` で作成したボリュームをコンテナにマウントする
- ボリュームの実体はホストマシンのどこかであり Docker によって管理される
- ホストマシンからのボリュームの読み書きは行わない
- コンテナ内でどのような操作をしても、影響は Docker の管理するボリュームの範囲内にとどまる

これらの特徴を活かして、次のように活用できます。

- ログや DB のデータをとにかく残す
- 複数のコンテナでファイル連携をする

todo e

バインドマウントは次のような特徴があります。

- ホストマシンのファイルやディレクトリをコンテナにマウントする
- マウントするファイルやディレクトリは Docker に管理されない
- ホストマシンで Docker 以外のプロセスでファイルやディレクトリを変更して良い
- コンテナ内でのファイル削除などがホストマシンに影響する

これらの特徴を活かして、次のように活用できます。

- ホストマシンとコンテナでソースコードを共有する
- ホストマシンの設定ファイルをコンテナに共有する
  - 特に認証情報を含むものやチューニングなど頻繁に変更するものなど、Dockerfile の `COPY` であらかじめイメージに含めづらいもの

ただし Docker でも Git でも管理していないファイルなどが破壊される可能性があることを理解しておく必要があります。

todo e

## --volume と --mount の比較
`--volue` オプションは次のような特徴があります。

- 指定する順番が定まっている
- 短く書けるが知らないと読めない
- 1 つめの書き方によってボリュームかバインドマウントか決まる
  - ボリューム名の場合はボリューム
  - 絶対パスの場合はバインドマウント

`--mount` オプションは次のような特徴があります。

- 順番は自由
- 長くなるが読みやすい