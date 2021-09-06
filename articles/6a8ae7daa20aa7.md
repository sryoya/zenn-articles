---
title: "Goのsliceのcapacityはどのように増加していくか"
emoji: "🗄️"
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

`
capacity: sliceが参照しているunderlying arrayの容量
`
`
length: sliceが参照しているunderlying arrayに入っている要素の数
`

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

前項で確認したように、underlying arrayを新しく確保することでsliceのcapacityは増加するわけですが、その増え方は実際にどのようになっているのでしょうか。

以下のようなコードで試してみましょう。

TODO: playgroundをはる

実行結果から、以下のようなことがわかります。

`
capacityが1024になるまでは、2倍で増えていく
`
`
1024を超えてから、増加率が低くなる
`

さらに、以下のように、いろんなelementのtypeでも実行してみましょう。

TODO: playgroundをはる

長いのですべては貼りきれませんが、上記の実行結果から、

`
sliceのelementのtypeによって、1024を超えてからのcapacityの増え方には違いがある
`

ということもわかります。

さらに、それぞれの結果をよく見比べると、

`
sliceのelementのtypeによっては、違うtype同士でも1024を超えてからのcapacityの増え方が同じものもある
`

ということもわかります。これは、例えば、boolとbyte、int32とfloat32, int64とfloat64などを見比べるとわかります。

あらためてまとめると、下記のの特徴がわかったので、次項で内部実装を追いかけてそれらの根拠を探ってみましょう。

- capacityが1024になるまでは、2倍で増えていく
- 1024を超えてから、増加率が低くなる
- 1024を超えてからの増加率は一定ではない
- sliceのelementのtypeによって、1024を超えてからのcapacityの増え方には違いがある(ただし、違うtype同士でも増え方が同じものもある)

### 実際の内部実装

