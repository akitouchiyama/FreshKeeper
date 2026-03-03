# FreshKeeper 技術設計書

**バージョン**: 1.0
**作成日**: 2026-03-02
**ステータス**: ドラフト

---

## 1. アーキテクチャ概要

### 1.1 全体構成

```text
┌─────────────────┐     ┌──────────────────────┐     ┌─────────────┐
│  Cloudflare      │     │  Cloudflare Workers   │     │   Neon       │
│  Pages           │────▶│  (Hono API)           │────▶│  PostgreSQL  │
│  (React SPA)     │     │  + Better Auth        │     │              │
└─────────────────┘     └──────────────────────┘     └─────────────┘
      フロントエンド            バックエンド               データベース
```

### 1.2 技術スタック

| レイヤー | 技術 | バージョン |
|---------|------|-----------|
| フロントエンド | React + TypeScript | React 19 |
| ビルドツール | Vite | 6.x |
| スタイリング | Tailwind CSS | 4.x |
| ルーティング（FE） | React Router | 7.x |
| 状態管理 | TanStack Query | 5.x |
| バックエンド | Hono + TypeScript | 4.x |
| バリデーション | Zod | 3.x |
| ORM | Drizzle ORM | latest |
| データベース | PostgreSQL (Neon) | 17 |
| 認証 | Better Auth | 1.5.x |
| パッケージマネージャー | pnpm | 10.x |
| デプロイ | Cloudflare (Pages / Workers) | - |

---

## 2. プロジェクト構成（モノレポ）

### 2.1 ディレクトリ構成

```text
FreshKeeper/
├── packages/
│   ├── frontend/                 # React SPA
│   │   ├── src/
│   │   │   ├── components/       # 共通UIコンポーネント
│   │   │   │   ├── ui/           # ボタン、入力等の基礎コンポーネント
│   │   │   │   └── layout/       # ヘッダー、フッター、レイアウト
│   │   │   ├── features/         # 機能別モジュール
│   │   │   │   ├── auth/         # 認証関連（ログイン、登録画面等）
│   │   │   │   ├── foods/        # 食品管理（一覧、登録、編集等）
│   │   │   │   └── notifications/# 期限間近の警告表示関連
│   │   │   ├── hooks/            # カスタムフック
│   │   │   ├── lib/              # ユーティリティ、API クライアント
│   │   │   ├── routes/           # ルーティング定義
│   │   │   ├── types/            # 型定義
│   │   │   ├── App.tsx
│   │   │   └── main.tsx
│   │   ├── public/
│   │   ├── index.html
│   │   ├── tailwind.config.ts
│   │   ├── tsconfig.json
│   │   ├── vite.config.ts
│   │   └── package.json
│   │
│   └── backend/                  # Hono API (Cloudflare Workers)
│       ├── src/
│       │   ├── routes/           # APIルート定義
│       │   │   ├── foods.ts      # 食品CRUD
│       │   │   ├── notifications.ts # 通知設定
│       │   │   └── auth.ts       # Better Auth ハンドラー
│       │   ├── middleware/       # ミドルウェア（認証チェック等）
│       │   ├── db/
│       │   │   ├── schema.ts     # Drizzle スキーマ定義
│       │   │   ├── migrations/   # マイグレーションファイル
│       │   │   └── index.ts      # DB接続
│       │   ├── services/         # ビジネスロジック
│       │   ├── shared/           # フロント・バック共有コード
│       │   │   ├── types/        # 共通型定義（APIレスポンス等）
│       │   │   ├── constants/    # 共通定数（カテゴリ、単位等）
│       │   │   └── validators/   # 共通バリデーションスキーマ
│       │   ├── validators/       # Zod スキーマ（リクエストバリデーション）
│       │   ├── lib/              # ユーティリティ
│       │   │   └── auth.ts       # Better Auth 設定
│       │   └── index.ts          # エントリーポイント
│       ├── drizzle.config.ts
│       ├── tsconfig.json
│       ├── wrangler.toml         # Cloudflare Workers 設定
│       └── package.json
│
├── docs/                         # ドキュメント
│   ├── requirements.md
│   ├── technical-design.md
│   └── future-scope.md
├── .github/
│   └── workflows/                # CI/CD
├── pnpm-workspace.yaml
├── package.json                  # ルートpackage.json
├── tsconfig.base.json            # 共通TSConfig
├── .gitignore
├── .env.example
├── CLAUDE.md
└── README.md
```

