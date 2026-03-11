# GitHub Issue ドラフト（実装フェーズ対応）

本ファイルは、`gh issue create` が使えない環境でもそのまま転記できるように、Issue 本文を定義したもの。

## Issue 1: Phase 0 - API/セキュリティの実装既定値を固定する

- Labels: `phase:0`, `type:spec`, `priority:high`

### Title
`Phase 0: API/セキュリティ実装既定値の固定`

### Body
```
## 背景
詳細仕様に実装前提の既定値が不足しており、実装時の解釈ぶれリスクがある。

## スコープ
- Cookie属性（HttpOnly/Secure/SameSite/Max-Age）
- CSRF方式（Double Submit Cookie）
- pagination meta 形式
- validation_error.details.fields 形式
- category/tag 正規化

## 受け入れ条件
- [ ] detailed-specification.md に既定値が反映されている
- [ ] 非対象（MVP外）が明示されている
- [ ] 次フェーズ（認証実装）が迷わず着手できる
```

---

## Issue 2: Phase 1 - 認証/セッション API を実装する

- Labels: `phase:1`, `type:backend`, `priority:high`

### Title
`Phase 1: Google OAuth + Session + Me API の実装`

### Body
```
## 背景
記事機能着手前に、認証済みユーザーの同定を可能にする必要がある。

## スコープ
- GET /api/auth/google/start
- GET /api/auth/google/callback
- GET /api/session
- DELETE /api/session
- GET/PATCH /api/me

## 受け入れ条件
- [ ] state 不正時に validation_error を返す
- [ ] セッションCookieが仕様どおりに設定される
- [ ] ログイン後に /api/session で user/role を取得できる
```

---

## Issue 3: Phase 2 - 記事作成/公開と権限制御を実装する

- Labels: `phase:2`, `type:backend`, `priority:high`

### Title
`Phase 2: Post/Revision API と visibility 権限制御`

### Body
```
## 背景
MVP の中心価値である「記事作成→公開→閲覧制御」を成立させる。

## スコープ
- POST /api/posts
- POST /api/posts/:postId/revisions
- GET /api/posts
- GET /api/posts/:postId
- wip/published の状態遷移制御

## 受け入れ条件
- [ ] wip -> published で初回公開通知イベントを作成
- [ ] 権限外アクセスは 404 not_found
- [ ] expected_revision_number 不一致で 409 conflict
```

---

## Issue 4: Phase 3 - コメント/メンション/通知を実装する

- Labels: `phase:3`, `type:backend`, `priority:medium`

### Title
`Phase 3: コメント・メンション・通知 API 実装`

### Body
```
## 背景
公開された記事に対するコラボレーション体験を提供する。

## スコープ
- GET/POST /api/posts/:postId/comments
- POST /api/comments/:commentId/revisions
- POST /api/comments/:commentId/archive
- GET /api/notifications
- POST /api/notifications/read

## 受け入れ条件
- [ ] 記事作成者への comment_created 通知
- [ ] mention_created 通知（重複抑止）
- [ ] 自己通知が作成されない
```

---

## Issue 5: Phase 4 - 管理機能と運用基盤を実装する

- Labels: `phase:4`, `type:backend`, `priority:medium`

### Title
`Phase 4: グループ管理・Slack設定・bootstrap運用`

### Body
```
## 背景
管理者が本番運用を開始/継続できる機能が必要。

## スコープ
- /api/groups 系
- /api/admin/settings/slack
- /api/admin/system/init-status
- /api/admin/init/bootstrap-admin

## 受け入れ条件
- [ ] 管理者グループ所属者のみ管理 API を利用可能
- [ ] Slack 設定変更を監査可能
- [ ] bootstrap が再実行可能である
```
