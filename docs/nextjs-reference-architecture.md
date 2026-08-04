# Next.js リファレンスアーキテクチャガイド

## 0. このドキュメントの目的と使い方

このドキュメントは、業務アプリケーション向けの Next.js Web アプリケーションを開発する際の**参照アーキテクチャ**を定義するものです。

AI コーディングエージェント（Claude Code など）にコード生成・修正を依頼する際は、**必ずこのドキュメントを参照させてください**。目的は以下の 2 点です。

1. 実装ごとにアーキテクチャ判断がブレることを防ぎ、品質を安定させる
2. 「どこに何を書くか」を一意に決め、レビューコストとリファクタリングコストを下げる

このドキュメントに書かれていない判断が必要になった場合、AI エージェントは**推測で実装せず、このドキュメントの思想（層の責務分離、feature 単位の凝集度）に従って最も近い既存パターンを踏襲する**こと。判断が難しい場合は人間に確認する。

### 前提とする技術スタック

| 分野 | 採用技術 |
|---|---|
| フレームワーク | Next.js 16 (App Router) |
| ランタイム | Node.js 20.9 以上（Next.js 16 の必須要件。18 系は非サポート） |
| バンドラ | Turbopack（Next.js 16 でデフォルト・安定版。追加設定は不要） |
| 言語 | TypeScript 5.1 以上 (strict) |
| ORM | Prisma |
| DB | PostgreSQL |
| 認証 | Auth.js (NextAuth v5) |
| 認可 | 自前実装のロールベース制御 (RBAC) |
| バリデーション | Zod |
| スタイリング | Tailwind CSS |
| UI コンポーネント | shadcn/ui（任意） |
| フォーム | React Hook Form + Zod |
| テスト | Vitest（単体・結合）、Playwright（E2E） |
| Lint/Format | ESLint + Prettier |
| コンテナ | Docker（マルチステージビルド） |
| ローカル環境 | Docker Compose（app + PostgreSQL） |
| ログ | pino（構造化ログ、stdout 出力） |
| パッケージ管理 | pnpm |

### Next.js 16 特有の規約（AIエージェントが混同しやすい点）

Next.js 16 では Next.js 15 以前と比べていくつかの破壊的変更があり、学習データが古い AI エージェントは古いパターン（`middleware.ts`、`params` の同期アクセスなど）で実装してしまうことがある。以下は必ず新しい規約に従うこと。

- **`middleware.ts` は使わない**。Next.js 16 では `proxy.ts`（エクスポート関数名も `proxy`）にリネームされている。`middleware.ts` は非推奨警告付きで一時的に動作するが、将来削除される。詳細は 6.3 章。
- **`params` / `searchParams` は必ず `await` する**。Next.js 15 で導入された非同期化が 16 で完全必須化され、同期アクセスは廃止された。`cookies()` / `headers()` / `draftMode()` も同様。
- **キャッシュはデフォルトで完全に動的（キャッシュされない）**。Next.js 16 の Cache Components モデルでは `"use cache"` ディレクティブを明示しない限りキャッシュされない。本ガイドの業務アプリでは基本的にこのデフォルト動作（＝リクエストごとに最新データを取得）を前提とし、キャッシュは必要な箇所にのみ明示的に追加する（詳細は 3.5 章）。
- Turbopack がデフォルトバンドラであり、追加設定は不要（Webpack への切り戻しは非推奨）。

### アーキテクチャの基本方針

- **データ取得・更新の方式**: Server Actions を基本とする。Server Component からの直接データ取得（読み取り）と、Server Actions によるミューテーション（更新）を組み合わせる。外部連携（Webhook、モバイルアプリ、他システム API）が必要になった場合のみ、Route Handlers を個別に追加する。
- **ディレクトリ構成**: 機能単位（feature-based）。`app/` はルーティングと薄い Page/Layout のみを持ち、実際のロジックは `features/<feature>/` に集約する。
- **権限管理**: シンプルなロールベース（RBAC）。`admin` / `editor` / `viewer` のような固定ロールをユーザーに割り当て、ロール×操作の許可マトリクスで判定する。将来的にレコード単位の所有者チェックが必要になった場合も、同じ `can()` 関数のインターフェースを拡張する形で対応できるようにしておく（詳細は 6 章）。

---

## 1. レイヤー構成

各 feature は以下 4 層に分割する。**AI エージェントは新しいコードを書く際、必ずこの層のどこに属するかを最初に判断すること。**

```
┌─────────────────────────────────────────────┐
│ Presentation層                                │
│  - Server Component / Client Component        │
│  - app/ 配下の page.tsx, layout.tsx（薄い）    │
│  - features/<feature>/components/              │
├─────────────────────────────────────────────┤
│ Application層（Use Case）                     │
│  - Server Actions                              │
│  - features/<feature>/actions/                 │
│  - 入力バリデーション、認可チェック、           │
│    トランザクション境界、Domain層の呼び出し      │
├─────────────────────────────────────────────┤
│ Domain層（ビジネスロジック）                    │
│  - features/<feature>/domain/                  │
│  - フレームワーク非依存のロジック・型            │
│  - ビジネスルール、計算、状態遷移の判定           │
├─────────────────────────────────────────────┤
│ Infrastructure層（データアクセス）              │
│  - features/<feature>/repositories/            │
│  - Prisma Client を直接使うのはここだけ          │
└─────────────────────────────────────────────┘
```

### 各層の責務と禁止事項

| 層 | 責務 | やってよいこと | やってはいけないこと |
|---|---|---|---|
| Presentation | 表示、ユーザー操作の受付 | Server Action の呼び出し、UI 状態管理 | Prisma への直接アクセス、ビジネスルールの記述 |
| Application | 1 ユースケースの実行手順の制御 | 入力検証、認可判定、Repository/Domain の呼び出し、トランザクション制御 | UI に関する処理、SQL の直接記述 |
| Domain | ビジネスルールそのもの | 純粋な TypeScript 関数・クラスとしての判定・計算 | Prisma・Next.js・DB への依存 |
| Infrastructure | DB・外部 API との入出力 | Prisma Client の呼び出し、外部 API クライアントの呼び出し | ビジネスルールの判定、認可判定 |

**依存方向は必ず上から下（Presentation → Application → Domain / Infrastructure）。逆方向の依存（例: Domain 層から Server Action を呼ぶ）は禁止。**

---

## 2. ディレクトリ構成

