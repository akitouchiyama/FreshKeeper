# FreshKeeper 将来対応スコープ

**作成日**: 2026-03-03
**目的**: 初期リリースでは見送った機能・構成を記録し、将来の拡張時に参照する

---

## 1. 認証機能の拡張

### Google OAuth ログイン
- Better Auth の `socialProviders` に Google を追加
- 必要な設定: GCP Console でのOAuthクライアント作成、リダイレクトURI設定
- 環境変数の追加: `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET`
- エンドポイント: `POST /api/auth/sign-in/social`

### パスワードリセット
- Better Auth の `emailAndPassword.sendResetPassword` を有効化
- メール送信サービス（Resend等）の契約・設定が必要
- エンドポイント: `POST /api/auth/request-password-reset`, `POST /api/auth/reset-password`

---

## 2. 通知機能の拡張

### Webプッシュ通知
- Service Worker + Push API の実装が必要
- 通知サーバー（Cloudflare Workers の Cron Triggers 等）で定期実行
- `notification_setting` テーブルに `enablePush` (boolean) カラムを追加

### メール通知
- メール送信サービス（Resend等）の契約が必要
- `notification_setting` テーブルに `enableEmail` (boolean) カラムを追加
- Cron Triggers で期限間近の食品をチェックし、メール送信

---

## 3. プロジェクト構成の拡張

### shared パッケージの分離
- 現在は `backend/src/shared/` に共有コードを配置
- 共有コードが増えた場合、独立した `packages/shared/` パッケージに分離を検討
- 分離の判断基準: backend 固有のコードと共有コードの境界が曖昧になったとき

---

## 4. CI/CD の拡張

### 自動デプロイ
- **main マージ時**: GitHub Actions で自動デプロイ（Cloudflare Pages + Workers）
- PR時のテスト実行の追加

### テスト自動実行
- PR時の CI に `pnpm test` を追加（ユニットテスト + 統合テスト）

---

## 5. インフラの拡張

### Cloudflare Hyperdrive
- Neon への接続レイテンシが問題になった場合に導入を検討
- 月額 $5〜 のコストが発生
- 現在は `@neondatabase/serverless` ドライバーの直接接続で運用