### 2.2 pnpm workspace 設定

```yaml
# pnpm-workspace.yaml
packages:
  - "packages/*"
```

### 2.3 パッケージ間の依存関係

```text
backend ← frontend
```

- 共有コード（API の型定義、定数、共通バリデーションスキーマ）は `backend/src/shared/` に配置し、`backend` パッケージからexportする
- `frontend` は `backend` パッケージの共有コードをimportすることでフロント・バック間の型安全性を保証

---

## 3. データベース設計

### 3.1 ER図

```text
┌──────────────┐     ┌──────────────────┐     ┌──────────────────┐
│    user       │     │     session       │     │    account        │
│──────────────│     │──────────────────│     │──────────────────│
│ id (PK)       │◀───│ userId (FK)       │     │ userId (FK)       │──▶│
│ name          │     │ id (PK)           │     │ id (PK)           │
│ email         │     │ token             │     │ providerId        │
│ emailVerified │     │ expiresAt         │     │ accountId         │
│ image         │     │ ipAddress         │     │ ...               │
│ createdAt     │     │ userAgent         │     └──────────────────┘
│ updatedAt     │     │ createdAt         │
└──────────────┘     │ updatedAt         │
       │              └──────────────────┘
       │
       │ 1:N
       ▼
┌──────────────────────┐     ┌──────────────────────┐
│       food            │     │  notification_setting  │
│──────────────────────│     │──────────────────────│
│ id (PK)               │     │ id (PK)               │
│ userId (FK)           │     │ userId (FK, UNIQUE)    │
│ name                  │     │ consumeDaysBefore      │
│ quantity              │     │ bestBeforeDaysBefore   │
│ unit                  │     │ createdAt              │
│ expiryType            │     │ updatedAt              │
│ expiryDate            │     └──────────────────────┘
│ purchaseDate          │
│ category              │
│ storageLocation       │
│ memo                  │
│ createdAt             │
│ updatedAt             │
└──────────────────────┘
```

### 3.2 テーブル定義

#### user / session / account / verification テーブル
Better Auth が自動生成・管理するテーブル。`npx auth generate` でスキーマを生成。

#### food テーブル

| カラム | 型 | 制約 | 説明 |
|--------|-----|------|------|
| id | uuid | PK, DEFAULT gen_random_uuid() | 食品ID |
| userId | text | FK → user.id, NOT NULL | 所有ユーザー |
| name | varchar(100) | NOT NULL | 食品名 |
| quantity | decimal(10,2) | NOT NULL, DEFAULT 1 | 数量 |
| unit | varchar(20) | NOT NULL | 単位（個、本、パック等） |
| expiryType | varchar(20) | NOT NULL | 期限種別（consume / best_before） |
| expiryDate | date | NOT NULL | 期限日 |
| purchaseDate | date | | 購入日 |
| category | varchar(20) | NOT NULL | カテゴリ（refrigerated / frozen / room_temp） |
| storageLocation | varchar(20) | NOT NULL | 保存場所（fridge / freezer / vegetable_room / pantry / other） |
| memo | text | | メモ |
| createdAt | timestamp | NOT NULL, DEFAULT now() | 作成日時 |
| updatedAt | timestamp | NOT NULL, DEFAULT now() | 更新日時 |

**インデックス**:
- `idx_food_user_id` : userId
- `idx_food_user_expiry` : userId, expiryDate（期限順一覧の高速化）
- `idx_food_user_category` : userId, category（カテゴリフィルターの高速化）

#### notification_setting テーブル

画面上で「期限間近」の警告表示を何日前から出すかをユーザーが設定するためのテーブル。

| カラム | 型 | 制約 | 説明 |
|--------|-----|------|------|
| id | uuid | PK, DEFAULT gen_random_uuid() | 設定ID |
| userId | text | FK → user.id, UNIQUE, NOT NULL | ユーザー |
| consumeDaysBefore | integer[] | NOT NULL, DEFAULT '{3,1,0}' | 消費期限の警告表示日（何日前） |
| bestBeforeDaysBefore | integer[] | NOT NULL, DEFAULT '{7,3,0}' | 賞味期限の警告表示日（何日前） |
| createdAt | timestamp | NOT NULL, DEFAULT now() | 作成日時 |
| updatedAt | timestamp | NOT NULL, DEFAULT now() | 更新日時 |