```
.
├── app/                          # ルーティング専用（薄い層）
│   ├── (public)/                 # 未認証でアクセス可能なルート群
│   │   ├── login/
│   │   │   └── page.tsx
│   │   └── layout.tsx
│   ├── (authenticated)/          # 認証必須のルート群
│   │   ├── layout.tsx            # ここでセッション検証
│   │   ├── orders/
│   │   │   ├── page.tsx          # 一覧
│   │   │   ├── [id]/
│   │   │   │   └── page.tsx      # 詳細
│   │   │   └── new/
│   │   │       └── page.tsx      # 新規作成
│   │   └── users/
│   │       └── page.tsx
│   ├── api/
│   │   ├── auth/[...nextauth]/route.ts   # Auth.js のハンドラのみ
│   │   └── health/
│   │       ├── live/route.ts             # liveness（12.3節）
│   │       └── ready/route.ts            # readiness（12.3節）
│   ├── layout.tsx                # RootLayout
│   ├── error.tsx                 # グローバルエラーバウンダリ
│   ├── not-found.tsx
│   └── global-error.tsx
│
├── features/                     # 機能単位（ビジネスロジックの本体）
│   ├── order/
│   │   ├── actions/              # Server Actions（Application層）
│   │   │   ├── create-order.ts
│   │   │   ├── update-order-status.ts
│   │   │   └── delete-order.ts
│   │   ├── domain/                # ビジネスルール（Domain層）
│   │   │   ├── order.ts           # Order エンティティ・型
│   │   │   ├── order-status.ts    # 状態遷移ルール
│   │   │   └── order.test.ts
│   │   ├── repositories/          # データアクセス（Infrastructure層）
│   │   │   └── order-repository.ts
│   │   ├── components/            # このfeature専用のUIコンポーネント
│   │   │   ├── order-list.tsx
│   │   │   ├── order-form.tsx
│   │   │   └── order-status-badge.tsx
│   │   ├── schemas/               # Zodスキーマ（入力検証）
│   │   │   └── order-schema.ts
│   │   └── types.ts               # このfeature内で共有する型
│   │
│   ├── user/
│   │   └── (同様の構成)
│   │
│   └── auth/
│       ├── config/
│       │   └── auth-config.ts     # Auth.js の設定
│       ├── domain/
│       │   └── permissions.ts     # RBAC 許可マトリクス（6章参照）
│       └── lib/
│           └── session.ts         # セッション取得ヘルパー
│
├── shared/                        # feature間で共有するもの
│   ├── components/                # 汎用UIコンポーネント（Button, Dialog等）
│   ├── lib/
│   │   ├── prisma.ts              # PrismaClient シングルトン
│   │   ├── logger.ts              # ロガー初期化
│   │   └── env.ts                 # 環境変数の検証・型付け
│   ├── errors/
│   │   └── app-error.ts           # 共通エラークラス
│   └── utils/
│
├── prisma/
│   ├── schema.prisma
│   ├── migrations/
│   └── seed.ts
│
├── tests/
│   └── e2e/                       # Playwright E2Eテスト
│
├── proxy.ts                       # 認証チェック（軽量な処理のみ。旧middleware.ts）
├── Dockerfile
├── docker-compose.yml
├── docker-compose.override.yml    # ローカル開発用の上書き設定
├── .env.example
└── next.config.ts
```

### 配置ルール（AI エージェント向けチェックリスト）

- 新しい業務機能を追加するときは、必ず `features/<機能名>/` 配下にディレクトリを新規作成する。
- `app/` 配下には **ルーティングに必要な最小限のコード** のみを置く。page.tsx は基本的に「データを取得して features のコンポーネントに渡すだけ」の薄い実装にする。
- 2 つ以上の feature で使う UI コンポーネントは `shared/components/` に置く。1 つの feature でしか使わないものは、その feature の `components/` に置く。判断がつかない場合は feature 側に置き、2 箇所目の利用が発生した時点で `shared/` に移動する（早すぎる共通化を避ける）。
- Prisma Client のインスタンス化は `shared/lib/prisma.ts` の 1 箇所のみで行う。他のファイルで `new PrismaClient()` を書かない。

---

## 3. データフロー（Server Actions 中心）

### 3.1 読み取り（Query）

Server Component から直接 Repository を呼び出す。**読み取りに Server Actions は使わない。**

```tsx
// app/(authenticated)/orders/page.tsx
import { getOrders } from "@/features/order/repositories/order-repository";
import { OrderList } from "@/features/order/components/order-list";
import { requireSession } from "@/features/auth/lib/session";

export default async function OrdersPage() {
  const session = await requireSession();
  const orders = await getOrders({ organizationId: session.user.organizationId });

  return <OrderList orders={orders} />;
}
```

動的ルート（`[id]/page.tsx` など）の `params` は Next.js 16 では**必ず Promise として渡され、同期アクセスはできない**。`await` してから使うこと。

```tsx
// app/(authenticated)/orders/[id]/page.tsx
import { getOrderById } from "@/features/order/repositories/order-repository";
import { notFound } from "next/navigation";

export default async function OrderDetailPage(props: PageProps<"/orders/[id]">) {
  const { id } = await props.params;
  const order = await getOrderById({ id });

  if (!order) {
    notFound();
  }

  return <OrderDetail order={order} />;
}
```

### 3.2 更新（Mutation）: Server Actions

更新系はすべて Server Actions を経由する。1 ファイル 1 ユースケースを原則とする。

```ts
// features/order/actions/create-order.ts
"use server";

import { revalidatePath } from "next/cache";
import { requireSession } from "@/features/auth/lib/session";
import { can } from "@/features/auth/domain/permissions";
import { createOrderSchema } from "@/features/order/schemas/order-schema";
import { createOrder as createOrderInDomain } from "@/features/order/domain/order";
import { insertOrder } from "@/features/order/repositories/order-repository";
import { AppError } from "@/shared/errors/app-error";

export async function createOrder(formData: FormData) {
  // 1. 認証
  const session = await requireSession();

  // 2. 認可
  if (!can(session.user, "order:create")) {
    throw new AppError("FORBIDDEN", "この操作を行う権限がありません");
  }

  // 3. 入力検証
  const parsed = createOrderSchema.safeParse({
    title: formData.get("title"),
    amount: formData.get("amount"),
  });
  if (!parsed.success) {
    return { success: false, errors: parsed.error.flatten().fieldErrors };
  }

  // 4. ドメインロジックの適用
  const order = createOrderInDomain({
    ...parsed.data,
    organizationId: session.user.organizationId,
    createdBy: session.user.id,
  });

  // 5. 永続化
  await insertOrder(order);

  // 6. キャッシュ再検証
  revalidatePath("/orders");

  return { success: true };
}
```

### 3.3 Server Action の共通ルール