上記について、実際の実装を追いながら確認していきましょう。
runtimeパッケージの[`growslice`](https://github.com/golang/go/blob/9133245be7365c23fcd60e3bb60ebb614970cdab/src/runtime/slice.go#L164)という関数を見ていきます。

#### 概要

この関数は、append時にcapacityがなくなった際に、sliceを拡張するために呼ばれます。
IOと挙動の概要はコメントに書いてあります。

```go
// growslice handles slice growth during append.
// It is passed the slice element type, the old slice, and the desired new minimum capacity,
// and it returns a new slice with at least that capacity, with the old data
// copied into it.
```

コメントの通り、inputとして、sliceの要素のtype, (拡張前の)slice, 必要なcapを受け取るようです。
そして、要求された数かそれ以上のcapを持った新しいsliceを作って、そこに古いデータをコピーして返してくれるようです。

一点注目する点として、説明にある通り、この関数は、`新しいslice`を作って返してくれるようです。
つまり、「検証1」で試した結果では、capacityが増えてもsliceのポインタはそのままでしたが、あくまでポインタがそのままなだけで、slice自体は新しいもので置き換えているようです。

#### IO

関数のIOはこのようになっています。
コメントの通り、sliceの要素のtype, (拡張前の)slice, 必要なcapの順番でparameterを受け取っているようです。

```go
func growslice(et *_type, old slice, cap int) slice {
```

#### 流れ

細かいケースを省くと、この`growslice`は、ざっくり3つの処理にわけられそうです。

1. 仮決めのcapacityを設定する

2. 仮決めのcapacityから、新しく確保するメモリ、capacityを決定する

3. 新しいcapacityに合わせて、新しいunderlying arrayのメモリを確保し、古いunderlying arrayから要素を移動させる。

#### 詳細

それでは、上記の流れに沿って見ていきましょう

##### 1. 仮決めのcapacityを設定する

```go
newcap := old.cap
doublecap := newcap + newcap
if cap > doublecap {
    newcap = cap
} else {
    if old.cap < 1024 {
        newcap = doublecap
    } else {
        // Check 0 < newcap to detect overflow
        // and prevent an infinite loop.
        for 0 < newcap && newcap < cap {
            newcap += newcap / 4
        }
        // Set newcap to the requested cap when
        // the newcap calculation overflowed.
        if newcap <= 0 {
            newcap = cap
        }
    }
}
```

この部分は、順番に読んでいけば、比較的わかりやすそうです。
順番に見ていくと、仮決めのcapacityの決め方は、

1. 必要なcapcityが、現状のcapacityの2倍以上だったら、必要とされているcapacityを採用する
2. 現状のcapacityが1024未満だったら、現状のcapacityを2倍したものを採用する
3. 現状のcapacityが1024以上だったら、現状のcapacityを2倍したものを採用する
(3のパターンで実行結果で決まった仮決めのcapacityが0以下だったら(コメント曰くoverflowしたとき用？)、必要とされているcapacityを採用する)

となります。
この時点で、先程の実際にコードを動かしてわかった、

- capacityが1024になるまでは、2倍で増えていく
- 1024を超えてから、増加率が低くなる

という点に関しては、このコードが答えになりそうです。
一方で、上記のコードだと、基本的には、1024以降のcapacityの増加率は1.25倍で一定になりそうですが、実際にはそうではありません。そして、typeによって、増加率も異なっています。
その点について、後続のコードから理由を探ってみましょう。

##### 2. 仮決めのcapacityから、新しく確保するメモリ、capacityを決定する

それでは、最終的なcapacityの決め方を見て行きましょう。
ここは、少しややこしくなります。

```go
var overflow bool
var lenmem, newlenmem, capmem uintptr
// Specialize for common values of et.size.
// For 1 we don't need any division/multiplication.
// For sys.PtrSize, compiler will optimize division/multiplication into a shift by a constant.
// For powers of 2, use a variable shift.
switch {
case et.size == 1:
    lenmem = uintptr(old.len)
    newlenmem = uintptr(cap)
    capmem = roundupsize(uintptr(newcap))
    overflow = uintptr(newcap) > maxAlloc
    newcap = int(capmem)
case et.size == sys.PtrSize:
    lenmem = uintptr(old.len) * sys.PtrSize
    newlenmem = uintptr(cap) * sys.PtrSize
    capmem = roundupsize(uintptr(newcap) * sys.PtrSize)
    overflow = uintptr(newcap) > maxAlloc/sys.PtrSize
    newcap = int(capmem / sys.PtrSize)
case isPowerOfTwo(et.size):
    var shift uintptr
    if sys.PtrSize == 8 {
        // Mask shift for better code generation.
        shift = uintptr(sys.Ctz64(uint64(et.size))) & 63
    } else {
        shift = uintptr(sys.Ctz32(uint32(et.size))) & 31
    }
    lenmem = uintptr(old.len) << shift
    newlenmem = uintptr(cap) << shift
    capmem = roundupsize(uintptr(newcap) << shift)
    overflow = uintptr(newcap) > (maxAlloc >> shift)
    newcap = int(capmem >> shift)
default:
    lenmem = uintptr(old.len) * et.size
    newlenmem = uintptr(cap) * et.size
    capmem, overflow = math.MulUintptr(et.size, uintptr(newcap))
    capmem = roundupsize(capmem)
    newcap = int(capmem / et.size)
	}
```

まず、ここでは、単純にcapacityを決めるだけでなく、メモリサイズ(capacity, 古い要素を入れるlength, 新しく宣言するlength)も決めています。
(これらは、`uintptr`というGoの組み込み型のtypeで表現されています。`uintptr`は、ポインタを格納できる整数型です。)

そして、sliceの要素のtypeのサイズに応じて、決め方が少しづつ異なるようです。
細かく見ると、シフトが行われていたりややこしいですが、やっていることは基本的に同じです。

まず、新しいunderlying arrayのためにメモリを確保するために、メモリ容量を決めます。
さらに、そのunderlying arrayに古いsliceに、すでに入っている要素をコピーする必要があるので、そのためのメモリ容量(また要素をまだ入れない容量も)決める必要があるので、それぞれ計算しています。
そして、実際に必要なメモリ容量は、要素のtypeのサイズに依存するので、要素のtypeによって条件分岐して、要素のtypeも使って計算しています。

なので、サイズが1のtype(`et.size == 1`)の場合はメモリサイズは計算に使われていない(`byte`や`bool`など1byteで表現できるtypeがこれにあたる)一方で、それ以外の場合はサイズが計算に利用されています。

さらに詳しく、肝心のcapacityの決め方を見てみましょう。
いずれの場合にも、実際に確保するメモリを決めてから、それをcapとして計算し直しています。

そして、メモリ容量の決め方ですが、`roundupsize`という関数を呼び出しています。
こちらの関数も簡単に見てみましょう。

```go
// Returns size of the memory block that mallocgc will allocate if you ask for the size.
func roundupsize(size uintptr) uintptr {
	if size < _MaxSmallSize {
		if size <= smallSizeMax-8 {
			return uintptr(class_to_size[size_to_class8[divRoundUp(size, smallSizeDiv)]])
		} else {
			return uintptr(class_to_size[size_to_class128[divRoundUp(size-smallSizeMax, largeSizeDiv)]])
		}
	}
	if size+_PageSize < size {
		return size
	}
	return alignUp(size, _PageSize)
}
```

コメントにあるように、この関数は、実際のメモリブロックの単位に合わせて、確保すべきメモリ容量を調整してくれているようです。
細かい説明はここでは割愛しますが、これは、`TCMalloc` というメモリアロケータに合わせたものです。
この`TCMalloc`は、格納するデータのサイズに応じて、格納するべきメモリの単位を指定しているので、その規定に合うような実際に確保するメモリ容量を切り上げて調整しています。

このようにしてメモリ容量を決めています。
そして、最後に、計算したメモリ容量から、実際に入れられる要素の数であるcapacityを計算し直しています。

これで、以下の特徴の理由がわかったと思います。

- 1024を超えてからの増加率は一定ではない
- sliceのelementのtypeによって、1024を超えてからのcapacityの増え方には違いがある(ただし、違うtype同士でも増え方が同じものもある)


最終的なcapacityは、sliceの要素のサイズを考慮してメモリ容量に基づいていて、かつメモリ容量がメモリアロケータの仕様に合わせて最適化されているので、上記のような特徴となるということです。

##### 3. 新しいcapacityに合わせて、新しいunderlying arrayのメモリを確保し、古いunderlying arrayから要素を移動させる

すでにcapacityの決まり方はわかりましたが、実際に決まったcapacityとメモリ容量を使ってどのような処理をしているかを、残りのコードを見て確認していきましょう。

```go
var p unsafe.Pointer
if et.ptrdata == 0 {
    p = mallocgc(capmem, nil, false)
    // The append() that calls growslice is going to overwrite from old.len to cap (which will be the new length).
    // Only clear the part that will not be overwritten.
    memclrNoHeapPointers(add(p, newlenmem), capmem-newlenmem)
} else {
    // Note: can't use rawmem (which avoids zeroing of memory), because then GC can scan uninitialized memory.
    p = mallocgc(capmem, et, true)
    if lenmem > 0 && writeBarrier.enabled {
        // Only shade the pointers in old.array since we know the destination slice p
        // only contains nil pointers because it has been cleared during alloc.
        bulkBarrierPreWriteSrcOnly(uintptr(p), uintptr(old.array), lenmem-et.size+et.ptrdata)
    }
}
memmove(p, old.array, lenmem)

return slice{p, old.len, newcap}
```

## まとめ

## 参考

[Go Slices: usage and internals](https://go.dev/blog/slices-intro)
[Arrays, slices (and strings): The mechanics of 'append'](https://go.dev/blog/slices#TOC_9.)
[Go slice ベストプラクティス](https://qiita.com/imoty/items/bb18fb50d526474d2d10)
[The Go Programming Language Specification - Conversions from slice to array pointer](https://golang.org/ref/spec#Conversions_from_slice_to_array_pointer)
[TCMalloc, Google's Customized Memory Allocator for C and C++, Now Open Source](https://www.infoq.com/news/2020/02/google-tc-malloc-open-source/)