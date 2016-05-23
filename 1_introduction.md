
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

※さらっとなので詳細や網羅性はないです。今後の勉強会で深めてね！

---

# 進め方の提案

- 質問は最後に（時間なくなるから）
- 質問の答えは当日に回収しなくてよい。次回発表時にでも。（発表者が御飯食べる時間なくなる）
- 内容は小さくてもいいし完璧じゃなくてもいい。継続が大事。徐々にレベルアップを。
- 内容は自由。でも実務に関係していたほうがよい。
- 提案募集

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
- ストリームによるデータアクセス: write-once, read-manyなデータを想定。解析などに向く。
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

**namenode**

- ファイルシステムのツリーとメタデータを管理する
- ファイルのブロックがどのdatanodeに属するかを知っている。起動時にdatanodeからブロックの情報を受け取り、メモリー上に保存する。
- namespace imageとedit logの２つのファイルを永続化する
- namenodeの情報が失われるとdatanodeのブロックからファイルを再現する方法が分からなくなりファイルシステムとして機能しなくなる

**datanode**

- ブロックを保存する

---

# サポートされているオペレーション

- FileSystem#create
- FileSystem.get
- FileSystem#open
- FileSystem#listStatus
- FileSystem.mkdirs
- FileSystem.exists

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

# Master Server

master serverの役割

- regionの割当て
- ロードバランシング
- region serverの復旧
- region splitの完了のモニタリング
- サーバーの死活監視

※ HBaseクラスターはregion serverが落ちないかつregion splitが起きないかぎりmasterがいなくても生存できる

