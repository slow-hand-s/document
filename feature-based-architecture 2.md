# React + Vite Feature-based アーキテクチャ 構築手順
> Angular 経験者向け解説付き

---

## はじめに：Angular との対応表

React には Angular のような「フレームワーク公式の構成ルール」がない。  
その代わり、よく使われるライブラリを組み合わせて Angular に近い構成を作る。

| Angular | React 相当 | 備考 |
|---|---|---|
| `NgModule` | `feature/index.ts` | 機能の境界を明示する |
| `Component` | `*.tsx` ファイル | テンプレートと TS が1ファイルに同居 |
| `Service` + `HttpClient` | `api/*.ts` + axios | DI はなく、直接 import して使う |
| `Injectable` サービス | `hooks/use*.ts` | カスタムフック = 再利用可能なロジック |
| `RxJS Observable` | TanStack Query | 非同期データ取得・キャッシュを担当 |
| `NgRx` / `BehaviorSubject` | Zustand | グローバル状態管理 |
| `RouterModule` | TanStack Router | ルート定義の方法が似ている |
| `router-outlet` | `<Outlet />` | 子ルートの描画場所 |
| `*ngIf` / `*ngFor` | `{condition && <X />}` / `.map()` | JSX 内に直接 JS を書く |

---

## 全体像

```
依存方向: pages → features → shared（逆方向は禁止）

src/
├── app/        アプリ起動・ルーター・Provider の設定
├── pages/      URL に対応する薄いページ（ほぼ feature を並べるだけ）
├── features/   業務機能ごとのドメイン（ここが主役）
└── shared/     全 feature が使う共通部品
```

---

## STEP 1: Vite プロジェクト作成

Angular CLI の `ng new` に相当する。

```bash
pnpm create vite my-app --template react-ts
cd my-app
pnpm install
```

> **Vite とは？**  
> Angular の webpack に相当するビルドツール。起動が速い。

---

## STEP 2: 必須パッケージの導入

```bash
# ルーティング（Angular の RouterModule 相当）
pnpm add @tanstack/react-router

# APIデータ取得・キャッシュ（RxJS + HttpClient の代替）
pnpm add @tanstack/react-query axios

# グローバル状態管理（NgRx の軽量版）
pnpm add zustand

# CSS ユーティリティ（Angular Material の代わりに使うことが多い）
pnpm add -D tailwindcss @tailwindcss/vite
```

---

## STEP 3: パスエイリアス設定

`../../shared/lib/apiClient` のような深い相対パスを避けるための設定。  
Angular の `tsconfig.paths` と同じ仕組み。

### `vite.config.ts`

```ts
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'
import tailwindcss from '@tailwindcss/vite'
import path from 'path'

export default defineConfig({
  plugins: [react(), tailwindcss()],
  resolve: {
    alias: {
      '@':         path.resolve(__dirname, './src'),
      '@features': path.resolve(__dirname, './src/features'),
      '@shared':   path.resolve(__dirname, './src/shared'),
      '@pages':    path.resolve(__dirname, './src/pages'),
      '@app':      path.resolve(__dirname, './src/app'),
    },
  },
})
```

### `tsconfig.json`（paths に追記）

```json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@/*":         ["./src/*"],
      "@features/*": ["./src/features/*"],
      "@shared/*":   ["./src/shared/*"],
      "@pages/*":    ["./src/pages/*"],
      "@app/*":      ["./src/app/*"]
    }
  }
}
```

これで `import { inspectionApi } from '@features/inspection'` のように書ける。

---

## STEP 4: ディレクトリ骨格の作成

```bash
mkdir -p src/{app,pages,features,shared/{components/{ui,layout},config,hooks,lib,types,utils}}
```

完成形：

