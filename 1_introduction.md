
class: center, middle

# 分散データベース勉強会

安田裕介

---

class: center, middle

# 僕と分散データベース

これらの本読んだぐらい。まだ詳しくはない。勉強中。

![Hadoop: The Definitive Guide, 4th Edition](http://akamaicovers.oreilly.com/images/0636920033448/cat.gif)
![Cassandra: The Definitive Guide, 2nd Edition](http://akamaicovers.oreilly.com/images/0636920043041/rc_cat.gif)
![HBase: The Definitive Guide, 2nd Edition](http://akamaicovers.oreilly.com/images/0636920033943/rc_cat.gif)
![ZooKeeperによる分散システム管理](http://www.oreilly.co.jp/books/images/picture978-4-87311-693-8.gif)

---

class: center, middle

# この発表の目的

分散データベースであるHDFS, HBase, Cassandra, ZooKeeperをいくつかのキーワードとともにさらっと紹介し、今後の勉強会で掘り下げるテーマを探すきっかけをつくる

---

# 歴史

- 2003 Googleが[GFS論文](http://research.google.com/archive/gfs.html)を公開
- 2004 Googleが[MapReduce論文](http://research.google.com/archive/mapreduce.html)を公開
- 2006 Hadoopの登場
- 2006 Googleが[Bigtable論文](http://research.google.com/archive/bigtable.html)を公開
- 2007 Amazonが[Dynamo論文](http://www.allthingsdistributed.com/2007/10/amazons_dynamo.html)を公開
- 2007 HBaseの登場
- 2007 Cassandraの登場

---

# 見ていく項目

- どうして作られたのか
- CAPの定理に当てはめるとCP?, AP?
- アーキテクチャー
- サポートされているクエリー
- Write Path, Read Path
- データモデリング
- データの内部表現

---

class: center, middle

# HDFS

---

class: center, middle

# HDFSって何？

Hadoop Distributed File System

Hadoopプロジェクトの分散ファイルシステム

---

# GFS, HDFSが生まれた理由

加速度的に増え続けるWebインデックスを速く、安全に、高可用性かつスケーラブル扱える分散ファイルシステムをGoogleやYahooが必要としていたから

---

# CAPの定理に当てはめると

CP: 可用性よりも整合性を選んだ

---

# デザイン

## 得意なこと

- 大きなファイル: 数百メガ、ギガ、テラバイトクラスのファイルを扱うよう最適化
- ストリームによるデータアクセス: write-once, read-mannyなデータを想定。解析などに向く。
- コモディティーなハードウェア: 特殊なハードウェアを必要とせず、ありふれたマシンでクラスターを組む。壊れること前提。


.footnote[[HDG4](http://shop.oreilly.com/product/0636920033448.do) Ch3. The Design of HDFS]


---

# デザイン

## 苦手なこと

- 低レイテンシなデータアクセス: 数10msオーダーのアクセスは無理。レイテンシよりスループットを重視。
- 大量の小さなファイル: 1台のnamenodeがすべてのファイルのメタデータをメモリー上に持つので、多くのファイルは扱えない。100万オーダーはいけるが10億オーダーは無理。

.footnote[[HDG4](http://shop.oreilly.com/product/0636920033448.do) Ch3. The Design of HDFS]

---

# アーキテクチャー

![HDFS Architecture](http://itm-vm.shidler.hawaii.edu/HDFS/BlockReplication.gif)

---

# サポートされているオペレーション

---

# Write Path

![HDFS Write Path](http://cfs22.simplicdn.net/ice9/free_resources_article_thumb/Cracking_Hadoop_Dev_Interview_63.jpg)

.footnote[[HDG4](http://shop.oreilly.com/product/0636920033448.do) Ch3. Data Flow Anatomy of a File Write]

???
1. DistributedFileSystem#create()を呼ぶ
2. namenodenにRPCしてブロックが割り与えられていないファイルシステムの名前空間にファイルを作る. namenodeはファイルが存在しないことを確認し権限をチェックする。DistributedFileSystemはFSDataOutputStreamを返す.
3. データはデータキューに書かれ、DataStreamerがそれを消費し
4. replication level分の一連のdataノードのはパイプラインを構築し、パケットを保存しては次のdatanodeにフォワードする。
5. DFSOutputStreamはackキューも持っており、datanodeからackが返ってきたらパケットをデキューする。最小レプリカ数(デフォルトは1)を満たせたら書き込み成功。残り(デフォルトは3)は非同期にレプリケーションされる。
6. 書き込みが終了したらclose()を呼び、パケットをフラッシュする。
7. namenodeに完了を通知

---

# Read Path

![HDFS Read Path](http://songcy.net/post-attachments/1/data-reading-in-HDFS.jpg)

.footnote[[HDG4](http://shop.oreilly.com/product/0636920033448.do) Ch3. Data Flow Anatomy of a File Read]

???
1. DistributedFileSystem#open()を呼びファイルを開く
2. namenodeにRPCし最初の数ブロックの位置を取得. namenodeはブロックごとにdatanodeのアドレスを返す
3. DistributedFileSystemが返したDFSInputStreamのread()を呼び、最初のブロックのdatanodeに接続
4. read()を繰り返し呼び、datanodeからデータを読む
5. ブロックの終了位置に来たらDFSInputStreamはdatanodeとコネクションを切り次のブロックを別のdatanodeから読む
6. 終わったらFSDataInputStreamのclose()を呼ぶ

---

# さらに掘り下げる項目

- 大きなデータに最適化されているとはどういうことか？
- 書き込みが失敗した場合の対処
- ファイルシステムの整合性モデル
- namenodeの冗長性

---

class: center, middle

# HBase

---

# HBaseって何？

分散ファイルシステム上で動くHadoopプロジェクトのBigTableクローン

---

# HBase, BigTableが生まれた理由

GFSやHDFSでは大量の小さなファイルを扱うことやランダムアクセスに弱かった。GFSやHDFS上のデータ処理に使っていたMapReduceではリアルタイムにデータを処理することができなかった。これらをするには分散ファイルシステム上にストレージエンジンを構築する必要があった。

---

# CAPの定理に当てはめると

CP: 可用性よりも整合性を選んだ

---

# データモデル

用語
- namespace: tableの集合
- table: rowの集合
- row: row keyで一意に識別されるcolumnの集合。row keyに基づき辞書順にソートされている。
- column family: columnのグループ。テーブルを作るときに指定する。HFileの保存単位。大抵tableに１つ。
- column: 値。 *family:qualifier*で参照できる。
- column qualifier: columnの名前。column family中のqualifierの数に上限はない。
- version: 同一columnは複数のバージョンをもつことができる
- cell: あるバージョンのcolumnの値
- tag: cellが持てるメタデータ

.footnote[[HBDG2](http://shop.oreilly.com/product/0636920033943.do) Ch1. Namespaces, Tables, Rows, Columns, and Cells]

---

# アーキテクチャー

![HBase Architecture](https://www.safaribooksonline.com/library/view/hbase-the-definitive/9781449314682/httpatomoreillycomsourceoreillyimages889242.png)

.footnote[[HBDG2](http://shop.oreilly.com/product/0636920033943.do) Ch1. Building Blocks]

---

# 自動シャーディング

- スケールとロードバランシングの単位はregion
- regionは連続した一連のrow
- region serverが複数のregionを担当する
- regionが一定以上の大きさになると２つに分割される( *splitting*)
- splittingはコンパクションとは反対の動きをする

.footnote[
[HBDG2](http://shop.oreilly.com/product/0636920033943.do) Ch1. Auto-Sharding
[AHA](http://shop.oreilly.com/product/0636920035688.do) Ch2. Splits (Auto-Sharding)
] 

---

# ロードバランシング

- region serverが新しく参加したり落ちたり、regionが分割されたりすると、regionの分布に偏りが出る
- master serverはregionの分布が均一になるように一定間隔でロードバランサーを実行する
- regionの移動中は一時的にデータが見えなくなる

---

# コンパクション

- memstoreがいっぱいになりディスクにフラッシュすると、小さなファイルがHDFS上にたくさんできてしまう
- **コンパクション**は定期的に小さなファイルを１つのファイルにまとめ上げる
- コンパクションには **minor compaction**と **major compaction**がある
- minor compactionは一部のHFileのみを対象にする。TTLや最大バージョン数を超えたcellやdelete markerがついたcellもここで削除する（delete markerは消さない）。
- major compactionはすべてのHFileを対象にする。minor compactionの動作に加えdelete markerも削除する。

.footnote[
[AHA](http://shop.oreilly.com/product/0636920035688.do) Ch2. Compaction
] 

---

# サポートされているオペレーション

---

class: center, middle

# Cassandra

---

# Cassandraって何？

class: center, middle

Cassandraはオープンソースの非集中型row-oriented分散データベース

---

# Cassandra, Dynamoが生まれた理由

- DynamoはAmazonのサービスの状態を高い可用性を保ち管理するために考えられた技術要素。Amazonにおいて可用性は売上に直結する非常に重要な要素で、特に書き込みの可用性は例えばショッピングカートなどのサービスにおいて重視されている。製品としてはS3やDynamoDBで生かされている。
- CassandraはFacebookのInbox検索のために作られた。メッセージのコピー、メッセージの逆引きインデックス、多くのランダムRead、多くの同時ランダムWriteを伴う大量のデータを扱う必要があった。


.footnote[
[Dynamo: Amazon’s Highly Available Key-value Store](http://www.allthingsdistributed.com/2007/10/amazons_dynamo.html)

[Cassandra - A Decentralized Structured Storage System](http://docs.datastax.com/en/articles/cassandra/cassandrathenandnow.html)

[CDG2](http://shop.oreilly.com/product/0636920043041.do) Ch2. Where Did Cassandra Come From?
]

---

# CAPの定理に当てはめると

AP: 整合性よりも可用性を選んだ

---

class: center, middle

# アーキテクチャー

---

# Gossipプロトコルと Failure Detection

- Cassandraにはmasterがなく、すべてのノードが同じ役割を果たす
- ノード同士の情報は **Gossipプロトコル**で同期する
- **Phi Accrual Failure Detector**にもとづきノードが生きているか死んでいるか判断する

.footnote[
[CDG2](http://shop.oreilly.com/product/0636920043041.do) Ch6. Gossip and Failure Detection

[Phi Accrual Failure Detector](http://www.jaist.ac.jp/~defago/files/pdf/IS_RR_2004_010.pdf)
]

---

# Snitch

- クラスター中のノードのネットワーク上の近さを知る仕組みを **snitch**という
- snitchhはどのノードから読み込み、書き込みをすべきかをきめるために使われる
- あらかじめネットワークトポロジーを教えずに近さを判断するdynamic snitchもある
- dynamic snitchのために修正したPhi failure detectionを使っている

.footnote[
[CDG2](http://shop.oreilly.com/product/0636920043041.do) Ch6. Snitches, Ch7. Snitches
]

---

# リングとトークン

<img src="http://g33ktalk.com/wp-content/uploads/2014/04/Screen-Shot-2014-04-30-at-10.41.53.png" width="320px">

- クラスターはデータを **リング**として管理する
- **トークン**はデータのリング中の位置を決定する
- ノードはリングに属し、トークンに指定されたデータ集合を割り当てられる
- トークンはpartition keyからハッシュ関数で計算される

.footnote[
[Node of the Rings: Fellowship of the Clusters (Or: Understanding How Cassandra Stores Data)](http://www.planetcassandra.org/blog/node-of-the-rings-fellowship-of-the-clusters-or-understanding-how-cassandra-stores-data/)

[CDG2](http://shop.oreilly.com/product/0636920043041.do) Ch6. Rings and Tokens
]

---

# Virtual Node

- 実際には１つのトークンをノードに静的に割与えはしない
- トークンを小さな領域に分割し、複数のトークンを物理ノードに割与える
- 分割されたトークン領域を **virtual node**という

.footnote[
[CDG2](http://shop.oreilly.com/product/0636920043041.do) Ch6. Virtual Nodes
]

---

# Partitioner

- **partitioner**はデータをどのようにクラスター中のノードに分散するか決定する
- partitionerはpartition keyからトークンを計算するハッシュ関数

.footnote[
[CDG2](http://shop.oreilly.com/product/0636920043041.do) Ch6. Partitioners
]

---

# レプリケーションと整合性レベル



.footnote[
[CDG2](http://shop.oreilly.com/product/0636920043041.do) Ch6. Replication Strategies, Ch6. Consistency Levels
]


---

# クエリーとコーディネーターノード

- クライアントはクエリーの際にクラスターのどのノードと接続してもよい
- 接続されたノードはコーディネーターノードとなる
- コーディネーターノードはどのノードが求められたデータのレプリカか判定し、クエリーをフォワードする
- 書き込み時は整合性レベルとレプリケーションファクターに基づきすべてのノードに書き込みを行う
- 読み込み時は整合性レベルを満たせる数のレプリカにアクセスする

.footnote[
[CDG2](http://shop.oreilly.com/product/0636920043041.do) Ch6. Queries and Coordinator Nodes
]

---

# Hinted Handoff

- 書き込み時にレプリカノードが（ネットワーク分断などで）利用できない時がある
- コーディネーターノードは書き込めなかったときにヒントを残す
- レプリカノードが復活した時はヒントをもとに書き込みを回復させる
- この機能を **hinted handoff**という

.footnote[
[CDG2](http://shop.oreilly.com/product/0636920043041.do) Ch6. Hinted Handoff
]

---

# Anti-Entropyプロトコルとリペア

- Cassandraはレプリケーションされたデータ間の不整合を修復するために **anti-entropyプロトコル**を使う
- anti-entropyプロトコルはレプリカデータを比較して違いを調和させる
- レプリカの同期には **readリペア**と **anti-entropyリペア**の２つのモードがある
- readリペアは整合性レベルを満たすために複数のレプリカからデータを読み、古いデータをもつ場合には更新をかける
- anti-entropyリペアは手動メンテナンスの１つで、major compactionを引き起こし、テーブルのハッシュ表現であるMerkleツリーを比較し修正を行う
- hinted handoffは書き込み時のanti-entropyメカニズムと考える事ができる

.footnote[
[CDG2](http://shop.oreilly.com/product/0636920043041.do) Ch6. Anti-Entropy, Repair and Merkle Trees
]

---

# 内部データ構造

ディスク
- commit log: commit logは書き込みを行った時に最初に書き込まれる場所。クラッシュリカバリーのために使われる。SSTableにデータがフラッシュされると役目を終える。
- SSTable: memtableのオブジェクトの数が一定数以上になるとSSTableに非同期にフラッシュされる。このとき新しいmemtableが作られる。SSTableはイミュータブル。

メモリー
- memtable: commit logのデータは次にmemtableに移動される。テーブルごとにmemtableは作られる。JVMヒープとoff-heapメモリーに保存される。
- key cache: partition keyからrowインデックスへのmap。JVMヒープ上に保存される
- row cache: row全体のキャッシュ。off-heapメモリーに保存される。
- counter cache: カウンタのキャッシュ

.footnote[
[CDG2](http://shop.oreilly.com/product/0636920043041.do) Ch6. Memtables, SSTables, and Commit Logs
]

---

# コンパクション

- SSTableは追記しかされないので書き込みの性能は高い。しかし読み込み時に無駄が発生してしまう。
- コンパクションはキーをマージしtombstoneを破棄しソートを行い新しくインデックスを作ることで読み込みを最適化する
- コンパクションは定期的に行われる
- 実行時には古いSSTablesの読み込みと新しいSSTableの書き込みが発生し、一時的にディスクI/Oとデータ使用量が上昇する
- major compactionは複数のSSTableを１つにまとめる。使用は推奨されない。

.footnote[
[CDG2](http://shop.oreilly.com/product/0636920043041.do) Ch6. Compaction, Ch12. Compaction
]

---

# サポートされているオペレーション

---

# データモデル

用語
- keyspace: tableのコンテナ
- table: rowのコンテナ
- row: primary keyで参照されるcolumnのコンテナ
- column: 名前と値のペア
- wide row: 非常に多くのcolumnをもつrow。partitionとも言う。
- composite key: wide rowを表現する特殊なprimary key
- partition key: composite keyを構成するキー。保存するノードを決定するために使う。複数のcolumnから成ることもある。
- clustering key: composite keyを構成するキー。データのソート順を決める。
- static column: primary keyの一部ではなくすべてのrowで共有されるcolumn
- secondary index: primary key以外のcolumnでクエリーをすることを可能にするキー

.footnote[
[CDG2](http://shop.oreilly.com/product/0636920043041.do) Ch4. Cassandra’s Data Model
]

---

# データモデリング

- 静的データ型をもつスキーマ
- クエリーファースト
- ストレージを意識して最適化
- 非正規化
- No join, no relation
- Chebotko Diagramなどで可視化

<img src="https://d3ansictanv2wj.cloudfront.net/cass_05_chebotko_logical-733194ef2128d09b7f13ab078dc0e889.png" height="240px">

.footnote[
[CDG2](http://shop.oreilly.com/product/0636920043041.do) Ch5. Data Modeling
]

---

# Write Path

---

# Read Path

---

# Lightweight Transaction

---

class: center, middle

# ZooKeeper