- **すべての Server Action の先頭で「認証 → 認可 → 入力検証 → ドメイン処理 → 永続化 → 再検証」の順に処理する。** この順序を変えない。
- 戻り値は `{ success: boolean, errors?: ..., data?: ... }` の形に統一する（例外は投げるが UI 表示用のバリデーションエラーは戻り値で返す）。
- `useFormState` / `useActionState`（React 19 / Next.js 16 で標準）と組み合わせて、クライアント側でのフィードバック表示を行う。
- 1 つの Server Action ファイルには 1 つの公開関数のみを置く（`create-order.ts` には `createOrder` のみ）。

### 3.4 外部連携が必要な場合（Route Handlers）

外部システムからの Webhook 受信、モバイルアプリ向け API など、Server Actions で表現できない用途に限り `app/api/<domain>/route.ts` を追加する。この場合も、実処理は `features/<feature>/actions/` または専用の `features/<feature>/handlers/` に委譲し、route.ts 自体は薄く保つ。

**CSRF 対策における Server Actions と Route Handlers の違いに注意する。** Next.js の Server Actions は、リクエストの `Origin` ヘッダーと `Host`（または `X-Forwarded-Host`）ヘッダーを比較する CSRF 対策が**標準で組み込まれている**（POST メソッドのみ許可、かつオリジン不一致は拒否）。そのため Server Actions に対して独自の CSRF トークンを実装する必要は基本的にない。**一方、Route Handlers（`app/api/`）にはこの保護が自動で付かない。** Webhook 以外の用途（Cookie ベースのセッションを使うブラウザからの呼び出し）で Route Handler を追加する場合は、Origin ヘッダーの検証を個別に実装すること。Webhook 受信のように署名検証（HMAC 等）で送信元を確認する用途では、その署名検証が CSRF 対策を兼ねる。

### 3.5 キャッシュ方針（Cache Components）

Next.js 16 の Cache Components モデルでは、キャッシュは**完全にオプトイン**である。`"use cache"` ディレクティブを付けない限り、ページ・レイアウト・関数はすべてリクエスト時に動的実行される。

本ガイドの業務アプリケーション（CRUD中心、権限によって表示が変わる、常に最新データを見せたい）にはこのデフォルト動作がそのまま適合するため、**基本方針として `"use cache"` は使わない**。以下のように明確に効果が見込める場合のみ、個別に検討する。

- 変更頻度が極めて低く、全ユーザー共通のマスタデータ（例: 都道府県一覧）を表示する関数
- 同一ページ内で頻繁に再訪される、計算コストの高い集計処理

```ts
// 例: マスタデータなど、明示的にキャッシュしたい場合のみ使用
"use cache";

export async function getPrefectures() {
  return prisma.prefecture.findMany();
}
```

ミューテーション直後に「自分が書いた変更をすぐに画面へ反映したい」場合は、`revalidatePath` に加えて Next.js 16 で追加された `updateTag()`（Server Actions 専用、read-your-writes を保証）も選択肢になる。ただし本ガイドでは `"use cache"` を基本的に使わないため、通常は 3.2 節の `revalidatePath` で十分であり、`updateTag()` は `"use cache"` を採用した領域と組み合わせて使う。

### 3.6 トランザクション境界

複数テーブルを更新するユースケースや、読み取り→判定→書き込みの間に他プロセスの介入を許したくないユースケースでは、**トランザクション境界は Server Action（Application層）が持つ**。Repository 関数は Prisma の `tx`（トランザクションクライアント）を受け取れるようにし、通常の `prisma` とインターフェースを揃える。

```ts
// features/order/repositories/order-repository.ts
import type { Prisma } from "@prisma/client";
import { prisma } from "@/shared/lib/prisma";

type DbClient = typeof prisma | Prisma.TransactionClient;

export async function insertOrder(order: NewOrder, db: DbClient = prisma) {
  return db.order.create({ data: order });
}

export async function insertOrderHistory(entry: OrderHistoryEntry, db: DbClient = prisma) {
  return db.orderHistory.create({ data: entry });
}
```

```ts
// features/order/actions/approve-order.ts
"use server";

import { prisma } from "@/shared/lib/prisma";
import { insertOrderHistory, updateOrderStatusById } from "@/features/order/repositories/order-repository";

export async function approveOrder(orderId: string) {
  // ...認証・認可・入力検証は省略（3.2節の順序に従う）

  await prisma.$transaction(async (tx) => {
    await updateOrderStatusById(orderId, "approved", tx);
    await insertOrderHistory({ orderId, action: "approved" }, tx);
  });

  revalidatePath(`/orders/${orderId}`);
}
```

- 1 つの Server Action 内で複数の Repository 呼び出しがある場合、それらが「1 つの業務トランザクション」を構成するなら `$transaction` でまとめる。単純な 1 テーブルへの単発更新であれば `$transaction` は不要。
- `$transaction` の中で外部 API 呼び出し（メール送信など）を行わない。トランザクションが長時間ロックを保持する、または外部呼び出し失敗時にロールバックしたくない場合があるため、外部呼び出しはトランザクション確定後に行う。

### 3.7 同時実行制御（楽観的ロック）

複数ユーザーが同じレコードを同時に編集する業務アプリでは、後勝ち（Last Write Wins）による意図しないデータ上書きを防ぐ必要がある。本ガイドでは **楽観的ロック（`version` カラム）を標準パターンとする**。

```prisma
// prisma/schema.prisma
model Order {
  // ...既存カラム
  version Int @default(1)
}
```

```ts
// features/order/repositories/order-repository.ts
export async function updateOrderWithVersion(
  params: { id: string; expectedVersion: number; data: Partial<Order> },
): Promise<Order> {
  const result = await prisma.order.updateMany({
    where: { id: params.id, version: params.expectedVersion },
    data: { ...params.data, version: { increment: 1 } },
  });

  if (result.count === 0) {
    throw new AppError("CONFLICT", "他のユーザーによってこのデータは更新されています。最新の内容を確認してください。");
  }

  return getOrderById({ id: params.id }) as Promise<Order>;
}
```

- 更新系フォームには非表示フィールドとして `version` を持たせ、Server Action の入力に含める。
- `CONFLICT` エラーを受け取った UI 側は、単にエラー表示するだけでなく「最新の内容を再取得して差分を提示する」導線を用意することが望ましい（詳細な UX は要件次第のため、実装時に検討する）。
- すべてのテーブルに `version` を機械的に追加する必要はない。**同時編集が起こりうる業務エンティティ（注文、申請、承認対象など）にのみ**適用する。マスタデータや追記専用のログテーブルには不要。

---

## 4. ドメイン層の設計方針

- Domain 層は Next.js・Prisma・Auth.js から独立した純粋な TypeScript として書く。テストは Vitest でフレームワークに依存せず実行できる状態にする。
- エンティティの状態遷移（例: 注文のステータス変更）はドメイン層に判定ロジックを持たせ、Server Action からはその関数を呼ぶだけにする。

