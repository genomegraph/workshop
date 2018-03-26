vg講習会で行われたチュートリアルを日本語に訳しました（[リンク]
(https://github.com/Pfern/PANGenomics/tree/master/exercises/toy)）。
## vg を準備する
vg をmacで動かすために、dockerを利用しています。[genome graph japan](https://github.com/genomegraph/workshop/blob/master/vg_tutorial/vg_tutorial.md)の資料を参考に設定などしました。
このページで使っているdockerイメージはver1.5だったのと、aliasでvgを呼ぶ場合、aliasを設定したディレクトリのみしかアクセスできないため、若干書き換えています（現在の最新版に変更してます）。

```bash
$ alias vg="docker run --rm -i -v $(pwd):/io -w /io quay.io/vgteam/vg:v1.5.0-1674-g8b3f26a8-t108-run vg"
```
↓

```bash:.bashrc
...
function vg (){
	docker run --rm -i -v $(pwd):/io -w /io quay.io/vgteam/vg:v1.6.0-713-g31ea5ed2-t146-run vg $@
}
...
```
### vg をcloneする
まず、任意のディレクトリにvgをクローンします。
```bash
$ git clone https://github.com/vgteam/vg.git
$ mkdir exercise1
$ cd exercise1
$ cp -r ../vg/test/1mb1kgp ./1mb1kgp 
```
### ディレクトリ構成
```bash
$ pwd && tree
/Users/K-lab/genomegraph/exercise1
.
└── 1mb1kgp
    ├── README.md
    ├── z.fa
    ├── z.fa.fai
    ├── z.vcf.gz
    └── z.vcf.gz.tbi

1 directory, 5 files
```
本家のチュートリアルとは<b>違い</b>、1mb1kgpを作業用ディレクトリに丸コピしています。
これで、今回使うデータが揃いました。

## graph の作成
graphを作成するには、入力に`*.fa`（塩基配列のデータ）を与えます。
```bash
$ vg construct -r 1mb1kgp/z.fa > ref.vg
```
graphの統計情報は`vg stats`で参照できます。
```bash
$ vg stats -lz ref.vg 
nodes	1000
edges	999
length	1000000
```
`ref.vg`はノードが1000個、エッジが999個の直線上のグラフで、リファレンスに相当します。
上の例では引数に`-r`しか設定していないため、一本モノのリファレンスグラフが作られます（ノードが1000個、エッジが999本）。
次に、バリアント込みのグラフを作る。
```bash
$ vg construct -r 1mb1kgp/z.fa -v 1mb1kgp/z.vcf.gz > z.vg
$ vg stats -lz z.vg
nodes	84559
edges	115375
length	1029273
```
`z.vcf.gz`にはバリアントの情報が含まれています。そのため、`z.vg`には先ほどのリファレンスグラフにはないバリアントを表すノードも含まれます。

作成したグラフは、`vg view z.vg`でGFA形式で出力でき、他にもJSON（`-j`）やdot形式（`-d`）で出力できます。
```bash
$ vg view ref.vg > ref.gfa
```
dot形式を経由して、グラフをpdf, jpg, png, psで描画できます。`dot`コマンドでpdfを生成するには、[Graphviz](http://www.graphviz.org)をインストールする必要があります。
```
$ vg view ref.vg -d | dot -Tpdf -o ref.pdf
```
`ref.pdf`の画像サイズは、手元の環境では135 KB,2888.94 × 4.8 cmになりました。
同じことをバリアント込みのグラフ（z.vg）でやってみましたが、全然計算が終わりませんでした。


## indexの作成
`vg`はグラフにインデックスを張ることで高速な動作を可能にしています。
> In fact, vg needs two different representations of a graph for read mapping XG (a succinct representation of the graph) and GCSA (a k-mer based index). 

以下のコマンドでインデックスを作成します。
```bash
$ vg index -x z.xg z.vg
$ vg index -g z.gcsa -k 16 z.vg
```
`z.xg`、`z.gcsa`はインデックス情報を格納しています。インデックスを用いて、`vg find`でノードの検索ができます。以下の例では、`z.xg`から、ノードidが2401のノードを見つけ、そこから3ステップで作れるサブグラフを返します。

```bash
$ vg find -n 2401 -x z.xg -c 3（標準出力にバイナリが吐かれる）
```
これをパイプでつなげて、pdf形式で出力してみます。
```bash
$ vg find -n 2401 -x z.xg -c 3 | vg view -dp - | dot -Tpdf -o 2401c3.pdf
$ open 2401c3.pdf
```



## simulation data の作成
`vg sim`でシミュレーション用のリードセットを作れます。
```
$ vg sim -x z.xg -l 100 -n 1000 -e 0.01 -i 0.005 -a >z.sim
```
それぞれのパラメータの意味は以下の通りです。

|||
| ---  | --- |
| `-x z.xg` | シミュレーションの元となるグラフを指定 |
| `-l 100` |シミュレーション配列の長さを指定（100bp）  |
| `-n 1000` | シミュレーション配列の数を指定 （1000本）|
| `-e 0.01` | エラー率を指定（1%） |
| `-i 0.005 ` |インデル率を指定（0.5%）  |
| `-a` |出力をGAM形式にし、元とのアラインメントを出力する  |

ちなみに、`-J`でJSON形式でアラインメントを出力できます。
アラインメントは`$ vg surject -x z.xg -s z.sim`でGAM→SAMと変換できます（GAMはSAM/BAM形式のグラフ版）。`-s`を`-b`とすればBAMにもなります。

## mapping してみる

さきほど作成したシミュレーション配列をmappingします。
```bash
$ vg map -x z.xg -g z.gcsa -G z.sim >z.gam
```
結果はGAM形式で得られます。バイナリのため普通には読めないです。以下のパイプラインでpdfにしてみます。
```bash
$ vg view -a z.gam | head -1 | vg view -JaG - >first_aln.gam
$ vg find -x z.xg -G first_aln.gam | vg view -dA first_aln.gam - | dot -Tpdf -o first_aln.pdf
```
（`vg view -a z.gam`でGAM形式のがJSON形式で出力される。）
アラインメントは、完全マッチは青色、ミスマッチは黄色でノードの上に示されます。
2行目の２つめを、`vg view -dA first_aln.gam -`から`vg view -dSA first_aln.gam -`とすると、デザインが変わります。配列が表示されなくなり、ノードIDだけになります。色の違いはノードごとのアイデンティティに対応してます。

以下のコマンドで、シミュレーション配列のマップ結果を、シミュレーションの元のアラインメントと比較することができます。
```
$ vg map -x z.xg -g z.gcsa -G z.sim --compare -j
```
JSON形式で比較結果が得られます。
以下のパイプラインで、mappingの評価ができます。ここでは`correct`の平均を求めています。
```
$ vg map -x z.xg -g z.gcsa -G z.sim --compare -j | jq .correct | sed s/null/0/ | awk '{i+=$1; n+=1} END {print i/n}'
0.993
```


