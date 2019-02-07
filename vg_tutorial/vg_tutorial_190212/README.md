# ハンズオン形式のゲノムグラフ入門

この資料は、過去にやった[ゲノムグラフの入門講義](https://drive.google.com/file/d/1R5Ub5MKmmfUI4pf_H5R2a_I8p2jQz84b/view)と[vgチュートリアル](https://github.com/genomegraph/workshop/blob/master/vg_tutorial/vg_tutorial.md)の中から、とっかかりに必要そうな最小限の内容をピックアップしたものです。



目標は

* ゲノムグラフのイメージを掴む
* ゲノムグラフはどんなファイルに保存されているのか
* グラフを作りかたの一例を学ぶ
* 可視化してグラフになっていることを確かめる
  * かつ、可視化ツールごとインプットの形式を知る

の4点を知ることです。







## 0. 準備

ここは当日は扱わないので、各自準備をお願いします。



### vgのインストール

#### Linuxのヒト

ビルド済みのstatic binaryが用意されています。[ここ](https://github.com/vgteam/vg/releases/tag/v1.13.0)から vg という名前のファイルをダウンロードして、

```bash
chmod +x vg
```

で実行権限をつけてください。必要に応じてパスを通してください。



#### macOSのヒト

static binaryがないので、自分でビルドする必要があります。ビルドする時間がない場合はDockerを使います。

* ビルドの仕方
  * [ここ](https://github.com/vgteam/vg/releases/tag/v1.13.0)から `vg-v1.13.0.tar.gz` という名前のファイルをダウンロードします。
  * 解凍する
  * [Building on MacOS](https://github.com/vgteam/vg#building-on-macos) に従って、必要なライブラリを入れて、環境変数を設定する
  * `. ./source_me.sh && make`
  * `./bin/vg` で動くか確認する

* Dockerを使う場合
  * Docker for mac を[ここ](https://docs.docker.com/docker-for-mac/install/)からインストールする
  * Dockerを立ち上げる
  * `quay.io/vgteam/vg:v1.13.0` をpullする



### Bandageのインストール

自分のOSに合ったものを https://rrwick.github.io/Bandage/  からダウンロードします。



### テストデータ

```
git clone hoge
```





## 1. ゲノムグラフとは？

ゲノム配列をグラフ理論的なデータ構造で表現したものです。



![ゲノムグラフ](figure/ex_genome_graph.pdf)





ゲノムではありませんが、文字列をグラフ（フローチャート）で表現したという点では[こういう](http://loveallthis.tumblr.com/post/166124704)のもあります。



## 2. どんなファイルフォーマットが使われるの？

ゲノムグラフの表現には、[GFA形式](https://github.com/GFA-spec/GFA-spec)や[FASTG形式](http://fastg.sourceforge.net) が使われることが比較的多いという印象です。ただ、ゲノム配列を記述するFASTA 形式のように広く浸透しているものはなく、ソフトウェアごとに独自にフォーマットが定義されていることも珍しくはないです。ここでは使用ツールとして vg, Bandageだけにフォーカスして知っておくと良いフォーマットを列挙します。



* VG形式
* GFA形式
* FASTG形式
* XG形式
* GCSA形式
* GAM形式





## 3. ハンズオン

### データの説明

https://www.nature.com/articles/ncomms11939



Figure4を扱う





### グラフ構築

ゲノムグラフの構築方法はいくつかありますが、ここでは`vg` が提供している方法として、 `msga` を扱ってみます。



```
vg msga -f data/FL-utilization.fna -a -P 0.95 -N > graph.vg  # ローカルアラインメントを繰り返して差分情報をグラフにしていく
```



### 可視化

#### Bandage

```
vg view graph.vg > graph.gfa  # GFA形式に吐く。Bandageでみることができる。
```



#### SequenceTubeMap

```
vg view -j graph.vg > graph.json # JSON形式に吐く。これはsequenceTubeMapの可視化に用いる。
```



#### （Graphviz）

```
vg view -d graph.vg | dot -Tpdf -o graph.pdf
```