```ts
// features/order/domain/order-status.ts
export type OrderStatus = "draft" | "submitted" | "approved" | "rejected";

const ALLOWED_TRANSITIONS: Record<OrderStatus, OrderStatus[]> = {
  draft: ["submitted"],
  submitted: ["approved", "rejected"],
  approved: [],
  rejected: ["draft"],
};

export function canTransition(from: OrderStatus, to: OrderStatus): boolean {
  return ALLOWED_TRANSITIONS[from]?.includes(to) ?? false;
}
```

- Prisma が生成する型（`@prisma/client` の型）を Domain 層に直接漏らさない。Domain 層専用の型を `features/<feature>/domain/order.ts` に定義し、Repository 層で Prisma の型からマッピングする。これにより、将来 ORM を変更してもドメイン層への影響を局所化できる。

---

## 5. Infrastructure層（Prisma）の設計方針

### 5.1 Prisma Client のシングルトン化

```ts
// shared/lib/prisma.ts
import { PrismaClient } from "@prisma/client";

const globalForPrisma = globalThis as unknown as { prisma?: PrismaClient };

export const prisma =
  globalForPrisma.prisma ??
  new PrismaClient({
    log: process.env.NODE_ENV === "development" ? ["query", "warn", "error"] : ["error"],
  });

if (process.env.NODE_ENV !== "production") {
  globalForPrisma.prisma = prisma;
}
```

### 5.2 スキーマ設計の基本ルール

- すべての主要テーブルに `id`（UUID または cuid）、`createdAt`、`updatedAt` を持たせる。
- マルチテナント（組織単位でデータを分ける）を想定する場合、業務テーブルには `organizationId` を必須カラムとして持たせ、Repository 層のすべての Query に `organizationId` の絞り込みを強制する。
- 論理削除が必要な場合は `deletedAt: DateTime?` を用い、物理削除は原則行わない。
- 監査が必要な操作（作成・更新・承認など）は `createdBy` / `updatedBy` を記録する。

```prisma
// prisma/schema.prisma（例）
model Order {
  id             String       @id @default(cuid())
  organizationId String
  title          String
  amount         Int
  status         OrderStatus  @default(draft)
  createdBy      String
  createdAt      DateTime     @default(now())
  updatedAt      DateTime     @updatedAt
  deletedAt      DateTime?

  organization   Organization @relation(fields: [organizationId], references: [id])

  @@index([organizationId])
}

enum OrderStatus {
  draft
  submitted
  approved
  rejected
}
```

### 5.3 Repository 層の書き方

- Repository 関数は「1 つの明確な目的を持つクエリ」を返す小さな関数として書く。汎用的すぎる `findMany(filter: any)` のような関数は作らない。
- `organizationId` によるスコープ絞り込みは Repository 層で必ず行い、呼び出し側（Server Action）に絞り込み漏れの責任を持たせない。

```ts
// features/order/repositories/order-repository.ts
import { prisma } from "@/shared/lib/prisma";
import type { Order } from "@/features/order/domain/order";

export async function getOrders(params: { organizationId: string }): Promise<Order[]> {
  const rows = await prisma.order.findMany({
    where: { organizationId: params.organizationId, deletedAt: null },
    orderBy: { createdAt: "desc" },
  });
  return rows.map(toDomainOrder);
}

function toDomainOrder(row: /* Prisma.OrderGetPayload<...> */ any): Order {
  return {
    id: row.id,
    title: row.title,
    amount: row.amount,
    status: row.status,
  };
}
```

### 5.4 マイグレーション運用

- 開発時: `prisma migrate dev` でマイグレーションファイルを生成し、必ずコミットする。
- 本番デプロイ時: コンテナ起動前（entrypoint）で `prisma migrate deploy` を実行する。`db push` は本番で使用しない。
- シードデータは `prisma/seed.ts` に集約し、`package.json` の `prisma.seed` に登録する。

> **【検討事項】マイグレーション実行タイミングの方式**
> ローリングデプロイで複数インスタンスが順次入れ替わる場合、新旧バージョンのコードが一時的に混在する。`migrate deploy` はコンテナ起動時に実行しても Prisma の advisory lock により多重実行は安全だが、「新スキーマを前提とした新コードが、旧スキーマのままの旧インスタンスと同時に稼働する時間帯」が生じうる。破壊的なスキーマ変更（カラム削除・型変更等）を行う際は、以下のいずれかを採用する。要件（許容ダウンタイム、デプロイ頻度）に応じて選ぶ。
> - **方式A: entrypoint実行（本ガイドのデフォルト）** — 実装が単純。後方互換性のあるマイグレーション（カラム追加等）が中心の場合に適する。
> - **方式B: CI/CDパイプラインの専用ステップで1回だけ実行** — コンテナ起動とマイグレーションを分離できるため、破壊的変更が多いプロジェクトや、デプロイ順序を厳密に制御したい場合に適する。
> - **方式C: Expand-and-Contract（後方互換の2段階リリース）** — 破壊的変更を「追加→両対応→旧カラム削除」の複数リリースに分割する。ダウンタイムを避けたい要件が強い場合に検討する。

### 5.5 コネクションプール管理

> **【検討事項】このアーキテクチャ特有の重要な論点**
> 12 章の前提により、アプリケーションは複数コンテナインスタンスに水平スケールする。各インスタンスは Prisma Client を 1 つ保持し、Prisma は既定でインスタンスあたり `num_physical_cpus * 2 + 1` 程度のコネクションプールを確保する。**インスタンス数 × プールサイズが PostgreSQL の `max_connections` を超えると、スケールアウトした瞬間に新規接続が拒否される**。この問題は、要件（想定インスタンス数、DBの max_connections、利用する PostgreSQL の提供形態）に応じて以下のいずれかで対処する。
>
> - **方式A: PgBouncer 等のコネクションプーラーを DB の前段に置く**（トランザクションプーリングモード）。アプリ側は少数のコネクションで済み、インスタンス数のスケールに強い。マネージド PostgreSQL（RDS Proxy、Cloud SQL の場合は別途構成）を使う場合はこの方式が一般的。
> - **方式B: Prisma の `connection_limit` を明示的に指定し、`(max_connections ÷ 想定最大インスタンス数)` を上限に設定する**。追加コンポーネントが不要でシンプルだが、インスタンス数の上限を運用で守る必要がある。小規模〜中規模で最大インスタンス数が事前に決まっている場合に向く。
> - **方式C: Prisma Accelerate 等のマネージドコネクションプーリングサービスを利用する。** 運用負荷を下げたい場合の選択肢。
>
> 本ガイドは特定の方式を強制しない。**採用する方式と `max_connections` の設定値は、想定最大インスタンス数が決まった時点で確定させ、このドキュメントに追記すること。**