```
src/
├── app/
│   ├── router.tsx             # ルート定義（RouterModule 相当）
│   ├── providers.tsx          # QueryClient など Provider の設定
│   └── App.tsx
├── pages/
│   ├── dashboard/
│   │   └── index.tsx
│   └── saitama/
│       └── inspection/
│           └── list.tsx       # 日常点検結果参照ページ
├── features/
│   ├── auth/                  # 認証機能
│   │   ├── components/
│   │   ├── hooks/
│   │   ├── api/
│   │   └── index.ts           # ← NgModule の exports 相当
│   └── inspection/            # 日常点検ドメイン
│       ├── components/
│       │   └── InspectionResultTable.tsx
│       ├── hooks/
│       │   └── useInspectionResult.ts
│       ├── api/
│       │   └── inspectionApi.ts
│       └── index.ts
├── shared/
│   ├── components/
│   │   ├── ui/                # shadcn/ui ラッパー（共通 UI 部品）
│   │   └── layout/
│   │       ├── AppShell.tsx   # サイドバー + メインエリアのレイアウト
│   │       └── SideNav.tsx    # menuConfig を読んでサイドバーを描画
│   ├── config/
│   │   └── menuConfig.ts      # ★ メニュー定義（一元管理）
│   ├── hooks/                 # 全 feature で使う汎用フック
│   ├── lib/
│   │   └── apiClient.ts       # axios インスタンス（HttpClient 相当）
│   ├── types/
│   │   └── common.ts          # 共通型定義
│   └── utils/
└── main.tsx                   # エントリポイント（main.ts 相当）
```

---

## STEP 5: shared レイヤーの実装

まず「全 feature が使う共通部品」から作る。

### `src/shared/lib/apiClient.ts`

Angular の `HttpClient` に相当。axios でベース URL・ヘッダー・認証を一箇所で設定する。

```ts
import axios from 'axios'

// axios.create() でベース設定済みのインスタンスを作る
// Angular の HttpClient を AppModule で設定するイメージ
export const apiClient = axios.create({
  baseURL: import.meta.env.VITE_API_BASE_URL, // .env ファイルから読む
  headers: { 'Content-Type': 'application/json' },
})

// リクエスト前に自動で認証トークンを付ける（HttpInterceptor 相当）
apiClient.interceptors.request.use((config) => {
  const token = localStorage.getItem('token')
  if (token) config.headers.Authorization = `Bearer ${token}`
  return config
})
```

### `src/shared/types/common.ts`

API レスポンスの共通型。全 feature で使い回す。

```ts
// バックエンドが返す標準レスポンス形式
export type ApiResponse<T> = {
  data: T
  message: string
  success: boolean
}

// ページネーション付きリスト
export type PaginatedResponse<T> = {
  items: T[]
  total: number
  page: number
  pageSize: number
}
```

---

## STEP 6: feature の実装（inspection を例に）

feature は **api → hooks → components → index.ts** の順に作る。

### 6-1. `src/features/inspection/api/inspectionApi.ts`

Angular の Service + HttpClient 呼び出しに相当。  
ここに API コールをまとめておくと、URL が変わったときに1ファイルだけ直せばよい。

```ts
import { apiClient } from '@shared/lib/apiClient'
import type { ApiResponse, PaginatedResponse } from '@shared/types/common'

// この feature で扱うデータの型（Angular の interface 定義と同じ）
export type InspectionResult = {
  id: number
  equipmentId: number
  equipmentName: string
  inspectedAt: string        // ISO 8601 形式の日時文字列
  result: 'ok' | 'ng' | 'warning'
  note: string
}

// API 呼び出し関数をオブジェクトにまとめる
export const inspectionApi = {
  // 一覧取得（ページネーション対応）
  getList: (page = 1, pageSize = 20) =>
    apiClient
      .get<PaginatedResponse<InspectionResult>>('/inspection', {
        params: { page, pageSize },
      })
      .then((r) => r.data),

  // 1件取得
  getById: (id: number) =>
    apiClient
      .get<ApiResponse<InspectionResult>>(`/inspection/${id}`)
      .then((r) => r.data),
}
```

### 6-2. `src/features/inspection/hooks/useInspectionResult.ts`

> **カスタムフックとは？**  
> `use` で始まる関数で、コンポーネントから「データ取得ロジック」を切り出したもの。  
> Angular の Injectable サービスに近い概念だが、DI ではなく import で使う。

TanStack Query の `useQuery` は RxJS の Observable に近いが、  
**キャッシュ・ローディング状態・エラー状態**を自動で管理してくれる。

```ts
import { useQuery } from '@tanstack/react-query'
import { inspectionApi } from '../api/inspectionApi'

// キャッシュのキー。同じキーなら同じデータを使い回す
const QUERY_KEY = 'inspection'

// 一覧取得フック
// コンポーネントから呼ぶと { data, isLoading, error } が返る
export const useInspectionResultList = (page = 1) =>
  useQuery({
    queryKey: [QUERY_KEY, page],           // ページが変わると再取得される
    queryFn: () => inspectionApi.getList(page),
  })

// 詳細取得フック
export const useInspectionResultDetail = (id: number) =>
  useQuery({
    queryKey: [QUERY_KEY, id],
    queryFn: () => inspectionApi.getById(id),
    enabled: !!id,  // id が未設定のときは API を叩かない
  })
```

