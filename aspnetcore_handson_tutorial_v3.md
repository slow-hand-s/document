---
created: 2026-05-24T21:04
updated: 2026-05-24T21:30
---
# ASP.NET Core コントローラーベース API ハンズオンチュートリアル

> 対象：他言語経験者 / .NET 8 / C# 12 / リポジトリパターン  
> このチュートリアルを最後まで進めると「商品管理 API」が完成する。  
> 各ステップに「✅ 確認」を設けているので、詰まったときの切り分けに使うこと。

---

## 目次

1. [C# 基礎ハンズオン](#1-c-基礎ハンズオン)
2. [プロジェクト作成](#2-プロジェクト作成)
3. [プロジェクトファイル解説](#3-プロジェクトファイル解説)
4. [アーキテクチャ設計](#4-アーキテクチャ設計)
5. [エンティティと DbContext](#5-エンティティと-dbcontext)
6. [リポジトリ層](#6-リポジトリ層)
7. [サービス層](#7-サービス層)
8. [モデル（DTO）とバリデーション](#8-モデルdtoとバリデーション)
9. [コントローラー](#9-コントローラー)
10. [エラーハンドリング](#10-エラーハンドリング)
11. [認証・認可（JWT）](#11-認証認可jwt)
12. [Swagger](#12-swagger)
13. [API 設計原則](#13-api-設計原則)
14. [IP 制御・レート制限](#14-ip-制御レート制限)

---

## 1. C# 基礎ハンズオン

### このステップのゴール
C# 特有の書き方を「書いて動かして」体に馴染ませる。  
.NET が入っていれば `dotnet-script` または https://dotnetfiddle.net で今すぐ試せる。

---

### 1-1. null 安全

**他言語との違い**：C# 8 以降、null を入れられる型と入れられない型をコンパイラが区別する。
`?` がなければ「絶対に null にならない」という契約になる。

```csharp
// --- 試してみよう ---
string name = "Alice";
// name = null;  // ← コメントを外すと CS8600 警告

string? maybeName = null;   // ? で null 許容を宣言
Console.WriteLine(maybeName?.Length ?? 0);  // null なら 0

// null 合体代入（null のときだけ代入）
maybeName ??= "default";
Console.WriteLine(maybeName); // default
```

**API 開発での使い方**：
```csharp
// サービス層でリソースが見つからない場合は T? を返す
// → 呼び出し側が「null かもしれない」と意識せざるを得ない
public async Task<Product?> GetByIdAsync(int id) { ... }

var product = await GetByIdAsync(99);
// product を使う前に必ず null チェックが必要（コンパイラが警告）
Console.WriteLine(product?.Name ?? "not found");
```

✅ **確認**：`string name = null;` のコメントを外してコンパイルエラーが出ることを確認する。

---

### 1-2. record 型

**なぜ使うか**：API のレスポンス DTO に使うと「外から値を書き換えられない」ことをコンパイラが保証する。
通常の class との違いは等値比較と不変性。

```csharp
// --- 試してみよう ---

// class: 参照比較（同じオブジェクトかどうか）
class ProductClass { public int Id; public string Name = ""; }
var c1 = new ProductClass { Id = 1, Name = "Widget" };
var c2 = new ProductClass { Id = 1, Name = "Widget" };
Console.WriteLine(c1 == c2); // False（別オブジェクト）

// record: 値比較（中身が同じかどうか）
record ProductRecord(int Id, string Name);
var r1 = new ProductRecord(1, "Widget");
var r2 = new ProductRecord(1, "Widget");
Console.WriteLine(r1 == r2); // True（中身が同じ）

// toString も自動で充実
Console.WriteLine(r1); // ProductRecord { Id = 1, Name = Widget }

// with 式：一部だけ変えたコピーを作る（元は変更されない）
var r3 = r1 with { Name = "Gadget" };
Console.WriteLine(r1); // ProductRecord { Id = 1, Name = Widget } ← 変わっていない
Console.WriteLine(r3); // ProductRecord { Id = 1, Name = Gadget }
```

✅ **確認**：`r1 with { Name = "Gadget" }` のあとに `r1.Name` を出力して元が変わっていないことを確認する。

---

### 1-3. async / await

**なぜ必要か**：Web API は同時に複数リクエストを処理する。
DB待ちの間にスレッドをブロックすると、サーバーのスループットが落ちる。
`await` はスレッドをブロックせず、DB が応答するまでの間に別のリクエストを処理できる。

```
同期（悪い）                    非同期（良い）
Thread A: [処理]─[DB待ちブロック]─[返却]   Thread A: [処理]─await─      ─[返却]
Thread B: 待機中...                        Thread A:         ↑ 別のリクエストを処理
Thread B: 待機中...                        Thread A:                   ↑ DB応答が来たら再開
```

```csharp
// --- 書き方パターンを覚える ---

// Task<T>: 非同期で T を返す（最もよく使う）
public async Task<string> FetchDataAsync()
{
    await Task.Delay(100); // DB/HTTP 待ちのシミュレーション
    return "data";
}

// Task: 非同期で何も返さない（void の代わり）
public async Task SaveAsync(string data)
{
    await Task.Delay(50);
    Console.WriteLine($"saved: {data}");
}

// 呼び出し側も必ず await する
var result = await FetchDataAsync();
Console.WriteLine(result);
```

**実務で絶対に守るルール**：
```csharp
// NG: .Result でブロック（デッドロックの危険）
var data = FetchDataAsync().Result;

// NG: async void（例外をキャッチできない）
async void BadMethod() { await Task.Delay(1); }

// OK: 常に await で待つ
var data = await FetchDataAsync();
```

✅ **確認**：`FetchDataAsync().Result` に変えて実行し、応答が返ってくることを確認する（デッドロックはコンソールアプリでは発生しにくいが本番コードでは危険）。

---

### 1-4. LINQ

**使い所**：DB クエリ（EF Core）・コレクション変換・フィルタ全般。
SQL を C# で書けるイメージ。

```csharp
// --- 試してみよう ---
var products = new List<(int Id, string Name, decimal Price, int CategoryId)>
{
    (1, "Widget", 980m,  1),
    (2, "Gadget", 1500m, 2),
    (3, "Donut",  200m,  1),
    (4, "Cable",  400m,  2),
};

// Where → Select → OrderBy のチェーン（最もよく使う）
var result = products
    .Where(p => p.CategoryId == 1)           // カテゴリ1のみ
    .OrderByDescending(p => p.Price)          // 価格の高い順
    .Select(p => new { p.Id, p.Name });       // Id と Name だけ取り出す

foreach (var p in result)
    Console.WriteLine($"{p.Id}: {p.Name}");
// 1: Widget
// 3: Donut

// FirstOrDefault: 1件取得（見つからなければ null）
var found = products.FirstOrDefault(p => p.Id == 99);
Console.WriteLine(found); // (0, null, 0, 0) のデフォルト値

// GroupBy: 集計
var summary = products
    .GroupBy(p => p.CategoryId)
    .Select(g => new {
        Category = g.Key,
        Count    = g.Count(),
        Avg      = g.Average(p => p.Price)
    });

foreach (var s in summary)
    Console.WriteLine($"Category {s.Category}: {s.Count}件, 平均 {s.Avg:F0}円");
```

**重要**：LINQ は遅延評価。`ToList()` または `await ToListAsync()` を呼ぶまで実際の処理は走らない。

```csharp
var query = products.Where(p => p.Price > 500); // まだ実行されていない
var list  = query.ToList();                      // ここで初めて評価される
```

✅ **確認**：`Where` の条件を変えて出力が変わることを確認する。

---

### 1-5. インターフェースとジェネリクス（リポジトリパターンの核心）

**なぜインターフェースを使うか**：
実装の詳細（EF Core か Dapper か、SQL Server か PostgreSQL か）を隠蔽し、
テスト時にモックに差し替えられるようにするため。

```csharp
// --- 試してみよう ---

// インターフェース: 「何ができるか」の定義
interface IStorage
{
    void Save(string data);
    string? Load(string key);
}

// 実装A: インメモリ（テスト用）
class InMemoryStorage : IStorage
{
    private readonly Dictionary<string, string> _store = new();
    public void Save(string data)       => _store["key"] = data;
    public string? Load(string key)     => _store.TryGetValue(key, out var v) ? v : null;
}

// 実装B: ファイル（本番用）
class FileStorage : IStorage
{
    public void Save(string data)       => File.WriteAllText("data.txt", data);
    public string? Load(string key)     => File.Exists("data.txt") ? File.ReadAllText("data.txt") : null;
}

// 使う側はインターフェース型で受け取る
void Process(IStorage storage)
{
    storage.Save("hello");
    Console.WriteLine(storage.Load("key")); // hello
}

Process(new InMemoryStorage()); // テスト時
// Process(new FileStorage()); // 本番時（差し替えるだけ）
```

**ジェネリクス**：型を引数として受け取る仕組み。汎用リポジトリを作るときに必須。

```csharp
// <T> が型パラメータ
class Box<T>
{
    private T _value;
    public Box(T value) => _value = value;
    public T Get() => _value;
}

var intBox    = new Box<int>(42);
var stringBox = new Box<string>("hello");
Console.WriteLine(intBox.Get());    // 42
Console.WriteLine(stringBox.Get()); // hello
```

✅ **確認**：`Process` に渡す実装を `InMemoryStorage` から別のクラスに変えても動くことを確認する。

---

### 1-6. パターンマッチング

サービス層の戻り値分岐やエラー処理を簡潔に書ける。

```csharp
// --- 試してみよう ---

// switch 式（値を返す）
string Classify(int n) => n switch
{
    < 0         => "負",
    0           => "ゼロ",
    < 100       => "小さい正",
    _           => "大きい正"   // _ はデフォルト
};

Console.WriteLine(Classify(-5));  // 負
Console.WriteLine(Classify(0));   // ゼロ
Console.WriteLine(Classify(50));  // 小さい正
Console.WriteLine(Classify(200)); // 大きい正

// is パターン（型チェックと変数束縛を同時に）
object? value = "hello";
if (value is string s && s.Length > 3)
    Console.WriteLine(s.ToUpper()); // HELLO

// API のエラー分岐でよく使う形
record Result(bool Success, string? Error);
var result = new Result(false, "Not found");

var statusCode = result switch
{
    { Success: true }              => 200,
    { Error: "Not found" }        => 404,
    { Error: "Validation error" } => 400,
    _                              => 500
};
Console.WriteLine(statusCode); // 404
```

✅ **確認**：`Classify` の引数を変えて全パターンを試す。

---

## 2. プロジェクト作成

### このステップのゴール
`dotnet run` で API が起動し、ブラウザで Swagger UI が開ける状態にする。

---

### Step 1: プロジェクト生成

```bash
dotnet new webapi -n MyApi --framework net8.0 --use-controllers
cd MyApi
```

生成直後のフォルダ構成を確認する：

```bash
ls
# Controllers/  Program.cs  Properties/  appsettings.json  appsettings.Development.json  MyApi.csproj
```

`--use-controllers` をつけないとミニマル API 形式になるので注意。

---

### Step 2: 不要ファイルを削除する

生成されるサンプルコードは今後の邪魔になるので消す。

```bash
rm Controllers/WeatherForecastController.cs
rm WeatherForecast.cs
```

---

### Step 3: パッケージを追加する

```bash
# EF Core（DB 操作）
dotnet add package Microsoft.EntityFrameworkCore.SqlServer
dotnet add package Microsoft.EntityFrameworkCore.Tools

# JWT 認証
dotnet add package Microsoft.AspNetCore.Authentication.JwtBearer

# API バージョニング
dotnet add package Asp.Versioning.Mvc

# レート制限は .NET 8 に組み込み済みなので追加不要
```

✅ **確認**：`dotnet build` がエラーなく通ること。

---

### Step 4: フォルダ構成を作る

```bash
mkdir -p Controllers \
         Services/Interfaces \
         Repositories/Interfaces \
         Data/Migrations \
         Models/Entities \
         Models/Requests \
         Models/Responses \
         Middleware \
         Extensions \
         Helpers
```

完成後のフォルダ構成（これが今チュートリアルのゴール）：

```
MyApi/
├── Controllers/
│   ├── AuthController.cs
│   └── ProductsController.cs
├── Data/
│   ├── AppDbContext.cs
│   └── Migrations/
├── Extensions/
│   └── ServiceExtensions.cs
├── Helpers/
│   └── IpHelper.cs
├── Middleware/
│   ├── GlobalExceptionMiddleware.cs
│   ├── IpWhitelistMiddleware.cs
│   └── IpBlocklistMiddleware.cs
├── Models/
│   ├── Entities/
│   │   ├── Category.cs
│   │   └── Product.cs
│   ├── Requests/
│   │   ├── LoginRequest.cs
│   │   ├── CreateProductRequest.cs
│   │   └── UpdateProductRequest.cs
│   └── Responses/
│       ├── ProductResponse.cs
│       └── PagedResult.cs
├── Repositories/
│   ├── Interfaces/
│   │   ├── IRepository.cs
│   │   └── IProductRepository.cs
│   ├── Repository.cs
│   └── ProductRepository.cs
├── Services/
│   ├── Interfaces/
│   │   └── IProductService.cs
│   ├── IpBlocklistService.cs
│   ├── LoginAttemptService.cs
│   └── ProductService.cs
├── appsettings.json
├── appsettings.Development.json
└── Program.cs
```

---

### Step 5: appsettings.json を書く

`appsettings.json` を以下に置き換える：

```json
{
  "ConnectionStrings": {
    "Default": "Server=localhost;Database=MyApiDb;Trusted_Connection=True;TrustServerCertificate=True"
  },
  "Jwt": {
    "Key": "CHANGE_THIS_IN_PRODUCTION_USE_USER_SECRETS",
    "Issuer": "MyApi",
    "Audience": "MyApiUsers",
    "ExpiryMinutes": 60
  },
  "IpWhitelist": {
    "AllowedIps": [ "127.0.0.1", "::1" ],
    "ProtectedPaths": [ "/api/admin" ]
  },
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning",
      "Microsoft.EntityFrameworkCore.Database.Command": "Information"
    }
  },
  "AllowedHosts": "*"
}
```

開発用シークレット（JWT キーを appsettings に書かないための設定）：

```bash
dotnet user-secrets init
dotnet user-secrets set "Jwt:Key" "dev-secret-key-must-be-32chars-or-more!!"
```

---

### Step 6: Program.cs のひな形を用意する

この段階では最小限の記述にする。章が進むたびに追記していく。

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

var app = builder.Build();

app.UseSwagger();
app.UseSwaggerUI();
app.UseHttpsRedirection();
app.UseAuthorization();
app.MapControllers();

app.Run();
```

---

### Step 7: 起動確認

```bash
dotnet run
```

ブラウザで `https://localhost:{ポート番号}/swagger` を開く。  
ポート番号は起動ログの `Now listening on: https://localhost:XXXX` で確認する。

✅ **確認**：Swagger UI が表示される。この時点ではエンドポイントは 0 件でよい。

---

## 3. プロジェクトファイル解説

### このステップのゴール
`dotnet new` で生成されたファイルが何をしているのかを理解する。
「なぜこのファイルが必要か」がわかると、後の章でファイルを編集するときに迷わなくなる。

---

### 3-1. 生成直後のファイル一覧と役割

```
MyApi/
├── Controllers/                  ← HTTP リクエストの入り口（自分で作るクラス置き場）
│   └── WeatherForecastController.cs   ← サンプル（削除済み）
├── Properties/
│   └── launchSettings.json       ← 開発時の起動設定（ポート番号など）
├── appsettings.json              ← アプリ全体の設定ファイル（本番・共通）
├── appsettings.Development.json  ← 開発環境だけに適用される設定（本番設定を上書き）
├── MyApi.csproj                  ← プロジェクト定義ファイル（パッケージ・.NET バージョン）
├── Program.cs                    ← アプリのエントリーポイント（起動設定を書く）
└── WeatherForecast.cs            ← サンプル（削除済み）
```

---

### 3-2. MyApi.csproj — プロジェクト定義ファイル

他言語での対応：`package.json`（Node.js）/ `pom.xml`（Java）/ `requirements.txt`（Python）に近い。
**.NET のバージョン**・**追加パッケージ（NuGet）**・**ビルド設定**を一元管理する XML ファイル。

```xml
<!-- MyApi.csproj -->
<Project Sdk="Microsoft.NET.Sdk.Web">

  <PropertyGroup>
    <!-- 使用する .NET バージョン -->
    <TargetFramework>net8.0</TargetFramework>

    <!-- null 安全機能を有効にする（C# 8+）-->
    <!-- string と string? の区別をコンパイラが強制する -->
    <Nullable>enable</Nullable>

    <!-- 暗黙的な using ディレクティブを有効にする -->
    <!-- System, System.Linq など頻出の using を自動追加してくれる -->
    <ImplicitUsings>enable</ImplicitUsings>
  </PropertyGroup>

</Project>
```

**`dotnet add package` をするとここに追記される：**

```xml
<!-- dotnet add package Microsoft.EntityFrameworkCore.SqlServer を実行すると自動で追加 -->
<ItemGroup>
  <PackageReference Include="Microsoft.EntityFrameworkCore.SqlServer" Version="8.x.x" />
  <PackageReference Include="Microsoft.AspNetCore.Authentication.JwtBearer" Version="8.x.x" />
</ItemGroup>
```

> **実務Tips**：バージョンを固定することで「チームメンバー全員が同じバージョンのパッケージを使う」ことが保証される。
> `dotnet restore` を実行すれば csproj に書かれたパッケージを一括ダウンロードできる。

---

### 3-3. Program.cs — アプリのエントリーポイント

**C# プロジェクトで最初に実行されるファイル**。他言語での `main()` 関数に相当する。
ASP.NET Core では「サービスの登録」と「ミドルウェアの設定」という 2 つの役割を持つ。

```csharp
// Program.cs の骨格
var builder = WebApplication.CreateBuilder(args);
//  ↑ WebApplicationBuilder を生成。
//    appsettings.json の読み込み・ログ設定・DI コンテナの準備などを行う

// ─── ① サービス登録（DIコンテナへの登録）───
// 「このインターフェースが要求されたら、このクラスを使って」という設定を書く
builder.Services.AddControllers();
builder.Services.AddDbContext<AppDbContext>(...);
builder.Services.AddScoped<IProductService, ProductService>();

var app = builder.Build();
//        ↑ Build() を呼ぶと登録した設定が確定し、WebApplication（アプリ本体）が生成される
//          Build() より後はサービスの追加登録はできない

// ─── ② ミドルウェアパイプライン設定 ───
// リクエストが「どの順番で」どのミドルウェアを通るかを定義する
// 上から順番に実行される（順序が重要！）
app.UseMiddleware<GlobalExceptionMiddleware>(); // 例外キャッチ
app.UseAuthentication();                        // 認証
app.UseAuthorization();                         // 認可
app.MapControllers();                           // コントローラーへのルーティング

app.Run(); // サーバー起動（ここで処理が止まり、リクエスト待ちになる）
```

**サービス登録とミドルウェアの違い**：

| | サービス登録（`builder.Services`） | ミドルウェア（`app.Use...`） |
|--|--|--|
| **タイミング** | 起動前（Build前） | 起動後（Build後） |
| **役割** | 「何を使えるか」を定義 | 「どう処理するか」を定義 |
| **例** | DBコンテキスト・サービスクラス | 認証・HTTPS・ルーティング |

---

### 3-4. ミドルウェアパイプラインとは

リクエストがコントローラーに届くまでの「処理の連鎖」。
**ロシアのマトリョーシカ人形**のように、外側から順番に処理が通過する。

```
クライアント
    │
    ▼  リクエスト（内側へ）
┌───────────────────────────────┐
│ GlobalExceptionMiddleware      │  ← 例外を全てキャッチする外側の壁
│  ┌─────────────────────────┐  │
│  │ IpBlocklistMiddleware    │  │  ← ブロックされたIPを弾く
│  │  ┌───────────────────┐  │  │
│  │  │ RateLimiter        │  │  │  ← 連打を制限する
│  │  │  ┌─────────────┐  │  │  │
│  │  │  │ Authentication │  │  │  │  ← JWTを検証する
│  │  │  │  ┌─────────┐ │  │  │  │
│  │  │  │  │Controller│ │  │  │  │  ← 実際の処理
│  │  │  │  └─────────┘ │  │  │  │
│  │  │  └─────────────┘  │  │  │
│  │  └───────────────────┘  │  │
│  └─────────────────────────┘  │
└───────────────────────────────┘
    │
    ▼  レスポンス（外側へ）
クライアント
```

各ミドルウェアは `await _next(ctx)` を呼ぶことで次の処理に進む。
呼ばなければそこでリクエストを止めてレスポンスを返せる（IP ブロックはこの仕組みを使う）。

---

### 3-5. appsettings.json — 設定ファイル

アプリの設定値を外部ファイルに書き出す仕組み。
コードの中にベタ書きしないことで、環境ごとに値を変えやすくなる。

```json
// appsettings.json（全環境共通の設定）
{
  "ConnectionStrings": {
    "Default": "Server=localhost;..."
  },
  "Jwt": {
    "Issuer": "MyApi",
    "ExpiryMinutes": 60
  },
  "Logging": {
    "LogLevel": {
      "Default": "Information"
    }
  }
}
```

```json
// appsettings.Development.json（開発環境だけ上書き）
// 本番の appsettings.json を汚さずに開発専用の設定を追加できる
{
  "Logging": {
    "LogLevel": {
      // 開発中だけ SQL クエリのログを出す
      "Microsoft.EntityFrameworkCore.Database.Command": "Information"
    }
  }
}
```

**設定の優先順位（高い順）**：
```
環境変数                         （本番サーバー・Kubernetes）
  > dotnet user-secrets          （開発マシンのシークレット）
    > appsettings.{環境名}.json  （appsettings.Development.json）
      > appsettings.json         （共通設定）
```

コードから読む方法：
```csharp
// 直接読む
var issuer = builder.Configuration["Jwt:Issuer"];

// セクションごとにまとめて読む
var jwtSection = builder.Configuration.GetSection("Jwt");
var expiry = jwtSection["ExpiryMinutes"];

// 接続文字列専用メソッド
var connStr = builder.Configuration.GetConnectionString("Default");
```

---

### 3-6. Properties/launchSettings.json — 起動設定

`dotnet run` や Visual Studio でデバッグ起動するときの設定。
**本番環境には影響しない**（開発専用）。

```json
{
  "profiles": {
    "http": {
      "commandName": "Project",
      "applicationUrl": "http://localhost:5000",
      "environmentVariables": {
        // ASPNETCORE_ENVIRONMENT の値に応じて appsettings.{値}.json が読まれる
        "ASPNETCORE_ENVIRONMENT": "Development"
      }
    },
    "https": {
      "commandName": "Project",
      "applicationUrl": "https://localhost:7001;http://localhost:5001",
      "environmentVariables": {
        "ASPNETCORE_ENVIRONMENT": "Development"
      }
    }
  }
}
```

ポート番号を変えたい場合はここを編集する。

---

### 3-7. namespace と using

C# のコードを書くときに必ず出てくる 2 つのキーワード。

```csharp
// namespace: このファイルが「どのグループに属するか」を宣言する
// フォルダ構成に合わせて命名するのが慣習
namespace MyApi.Services;  // → MyApi/Services/ フォルダのファイル

// using: 他の namespace のクラスを「名前だけ」で使えるようにする
// using なしだと完全修飾名（MyApi.Models.Entities.Product）が必要
using MyApi.Models.Entities;  // これがあると Product と書くだけで済む
using Microsoft.EntityFrameworkCore;
```

**.csproj に `<ImplicitUsings>enable</ImplicitUsings>` があると**、
`System`・`System.Linq`・`System.Threading.Tasks` などの頻出 using は自動でインポートされる。
明示的に書く必要があるのは `Microsoft.*` や自分のプロジェクトの namespace だけ。

✅ **確認**：`MyApi.csproj` を開いて `<TargetFramework>` と `<Nullable>` の記述があることを確認する。
`appsettings.json` と `appsettings.Development.json` の違いを確認する。

---

## 4. アーキテクチャ設計

### このステップのゴール
コードを書き始める前に、「どのファイルが何をする責任を持つか」を理解する。
設計の理由がわかると、コードをどこに書くべきか迷わなくなる。

---

### 4-1. このプロジェクトで採用するアーキテクチャ

**レイヤードアーキテクチャ（Layered Architecture）** +
**リポジトリパターン（Repository Pattern）** の組み合わせ。

```
┌──────────────────────────────────────────────────────────┐
│                    HTTP リクエスト                         │
└──────────────────────────┬───────────────────────────────┘
                           │
┌──────────────────────────▼───────────────────────────────┐
│                   プレゼンテーション層                      │
│                  Controllers/                             │
│   役割：HTTP の受け口。リクエスト→C#オブジェクト変換のみ    │
│         ビジネスロジックは一切書かない                      │
└──────────────────────────┬───────────────────────────────┘
                           │ インターフェース経由で呼ぶ
┌──────────────────────────▼───────────────────────────────┐
│                   ビジネスロジック層                        │
│                    Services/                              │
│   役割：「在庫があるか」「名前が重複しないか」などの判断    │
│         DBの知識は持たない（Repositoryに委ねる）            │
└──────────────────────────┬───────────────────────────────┘
                           │ インターフェース経由で呼ぶ
┌──────────────────────────▼───────────────────────────────┐
│                   データアクセス層                         │
│                  Repositories/                            │
│   役割：DBへの読み書きだけ。SQLの知識をここに閉じ込める     │
│         ビジネスルールは知らない                           │
└──────────────────────────┬───────────────────────────────┘
                           │
┌──────────────────────────▼───────────────────────────────┐
│                    データベース                            │
│              Data/AppDbContext.cs                         │
│   役割：EF Core が C# ↔ DB を仲介する                      │
└──────────────────────────────────────────────────────────┘
```

---

### 4-2. 各層の責任を具体例で理解する

「商品を作成する」という操作を例に、各層が何をするかを見る。

```
① クライアントが送るもの：
  POST /api/products
  { "name": "Widget", "price": 980, "stock": 100, "categoryId": 1 }

② Controller（ProductsController）がすること：
  - JSON を CreateProductRequest オブジェクトに変換する
  - バリデーション属性のチェック（[Required][Range] など）を通過させる
  - IProductService.CreateAsync(request) を呼ぶ
  - 戻り値を 201 レスポンスに変換して返す
  ここでは「名前が重複しているか」などは判断しない

③ Service（ProductService）がすること：
  - CategoryId=1 のカテゴリが DB に存在するか確認する
  - "Widget" という名前の商品が既に存在しないか確認する
  - 問題なければ Product エンティティを作り、Repository に保存を依頼する
  ここでは「どうやって DB に保存するか」は知らない

④ Repository（ProductRepository）がすること：
  - _db.Products.AddAsync(product) を呼んで EF Core に渡す
  - _db.SaveChangesAsync() で DB に確定する
  ここでは「名前が重複してはいけない」などのルールは知らない

⑤ クライアントが受け取るもの：
  HTTP 201 Created
  { "id": 4, "name": "Widget", "price": 980, ... }
```

**各層の「知ってよいこと」と「知ってはいけないこと」**：

| 層 | 知ってよいこと | 知ってはいけないこと |
|----|--------------|-------------------|
| Controller | HTTP, DTO, Service のインターフェース | ビジネスルール, DB の構造 |
| Service | ビジネスルール, Repository のインターフェース | HTTP, EF Core の書き方 |
| Repository | EF Core, SQL, DB の構造 | ビジネスルール, HTTP |

---

### 4-3. インターフェースと依存性注入（DI）

**なぜインターフェースを経由させるのか**

Controller が Service の「具体的なクラス」に直接依存すると：
```csharp
// NG: 具体クラスに直接依存
public class ProductsController
{
    private readonly ProductService _service; // ← 具体クラス

    // テストのとき「本物の DB を使うサービス」しか使えない
    // ProductService を差し替えることができない
}
```

インターフェースを経由すると：
```csharp
// OK: インターフェースに依存
public class ProductsController
{
    private readonly IProductService _service; // ← インターフェース

    // テストのとき MockProductService を差し込める
    // 本番と開発で実装を切り替えられる
}
```

**DI（依存性注入）の仕組み**：

```csharp
// Program.cs: 「このインターフェースが来たらこのクラスを使え」と登録
builder.Services.AddScoped<IProductService, ProductService>();

// ProductsController のコンストラクタ
public ProductsController(IProductService service)
//                         ↑ IProductService を要求するだけ
//    DI コンテナが自動で ProductService を生成して渡してくれる
```

```
DI コンテナ（Program.cs で管理）
┌─────────────────────────────────────┐
│ IProductService → ProductService    │
│ IProductRepository → ProductRepo    │
│ IRepository<T> → Repository<T>      │
│ AppDbContext → AppDbContext          │
└─────────────────────────────────────┘
        ↑ 登録                ↓ 自動解決
  Program.cs           コンストラクタ引数
```

**DI のライフタイム（重要）**：

```csharp
// Scoped: HTTPリクエスト1回につき1インスタンス
// → DB コンテキストはこれ（1リクエスト中は同じトランザクションを共有）
builder.Services.AddScoped<IProductService, ProductService>();
builder.Services.AddDbContext<AppDbContext>(...); // DbContext は自動で Scoped

// Transient: 注入されるたびに新しいインスタンス
// → 軽量でステートレスなサービスに使う
builder.Services.AddTransient<IEmailSender, SmtpEmailSender>();

// Singleton: アプリ起動から終了まで1インスタンス
// → キャッシュ・設定管理など。Scoped/Transient なものを注入してはいけない
builder.Services.AddSingleton<IIpBlocklistService, IpBlocklistService>();
```

---

### 4-4. データの流れとオブジェクトの変換

API では「DB のデータ形式」と「クライアントに返す形式」を意図的に分離する。

```
クライアント送信
  { "name": "Widget", "price": 980, "categoryId": 1 }
       ↓ JSON デシリアライズ
  CreateProductRequest  ← リクエスト DTO（バリデーション属性つき）
       ↓ Service が受け取る
  Product エンティティ  ← DB のテーブル構造に対応（直接クライアントに返さない）
       ↓ DB に保存・取得
  Product エンティティ  ← DB から取得（Category も JOIN して取得）
       ↓ Service が変換
  ProductResponse       ← レスポンス DTO（record で不変）
       ↓ JSON シリアライズ
クライアント受信
  { "id": 4, "name": "Widget", "price": 980, "categoryName": "電子機器" }
```

**エンティティをそのまま返してはいけない理由**：
- DB のカラム名・リレーションがクライアントに漏れる
- パスワードハッシュなど見せてはいけないフィールドが混入する恐れがある
- DB スキーマ変更のたびに API の仕様が壊れる

---

### 4-5. ファイルと層の対応表

チュートリアルで作る全ファイルがどの層に属するかを先に把握しておく。

| ファイル | 層 | 役割 |
|---------|-----|------|
| `Controllers/ProductsController.cs` | プレゼンテーション | HTTPの受け口 |
| `Controllers/AuthController.cs` | プレゼンテーション | 認証 |
| `Services/ProductService.cs` | ビジネスロジック | 商品操作のルール |
| `Services/IpBlocklistService.cs` | インフラ | IPブロック管理 |
| `Services/LoginAttemptService.cs` | インフラ | ログイン失敗管理 |
| `Repositories/ProductRepository.cs` | データアクセス | 商品のDB操作 |
| `Repositories/Repository.cs` | データアクセス | 汎用DB操作 |
| `Data/AppDbContext.cs` | データアクセス | EF Core設定 |
| `Models/Entities/Product.cs` | データ | DBテーブル対応クラス |
| `Models/Requests/CreateProductRequest.cs` | データ | API入力DTO |
| `Models/Responses/ProductResponse.cs` | データ | API出力DTO |
| `Middleware/GlobalExceptionMiddleware.cs` | インフラ | 例外処理 |
| `Middleware/IpWhitelistMiddleware.cs` | インフラ | IPホワイトリスト |
| `Extensions/ServiceExtensions.cs` | 設定 | DI登録の整理 |
| `Helpers/IpHelper.cs` | ユーティリティ | IP取得ロジック |

✅ **確認**：フォルダ構成（2章 Step4 で作成）を見ながら、各フォルダがどの層に対応するか答えられること。

---

## 5. エンティティと DbContext

### このステップのゴール
DB テーブルに対応するクラスを定義し、マイグレーションでテーブルを作る。

---

### Step 1: エンティティを作る

エンティティ = DB のテーブル 1 行に対応する C# クラス。

```csharp
// Models/Entities/Category.cs
namespace MyApi.Models.Entities;

public class Category
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;

    // ナビゲーションプロパティ: EF Core がリレーションを管理する
    // Category を取得するとき Include() すれば Products も一緒に取れる
    public ICollection<Product> Products { get; set; } = new List<Product>();
}
```

```csharp
// Models/Entities/Product.cs
namespace MyApi.Models.Entities;

public class Product
{
    public int     Id         { get; set; }
    public string  Name       { get; set; } = string.Empty;
    public decimal Price      { get; set; }
    public int     Stock      { get; set; }

    // 外部キー
    public int      CategoryId { get; set; }
    // null! = 「EF Core が必ず設定するので自分では null を入れない」という宣言
    public Category Category   { get; set; } = null!;

    public DateTime  CreatedAt { get; set; }
    public DateTime? UpdatedAt { get; set; }
}
```

---

### Step 2: DbContext を作る

DbContext = EF Core が DB と C# の間を仲介するクラス。

```csharp
// Data/AppDbContext.cs
using Microsoft.EntityFrameworkCore;
using MyApi.Models.Entities;

namespace MyApi.Data;

public class AppDbContext : DbContext
{
    public AppDbContext(DbContextOptions<AppDbContext> options) : base(options) { }

    // DbSet = テーブルを表すプロパティ
    // _db.Products と書くと Products テーブルを操作できる
    public DbSet<Product>  Products   => Set<Product>();
    public DbSet<Category> Categories => Set<Category>();

    protected override void OnModelCreating(ModelBuilder mb)
    {
        mb.Entity<Category>(e =>
        {
            e.ToTable("Categories");
            e.Property(c => c.Name).HasMaxLength(50).IsRequired();
        });

        mb.Entity<Product>(e =>
        {
            e.ToTable("Products");
            e.Property(p => p.Name).HasMaxLength(100).IsRequired();
            // decimal は精度を明示しないと EF が警告を出す
            e.Property(p => p.Price).HasPrecision(18, 2);
            // CategoryId に INDEX を張る（JOIN クエリの高速化）
            e.HasIndex(p => p.CategoryId);
            // リレーション設定
            e.HasOne(p => p.Category)
             .WithMany(c => c.Products)
             .HasForeignKey(p => p.CategoryId)
             // CASCADE 削除を禁止: カテゴリを消しても商品は消えない
             .OnDelete(DeleteBehavior.Restrict);
        });
    }

    // SaveChanges をオーバーライドして CreatedAt / UpdatedAt を自動セット
    // → コントローラーやサービスで日時をセットしなくていい
    public override Task<int> SaveChangesAsync(CancellationToken ct = default)
    {
        var now = DateTime.UtcNow;
        foreach (var entry in ChangeTracker.Entries<Product>())
        {
            if (entry.State == EntityState.Added)
                entry.Entity.CreatedAt = now;
            if (entry.State == EntityState.Modified)
                entry.Entity.UpdatedAt = now;
        }
        return base.SaveChangesAsync(ct);
    }
}
```

---

### Step 3: Program.cs に DB 接続を追加する

```csharp
// Program.cs に追記（AddControllers() の下）
using Microsoft.EntityFrameworkCore;
using MyApi.Data;

// DB 接続の登録
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("Default")));
```

---

### Step 4: マイグレーションを実行する

```bash
# EF Core ツールをグローバルインストール（初回のみ）
dotnet tool install --global dotnet-ef

# マイグレーションファイルを生成（DB はまだ変更しない）
dotnet ef migrations add InitialCreate --output-dir Data/Migrations

# DB にテーブルを作る
dotnet ef database update
```

✅ **確認**：`Data/Migrations/` フォルダにファイルが生成されていること。  
SQL Server Management Studio（または Azure Data Studio）でテーブルが作られていることを確認する。

---

### Step 5: シードデータを投入する（動作確認用）

```csharp
// Data/AppDbContext.cs の OnModelCreating に追加
mb.Entity<Category>().HasData(
    new Category { Id = 1, Name = "電子機器" },
    new Category { Id = 2, Name = "食品" }
);

mb.Entity<Product>().HasData(
    new Product { Id = 1, Name = "Widget",   Price = 980m,  Stock = 100, CategoryId = 1, CreatedAt = DateTime.UtcNow },
    new Product { Id = 2, Name = "Gadget",   Price = 1500m, Stock = 50,  CategoryId = 1, CreatedAt = DateTime.UtcNow },
    new Product { Id = 3, Name = "おにぎり", Price = 150m,  Stock = 200, CategoryId = 2, CreatedAt = DateTime.UtcNow }
);
```

```bash
dotnet ef migrations add SeedData --output-dir Data/Migrations
dotnet ef database update
```

✅ **確認**：DB の Products テーブルに 3 件のデータが入っていること。

---

## 6. リポジトリ層

### このステップのゴール
DB アクセスのコードを Repository クラスに閉じ込める。
Service 層は Repository インターフェースしか知らない状態にする。

---

### なぜリポジトリパターンか

```
分けない場合（Service に直接 EF Core）
  ProductService の中に _db.Products.Where(...)... が散在
  → DB を MySQL に変えたらサービス全体を修正
  → テストで本物の DB が必要

リポジトリパターン
  ProductService は IProductRepository にしか依存しない
  → テスト時は MockRepository に差し替えるだけ
  → DB の詳細はリポジトリの中だけに閉じる
```

---

### Step 1: 汎用リポジトリ インターフェースを作る

```csharp
// Repositories/Interfaces/IRepository.cs
namespace MyApi.Repositories.Interfaces;

// where T : class = 値型（int など）は不可。クラスだけ受け付ける
public interface IRepository<T> where T : class
{
    Task<T?>              GetByIdAsync(int id);
    Task<IEnumerable<T>> GetAllAsync();
    Task                  AddAsync(T entity);
    void                  Update(T entity);   // EF の Update は同期で OK
    void                  Delete(T entity);
    Task<int>             SaveChangesAsync();
}
```

---

### Step 2: 汎用リポジトリの実装を作る

```csharp
// Repositories/Repository.cs
using Microsoft.EntityFrameworkCore;
using MyApi.Data;
using MyApi.Repositories.Interfaces;

namespace MyApi.Repositories;

public class Repository<T> : IRepository<T> where T : class
{
    protected readonly AppDbContext _db;
    protected readonly DbSet<T>     _dbSet;

    public Repository(AppDbContext db)
    {
        _db    = db;
        _dbSet = db.Set<T>(); // 型 T に対応する DbSet を取得
    }

    public async Task<T?> GetByIdAsync(int id)
        => await _dbSet.FindAsync(id);

    public async Task<IEnumerable<T>> GetAllAsync()
        // AsNoTracking: 読み取り専用クエリで変更追跡を外す
        //               → EF の内部処理が減り、パフォーマンス向上
        => await _dbSet.AsNoTracking().ToListAsync();

    public async Task AddAsync(T entity)
        => await _dbSet.AddAsync(entity);

    public void Update(T entity)
        => _dbSet.Update(entity);

    public void Delete(T entity)
        => _dbSet.Remove(entity);

    public async Task<int> SaveChangesAsync()
        => await _db.SaveChangesAsync();
}
```

---

### Step 3: Product 専用インターフェースを作る

汎用リポジトリでカバーできない「結合・ページング・重複チェック」を追加する。

```csharp
// Repositories/Interfaces/IProductRepository.cs
using MyApi.Models.Entities;

namespace MyApi.Repositories.Interfaces;

public interface IProductRepository : IRepository<Product>
{
    // Category を JOIN して取得（汎用の GetById は JOIN しない）
    Task<Product?> GetByIdWithCategoryAsync(int id);

    // ページング + フィルタ
    // タプルで「アイテムリスト」と「総件数」を返す
    Task<(IEnumerable<Product> Items, int Total)> GetPagedAsync(
        int page, int pageSize, int? categoryId, string? searchTerm);

    // 重複チェック（excludeId = 更新時に自分自身を除く）
    Task<bool> ExistsByNameAsync(string name, int? excludeId = null);
}
```

---

### Step 4: Product 専用リポジトリの実装を作る

```csharp
// Repositories/ProductRepository.cs
using Microsoft.EntityFrameworkCore;
using MyApi.Data;
using MyApi.Models.Entities;
using MyApi.Repositories.Interfaces;

namespace MyApi.Repositories;

public class ProductRepository : Repository<Product>, IProductRepository
{
    public ProductRepository(AppDbContext db) : base(db) { }

    public async Task<Product?> GetByIdWithCategoryAsync(int id)
        => await _dbSet
            .Include(p => p.Category) // Category テーブルを JOIN
            .AsNoTracking()
            .FirstOrDefaultAsync(p => p.Id == id);

    public async Task<(IEnumerable<Product> Items, int Total)> GetPagedAsync(
        int page, int pageSize, int? categoryId, string? searchTerm)
    {
        // IQueryable = まだ実行されていないクエリ
        // Where を重ねても SQL は 1 本になる（N+1 にならない）
        var query = _dbSet
            .Include(p => p.Category)
            .AsNoTracking()
            .AsQueryable();

        // 動的フィルタ: パラメータがあるときだけ WHERE 句を追加
        if (categoryId.HasValue)
            query = query.Where(p => p.CategoryId == categoryId.Value);

        if (!string.IsNullOrWhiteSpace(searchTerm))
            query = query.Where(p => p.Name.Contains(searchTerm));

        // Count は Skip/Take の前に取る
        var total = await query.CountAsync();

        var items = await query
            .OrderByDescending(p => p.CreatedAt)
            .Skip((page - 1) * pageSize) // OFFSET
            .Take(pageSize)               // LIMIT
            .ToListAsync();               // ここで SQL が発行される

        return (items, total);
    }

    public async Task<bool> ExistsByNameAsync(string name, int? excludeId = null)
        => await _dbSet.AnyAsync(p =>
            p.Name == name && (excludeId == null || p.Id != excludeId));
}
```

---

### Step 5: DI に登録する

```csharp
// Program.cs に追記
using MyApi.Repositories;
using MyApi.Repositories.Interfaces;

// 汎用リポジトリ: typeof() でオープンジェネリクスを登録
// IRepository<Product> → Repository<Product>
// IRepository<Category> → Repository<Category>
// を自動解決してくれる
builder.Services.AddScoped(typeof(IRepository<>), typeof(Repository<>));
builder.Services.AddScoped<IProductRepository, ProductRepository>();
```

✅ **確認**：`dotnet build` がエラーなく通ること。  
この段階ではまだ API エンドポイントは増えないが、ビルドが通れば OK。

---

## 7. サービス層

### このステップのゴール
ビジネスルールを Service に集約する。
「カテゴリが存在するか」「同名商品が既にあるか」などの判断はここで行う。

---

### Step 1: サービス インターフェースを作る

```csharp
// Services/Interfaces/IProductService.cs
using MyApi.Models.Requests;
using MyApi.Models.Responses;

namespace MyApi.Services.Interfaces;

public interface IProductService
{
    Task<PagedResult<ProductResponse>> GetPagedAsync(
        int page, int pageSize, int? categoryId, string? searchTerm);

    // 見つからなければ NotFoundException を throw する（null を返さない）
    Task<ProductResponse> GetByIdAsync(int id);
    Task<ProductResponse> CreateAsync(CreateProductRequest request);
    Task<ProductResponse> UpdateAsync(int id, UpdateProductRequest request);
    Task                  DeleteAsync(int id);
}
```

---

### Step 2: カスタム例外を先に定義する

サービスから例外を throw することでコントローラーの if 文を減らす。
（グローバルハンドラーが後で 404/400 に変換する）

```csharp
// Middleware/AppExceptions.cs
namespace MyApi.Middleware;

// 404: リソースが見つからない
public class NotFoundException : Exception
{
    public NotFoundException(string entityName, object key)
        : base($"{entityName}（ID: {key}）は見つかりません") { }
}

// 400: ビジネスバリデーション違反
public class ValidationException : Exception
{
    public ValidationException(string message) : base(message) { }
}

// 403: 権限不足
public class ForbiddenException : Exception
{
    public ForbiddenException(string message = "この操作は許可されていません")
        : base(message) { }
}

// 409: リソースの競合（重複登録など）
public class ConflictException : Exception
{
    public ConflictException(string message) : base(message) { }
}
```

---

### Step 3: サービスの実装を作る

```csharp
// Services/ProductService.cs
using MyApi.Middleware;
using MyApi.Models.Entities;
using MyApi.Models.Requests;
using MyApi.Models.Responses;
using MyApi.Repositories.Interfaces;
using MyApi.Services.Interfaces;

namespace MyApi.Services;

public class ProductService : IProductService
{
    private readonly IProductRepository  _productRepo;
    private readonly IRepository<Category> _categoryRepo;

    public ProductService(
        IProductRepository productRepo,
        IRepository<Category> categoryRepo)
    {
        _productRepo  = productRepo;
        _categoryRepo = categoryRepo;
    }

    public async Task<PagedResult<ProductResponse>> GetPagedAsync(
        int page, int pageSize, int? categoryId, string? searchTerm)
    {
        var (items, total) = await _productRepo.GetPagedAsync(
            page, pageSize, categoryId, searchTerm);

        var responses = items.Select(ToResponse);
        return new PagedResult<ProductResponse>(responses, total, page, pageSize);
    }

    public async Task<ProductResponse> GetByIdAsync(int id)
    {
        var product = await _productRepo.GetByIdWithCategoryAsync(id)
            ?? throw new NotFoundException("Product", id);

        return ToResponse(product);
    }

    public async Task<ProductResponse> CreateAsync(CreateProductRequest req)
    {
        // ── ビジネスバリデーション ──
        // 1. カテゴリが存在するか
        var category = await _categoryRepo.GetByIdAsync(req.CategoryId)
            ?? throw new ValidationException($"CategoryId {req.CategoryId} は存在しません");

        // 2. 同名商品が既にないか
        if (await _productRepo.ExistsByNameAsync(req.Name))
            throw new ConflictException($"商品名 '{req.Name}' は既に登録されています");

        // ── エンティティ生成 ──
        var product = new Product
        {
            Name       = req.Name,
            Price      = req.Price,
            Stock      = req.Stock,
            CategoryId = req.CategoryId,
            // CreatedAt は DbContext.SaveChangesAsync で自動セットされる
        };

        await _productRepo.AddAsync(product);
        await _productRepo.SaveChangesAsync();

        // 保存後に Category を含めて再取得して返す
        return await GetByIdAsync(product.Id);
    }

    public async Task<ProductResponse> UpdateAsync(int id, UpdateProductRequest req)
    {
        var product = await _productRepo.GetByIdWithCategoryAsync(id)
            ?? throw new NotFoundException("Product", id);

        if (req.Name is not null)
        {
            // 他の商品と名前が被っていないか（自分自身は除外）
            if (await _productRepo.ExistsByNameAsync(req.Name, excludeId: id))
                throw new ConflictException($"商品名 '{req.Name}' は既に使われています");

            product.Name = req.Name;
        }

        if (req.Price is not null) product.Price = req.Price.Value;
        if (req.Stock is not null) product.Stock = req.Stock.Value;

        _productRepo.Update(product);
        await _productRepo.SaveChangesAsync();

        return await GetByIdAsync(id);
    }

    public async Task DeleteAsync(int id)
    {
        var product = await _productRepo.GetByIdAsync(id)
            ?? throw new NotFoundException("Product", id);

        _productRepo.Delete(product);
        await _productRepo.SaveChangesAsync();
    }

    // エンティティ → レスポンス DTO への変換を一箇所にまとめる
    private static ProductResponse ToResponse(Product p) =>
        new(p.Id, p.Name, p.Price, p.Stock, p.Category.Name, p.CreatedAt);
}
```

---

### Step 4: DI に登録する

```csharp
// Program.cs に追記
using MyApi.Services;
using MyApi.Services.Interfaces;

builder.Services.AddScoped<IProductService, ProductService>();
```

✅ **確認**：`dotnet build` がエラーなく通ること。  
`ProductResponse` と `PagedResult` がまだないのでエラーが出る場合は次章に進む。

---

## 8. モデル（DTO）とバリデーション

### このステップのゴール
API の入出力専用クラスを定義する。
エンティティをそのまま返さず DTO に詰め替える理由：
- DB の内部構造（カラム名・リレーション）を外部に漏らさない
- レスポンスに含める情報を明示的に制御できる
- エンティティに変更があっても API の互換性を維持しやすい

---

### Step 1: レスポンス DTO を作る

```csharp
// Models/Responses/ProductResponse.cs
namespace MyApi.Models.Responses;

// record: 不変・値比較・ToString が自動で充実
// コンストラクタの引数がそのままプロパティになる
public record ProductResponse(
    int      Id,
    string   Name,
    decimal  Price,
    int      Stock,
    string   CategoryName,
    DateTime CreatedAt
);
```

```csharp
// Models/Responses/PagedResult.cs
namespace MyApi.Models.Responses;

public record PagedResult<T>(
    IEnumerable<T> Items,
    int            TotalCount,
    int            Page,
    int            PageSize
)
{
    // 計算プロパティ（コンストラクタ後に定義）
    public int  TotalPages => (int)Math.Ceiling((double)TotalCount / PageSize);
    public bool HasNext    => Page < TotalPages;
    public bool HasPrev    => Page > 1;
}
```

---

### Step 2: リクエスト DTO を作る

```csharp
// Models/Requests/CreateProductRequest.cs
using System.ComponentModel.DataAnnotations;

namespace MyApi.Models.Requests;

public class CreateProductRequest
{
    // [Required]: null・空文字を拒否。[ApiController] が自動で 400 を返す
    [Required(ErrorMessage = "商品名は必須です")]
    [MaxLength(100, ErrorMessage = "商品名は100文字以内で入力してください")]
    public string Name { get; set; } = string.Empty;

    [Range(0, 9_999_999, ErrorMessage = "価格は 0〜9,999,999 の範囲で入力してください")]
    public decimal Price { get; set; }

    [Range(0, int.MaxValue, ErrorMessage = "在庫数は0以上で入力してください")]
    public int Stock { get; set; }

    [Required]
    [Range(1, int.MaxValue, ErrorMessage = "カテゴリIDは1以上を指定してください")]
    public int CategoryId { get; set; }
}
```

```csharp
// Models/Requests/UpdateProductRequest.cs
using System.ComponentModel.DataAnnotations;

namespace MyApi.Models.Requests;

// 更新は全フィールドを任意にする（null = 変更しない）
public class UpdateProductRequest
{
    [MaxLength(100)]
    public string? Name { get; set; }

    [Range(0, 9_999_999)]
    public decimal? Price { get; set; }

    [Range(0, int.MaxValue)]
    public int? Stock { get; set; }
}
```

```csharp
// Models/Requests/LoginRequest.cs
using System.ComponentModel.DataAnnotations;

namespace MyApi.Models.Requests;

public class LoginRequest
{
    [Required]
    [EmailAddress]
    public string Email { get; set; } = string.Empty;

    [Required]
    [MinLength(8, ErrorMessage = "パスワードは8文字以上で入力してください")]
    public string Password { get; set; } = string.Empty;
}
```

---

### Step 3: カスタムバリデーション属性（任意・応用）

標準の属性では表現できないルールを属性として定義する。

```csharp
// Models/Requests/Attributes/FutureDateAttribute.cs
using System.ComponentModel.DataAnnotations;

namespace MyApi.Models.Requests.Attributes;

public class FutureDateAttribute : ValidationAttribute
{
    protected override ValidationResult? IsValid(object? value, ValidationContext ctx)
    {
        if (value is DateTime dt && dt <= DateTime.UtcNow)
            return new ValidationResult("未来の日付を指定してください");

        return ValidationResult.Success;
    }
}

// 使い方（キャンペーン作成などで）
// [FutureDate]
// public DateTime StartDate { get; set; }
```

✅ **確認**：`dotnet build` がエラーなく通ること。

---

## 9. コントローラー

### このステップのゴール
HTTP リクエストを受け取り、サービスを呼んで結果を返すコントローラーを実装する。
コントローラーの責任は「HTTP の変換のみ」。ビジネスロジックは書かない。

---

### Step 1: ProductsController を作る

```csharp
// Controllers/ProductsController.cs
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc;
using MyApi.Models.Requests;
using MyApi.Services.Interfaces;

namespace MyApi.Controllers;

[ApiController]
// [controller] = クラス名から "Controller" を除いた部分（Products）
[Route("api/[controller]")]
[Produces("application/json")]
public class ProductsController : ControllerBase
{
    private readonly IProductService _service;
    private readonly ILogger<ProductsController> _logger;

    // コンストラクタ注入: DI が IProductService の実装を自動で渡す
    public ProductsController(IProductService service, ILogger<ProductsController> logger)
    {
        _service = service;
        _logger  = logger;
    }

    /// <summary>商品一覧をページングで取得</summary>
    [HttpGet]
    [ProducesResponseType(StatusCodes.Status200OK)]
    public async Task<IActionResult> GetAll(
        [FromQuery] int    page       = 1,
        [FromQuery] int    pageSize   = 20,
        [FromQuery] int?   categoryId = null,
        [FromQuery] string? search    = null)
    {
        // ページサイズを 1〜100 の範囲に収める（大量取得の防止）
        var safePageSize = Math.Clamp(pageSize, 1, 100);
        var result = await _service.GetPagedAsync(page, safePageSize, categoryId, search);
        return Ok(result);
    }

    /// <summary>商品を ID で取得</summary>
    [HttpGet("{id:int}")]
    [ProducesResponseType(StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public async Task<IActionResult> GetById(int id)
    {
        // NotFoundException が throw されたらグローバルハンドラーが 404 に変換する
        // → コントローラーで null チェックや try-catch は不要
        var product = await _service.GetByIdAsync(id);
        return Ok(product);
    }

    /// <summary>商品を新規作成</summary>
    [HttpPost]
    [Authorize]
    [ProducesResponseType(StatusCodes.Status201Created)]
    [ProducesResponseType(StatusCodes.Status400BadRequest)]
    [ProducesResponseType(StatusCodes.Status401Unauthorized)]
    [ProducesResponseType(StatusCodes.Status409Conflict)]
    public async Task<IActionResult> Create([FromBody] CreateProductRequest request)
    {
        // [ApiController] が ModelState を自動チェックして
        // バリデーション違反なら 400 を返す（ここには届かない）
        var created = await _service.CreateAsync(request);

        // 201 Created + Location ヘッダーに /api/products/{id} を付ける
        return CreatedAtAction(nameof(GetById), new { id = created.Id }, created);
    }

    /// <summary>商品を更新</summary>
    [HttpPut("{id:int}")]
    [Authorize]
    [ProducesResponseType(StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status400BadRequest)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    [ProducesResponseType(StatusCodes.Status409Conflict)]
    public async Task<IActionResult> Update(int id, [FromBody] UpdateProductRequest request)
    {
        var updated = await _service.UpdateAsync(id, request);
        return Ok(updated);
    }

    /// <summary>商品を削除</summary>
    [HttpDelete("{id:int}")]
    [Authorize(Roles = "Admin")]
    [ProducesResponseType(StatusCodes.Status204NoContent)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public async Task<IActionResult> Delete(int id)
    {
        await _service.DeleteAsync(id);
        return NoContent(); // 204: 成功・返すボディなし
    }
}
```

---

### Step 2: 起動して動作確認する

```bash
dotnet run
```

Swagger UI（`/swagger`）を開いて `GET /api/products` を実行する。  
ただし現時点では認証ミドルウェアをまだ設定していないため、  
`[Authorize]` がついているエンドポイントは 500 になる可能性がある。  
次章（エラーハンドリング）と認証章を実装してから再確認する。

✅ **確認**：`GET /api/products` を叩いてシードデータの 3 件が返ること。

---

### Step 3: ルーティングのパターンを理解する

```csharp
// ルート制約（型が合わなければ 404）
[HttpGet("{id:int}")]         // /api/products/5   → OK
                              // /api/products/abc  → 404

// クエリパラメータをクラスで受け取る（パラメータが多いときに便利）
public class ProductQueryParams
{
    public int    Page       { get; set; } = 1;
    public int    PageSize   { get; set; } = 20;
    public int?   CategoryId { get; set; }
    public string? Search    { get; set; }
    public string  SortBy    { get; set; } = "createdAt";
    public string  SortOrder { get; set; } = "desc";
}

[HttpGet]
public async Task<IActionResult> GetAll([FromQuery] ProductQueryParams query) { ... }

// ネストされたルート（商品に紐づくレビューなど）
[HttpGet("{productId:int}/reviews")]
public async Task<IActionResult> GetReviews(int productId) { ... }
```

---

## 10. エラーハンドリング

### このステップのゴール
グローバル例外ミドルウェアを追加し、コントローラー内の try-catch を不要にする。
全エラーが統一フォーマットで返るようにする。

---

### Step 1: グローバル例外ミドルウェアを作る

```csharp
// Middleware/GlobalExceptionMiddleware.cs
using MyApi.Middleware;  // AppExceptions

namespace MyApi.Middleware;

public class GlobalExceptionMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<GlobalExceptionMiddleware> _logger;

    public GlobalExceptionMiddleware(RequestDelegate next,
        ILogger<GlobalExceptionMiddleware> logger)
    {
        _next   = next;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext ctx)
    {
        try
        {
            await _next(ctx); // 次のミドルウェア or コントローラーへ進む
        }
        catch (Exception ex)
        {
            await HandleAsync(ctx, ex);
        }
    }

    private async Task HandleAsync(HttpContext ctx, Exception ex)
    {
        // パターンマッチで例外の種類ごとにステータスコードを決定
        var (status, message) = ex switch
        {
            NotFoundException   e => (404, e.Message),
            ValidationException e => (400, e.Message),
            ConflictException   e => (409, e.Message),
            ForbiddenException  e => (403, e.Message),
            // 予期しない例外: 詳細はログに残し、ユーザーには汎用メッセージを返す
            _ => (500, "内部エラーが発生しました")
        };

        if (status >= 500)
            _logger.LogError(ex, "Unhandled exception");
        else
            _logger.LogWarning("Handled [{Status}]: {Message}", status, ex.Message);

        ctx.Response.StatusCode  = status;
        ctx.Response.ContentType = "application/json";

        // 統一エラーフォーマット
        await ctx.Response.WriteAsJsonAsync(new
        {
            status,
            message,
            // traceId をログと紐付けることで問い合わせ対応がしやすくなる
            traceId = ctx.TraceIdentifier
        });
    }
}
```

---

### Step 2: Program.cs に登録する（順序に注意）

```csharp
// Program.cs の app.Use... より前に追加
// ミドルウェアは登録順に実行される
// GlobalExceptionMiddleware は最初に登録して全ての例外を拾う

app.UseMiddleware<GlobalExceptionMiddleware>(); // ← 先頭

// 以降は既存のまま
app.UseSwagger();
// ...
```

---

### Step 3: 動作確認する

```bash
dotnet run
```

Swagger UI で以下を試す：

```
GET /api/products/9999
→ { "status": 404, "message": "Product（ID: 9999）は見つかりません", "traceId": "..." }

POST /api/products  ← Body を空にして送る
→ { "status": 400, ... }  ← [ApiController] のバリデーションエラー
```

✅ **確認**：存在しない ID を取得したとき `404` が返ること。  
ターミナルのログに `Handled [404]:` と出ること。

---

## 11. 認証・認可（JWT）

### このステップのゴール
ログイン API でトークンを発行し、`[Authorize]` のついたエンドポイントを保護する。

---

### JWT の仕組みを理解する

```
クライアント                          サーバー
  1. POST /api/auth/login
     { email, password }    →    DB でユーザー確認
                            ←    JWT トークン（文字列）

  2. GET /api/products
     Authorization: Bearer <token>   →   署名を検証（DB 不要）
                                     ←   データ

JWT = Header.Payload.Signature を Base64 でつないだ文字列
サーバーは署名を検証するだけで DB に問い合わせなくていい = 高速
```

---

### Step 1: JWT 認証設定を Extensions に書く

```csharp
// Extensions/ServiceExtensions.cs
using System.Text;
using Microsoft.AspNetCore.Authentication.JwtBearer;
using Microsoft.IdentityModel.Tokens;

namespace MyApi.Extensions;

public static class ServiceExtensions
{
    public static IServiceCollection AddJwtAuthentication(
        this IServiceCollection services, IConfiguration config)
    {
        var jwtConfig = config.GetSection("Jwt");
        var key       = Encoding.UTF8.GetBytes(jwtConfig["Key"]!);

        services
            .AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
            .AddJwtBearer(options =>
            {
                options.TokenValidationParameters = new TokenValidationParameters
                {
                    ValidateIssuer           = true,
                    ValidateAudience         = true,
                    ValidateLifetime         = true,
                    ValidateIssuerSigningKey = true,
                    ValidIssuer              = jwtConfig["Issuer"],
                    ValidAudience            = jwtConfig["Audience"],
                    IssuerSigningKey         = new SymmetricSecurityKey(key),
                    // 有効期限のズレを許容しない（デフォルトは 5 分許容）
                    ClockSkew                = TimeSpan.Zero
                };
            });

        return services;
    }
}
```

---

### Step 2: AuthController を作る

```csharp
// Controllers/AuthController.cs
using System.IdentityModel.Tokens.Jwt;
using System.Security.Claims;
using System.Text;
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc;
using Microsoft.IdentityModel.Tokens;
using MyApi.Models.Requests;

namespace MyApi.Controllers;

[ApiController]
[Route("api/auth")]
public class AuthController : ControllerBase
{
    private readonly IConfiguration _config;
    private readonly ILogger<AuthController> _logger;

    public AuthController(IConfiguration config, ILogger<AuthController> logger)
    {
        _config = config;
        _logger = logger;
    }

    /// <summary>ログインしてJWTトークンを取得</summary>
    [HttpPost("login")]
    [AllowAnonymous] // このエンドポイントは認証不要
    [ProducesResponseType(StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status401Unauthorized)]
    public IActionResult Login([FromBody] LoginRequest request)
    {
        // 実務では DB でユーザーを検索し、ハッシュ化されたパスワードを検証する
        // ここではチュートリアル用の固定値
        if (request.Email != "admin@example.com" || request.Password != "password1")
            return Unauthorized(new { message = "メールアドレスまたはパスワードが違います" });

        var token     = GenerateJwt(request.Email, ["Admin", "User"]);
        var expiresAt = DateTime.UtcNow.AddMinutes(
            int.Parse(_config["Jwt:ExpiryMinutes"] ?? "60"));

        _logger.LogInformation("Login success: {Email}", request.Email);

        return Ok(new { token, expiresAt });
    }

    /// <summary>自分の情報を取得（認証確認用）</summary>
    [HttpGet("me")]
    [Authorize]
    public IActionResult Me()
    {
        // JWT のクレームからユーザー情報を取得
        var email  = User.FindFirst(ClaimTypes.Email)?.Value;
        var roles  = User.FindAll(ClaimTypes.Role).Select(c => c.Value);
        return Ok(new { email, roles });
    }

    private string GenerateJwt(string email, string[] roles)
    {
        var jwtConfig = _config.GetSection("Jwt");
        var key       = new SymmetricSecurityKey(
            Encoding.UTF8.GetBytes(jwtConfig["Key"]!));

        var claims = new List<Claim>
        {
            new(ClaimTypes.Email, email),
            // Jti: トークンの一意 ID（失効リスト実装時に使う）
            new(JwtRegisteredClaimNames.Jti, Guid.NewGuid().ToString()),
        };
        // ロールをクレームに追加
        claims.AddRange(roles.Select(r => new Claim(ClaimTypes.Role, r)));

        var token = new JwtSecurityToken(
            issuer:   jwtConfig["Issuer"],
            audience: jwtConfig["Audience"],
            claims:   claims,
            expires:  DateTime.UtcNow.AddMinutes(
                int.Parse(jwtConfig["ExpiryMinutes"] ?? "60")),
            signingCredentials: new SigningCredentials(
                key, SecurityAlgorithms.HmacSha256)
        );

        return new JwtSecurityTokenHandler().WriteToken(token);
    }
}
```

---

### Step 3: Program.cs を更新する

```csharp
// Program.cs 全体（この時点での完全版）
using Microsoft.EntityFrameworkCore;
using MyApi.Data;
using MyApi.Extensions;
using MyApi.Middleware;
using MyApi.Repositories;
using MyApi.Repositories.Interfaces;
using MyApi.Services;
using MyApi.Services.Interfaces;

var builder = WebApplication.CreateBuilder(args);

// ── サービス登録 ──
builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();
builder.Services.AddMemoryCache();

builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("Default")));

builder.Services.AddScoped(typeof(IRepository<>), typeof(Repository<>));
builder.Services.AddScoped<IProductRepository, ProductRepository>();
builder.Services.AddScoped<IProductService, ProductService>();

builder.Services.AddJwtAuthentication(builder.Configuration);

// ── ミドルウェア ──
var app = builder.Build();

app.UseMiddleware<GlobalExceptionMiddleware>(); // 先頭

app.UseSwagger();
app.UseSwaggerUI();
app.UseHttpsRedirection();
app.UseAuthentication(); // ← UseAuthorization より前
app.UseAuthorization();
app.MapControllers();

app.Run();
```

---

### Step 4: 認証の動作確認

```bash
dotnet run
```

Swagger UI で以下の順に試す：

1. `POST /api/auth/login` に `{ "email": "admin@example.com", "password": "password1" }` を送る
2. レスポンスの `token` をコピーする
3. Swagger UI 右上の「Authorize」ボタンをクリックし、`Bearer <コピーしたtoken>` を入力
4. `POST /api/products` を実行する

✅ **確認**：
- 認証なしで `POST /api/products` → 401
- ログイン後トークンをセットして `POST /api/products` → 201
- `GET /api/auth/me` → ログインしたメールアドレスとロールが返る

---

## 12. Swagger

### このステップのゴール
Swagger UI で JWT 認証を試せる状態にする。
コントローラーの XML コメントがドキュメントに反映されるようにする。

---

### Step 1: Swagger に JWT 対応を追加する

```csharp
// Extensions/ServiceExtensions.cs に追記
using Microsoft.OpenApi.Models;

public static IServiceCollection AddSwaggerWithJwt(this IServiceCollection services)
{
    services.AddSwaggerGen(options =>
    {
        options.SwaggerDoc("v1", new OpenApiInfo
        {
            Title       = "商品管理 API",
            Version     = "v1",
            Description = "ASP.NET Core コントローラーベース API チュートリアル",
        });

        // Swagger UI の「Authorize」ボタンで JWT を入力できるようにする
        options.AddSecurityDefinition("Bearer", new OpenApiSecurityScheme
        {
            Type         = SecuritySchemeType.Http,
            Scheme       = "bearer",
            BearerFormat = "JWT",
            Description  = "ログイン後に取得した JWT トークンを入力してください"
        });

        options.AddSecurityRequirement(new OpenApiSecurityRequirement
        {
            {
                new OpenApiSecurityScheme
                {
                    Reference = new OpenApiReference
                    {
                        Type = ReferenceType.SecurityScheme,
                        Id   = "Bearer"
                    }
                },
                Array.Empty<string>()
            }
        });

        // /// コメントを Swagger に反映
        var xmlFile = $"{System.Reflection.Assembly.GetExecutingAssembly().GetName().Name}.xml";
        var xmlPath = Path.Combine(AppContext.BaseDirectory, xmlFile);
        if (File.Exists(xmlPath))
            options.IncludeXmlComments(xmlPath);
    });

    return services;
}
```

---

### Step 2: XML コメントを有効化する

```xml
<!-- MyApi.csproj の <PropertyGroup> に追加 -->
<GenerateDocumentationFile>true</GenerateDocumentationFile>
<!-- コメントなしの警告を抑制 -->
<NoWarn>$(NoWarn);1591</NoWarn>
```

---

### Step 3: Program.cs で AddSwaggerWithJwt() に差し替える

```csharp
// Program.cs（AddSwaggerGen() を差し替え）
using MyApi.Extensions;

// builder.Services.AddSwaggerGen(); ← 削除
builder.Services.AddSwaggerWithJwt(); // ← 追加
```

✅ **確認**：`dotnet run` 後に Swagger UI を開き「Authorize」ボタンが表示されること。  
ログインして取得したトークンをセットし、`[Authorize]` のエンドポイントが実行できること。

---

## 13. API 設計原則

### このステップのゴール
実装したコードを見直しながら REST API 設計の原則を理解する。

---

### 11-1. URI 設計

```
# 原則: URI は「リソース（名詞）」を表す。動詞は HTTP メソッドに委ねる

# NG: 動詞を URI に含める
GET  /api/getProducts
POST /api/createProduct
POST /api/deleteProduct/5

# OK: リソースを名詞・複数形で表す
GET    /api/products         一覧取得
GET    /api/products/5       1件取得
POST   /api/products         新規作成
PUT    /api/products/5       全体更新
PATCH  /api/products/5       部分更新
DELETE /api/products/5       削除
```

**ネストはリソースの親子関係を表す（2階層まで）**

```
GET    /api/products/5/reviews      商品5のレビュー一覧
POST   /api/products/5/reviews      商品5にレビュー追加

# NG: 3階層以上は可読性が落ちる
GET /api/stores/1/categories/3/products/5/reviews

# OK: クエリパラメータに逃がす
GET /api/reviews?productId=5&storeId=1
```

**CRUD で表現できないアクションの扱い**

```
POST /api/products/5/publish    商品を公開する
POST /api/orders/12/cancel      注文をキャンセルする
POST /api/auth/refresh          トークンリフレッシュ
```

---

### 11-2. HTTP メソッドと冪等性

| メソッド | 意味 | 冪等性 | 使い所 |
|---------|------|:------:|--------|
| GET | 取得 | ✅ | 一覧・1件取得 |
| POST | 作成 | ❌ | 新規作成・アクション系 |
| PUT | 全体更新 | ✅ | 全フィールドを送って上書き |
| PATCH | 部分更新 | ❌ | 一部フィールドだけ更新 |
| DELETE | 削除 | ✅ | 削除 |

**冪等性**：同じリクエストを何度送っても結果が変わらない性質。  
`DELETE /api/products/5` を 3 回送っても「5 番商品は存在しない」という状態は同じ。

```csharp
// PUT: 全フィールドが必須。送らなかったフィールドはデフォルト値になる
public class ReplaceProductRequest  // 全フィールド Required
{
    [Required] public string  Name       { get; set; } = "";
    [Required] public decimal Price      { get; set; }
    [Required] public int     Stock      { get; set; }
    [Required] public int     CategoryId { get; set; }
}

// PATCH（UpdateProductRequest）: 全フィールドが任意。null = 変更しない
public class UpdateProductRequest   // 全フィールド nullable
{
    public string?  Name  { get; set; }
    public decimal? Price { get; set; }
    public int?     Stock { get; set; }
}
```

---

### 11-3. ステータスコード

```csharp
// コントローラー内での返し方と対応するコード
return Ok(data);            // 200: GET・PUT・PATCH 成功
return CreatedAtAction(...); // 201: POST 成功（Location ヘッダーつき）
return NoContent();          // 204: DELETE 成功（ボディなし）

return BadRequest(...);      // 400: 入力値エラー・バリデーション違反
return Unauthorized(...);    // 401: 未認証（トークンなし・期限切れ）
return Forbid();             // 403: 認証済みだが権限不足
return NotFound(...);        // 404: リソースが存在しない
return Conflict(...);        // 409: 重複登録など競合
StatusCode(429, ...);        // 429: レート制限超過
```

---

### 11-4. バージョニング

API を公開後に破壊的変更をするときはバージョンを上げる。

```bash
dotnet add package Asp.Versioning.Mvc
```

```csharp
// Program.cs に追加
builder.Services
    .AddApiVersioning(options =>
    {
        options.DefaultApiVersion = new Asp.Versioning.ApiVersion(1, 0);
        options.AssumeDefaultVersionWhenUnspecified = true;
        options.ReportApiVersions = true; // レスポンスヘッダーにバージョンを含める
    });
```

```csharp
// Controllers/V1/ProductsController.cs
[ApiController]
[ApiVersion("1.0")]
[Route("api/v{version:apiVersion}/[controller]")]
public class ProductsController : ControllerBase { ... }

// Controllers/V2/ProductsController.cs（v2 で仕様変更）
[ApiController]
[ApiVersion("2.0")]
[Route("api/v{version:apiVersion}/[controller]")]
public class ProductsController : ControllerBase { ... }
```

```
# クライアントのリクエスト
GET /api/v1/products
GET /api/v2/products
```

---

### 11-5. フィルタ・ソート・ページング設計

```
# フィルタ
GET /api/products?categoryId=1&minPrice=500&maxPrice=2000

# 部分一致検索
GET /api/products?search=widget

# ソート
GET /api/products?sortBy=price&sortOrder=asc

# ページング
GET /api/products?page=2&pageSize=20

# 組み合わせ
GET /api/products?categoryId=1&search=w&sortBy=price&sortOrder=asc&page=1&pageSize=10
```

レスポンスのページング情報：

```json
{
  "items": [...],
  "totalCount": 150,
  "page": 2,
  "pageSize": 20,
  "totalPages": 8,
  "hasNext": true,
  "hasPrev": true
}
```

---

### 11-6. CORS 設定

```csharp
// Program.cs に追加
builder.Services.AddCors(options =>
{
    options.AddPolicy("Development", policy =>
        policy.AllowAnyOrigin().AllowAnyMethod().AllowAnyHeader());

    options.AddPolicy("Production", policy =>
        policy.WithOrigins("https://app.example.com", "https://admin.example.com")
              .WithMethods("GET", "POST", "PUT", "DELETE", "PATCH")
              .WithHeaders("Authorization", "Content-Type")
              .AllowCredentials());
});

var corsPolicy = app.Environment.IsDevelopment() ? "Development" : "Production";
app.UseCors(corsPolicy); // UseAuthentication より前
```

✅ **確認**：`GET /api/products?search=Widget` を叩いて絞り込み結果が返ること。

---

## 14. IP 制御・レート制限

### このステップのゴール
不審な IP を弾く・レート制限を設けてサーバーを保護する。

---

### IP 制御の流れ

```
リクエスト
    ↓
[ForwardedHeaders]    プロキシ経由でも本当のクライアント IP を取得
    ↓
[IpBlocklistMiddleware]   ブロックリストに載っている IP を弾く
    ↓
[IpWhitelistMiddleware]   管理系パスは許可 IP からしかアクセスできない
    ↓
[RateLimiter]             同一 IP からの連打を制限
    ↓
[Authentication / Authorization]
    ↓
コントローラー
```

---

### Step 1: IP 取得ヘルパーを作る

```csharp
// Helpers/IpHelper.cs
using System.Net;

namespace MyApi.Helpers;

public static class IpHelper
{
    public static string? GetClientIp(HttpContext ctx)
    {
        // ロードバランサー・リバースプロキシ経由の場合
        // X-Forwarded-For: client, proxy1, proxy2 の先頭が本来のクライアント IP
        var forwarded = ctx.Request.Headers["X-Forwarded-For"].FirstOrDefault();
        if (!string.IsNullOrEmpty(forwarded))
        {
            var ip = forwarded.Split(',')[0].Trim();
            if (IPAddress.TryParse(ip, out _))
                return ip;
        }

        return ctx.Connection.RemoteIpAddress?.MapToIPv4().ToString();
    }
}
```

> **注意**：`X-Forwarded-For` はクライアントが偽装できる。
> 本番では `KnownProxies` で信頼するプロキシを限定する。

---

### Step 2: プロキシ信頼設定を Program.cs に追加する

```csharp
// Program.cs（builder.Build() より前）
using Microsoft.AspNetCore.HttpOverrides;

builder.Services.Configure<ForwardedHeadersOptions>(options =>
{
    options.ForwardedHeaders =
        ForwardedHeaders.XForwardedFor | ForwardedHeaders.XForwardedProto;
    // 本番では信頼するロードバランサーの IP を限定する
    // options.KnownProxies.Add(IPAddress.Parse("10.0.0.1"));
});
```

```csharp
// app.UseMiddleware<GlobalExceptionMiddleware>(); の前に追加
app.UseForwardedHeaders();
```

---

### Step 3: ブロックリストサービスを作る

```csharp
// Services/IpBlocklistService.cs
using Microsoft.Extensions.Caching.Memory;

namespace MyApi.Services;

public interface IIpBlocklistService
{
    bool IsBlocked(string ip);
    void Block(string ip, TimeSpan? duration = null);
    void Unblock(string ip);
}

// IMemoryCache を使ったインメモリ実装（本番は Redis 推奨）
public class IpBlocklistService : IIpBlocklistService
{
    private readonly IMemoryCache _cache;
    private const string Prefix = "ipblock:";

    public IpBlocklistService(IMemoryCache cache) => _cache = cache;

    public bool IsBlocked(string ip)
        => _cache.TryGetValue(Prefix + ip, out _);

    public void Block(string ip, TimeSpan? duration = null)
    {
        var opts = new MemoryCacheEntryOptions();
        if (duration.HasValue)
            opts.AbsoluteExpirationRelativeToNow = duration;
        _cache.Set(Prefix + ip, true, opts);
    }

    public void Unblock(string ip)
        => _cache.Remove(Prefix + ip);
}
```

---

### Step 4: ブロックリストミドルウェアを作る

```csharp
// Middleware/IpBlocklistMiddleware.cs
using MyApi.Helpers;
using MyApi.Services;

namespace MyApi.Middleware;

public class IpBlocklistMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<IpBlocklistMiddleware> _logger;

    public IpBlocklistMiddleware(RequestDelegate next, ILogger<IpBlocklistMiddleware> logger)
    {
        _next   = next;
        _logger = logger;
    }

    // IIpBlocklistService は Scoped なのでコンストラクタではなく引数で受け取る
    public async Task InvokeAsync(HttpContext ctx, IIpBlocklistService blocklist)
    {
        var ip = IpHelper.GetClientIp(ctx);
        if (ip is not null && blocklist.IsBlocked(ip))
        {
            _logger.LogWarning("Blocked IP: {Ip} → {Path}", ip, ctx.Request.Path);
            ctx.Response.StatusCode  = StatusCodes.Status403Forbidden;
            ctx.Response.ContentType = "application/json";
            await ctx.Response.WriteAsJsonAsync(new
            {
                status  = 403,
                message = "アクセスが拒否されました"
            });
            return;
        }
        await _next(ctx);
    }
}
```

---

### Step 5: ホワイトリストミドルウェアを作る

```csharp
// Middleware/IpWhitelistMiddleware.cs
using MyApi.Helpers;

namespace MyApi.Middleware;

public class IpWhitelistMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<IpWhitelistMiddleware> _logger;
    private readonly HashSet<string> _allowedIps;
    private readonly string[] _protectedPrefixes;

    public IpWhitelistMiddleware(
        RequestDelegate next,
        ILogger<IpWhitelistMiddleware> logger,
        IConfiguration config)
    {
        _next   = next;
        _logger = logger;

        _allowedIps = config
            .GetSection("IpWhitelist:AllowedIps")
            .Get<string[]>()
            ?.ToHashSet() ?? [];

        _protectedPrefixes = config
            .GetSection("IpWhitelist:ProtectedPaths")
            .Get<string[]>() ?? ["/api/admin"];
    }

    public async Task InvokeAsync(HttpContext ctx)
    {
        var path = ctx.Request.Path.Value ?? "";
        var isProtected = _protectedPrefixes
            .Any(p => path.StartsWith(p, StringComparison.OrdinalIgnoreCase));

        if (isProtected)
        {
            var ip = IpHelper.GetClientIp(ctx);
            if (ip is null || !_allowedIps.Contains(ip))
            {
                _logger.LogWarning("Whitelist denied: {Ip} → {Path}", ip, path);
                ctx.Response.StatusCode  = StatusCodes.Status403Forbidden;
                ctx.Response.ContentType = "application/json";
                await ctx.Response.WriteAsJsonAsync(new
                {
                    status  = 403,
                    message = "アクセスが許可されていません"
                });
                return;
            }
        }
        await _next(ctx);
    }
}
```

---

### Step 6: レート制限を追加する

```csharp
// Program.cs に追加（builder.Services 部分）
using System.Threading.RateLimiting;
using Microsoft.AspNetCore.RateLimiting;

builder.Services.AddRateLimiter(options =>
{
    // ── 一般 API 用: スライディングウィンドウ ──
    // 1分間に 60リクエスト。12秒ごとに 12リクエスト補充
    // Fixed Window より自然な制限（窓の境目のバーストを防ぐ）
    options.AddSlidingWindowLimiter("api", opt =>
    {
        opt.PermitLimit       = 60;
        opt.Window            = TimeSpan.FromMinutes(1);
        opt.SegmentsPerWindow = 5;
        opt.QueueLimit        = 0; // 超過したら即拒否
    });

    // ── ログイン用: 厳しく制限（ブルートフォース対策）──
    // 5分間に 10回まで
    options.AddFixedWindowLimiter("auth", opt =>
    {
        opt.PermitLimit = 10;
        opt.Window      = TimeSpan.FromMinutes(5);
        opt.QueueLimit  = 0;
    });

    // ── 制限超過時のレスポンス ──
    options.OnRejected = async (ctx, token) =>
    {
        ctx.HttpContext.Response.StatusCode = StatusCodes.Status429TooManyRequests;

        // Retry-After: いつリトライできるかをクライアントに通知
        if (ctx.Lease.TryGetMetadata(MetadataName.RetryAfter, out var retryAfter))
            ctx.HttpContext.Response.Headers.RetryAfter =
                ((int)retryAfter.TotalSeconds).ToString();

        ctx.HttpContext.Response.ContentType = "application/json";
        await ctx.HttpContext.Response.WriteAsJsonAsync(new
        {
            status  = 429,
            message = "リクエスト数が上限を超えました。しばらくしてから再試行してください"
        }, token);
    };
});
```

---

### Step 7: ログイン失敗で自動 IP ブロックする

```csharp
// Services/LoginAttemptService.cs
using Microsoft.Extensions.Caching.Memory;

namespace MyApi.Services;

public class LoginAttemptService
{
    private readonly IMemoryCache       _cache;
    private readonly IIpBlocklistService _blocklist;
    private readonly ILogger<LoginAttemptService> _logger;

    private const int MaxFailures   = 5;   // 何回失敗でブロック
    private const int WindowMinutes = 10;  // 失敗カウントのリセット期間
    private const int BlockMinutes  = 30;  // ブロック時間

    public LoginAttemptService(
        IMemoryCache cache,
        IIpBlocklistService blocklist,
        ILogger<LoginAttemptService> logger)
    {
        _cache     = cache;
        _blocklist = blocklist;
        _logger    = logger;
    }

    public void RecordFailure(string ip)
    {
        var key   = $"loginfail:{ip}";
        var count = _cache.GetOrCreate(key, entry =>
        {
            entry.AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(WindowMinutes);
            return 0;
        });

        count++;
        _cache.Set(key, count, TimeSpan.FromMinutes(WindowMinutes));

        _logger.LogWarning("Login failure {Count}/{Max} from {Ip}", count, MaxFailures, ip);

        if (count >= MaxFailures)
        {
            _blocklist.Block(ip, TimeSpan.FromMinutes(BlockMinutes));
            _logger.LogWarning("IP auto-blocked {Min}min: {Ip}", BlockMinutes, ip);
        }
    }

    public void RecordSuccess(string ip)
        => _cache.Remove($"loginfail:{ip}"); // 成功したら失敗カウントをリセット
}
```

`AuthController` に組み込む：

```csharp
// Controllers/AuthController.cs の Login メソッドを更新
private readonly LoginAttemptService _attemptService;

public AuthController(
    IConfiguration config,
    ILogger<AuthController> logger,
    LoginAttemptService attemptService)
{
    _config         = config;
    _logger         = logger;
    _attemptService = attemptService;
}

[HttpPost("login")]
[AllowAnonymous]
[EnableRateLimiting("auth")] // ログインには厳しいレート制限を適用
public IActionResult Login([FromBody] LoginRequest request)
{
    var ip = MyApi.Helpers.IpHelper.GetClientIp(HttpContext) ?? "unknown";

    if (request.Email != "admin@example.com" || request.Password != "password1")
    {
        _attemptService.RecordFailure(ip);
        // 存在しないメールと間違いパスワードで同じメッセージを返す（ユーザー列挙防止）
        return Unauthorized(new { message = "メールアドレスまたはパスワードが違います" });
    }

    _attemptService.RecordSuccess(ip);
    var token     = GenerateJwt(request.Email, ["Admin", "User"]);
    var expiresAt = DateTime.UtcNow.AddMinutes(int.Parse(_config["Jwt:ExpiryMinutes"] ?? "60"));

    return Ok(new { token, expiresAt });
}
```

---

### Step 8: Program.cs の最終版を組み上げる

```csharp
// Program.cs（完成版）
using Microsoft.AspNetCore.HttpOverrides;
using Microsoft.EntityFrameworkCore;
using MyApi.Data;
using MyApi.Extensions;
using MyApi.Middleware;
using MyApi.Repositories;
using MyApi.Repositories.Interfaces;
using MyApi.Services;
using MyApi.Services.Interfaces;

var builder = WebApplication.CreateBuilder(args);

// ── サービス登録 ──
builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddMemoryCache();

builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("Default")));

builder.Services.AddScoped(typeof(IRepository<>), typeof(Repository<>));
builder.Services.AddScoped<IProductRepository, ProductRepository>();
builder.Services.AddScoped<IProductService, ProductService>();
builder.Services.AddSingleton<IIpBlocklistService, IpBlocklistService>();
builder.Services.AddScoped<LoginAttemptService>();

builder.Services.AddJwtAuthentication(builder.Configuration);
builder.Services.AddSwaggerWithJwt();
builder.Services.AddRateLimiter(/* ... 省略 ... */);

builder.Services.Configure<ForwardedHeadersOptions>(options =>
{
    options.ForwardedHeaders =
        ForwardedHeaders.XForwardedFor | ForwardedHeaders.XForwardedProto;
});

// ── ミドルウェアパイプライン（順序が重要）──
var app = builder.Build();

app.UseForwardedHeaders();                          // 1. プロキシヘッダー解釈
app.UseMiddleware<GlobalExceptionMiddleware>();      // 2. 全例外をキャッチ
app.UseMiddleware<IpBlocklistMiddleware>();          // 3. ブロックリスト
app.UseMiddleware<IpWhitelistMiddleware>();          // 4. ホワイトリスト
app.UseRateLimiter();                               // 5. レート制限
app.UseSwagger();
app.UseSwaggerUI();
app.UseHttpsRedirection();
app.UseAuthentication();                            // 6. 認証（Authorizationより前）
app.UseAuthorization();                             // 7. 認可
app.MapControllers();

app.Run();
```

---

### Step 9: 最終動作確認

```bash
dotnet run
```

以下の順で Swagger UI から確認する：

| # | 操作 | 期待する結果 |
|---|------|------------|
| 1 | `GET /api/products` | シードデータ 3 件が返る |
| 2 | `GET /api/products/9999` | 404 + エラーメッセージ |
| 3 | `POST /api/products`（認証なし） | 401 |
| 4 | `POST /api/auth/login`（正しい情報） | トークン取得 |
| 5 | トークンをセットして `POST /api/products` | 201 + 作成した商品 |
| 6 | `POST /api/auth/login`（間違いを 5 回） | 5 回目以降 403（IP ブロック） |
| 7 | `GET /api/products?search=Widget` | 絞り込み結果 |
| 8 | `GET /api/products?page=1&pageSize=2` | 2 件 + ページング情報 |

✅ **全ての確認が通ればチュートリアル完了。**

---

## 最終チェックリスト

| カテゴリ | 項目 |
|---------|------|
| **設計** | リポジトリパターンで DB アクセスを Service から分離した |
| **設計** | エンティティを直接 API レスポンスとして返していない（DTO 使用） |
| **設計** | コントローラーにビジネスロジックを書いていない |
| **設計** | インターフェース経由で DI している（差し替え可能） |
| **設計** | PUT と PATCH の使い分けができている |
| **設計** | ステータスコードを適切に返している（201/204/404/409 など） |
| **セキュリティ** | JWT 認証を実装した |
| **セキュリティ** | シークレットを `user-secrets` で管理した（appsettings に直書きしない） |
| **セキュリティ** | HTTPS を強制した |
| **セキュリティ** | IP ホワイトリストで管理系パスを保護した |
| **セキュリティ** | ログイン失敗の連続で IP を自動ブロックした |
| **信頼性** | グローバル例外ハンドラーを実装した |
| **信頼性** | 全 I/O 処理を `async/await` で実装した |
| **パフォーマンス** | 参照クエリに `AsNoTracking()` を適用した |
| **パフォーマンス** | 一覧取得にページングを実装した |
| **パフォーマンス** | `Include()` で N+1 を防いだ |
| **運用** | Swagger（JWT 対応）で API ドキュメントを生成した |
| **運用** | レート制限を設定した（一般用・ログイン用） |
| **運用** | `CreatedAt` / `UpdatedAt` を DbContext で自動管理した |

---

*対象バージョン: .NET 8 / C# 12 / EF Core 8 / Asp.Versioning 8*