### 5.6 ページネーションと N+1 対策

- 一覧取得系の Repository 関数は、件数の上限がないクエリを書かない。5.3 節の `getOrders` のような関数には必ずページネーションを実装する。

```ts
// features/order/repositories/order-repository.ts
export async function getOrders(params: {
  organizationId: string;
  cursor?: string;
  limit?: number;
}): Promise<{ items: Order[]; nextCursor: string | null }> {
  const limit = params.limit ?? 20;

  const rows = await prisma.order.findMany({
    where: { organizationId: params.organizationId, deletedAt: null },
    orderBy: { createdAt: "desc" },
    take: limit + 1,
    ...(params.cursor ? { cursor: { id: params.cursor }, skip: 1 } : {}),
  });

  const hasMore = rows.length > limit;
  const items = hasMore ? rows.slice(0, limit) : rows;

  return {
    items: items.map(toDomainOrder),
    nextCursor: hasMore ? items[items.length - 1].id : null,
  };
}
```

> **【検討事項】ページネーション方式**
> 上記はカーソルベースの例。件数が数千件程度までで「ページ番号」によるUIが必要な場合はオフセットベース（`skip`/`take`）でも構わない。データ量とUI要件に応じて選択する。カーソルベースは大量データでも性能劣化しにくいが、「n ページ目に飛ぶ」UIには向かない。

- 関連データを N+1 クエリで取得しない。Prisma の `include` / `select` を使って 1 クエリで取得する。

```ts
// 悪い例: N+1が発生する
const orders = await prisma.order.findMany();
for (const order of orders) {
  const org = await prisma.organization.findUnique({ where: { id: order.organizationId } });
}

// 良い例: includeで1クエリにまとめる
const orders = await prisma.order.findMany({
  include: { organization: true },
});
```

- 絞り込み・並び替えに使うカラムには複合 index を検討する（例: `@@index([organizationId, status, createdAt])`）。5.2 節のスキーマは単一カラム index の例のみを示しているが、実際のクエリパターンが固まった時点で複合 index を追加する。

---

## 6. 認証（Auth.js）

### 6.1 基本方針

- セッション戦略は **JWT（stateless）** を基本とする。理由: コンテナが複数台にスケールしても DB ルックアップやセッションストアの共有が不要になるため。ただし、即時のセッション無効化（強制ログアウト）が要件として必要な場合は database 戦略を検討する（要相談事項として明示する）。
- 認証プロバイダーは Credentials（メール＋パスワード）を最小構成とし、要件に応じて OAuth（Google 等）を追加する。パスワードは必ず bcrypt/argon2 でハッシュ化して保存する。

```ts
// features/auth/config/auth-config.ts
import Credentials from "next-auth/providers/credentials";
import type { NextAuthConfig } from "next-auth";
import { verifyCredentials } from "@/features/auth/domain/verify-credentials";

export const authConfig: NextAuthConfig = {
  session: { strategy: "jwt" },
  providers: [
    Credentials({
      credentials: { email: {}, password: {} },
      authorize: async (credentials) => {
        return verifyCredentials(credentials);
      },
    }),
  ],
  callbacks: {
    jwt({ token, user }) {
      if (user) {
        token.role = user.role;
        token.organizationId = user.organizationId;
      }
      return token;
    },
    session({ session, token }) {
      session.user.role = token.role as string;
      session.user.organizationId = token.organizationId as string;
      return session;
    },
  },
  pages: { signIn: "/login" },
};
```

### 6.2 セッション取得ヘルパー

Server Component / Server Action からのセッション取得は、以下のヘルパーに統一する（各所で `auth()` を直接呼ばない）。

```ts
// features/auth/lib/session.ts
import { auth } from "@/features/auth/config/auth-config";
import { AppError } from "@/shared/errors/app-error";

export async function requireSession() {
  const session = await auth();
  if (!session?.user) {
    throw new AppError("UNAUTHENTICATED", "ログインが必要です");
  }
  return session;
}
```

### 6.3 proxy.ts の役割

Next.js 16 では、旧 `middleware.ts` は **`proxy.ts`** にリネームされている（エクスポートする関数名も `proxy`）。`middleware.ts` は非推奨警告付きで一時的に動作するが、新規実装では必ず `proxy.ts` を使うこと。あわせて以下の点に注意する。

- `proxy.ts` は**デフォルトで Node.js ランタイム**で動作する（旧 middleware は Edge Runtime 固定だった）。ランタイムを明示的に指定する `export const runtime = ...` はエラーになるため書かない。
- 役割は変わらず、**認証の有無のみ**を軽量にチェックし、未認証ユーザーを `/login` にリダイレクトする。認可（ロールによる操作許可）判定はここで行わず、各 Server Action / Page 側の `requireSession` + `can()` に委ねる。
- **`proxy.ts` でのチェックはあくまで最初の防衛線であり、唯一のゲートにしない。** プロキシ層の設定ミスやマッチャーの記述漏れで認証チェックが素通りするリスクがあるため、Server Action / Server Component 側の `requireSession()` を必ず併用する（本ガイドは 3 章・6.2 節の設計により、この二重チェックを既定にしている）。

```ts
// proxy.ts（プロジェクトルート。旧 middleware.ts から名称変更）
import { auth } from "@/features/auth/config/auth-config";

export const proxy = auth((req) => {
  const isAuthenticated = !!req.auth;
  const isPublicRoute = req.nextUrl.pathname.startsWith("/login");

  if (!isAuthenticated && !isPublicRoute) {
    return Response.redirect(new URL("/login", req.url));
  }
});

export const config = {
  matcher: ["/((?!api/auth|api/health|_next/static|_next/image|favicon.ico).*)"],
};
```

---

## 7. 認可（RBAC）

### 7.1 許可マトリクス方式

ロールと操作の組み合わせを 1 箇所（許可マトリクス）で定義し、判定はすべて `can()` 関数経由で行う。**Server Action・Page・UI 表示制御のいずれからも同じ `can()` を呼ぶことで、判定ロジックの重複を防ぐ。**

```ts
// features/auth/domain/permissions.ts
export type Role = "admin" | "editor" | "viewer";

export type Permission =
  | "order:create"
  | "order:update"
  | "order:delete"
  | "order:approve"
  | "user:invite"
  | "user:manage-role";

const ROLE_PERMISSIONS: Record<Role, Permission[]> = {
  admin: [
    "order:create", "order:update", "order:delete", "order:approve",
    "user:invite", "user:manage-role",
  ],
  editor: ["order:create", "order:update"],
  viewer: [],
};

export function can(user: { role: Role }, permission: Permission): boolean {
  return ROLE_PERMISSIONS[user.role]?.includes(permission) ?? false;
}
```

