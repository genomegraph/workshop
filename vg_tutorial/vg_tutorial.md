#  vgを使ってみる

vgはゲノムグラフを扱うツールです。グラフ構築、マッピング、変異検出など種々の操作を行うことができます。  

ここでは、サンプルデータを用いて、オーソドックスな使い方を学んでいきます。



## 準備

まずは、このドキュメントとサンプルデータを落としてきます。

```bash
$ git clone https://github.com/genomegraph/workshop.git
```

サンプルデータは、

```bash
$ cd vg_tutorial/small
$ ls
```

で確認できます。今回はこのディレクトリで動かします。



次にvgを使えるようにします。直接インストールするのは時間がかかるので、dockerを使います。

dockerが入っていない場合は、

-   Mac: https://www.docker.com/docker-mac
-   Linux: https://docs.docker.com/engine/installation/

から、最新版をインストールしてください(Macの場合は、homebrewを使うより、直接URLからインストールすることをおすすめします)。



インストールが終わったら、ターミナルを起動して、

```bash
alias vg="docker run --rm -i -v $(pwd):/io -w /io quay.io/vgteam/vg:v1.5.0-1674-g8b3f26a8-t108-run vg"
```

を`~/.bashrc`や`~/.zshrc`に書いてください。そのあと、

```bash
$ vg
```

と打つと、dockerイメージがない場合は、自動的に引っ張ってきてくれます。

```
vg: variation graph tool, version v1.5.0-1674-g8b3f26a

usage: vg <command> [options]

commands:
  -- add           add variants from a VCF to a graph
  -- align         local alignment
  -- annotate      annotate alignments with graphs and graphs with alignments
(中略)
  -- vectorize     transform alignments to simple ML-compatible vectors
  -- version       version information
  -- view          format conversions for graphs and alignments
  -- xg            manipulate xg files
```

と表示されれば準備完了です。



## vgを動かす

### リファレンスゲノムグラフの構築

まずは、ゲノムグラフを構築します。構築の仕方は大きく分けて3通りあります。

-   リファレンスゲノム配列に対する変異イベントをVCF形式で記述し、`vg construct`で変換する
-   `vg msga`を用いて、複数の配列を順番にアラインメントしながら、グラフにまとめていく
-   自分で直接グラフをGFA形式やJSON形式で書き、それを`vg view`で変換する

様々な方法がありますが、`.vg`というバイナリファイルにすることは同じです。  



今回は、`vg construct`で構築します。

```bash
$ vg construct -r x.fa -v x.vcf.gz > graph.vg
```



### グラフの可視化

内容をテキスト形式で確認するには、`vg view`を用います。

```bash
$ vg view graph.vg  # GFA形式で標準出力する
$ vg view -j graph.vg  # JSON形式で標準出力する
$ vg view -d graph.vg  # dot形式で標準出力する
```



既存の[グラフゲノムブラウザ](http://graphgenome.tk/demo3/)で可視化する場合は、`-j`で出力した結果をコピペします。





### マッピング

ショートリードをリファレンスゲノムグラフにマップします。

```bash
$ vg index -x graph.xg -g graph.gcsa -k 16 graph.vg  # 2種類のindexファイルを作成する
$ vg map -x graph.xg -g graph.gcsa -f x.fa_1.fastq -f x.fa_2.fastq > mapped.gam
```

マッピングの結果は、`.gam`というバイナリファイルに出力されます。



注：現在は`vg map`よりも高速にマッピングできる`vg mpmap`が実装されつつあります。今後は`vg map`の代わりに`vg mpmap`を使うことが主流になっていくのかもしれません。



### GAMファイルの取り扱い

GAMファイルは、SAM/BAMファイルのグラフバージョンです。`vg`はgamファイルのハンドリング機能も備えています。



中身は、`vg view`で見ることができます。

```bash
$ vg view -a mapped.gam  # JSON形式で標準出力
```

`.vg`のJSON形式と基本的には同じですが、それに加えてマッピングのクオリティやスコアなどアラインメントの情報も含まれています。



線形配列用のSAM/BAM形式に変換することもできます。

```bash
$ vg surject -x graph.xg -s mapped.gam > mapped.sam  # -sの代わりに-bを指定するとBAMで出力
```





### pile upする

カバレッジの計算など下流の解析を自分でGAM/JSONをパースして行うこともできますが、vgにあるpile up機能を使うこともできます。

```bash
$ vg index -d mapped.gam.index -N mapped.gam  # gamファイル用のインデックスの作成
$ vg pileup -j graph.vg mapped.gam > pileup.json  # JSON形式で出力。あとは自分で処理
```



得られた`pileup.json`はリファレンス側の1塩基ずつマッピング情報を出力してくれています。

ちなみに、このあとノードごとのカバレッジを出すには、ノードごとに`num_bases`の数を数えればよいです。



### 変異検出

最後に変異検出を行います。`vg index`で作った`mapped.gam.index`があることを用います。

```bash
$ vg genotype -v graph.vg mapped.gam.index > calls.vcf
```



### おまけ

GFAやJSON形式で直書きしたグラフを`.vg`に変換する

```bash
$ vg view -Fv graph.gfa > graph.vg
$ vg view -Jv graph.json > graph.vg
```



グラフの統計情報を調べる

```bash
$ vg stats -lz graph.vg  # ノード数、エッジ数、全塩基数を出力
```

