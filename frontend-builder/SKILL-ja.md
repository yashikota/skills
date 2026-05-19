---
name: frontend-builder
description: Vite+、React、TypeScript、TanStack Router でフロントエンドアプリを構築する
license: MIT
---

# Frontend Builder

Vite+ の `vp` CLI + React + TypeScript を標準構成として、フロントエンドを構築する。ルーティングには TanStack Router を使う。

## 基本方針

- 新規作成では Vite+ (`vp`) で React + TypeScript プロジェクトを作る。
- 既存 Vite プロジェクトを Vite+ 化する場合は `vp migrate` を使う。
- パッケージマネージャーには `pnpm` を使う。
- 必要に応じて `playwright-cli` を使う。
- コード変更後は毎回 `vp check --fix` を行う。
- ライフサイクルコマンドは可能な限り `vp dev`、`vp build`、`vp test`、`vp lint`、`vp fmt`、`vp check` に寄せる。
- `vite.config.ts` に lint/fmt/test/run/pack/staged などの Vite+ 設定がある場合は、その設定を尊重する。
- Vite+ 使用時は Oxlint、Oxfmt、Vitest、tsdown 用の個別設定ファイルを新規作成せず、設定を `vite.config.ts` に集約する。
- 画面は最初の viewport から実用画面にする。説明用ランディングではなく、ユーザーが求めたアプリ本体を初期表示にする。
- 開発サーバーの stdout/stderr は必ずプロジェクト root の `web.log` に出す。

## 新規作成

1. `vp` でプロジェクトを作る。

```bash
vp create vite -- --template react-ts
```

2. 依存関係を入れる。

```bash
pnpm install @tanstack/react-router @tanstack/router-devtools lucide-react
```

3. CSS は既存方針がなければ `src/index.css` とコンポーネント CSS で完結させる。必要以上に UI フレームワークを増やさない。

## Vite+ 設定

Vite+ では全 tool 設定を `vite.config.ts` に集約する。

```ts
import { defineConfig } from "vite-plus";

export default defineConfig({
  server: {},
  build: {},
  preview: {},
  test: {},
  lint: {},
  fmt: {},
  run: {},
  pack: {},
  staged: {},
});
```

type-aware linting をしたいので `lint.options.typeAware` と `lint.options.typeCheck` を有効にする。

```ts
export default defineConfig({
  lint: {
    ignorePatterns: ["dist/**"],
    options: {
      typeAware: true,
      typeCheck: true,
    },
  },
  fmt: {
    singleQuote: true,
  },
});
```

## 既存 Vite からの移行

既存 Vite プロジェクトでは、先に状態を確認してから Vite+ に移行する。

```bash
vp migrate
```

移行後は `vite.config.ts`、package scripts、lockfile の差分を確認する。既存のテスト、lint、format、build がある場合は `vp check` または個別の `vp build`、`vp test`、`vp lint` を実行する。

## Vite+ コマンド

- `vp dev`: Vite dev server を HMR 付きで起動する。
- `vp build`: Rolldown 経由で production build を実行する。package.json の custom build script ではなく Vite build を実行する。
- `vp preview`: production build をローカル配信する。
- `vp test`: Vitest を単発実行する。standalone Vitest と違い watch mode ではない。
- `vp test watch`: watch mode で test を実行する。
- `vp test run --coverage`: coverage 付きで test を実行する。
- `vp lint`: Oxlint を実行する。
- `vp fmt`: Oxfmt を実行する。
- `vp check`: format、lint、typecheck を一括実行する。
- `vp check --fix`: format と lint の自動修正を行う。
- `vp run build`: package.json の `build` script を実行する。
- `vp run build -r`: workspace 全体で依存順に script を実行する。
- `vp pack`: tsdown integration で library を build する。

## TanStack Router 優先ルール

以下の優先度で実装する。

- Critical: router 型登録、`from` 指定による型 narrowing、root context 型、`queryOptions` を使った loader 型推論を守る。
- Critical: route tree は階層、layout、index route、pathless layout を意識して整理する。
- High: router default options、loader、loaderDeps、TanStack Query 連携、search params validation、not-found/error handling を設定する。
- Medium: `Link`、active state、`useNavigate`、相対 path、code splitting、preloading を必要に応じて使う。
- Low: root context、`beforeLoad`、dependency injection は認証、API client、feature flag など共有依存がある場合に使う。

## Router 実装

`src/router.tsx` を作り、`createRootRouteWithContext`、`createRoute`、`createRouter` で path ベースに route tree を定義する。

