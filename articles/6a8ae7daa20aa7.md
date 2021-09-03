---
title: "Goのsliceのcapacityの増加ロジック"
emoji: "🐥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["go", "golang", "言語仕様"]
published: false
---

# 概要

Goのsliceには容量(capacity)があります。このcapacityは、sliceに要素が追加されて足りなくなった際に、自動で増加するようになっています。
この記事では、capacityがどのような規則で増えていくかを、内部実装も追いながら解説してみようと思います。

# 結論だけ知りたい人用のまとめ

Capacityの計算方法は下記の通り。

1. 新しいcapacity(仮)を決める。元のcapacityが1024未満なら元のcapacityの2倍、1024以上なら1.25倍する。
2. 1の計算結果から、実際に確保するメモリ容量を計算する。メモリ容量はsliceの要素のtypeにも依存する。
3. 2で計算したメモリ容量から、最終的なcapacityを計算する。

# 前提: sliceのcapacityとは
前提として、sliceのcapacityとはなにかをあらためて確認しましょう。
理解している方はこの項目は読み飛ばしてください。




TODO: 説明つづき。

# 検証: sliceのcapacityは実際にどのように増えるか

以下のようなコードで試して見ましょう

TODO: playgroundをはる

実行結果から、以下のようなことがわかります。

- Capacityが1024になるまでは、２倍で増えていく
- 1024を超えてから、増加率が低くなる
- 1024以降の増加率は、sliceのelementのtypeによって異なる（ただし、違うtypeでも一致するものものある。）

#　実際の内部実装

# 結論

# 参考

[Go Slices: usage and internals](https://go.dev/blog/slices-intro)
[Go slice ベストプラクティス](https://qiita.com/imoty/items/bb18fb50d526474d2d10)