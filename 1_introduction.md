
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


---

# HBase, BigTableが生まれた理由

---

# CAPの定理に当てはめると

CP: 可用性よりも整合性を選んだ

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

.footnote[
[CDG2](http://shop.oreilly.com/product/0636920043041.do) Ch6. Hinted Handoff
]

---

# Compaction

.footnote[
[CDG2](http://shop.oreilly.com/product/0636920043041.do) Ch6. Compaction
]

---

# Anti-Entropy, Repair and Merkle Trees

.footnote[
[CDG2](http://shop.oreilly.com/product/0636920043041.do) Ch6. Anti-Entropy, Repair and Merkle Trees
]

---

# 内部データ構造

---

# サポートされているオペレーション

---

# データモデリング

---

# Write Path

---

# Read Path

---

# Lightweight Transaction

---

class: center, middle

# ZooKeeper


