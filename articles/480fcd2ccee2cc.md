---
title: "Parquetファイルについて調べてみた"
emoji: "💭"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Parquet"]
published: true
---

Parquetに初めて触れて、データ構造とか色々調べたのでメモとして残します。
# Parquetとは
Apache ParquetはHadoopエコシステムなどで主に利用される
オープンソースのファイルフォーマット。
https://parquet.apache.org/docs/overview/

### 特徴
- カラムナフォーマット(列志向)
  - csvなど行志向フォーマットと比べて、不要なカラムを読まずにすむので分析クエリが高速になる。
- プログラム言語やデータ処理基盤(Hadoop, Spark etc)に依存せずに利用可能。
- ネストされたデータタイプもサポートしている。

# フォーマット
![](/images/parquet_format.gif)
公式のドキュメントによると
- FileはいくつかのRawGroupに論理的に水平分割される。
- RawGroupには1つ以上のColumn Chunkに分けられる。
- Column Chunkははさらに1つ以上のPageに分割される。
  - 圧縮とエンコーディングはPageのメタデータで定義されているため以上分割できない。
- メタデータは一方向への書き込みを可能にするために、Raw Groupの後ろにある。
- Parquetで使用可能なデータ型は以下の通り。
```
BOOLEAN: 1 bit boolean
INT32: 32 bit signed ints
INT64: 64 bit signed ints
INT96: 96 bit signed ints
FLOAT: IEEE 32-bit floating point values
DOUBLE: IEEE 64-bit floating point values
BYTE_ARRAY: arbitrarily long byte arrays
```
16bitのintは32bit intでカバーできるのでサポートしないなど、意図的にサポートする型を少なくしているらしい。
文字列型はBYTE_ARRAYに変換する。

# 参考記事
- https://parquet.apache.org
- https://www.sambaiz.net/article/386/