### 7.2 UI 側での利用

権限に応じたボタンの表示・非表示は、あくまで UX 上の配慮であり**セキュリティ境界にはならない**。UI で隠していても、Server Action 側の `can()` チェックは必ず行うこと（二重実装ではなく、UI は「便宜的なショートカット」、Server Action が「本当のゲート」という位置づけ）。

```tsx
{can(session.user, "order:approve") && <ApproveButton orderId={order.id} />}
```

### 7.3 将来のレコード単位権限への拡張

現時点ではロールベースのみを実装するが、将来的にレコード単位の所有者チェック（例: 「自分が作成した注文のみ編集可」）が必要になった場合は、`can()` の第 3 引数にリソースを渡せるように拡張する。

```ts
// 将来の拡張イメージ（現時点では未実装）
export function can(user: User, permission: Permission, resource?: { ownerId: string }): boolean {
  // ロールチェックに加えて、resource.ownerId === user.id 等の判定を追加できる
}
```

AI エージェントは、現時点でこの拡張を先取りして実装しない（過剰設計を避ける）。要件が発生した時点で拡張する。

---

## 8. バリデーション

- すべての外部入力（フォーム、Server Action の引数、Route Handler のリクエストボディ）は Zod スキーマで検証する。
- スキーマは `features/<feature>/schemas/` に置き、Server Action とクライアント側のフォーム（React Hook Form の `zodResolver`）の両方から同じスキーマを import して使う（検証ロジックの重複を避ける）。

```ts
// features/order/schemas/order-schema.ts
import { z } from "zod";

export const createOrderSchema = z.object({
  title: z.string().min(1).max(200),
  amount: z.coerce.number().int().positive(),
});

export type CreateOrderInput = z.infer<typeof createOrderSchema>;
```

---

## 9. エラーハンドリング

### 9.1 共通エラークラス

```ts
// shared/errors/app-error.ts
export type AppErrorCode =
  | "UNAUTHENTICATED"
  | "FORBIDDEN"
  | "NOT_FOUND"
  | "VALIDATION_ERROR"
  | "CONFLICT"
  | "INTERNAL_ERROR";

export class AppError extends Error {
  constructor(public code: AppErrorCode, message: string) {
    super(message);
    this.name = "AppError";
  }
}
```

### 9.2 方針

- Domain / Infrastructure 層で発生したエラーは `AppError` に変換して上位に投げる。
- Server Action 内で発生した予期しない例外は catch し、ログに出力してから汎用的なエラーメッセージを UI に返す（内部エラーの詳細を UI に漏らさない）。
- `app/error.tsx`（Route Segment 単位）と `app/global-error.tsx`（アプリ全体）で予期しない例外の最終防衛線を用意する。
- 404 系は Repository 層で `null` を返し、Page 側で `notFound()` を呼ぶ。例外を投げて表現しない。

---

## 10. ログ

- 構造化ログ（JSON）を stdout に出力し、CaaS 側のログ収集機構（CloudWatch Logs、Cloud Logging 等）に committed する前提とする。ファイルへの書き込みは行わない。
- ログには最低限 `timestamp`, `level`, `message`, `requestId`（可能であれば）、`userId`（認証済みの場合）を含める。
- パスワード、トークン、個人情報を平文でログに出力しない。

---

## 11. テスト戦略

| 種別 | 対象 | ツール | 配置 |
|---|---|---|---|
| 単体テスト | Domain層のロジック | Vitest | `features/<feature>/domain/*.test.ts` |
| 結合テスト | Server Actions（DBアクセスを含む） | Vitest + テスト用PostgreSQL | `features/<feature>/actions/*.test.ts` |
| E2Eテスト | 主要なユーザーシナリオ | Playwright | `tests/e2e/` |

- Domain 層のテストは DB・ネットワークに依存しないため、最も手厚くカバレッジを取る。
- 結合テストは Docker Compose で立てたテスト用 PostgreSQL に対して実行し、各テスト後にトランザクションロールバックまたはテーブル truncate でクリーンな状態に戻す。
- 認可ロジック（`can()`）は権限マトリクスの全パターンを単体テストで網羅する。

---

## 12. コンテナ化

### 12.1 Dockerfile（マルチステージビルド）

```dockerfile
# ---- deps ----
FROM node:22-slim AS deps
WORKDIR /app
RUN corepack enable
COPY package.json pnpm-lock.yaml ./
RUN pnpm install --frozen-lockfile

# ---- build ----
FROM node:22-slim AS build
WORKDIR /app
RUN corepack enable
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN pnpm prisma generate
RUN pnpm build

# ---- runtime ----
FROM node:22-slim AS runtime
WORKDIR /app
ENV NODE_ENV=production
RUN corepack enable
COPY --from=build /app/public ./public
COPY --from=build /app/.next/standalone ./
COPY --from=build /app/.next/static ./.next/static
COPY --from=build /app/prisma ./prisma
COPY --from=build /app/node_modules/.prisma ./node_modules/.prisma

EXPOSE 3000
CMD ["sh", "-c", "node node_modules/prisma/build/index.js migrate deploy && node server.js"]
```

- `next.config.ts` で `output: "standalone"` を必ず設定し、実行時イメージを最小化する。
- マイグレーション（`migrate deploy`）はコンテナ起動時の entrypoint で実行する。複数インスタンス同時起動時の competing migration を避けたい場合は、CI/CD パイプライン側で 1 回だけ実行するステップに分離してもよい（構成次第で要相談）。

### 12.2 docker-compose.yml（ローカル開発用）

```yaml
services:
  app:
    build:
      context: .
      target: build
    command: pnpm dev
    ports:
      - "3000:3000"
    environment:
      DATABASE_URL: postgresql://app:app@db:5432/app_dev
      AUTH_SECRET: ${AUTH_SECRET}
    volumes:
      - .:/app
      - /app/node_modules
    depends_on:
      db:
        condition: service_healthy

  db:
    image: postgres:17-alpine
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: app
      POSTGRES_DB: app_dev
    ports:
      - "5432:5432"
    volumes:
      - db-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app"]
      interval: 5s
      timeout: 5s
      retries: 5

volumes:
  db-data:
```

### 12.3 CaaS へのデプロイに関する前提

このアーキテクチャは特定の CaaS（Cloud Run / ECS Fargate / Azure Container Apps 等）に依存しない。共通の前提は以下。

- アプリケーションはステートレス（セッションを JWT で持つため、ローカルディスクや直近のメモリ状態にセッション情報を依存しない）とし、複数インスタンスへの水平スケールに対応する。
- 環境変数はコンテナ実行環境のシークレット管理機能（Secrets Manager 等）から注入する。`.env` ファイルをイメージに含めない。