### 6-3. `src/features/inspection/components/InspectionResultTable.tsx`

> **JSX とは？**  
> Angular の HTML テンプレートに相当するが、TypeScript ファイルの中に直接書く。  
> `*ngIf` は `{condition && <要素>}`、`*ngFor` は `.map()` で書く。

```tsx
import { useInspectionResultList } from '../hooks/useInspectionResult'

export function InspectionResultTable() {
  // フックを呼ぶだけでデータ取得・ローディング状態が得られる
  const { data, isLoading, error } = useInspectionResultList()

  // Angular の *ngIf="isLoading" 相当
  if (isLoading) return <div>読み込み中...</div>
  if (error)     return <div>エラーが発生しました</div>

  return (
    <table className="w-full text-sm">
      <thead>
        <tr>
          <th>設備名</th>
          <th>点検日時</th>
          <th>結果</th>
          <th>備考</th>
        </tr>
      </thead>
      <tbody>
        {/* Angular の *ngFor="let item of items" 相当 */}
        {data?.items.map((item) => (
          <tr key={item.id}>  {/* key は Angular の trackBy 相当 */}
            <td>{item.equipmentName}</td>
            <td>{item.inspectedAt}</td>
            <td>{item.result}</td>
            <td>{item.note}</td>
          </tr>
        ))}
      </tbody>
    </table>
  )
}
```

### 6-4. `src/features/inspection/index.ts`（public API）

NgModule の `exports` 相当。**外部に公開するものだけをここに書く。**  
他の feature やページは必ずこのファイル経由で import する。

```ts
// ✅ 外部に公開するもの
export { InspectionResultTable } from './components/InspectionResultTable'
export { useInspectionResultList, useInspectionResultDetail } from './hooks/useInspectionResult'
export type { InspectionResult } from './api/inspectionApi'

// ❌ 内部実装（api/ など）は export しない
//    他の feature から直接 api/ を触られると結合度が上がるため
```

---

## STEP 7: pages レイヤーの実装

pages は **薄いレイヤー**。feature を import してレイアウトするだけ。  
ビジネスロジックは絶対に書かない。

```tsx
// src/pages/saitama/inspection/list.tsx
import { InspectionResultTable } from '@features/inspection'
// ↑ index.ts 経由で import。api/ や hooks/ を直接 import しない

export default function InspectionListPage() {
  return (
    <div>
      <h1 className="text-xl font-semibold mb-4">日常点検結果参照</h1>
      <InspectionResultTable />
    </div>
  )
}
```

---

## STEP 8: app レイヤーの実装

アプリ全体の設定を担う。Angular の `AppModule` に相当。

### `src/app/providers.tsx`

TanStack Query を使うには、アプリ全体を `QueryClientProvider` で囲む必要がある。  
Angular の `AppModule` で `HttpClientModule` を imports するのと同じ発想。

```tsx
import { QueryClient, QueryClientProvider } from '@tanstack/react-query'

// キャッシュ設定。アプリ全体で1つだけ作る。
const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      retry: 1,           // 失敗時に1回だけリトライ
      staleTime: 1000 * 60, // 1分間はキャッシュを新鮮とみなす
    },
  },
})

// children を QueryClient の管理下に置く
// Angular の NgModule providers に登録するイメージ
export function Providers({ children }: { children: React.ReactNode }) {
  return (
    <QueryClientProvider client={queryClient}>
      {children}
    </QueryClientProvider>
  )
}
```

### `src/app/router.tsx`

rootRoute に `AppShell`（サイドバー常駐レイアウト）を当て、  
子ルートのページが `<Outlet />` に描画される構造にする。

