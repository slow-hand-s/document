---
created: 2026-05-24T21:26
updated: 2026-05-24T21:30
---
# C# 完全基礎ハンズオン

> 対象：他言語経験者で C# が初めての人  
> 最後まで進めると、API チュートリアルで詰まらない C# の基礎が身につく。  
> すぐ試せる環境：https://dotnetfiddle.net（ブラウザのみ、インストール不要）

---

## 目次

1. [.NET と C# の全体像](#1-net-と-c-の全体像)
2. [最初のプログラムを動かす](#2-最初のプログラムを動かす)
3. [変数と基本データ型](#3-変数と基本データ型)
4. [演算子](#4-演算子)
5. [文字列操作](#5-文字列操作)
6. [制御フロー](#6-制御フロー)
7. [メソッドの定義と呼び出し](#7-メソッドの定義と呼び出し)
8. [クラスとオブジェクト](#8-クラスとオブジェクト)
9. [継承と多態性](#9-継承と多態性)
10. [インターフェース](#10-インターフェース)
11. [コレクション](#11-コレクション)
12. [null 安全](#12-null-安全)
13. [例外処理](#13-例外処理)
14. [ジェネリクス](#14-ジェネリクス)
15. [ラムダ式とデリゲート](#15-ラムダ式とデリゲート)
16. [LINQ](#16-linq)
17. [async / await](#17-async--await)
18. [record 型](#18-record-型)
19. [パターンマッチング](#19-パターンマッチング)
20. [enum](#20-enum)
21. [拡張メソッド](#21-拡張メソッド)
22. [よく使う省略記法まとめ](#22-よく使う省略記法まとめ)

---

## 1. .NET と C# の全体像

コードを書く前に、自分が何を使っているかを理解しておく。

---

### 1-1. C# と .NET の関係

```
C#（言語）
  └─ .NET（実行基盤・クラスライブラリ）
       ├─ ASP.NET Core（Web / API）
       ├─ WPF / MAUI（デスクトップ / モバイル）
       ├─ Unity（ゲーム）
       └─ Console App / Worker Service（CLIツール / バッチ）
```

| 用語 | 説明 |
|------|------|
| **C#** | プログラミング言語。Microsoft 製。Java/TypeScript に近い |
| **.NET** | C# コードを動かす実行環境 + 標準ライブラリの総称 |
| **CLR** | Common Language Runtime。C# コードを実際に実行するエンジン |
| **NuGet** | .NET のパッケージマネージャー（npm / pip 相当） |
| **SDK** | `dotnet` コマンドが使えるようになる開発ツール一式 |

---

### 1-2. コードが動くまでの流れ

```
C# ソースコード (.cs)
        ↓  コンパイル（dotnet build）
  中間言語 IL (.dll)
        ↓  JIT コンパイル（実行時）
   ネイティブコード
        ↓
      実行
```

**他言語との比較**：

| | C# | Java | Python | TypeScript |
|--|----|----|-----|----|
| 型付け | 静的 | 静的 | 動的 | 静的 |
| 実行方式 | JIT | JIT | インタープリタ | トランスパイル→JS |
| パッケージ管理 | NuGet | Maven/Gradle | pip | npm |
| 主な用途 | Web/デスクトップ/ゲーム | Web/Android | スクリプト/ML | Web |

---

### 1-3. dotnet コマンドの基本

```bash
# プロジェクト作成
dotnet new console -n HelloWorld   # コンソールアプリ
dotnet new webapi -n MyApi         # Web API

# 実行
dotnet run

# ビルド（実行ファイル生成）
dotnet build

# パッケージ追加
dotnet add package Newtonsoft.Json

# パッケージ一覧
dotnet list package
```

---

## 2. 最初のプログラムを動かす

### Step 1: コンソールアプリを作る

```bash
dotnet new console -n CsharpBasics
cd CsharpBasics
dotnet run
# → Hello, World!
```

生成された `Program.cs` を開くと：

```csharp
// Program.cs（.NET 6 以降はトップレベルステートメント形式）
Console.WriteLine("Hello, World!");
```

以前（.NET 5 以前）の書き方も覚えておく：

```csharp
// 旧スタイル（現在も有効、大規模プロジェクトでは見かける）
namespace HelloWorld;

class Program
{
    static void Main(string[] args)
    {
        Console.WriteLine("Hello, World!");
    }
}
```

**新スタイルが動く理由**：コンパイラが `Main` メソッドを自動生成してくれる。
どちらも同じ IL に変換される。

---

### Step 2: Console の基本操作

```csharp
// 出力
Console.WriteLine("改行あり出力");   // 末尾に改行
Console.Write("改行なし");           // 改行なし
Console.Write("出力\n");             // \n で明示的に改行

// 入力（コンソールアプリで使う）
Console.Write("名前を入力: ");
string? input = Console.ReadLine(); // string? が返る（Ctrl+Z で null になる）
Console.WriteLine($"こんにちは、{input}さん！");

// 書式付き出力
Console.WriteLine("{0} + {1} = {2}", 3, 4, 7);  // 番号プレースホルダ
Console.WriteLine($"{3} + {4} = {3 + 4}");        // 文字列補間（推奨）
```

✅ **確認**：`dotnet run` で "Hello, World!" が出ること。`Console.Write` と `WriteLine` の違いを確認する。

---

## 3. 変数と基本データ型

### 3-1. 変数宣言の構文

```csharp
// 型名 変数名 = 値;
int count = 10;

// var: 右辺から型を推論（型が明らかなときに使う）
var name = "Alice"; // string と推論される

// const: 変更不可の定数
const double Pi = 3.14159;
// Pi = 3.0; // コンパイルエラー

// readonly: フィールドの定数（初期化後は変更不可）
// クラスの中で使う（後述）
```

---

### 3-2. 数値型

```csharp
// ── 整数 ──
byte   b  = 255;           // 0〜255（8bit）
short  s  = 32_767;        // 約-3万〜3万（16bit）
int    i  = 2_147_483_647; // 約-21億〜21億（32bit）★最もよく使う
long   l  = 9_223_372_036_854_775_807L; // 約±920京（64bit）

// ── 浮動小数点 ──
float  f  = 3.14f;   // 32bit（精度低い・メモリ節約目的）
double d  = 3.14;    // 64bit（科学計算向け）
decimal m = 3.14m;   // 128bit（高精度）★金額は必ずこれ

// なぜ金額に decimal を使うか
double a = 0.1 + 0.2;
Console.WriteLine(a);         // 0.30000000000000004（誤差が出る）
decimal b = 0.1m + 0.2m;
Console.WriteLine(b);         // 0.3（正確）

// 数値リテラルのアンダースコア（読みやすくするだけ、値に影響なし）
int million = 1_000_000;
long bigNum = 9_223_372_036_854_775_807L;
```

---

### 3-3. 文字列・文字・真偽値・日時

```csharp
// ── 文字列 ──
string s1 = "ダブルクォートで囲む";
string s2 = "";            // 空文字列
string s3 = string.Empty;  // 空文字列（同じ意味、こちらが推奨）

// 特殊文字
string tab     = "列1\t列2";   // タブ
string newline = "行1\n行2";   // 改行
string escaped = "パス: C:\\Users\\alice"; // バックスラッシュのエスケープ

// @（逐語的文字列）: バックスラッシュをエスケープ不要
string path    = @"C:\Users\alice\Documents";
string multiline = @"
    複数行
    そのまま書ける
";

// ── 文字 ──
char c = 'A';                // シングルクォートで1文字
char digit = '5';
Console.WriteLine((int)c);  // 65（Unicode コードポイント）

// ── 真偽値 ──
bool isActive = true;
bool isEmpty  = false;
Console.WriteLine(!isActive); // false（否定）

// ── 日時 ──
DateTime now   = DateTime.Now;       // ローカル時刻
DateTime utcNow = DateTime.UtcNow;  // UTC時刻 ← API開発では常にこれ
DateTime specific = new DateTime(2024, 1, 15, 9, 30, 0);

Console.WriteLine(utcNow.ToString("yyyy-MM-dd HH:mm:ss")); // 2024-01-15 09:30:00
Console.WriteLine(utcNow.AddDays(7));  // 7日後
Console.WriteLine(utcNow.Year);        // 年だけ取り出す
```

---

### 3-4. 型変換

```csharp
// ── 暗黙の変換（安全な変換は自動）──
int i = 42;
long l = i;     // int → long（範囲が広がるので安全）
double d = i;   // int → double（安全）

// ── 明示的キャスト（精度が失われる可能性がある変換）──
double pi  = 3.14159;
int    n   = (int)pi;      // 3（小数点以下切り捨て）

long bigL  = 9_999_999_999L;
int  small = (int)bigL;    // データが欠ける（オーバーフロー）

// ── 文字列 ↔ 数値の変換 ──
string s = "42";

// Parse: 変換失敗すると例外（文字列が数値であると確信できる場合）
int fromParse = int.Parse(s);   // 42
// int.Parse("abc"); // FormatException が発生

// TryParse: 変換失敗しても例外にならない（ユーザー入力など不確かな場合）
if (int.TryParse(s, out int result))
    Console.WriteLine($"変換成功: {result}"); // 42
else
    Console.WriteLine("変換失敗");

bool ok = int.TryParse("abc", out int bad);
Console.WriteLine(ok);  // False
Console.WriteLine(bad); // 0（変換失敗時はデフォルト値）

// 数値 → 文字列
int num = 42;
string str1 = num.ToString();          // "42"
string str2 = num.ToString("D5");      // "00042"（ゼロ埋め5桁）
string str3 = 1234.5m.ToString("C");   // "¥1,235"（通貨書式）
string str4 = 0.75.ToString("P");      // "75.00%"（パーセント）
```

✅ **確認**：`int.Parse("abc")` を実行して例外が発生することを確認する。
`TryParse` を使うと例外が出ずに `false` が返ることを確認する。

---

## 4. 演算子

### 4-1. 算術演算子

```csharp
int a = 10, b = 3;

Console.WriteLine(a + b);   // 13
Console.WriteLine(a - b);   // 7
Console.WriteLine(a * b);   // 30
Console.WriteLine(a / b);   // 3（整数どうしの割り算は小数点切り捨て）
Console.WriteLine(a % b);   // 1（剰余・余り）

// 整数の割り算に注意
Console.WriteLine(10 / 3);      // 3（切り捨て）
Console.WriteLine(10.0 / 3);    // 3.3333...（どちらかを double にする）
Console.WriteLine((double)10 / 3); // 3.3333...（キャストで対処）

// インクリメント・デクリメント
int x = 5;
x++;       // x = 6（後置インクリメント）
++x;       // x = 7（前置インクリメント）
x--;       // x = 6

// 複合代入
x += 10;   // x = x + 10
x -= 2;    // x = x - 2
x *= 3;    // x = x * 3
x /= 4;    // x = x / 4
x %= 5;    // x = x % 5
```

---

### 4-2. 比較演算子・論理演算子

```csharp
int a = 5, b = 10;

// 比較（結果は bool）
Console.WriteLine(a == b);  // False（等しい）
Console.WriteLine(a != b);  // True（等しくない）
Console.WriteLine(a < b);   // True
Console.WriteLine(a > b);   // False
Console.WriteLine(a <= b);  // True
Console.WriteLine(a >= b);  // False

// 論理演算子
bool x = true, y = false;
Console.WriteLine(x && y);  // False（AND: 両方 true のとき true）
Console.WriteLine(x || y);  // True（OR: どちらかが true のとき true）
Console.WriteLine(!x);      // False（NOT: 反転）

// 短絡評価（重要）
// && は左が false なら右を評価しない
// || は左が true なら右を評価しない
string? s = null;
if (s != null && s.Length > 0) // s が null のとき s.Length は評価されない（安全）
    Console.WriteLine(s);

// ビット演算子（低レベル処理・フラグ管理で使う）
int flags = 0b1010; // 10（2進数リテラル）
int mask  = 0b1100; // 12
Console.WriteLine(flags & mask);  // 0b1000 = 8（AND）
Console.WriteLine(flags | mask);  // 0b1110 = 14（OR）
Console.WriteLine(flags ^ mask);  // 0b0110 = 6（XOR）
Console.WriteLine(~flags);        // ビット反転
Console.WriteLine(flags << 1);    // 0b10100 = 20（左シフト）
Console.WriteLine(flags >> 1);    // 0b0101 = 5（右シフト）
```

---

### 4-3. null 関連の演算子

```csharp
string? name = null;

// null 合体演算子（??）: null なら右辺を使う
string display = name ?? "名無し";
Console.WriteLine(display); // 名無し

// null 合体代入（??=）: null のときだけ代入する
name ??= "デフォルト";
Console.WriteLine(name); // デフォルト

// null 条件演算子（?.）: null ならスキップして null を返す
int? len = name?.Length;    // name が null なら null、そうでなければ Length
Console.WriteLine(len);     // 7

// チェーン
string? city = null;
int? zipLen = city?.ToUpper()?.Length; // どこかが null なら全体が null
Console.WriteLine(zipLen ?? -1);       // -1
```

✅ **確認**：`10 / 3` と `10.0 / 3` の結果の違いを確認する。
`null?.Length` が `null` を返すことを確認する。

---

## 5. 文字列操作

### 5-1. 文字列補間と書式指定

```csharp
string name  = "Alice";
int    score = 92;
decimal price = 1980.5m;

// 文字列補間（最も使う）
Console.WriteLine($"名前: {name}, スコア: {score}");

// 書式指定（: のあとに書式を指定）
Console.WriteLine($"価格: {price:C}");     // ¥1,981（通貨）
Console.WriteLine($"価格: {price:F2}");    // 1980.50（小数点2桁）
Console.WriteLine($"割合: {0.753:P1}");   // 75.3%
Console.WriteLine($"番号: {42:D6}");      // 000042（ゼロ埋め）
Console.WriteLine($"日時: {DateTime.Now:yyyy/MM/dd HH:mm}");
```

---

### 5-2. よく使うメソッド

```csharp
string s = "  Hello, World!  ";

// 空白除去
Console.WriteLine(s.Trim());        // "Hello, World!"
Console.WriteLine(s.TrimStart());   // "Hello, World!  "
Console.WriteLine(s.TrimEnd());     // "  Hello, World!"

// 大文字/小文字
Console.WriteLine(s.ToUpper());     // "  HELLO, WORLD!  "
Console.WriteLine(s.ToLower());     // "  hello, world!  "

// 検索
Console.WriteLine(s.Contains("World"));     // True
Console.WriteLine(s.StartsWith("  Hello")); // True
Console.WriteLine(s.EndsWith("!  "));       // True
Console.WriteLine(s.IndexOf("World"));      // 9（見つからなければ -1）

// 置換・分割・結合
Console.WriteLine(s.Replace("World", "C#")); // "  Hello, C#!  "
string[] parts = "a,b,c,d".Split(',');        // ["a","b","c","d"]
string joined  = string.Join(" | ", parts);   // "a | b | c | d"

// 部分文字列
string sub1 = "Hello, World!".Substring(7);      // "World!"
string sub2 = "Hello, World!".Substring(7, 5);   // "World"
string sub3 = "Hello, World!"[7..12];             // "World"（範囲インデックス、C# 8+）

// 長さ
Console.WriteLine("Hello".Length); // 5

// 空文字・null チェック
Console.WriteLine(string.IsNullOrEmpty(""));        // True
Console.WriteLine(string.IsNullOrEmpty(null));       // True
Console.WriteLine(string.IsNullOrWhiteSpace("   ")); // True（空白のみも True）
```

---

### 5-3. StringBuilder（大量の文字列連結）

```csharp
// NG: string は不変なので + で繋ぐと毎回新しいオブジェクトが作られる
// ループの中で繰り返すとメモリを大量消費する
string result = "";
for (int i = 0; i < 1000; i++)
    result += i.ToString(); // 1000個の新しい string オブジェクトが作られる

// OK: StringBuilder はバッファを内部で持ち、最後に一度だけ string に変換する
var sb = new System.Text.StringBuilder();
for (int i = 0; i < 1000; i++)
    sb.Append(i);
string result2 = sb.ToString(); // 最後に一度だけ string に変換

// よく使うメソッド
var sb2 = new System.Text.StringBuilder();
sb2.Append("Hello");
sb2.AppendLine(", World!"); // 末尾に改行も追加
sb2.Insert(0, ">>> ");      // 先頭に挿入
sb2.Replace("Hello", "Hi"); // 置換
Console.WriteLine(sb2.ToString());
```

✅ **確認**：`"Hello, World!"[7..12]` と `Substring(7, 5)` が同じ結果になることを確認する。

---

## 6. 制御フロー

### 6-1. if / else

```csharp
int score = 75;

// 基本形
if (score >= 90)
    Console.WriteLine("優");
else if (score >= 70)
    Console.WriteLine("良");
else if (score >= 50)
    Console.WriteLine("可");
else
    Console.WriteLine("不可");

// 複数行のとき波括弧を使う（1行でも使う習慣をつけると安全）
if (score >= 90)
{
    Console.WriteLine("優");
    Console.WriteLine("よくできました！");
}

// 三項演算子（短い条件分岐）
string grade = score >= 60 ? "合格" : "不合格";
Console.WriteLine(grade); // 合格
```

**ガード節（Guard Clause）**：実務で推奨されるスタイル。
条件を満たさない場合は早めに return/throw してネストを浅く保つ。

```csharp
// NG: ネストが深い
string ProcessName(string? name)
{
    if (name != null)
    {
        if (name.Length > 0)
        {
            if (name.Length <= 50)
            {
                return name.Trim().ToUpper();
            }
        }
    }
    return "";
}

// OK: ガード節で早期リターン（同じロジック、可読性が高い）
string ProcessName(string? name)
{
    if (name is null)      return "";
    if (name.Length == 0)  return "";
    if (name.Length > 50)  return "";
    return name.Trim().ToUpper();
}
```

---

### 6-2. switch 文 と switch 式

```csharp
string status = "Active";

// 従来の switch 文
switch (status)
{
    case "Pending":
        Console.WriteLine("処理待ち");
        break; // break を忘れると次の case も実行される（フォールスルー）
    case "Active":
    case "Running":           // 複数のケースを同じ処理にまとめる
        Console.WriteLine("実行中");
        break;
    case "Done":
        Console.WriteLine("完了");
        break;
    default:
        Console.WriteLine("不明");
        break;
}

// C# 8 以降: switch 式（値を返す・簡潔）
string message = status switch
{
    "Pending" => "処理待ち",
    "Active"  => "実行中",
    "Running" => "実行中",
    "Done"    => "完了",
    _         => "不明"       // _ = default（必ず最後に書く）
};
Console.WriteLine(message);

// 条件を組み合わせる（when ガード）
int n = 42;
string label = n switch
{
    < 0             => "負の数",
    0               => "ゼロ",
    > 0 and < 100   => "1〜99",
    _               => "100以上"
};
Console.WriteLine(label); // 1〜99
```

---

### 6-3. ループ

```csharp
// ── foreach: コレクション走査の基本（インデックスが不要なら常にこれ）──
var fruits = new List<string> { "Apple", "Banana", "Cherry" };
foreach (var fruit in fruits)
    Console.WriteLine(fruit);

// ── for: インデックスが必要なとき ──
for (int i = 0; i < fruits.Count; i++)
    Console.WriteLine($"{i + 1}番目: {fruits[i]}");

// 逆順
for (int i = fruits.Count - 1; i >= 0; i--)
    Console.WriteLine(fruits[i]);

// ── while: 条件が true の間繰り返す ──
int count = 0;
while (count < 3)
{
    Console.WriteLine($"カウント: {count}");
    count++;
}

// ── do-while: 最低1回は実行する ──
int x = 10;
do
{
    Console.WriteLine($"x = {x}");
    x++;
} while (x < 5); // 条件が false でも1回は実行された

// ── break / continue ──
for (int i = 0; i < 10; i++)
{
    if (i == 3) continue; // i=3 をスキップして次へ
    if (i == 6) break;    // i=6 でループを終了
    Console.Write($"{i} "); // 0 1 2 4 5
}
Console.WriteLine();

// ── ネストしたループを一度に抜ける ──
bool found = false;
for (int row = 0; row < 3 && !found; row++)
{
    for (int col = 0; col < 3; col++)
    {
        if (row == 1 && col == 2)
        {
            Console.WriteLine($"見つかった: ({row}, {col})");
            found = true;
            break; // 内側のループを抜ける
        }
    }
}
```

✅ **確認**：`break` と `continue` の動作を確認する。
ガード節パターンの2つのバージョンを書いて同じ結果になることを確認する。

---

## 7. メソッドの定義と呼び出し

### 7-1. 基本的なメソッド

```csharp
// 構文: アクセス修飾子 戻り値の型 メソッド名(引数) { 処理 }

// 引数なし・戻り値なし
void Greet()
{
    Console.WriteLine("こんにちは！");
}

// 引数あり・戻り値あり
int Add(int a, int b)
{
    return a + b;
}

// 式形式メンバー（1行で書ける）
int Multiply(int a, int b) => a * b;
string GetLabel(bool flag) => flag ? "YES" : "NO";

// 呼び出し
Greet();                      // こんにちは！
Console.WriteLine(Add(3, 4)); // 7
Console.WriteLine(Multiply(3, 4)); // 12
```

---

### 7-2. 引数のバリエーション

```csharp
// ── デフォルト引数（省略可能な引数）──
void PrintLine(string text, int repeat = 1, char separator = '-')
{
    for (int i = 0; i < repeat; i++)
        Console.Write(text + separator);
    Console.WriteLine();
}

PrintLine("Hello");            // Hello-
PrintLine("Hello", 3);         // Hello-Hello-Hello-
PrintLine("Hello", 3, '*');    // Hello*Hello*Hello*

// ── 名前付き引数（どの引数か明示する）──
PrintLine(text: "Hi", separator: '=', repeat: 2); // 引数の順序を変えられる

// ── params: 可変長引数 ──
int Sum(params int[] numbers)
{
    int total = 0;
    foreach (var n in numbers)
        total += n;
    return total;
}

Console.WriteLine(Sum(1, 2, 3));        // 6（バラバラに渡せる）
Console.WriteLine(Sum(1, 2, 3, 4, 5)); // 15
int[] arr = { 10, 20, 30 };
Console.WriteLine(Sum(arr));            // 60（配列でも渡せる）

// ── ref / out: 参照渡し ──
// ref: 呼び出し前に初期化が必要、メソッド内で変更が呼び出し側に反映される
void Double(ref int value) { value *= 2; }
int n = 5;
Double(ref n);
Console.WriteLine(n); // 10

// out: 初期化不要、メソッド内で必ず値をセットしなければならない
// TryParse などで使われているパターン
bool TryDivide(int a, int b, out double result)
{
    if (b == 0) { result = 0; return false; }
    result = (double)a / b;
    return true;
}
if (TryDivide(10, 3, out double quotient))
    Console.WriteLine($"結果: {quotient:F2}"); // 3.33
```

---

### 7-3. ローカル関数

```csharp
// メソッドの中にメソッドを定義できる（外からは呼べない）
// 処理をグループ化してコードを整理するときに使う
void ProcessOrders(List<int> orderIds)
{
    foreach (var id in orderIds)
    {
        var result = Validate(id); // ローカル関数を呼ぶ
        Console.WriteLine($"{id}: {result}");
    }

    // ローカル関数の定義（メソッドの中に書く）
    string Validate(int id)
    {
        if (id <= 0)     return "無効なID";
        if (id > 99999)  return "範囲外";
        return "OK";
    }
}

ProcessOrders(new List<int> { 1, 50000, 100000, -1 });
```

✅ **確認**：`params` を使ったメソッドに 0 個・3 個・10 個の引数を渡して動くことを確認する。
`out` を使った `TryDivide` に `b=0` を渡して `false` が返ることを確認する。

---

## 8. クラスとオブジェクト

### 8-1. クラスの定義

```csharp
// クラス = データ（プロパティ）と処理（メソッド）をまとめた設計図
class BankAccount
{
    // ── フィールド（内部状態、外から直接触らせない）──
    private decimal _balance;        // _ プレフィックスはフィールドの慣習
    private readonly string _owner;  // readonly: 一度セットしたら変更不可

    // ── プロパティ（外からアクセスする窓口）──
    public string Owner => _owner;   // 読み取り専用プロパティ
    public decimal Balance => _balance;

    // get/set で独自ロジックを入れられる
    public string DisplayName { get; private set; } = ""; // 外から読める・外から書けない

    // ── コンストラクタ（new するときに呼ばれる）──
    public BankAccount(string owner, decimal initialBalance = 0)
    {
        if (string.IsNullOrWhiteSpace(owner))
            throw new ArgumentException("名前は必須です");
        if (initialBalance < 0)
            throw new ArgumentException("初期残高は0以上にしてください");

        _owner      = owner;
        _balance    = initialBalance;
        DisplayName = $"[{owner}の口座]";
    }

    // ── メソッド ──
    public void Deposit(decimal amount)
    {
        if (amount <= 0) throw new ArgumentException("入金額は0より大きくしてください");
        _balance += amount;
        Console.WriteLine($"{amount:C} 入金。残高: {_balance:C}");
    }

    public bool Withdraw(decimal amount)
    {
        if (amount <= 0) return false;
        if (amount > _balance)
        {
            Console.WriteLine("残高不足");
            return false;
        }
        _balance -= amount;
        Console.WriteLine($"{amount:C} 出金。残高: {_balance:C}");
        return true;
    }

    // ToString のオーバーライド（デバッグ・ログに便利）
    public override string ToString()
        => $"{DisplayName}: 残高 {_balance:C}";
}

// 使い方
var account = new BankAccount("Alice", 10_000m);
Console.WriteLine(account); // [Aliceの口座]: 残高 ¥10,000

account.Deposit(5_000m);    // ¥5,000 入金。残高: ¥15,000
account.Withdraw(3_000m);   // ¥3,000 出金。残高: ¥12,000
account.Withdraw(20_000m);  // 残高不足

// account._balance = 999; // コンパイルエラー（private なのでアクセス不可）
```

---

### 8-2. アクセス修飾子

```csharp
public class Example
{
    public    int A { get; set; } // どこからでもアクセス可
    private   int B { get; set; } // このクラスの内側からのみ
    protected int C { get; set; } // このクラスと継承クラスから
    internal  int D { get; set; } // 同じプロジェクト（アセンブリ）内から
    private protected int E { get; set; } // 同プロジェクト内の継承クラスから

    // 実務でよく使う組み合わせ
    public string Name { get; private set; } = ""; // 外から読める・外から書けない
    public string Code { get; init; } = "";        // 初期化のみ可（後から変更不可）
}
```

---

### 8-3. static（静的メンバー）

```csharp
class Counter
{
    // static フィールド: インスタンスではなくクラス自体に属する
    private static int _totalCount = 0;
    public  static int TotalCount => _totalCount;

    private int _id;

    public Counter()
    {
        _totalCount++;
        _id = _totalCount;
    }

    public override string ToString() => $"Counter #{_id}";

    // static メソッド: インスタンス不要で呼べる
    public static void Reset() => _totalCount = 0;
}

var c1 = new Counter();
var c2 = new Counter();
var c3 = new Counter();
Console.WriteLine(Counter.TotalCount); // 3（全インスタンス共通）
Counter.Reset();
Console.WriteLine(Counter.TotalCount); // 0

// static クラス: インスタンス化できない（ユーティリティに使う）
static class MathUtils
{
    public static double Clamp(double value, double min, double max)
        => Math.Max(min, Math.Min(max, value));
}
Console.WriteLine(MathUtils.Clamp(150, 0, 100)); // 100
```

---

### 8-4. struct（構造体）

```csharp
// struct: 軽量な値型。小さなデータの塊を表すのに使う
// class との違い: 値型なので代入するとコピーされる
struct Point
{
    public double X { get; init; }
    public double Y { get; init; }

    public Point(double x, double y) { X = x; Y = y; }

    public double DistanceTo(Point other)
        => Math.Sqrt(Math.Pow(X - other.X, 2) + Math.Pow(Y - other.Y, 2));

    public override string ToString() => $"({X}, {Y})";
}

var p1 = new Point(0, 0);
var p2 = new Point(3, 4);
Console.WriteLine(p1.DistanceTo(p2)); // 5（3-4-5 の直角三角形）

// コピーの動作確認
var p3 = p1; // 値がコピーされる
// p3.X = 10; // init なので変更不可（コンパイルエラー）
Console.WriteLine(p1); // (0, 0) ← p3 を変えても影響なし
```

✅ **確認**：`BankAccount` で `_balance` に直接アクセスしようとしてコンパイルエラーになることを確認する。
`static TotalCount` が全インスタンスで共有されることを確認する。

---

## 9. 継承と多態性

### 9-1. 継承の基本

```csharp
// 親クラス（基底クラス）
class Shape
{
    public string Color { get; set; } = "Black";

    // virtual: 子クラスで上書き可能
    public virtual double Area() => 0;

    // virtual: 子クラスで上書き可能（ToString は object から継承）
    public override string ToString()
        => $"{GetType().Name}（{Color}）: 面積 = {Area():F2}";
}

// 子クラス: Shape を継承
class Circle : Shape
{
    public double Radius { get; set; }

    public Circle(double radius) { Radius = radius; }

    // override: 親の実装を上書き
    public override double Area() => Math.PI * Radius * Radius;
}

class Rectangle : Shape
{
    public double Width { get; set; }
    public double Height { get; set; }

    public Rectangle(double width, double height)
    {
        Width  = width;
        Height = height;
    }

    public override double Area() => Width * Height;
}

// 使い方
var c = new Circle(5)      { Color = "Red"  };
var r = new Rectangle(4, 6){ Color = "Blue" };
Console.WriteLine(c); // Circle（Red）: 面積 = 78.54
Console.WriteLine(r); // Rectangle（Blue）: 面積 = 24.00

// 多態性: 親クラス型の変数に子クラスのインスタンスを入れられる
Shape[] shapes = { c, r, new Circle(3) };
double totalArea = 0;
foreach (var shape in shapes)
    totalArea += shape.Area(); // それぞれの子クラスの Area() が呼ばれる

Console.WriteLine($"合計面積: {totalArea:F2}"); // 合計面積: 130.83
```

---

### 9-2. base キーワード

```csharp
class Vehicle
{
    public string Make  { get; set; }
    public string Model { get; set; }
    protected int _year;

    public Vehicle(string make, string model, int year)
    {
        Make   = make;
        Model  = model;
        _year  = year;
    }

    public virtual string Describe()
        => $"{_year}年式 {Make} {Model}";
}

class ElectricVehicle : Vehicle
{
    public int BatteryCapacityKwh { get; set; }

    // base(...) で親のコンストラクタを呼ぶ（必須）
    public ElectricVehicle(string make, string model, int year, int battery)
        : base(make, model, year)
    {
        BatteryCapacityKwh = battery;
    }

    // base.メソッド() で親の実装を呼び出して追加情報を付け加える
    public override string Describe()
        => $"{base.Describe()} EV（バッテリー: {BatteryCapacityKwh}kWh）";
}

var tesla = new ElectricVehicle("Tesla", "Model 3", 2024, 75);
Console.WriteLine(tesla.Describe());
// 2024年式 Tesla Model 3 EV（バッテリー: 75kWh）
```

---

### 9-3. abstract クラス

```csharp
// abstract: インスタンスを直接作れない「不完全なクラス」
// → 「継承して使うことを前提にしたクラス」を定義するときに使う
abstract class Animal
{
    public string Name { get; set; }

    protected Animal(string name) { Name = name; }

    // abstract メソッド: 子クラスで必ず実装しなければならない（実装を強制する）
    public abstract string Speak();

    // 通常メソッド: 子クラスがそのまま使える共通実装
    public void Sleep() => Console.WriteLine($"{Name} が寝ている...");
}

class Dog : Animal
{
    public Dog(string name) : base(name) { }

    // abstract は必ず override しなければコンパイルエラー
    public override string Speak() => "ワンワン！";
}

class Cat : Animal
{
    public Cat(string name) : base(name) { }
    public override string Speak() => "ニャー";
}

// Animal a = new Animal("Test"); // コンパイルエラー（abstract はインスタンス化不可）
Animal[] animals = { new Dog("ポチ"), new Cat("タマ") };
foreach (var animal in animals)
{
    Console.WriteLine($"{animal.Name}: {animal.Speak()}");
    animal.Sleep();
}
```

**インターフェース vs 抽象クラスの使い分け**：

```
インターフェース → 「何ができるか」の契約（複数持てる）
抽象クラス      → 「共通の実装を持ちつつ、一部を子クラスに強制する」基底クラス

例:
abstract class Repository<T>  // 共通 CRUD の実装を持つ基底クラス
    ↑ ProductRepository, CategoryRepository が継承する

interface INotifiable          // 「通知できる」という契約
    ↑ EmailService, SlackService, SMSService が実装する
```

✅ **確認**：`shapes` 配列にさらに `Rectangle` を追加して `totalArea` が増えることを確認する。
`abstract Animal` を `new Animal(...)` しようとしてコンパイルエラーになることを確認する。

---

## 10. インターフェース

### 10-1. インターフェースの定義と実装

```csharp
// インターフェース: 「何ができるか」の契約（実装は書かない）
// クラスは複数のインターフェースを実装できる
interface IPayable
{
    decimal Amount { get; }
    bool ProcessPayment();
}

interface IRefundable
{
    bool ProcessRefund(decimal amount);
}

// 複数のインターフェースを実装
class CreditCardPayment : IPayable, IRefundable
{
    public decimal Amount { get; private set; }
    private string _cardNumber;

    public CreditCardPayment(decimal amount, string cardNumber)
    {
        Amount      = amount;
        _cardNumber = cardNumber;
    }

    public bool ProcessPayment()
    {
        Console.WriteLine($"クレジットカード {_cardNumber[^4..]} で {Amount:C} 決済");
        return true;
    }

    public bool ProcessRefund(decimal amount)
    {
        Console.WriteLine($"{amount:C} を返金");
        return true;
    }
}

// 使う側はインターフェース型で受け取る → 実装の詳細を知らなくていい
void Checkout(IPayable payment)
{
    Console.WriteLine($"合計: {payment.Amount:C}");
    var success = payment.ProcessPayment();
    Console.WriteLine(success ? "決済成功" : "決済失敗");
}

var card = new CreditCardPayment(5_000m, "4111-1111-1111-1234");
Checkout(card); // IPayable として渡す

// IRefundable としてもキャストできる
if (card is IRefundable refundable)
    refundable.ProcessRefund(1_000m);
```

---

### 10-2. インターフェースが DI で重要な理由

```csharp
// 支払い方法が増えても Checkout は変更不要
class PayPayPayment : IPayable
{
    public decimal Amount { get; }
    public PayPayPayment(decimal amount) { Amount = amount; }
    public bool ProcessPayment()
    {
        Console.WriteLine($"PayPay で {Amount:C} 決済");
        return true;
    }
}

// Checkout はインターフェースに依存しているので差し替えるだけ
Checkout(new CreditCardPayment(3_000m, "4111-1111-1111-1234"));
Checkout(new PayPayPayment(3_000m));

// テスト用モック（本物の決済 API を呼ばずにテストできる）
class MockPayment : IPayable
{
    public decimal Amount { get; set; }
    public bool ShouldSucceed { get; set; } = true;
    public bool ProcessPayment()
    {
        Console.WriteLine($"[テスト] {Amount:C} 決済シミュレーション");
        return ShouldSucceed;
    }
}
Checkout(new MockPayment { Amount = 1_000m, ShouldSucceed = false });
```

✅ **確認**：`MockPayment` を `Checkout` に渡して動くことを確認する（本物の決済なしにテストできる）。

---

## 11. コレクション

### 11-1. List\<T\>

```csharp
var list = new List<int> { 3, 1, 4, 1, 5, 9, 2, 6 };

// 追加・削除
list.Add(7);               // 末尾に追加
list.Insert(0, 0);         // 先頭に挿入
list.Remove(1);            // 最初に見つかった 1 を削除
list.RemoveAt(0);          // インデックス 0 の要素を削除
list.RemoveAll(n => n < 3); // 条件に合う全要素を削除

// 検索
Console.WriteLine(list.Contains(5));   // True
Console.WriteLine(list.IndexOf(5));    // インデックス（なければ -1）
Console.WriteLine(list.Find(n => n > 7)); // 最初に条件を満たす要素

// 並び替え
list.Sort();                          // 昇順ソート（インプレース）
list.Sort((a, b) => b.CompareTo(a)); // 降順ソート

// 変換・確認
Console.WriteLine(list.Count);        // 要素数
int[] arr = list.ToArray();           // 配列に変換
bool any  = list.Any(n => n > 8);     // 1つでも条件を満たすか
bool all  = list.All(n => n >= 0);    // 全て条件を満たすか
```

---

### 11-2. Dictionary\<TKey, TValue\>

```csharp
var dict = new Dictionary<string, int>
{
    { "Alice", 90 },
    { "Bob",   75 },
    ["Charlie"] = 85  // 別の初期化構文
};

// 追加・更新・削除
dict["Dave"]   = 92;          // 追加（キーがなければ作成）
dict["Alice"]  = 95;          // 更新（キーがあれば上書き）
dict.Remove("Bob");           // 削除

// 取得（KeyNotFoundException を避ける）
if (dict.TryGetValue("Alice", out int score))
    Console.WriteLine($"Alice: {score}"); // Alice: 95

// 存在確認
Console.WriteLine(dict.ContainsKey("Dave"));   // True
Console.WriteLine(dict.ContainsValue(85));     // True

// 走査
foreach (var (name, s) in dict)
    Console.WriteLine($"{name}: {s}");

// キー一覧・値一覧
foreach (var key in dict.Keys)   Console.WriteLine(key);
foreach (var val in dict.Values) Console.WriteLine(val);

// GetValueOrDefault: キーがなくてもデフォルト値を返す（例外なし）
int missing = dict.GetValueOrDefault("Zara", -1); // -1
```

---

### 11-3. HashSet\<T\>

```csharp
// HashSet: 重複なし・順序なし・存在確認が O(1) で高速
var set = new HashSet<string> { "Apple", "Banana", "Cherry" };
set.Add("Apple");      // 重複なので追加されない
set.Add("Date");       // 追加される
Console.WriteLine(set.Count); // 4

Console.WriteLine(set.Contains("Banana")); // True（高速）

// 集合演算
var setA = new HashSet<int> { 1, 2, 3, 4, 5 };
var setB = new HashSet<int> { 3, 4, 5, 6, 7 };

setA.UnionWith(setB);        // A = A ∪ B: { 1, 2, 3, 4, 5, 6, 7 }
setA.IntersectWith(setB);    // A = A ∩ B: { 3, 4, 5, 6, 7 }
setA.ExceptWith(setB);       // A = A - B: { 1, 2 }

// IP ブロックリストなど「存在するかどうかを高速チェックしたい」ときに使う
var blockedIps = new HashSet<string> { "192.168.1.1", "10.0.0.1" };
bool isBlocked = blockedIps.Contains("192.168.1.1"); // 高速
```

---

### 11-4. Queue と Stack

```csharp
// Queue<T>: FIFO（先入れ先出し）
var queue = new Queue<string>();
queue.Enqueue("タスク1");   // 末尾に追加
queue.Enqueue("タスク2");
queue.Enqueue("タスク3");
Console.WriteLine(queue.Dequeue()); // "タスク1"（先頭を取り出す）
Console.WriteLine(queue.Peek());    // "タスク2"（取り出さずに先頭を確認）

// Stack<T>: LIFO（後入れ先出し）
var stack = new Stack<string>();
stack.Push("ページ1");
stack.Push("ページ2");
stack.Push("ページ3");
Console.WriteLine(stack.Pop());  // "ページ3"（最後に積んだものを取り出す）
Console.WriteLine(stack.Peek()); // "ページ2"（取り出さずに確認）
```

---

### 11-5. IEnumerable\<T\> と遅延評価

```csharp
// IEnumerable<T>: 「foreach できるもの」の共通インターフェース
// List, Array, Dictionary.Keys, LINQ クエリ結果など全て IEnumerable

// 読み取り専用で返したいとき IEnumerable を使う
IEnumerable<int> GetNumbers()
{
    yield return 1; // yield return: 一つずつ返す（遅延評価）
    yield return 2;
    yield return 3;
    // このメソッドはイテレータと呼ばれる
}

foreach (var n in GetNumbers())
    Console.Write($"{n} "); // 1 2 3

// 遅延評価: ToList() / foreach を呼ぶまで実際の処理は走らない
var numbers = new List<int> { 1, 2, 3, 4, 5 };
var query = numbers.Where(n => n % 2 == 0); // まだ実行されない
numbers.Add(6);                              // クエリ実行前に追加
var result = query.ToList();                 // ここで実行される
Console.WriteLine(string.Join(", ", result)); // 2, 4, 6（後から追加した 6 も含まれる）
```

✅ **確認**：`HashSet` に同じ値を2回 `Add` して `Count` が増えないことを確認する。
遅延評価の例で `numbers.Add(6)` の位置を `ToList()` の後に移して結果が変わることを確認する。

---

## 12. null 安全

### 12-1. null 許容参照型（Nullable Reference Types）

```csharp
// .csproj に <Nullable>enable</Nullable> があると有効になる（.NET 6 以降デフォルト）

// null を入れられない（コンパイラが保証）
string name = "Alice";
// name = null; // CS8600 警告

// null を入れられる（? をつける）
string? maybeName = null; // OK

// null かどうかチェックせずに使うと警告
// Console.WriteLine(maybeName.Length); // CS8602 警告

// ── 安全に使う方法 ──

// パターン1: null チェック（最も明示的）
if (maybeName is not null)
    Console.WriteLine(maybeName.Length); // ここでは non-null が保証される

// パターン2: null 条件演算子（?.）
int? len = maybeName?.Length; // null なら null が返る

// パターン3: null 合体演算子（??）
int safeLen = maybeName?.Length ?? 0; // null なら 0

// パターン4: null 合体代入（??=）
maybeName ??= "デフォルト";
Console.WriteLine(maybeName); // デフォルト

// null 免除演算子（!）: 自分の責任で「絶対 null じゃない」と宣言
// 使いすぎると null 安全の恩恵が失われるので最小限に
string definitelyNotNull = maybeName!; // 警告を抑制
```

---

### 12-2. 値型の null 許容

```csharp
// int, bool, decimal など値型はデフォルトで null にならない
int count = 0;
// count = null; // コンパイルエラー

// T? で null を許容できる
int? maybeCount = null;

// HasValue / Value でアクセス
Console.WriteLine(maybeCount.HasValue);       // False
// Console.WriteLine(maybeCount.Value);        // InvalidOperationException（null の時）
Console.WriteLine(maybeCount.GetValueOrDefault()); // 0
Console.WriteLine(maybeCount ?? -1);           // -1

maybeCount = 42;
Console.WriteLine(maybeCount.HasValue); // True
Console.WriteLine(maybeCount.Value);    // 42

// 実務での使い道
// クエリパラメータが省略されたことを null で表現する
int? categoryId = null; // GET /api/products（カテゴリ指定なし）
int? withCat    = 3;    // GET /api/products?categoryId=3

if (categoryId.HasValue)
    Console.WriteLine($"カテゴリ {categoryId.Value} でフィルタ");
else
    Console.WriteLine("全カテゴリを表示");
```

✅ **確認**：`null?.Length` が `null` を返すことと、`null ?? 0` が `0` を返すことを確認する。

---

## 13. 例外処理

### 13-1. try / catch / finally

```csharp
void Divide(int a, int b)
{
    try
    {
        int result = a / b; // b=0 のとき DivideByZeroException
        Console.WriteLine($"{a} / {b} = {result}");
    }
    catch (DivideByZeroException)
    {
        // 特定の例外を先に捕まえる（特定 → 汎用の順に書く）
        Console.WriteLine("0で割ることはできません");
    }
    catch (Exception ex)
    {
        // 全ての例外を捕まえる（最後に書く）
        Console.WriteLine($"予期しないエラー: {ex.Message}");
        Console.WriteLine($"スタックトレース:\n{ex.StackTrace}");
    }
    finally
    {
        // 例外の有無に関わらず必ず実行（DB 接続を閉じる・ログ記録など）
        Console.WriteLine("処理終了");
    }
}

Divide(10, 2); // 10 / 2 = 5 → 処理終了
Divide(10, 0); // 0で割ることはできません → 処理終了
```

---

### 13-2. 例外を throw する

```csharp
// throw: 例外を意図的に投げる
void SetAge(int age)
{
    if (age < 0 || age > 150)
        throw new ArgumentOutOfRangeException(nameof(age), "年齢は0〜150の範囲で指定してください");
    Console.WriteLine($"年齢を {age} に設定");
}

try { SetAge(200); }
catch (ArgumentOutOfRangeException ex)
{
    Console.WriteLine(ex.Message);
    // "年齢は0〜150の範囲で指定してください (Parameter 'age')"
}

// 例外を再 throw（catch して処理した後、上に伝える）
void Process(string input)
{
    try
    {
        int n = int.Parse(input);
        Console.WriteLine($"処理: {n}");
    }
    catch (FormatException ex)
    {
        Console.WriteLine($"入力値エラーをログ: {ex.Message}");
        throw; // ← 元の例外をそのまま再 throw（スタックトレースを保持）
        // throw ex; は使わない（スタックトレースがリセットされる）
    }
}
```

---

### 13-3. カスタム例外クラス

```csharp
// API チュートリアルで使うパターン
// Exception を継承して独自の例外を作る

public class NotFoundException : Exception
{
    public NotFoundException(string entityName, object key)
        : base($"{entityName}（ID: {key}）は見つかりません") { }
}

public class ValidationException : Exception
{
    public string Field { get; } // どのフィールドで起きたか情報を持たせる

    public ValidationException(string field, string message)
        : base(message)
    {
        Field = field;
    }
}

// 使い方
void GetProduct(int id)
{
    if (id <= 0)
        throw new ValidationException("id", "ID は1以上を指定してください");

    bool exists = id < 100; // DB から取得のシミュレーション
    if (!exists)
        throw new NotFoundException("Product", id);

    Console.WriteLine($"商品 {id} を取得");
}

try { GetProduct(0); }
catch (ValidationException ex) { Console.WriteLine($"バリデーションエラー: {ex.Message}"); }

try { GetProduct(999); }
catch (NotFoundException ex)   { Console.WriteLine($"404: {ex.Message}"); }
```

---

### 13-4. using 文（リソース自動解放）

```csharp
// IDisposable を実装したリソース（DB 接続・ファイル・ストリームなど）は
// 使い終わったら Dispose() を呼んでリソースを解放する必要がある

// 旧スタイル: using ブロック
using (var reader = new System.IO.StringReader("Hello\nWorld"))
{
    string? line;
    while ((line = reader.ReadLine()) != null)
        Console.WriteLine(line);
} // ← スコープを抜けると自動で Dispose() が呼ばれる

// C# 8 以降の簡潔な書き方: メソッドのスコープで自動解放
using var reader2 = new System.IO.StringReader("Hello");
Console.WriteLine(reader2.ReadLine()); // Hello
// メソッドを抜けると自動で Dispose() が呼ばれる
```

✅ **確認**：`finally` が例外あり・なし両方のケースで実行されることを確認する。
`throw` と `throw ex` の違いを調べる（スタックトレースの保持）。

---

## 14. ジェネリクス

### 14-1. ジェネリクスとは

```csharp
// ジェネリクスなし: 型ごとに同じコードを書く羽目になる
class IntBox
{
    public int Value { get; set; }
}
class StringBox
{
    public string Value { get; set; } = "";
}

// ジェネリクスあり: 型を引数として受け取り、一つのクラスで全ての型に対応
class Box<T>              // <T> が型パラメータ
{
    public T Value { get; set; }

    // デフォルトコンストラクタ
    public Box(T value) { Value = value; }

    // 型情報を使ったメソッド
    public string TypeName() => typeof(T).Name;

    public override string ToString() => $"Box<{TypeName()}>: {Value}";
}

var intBox    = new Box<int>(42);
var strBox    = new Box<string>("Hello");
var dateBox   = new Box<DateTime>(DateTime.Now);

Console.WriteLine(intBox);  // Box<Int32>: 42
Console.WriteLine(strBox);  // Box<String>: Hello
```

---

### 14-2. 型制約（where）

```csharp
// where T : 制約  型 T が満たさなければならない条件を指定する

// class: T は参照型（class, interface, delegate）
void PrintIfNotNull<T>(T? value) where T : class
{
    if (value is not null)
        Console.WriteLine(value);
}

// struct: T は値型（int, decimal, struct）
T DefaultValue<T>() where T : struct => default;
Console.WriteLine(DefaultValue<int>());     // 0
Console.WriteLine(DefaultValue<decimal>()); // 0

// new(): T はパラメータなしコンストラクタを持つ
T CreateNew<T>() where T : new()
{
    return new T(); // new T() が使えるようになる
}

// インターフェース制約
void Process<T>(T item) where T : IComparable<T>
{
    // T が IComparable を実装していることが保証される
    Console.WriteLine(item.CompareTo(item)); // 0（自分自身との比較）
}

// 複数の制約を組み合わせる（API チュートリアルのリポジトリで使う）
interface IEntity { int Id { get; } }

class Repository<T> where T : class, IEntity
{
    // T は class かつ IEntity を実装していることが保証される
}
```

---

### 14-3. ジェネリクスメソッド

```csharp
// クラス全体をジェネリクスにしなくても、メソッドだけジェネリクスにできる
class Converter
{
    public static T? TryConvert<T>(string input) where T : struct
    {
        try { return (T)Convert.ChangeType(input, typeof(T)); }
        catch { return null; }
    }
}

var n = Converter.TryConvert<int>("42");
var d = Converter.TryConvert<double>("3.14");
var f = Converter.TryConvert<int>("abc"); // null（変換失敗）

Console.WriteLine(n ?? -1); // 42
Console.WriteLine(f ?? -1); // -1

// よく使う標準ライブラリのジェネリクスメソッド
var list = new List<int> { 3, 1, 4, 1, 5 };
T? First<T>(IEnumerable<T> source) => source.Any() ? source.First() : default;
```

✅ **確認**：`Box<T>` にいろいろな型（`bool`, `decimal`, 自作クラス）を入れて動くことを確認する。

---

## 15. ラムダ式とデリゲート

### 15-1. デリゲートとは

```csharp
// デリゲート = 「メソッドの型」
// 引数と戻り値の形が同じならどのメソッドでも入れられる

// Func<...>: 戻り値ありのデリゲート（最後の型引数が戻り値）
Func<int, int, int>   add       = (a, b) => a + b;
Func<string, int>     getLength = s => s.Length;
Func<string>          hello     = () => "Hello!";

Console.WriteLine(add(3, 4));          // 7
Console.WriteLine(getLength("Alice")); // 5
Console.WriteLine(hello());            // Hello!

// Action<...>: 戻り値なし（void）のデリゲート
Action<string>        print   = s => Console.WriteLine(s);
Action<string, int>   repeat  = (s, n) => { for(int i=0;i<n;i++) Console.Write(s); };
Action                newLine = () => Console.WriteLine();

print("Hello");       // Hello
repeat("* ", 5);      // * * * * *
newLine();

// Predicate<T>: bool を返すデリゲート（Where などで使う）
Predicate<int>    isEven    = n => n % 2 == 0;
Predicate<string> isLong    = s => s.Length > 5;

Console.WriteLine(isEven(4));            // True
Console.WriteLine(isLong("Hello World")); // True
```

---

### 15-2. ラムダ式の書き方

```csharp
// 基本形: (引数) => 処理

// 引数なし
Func<int> getRandom = () => new Random().Next(100);

// 引数1つ（括弧省略可）
Func<int, bool> isPositive = n => n > 0;

// 引数複数（括弧必須）
Func<int, int, int> max = (a, b) => a > b ? a : b;

// 複数行（波括弧 + return が必要）
Func<int, string> classify = n =>
{
    if (n < 0) return "負";
    if (n == 0) return "ゼロ";
    return "正";
};

Console.WriteLine(isPositive(-5));     // False
Console.WriteLine(max(3, 7));          // 7
Console.WriteLine(classify(-10));      // 負
```

---

### 15-3. メソッドをデリゲートとして渡す

```csharp
// 既存のメソッドもデリゲートとして渡せる
int Add(int a, int b) => a + b;
int Subtract(int a, int b) => a - b;
int Multiply(int a, int b) => a * b;

void Calculate(int x, int y, Func<int, int, int> operation, string opName)
{
    int result = operation(x, y);
    Console.WriteLine($"{x} {opName} {y} = {result}");
}

Calculate(10, 3, Add,      "+");  // 10 + 3 = 13
Calculate(10, 3, Subtract, "-");  // 10 - 3 = 7
Calculate(10, 3, Multiply, "*");  // 10 * 3 = 30
Calculate(10, 3, (a,b) => (int)Math.Pow(a,b), "^"); // 10 ^ 3 = 1000
```

---

### 15-4. クロージャ（変数のキャプチャ）

```csharp
// ラムダ式は外側のスコープの変数を「キャプチャ」できる
int multiplier = 3;
Func<int, int> triple = n => n * multiplier; // multiplier をキャプチャ

Console.WriteLine(triple(5)); // 15

// キャプチャした変数が変わると結果も変わる
multiplier = 10;
Console.WriteLine(triple(5)); // 50（multiplier が変わった！）

// よくあるバグ: ループ変数のキャプチャ
var funcs = new List<Func<int>>();
for (int i = 0; i < 3; i++)
{
    int captured = i; // ループ変数を別変数にコピーしてキャプチャ
    funcs.Add(() => captured); // i をキャプチャするとバグになる
}

foreach (var f in funcs)
    Console.Write(f() + " "); // 0 1 2（正しい）
Console.WriteLine();
```

✅ **確認**：`Calculate` に `(a, b) => a % b` を渡して剰余が計算できることを確認する。
クロージャのバグパターンで `captured = i` を `i` のままにすると全て `3` になることを確認する。

---

## 16. LINQ

### 16-1. LINQ の基本

```csharp
// Language Integrated Query（統合言語クエリ）
// コレクションを SQL ライクに操作できる
// 実体は IEnumerable<T> への拡張メソッド群

record Product(int Id, string Name, decimal Price, string Category, int Stock);

var products = new List<Product>
{
    new(1, "Widget",    980m,  "Electronics", 100),
    new(2, "Gadget",    1500m, "Electronics",  50),
    new(3, "おにぎり",  150m,  "Food",         200),
    new(4, "ペットボトル", 120m, "Food",        150),
    new(5, "Cable",     400m,  "Electronics",  75),
};
```

---

### 16-2. フィルタ・変換・ソート

```csharp
// ── Where: フィルタ ──
var expensive = products.Where(p => p.Price >= 500).ToList();
// [Widget, Gadget, Cable]

// ── Select: 変換・射影 ──
var names = products.Select(p => p.Name).ToList();
// ["Widget", "Gadget", "おにぎり", ...]

// 匿名型に射影（名前と価格だけ取り出す）
var summaries = products
    .Select(p => new { p.Name, p.Price })
    .ToList();

// ── OrderBy / OrderByDescending: ソート ──
var byPrice    = products.OrderBy(p => p.Price).ToList();       // 安い順
var byPriceDesc = products.OrderByDescending(p => p.Price).ToList(); // 高い順

// 複数条件でのソート
var sorted = products
    .OrderBy(p => p.Category)
    .ThenByDescending(p => p.Price)
    .ToList();

// ── Skip / Take: ページング ──
int page = 2, pageSize = 2;
var paged = products
    .OrderBy(p => p.Id)
    .Skip((page - 1) * pageSize) // (2-1)*2 = 2件スキップ
    .Take(pageSize)               // 2件取得
    .ToList();
// [おにぎり, ペットボトル]
```

---

### 16-3. 集計・検索・グループ化

```csharp
// ── 集計 ──
int    count   = products.Count();                           // 5
int    countEx = products.Count(p => p.Price > 500);         // 2
decimal sum    = products.Sum(p => p.Price);                 // 3150
decimal avg    = products.Average(p => p.Price);             // 630
decimal max    = products.Max(p => p.Price);                 // 1500
decimal min    = products.Min(p => p.Price);                 // 120

// ── 存在確認 ──
bool anyExp  = products.Any(p => p.Price > 1000);   // True
bool allPositive = products.All(p => p.Price > 0);  // True

// ── 1件取得 ──
var first  = products.First();                              // 先頭（なければ例外）
var firstF = products.FirstOrDefault(p => p.Category == "Food"); // 最初の食品（なければ null）
var single = products.SingleOrDefault(p => p.Id == 3);      // 1件だけある場合（2件以上で例外）

// ── GroupBy: グループ化 ──
var byCategory = products
    .GroupBy(p => p.Category)
    .Select(g => new
    {
        Category   = g.Key,
        Count      = g.Count(),
        TotalPrice = g.Sum(p => p.Price),
        AvgPrice   = g.Average(p => p.Price)
    });

foreach (var g in byCategory)
    Console.WriteLine($"{g.Category}: {g.Count}件, 合計 {g.TotalPrice:C}, 平均 {g.AvgPrice:C}");
// Electronics: 3件, 合計 ¥2,880, 平均 ¥960
// Food: 2件, 合計 ¥270, 平均 ¥135
```

---

### 16-4. チェーンと遅延評価

```csharp
// 複数の操作をチェーンして1つのクエリにする
var result = products
    .Where(p => p.Stock > 60)              // 在庫60以上
    .OrderByDescending(p => p.Price)       // 価格の高い順
    .Take(3)                               // 上位3件
    .Select(p => $"{p.Name}: {p.Price:C}") // 文字列に変換
    .ToList();                             // ← ここで初めて実行される

foreach (var r in result)
    Console.WriteLine(r);

// 遅延評価の注意点: ToList() の前後で元データを変えると結果が変わる
var query = products.Where(p => p.Category == "Food");
products.Add(new(6, "パン", 200m, "Food", 80)); // クエリ実行前に追加
var list = query.ToList(); // パンも含まれる（遅延評価のため）
Console.WriteLine(list.Count); // 3（後から追加したパンも入る）

// 即時評価が必要な場面では先に ToList() してから操作する
var snapshot = products.Where(p => p.Category == "Food").ToList(); // 即時評価
products.Add(new(7, "おかし", 100m, "Food", 50));
Console.WriteLine(snapshot.Count); // 3（スナップショットなので変化しない）
```

---

### 16-5. Distinct, Join, Zip

```csharp
// ── Distinct: 重複除去 ──
var categories = products.Select(p => p.Category).Distinct().ToList();
// ["Electronics", "Food"]

// ── Zip: 2つのシーケンスを合わせる ──
var keys   = new[] { "A", "B", "C" };
var values = new[] { 1, 2, 3 };
var dict = keys.Zip(values, (k, v) => new { Key = k, Value = v });
foreach (var pair in dict)
    Console.WriteLine($"{pair.Key} = {pair.Value}");

// ── SelectMany: ネストしたコレクションを平坦化 ──
var sentences = new[] { "Hello World", "Foo Bar Baz" };
var words = sentences.SelectMany(s => s.Split(' ')).ToList();
// ["Hello", "World", "Foo", "Bar", "Baz"]
Console.WriteLine(string.Join(", ", words));
```

✅ **確認**：`GroupBy` の結果を変えて Electronics と Food の統計が変わることを確認する。
遅延評価のコードで `ToList()` の前後に要素を追加して結果の違いを確認する。

---

## 17. async / await

### 17-1. 非同期処理が必要な理由

```
同期（スレッドブロック）
Thread 1: [受付]──[DB待ち 100ms ブロック]──[返却]
Thread 2: [受付]──[DB待ち 100ms ブロック]──[返却]
Thread 3: [受付]──[DB待ち 100ms ブロック]──[返却]
↑ 100リクエストあったら100スレッドが必要。メモリを大量消費

非同期（スレッド解放）
Thread 1: [受付]─await─                    ─[返却]
              ↑ 待ち時間中にスレッドを解放
Thread 1:          [別のリクエスト受付]─await─
↑ 少ないスレッドで大量のリクエストをさばける
```

---

### 17-2. Task と async/await の書き方

```csharp
using System.Diagnostics;

// Task<T>: 非同期で T を返す
async Task<string> FetchFromDbAsync(int id)
{
    await Task.Delay(100); // DB クエリのシミュレーション
    return $"Product #{id}";
    // return は Task<string> ではなく string を書く（async が自動でラップ）
}

// Task: 非同期で何も返さない（void の代わり）
async Task SaveToDbAsync(string data)
{
    await Task.Delay(50);
    Console.WriteLine($"保存: {data}");
}

// ValueTask<T>: 軽量版 Task（頻繁に呼ばれる高パフォーマンス用途）
async ValueTask<int> GetCountAsync()
{
    await Task.Delay(10);
    return 42;
}

// 呼び出し
var sw = Stopwatch.StartNew();

var result = await FetchFromDbAsync(1);
Console.WriteLine(result); // Product #1

await SaveToDbAsync("Hello");

sw.Stop();
Console.WriteLine($"経過: {sw.ElapsedMilliseconds}ms"); // ~150ms
```

---

### 17-3. 並行実行

```csharp
// 順番に待つ（合計 300ms）
var sw = Stopwatch.StartNew();
var r1 = await FetchFromDbAsync(1);  // 100ms 待つ
var r2 = await FetchFromDbAsync(2);  // また100ms 待つ
var r3 = await FetchFromDbAsync(3);  // また100ms 待つ
sw.Stop();
Console.WriteLine($"順番: {sw.ElapsedMilliseconds}ms"); // ~300ms

// Task.WhenAll で並行実行（合計 ~100ms）
sw.Restart();
var t1 = FetchFromDbAsync(1);  // 開始だけして待たない
var t2 = FetchFromDbAsync(2);  // 開始だけして待たない
var t3 = FetchFromDbAsync(3);  // 開始だけして待たない
var results = await Task.WhenAll(t1, t2, t3); // 全部が終わるまで待つ
sw.Stop();
Console.WriteLine($"並行: {sw.ElapsedMilliseconds}ms"); // ~100ms

foreach (var r in results)
    Console.WriteLine(r);

// Task.WhenAny: どれか1つが終わったら先に進む
var first = await Task.WhenAny(t1, t2, t3);
Console.WriteLine(await first);
```

---

### 17-4. 絶対に守るルール

```csharp
// ────────── NG パターン ──────────

// NG1: .Result / .Wait() でブロック → デッドロックの危険
var data = FetchFromDbAsync(1).Result;   // 絶対に書かない
FetchFromDbAsync(1).Wait();              // 絶対に書かない

// NG2: async void → 例外がキャッチできない（アプリがクラッシュする）
async void BadHandler()
{
    await Task.Delay(1);
    throw new Exception("この例外はキャッチできない！");
}

// NG3: await せずに Task を捨てる → 例外が握りつぶされる
FetchFromDbAsync(1); // CS4014 警告が出る

// NG4: 不必要な async → オーバーヘッドが増える
async Task<string> JustReturn() => await Task.FromResult("hello"); // async 不要
string JustReturn() => "hello"; // これでいい

// ────────── OK パターン ──────────
var data = await FetchFromDbAsync(1);    // 常に await
_ = Task.Run(async () => { ... });      // fire-and-forget が必要な場合の書き方
```

---

### 17-5. CancellationToken（キャンセル対応）

```csharp
// 長時間処理をキャンセルできるようにする（API ではよく使う）
async Task LongProcessAsync(CancellationToken ct = default)
{
    for (int i = 0; i < 10; i++)
    {
        ct.ThrowIfCancellationRequested(); // キャンセルされていれば例外
        Console.WriteLine($"処理中... {i}");
        await Task.Delay(200, ct); // Delay もキャンセル対応
    }
}

// キャンセルトークンの作成
using var cts = new CancellationTokenSource();
cts.CancelAfter(500); // 500ms 後に自動キャンセル

try
{
    await LongProcessAsync(cts.Token);
}
catch (OperationCanceledException)
{
    Console.WriteLine("キャンセルされました");
}
```

✅ **確認**：`Task.WhenAll` が並行実行で `Task.Delay` を合計より短時間で完了することを `Stopwatch` で計測する。

---

## 18. record 型

### 18-1. record vs class

```csharp
// class: 参照比較・可変
class PersonClass
{
    public string Name { get; set; } = "";
    public int    Age  { get; set; }
}

var c1 = new PersonClass { Name = "Alice", Age = 30 };
var c2 = new PersonClass { Name = "Alice", Age = 30 };
Console.WriteLine(c1 == c2);   // False（別のオブジェクト）
Console.WriteLine(c1.Equals(c2)); // False
Console.WriteLine(c1);         // PersonClass（型名だけ）
c1.Name = "Bob";               // 変更可能

// record: 値比較・不変（デフォルト）
record PersonRecord(string Name, int Age);

var r1 = new PersonRecord("Alice", 30);
var r2 = new PersonRecord("Alice", 30);
Console.WriteLine(r1 == r2);   // True（中身で比較）
Console.WriteLine(r1.Equals(r2)); // True
Console.WriteLine(r1);         // PersonRecord { Name = Alice, Age = 30 }
// r1.Name = "Bob"; // コンパイルエラー（不変）
```

---

### 18-2. record の機能

```csharp
record Product(int Id, string Name, decimal Price, string Category);

var p1 = new Product(1, "Widget", 980m, "Electronics");

// with 式: 一部だけ変えたコピーを作る（元は変わらない）
var p2 = p1 with { Price = 1200m };
Console.WriteLine(p1.Price); // 980 ← 変わっていない
Console.WriteLine(p2.Price); // 1200

// 分解（Deconstruct）
var (id, name, price, category) = p1;
Console.WriteLine($"{id}: {name} - {price:C}"); // 1: Widget - ¥980

// パターンマッチングと組み合わせる
string Describe(Product p) => p switch
{
    { Price: > 1000, Category: "Electronics" } => "高価な電子機器",
    { Price: > 1000 }                          => "高価な商品",
    { Category: "Food" }                       => "食品",
    _                                          => "一般商品"
};
Console.WriteLine(Describe(p1)); // 一般商品
Console.WriteLine(Describe(p2)); // 高価な電子機器
```

---

### 18-3. record class / record struct

```csharp
// record class（デフォルト）: 参照型・不変
record class Point2D(double X, double Y);

// record struct: 値型・不変（スタックに積まれるので軽量）
record struct Size(double Width, double Height);

// mutable record struct: 値型・可変
record struct MutablePoint
{
    public double X { get; set; }
    public double Y { get; set; }
}

var s = new Size(100, 200);
Console.WriteLine(s); // Size { Width = 100, Height = 200 }
```

**record の使い所**：

| ケース | 推奨する型 |
|--------|----------|
| API レスポンス DTO | `record` |
| DB エンティティ | `class`（EF Core が変更追跡するため） |
| リクエスト DTO | `class`（バリデーション属性をつけるため） |
| 小さな値オブジェクト（座標など） | `record struct` |

✅ **確認**：`with` 式で変えたコピーが元の record に影響しないことを確認する。

---

## 19. パターンマッチング

### 19-1. is パターン

```csharp
// 型チェック + 変数束縛を同時に行う
object value = "Hello, World!";

// 旧スタイル
if (value is string)
{
    string s = (string)value;
    Console.WriteLine(s.ToUpper());
}

// パターンマッチング（型チェックと変数束縛を1行で）
if (value is string s2)
    Console.WriteLine(s2.ToUpper()); // HELLO, WORLD!

// null チェック
object? maybeNull = null;
if (maybeNull is null)       Console.WriteLine("null");
if (maybeNull is not null)   Console.WriteLine("not null"); // 出力されない

// 型チェック + 条件
if (value is string { Length: > 5 } longStr)
    Console.WriteLine($"長い文字列: {longStr}");
```

---

### 19-2. switch 式（パターンマッチングと組み合わせる）

```csharp
record Shape(string Type);
record Circle(double Radius)        : Shape("Circle");
record Rectangle(double W, double H): Shape("Rectangle");
record Triangle(double Base, double H): Shape("Triangle");

// 型パターン + プロパティパターン
double CalculateArea(Shape shape) => shape switch
{
    Circle { Radius: var r }           => Math.PI * r * r,
    Rectangle { W: var w, H: var h }   => w * h,
    Triangle  { Base: var b, H: var h } => 0.5 * b * h,
    _                                  => throw new ArgumentException($"不明な形状: {shape}")
};

Console.WriteLine(CalculateArea(new Circle(5)));           // 78.54
Console.WriteLine(CalculateArea(new Rectangle(4, 6)));     // 24
Console.WriteLine(CalculateArea(new Triangle(3, 4)));      // 6

// タプルパターン
string Combine(bool a, bool b) => (a, b) switch
{
    (true,  true)  => "両方 true",
    (true,  false) => "a だけ true",
    (false, true)  => "b だけ true",
    (false, false) => "両方 false",
};
Console.WriteLine(Combine(true, false)); // a だけ true
```

---

### 19-3. 実務パターン

```csharp
// グローバル例外ハンドラーで使う形（API チュートリアルで登場）
Exception ex = new NotFoundException("Product", 42);

var (statusCode, message) = ex switch
{
    NotFoundException   e => (404, e.Message),
    ValidationException e => (400, e.Message),
    UnauthorizedException _ => (401, "認証が必要です"),
    _                    => (500, "内部エラーが発生しました")
};

Console.WriteLine($"Status: {statusCode}, Message: {message}");
// Status: 404, Message: Product（ID: 42）は見つかりません

// リストパターン（C# 11）
int[] numbers = { 1, 2, 3 };
string desc = numbers switch
{
    []            => "空",
    [var single]  => $"要素1つ: {single}",
    [var first, var second] => $"要素2つ: {first}, {second}",
    [var first, .., var last] => $"先頭: {first}, 末尾: {last}, 計{numbers.Length}個"
};
Console.WriteLine(desc); // 先頭: 1, 末尾: 3, 計3個
```

✅ **確認**：`CalculateArea` に新しい形状クラスを追加して `_` が例外を投げることを確認する。

---

## 20. enum

### 20-1. 基本

```csharp
// enum: 限られた選択肢を型安全に表現する
enum OrderStatus
{
    Pending   = 0, // 明示的に数値を指定（省略すると 0 から自動採番）
    Active    = 1,
    Completed = 2,
    Cancelled = 3
}

// 使い方
var status = OrderStatus.Pending;
Console.WriteLine(status);         // Pending（名前で表示される）
Console.WriteLine((int)status);    // 0（数値にキャスト）

// 比較
if (status == OrderStatus.Pending)
    Console.WriteLine("まだ処理されていません");

// switch 式との組み合わせ
string label = status switch
{
    OrderStatus.Pending   => "処理待ち",
    OrderStatus.Active    => "処理中",
    OrderStatus.Completed => "完了",
    OrderStatus.Cancelled => "キャンセル済",
    _                     => throw new ArgumentOutOfRangeException()
};
Console.WriteLine(label); // 処理待ち

// 数値 → enum
var fromInt = (OrderStatus)2;
Console.WriteLine(fromInt); // Completed

// 文字列 → enum（クエリパラメータのパースで使う）
if (Enum.TryParse<OrderStatus>("Active", ignoreCase: true, out var parsed))
    Console.WriteLine(parsed); // Active

// 全値の列挙
foreach (var v in Enum.GetValues<OrderStatus>())
    Console.WriteLine($"{(int)v}: {v}");
```

---

### 20-2. [Flags] 属性

```csharp
// 複数の値を組み合わせてフラグとして扱う
[Flags]
enum Permission
{
    None   = 0,
    Read   = 1 << 0, // 1  = 0001
    Write  = 1 << 1, // 2  = 0010
    Delete = 1 << 2, // 4  = 0100
    Admin  = 1 << 3, // 8  = 1000
    // 組み合わせ定義も可能
    ReadWrite = Read | Write,
    All       = Read | Write | Delete | Admin
}

var userPerm = Permission.Read | Permission.Write; // 0011 = 3
Console.WriteLine(userPerm);                       // Read, Write

// HasFlag で特定の権限があるか確認
Console.WriteLine(userPerm.HasFlag(Permission.Read));   // True
Console.WriteLine(userPerm.HasFlag(Permission.Delete)); // False

// 権限を追加・削除
userPerm |= Permission.Delete;  // Delete を追加 → 0111
userPerm &= ~Permission.Write;  // Write を削除 → 0101
Console.WriteLine(userPerm);    // Read, Delete
```

✅ **確認**：`Enum.GetValues<OrderStatus>()` で全ての値が列挙されることを確認する。
`[Flags]` で権限を組み合わせて `HasFlag` が正しく動くことを確認する。

---

## 21. 拡張メソッド

### 21-1. 拡張メソッドの仕組み

```csharp
// 既存のクラスを変更せずにメソッドを追加できる
// ルール: static クラス + static メソッド + 第1引数に「this 対象の型」

public static class StringExtensions
{
    // string に IsNullOrEmpty の逆バージョンを追加
    public static bool HasValue(this string? value)
        => !string.IsNullOrWhiteSpace(value);

    // string を指定文字数で切り捨て
    public static string Truncate(this string value, int maxLength)
    {
        if (value.Length <= maxLength) return value;
        return value[..maxLength] + "...";
    }

    // メールアドレスを正規化（トリム + 小文字化）
    public static string NormalizeEmail(this string email)
        => email.Trim().ToLower();
}

// 使い方: 拡張メソッドは元のクラスのメソッドのように見える
string? name = "  Alice  ";
Console.WriteLine(name.HasValue());          // True
Console.WriteLine("alice@EXAMPLE.COM".NormalizeEmail()); // alice@example.com
Console.WriteLine("これは長いテキストです".Truncate(5));  // これは長い...
Console.WriteLine("".HasValue());            // False
Console.WriteLine(((string?)null).HasValue()); // False
```

---

### 21-2. コレクションへの拡張

```csharp
public static class EnumerableExtensions
{
    // null 要素を除いた IEnumerable を返す
    public static IEnumerable<T> WhereNotNull<T>(
        this IEnumerable<T?> source) where T : class
        => source.Where(item => item is not null)!;

    // コレクションが null または空でないか確認
    public static bool HasItems<T>(this IEnumerable<T>? source)
        => source?.Any() == true;

    // バッチ処理（大量データを n 件ずつ分割）
    public static IEnumerable<IEnumerable<T>> Chunk<T>(
        this IEnumerable<T> source, int size)
    {
        var list = source.ToList();
        for (int i = 0; i < list.Count; i += size)
            yield return list.Skip(i).Take(size);
    }
}

var items = new List<string?> { "Alice", null, "Bob", null, "Charlie" };
var nonNull = items.WhereNotNull().ToList();
Console.WriteLine(string.Join(", ", nonNull)); // Alice, Bob, Charlie

var numbers = Enumerable.Range(1, 10); // 1〜10
foreach (var batch in numbers.Chunk(3))
    Console.WriteLine(string.Join(", ", batch));
// 1, 2, 3
// 4, 5, 6
// 7, 8, 9
// 10
```

---

### 21-3. IServiceCollection への拡張（API 開発の要）

```csharp
// ASP.NET Core の Program.cs をスッキリさせるパターン
// builder.Services.AddJwtAuthentication() などの正体

public static class ServiceCollectionExtensions
{
    // IServiceCollection に独自のサービスをまとめて登録するメソッドを追加
    public static IServiceCollection AddRepositories(
        this IServiceCollection services)
    {
        services.AddScoped(typeof(IRepository<>), typeof(Repository<>));
        services.AddScoped<IProductRepository, ProductRepository>();
        return services; // ← メソッドチェーンできるよう自分を返す
    }

    public static IServiceCollection AddAppServices(
        this IServiceCollection services)
    {
        services.AddScoped<IProductService, ProductService>();
        return services;
    }
}

// Program.cs での使い方（拡張メソッドでスッキリ書ける）
// builder.Services
//     .AddRepositories()
//     .AddAppServices()     // メソッドチェーンできる
//     .AddJwtAuthentication(config);
```

✅ **確認**：`Truncate` で日本語文字列（1文字が複数バイト）を切り捨てても文字数でカウントされることを確認する。
`return services` を `void` に変えたとき、メソッドチェーンができなくなることを確認する。

---

## 22. よく使う省略記法まとめ

### 22-1. 式形式メンバー（Expression-bodied member）

```csharp
class Calculator
{
    private double _value;
    public Calculator(double value) { _value = value; }

    // メソッドを1行で書く
    public double Double()     => _value * 2;
    public double Half()       => _value / 2;
    public string AsString()   => _value.ToString("F2");

    // プロパティも1行で
    public bool IsPositive     => _value > 0;
    public bool IsZero         => _value == 0;
    public double Absolute     => Math.Abs(_value);

    // コンストラクタも1行で（C# 7+）
    // public Calculator(double value) => _value = value;

    public override string ToString() => $"Calculator({_value:F2})";
}
```

---

### 22-2. タプル

```csharp
// メソッドから複数の値を返す
(string Name, int Age) GetPerson() => ("Alice", 30);

var (name, age) = GetPerson(); // 分解代入
Console.WriteLine($"{name}: {age}歳"); // Alice: 30歳

// 名前付きタプル
var stats = (Min: 10, Max: 100, Average: 55.5);
Console.WriteLine(stats.Min);     // 10
Console.WriteLine(stats.Average); // 55.5

// リポジトリのページングで使う（API チュートリアル）
(IEnumerable<string> Items, int Total) GetPaged(int page, int size)
{
    var all = Enumerable.Range(1, 20).Select(i => $"Item{i}");
    var items = all.Skip((page - 1) * size).Take(size);
    return (items, 20);
}

var (items, total) = GetPaged(2, 5);
Console.WriteLine($"取得: {items.Count()}件 / 全{total}件");
```

---

### 22-3. その他の便利な記法

```csharp
// ── コレクション式（C# 12）──
int[] arr  = [1, 2, 3, 4, 5];
List<int> list = [1, 2, 3];
// 2つのコレクションを結合（スプレッド演算子）
int[] combined = [..arr, 6, 7, 8];

// ── nameof 演算子 ──
// リファクタリング時に文字列の変更漏れを防ぐ
string propName = nameof(DateTime.Now); // "Now"（リテラルではなくシンボルを参照）
throw new ArgumentNullException(nameof(list)); // "list" という文字列になる

// ── throw 式 ──
string? input = null;
string value = input ?? throw new ArgumentNullException(nameof(input));
// サービス層でよく使う
// var product = await repo.GetByIdAsync(id) ?? throw new NotFoundException("Product", id);

// ── using 宣言 ──
using var stream = File.OpenRead("data.txt");
// メソッドを抜けると自動 Dispose

// ── インデックスと範囲 ──
var nums = new[] { 0, 1, 2, 3, 4, 5, 6, 7, 8, 9 };
Console.WriteLine(nums[^1]);     // 9（末尾から1番目）
Console.WriteLine(nums[^2]);     // 8（末尾から2番目）
var slice1 = nums[2..5];         // [2, 3, 4]（インデックス2〜4）
var slice2 = nums[..3];          // [0, 1, 2]（先頭から3つ）
var slice3 = nums[7..];          // [7, 8, 9]（7番目から末尾まで）
var last3  = nums[^3..];         // [7, 8, 9]（末尾3つ）

// ── パターンマッチングによる null チェック ──
string? s = GetSomething();
if (s is { Length: > 0 } nonEmpty)
    Console.WriteLine(nonEmpty.ToUpper());

// ── required プロパティ（C# 11）──
class Config
{
    // required: new するときに必ず設定しなければコンパイルエラー
    public required string ApiKey  { get; init; }
    public required string BaseUrl { get; init; }
    public int TimeoutSeconds { get; init; } = 30; // 省略可能
}

var config = new Config
{
    ApiKey  = "abc123",
    BaseUrl = "https://api.example.com"
    // TimeoutSeconds は省略可能
};
// var bad = new Config(); // コンパイルエラー（required が設定されていない）
```

---

### 22-4. 早見表

| 書き方 | 意味 | 例 |
|--------|------|----|
| `var` | 型推論 | `var x = 42;` |
| `x ??= y` | null のときだけ代入 | `name ??= "default";` |
| `x?.y` | null なら null | `user?.Name` |
| `x ?? y` | null なら y | `name ?? "unknown"` |
| `x!` | null 免除（自己責任） | `name!` |
| `x is T v` | 型チェック + 束縛 | `if (obj is string s)` |
| `x is not null` | null でないチェック | `if (name is not null)` |
| `x[^1]` | 末尾から1番目 | `arr[^1]` |
| `x[2..5]` | スライス | `arr[2..5]` |
| `x => expr` | ラムダ式 | `n => n * 2` |
| `nameof(x)` | 変数名を文字列で | `nameof(userId)` |
| `throw new ...()` | 例外を投げる | `throw new NotFoundException(...)` |

✅ **確認**：`nums[^3..]` と `nums[7..]` が同じ結果になることを確認する。
`required` プロパティを省略した `new Config()` がコンパイルエラーになることを確認する。

---

## おわりに

このファイルで学んだことが API チュートリアルのどこで使われるかの対応表：

| 概念 | API チュートリアルでの使われ方 |
|------|-------------------------------|
| クラス・プロパティ | エンティティ（`Product`, `Category`） |
| 継承・抽象クラス | `Repository<T>` ← `ProductRepository` |
| インターフェース | `IRepository<T>`, `IProductService` |
| ジェネリクス | `Repository<T>`, `PagedResult<T>` |
| null 安全 | `T?` 戻り値、クエリパラメータ |
| 例外処理 | カスタム例外 + グローバルハンドラー |
| async/await | 全 DB・HTTP 操作 |
| LINQ | EF Core クエリ（`Where`, `Select`, `ToListAsync`） |
| record | レスポンス DTO（`ProductResponse`, `PagedResult`） |
| パターンマッチング | グローバルエラーハンドラーの分岐 |
| enum | ステータス管理 |
| 拡張メソッド | `ServiceExtensions`（DI 登録のまとめ） |
| タプル | リポジトリのページング戻り値 |

---

*対象バージョン: .NET 8 / C# 12*
