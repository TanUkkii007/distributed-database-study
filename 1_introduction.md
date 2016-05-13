
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
- 2007 Amazonが[Dynamo論文](http://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf)を公開
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

# CAPの定理でいうと

CP: 可用性よりも整合性を選んだ

---

# デザイン

## 得意なこと

- 大きなファイル: 数百メガ、ギガ、テラバイトクラスのファイルを扱うよう最適化
- ストリームによるデータアクセス: write-once, read-mannyなデータを想定。解析などに向く。
- コモディティーなハードウェア: 特殊なハードウェアを必要とせず、ありふれたマシンでクラスターを組む。壊れること前提。


[HDG4](http://shop.oreilly.com/product/0636920033448.do) Ch3. The Design of HDFS


---

# デザイン

## 苦手なこと

- 低レイテンシなデータアクセス: 数10msオーダーのアクセスは無理。レイテンシよりスループットを重視。
- 大量の小さなファイル: 1台のnamenodeがすべてのファイルのメタデータをメモリー上に持つので、多くのファイルは扱えない。100万オーダーはいけるが10億オーダーは無理。

[HDG4](http://shop.oreilly.com/product/0636920033448.do) Ch3. The Design of HDFS

---

# アーキテクチャー

---

class: center, middle

# HBase


---

class: center, middle

# Cassandra


---

class: center, middle

# ZooKeeper