```tsx
import { createRouter, createRoute, createRootRoute } from '@tanstack/react-router'
import { lazy, Suspense } from 'react'
import { AppShell } from '@shared/components/layout/AppShell'

// lazy() = 画面を開いたときに初めてファイルを読み込む（遅延ロード）
// Angular の loadChildren: () => import(...) と同じ
const DashboardPage      = lazy(() => import('@pages/dashboard'))
const InspectionListPage = lazy(() => import('@pages/saitama/inspection/list'))

// Suspense = lazy ロード中に fallback を表示するラッパー
const Lazy = ({ Page }: { Page: React.LazyExoticComponent<() => JSX.Element> }) => (
  <Suspense fallback={<div className="p-6 text-sm text-muted-foreground">読み込み中...</div>}>
    <Page />
  </Suspense>
)

// rootRoute = 全ページ共通のレイアウト（AppShell = サイドバー）
// Angular の RouterModule の component: AppComponent に相当
const rootRoute = createRootRoute({ component: AppShell })

const dashboardRoute = createRoute({
  getParentRoute: () => rootRoute,
  path: '/',
  component: () => <Lazy Page={DashboardPage} />,
})

const inspectionListRoute = createRoute({
  getParentRoute: () => rootRoute,
  path: '/saitama/inspection/list',   // ← menuConfig の path と必ず一致させる
  component: () => <Lazy Page={InspectionListPage} />,
})

export const router = createRouter({
  routeTree: rootRoute.addChildren([
    dashboardRoute,
    inspectionListRoute,
    // 新しいメニュー項目はここに追加
  ]),
})
```

### `src/main.tsx`

Angular の `main.ts` に相当するエントリポイント。

```tsx
import React from 'react'
import ReactDOM from 'react-dom/client'
import { RouterProvider } from '@tanstack/react-router'
import { Providers } from '@app/providers'
import { router } from '@app/router'
import './index.css'

ReactDOM.createRoot(document.getElementById('root')!).render(
  <React.StrictMode>
    {/* Providers = QueryClient など全体設定のラッパー */}
    <Providers>
      {/* RouterProvider = Angular の router-outlet の起点 */}
      <RouterProvider router={router} />
    </Providers>
  </React.StrictMode>
)
```

---

## STEP 9: メニューナビゲーションの構築

メニュー定義・ルーティング・feature を**疎結合**に保つ。  
新しい画面を追加するときは `menuConfig.ts` と `router.tsx` の2ファイルだけ変更すればよい。

### メニュー構造 ↔ URL ↔ feature の対応

```
メニュー階層                        URL                              feature
───────────────────────────────────────────────────────────────────────────
埼玉工場業務
  └─ 日常点検結果参照              /saitama/inspection/list         inspection
  └─ 設備マスタ                    /saitama/equipment/list          equipment
```

### `src/shared/config/menuConfig.ts`

メニューの構造をデータとして定義する。  
SideNav コンポーネントはこれを読んでサイドバーを描画するので、  
**メニューを追加・変更するときはここだけ触ればよい。**

```ts
export type MenuItem = {
  label: string
  path?: string          // 末端メニューのみ設定（子を持つ項目は不要）
  children?: MenuItem[]
  feature?: string       // どの feature か記録しておく（権限管理などで使う）
}

export const menuConfig: MenuItem[] = [
  {
    label: '埼玉工場業務',
    children: [
      {
        label: '日常点検結果参照',
        path: '/saitama/inspection/list',
        feature: 'inspection',
      },
      {
        label: '設備マスタ',
        path: '/saitama/equipment/list',
        feature: 'equipment',
      },
    ],
  },
]
```

### `src/shared/components/layout/SideNav.tsx`

`menuConfig` を再帰的に走査して、アコーディオン式サイドバーを描画する。  
メニュー項目の追加は `menuConfig.ts` だけで完結するので、このファイルは触らない。