**ヘルスチェックは liveness（生存確認）と readiness（受信可否確認）を分ける。** 1 つのエンドポイントで DB 接続まで確認すると、DB が一時的に遅延しただけでコンテナ自体が再起動を繰り返す「crash loop」を招きうる。

```ts
// app/api/health/live/route.ts
// liveness: プロセスが生きているかのみを返す。DBには問い合わせない。
export async function GET() {
  return Response.json({ status: "ok" });
}
```

```ts
// app/api/health/ready/route.ts
// readiness: DB接続を含め、リクエストを受けてよい状態かを返す。
import { prisma } from "@/shared/lib/prisma";

export async function GET() {
  try {
    await prisma.$queryRaw`SELECT 1`;
    return Response.json({ status: "ok" });
  } catch {
    return Response.json({ status: "error" }, { status: 503 });
  }
}
```

> **【検討事項】liveness/readiness の設定方法は CaaS ごとに異なる。** Cloud Run はリビジョンレベルのヘルスチェック機能、ECS/Fargate はタスク定義の `healthCheck` とロードバランサのターゲットグループヘルスチェック、Kubernetes 系（GKE/EKS等）は `livenessProbe`/`readinessProbe` を使う。デプロイ先が決まった時点で、上記 2 エンドポイントをどちらの仕組みに割り当てるかを確認すること。

**グレースフルシャットダウン**: CaaS はスケールインやデプロイ時にコンテナへ `SIGTERM` を送る。何もしないと処理中のリクエストが強制終了される可能性があるため、`server.js`（standalone 出力）が `SIGTERM` を受けたら新規リクエストの受付を止めつつ、処理中のリクエストの完了と Prisma の切断を待つ。

```ts
// 例: instrumentation.ts（Next.js の起動時フックに登録する）
export function register() {
  process.on("SIGTERM", async () => {
    const { prisma } = await import("@/shared/lib/prisma");
    await prisma.$disconnect();
    process.exit(0);
  });
}
```

- CaaS 側の「猶予期間（termination grace period）」は、想定される最長処理時間（大きめのトランザクションや長いServer Actionの実行時間）より長く設定する。

---

## 13. 環境変数管理

- すべての環境変数は `shared/lib/env.ts` で Zod により検証し、型付けされた形でアプリ全体から参照する。**`process.env.XXX` を各所で直接参照しない。**

```ts
// shared/lib/env.ts
import { z } from "zod";

const envSchema = z.object({
  DATABASE_URL: z.string().url(),
  AUTH_SECRET: z.string().min(32),
  NODE_ENV: z.enum(["development", "production", "test"]),
});

export const env = envSchema.parse(process.env);
```

- `.env.example` に必要な変数名とダミー値を列挙し、リポジトリにコミットする。実際の `.env` はコミットしない。

---

## 14. コーディング規約（AIエージェント向け必須ルール）

1. **命名規則**: ファイル名は kebab-case（`create-order.ts`）、関数・変数は camelCase、型・クラスは PascalCase、Server Action の公開関数名は動詞から始める（`createOrder`, `updateOrderStatus`）。
2. **any の禁止**: `any` 型を使わない。Prisma の生成型を Domain 層に漏らす場合を除き、明示的な型付けを行う。
3. **1 ファイル 1 責務**: Server Action ファイルは 1 ユースケース 1 ファイル。巨大な `actions.ts` にすべてをまとめない。
4. **直接 Prisma 呼び出しの禁止**: `features/<feature>/repositories/` 以外から `prisma.*` を呼び出さない。
5. **認可チェックの位置**: 認可チェック（`can()`）は必ず Server Action の入力検証より前、永続化処理より前に行う。
6. **バリデーションの重複禁止**: 同じ入力チェックを複数箇所に書かない。Zod スキーマを共有する。
7. **新規 feature 追加時**: 2 章のディレクトリ構成に従い、`actions/ domain/ repositories/ components/ schemas/` を作成する。既存 feature（例: `order`）の構成をテンプレートとして踏襲する。
8. **不明な設計判断が必要な場合**: このドキュメントに明記がない場合、独自判断で新しいパターンを作らず、既存の類似 feature の実装パターンを踏襲する。それでも判断がつかない場合は人間に確認する。
9. **一覧取得系の Repository 関数には必ずページネーションを実装する**（5.6節）。上限のない `findMany` を書かない。
10. **同時編集が想定される業務エンティティ（注文・申請・承認対象など）には `version` カラムによる楽観的ロックを実装する**（3.7節）。マスタデータや追記専用ログには不要。
11. **複数テーブルを更新するユースケースは `$transaction` でトランザクション境界を明示する**（3.6節）。トランザクション内で外部API呼び出しを行わない。

---

## 15. セキュリティ共通対策

### 15.1 レート制限

ログイン試行のブルートフォース対策、および Server Actions / Route Handlers への過剰リクエスト対策として、レート制限を導入する。**業務アプリの認証エンドポイントに対するレート制限は必須とし、その他のミューテーション系エンドポイントへの適用範囲は要件に応じて決める。**

> **【検討事項】レート制限の実装場所**
> - **方式A: アプリケーションコード内（Redis 等を使ったカウンタ）** — Upstash Redis + `@upstash/ratelimit` のようなライブラリを Server Action / proxy.ts の先頭で呼ぶ。CaaS を問わず同じ実装で動くため、特定クラウドに依存したくない場合に向く。
> - **方式B: CaaS 側のインフラ機能（WAF、API Gateway のレート制限、ロードバランサのスロットリング）** — アプリケーションコードを汚さずに済むが、デプロイ先 CaaS が確定してから設定できる。
> - **方式C: 両者の併用（インフラ側で粗い制限、アプリ側でログイン試行等の細かい制限）** — 多重防御を重視する場合。
>
> 最低限、ログインの Server Action（`authorize` 呼び出し）には方式A または C のようなアプリケーション側の対策を入れることを推奨する。理由: 認証失敗回数はビジネスロジック（アカウントロック等）と結びつくことが多く、インフラ側だけでは表現しづらい。

### 15.2 セキュリティヘッダー

`next.config.ts` の `headers()` で、以下のヘッダーをすべてのレスポンスに付与する。

```ts
// next.config.ts
const securityHeaders = [
  { key: "X-Content-Type-Options", value: "nosniff" },
  { key: "X-Frame-Options", value: "DENY" },
  { key: "Referrer-Policy", value: "strict-origin-when-cross-origin" },
  { key: "Strict-Transport-Security", value: "max-age=63072000; includeSubDomains; preload" },
];

export default {
  async headers() {
    return [{ source: "/(.*)", headers: securityHeaders }];
  },
};
```

