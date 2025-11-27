---
title: '読書メモ [作って学ぶ]OSのしくみⅠ'
tags:
  - Book
  - Rust
  - OS
private: false
updated_at: '2025-07-30T08:04:32+09:00'
id: 89cf824705313f587197
organization_url_name: null
slide: false
ignorePublish: false
---
# はじめに

<!-- 今までOSが何をしているかなんとなくも理解していなかったが、
「ハードウェアの制御と抽象化」「資源の分配」という大枠で
なんとなく捉えることが出来た気がする、、、
ゼロから組み上げていくことで理解もしやすかった。
その分、黙々とコードを写す時間は忍耐の時間であった。 -->

https://gihyo.jp/book/2025/978-4-297-14859-1

以前より少し気になっていた本を読んだ（読んでいる途中）ので記録します。
Rustをちゃんと使えるようになりたい、OSを理解したいと考えこの本を手に取りました。
初投稿につき、何卒ご容赦。ツッコミ所があればぜひ。

# 第2章
「Hello World!」を出力するためにはOSの力が必要。それを自分で組み立ててやってみる章。
コンピュータの中では文字も画像もバイナリだというところから、ファームウェア（ハードウェアに内蔵されたソフトウェア）の力を借り「Helo World!」、その後、自力で「Hello World!」する。
途中、バイナリでプログラムを動かしたがイマイチ理解出来なかった、、、

### OSなしで動くプログラム
普段はプログラムをOSの上で動かす。ただ、OSを作るためにOSを使うわけにはいかないので、OSなしで動くプログラムを書く必要がある。ここで、ファームウェアの1種であるUEFIが登場。普段、メモリにプログラムを展開するという作業を、UEFIに代わりにしてもらうことでOSをメモリ上に展開する。RustのビルドターゲットをUEFIにすることでRustでOSを作り始めることが出来る！
```
cargo build --target x86_64-unknown-uefi
```

### ベアメタル
ハードウェアが提供する機能のみを利用できる環境をベアメタルという。"bare"（裸の、むき出しの）"metal"（金属）。
`println!`はstdクレート内で定義されており、辿っていくと最終的に分岐へと辿り着きWindowsやUEFIへと処理を渡す。下記は辿った際のメモ。マクロの内容を辿ったことはなかったので面白かった。
||||
|-----|-----|-----|
|rust/library/std/src/macros.rs|`println`||
|rust/library/std/src/io/mod.rs|`pub use self::stdio::{_eprint, _print};`||
|rust/library/std/src/io/stdio.rs|`_print`||
|同上|`print_to`||
|同上|`stdout`|stdout_rawのインスタンスを1つだけに制限
|同上|`stdout_raw`||
|rust/library/std/src/sys/mod.rs|`pub use pal::*;`|再エクスポート|
|rust/library/std/src/sys/pal/mod.rs|`cfg_if::cfg_if! {`|各プラットフォームへの分岐|
|rust/library/std/src/sys/pal/unix/stdio.rs|`impl Stdin {`|unix環境の定義|

これで`println!`がuefiへと繋がっていることが確認出来た。stdクレートを使うとUEFIやOSから逃れることは出来ないため、coreクレートを今後は使う。coreクレートはstdクレートからCPUとメモリだけで実現できる処理を切り出したもの。以下のように書くことでstdを使わずにプログラムを書くことが出来る。
```
#![no_std]
#![no_main]

#![no_mangle]
fn efi_main() {
  // println!("Hello, world!");  // println!は使えなくなった。
  loop{}
}

#[panic_handler]
fn panic(_info: &PanicInfo) -> ! {
  loop{}
}
```

> 関係ないが、`cd -`を初めて知った。ハイフンは前回いた場所を意味するらしい。
特殊変数の`$_`と合わせて覚えて日常的に使えるようにしたい。

後は、UEFIから画面上のピクセルを表すメモリマップを取得してそれを変化させることで描画を行う。書かなければいけないコードはUEFI関連の機能を借りたりメモリのやり取りと、画面描画関連が大部分だった。


# 3章
`unimplemented!("読み進め次第、書いていく。")`