### 3.3 Drizzle スキーマ（概要）

```typescript
// packages/backend/src/db/schema.ts
import { pgTable, text, varchar, decimal, date, timestamp, integer, uuid } from "drizzle-orm/pg-core";

// Better Auth テーブルは npx auth generate で生成

export const food = pgTable("food", {
  id: uuid("id").primaryKey().defaultRandom(),
  userId: text("user_id").notNull().references(() => user.id, { onDelete: "cascade" }),
  name: varchar("name", { length: 100 }).notNull(),
  quantity: decimal("quantity", { precision: 10, scale: 2 }).notNull().default("1"),
  unit: varchar("unit", { length: 20 }).notNull(),
  expiryType: varchar("expiry_type", { length: 20 }).notNull(),
  expiryDate: date("expiry_date").notNull(),
  purchaseDate: date("purchase_date"),
  category: varchar("category", { length: 20 }).notNull(),
  storageLocation: varchar("storage_location", { length: 20 }).notNull(),
  memo: text("memo"),
  createdAt: timestamp("created_at").notNull().defaultNow(),
  updatedAt: timestamp("updated_at").notNull().defaultNow(),
});

export const notificationSetting = pgTable("notification_setting", {
  id: uuid("id").primaryKey().defaultRandom(),
  userId: text("user_id").notNull().references(() => user.id, { onDelete: "cascade" }).unique(),
  consumeDaysBefore: integer("consume_days_before").array().notNull().default([3, 1, 0]),
  bestBeforeDaysBefore: integer("best_before_days_before").array().notNull().default([7, 3, 0]),
  createdAt: timestamp("created_at").notNull().defaultNow(),
  updatedAt: timestamp("updated_at").notNull().defaultNow(),
});
```

---

## 4. API設計

### 4.1 ベースURL

- 開発: `http://localhost:8787`
- 本番: `https://api.freshkeeper.example.com`

### 4.2 認証エンドポイント（Better Auth管理）

| メソッド | パス | 説明 |
|---------|------|------|
| POST | /api/auth/sign-up/email | メール/パスワードで新規登録 |
| POST | /api/auth/sign-in/email | メール/パスワードでログイン |
| POST | /api/auth/sign-out | ログアウト |
| GET | /api/auth/session | セッション取得 |

### 4.3 食品管理エンドポイント

すべて認証必須（Better Auth セッションミドルウェアで保護）。

| メソッド | パス | 説明 |
|---------|------|------|
| GET | /api/foods | 食品一覧取得 |
| POST | /api/foods | 食品登録 |
| GET | /api/foods/:id | 食品詳細取得 |
| PUT | /api/foods/:id | 食品情報更新 |
| PATCH | /api/foods/:id/quantity | 数量の部分更新（消費） |
| DELETE | /api/foods/:id | 食品削除 |

#### GET /api/foods クエリパラメータ

| パラメータ | 型 | 説明 |
|-----------|-----|------|
| sort | string | ソート項目（expiry_date, created_at, name） |
| order | string | 昇順/降順（asc, desc） |
| category | string | カテゴリフィルター |
| storageLocation | string | 保存場所フィルター |
| search | string | 食品名検索 |
| page | number | ページ番号（デフォルト: 1） |
| limit | number | 1ページあたり件数（デフォルト: 20） |

#### POST /api/foods リクエストボディ

```json
{
  "name": "牛乳",
  "quantity": 1,
  "unit": "本",
  "expiryType": "consume",
  "expiryDate": "2026-03-10",
  "purchaseDate": "2026-03-02",
  "category": "refrigerated",
  "storageLocation": "fridge",
  "memo": ""
}
```

#### PATCH /api/foods/:id/quantity リクエストボディ

```json
{
  "consumeQuantity": 2
}
```

### 4.4 通知設定エンドポイント（画面表示設定）

期限間近の警告表示を何日前から出すかの設定。認証必須。

| メソッド | パス | 説明 |
|---------|------|------|
| GET | /api/notification-settings | 警告表示設定取得 |
| PUT | /api/notification-settings | 警告表示設定更新 |

### 4.5 共通レスポンス形式

```typescript
// 成功
{
  "data": T,
  "meta"?: {
    "page": number,
    "limit": number,
    "total": number,
    "totalPages": number
  }
}

// エラー
{
  "error": {
    "code": string,       // "VALIDATION_ERROR", "NOT_FOUND", "UNAUTHORIZED" 等
    "message": string     // 人が読めるメッセージ
  }
}
```