```tsx
import {
  Link,
  Outlet,
  createRootRouteWithContext,
  createRoute,
  createRouter,
} from "@tanstack/react-router";
import { TanStackRouterDevtools } from "@tanstack/router-devtools";

type RouterContext = {
  queryClient?: unknown;
};

function RootLayout() {
  return (
    <>
      <nav>
        <Link to="/" activeOptions={{ exact: true }}>
          Home
        </Link>
        <Link to="/settings">Settings</Link>
      </nav>
      <Outlet />
      <TanStackRouterDevtools />
    </>
  );
}

const rootRoute = createRootRouteWithContext<RouterContext>()({
  component: RootLayout,
  notFoundComponent: NotFoundPage,
  errorComponent: ErrorPage,
});

const indexRoute = createRoute({
  getParentRoute: () => rootRoute,
  path: "/",
  component: HomePage,
});

const settingsRoute = createRoute({
  getParentRoute: () => rootRoute,
  path: "/settings",
  validateSearch: (search) => ({
    tab: typeof search.tab === "string" ? search.tab : "profile",
  }),
  component: SettingsPage,
});

const routeTree = rootRoute.addChildren([indexRoute, settingsRoute]);

export const router = createRouter({
  routeTree,
  defaultPreload: "intent",
  defaultPreloadStaleTime: 30_000,
  scrollRestoration: true,
  context: {},
});

declare module "@tanstack/react-router" {
  interface Register {
    router: typeof router;
  }
}
```

`src/main.tsx` では `RouterProvider` を使う。

```tsx
import { StrictMode } from "react";
import { createRoot } from "react-dom/client";
import { RouterProvider } from "@tanstack/react-router";
import { router } from "./router";
import "./index.css";

createRoot(document.getElementById("root")!).render(
  <StrictMode>
    <RouterProvider router={router} />
  </StrictMode>,
);
```

## Data Loading

- route data は component 内の `useEffect` ではなく、可能な限り route `loader` に寄せる。
- search params や params が loader cache に影響する場合は `loaderDeps` を定義する。
- TanStack Query を使う場合は `queryOptions` と `queryClient.ensureQueryData` を組み合わせる。
- critical data は loader で待ち、非 critical data は deferred loading や component 側取得に分ける。
- loader error は `errorComponent`、not found は `notFoundComponent` で受ける。
- 親子 route の loader は並列に動く前提で、不要な逐次依存を作らない。

## Search Params

- search params は必ず `validateSearch` で検証し、型付き URL state として扱う。
- default 値を設定し、未指定 URL でも画面が壊れないようにする。
- 親 route の search param 型を子 route で継承できる設計にする。
- URL に載せる値は共有、復元、ブックマークが必要な状態に限る。
- 複雑な serializer が必要な場合だけ custom serializer を設定する。

## Navigation

- 通常の画面遷移は `Link` を使う。
- 現在地表示には active state を設定する。
- フォーム送信後やコマンド操作後など、プログラム上の遷移には `useNavigate` を使う。
- 相対 path を使う場合は、どの route からの相対指定かを明確にする。
- Modal URL や背景画面を維持する必要があるときだけ route mask を検討する。

## Code Splitting と Preloading

- 大きなページ、設定画面、管理画面、ゲーム画面など初期表示に不要な route は lazy route に分ける。
- critical な route 定義、loader、validateSearch、error boundary は main route 側に残す。
- auto code splitting が使える環境では有効化を検討する。
- 主要ナビゲーションには `defaultPreload: "intent"` を設定する。
- 手動 preload は、次に開く確度が高い画面に限定する。

## 画面実装

- ルートごとに page component を分ける。小さなアプリでも `src/App.tsx` に集めず、`src/pages/` と `src/components/` を作る。
- 主要な状態、空状態、エラー状態、ローディング状態を必要に応じて実装する。
- アイコンが必要なボタンには `lucide-react` を使う。
- テキストがボタンやカードからはみ出さないよう、固定寸法、折り返し、レスポンシブ制約を設定する。
- 画像やゲーム等の視覚要素が必要なアプリでは、実際のアセット、検索画像、生成画像、Canvas、Three.js などを使い、空の装飾だけで済ませない。

## ログと起動

開発サーバーは project root で起動し、ログを root の `web.log` に出す。

```bash
vp dev --host 0.0.0.0 > web.log 2>&1
```

起動後は `web.log` を確認し、表示 URL、コンパイルエラー、runtime error を把握する。必要なら修正して再起動する。

## よくあるエラー

- `vp build` が期待した script を実行しない: `vp build` は Vite build 用。package.json script を実行する場合は `vp run build` を使う。
- `vp migrate` 後に import が壊れる: `vp install` を実行し、`vp check` で残りの問題を洗い出す。

## 検証

- `vp build` を実行し、型エラーと build error を潰す。
- 変更内容に応じて `vp test`、`vp lint`、`vp fmt`、`vp check` を実行する。
- 開発サーバーを起動して、`web.log` に Vite+ の起動ログが出ていることを確認する。
- TanStack Router の各 path に直接アクセスし、初期表示、search params、not-found、ナビゲーションが動くことを確認する。
- 最終回答では、変更ファイル、起動 URL、実行した検証、未検証事項を簡潔に伝える。
