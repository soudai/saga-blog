# 実装フェーズ計画（MVP）

Status: Draft  
Date: 2026-03-11

## 1. フェーズ全体像

| Phase | 目的 | 主な成果物 | 完了条件 |
|---|---|---|---|
| Phase 0: 実装基盤の確定 | 実装を止めない前提条件を決める | API 契約固定、セキュリティ既定値、Issue 分解 | 主要仕様の未確定論点が「保留」ではなく「採用案」になっている |
| Phase 1: 認証・ユーザー基盤 | ログインとセッションを成立させる | Google OAuth, セッション API, `/api/me` | ログイン〜ログアウトまで一連の手動検証が通る |
| Phase 2: 記事の作成/公開 | MVP の中核である記事公開を成立させる | Post/Revision API, 権限制御, 一覧/詳細 | WIP 作成→公開→閲覧制御の E2E が通る |
| Phase 3: 共同編集体験 | コメント・通知で運用できる状態にする | コメント API, メンション抽出, 通知一覧 | コメント通知・既読制御が通る |
| Phase 4: 管理機能/運用 | 管理者運用を可能にする | グループ管理, Slack 管理設定, bootstrap 運用 | 初期化・管理設定変更・監査ログ確認が通る |

## 2. フェーズ詳細

### Phase 0: 実装基盤の確定

- 対象:
  - API の共通 `meta` / `error.details` 形式の固定
  - Cookie / CSRF / Validation 境界の固定
  - カテゴリ/タグ正規化ルールの固定
- 非対象:
  - UI 実装
  - 外部通知の実送信
- DoD:
  - 詳細仕様に実装既定値が反映され、Issue 参照で追跡できる

### Phase 1: 認証・ユーザー基盤

- 対象:
  - `GET /api/auth/google/start`
  - `GET /api/auth/google/callback`
  - `GET /api/session`, `DELETE /api/session`
  - `GET /api/me`, `PATCH /api/me`
- 依存:
  - Phase 0 のセキュリティ既定値確定
- DoD:
  - state 不正時は `validation_error`
  - セッション Cookie 属性が仕様通りである

### Phase 2: 記事の作成/公開

- 対象:
  - `POST /api/posts`, `POST /api/posts/:postId/revisions`
  - `GET /api/posts`, `GET /api/posts/:postId`
  - visibility/public/group の権限制御
- 依存:
  - Phase 1 認証基盤
- DoD:
  - `wip -> published` で初回公開通知イベントが作成される
  - 権限なしアクセスは `404 not_found`

### Phase 3: 共同編集体験

- 対象:
  - コメント作成/更新/アーカイブ
  - メンション抽出
  - 通知一覧/既読 API
- 依存:
  - Phase 2 の公開状態遷移
- DoD:
  - 自己通知が抑止される
  - 同一コメント内の同一メンション重複通知が抑止される

### Phase 4: 管理機能/運用

- 対象:
  - グループ CRUD/メンバー管理
  - Slack 設定管理
  - bootstrap-admin API / `make init` 運用
- 依存:
  - Phase 1〜3
- DoD:
  - 管理者のみが管理 API を操作できる
  - 失敗時の再実行指針に沿って初期化をリカバリーできる