> **【検討事項】Content-Security-Policy（CSP）** は、利用する外部スクリプト・フォント・画像ソースの構成に強く依存するため、本ガイドでは具体的な値を固定しない。UI ライブラリ（shadcn/ui 等）やアナリティクスツールの採用が決まった時点で、それらが要求するオリジンを許可リストに含めた CSP を設計する。

### 15.3 CSRF・依存パッケージ脆弱性・HTTPS前提

- CSRF については 3.4 節を参照（Server Actions は標準保護あり、Route Handlers は個別対応が必要）。
- CI パイプラインに `pnpm audit` または Dependabot / Renovate による依存パッケージの脆弱性検知を組み込む。**重大度 High/Critical の検知を CI 失敗条件にするかは、リリース頻度と運用体制に応じて決める（検討事項）。**
- HTTPS 終端は CaaS 側のロードバランサ／Ingress で行う前提とし、アプリケーション側では Cookie に `Secure` 属性・`SameSite=Lax`（Auth.js のデフォルト）を設定する。アプリケーションコード内で HTTP を直接扱わない。

---

## 16. 可用性・耐障害性（全体方針）

12.3 節のコンテナ単位の対策に加え、以下をシステム全体の前提として定める。

- **DB 自体の可用性**は、採用する PostgreSQL の提供形態（マネージドサービスのマルチAZ構成か、シングルインスタンスか）に依存する。**このガイドはアプリケーション層の設計であり、DB のフェイルオーバー構成自体は範囲外とするが、「DB 接続断からの自動再接続」はアプリ側の責務である。** Prisma はデフォルトで一時的な接続断からの再試行を行うが、長時間の DB 障害時にアプリケーションがどう振る舞うか（readiness を false にしてトラフィックを止める、502/503 を返す等）は 12.3 節の readiness チェックに委ねる。
- 外部 API 連携がある場合、タイムアウトとリトライ（指数バックオフ）を個別の Repository/クライアント実装に持たせる。無制限リトライは行わない。

> **【検討事項】許容ダウンタイム・目標復旧時間（RTO）** は要件定義側で決まる事項であり、本ガイドでは既定値を設けない。デプロイ戦略（ローリング/Blue-Green/カナリア）や DB フェイルオーバー方式は、RTO の目標値が決まった時点で選定する。

---

## 17. 運用監視（Observability）

10 章のログ方針に加えて、以下を導入する。**具体的な採用ツールは、デプロイ先 CaaS や既存の監視基盤との連携しやすさに応じて選ぶため、検討事項として選択肢を示す。**

> **【検討事項】メトリクス収集・APM（トレーシング）**
> - **方式A: OpenTelemetry（OTel）** — Next.js は OpenTelemetry を標準サポートしており、`instrumentation.ts` で計装できる。特定ベンダーに依存せず、Datadog/New Relic/Grafana Cloud など収集先を後から選べる。
> - **方式B: CaaS ネイティブの監視機能**（CloudWatch/Cloud Monitoring/Azure Monitor 等） — 追加コンポーネントなしで導入できるが、CaaS を変更すると移行コストが発生する。
> - **方式C: SaaS型APM（Datadog、New Relic 等）** — 導入が早く高機能だが、コストと外部送信データの範囲を要検討。
>
> 最低限、リクエストのレスポンスタイム分布とエラーレートは可視化できる状態にすることを推奨する。

> **【検討事項】エラートラッキング**
> 9 章のエラーハンドリングは「ログに出力する」までを定めているが、**発生した例外を人が検知する仕組み（集約・通知）**は別途必要である。Sentry 等の専用サービスを使うか、ログ収集基盤（CloudWatch Logs Insights 等）のアラートルールで代替するかは、チームの運用体制に応じて決める。

- **トレースID / リクエストIDの伝播**: `proxy.ts` または各リクエストの入口で `crypto.randomUUID()` 等により生成し、`shared/lib/logger.ts` の子ロガーに束縛して、そのリクエストに関わる全てのログ行に同じ ID を含める。上記の APM を導入する場合は、APM が発行する trace ID をそのまま流用してもよい。

---

## 18. データ保持・バックアップ・災害復旧（DR）

> **【検討事項】バックアップ方式と RPO/RTO**
> - マネージド PostgreSQL（RDS、Cloud SQL 等）を利用する場合、自動スナップショット・ポイントインタイムリカバリ（PITR）機能に乗せるのが基本方針となる。**スナップショット頻度と保持期間、および目標復旧時点（RPO）・目標復旧時間（RTO）は要件定義側で確定させ、マネージドサービスの設定に反映する。**
> - セルフホストの PostgreSQL コンテナを使う場合は、`pg_dump`/`pg_basebackup` 等によるバックアップの取得・保管先（オブジェクトストレージ等）・リストア手順の検証を別途設計する必要がある。
> - リストア後、アプリケーション側のマイグレーション履歴（`_prisma_migrations` テーブル）とスキーマが整合していることを確認する手順を運用ドキュメントに残す。

> **【検討事項】データ保持期間・物理パージ**
> 5.2 節で論理削除（`deletedAt`）を方針化しているが、削除後のデータをいつまで保持するか（法令・社内規程によるデータ保持期間）は業種・要件に依存する。保持期間が確定した時点で、定期パージ処理（バッチジョブ等）の実装を検討する。

---

## 19. 今後の拡張ポイント（未決定事項）

以下は本ガイドの初版では確定していない、要件に応じて追加検討すべき項目です。実装前に人間側の判断を確認してください（各項目の詳細な選択肢は該当章を参照）。

- セッション無効化（強制ログアウト）が必要かどうか（JWT戦略 vs database戦略、6.1節）
- マルチテナント設計の要否（`organizationId` による分離が必要か、単一組織前提か）
- 監査ログ（誰が何を変更したかの履歴）の永続化要否
- コネクションプール管理の採用方式（5.5節: PgBouncer / connection_limit調整 / Prisma Accelerate）
- マイグレーション実行タイミングの方式（5.4節: entrypoint実行 / CI専用ステップ / Expand-and-Contract）
- レート制限の実装場所（15.1節: アプリ内 / CaaS側インフラ / 併用）
- Content-Security-Policy の具体的な値（15.2節、UIライブラリ確定後に設計）
- メトリクス収集・APM、エラートラッキングの採用ツール（17章）
- liveness/readiness のCaaS側での割り当て方法（12.3節）
- 許容ダウンタイム・RTOの目標値とそれに応じたデプロイ戦略（16章）
- バックアップ方式・RPO/RTO・データ保持期間（18章）
- デプロイ先 CaaS の確定（Cloud Run / ECS Fargate / Azure Container Apps 等）とそれに応じた CI/CD 設計

