# React + Vite Feature-based アーキテクチャ導入提案

**対象：** DXチーム内部  
**目的：** 新フロントエンド開発の設計指針策定

---

## 1. 現状の課題（Classic ASP の構成）

現行システムは Classic ASP で構築されており、ファイルの役割をサフィックスで区別する命名規則を採用している。

### 現行命名規則の例

| サフィックス | 役割 | 例 |
|---|---|---|
| `f` | 画面（フォーム） | `123f.asp` |
| `b` | DBアクセス（バックエンド処理） | `123b.asp` |
| `w` | モーダル・サブウィンドウ | `123w.asp` |

メニュー単位では `Resw`（埼玉工場業務）のようなグループ名で機能をまとめているが、**ファイルの物理的な配置は機能ではなく種別（f/b/w）で分散**している。

### この構成の問題点

- **関連ファイルが離れている**  
  ある機能を修正するとき、`123f`・`123b`・`123w` を別々に探す必要がある。

- **影響範囲が把握しにくい**  
  `123b` を変更したとき、どの画面（f）から呼ばれているか追跡が困難。

- **新メンバーが迷う**  
  ファイル命名規則を知らないと、どこに何があるか判断できない。

- **機能追加のたびに横断的な変更が発生する**  
  1つの機能を追加するだけで、複数の種別ディレクトリに散らばったファイルを触ることになる。

---

## 2. Feature-based アーキテクチャとは

> **「機能（Feature）を軸にファイルを集約する」** 設計思想。

種別（画面/DB/モーダル）ではなく、**業務機能ごとにディレクトリを切る**。
1つの機能に関係するすべてのファイルが1か所に集まる。

### 考え方の対比

```
【現行 Classic ASP：種別軸】          【Feature-based：機能軸】

/pages/                               /features/
  123f.asp  ← 受注入力 画面              saitama-orders/       ← 受注管理（埼玉）
  456f.asp  ← 在庫照会 画面                ├── OrderForm.tsx       画面
  123b.asp  ← 受注入力 DBアクセス           ├── orderApi.ts         APIアクセス
  456b.asp  ← 在庫照会 DBアクセス           ├── OrderDetailModal.tsx モーダル
  123w.asp  ← 受注入力 モーダル             ├── useOrderForm.ts     ロジック
  456w.asp  ← 在庫照会 モーダル             └── index.ts            公開IF
                                        saitama-inventory/    ← 在庫照会（埼玉）
                                          ├── InventoryList.tsx
                                          ├── inventoryApi.ts
                                          └── index.ts
```

---

## 3. 提案するディレクトリ構成

現行の **ASPメニュー名 → Feature ディレクトリ名** に対応させる。

```
src/
├── features/                        ← 業務機能の本体（ここが中心）
│   │
│   ├── saitama-orders/              ← Resw「埼玉工場業務 > 受注管理」に相当
│   │   ├── components/
│   │   │   ├── OrderForm.tsx        ← 旧 123f.asp（入力画面）
│   │   │   ├── OrderDetailModal.tsx ← 旧 123w.asp（モーダル）
│   │   │   └── OrderTable.tsx
│   │   ├── hooks/
│   │   │   └── useOrderForm.ts      ← フォームのロジック
│   │   ├── api/
│   │   │   └── orderApi.ts          ← 旧 123b.asp（DBアクセス）
│   │   ├── types/
│   │   │   └── order.ts
│   │   └── index.ts                 ← 外部への公開インターフェース
│   │
│   ├── saitama-inventory/           ← Resw「埼玉工場業務 > 在庫照会」に相当
│   │   ├── components/
│   │   ├── hooks/
│   │   ├── api/
│   │   └── index.ts
│   │
│   └── （他メニュー機能）
│
├── shared/                          ← 複数 Feature をまたぐ共通部品
│   ├── components/                  ← 汎用UIコンポーネント（ボタン、テーブル等）
│   ├── hooks/                       ← 汎用フック
│   ├── utils/                       ← 共通ユーティリティ
│   └── types/                       ← 共通型定義
│
├── pages/                           ← ルーティング用の薄いページ層
│   ├── SaitamaOrdersPage.tsx        ← features/saitama-orders を呼ぶだけ
│   └── SaitamaInventoryPage.tsx
│
└── app/
    ├── router.tsx                   ← ルート定義
    └── App.tsx
```

