---
tags:
  - ブロックチェーン
  - プログラミングビットコイン
  - 技術書
  - Zig
---

# プログラミングビットコインを Zig でやってみる - 1 章 有限体

<script src="https://cdnjs.cloudflare.com/ajax/libs/mathjax/2.7.4/latest.js?config=AM_CHTML"></script>

ブロックチェーン技術の基礎習得を目的に、[プログラミングビットコイン](https://www.oreilly.co.jp/books/9784873119021/)を読んでみようと思います。

実装したコードと読書ログを 1 章ずつ本ブログに載せていき、最後にまとめの記事を投稿する予定です。

若干特殊な点として、Zig 言語を使って進めます。

本書を購入する方、もしくは Zig に興味のある方の参考になると良いなと思います。

## 書籍 プログラミングビットコイン

日本では 2020 年に発売された書籍です。ビットコインに関する様々なアルゴリズムデータ構造を Python で作ってみようという本です。
Amazon はじめ各書店サイトでも高評価のようです。
章立ては以下の通りとなっており、面白そうです。

- 1 章 有限体
- 2 章 楕円曲線
- 3 章 楕円曲線暗号
- 4 章 シリアライズ
- 5 章 トランザクション
- 6 章 Script
- 7 章 トランザクションの作成と検証
- 8 章 Pay-to-script-hash
- 9 章 ブロック
- 10 章 ネットワーキング
- 11 章 SPV (Simplified Payment Verification)
- 12 章 ブルームフィルター
- 13 章 Segwit
- 14 章 応用トピックと次のステップ

## Zig 言語

本ブログでは、プログラミングビットコインを Python ではなく Zig を用いて読み進めます。

言語仕様を眺めたり簡単なプログラミングに Zig を使ってみたところ、可能性を感じました。
シンプルな言語でありながら、コーディングしやすく、かつ最速レベルの実行効率を実現できます。
一方、現代的な言語には珍しく、メモリ管理が手動であり、車でいうと最新型のマニュアル車のようです。

クリプトやエッジコンピューティング領域において WASM の勢いが増していますが、Zig は WASM に対応できる言語のひとつです。Zig をやっておくと将来的に WASM 周りの学習にも役立てられそうです。

私は、普段は Go や Python を書いているのですが、上記のような背景から Zig でやってみようと思います。

尚、Zig の文法について、他言語ユーザーの方が雰囲気で分からなそうなところのみ適宜説明します。

## 第 1 章 有限体

ここから読書ログです。要約しながら Zig のコードを載せていきます。

第 1 章は有限体です。有限体とは、数学（代数学）の概念ですね。ブロックチェーン技術の基礎として楕円曲線暗号がありますが、有限体は楕円曲線暗号の基礎です。

### 有限体を実装するには

有限体とは、ざっくりいうと有限個の要素を持つ集合（有限集合）について、四則演算（\`+\`, \`-\`, \`\*\`, \`//\`）が定義された構造です。

有限体を、Zig などのプログラムで実装するには、集合の「要素」と「四則演算」をそれぞれデータ構造及び関数として実装すれば良いです。

### 有限集合の要素

有限集合 \`F_p\` は、位数 \`p\` によって以下となります。

<div class="math">
  `F_p = {0, 1, 2, ... , p-1}`
</div>

まずは、上記有限集合の要素を実装します。

```
pub const FieldElement = struct {
    num: u32,
    prime: u32,
}
```

`FieldElement` 構造体の値が有限集合の要素に対応します。2 つのフィールドがあり、`num` は要素の番号、`prime` は位数です。例えば、値 `FieldElement { .num = 3, .prime = 7 }` は、有限集合 \`F_7\` の要素 \`3\` に対応します。

### 基本関数

後述の四則演算の実装前に、等値比較と文字列化のための関数、`eql()`, `format()` をそれぞれ実装します。テストやデバッグに便利なためです。

```
const std = @import("std");

pub const FieldElement = struct {
    :
    pub fn eql(self: *const FieldElement, other: *const FieldElement) bool {
        return self.num == other.num and self.prime == other.prime;
    }

    pub fn format(self: *const FieldElement, comptime _: []const u8, _: std.fmt.FormatOptions, writer: anytype) !void {
        try writer.print("FieldElement_{d}({d})", .{ self.prime, self.num });
    }
    :
};
```

Zig では、struct 中に関数定義を書くことで、いわゆるメソッドを定義できます。Python のように第一引数が `self` です。また、引数には const ポインタを使っています。引数の内容を書き換えないので const ポインタです。

`format()` は若干複雑な関数シグネチャです。Zig において特別な意味を持つ関数シグネチャです。Zig の標準ライブラリ関数 `std.fmt.format()` は構造体を文字列化するとき、本シグネチャが定義されていると特別に解釈してくれます。Zig 特有の言語機能であるコンパイル時ダックタイピングが活用されています。

### 加算

上記で定義した有限集合の要素について、四則演算を実装していきます。

まずは加算を実装します。

```
pub const FieldElement = struct {
    :
    pub fn add(self: *const FieldElement, other: *const FieldElement) FieldElement {
        std.debug.assert(self.prime == other.prime);
        return FieldElement{
            .num = (self.num + other.num) % self.prime,
            .prime = self.prime,
        };
    }
    :
}
```

`std.debug.assert()` で右左辺の位数が一致することをバリデーションしています。2 要素の位数が異なる場合は、そもそも異なる有限体上の要素となってしまい、演算が定義されないからです。

num フィールドの計算 `.num = (self.num + other.num) % self.prime` が有限体ならではのロジックです。
有限体における加算はモジュロ（\`%\`）演算により定義できます。モジュロ演算とは、左辺を右辺で割ったときの余りを求める演算で、例えば \`5 % 3 = 2\` です。FizzBuzz プログラムでおなじみですね。モジュロ演算により、何度、要素を加算をしても有限体の 1 要素であり続けるのがポイントです。

### 減算・乗算

減算・乗算についても加算とほぼ同様に実装できます。

```
pub const FieldElement = struct {
    :
    pub fn sub(self: *const FieldElement, other: *const FieldElement) FieldElement {
        std.debug.assert(self.prime == other.prime);
        return FieldElement{
            .num = (self.num + self.prime - other.num) % self.prime,
            .prime = self.prime,
        };
    }

    pub fn mul(self: *const FieldElement, other: *const FieldElement) FieldElement {
        std.debug.assert(self.prime == other.prime);
        return FieldElement{
            .num = self.num * other.num % self.prime,
            .prime = self.prime,
        };
    }
    :
}
```

### 除算

除算は若干理解が難しいです。

除算は、乗算の逆算です。有限体上の乗算を \`\*\_f\` 、除算を \`//\_f\` の記号で表します。
例えば有限体 \`F_19\` について

<div class="math">
  `3 \*\_f 7 = 21 % 19 = 2`
</div>
より
<div class="math">
  `2 //\_f 7 = 3`
</div>

となります。

ここで面白いのは、除算 \`2 //\_f 7 = 3\` を、乗算 \`3 \*\_f 7 = 2\` を知らずに求められることです。
フェルマーの小定理を用いると、除算に対応する乗算が分からなくても除算結果を得られます。

フェルマーの小定理は以下です。

<div class="math">
  `1 = n^(p-1)%p`
</div>

フェルマーの小定理より、有限集合の要素 \`b\` について

<div class="math">
  `b^(p-1) = 1`
</div>
<div class="math">
  `b^-1 = b^-1 *_f 1 = b^-1 *_f b^(p-1) = b^(p-2)`
</div>
なので
<div class="math">
  `a //_f b = a *_f b^-1 = a *_f b^(p-2)`
</div>

となり、有限体上での除算結果を得られます。

上記を Zig で実装します。

#### 冪乗の計算

冪乗の計算が必要なので、冪乗 `pow()` を実装します。

```
    :
    pub fn pow(self: *const FieldElement, exponent: i32) FieldElement {
        var a: u32 = 1;

        // ※1
        const exp = @mod(exponent, @intCast(i32, self.prime - 1));

        // ※2
        var cnt: i32 = 0;
        while (cnt < exp) : (cnt += 1) {
            a *= self.num;
            a %= self.prime;
        }

        return FieldElement{
            .prime = self.prime,
            .num = a,
        };
    }
    :
```

※1 有限体においては \`b^(p-1) = 1\` であることから、冪指数について \`p-1\` で余りをとっても等価です。  
※2 Zig では、他の言語によくある、`for (i = 0; i < N; i++)` といった構文がありません。変数初期化文と while 文を使って、このように書く必要があります。

#### 除算の計算

最後に、`pow()` を用いて \`a \*\_f b^(p-2)\` を実装し、除算とします。

```
    :
    pub fn div(self: *const FieldElement, other: *const FieldElement) FieldElement {
        std.debug.assert(self.prime == other.prime);
        return self.mul(&other.pow(@intCast(i32, self.prime - 2)));
    }
    :
```

## まとめ

以上より、有限集合の要素及びその四則演算を Zig で実装し、有限体の実装としました。

実装したコードを、[GitHub に公開](https://github.com/mizuochik/programming-bitcoin-in-zig/blob/main/src/field.zig)しています。

プログラミングビットコインの第 1 章 有限体はここまでです。次回は 2 章の楕円曲線を進めます。どのように実装した有限体が活用されていくのか楽しみです。

Zig は、Python に比べると流石に若干周りくどいですが、特に問題なく使えそうです。こちらも楽しみです。