.footnote[
[AHA](http://shop.oreilly.com/product/0636920035688.do) Ch2. HBase roles
] 

---

# Region Server

region serverの役割

- regionをホストする。つまりデータをホストする。
- splitやコンパクションをするか判断し実行する
- クライアントは初回以外はregion serverとダイレクトにやりとりする。（初回はZooKeeperからmaster serverを探し、master serverからregion server一覧を得る）

.footnote[
[AHA](http://shop.oreilly.com/product/0636920035688.do) Ch2. HBase roles
] 

---

# 自動シャーディング

- スケールとロードバランシングの単位は **region**
- regionは連続した一連のrow
- region serverが複数のregionを担当する
- regionが一定以上の大きさになると２つに分割される( **splitting**)
- splittingはコンパクションとは反対の動きをする

.footnote[
[HBDG2](http://shop.oreilly.com/product/0636920033943.do) Ch1. Auto-Sharding
[AHA](http://shop.oreilly.com/product/0636920035688.do) Ch2. Splits (Auto-Sharding)
] 

---

# ロードバランシング

- region serverが新しく参加したり落ちたり、regionが分割されたりすると、regionの分布に偏りが出る
- master serverはregionの分布が均一になるように一定間隔で **ロードバランサー**を実行する
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

- Put
- Get
- Delete
- Append: CAS
- Mutate: Put + Delete
- Batch: 複数のrowに対する操作
- Scan: 複数のrowを取得する範囲検索
- Filter
- Coprocessor
- Counter
- region-local transaction

.footnote[
[HBDG2](http://shop.oreilly.com/product/0636920033943.do) Ch3. CRUD Operations, Ch4. Client API: Advanced Features
]

---

class: center, middle

# Cassandra

---

class: center, middle

# Cassandraって何？

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
- snitchはどのノードから読み込み、書き込みをすべきかをきめるために使われる
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

# レプリケーション

- replication factorでいくつデータのコピーを持つか指定する
- 最初のノードのみpartitionerのトークンに基いて決められる。残りは **replication strategy**で位置を決定する。
- NetworkTopologyStrategyは可用性を最大化するためネットワークトポロジーを意識してレプリカノードを決める
- replication strategyはkeyspaceを作るときに指定する


.footnote[
[CDG2](http://shop.oreilly.com/product/0636920043041.do) Ch6. Replication Strategies
]

---

# 整合性レベル

- 書き込みや読み込みクエリーで **整合性レベル**を指定できる
- 整合性レベルが高いほど多くのノードが応答し、レプリカの値が等しいか確認する
- 弱い整合性レベルにはANY, ONE, TWO, THREE、強い整合性レベルにはQUORUM, ALLがある
- QUORUMレベルはレプリカノードの過半数となる(replication factor / 2 + 1)

.footnote[
[CDG2](http://shop.oreilly.com/product/0636920043041.do) Ch6. Consistency Levels
]

---

# クエリーとコーディネーターノード

- クライアントはクエリーの際にクラスターのどのノードと接続してもよい
- 接続されたノードは **コーディネーターノード**となる
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
- **コンパクション**はキーをマージし **tombstone**を破棄しソートを行い新しくインデックスを作ることで読み込みを最適化する
- コンパクションは定期的に行われる
- 実行時には古いSSTablesの読み込みと新しいSSTableの書き込みが発生し、一時的にディスクI/Oとデータ使用量が上昇する
- major compactionは複数のSSTableを１つにまとめる。使用は推奨されない。

.footnote[
[CDG2](http://shop.oreilly.com/product/0636920043041.do) Ch6. Compaction, Ch12. Compaction
]

---

# サポートされているオペレーション

- SELECT
- INSERT
- UPDATE
- DELETE
- TRUNCATE
- BEGIN BATCH
- Lightweight Transaction

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
- **Chebotko Diagram**などで可視化

<img src="https://d3ansictanv2wj.cloudfront.net/cass_05_chebotko_logical-733194ef2128d09b7f13ab078dc0e889.png" height="240px">

.footnote[
[CDG2](http://shop.oreilly.com/product/0636920043041.do) Ch5. Data Modeling
]

---

# Write Path

1. クライアントからクエリーを受け取ったノードがコーディネーターとなる
2. コーディネーターはpartitionerでレプリカノードを決定する
3. コーディネーターはすべてのレプリカに書き込みリクエストを送る
4. レプリカノードはリクエストを受け取ると直ちにcommit logに書き込む
5. 次にレプリカノードはデータをmemtableに書き込む
6. commit logやmemtableのサイズが規定値以上になったらフラッシュをスケジュールする
7. レプリカノードは書き込み成功のレスポンスを返す
8. コーディネーターは整合性レベルを満たす数の成功レスポンスが返ってきたらクライアントに返答する
9. コーディネーターはレプリカがタイムアウト以内に応答しなかったらヒントを残す
10. レプリカノードはスケジュールされたフラッシュを実行し、memtableをSSTableとして保存しcommit logを消去する。コンパクションが必要ならスケジュールする。

.footnote[
[CDG2](http://shop.oreilly.com/product/0636920043041.do) Ch9. Writing
]

---

# Read Path

1. クライアントからクエリーを受け取ったノードがコーディネーターとなる
2. コーディネーターはpartitionerでレプリカノードを決定する
3. snitchによって一番速いレプリカを決定し、読み込みリクエストを送る
4. コーディネーターは他のレプリカにdigest requestを送る
5. リクエストを受けたレプリカはrow cacheをチェックする
6. row cacheになければレプリカはmemtableとSSTableを探索する
7. 最新のタイムスタンプのcolumnを選択し、tombstoneが付いているデータは除去し、SSTableのデータとmemtableのデータをマージする
8. 一番速いレプリカは結果をコーディネーターに返す
9. digest requestを受信したレプリカはデータのdigest hashを計算しコーディネーターに返す
10. コーディネーターは一番速いレプリカが返したデータからdigest hashを計算し、他のレプリカのdigest hashと比較
11. 整合性レベルを満たせば一番速いレプリカのデータをクライアントに返す

.footnote[
[CDG2](http://shop.oreilly.com/product/0636920043041.do) Ch9. Reading
]

---

# Read Path

11. 整合性レベルを満たせば一番速いレプリカのデータをクライアントに返す
12. 整合性レベルを満たしていなければ、readリペアを行う
13. readリペアではすべてのレプリカに読み込みリクエストを送り、コーディネーターは最新のデータになるようマージした結果をクライアントに返す。古いレプリカは結果に基づき非同期に更新される。

.footnote[
[CDG2](http://shop.oreilly.com/product/0636920043041.do) Ch9. Reading
]

---

# Materialized View

- **secondary index**をカーディナリティーが高いデータに使うとリングの大多数のノードを探索するハメになり使えない
- **Materialized View**はオリジナルのclustering keyにないcolumnに対するクエリーが可能なビューを作る
- Materialized Viewによって複数の非正規テーブルをアプリケーション側で同期する必要がなくなる

```
cqlsh> CREATE MATERIALIZED VIEW reservation.reservations_by_confirmation AS SELECT *FROM reservation.reservations_by_hotel_dateWHERE confirm_number IS NOT NULL and hotel_id IS NOT NULL andstart_date IS NOT NULL and room_number IS NOT NULLPRIMARY KEY (confirm_number, hotel_id, start_date, room_number);
```

.footnote[
[CDG2](http://shop.oreilly.com/product/0636920043041.do) Ch5. Materialized Views
]

---

class: center, middle

# ZooKeeper

---

# ZooKeeperって何？

独立したアプリケーション同士が協調動作をすることを可能にするサービス

ZooKeeperは、アプリケーション開発者が、協調動作にではなくアプリケーションのロジックに専念できるように、脆弱になりがちな難しい部分を肩代わりする。

用途に特化せず汎用に設計されており、簡素なAPIを提供する。

---

# ZooKeeperの適用例

- HBase: マスターの選出、利用可能なサーバーとクラスターのメタデータ管理
- Kafka: 障害検出、トピックの検出・生成、消費状態の管理
- Solr: クラスターメタデータの保存と更新の協調
- Yahoo! Fetching Service: マスタ選出、故障検出、メタデータ保持
- Facebook Messages: 処理の分散、フェイルオーバー、サービスディスカバリー

.footnote[
[ZDPC](http://www.oreilly.co.jp/books/9784873116938/) 1.1 ZooKeeperの使命
]

---

# ZooKeeperが生まれた理由

> ZooKeeperは元々、Yahoo! で開発された。Yahoo! ではたくさんの大規模分散アプリケーショ ンが開発されている。われわれは、分散協調の側面が適切に扱われておらず、単一故障点を持つア プリケーションや、障害に対して脆いアプリケーションがデプロイされていることに気が付いた。 その一方で、分散協調の部分に人手を取られ、アプリケーションの機能に注力できていない開発者 もいた。さらに、われわれは、これらのアプリケーションが基本的な協調機能のいくつかを共有し ていることに気付いた。だから、われわれは、一度実装してしまえばさまざまなアプリケーション で使い回しできる一般的な解決法を見つけようとしたのだ。ZooKeeper は、われわれが思ってい たよりもはるかに汎用性があり、広く使われるようになった。

.footnote[
[ZDPC](http://www.oreilly.co.jp/books/9784873116938/) 1.4 ZooKeeperは成功した。但し書きつきで
]

---

# CAPの定理に当てはめると

CP: 可用性よりも整合性を選んだ（ZooKeeper本にはACと書いてある）

※ ZooKeeperはネットワークが分断した時でも読み出しだけはできる

---

# データ構造

ZooKeeperはデータをznodeという単位で管理する。

znodeはファイルシステムのように階層構造をなす。

znodeはパス形式の名前で参照できる。

znodeの値はバイト配列である。

<img src="https://s3.amazonaws.com/files.dezyre.com/images/blog/Zookeeper+and+Oozie%3A+Hadoop+Workflow+and+Cluster+managers/Zookeeper+Data+Model.png" height="240px">

.footnote[
[ZDPC](http://www.oreilly.co.jp/books/9784873116938/) 2.1 ZooKeeperの基本
]

---

# 永続znodeと短命znode

- 永続(persistent)znodeは明示的にdeleteを呼び出さないと消せない
- 短命(ephemeral)znodeはこのznodeを作ったクライアントがクラッシュしたり、接続がクローズされたりすると自動的に削除される

master-workerアプリケーションの場合`/master`や`workers/worker-*` znodeは短命znodeになる。
masterがクラッシュしたらフェイルオーバーしなければいけないし、workerがクラッシュしたらタスクを割り当ててはいけないからだ。
`/tasks/task-*`は永続znodeになる。タスクの登録者がクラッシュしようがタスクは消えてはいけないからだ。

.footnote[
[ZDPC](http://www.oreilly.co.jp/books/9784873116938/) 2.1.2 znodeのモード
]

---

# シーケンシャルznode

シーケンシャルznodeは単調に増加するユニークな整数値が与えられる

`/tasks/task-`を指定してシーケンシャルznodeを作ると、`/tasks/task-1`, `/tasks/task-2`, ...のようにパスが作られる

.footnote[
[ZDPC](http://www.oreilly.co.jp/books/9784873116938/) 2.1.2 znodeのモード
]

---

# サポートされているオペレーション

- create
- getData
- setData
- exists
- getChildren
- delete

---

# 監視と通知

ZooKeeperではポーリングではなく監視と通知を行う。

以下のオペレーションではznodeに監視を適用できる

- getData
- exists
- getChildren

監視は１度きりの操作で、１回の通知しか引き起こさない。継続的に通知をもらうには再度監視を登録する

ZookKeeperはクライアントが観測する更新の順序を保証する。通知は他の変更が同じznodeに加えられるよりも先にクライアントに配送される。

.footnote[
[ZDPC](http://www.oreilly.co.jp/books/9784873116938/) 2.1.3 監視と通知
]

---

# バージョン


---

# クォラム

ZooKeeperが稼働するために必要な最小の稼働しているサーバの数をクォラムという。

クォラムはデータツリーを複製しなければいけない最低の台数を決める。クォラムを満たせたら、クライアントは先に進むことができる。残りは非同期に同期される。

例えば、5台サーバーがあるとすると、クォラムは3台で、２台までの障害を許容することができる。

.footnote[
[ZDPC](http://www.oreilly.co.jp/books/9784873116938/) 2.2.1 ZooKeeperのクォラム
]

---

# セッション

クライアントはアンサンブルのいずれか1つのサーバにランダムに接続する

クライアントが発行するすべての操作は、セッションと紐付いている。セッションが終了するとセッションの間に作られた短命znodeは消える。


接続中のサーバからしばらくの間クライアントへの通信が途絶えると、セッションは他のサーバに移動する。セッションの他のサーバへの移動は透過的に行われる。

セッションは順序を保証する。1つのセッションの中で行われる複数のリクエストは FIFO順で行われる。

---

class: center, middle

# アーキテクチャー

---

# ZooKeeperアンサンブル

複数のZooKeeperサーバーでクラスターを組む。この単位をアンサンブルという。

ZooKeeperアンサンブルは一般的に３，５，７などの奇数台で構成する。

---

# リーダー、フォロワー、オブザーバー

ZooKeeperアンサンブルのうち１つはリーダーの役割をもち、それ以外はフォロワーとなる。

リーダーは、シー ケンサーとして機能し、ZooKeeper 状態の更新の順序を決定する。

フォロワーは、状態の更新が クラッシュに耐えることを保証するために、リーダーが提案する更新を受け取り、投票を行う。

オブザーバーはこの過程に関与せず、スケーラビリティのためにある。

---

# リクエストとトランザクション

読み出しリクエスト（exists、getData、getChildren）の場合、ローカルのデータから読み出す。

更新リクエスト（create、delete、setData）の場合、リクエストはリーダーに送られる。
リーダーは状態更新の手順の単位であるトランザクションを生成し、それらが干渉しないよう順番に適用する。
トランザクションはアトミックで冪等である。

.footnote[
[ZDPC](http://www.oreilly.co.jp/books/9784873116938/) 9.1 リクエスト、トランザクション、識別子
]

---

# リーダーの選出

リーダーはアンサンブルに１つしか存在しないため選出する必要がある。

個々のサーバーはLOCKING状態で開始する。

既存のリーダーがあれば、他のサーバが新たなサーバにどのサーバがリーダーであるか教える。新しいサーバーはリーダーと状態が整合しているか確認する。

アンサンブル全体がLOOKING状態である場合は、もっとも最新のトランザクションをもつものにリーダーになるよう投票する。クォラムを構成するに足る投票を受け取ったサーバーはリーダーと宣言し、リーダーの役割を開始する。それ以外はフォロワーとなりリーダーに接続し、状態を同期する。

.footnote[
[ZDPC](http://www.oreilly.co.jp/books/9784873116938/) 9.2 リーダーの選出
]

---

# Zab (ZooKeeper Atomic Broadcast) プロトコル

トランザクションがコミットされたことをサーバーが知る仕組みにZabを使っている。

1. リーダーはPROPOSALメッセージをすべてのフォロワーに送る
2. フォロワーはPROPOSALメッセージを受け取るとACKをリーダーに返す
3. リーダーは、クォラムを形成する数の受信通知を受け取るとフォロワーにCOMMITメッセージを送りコミットしたことを知らせる

.footnote[
[ZDPC](http://www.oreilly.co.jp/books/9784873116938/) 9.3 Zab:状態更新のブロードキャスト
]

---