### ASPメニューとの対応イメージ

```
【ASP メニュー構造】              【Feature ディレクトリ】

Resw（埼玉工場業務）
  ├── 受注管理          →    features/saitama-orders/
  ├── 在庫照会          →    features/saitama-inventory/
  ├── 出荷指示          →    features/saitama-shipping/
  └── 仕掛品確認        →    features/saitama-wip/

（他工場・他メニューも同様）
```

---

## 4. Feature 内部の責務分担

旧来の `f / b / w` サフィックスが担っていた役割を、Feature 内の**サブディレクトリ**で明示的に表現する。

| 旧 ASP | 新 Feature 内の位置 | 内容 |
|---|---|---|
| `123f.asp` | `components/OrderForm.tsx` | 入力・表示画面 |
| `123b.asp` | `api/orderApi.ts` | APIコール・データ取得 |
| `123w.asp` | `components/OrderDetailModal.tsx` | モーダル・サブ画面 |
| ―（暗黙） | `hooks/useOrderForm.ts` | 画面ロジック・状態管理 |
| ―（暗黙） | `types/order.ts` | 型定義 |

**命名規則の暗記が不要になり、ディレクトリを見れば役割がわかる。**

---

## 5. 導入のメリット

### 5-1. 変更の影響範囲が Feature 内に収まる

受注管理の仕様変更は `features/saitama-orders/` の中だけを触ればよい。
他 Feature への波及が起きにくく、**レビューと動作確認のスコープが明確**になる。

### 5-2. 機能単位で担当を分けやすい

```
Aさん → features/saitama-orders/    を担当
Bさん → features/saitama-inventory/ を担当
```

ファイルの衝突が減り、**並行開発がしやすい**。

### 5-3. 削除・廃止が安全にできる

機能を廃止するとき、`features/該当機能/` ディレクトリを丸ごと削除すればよい。
横断的なファイルを探し回る必要がない。

### 5-4. 新メンバーが迷わない

「どのメニューの画面？」→「その Feature ディレクトリを開く」  
ASP時代のサフィックス規則を覚えなくても、**業務知識でコードが探せる**。

### 5-5. テストが書きやすい

Feature 単位で API・ロジック・UIが揃っているため、  
`features/saitama-orders/__tests__/` にまとめてテストを置ける。

---

## 6. 守るべきルール（依存方向）

Feature-based を機能させるために、**依存の向きを一方向に保つ**ことが重要。

```
【OK】
pages/        → features/（ページはFeatureを呼ぶ）
features/     → shared/（FeatureはSharedを使う）

【NG：やってはいけない】
features/saitama-orders/  →  features/saitama-inventory/
（Feature同士が直接参照し合うと、Classic ASPと同じ問題が再発する）
```

Feature 間で共通化したいものが出てきたら、**`shared/` に昇格させる**ルールにする。

---

## 7. 段階的な移行イメージ

いきなり全機能を作り直す必要はない。新規画面から Feature-based で作り始め、既存機能は必要に応じて順次移行する。

```
Phase 1（新規開発）
  新しい画面・機能は最初から Feature-based で作る

Phase 2（順次移行）
  改修が発生した既存機能を Feature-based に切り出す
  ※ 触らない機能は無理に移行しない

Phase 3（整理）
  shared/ の共通部品を整備し、チーム全体のルールとして定着させる
```

---

## 8. まとめ

| 観点 | Classic ASP（現行） | React + Feature-based（提案） |
|---|---|---|
| ファイルの探し方 | サフィックス（f/b/w）を覚える | メニュー名 → Feature ディレクトリを開く |
| 変更の影響範囲 | 種別をまたいで散在 | Feature 内に集約 |
| 並行開発 | ファイル衝突が起きやすい | Feature 単位で分担しやすい |
| 機能の廃止 | 横断的なファイルを探す | ディレクトリを丸ごと削除 |
| 新メンバーの習熟 | 命名規則の暗記が必要 | 業務知識だけで探せる |

**ASP時代の「メニュー名で業務を把握する」感覚をそのまま引き継ぎながら、  
モダンな構成でチームが迷わず開発できる土台を作る。**  
それが Feature-based アーキテクチャ導入の狙いです。