```tsx
import { useState } from 'react'
import { Link, useLocation } from '@tanstack/react-router'
import { menuConfig, type MenuItem } from '@shared/config/menuConfig'
import { ChevronDown, ChevronRight } from 'lucide-react'
import { cn } from '@shared/utils/cn'

// 1つのメニュー項目を描画する（子があれば再帰的に呼ばれる）
function NavItem({ item, depth = 0 }: { item: MenuItem; depth?: number }) {
  // useState = Angular の変数宣言 + ChangeDetection のイメージ
  // open の値が変わると、このコンポーネントが再描画される
  const [open, setOpen] = useState(false)

  const { pathname } = useLocation()
  const isActive = item.path === pathname  // 現在のURLと一致したらハイライト

  // 子メニューを持つ項目 → クリックで開閉するアコーディオン
  if (item.children) {
    return (
      <div>
        <button
          onClick={() => setOpen((v) => !v)}  // クリックで open を反転
          className="w-full flex items-center justify-between px-3 py-2 text-sm hover:bg-muted rounded-md"
          style={{ paddingLeft: `${(depth + 1) * 12}px` }}
        >
          <span className={depth === 0 ? 'font-medium' : ''}>{item.label}</span>
          {/* open の値によってアイコンを切り替える（Angular の *ngIf 相当） */}
          {open ? <ChevronDown size={14} /> : <ChevronRight size={14} />}
        </button>
        {/* open が true のときだけ子メニューを表示 */}
        {open && (
          <div>
            {item.children.map((child) => (
              // 再帰的に NavItem を呼ぶ（depth を+1して字下げを増やす）
              <NavItem key={child.label} item={child} depth={depth + 1} />
            ))}
          </div>
        )}
      </div>
    )
  }

  // 子を持たない末端メニュー → クリックでページ遷移するリンク
  return (
    <Link
      to={item.path!}
      className={cn(
        'block px-3 py-2 text-sm rounded-md hover:bg-muted',
        isActive && 'bg-primary text-primary-foreground font-medium'
      )}
      style={{ paddingLeft: `${(depth + 1) * 12}px` }}
    >
      {item.label}
    </Link>
  )
}

export function SideNav() {
  return (
    <nav className="w-56 h-full border-r py-4 space-y-1 overflow-y-auto">
      {menuConfig.map((item) => (
        <NavItem key={item.label} item={item} />
      ))}
    </nav>
  )
}
```

### `src/shared/components/layout/AppShell.tsx`

サイドバーとメインエリアを並べる共通レイアウト。  
`<Outlet />` の部分に、現在のURLに対応したページが描画される。  
Angular の `<router-outlet>` と全く同じ役割。

```tsx
import { Outlet } from '@tanstack/react-router'
import { SideNav } from './SideNav'

export function AppShell() {
  return (
    <div className="flex h-screen">
      <SideNav />
      <main className="flex-1 overflow-y-auto p-6">
        <Outlet />  {/* ← ここに各ページが描画される（router-outlet 相当） */}
      </main>
    </div>
  )
}
```

---

## STEP 10: shadcn/ui の導入

Button・Table・Dialog などの UI コンポーネントを追加する。  
Angular Material に相当するが、コードがプロジェクトにコピーされるため自由にカスタマイズできる。

```bash
pnpm dlx shadcn@latest init
pnpm dlx shadcn@latest add button input table dialog
```

生成された `src/components/ui/` を `src/shared/components/ui/` に移動し、  
`components.json` の alias を合わせて修正する。

```json
{
  "aliases": {
    "components": "@/shared/components",
    "utils": "@/shared/utils"
  }
}
```

---

## STEP 11: 新 feature・メニュー項目を追加するときのチェックリスト

新しい機能（例：`equipment` 設備マスタ）を追加するたびにこの手順を繰り返す。

```
□ src/features/equipment/ ディレクトリ作成
□ api/equipmentApi.ts         ← API コール定義（Service 相当）
□ hooks/useEquipment.ts       ← TanStack Query フック（Injectable 相当）
□ components/                 ← UI コンポーネント
□ stores/（必要なら）         ← Zustand store（NgRx 相当）
□ index.ts                    ← public API のみ export（NgModule.exports 相当）
□ src/pages/saitama/equipment/list.tsx  ← 薄いページコンポーネント
□ menuConfig.ts に label + path + feature を1行追加
□ router.tsx にルート1件追加
```

**メニュー追加で変更するファイルは2つだけ（menuConfig + router）。**  
feature とページは独立して追加できる。

---

## アーキテクチャルール（チーム共有用）

| ルール | 理由 |
|---|---|
| `features/X` から `features/Y` を直接 import しない | 循環依存・密結合を防ぐ |
| feature 内部実装は `index.ts` 経由のみ外部公開 | NgModule.exports と同じ発想。リファクタ時の影響を限定 |
| `pages/` にビジネスロジックを書かない | feature に寄せることでテストしやすくする |
| API コールは必ず `features/*/api/` に置く | axios の呼び出しが散乱するのを防ぐ |
| 型定義は使う場所の近くに置く（共有は `shared/types/`） | 型の迷子を防ぐ |
| メニュー定義は `menuConfig.ts` のみで管理する | SideNav とルートの二重管理を防ぐ |
| `menuConfig` の path と `router.tsx` の path を必ず一致させる | リンク切れを防ぐ |
