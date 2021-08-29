---
title: "Goのsliceのcapacityの増加ロジック"
emoji: "🐥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["go", "golang", "言語仕様"]
published: false
---

## 概要

Goのsliceには容量(capacity)があります。このcapacityは、sliceに要素が追加されて足りなくなった際に、自動で増加するようになっています。
この記事では、capacityがどのような規則で増えていくかを、内部実装も追いながら解説してみようと思います。

## 結論だけ知りたい人用のまとめ

## sliceのcapacityとlengthとは

## (検証) sliceのcapacityは実際にどのように増えるか

## 実際の内部実装

## 結論
