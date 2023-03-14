---
tags:
  - ブロックチェーン
  - プログラミングビットコイン
  - 技術書
  - Zig
---

# プログラミングビットコインを Zig でやってみる - 2 章 楕円曲線

<script src="https://cdnjs.cloudflare.com/ajax/libs/mathjax/2.7.4/latest.js?config=AM_CHTML"></script>

[前回](https://blog.mizuochik.dev/20230310-programming-bitcoin-in-zig-chapter-1)に引き続き、[プログラミングビットコイン](https://www.oreilly.co.jp/books/9784873119021/) 2 章を読み進めて行きます。

内容の要約と、Zig のコードを載せていきます。

## 2 章 楕円曲線

今回は楕円曲線です。楕円曲線暗号の基礎となる内容です。

### 楕円曲線の定義

楕円曲線は、以下の式により定義される曲線です。

<div class="math">
  `y^2 = x^3 + ax + b`
</div>

定数 \`a\` と \`b\` の値によって、以下のように様々な形状を描きます。

<img src="https://upload.wikimedia.org/wikipedia/commons/d/db/EllipticCurveCatalog.svg" width="100%">
（[Wikipedia](https://en.wikipedia.org/wiki/Elliptic_curve) より引用）

楕円どころか、円にさえ見えないようなパターンもありますが、いずれも楕円曲線と呼ばれます。

### 楕円曲線上の点の加算

#### 加算の定義

楕円曲線の面白い性質は、楕円曲線上の点について、加算を定義できることです。

楕円曲線における加算は、2 つの点から同じ曲線上に 3 つめの点を得る演算として定義されます。

任意の点 \`A\` と点 \`B\` の加算結果 \`A+B\` は、以下のように得ます。

1. \`A\` と \`B\` を通る直線を引く
2. 直線が曲線と交わる第 3 の点 \`C\` を得る
3. 点 \`C\` を x 軸について対称にした点を \`A+B\` とする

図示すると、点 \`A\`, \`B\`, \`C\`, \`A+B\` それぞれの関係は以下です。

<img src="https://mizuochik-dev-blog.s3.ap-northeast-1.amazonaws.com/elliptic-curve-add.png" width="75%">

楕円曲線上の任意の点の集合と、上記の加算は

1. 結合法則 \`(A + B) + C = A + (B + C)\`
2. 同一性 \`A + I = A\`
3. 可逆性 \`A + (-A) = I\`

をいずれも満たすことが分かっており、整数における加算などと同様の振る舞いを持ちます。

#### 楕円曲線における単位元 = 無限遠点

尚、楕円曲線における単位元 \`I\` （整数における \`0\`）は、無限遠点と呼ばれ、以下のように x が同一の 2 点を結んだ直線上のことです。
前述の図の場合と異なり、2 点を結んだ直線が再び楕円曲線に交わることがありません。無限に遠い位置にある点と捉えられるため、無限遠点と呼ばれます。

<img src="https://mizuochik-dev-blog.s3.ap-northeast-1.amazonaws.com/elliptic-curve-point-at-infinity.png" width="75%">

### 楕円曲線をコーディング

上記を踏まえて Zig で楕円曲線をコーディングしていきます。

#### 点のコーディング

まずは楕円曲線上の点を `Point` 構造体として定義します。

##### フィールド定義

構造体のフィールドは以下のようになるでしょう。

```
pub const Point = struct {
    x: ?f32,
    y: ?f32,
    a: f32,
    b: f32,
}
```

構造体のフィールドは以上のように、楕円曲線の形状を決める、`a`, `b` 及び 点の位置を示す `x`, `y` が必要です。

`x` と `y` は型名に `?` マークがついています。これは Optional 型を示し、`?<ValueType>` と書きます。null でありうる型を示します。
前述の無限遠点をプログラムで表現するにあたり、無限遠点を、`x=null`, `y=null` の `Point` 値として扱いたいので、Optional 型を用います。

##### コンストラクタによるバリデーション

構造体 `Point` は「楕円曲線上の」点を表す構造体です。`x`, `y` が楕円曲線上の座標を必ず指していないとおかしなことになります。`Point` 値の生成時にバリデーションできると良いでしょう。

バリデーションのため、コンストラクタ関数 `new()` を定義します。

```
// ※3
const Error = error{
    NotOnCurve,
};

pub const Point = struct {
    :
    pub fn new(x: ?f32, y: ?f32, a: f32, b: f32) Error!Point {
        const p = Point{
            .x = x,
            .y = y,
            .a = a,
            .b = b,
        };

        // ※1
        if (x == null and y == null)
            return p;

        // ※2
        if (math.pow(f32, y.?, 2) != math.pow(f32, x.?, 3) + a * x.? + b)
            return Error.NotOnCurve;

        return p;
    }
    :
}
```

※1 は無限遠点の場合です。特にバリデーションせずに、`Point` 値を返します。

※2 がバリデーション部です。 \`y^2 = x^3 + ax + b\` が成り立つことを確認することで、点が曲線状に存在するかチェックします。`x.?` のような `.?` 記法は Optional 型の値の中身を取り出す記法です。

また、等式が成立しない場合にエラー値を返します。Zig において、エラー値は※3 のように定義します。
エラー値でありうる型は、`<ErrorType>!<ValueType>` として表現し、Error Union 型と呼びます。
Optional 型が `?<ValueType>` であったのに対し、Error Union 型は `<ErrorType>!<ValueType>` と対応しています。

ここで定義した `Point.new()` の戻り値の型は、`Error!Point` です。これは、「`Error` でありうる、`Point` 型」という意味です。

#### 点の加算のコーディング

点の定義ができたので、続いて点の加算をコーディングしていきます。

##### `add()` 関数の準備

`add()` 関数を定義します。まず、引数が無限遠点の場合を処理しておきます。

```
pub const Point = struct {
    :
    pub fn add(self: *const Point, other: *const Point) Point {
        std.debug.assert(self.a == other.a and self.b == other.b);
        if (self.x == null)
            return other.*;
        if (other.x == null)
            return self.*;
    }
    :
};
```

加算の片側が無限遠点の場合は、プログラム上では `self.x == null` もしくは、`other.x == null` に相当します。無限遠点は楕円曲線上において単位元 \`0\` として振る舞います。つまり、\`A + 0 = A, 0 + B = B\` に相当する処理を実装しました。

##### 点の加算のパターン

上記の `add()` 関数を拡張していきます。

まず、点\`P_1 (x_1, y_1)\` と点 \`P_2 (x_2, y_2)\` の加算は、以下の 4 つのパターンに分類できます。

1. \`x_1 = x_2\` かつ \`y_1 != y_2\` のとき
2. \`x_1 != x_2\` かつ \`y_1 != y_2\` のとき
3. \`x_1 = x_2\` かつ \`y_1 = y_2 != 0\` のとき
4. \`x_1 = x_2\` かつ \`y_1 = y_2 = 0\` のとき

1 から 4 を、それぞれを図示すると以下の通りです。

<div style="display: grid; grid-template-columns: 50% 50%">
  <div style="position: relative">
    <div style="font-size: 125%; z-index: 10; position: absolute; top: 0.25em; left: 0.25em">1. `x_1 = x_2, y_1 != y_2`</div>
    <img style="margin-left: -1em" src="https://mizuochik-dev-blog.s3.ap-northeast-1.amazonaws.com/elliptic-curve-add-1.png" width="100%">
  </div>
  <div style="position: relative">
    <div style="font-size: 125%; z-index: 10; position: absolute; top: 0.25em; left: 0.25em">2. `x_1 != x_2, y_1 != y_2`</div>
    <img style="margin-left: -1em" src="https://mizuochik-dev-blog.s3.ap-northeast-1.amazonaws.com/elliptic-curve-add-2.png" width="100%">
  </div>
  <div style="position: relative">
    <div style="font-size: 125%; z-index: 10; position: absolute; top: 0.25em; left: 0.25em">3. `x_1 = x_2, y_1 = y_2 != 0`</div>
    <img style="margin-left: -1em" src="https://mizuochik-dev-blog.s3.ap-northeast-1.amazonaws.com/elliptic-curve-add-3.png" width="100%">
  </div>
  <div style="position: relative">
    <div style="font-size: 125%; z-index: 10; position: absolute; top: 0.25em; left: 0.25em">4. `x_1 = x_2, y_1 = y_2 = 0`</div>
    <img style="margin-left: -1em" src="https://mizuochik-dev-blog.s3.ap-northeast-1.amazonaws.com/elliptic-curve-add-4.png" width="100%">
  </div>
</div>

1, 4 では、直線が垂直線すなわち加算結果が無限遠点となることが分かります。2 の 3 の違いは、2 点を結ぶ直線が楕円曲線について、接線となるかそうでないかです。接線でない 2 場合は、2 点の交点から簡単に傾きが求められそうです。一方、接線である 3 の場合は、傾きを求めるために微分計算が必要そうであることが推測されます。

以上の 4 つの加算のパターンについて、それぞれコーディングしていきます。

##### 1. \`x_1 = x_2\` かつ \`y_1 != y_2\` のとき

<img src="https://mizuochik-dev-blog.s3.ap-northeast-1.amazonaws.com/elliptic-curve-add-1.png" width="75%">

前述のように、x が同一である 2 点の加算結果は無限遠点です。無限遠点は `x = null`, `y = null` の `Point` 構造体として表すことにしていました。コードは以下の通りです。

```
pub const Point = struct {
    :
    pub fn add(self: *const Point, other: *const Point) Point {
        :
        if (self.x == other.x and self.y != other.y)
            return Point.new(null, null, self.a, self.b) catch unreachable;
        :
    }
    :
}
```

数学的な処理としては簡単ですが、Zig 特有の `catch`, `unreachable` 構文を使っているので説明します。

`Point.new()` 関数はバリデーション処理のために戻り値の型が、`Error!Point` でした。つまり `new()` はエラーでありうる処理です。

Zig における `catch` は Error Union 型の値がエラー値だったときの処理を記述する構文です。Java などの `catch` とは全く異なる文法です。

ここでは無限遠点の生成ということで、`Point` 構造体の `x`, `y` それぞれを `null` とハードコードしており、エラーになり得ません。なので、`catch` に続いて `unreachable` キーワードを書きました。 `unreachable` はその名の通り、到達し得ないこと示すキーワードです。仮に `unreachable` に処理が到達した場合プログラムはクラッシュしますが、この場合のように `catch unreachable` と書くことで、エラーになり得ない意図を示せます。

##### 2. \`x_1 != x_2\` かつ \`y_1 != y_2\` のとき

<img src="https://mizuochik-dev-blog.s3.ap-northeast-1.amazonaws.com/elliptic-curve-add-2.png" width="75%">

まず、傾き \`s\` を求めます。

<div class="math">
  `P_1 = (x_1, y_1), P_2 = (x_2, y_2), P_3 = (x_3, y_3)`<br>
  `P_1 + P_2 = P_3`<br>
</div>
のとき
<div class="math">
  `s = (y_2 - y_1) // (x_2 - x_1)`
</div>

です。

傾き \`s\` より

<div class="math">
  `x_3 = s^2 - x_1 - x_2`<br>
  `y_3 = s(x_1 - x_3) - y_1`
</div>

として加算結果 \`P_3\` を得ます。

以上を `add()` 関数内にコーディングします。

```
const std = @import("std");
const math = std.math;
:
pub const Point = struct {
    :
    pub fn add(self: *const Point, other: *const Point) Point {
        :
        if (self.x != other.x and self.y != other.y) {
            const s = (other.y.? - self.y.?) / (other.x.? - self.x.?);
            const x = math.pow(f32, s, 2) - self.x.? - other.x.?;
            const y = s * (self.x.? - x) - self.y.?;
            return Point.new(x, y, self.a, self.b) catch unreachable;
        }
        :
        unreachable;
    }
    :
};
```

##### 3. \`x_1 = x_2\` かつ \`y_1 = y_2 != 0\` のとき

<img src="https://mizuochik-dev-blog.s3.ap-northeast-1.amazonaws.com/elliptic-curve-add-3.png" width="75%">

\`P_1, P_2\` が互いに同一であるパターンです。2 点を結ぶ直線というよりは、楕円曲線上の 2 点の箇所について接線となります。 2.の場合と同様にまず傾き \`s\` を求めます。

<div class="math">
  `P_1 = (x_1, y_1), P_2 = (x_2, y_2), P_3 = (x_3, y_3)`<br>
  `P_1 + P_2 = P_3`<br>
</div>
のとき
<div class="math">
  `s = (3x_1^2 + a) // 2y_1`
</div>

です。この \`s\` の式は、楕円曲線の式 \`y^2 = x^3 + ax + b\` の両辺を微分することで得られます。

2.の場合と同様に

<div class="math">
  `x_3 = s^2 - x_1 - x_2`<br>
  `y_3 = s(x_1 - x_3) - y_1`
</div>

式より加算結果 \`P_3\` を得ます。

コードは以下です。

```
pub const Point = struct {
    :
    pub fn add(self: *const Point, other: *const Point) Point {
        :
        if (self.x == other.x and self.y == other.y and self.y.? != 0) {
            const s = (3 * math.pow(f32, other.x.?, 2) + self.a) / (2 * self.y.?);
            const x = math.pow(f32, s, 2) - 2 * self.x.?;
            const y = s * (self.x.? - x) - self.y.?;
            return Point.new(x, y, self.a, self.b) catch unreachable;
        }
        :
        unreachable;
    }
    :
};
```

##### 4. \`x_1 = x_2\` かつ \`y_1 = y_2 = 0\` のとき

<img src="https://mizuochik-dev-blog.s3.ap-northeast-1.amazonaws.com/elliptic-curve-add-4.png" width="75%">

3 の例外的な場合とも言えます。接線が垂直線となりました。垂直線の場合、楕円曲線に交わることはないので、無限遠点です。1 の場合と同じく `x = null`, `y = null` の `Point` 構造体を返却すれば良いです。

コードは以下です。

```
pub const Point = struct {
    :
    pub fn add(self: *const Point, other: *const Point) Point {
        :
        if (self.x == other.x and self.y.? == 0 and other.y.? == 0)
            return Point.new(null, null, self.a, self.b) catch unreachable;
        :
    }
    :
};
```

#### コードまとめ

ここまでで、点の定義と加算処理をそれぞれコーディングしました。コードをまとめると以下の通りです。

```
const std = @import("std");
const math = std.math;

const Error = error{
    NotOnCurve,
};

pub const Point = struct {
    x: ?f32,
    y: ?f32,
    a: f32,
    b: f32,

    pub fn new(x: ?f32, y: ?f32, a: f32, b: f32) Error!Point {
        const p = Point{
            .x = x,
            .y = y,
            .a = a,
            .b = b,
        };
        if (x == null and y == null)
            return p;
        if (math.pow(f32, y.?, 2) != math.pow(f32, x.?, 3) + a * x.? + b)
            return Error.NotOnCurve;
        return p;
    }

    pub fn add(self: *const Point, other: *const Point) Point {
        std.debug.assert(self.a == other.a and self.b == other.b);
        if (self.x == null)
            return other.*;
        if (other.x == null)
            return self.*;
        if (self.x == other.x and self.y != other.y)
            return Point.new(null, null, self.a, self.b) catch unreachable;
        if (self.x != other.x and self.y != other.y) {
            const s = (other.y.? - self.y.?) / (other.x.? - self.x.?);
            const x = math.pow(f32, s, 2) - self.x.? - other.x.?;
            const y = s * (self.x.? - x) - self.y.?;
            return Point.new(x, y, self.a, self.b) catch unreachable;
        }
        if (self.x == other.x and self.y == other.y and self.y.? != 0) {
            const s = (3 * math.pow(f32, other.x.?, 2) + self.a) / (2 * self.y.?);
            const x = math.pow(f32, s, 2) - 2 * self.x.?;
            const y = s * (self.x.? - x) - self.y.?;
            return Point.new(x, y, self.a, self.b) catch unreachable;
        }
        if (self.x == other.x and self.y.? == 0 and other.y.? == 0)
            return Point.new(null, null, self.a, self.b) catch unreachable;
        unreachable;
    }
};
```

GitHub にもアップロードしています。<br>
[elliptic_curve.zig](https://github.com/mizuochik/programming-bitcoin-in-zig/blob/main/src/elliptic_curve.zig)

プログラミングビットコイン 第 2 章 楕円曲線はここまでです。

### まとめ

プログラミングビットコイン 第 2 章 楕円曲線の内容を学習しました。楕円曲線上の点についての加算について学習し、Zig でコーディングしました。

また、Zig によるバリデーションの実装にあたり、Optional 型と Error Union の扱いを説明しました。

次回は、3 章 楕円曲線暗号です。今回の楕円曲線と前回の有限体が組み合わされる、暗号技術のキモといえる内容になるでしょう。