### 4.6 HTTPステータスコード

| コード | 用途 |
|-------|------|
| 200 | 取得・更新成功 |
| 201 | 作成成功 |
| 400 | バリデーションエラー |
| 401 | 未認証 |
| 403 | 権限なし |
| 404 | リソースが見つからない |
| 500 | サーバーエラー |

---

## 5. 認証設計

### 5.1 Better Auth 設定概要

```typescript
// packages/backend/src/lib/auth.ts
import { betterAuth } from "better-auth";
import { drizzleAdapter } from "better-auth/adapters/drizzle";
import { db } from "../db";

export const auth = betterAuth({
  baseURL: process.env.BETTER_AUTH_URL,
  secret: process.env.BETTER_AUTH_SECRET,
  database: drizzleAdapter(db, {
    provider: "pg",
  }),
  emailAndPassword: {
    enabled: true,
  },
});
```

### 5.2 認証フロー

1. **メール/パスワード登録**: `/api/auth/sign-up/email` → user + account レコード作成 → セッション発行
2. **メール/パスワードログイン**: `/api/auth/sign-in/email` → 認証 → セッション発行
3. **セッション検証**: 各APIリクエスト時にミドルウェアで `auth.api.getSession()` を呼び出し
4. **ログアウト**: `/api/auth/sign-out` → セッション無効化

### 5.3 認証ミドルウェア

```typescript
// packages/backend/src/middleware/auth.ts
import { createMiddleware } from "hono/factory";
import { auth } from "../lib/auth";

export const requireAuth = createMiddleware(async (c, next) => {
  const session = await auth.api.getSession({ headers: c.req.raw.headers });
  if (!session) {
    return c.json({ error: { code: "UNAUTHORIZED", message: "認証が必要です" } }, 401);
  }
  c.set("user", session.user);
  c.set("session", session.session);
  await next();
});
```

---

## 6. デプロイ構成

### 6.1 環境

| 環境 | フロントエンド | バックエンド | データベース |
|------|--------------|------------|------------|
| 開発 | localhost:5173 (Vite) | localhost:8787 (wrangler dev) | Neon (dev branch) |
| 本番 | Cloudflare Pages | Cloudflare Workers | Neon (main branch) |

### 6.2 環境変数

```env
# バックエンド
DATABASE_URL=postgresql://...@neon.tech/freshkeeper
BETTER_AUTH_URL=https://api.freshkeeper.example.com
BETTER_AUTH_SECRET=<ランダムな秘密鍵>

# フロントエンド
VITE_API_URL=https://api.freshkeeper.example.com
```

### 6.3 Cloudflare Workers 設定

```toml
# packages/backend/wrangler.toml
name = "freshkeeper-api"
main = "src/index.ts"
compatibility_date = "2026-03-01"

[vars]
BETTER_AUTH_URL = "https://api.freshkeeper.example.com"
```

Neon への接続は Cloudflare Hyperdrive または直接接続（`@neondatabase/serverless` ドライバー）を使用。

### 6.4 CI/CD（GitHub Actions）

- **PR時**: lint + 型チェック

デプロイは手動（`wrangler deploy` / Cloudflare Pages Git連携）で運用。

---

## 7. テスト方針

### 7.1 テスト種別

| 種別 | ツール | 対象 | カバレッジ目標 |
|------|-------|------|-------------|
| ユニットテスト | Vitest | バリデーション、ビジネスロジック | 主要ロジック |
| 統合テスト | Vitest | APIエンドポイント | 主要エンドポイント |
| E2Eテスト | Playwright | ユーザーフロー | 主要ユーザーストーリー |

### 7.2 テスト対象の優先順位

1. 食品のCRUD操作（バリデーション含む）
2. 認証フロー（登録、ログイン、ログアウト）
3. 数量の部分消費ロジック
4. 食品一覧のフィルター・ソート
5. 警告表示設定の更新

---

## 8. 開発フロー

### 8.1 ローカル開発

```bash
# 依存インストール
pnpm install

# 開発サーバー起動（フロント + バック同時）
pnpm dev

# DBマイグレーション
pnpm --filter backend db:migrate

# テスト実行
pnpm test
```

### 8.2 ブランチ戦略

CLAUDE.md に定義されたブランチ命名規則に従う（feature/ , fix/ , docs/ 等）。
