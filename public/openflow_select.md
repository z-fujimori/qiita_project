---
title: Snowflake OpenflowでSELECT方式の連携を組む(仮)
tags:
  - Snowflake
  - Openflow
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
---

# 概要

AWSなどのRDSとのOpenflowを構築しようとした際に、標準のコネクタとしてCDC方式のコネクタが用意されている。

SELECT方式については自らプロセッサーを配置していく必要がある。

そこで、そもそもOpenflowにおけるCDC方式とSELECT方式の違いは何かと、SELECT方式の構築についてまとめる。


# CDC方式 / SELECT方式

軽く説明


# SELECT方式のOpenflowの構築


# SELECT方式の設計 「何を差分とみなすか」
- updated_at ベース
- 連番IDベース
- スナップショット比較
- 複合条件

# CDCとSELECTの比較

使い分けやメリットデメリット


# まとめ

今回、私はCDCを使いました
