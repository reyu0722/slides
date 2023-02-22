---
# try also 'default' to start simple
theme: seriph
# random image from a curated Unsplash collection by Anthony
# like them? see https://unsplash.com/collections/94734566/slidev
background: false
# apply any windi css classes to the current slide
class: 'text-center'
# https://sli.dev/custom/highlighters.html
highlighter: shi
# show line numbers in code blocks
lineNumbers: false
# some information about the slides, markdown enabled
info: |
  traP 2023春合宿らんぷろ
# persist drawings in exports and build
drawings:
  persist: false
fonts:
  sans: 'Noto Sans JP'
# use UnoCSS
css: unocss
canvasWidth: 840
---

# WebAssemblyのはなし

traP2023春合宿らんぷろ<br>
@reyu

---

# WebAssemblyとは
<br>

> ネイティブに近いパフォーマンスで動作する、コンパクトなバイナリー形式の低レベルなアセンブリー風言語<br>
https://developer.mozilla.org/ja/docs/orphaned/WebAssembly

<br>

- ブラウザ上で動く言語
  - 最近はブラウザ以外で動かすのも流行りつつある
- バイナリフォーマットを持つが直接実行はできず、VMによって実行される
  - JVMや.NETとだいたい同じ

---
layout: cols
---


# Wasmランタイムを作っています
<div class="h-2" />

::left::
春休み中やることがないので作っています

https://github.com/reyu0722/wasm-runtime


::right::

![](/images/github.png)

---

# Wasmランタイムの作り方
<div class="h-2" />

仕様では3段階に分けられているので、それに従って実装するのが自然

- Decoding
  - バイナリフォーマットを読む
  - https://webassembly.github.io/spec/core/binary に書いてあるとおりに実装する
- Validation
  - モジュールの検証
  - 面倒だし正常系には影響しないのでとりあえず飛ばしてもいい
- Execution
  - モジュールを実行する
  - https://webassembly.github.io/spec/core/exec に書いてあるとおりに実装する

---

# 進捗
<div class="h-2" />

- Decoding
  - Custom Section以外は一通り実装した
- Validation
  - なにもしてない
- Execution
  - 最近手をつけはじめた

---

# 今回話す内容
<div class="h-2" />

- 本当ならWasmランタイム自作について話したいが...
  - Decodingはやるだけ感が強く、Executionは作り始めたばっかりなのであまり話す内容がない
- なので、WebAssemblyの仕様に関する話をします

---

# LEB128
Little Endian Base 128

Wasmでは整数はLEB128でエンコードされている

- 可変長整数
- 符号付き・なしがあるが、今回は符号なしのみ考える
- 1byte (8bit) のうち1bitが「次のバイトがあるかどうか」を表す


---

# LEB128: 例
<div class="h-2" />

|LEB128|2進数|10進数|
|--|--|--|
|$01000001$|$1000001$|$65$|
|$10000010\ 00000001$|$0000010\ 0000001$|$132$|

---

# 疑問
<div class="h-2" />

Wasmではすべての整数がLEB128でエンコードされている

- x86などではこういうことはしていない (それはそう)
- Wasmではサイズ削減のためにこういう圧縮をしているらしい
  - どの程度効果があるのか？

---

# 考える①

`u32` の場合

`u32` (4byte) はLEB128では1~5byteで表される


- 4バイト必要: $2^{28} - 2^{21}$ 個 (6.2%)
- 5バイト必要: $2^{32} - 2^{28}$ 個 (93.75%)

等確率だとすると効率は悪そう

---

# 考える②
<div class="h-2" />

- 大きい数をハードコーディングする機会は少なそう
- Wasmでは関数や変数などがインデックス (0,1,2...) によって表されている
  - これらも同様にLEB128でエンコードされている

---

# 確認してみる
<div class="h-2" />

適当なWasmを用意して、中に含まれている数を数えればよさそう

ちょうどデコーダーが手元にあるので、簡単にできる

---

# 計測
<div class="h-2" />

自作WasmランタイムをWasmにコンパイルし、自作Wasmランタイムでデコードする

```
[11043, 215, 25, 36, 1378]
```

<br>

- $0 \sim 127$ (`u32` 全体の数千万分の1) が大半を占めている
- 合計サイズ: 18KB
  - `u32` で表すとすると: 50KB
  - かなり効果的な圧縮ができていそう

---

# :thinking:
<div class="h-2" />

- この数値データ (18KB) って全体のなかでどのぐらいの割合なの？
  - (デコードできている範囲のうち) 40%ぐらい
  - (全体だと) 2%ぐらい
- デコードできている部分少なくない？
  - Custom Sectionがバイナリの大部分を占めている
- じゃあそのCustom Sectionには何が入ってるの？
  - わからん
  - デバッグ情報はそこまで入ってないと思うので、WASIとかの情報？

---

# もうちょっとまともな計測
<div class="h-2" />

インターネット上で適当に見つけたプロジェクト[^1]で試してみる

- バイナリのサイズ: 5338KB
- うちCustom Section: 3300KBぐらい (62%)



計測結果: `[442351, 34529, 406, 379, 76859]`


- `u32` だと 21500KB ぐらい必要だったのが、820KBぐらいで表現できている (62% 削減)
- Custom Sectionに数値が一切含まれていなくても全体で 20% の削減になっている


<br>

[^1]: https://github.com/jetli/rust-yew-realworld-example-app

---

# まとめ
<div class="h-2" />

- 
