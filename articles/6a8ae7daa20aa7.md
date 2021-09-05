---
title: "Goのsliceのcapacityはどのように増加していくか"
emoji: "🐥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["go", "golang", "言語仕様"]
published: false
---

## 概要

Goのsliceには容量(capacity)があります。このcapacityは、sliceに要素が追加されて足りなくなった際に、自動で増加するようになっています。
この記事では、capacityが増加した際にslice内部でどのような変更が行われているかと、capacityがどのような規則で増えていくかを検証していきます。

## 結論だけ知りたい人用のまとめ

sliceのcapacityを増加させるときは、underlying array(基底配列)を新しいcapacityで確保し直してポイントを貼り直している。
capacityを増やす際の、新しいcapacityの計算方法は下記の通り。

1. 新しいcapacity(仮)を決める。元のcapacityが1024未満なら元のcapacityの2倍、1024以上なら1.25倍する。
2. 1の計算結果から、実際に確保するメモリ容量を計算する。メモリ容量はsliceの要素のtypeにも依存する。
3. 2で計算したメモリ容量から、最終的なcapacityを計算する。

## 前提: sliceの構造とcapacity

前提として、あらためてsliceがどのような構造になっていて、capacityとはなにかを簡単に説明します。
すでに理解している方はこの項は読み飛ばしてください。

### sliceの構造

sliceがGoの内部でどのように定義されているかを見てみましょう。

```go
type slice struct {
	array unsafe.Pointer
	len   int
	cap   int
}
```

ご覧のように、sliceは内部にarrayへのpointerを持っています。
つまり、sliceというのは、別にarrayを用意して、そこへの参照を持ったものになります。
このsliceが参照しているarrayのことを、`underlying array (基底配列)`と呼んだりします。

なので、例えば、長さが5のbyteのsliceを作成した場合は、下記のように、長さが5のbyteのarrayを内部で作成し、そちらを参照している状態になります。

![slice and array relation](https://go.dev/blog/slices-intro/slice-1.png "slice and array relation")

引用元: [Go Slices: usage and internals](https://go.dev/blog/slices-intro)

詳しくは、[Go Slices: usage and internals](https://go.dev/blog/slices-intro)にあります。


### lengthとcapacity

上記のsliceの構造を前提として、`capacity`と`length`が何であるかを指しましょう。
一言で言ってしまえば、

```
capacity: sliceが参照しているunderlying arrayの容量
length: sliceが参照しているunderlying arrayに入っている要素の数
```

ということになります。


## 検証1 capacityが増えるときにsliceにはどのような変更がなされるか

ここまでの説明のようにsliceが内部でarrayを参照していることになると、sliceにcapacityを超える数の要素を入れたらどうなるのかという疑問が湧くことになると思います。
結論を言うと、underlying arrayの長さをそのまま後から変えることはできないので、`underlying arrayを新しく確保してpointerを貼り直している`ということになります。

それを実際に下記のコードで検証してみましょう。

```go
package main

import (
	"fmt"
)

func main() {
	s := make([]int, 0, 2)
	fmt.Printf("slice: %p, cap:%v , len: %v, underlying array:%p\n", &s, cap(s), len(s), (*[0]int)(s))
	s = append(s, 0)
	fmt.Printf("slice: %p, cap:%v , len: %v, underlying array:%p\n", &s, cap(s), len(s), (*[1]int)(s))
	s = append(s, 0)
	fmt.Printf("slice: %p, cap:%v , len: %v, underlying array:%p\n", &s, cap(s), len(s), (*[2]int)(s))
	s = append(s, 0)
	fmt.Printf("slice: %p, cap:%v , len: %v, underlying array:%p\n", &s, cap(s), len(s), (*[3]int)(s))
}
```

[The Go Playground](https://play.golang.org/p/Kf3xCSdyLng)

実行すると下記のような結果が得られると思います。

```
slice: 0xc00000c030, cap:2 , len: 0, underlying array:0xc000018030
slice: 0xc00000c030, cap:2 , len: 1, underlying array:0xc000018030
slice: 0xc00000c030, cap:2 , len: 2, underlying array:0xc000018030
slice: 0xc00000c030, cap:4 , len: 3, underlying array:0xc00007a000
```

出力内容は、順番に、sliceのポインタ、capacity, length, underlying arrayのポインタです。
最初にcapcityが2のsliceを作成し、そこに一つずつ要素を足していって、それぞれがどのように変化していくか見ています。
lengthが2になるまではただlengthが増えていくだけで他は変わりないですが、capcityが足りなくなったタイミングで、capacityが2から4に増加しています。
そして、それと同時に、underlying arrayのポインタが変更されています。
つまり、このタイミングで、capacityを増やすために、underlying arrayが確保し直されていることになります。

## 検証2: sliceのcapacityは実際にどのように増えるか

前項のようにunderlying arrayを新しく確保することで、sliceのcapacityは増加するわけですが、その増え方はどのようになっているのでしょうか。

以下のようなコードで試して見ましょう

TODO: playgroundをはる

実行結果から、以下のようなことがわかります。

- Capacityが1024になるまでは、２倍で増えていく
- 1024を超えてから、増加率が低くなる

さらに、以下のように、いろんなelementのtypeでも実行してみましょう。

TODO: playgroundをはる

長いのですべては貼りきれませんが、上記の実行結果から、

- sliceのelementのtypeによって、1024を超えてからのcapacityの増え方には違いがある

ということもわかります。

さらに、それぞれの結果をよく見比べると、

- sliceのelementのtypeによっては、違うtype同士でも1024を超えてからのcapacityの増え方が同じものもある

ということもわかります。これは、例えば、boolとbyte、int32とfloat32, int64とfloat64などを見比べるとわかります。

あらためてまとめると、下記の3点の特徴がわかったので、次項で内部実装を追いかけてそれらの根拠を探ってみましょう。

- Capacityが1024になるまでは、２倍で増えていく
- 1024を超えてから、増加率が低くなる
- sliceのelementのtypeによって、1024を超えてからのcapacityの増え方には違いがある(ただし、違うtype同士でも増え方が同じものもある)

### 実際の内部実装

上記について、実際の実装を追いながら確認していきましょう。
runtimeパッケージのgrowsliceという関数を見ていきます。

#### IO

#### 流れ

細かいオプションに合わせた処理等を省くと、この`growslice`は、ざっくり3つの処理にわけられそうです。

1. 仮決めのcapacityを設定する

2. 仮決めのcapacityから、新しく確保するメモリ、capacityを決定する

3. 新しいcapacityに合わせて、新しいunderlying arrayのメモリを確保し、古いunderlying arrayから要素を移動させる。

#### 詳細

それでは、上記の流れに沿って見ていきましょう

1. 仮決めのcapacityを設定する

2. 仮決めのcapacityから、新しく確保するメモリ、capacityを決定する

3. 新しいcapacityに合わせて、新しいunderlying arrayのメモリを確保し、古いunderlying arrayから要素を移動させる。


## 結論

## 参考

[Go Slices: usage and internals](https://go.dev/blog/slices-intro)
[Arrays, slices (and strings): The mechanics of 'append'](https://go.dev/blog/slices#TOC_9.)
[Go slice ベストプラクティス](https://qiita.com/imoty/items/bb18fb50d526474d2d10)