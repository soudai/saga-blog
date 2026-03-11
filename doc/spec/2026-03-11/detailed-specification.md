# 社内ナレッジ共有サービス 詳細仕様

Status: Draft
Date: 2026-03-11

## 1. 目的

本書は [仕様書](/home/soudai/takt/saga-blog/doc/spec/2026-03-11/product-specification.md) を実装可能な粒度に落とし込み、主要ユースケース、画面挙動、API 入出力、権限制御を定義する。

## 2. 共通仕様

### 2.1 API 共通規約

- ベースパスは `/api` とする
- 認証は Cookie ベースのセッション認証とする
- 状態変更 API では CSRF 対策を行う
- リクエストとレスポンスは JSON を基本とする
- タイムスタンプは UTC の ISO 8601 文字列で返す
- ID は UUID 文字列で扱う
- 一覧 API は cursor pagination を採用する
- `limit` の既定値は `20`、最大値は `100` とする

### 2.2 レスポンス形式

成功時:

```json
{
  "data": {},
  "meta": {}
}
```

失敗時:

```json
{
  "error": {
    "code": "forbidden",
    "message": "You do not have permission to access this resource.",
    "details": {}
  }
}
```

### 2.3 エラーコード

| code | 意味 |
|---|---|
| `unauthenticated` | 未ログイン |
| `forbidden` | 権限不足 |
| `not_found` | 対象が存在しない、または存在を秘匿する |
| `conflict` | 楽観ロック競合、重複、状態遷移不正 |
| `validation_error` | 入力不正 |
| `external_service_error` | Slack など外部サービス連携失敗 |

### 2.4 列挙値

| 名称 | 値 |
|---|---|
| `article_status` | `wip`, `published` |
| `visibility_scope` | `public`, `group` |
| `group_type` | `admin`, `regular` |
| `notification_type` | `article_published`, `comment_created`, `mention_created` |
| `delivery_channel` | `slack_dm`, `slack_channel` |
| `delivery_status` | `pending`, `sent`, `failed`, `skipped` |

## 3. 権限制御詳細

### 3.1 記事アクセスマトリクス

| 条件 | 一覧表示 | 詳細表示 | コメント | Star | 通知対象 |
|---|---|---|---|---|---|
| 自分の `WIP` | 可 | 可 | 不可 | 不可 | 不可 |
| 他人の `WIP` | 不可 | 不可 | 不可 | 不可 | 不可 |
| `公開 + public` | 可 | 可 | 可 | 可 | 可 |
| `公開 + group` かつ対象グループ所属 | 可 | 可 | 可 | 可 | 可 |
| `公開 + group` かつ対象グループ非所属 | 不可 | 不可 | 不可 | 不可 | 不可 |

### 3.2 管理権限

| 操作 | 一般ユーザー | 管理者グループ所属 |
|---|---|---|
| グループ作成 | 不可 | 可 |
| グループ更新 | 不可 | 可 |
| グループメンバー追加 / 削除 | 不可 | 可 |
| Slack 管理設定更新 | 不可 | 可 |
| bootstrap 状態確認 | 不可 | 可 |

### 3.3 状態遷移

- 新規記事は既定で `wip` とする
- `wip -> published` は許可する
- `published -> wip` は MVP では許可しない
- `published` 記事の本文、カテゴリ、タグ、公開範囲は新しい revision 追加で更新できる
- `visibility_scope=group` の場合は `visible_group_id` を必須とする
- `visibility_scope=public` の場合は `visible_group_id` を `null` とする

## 4. 主要ユースケース

### 4.1 初期管理者 bootstrap

1. 運用者が `make init ADMIN_EMAIL=... ADMIN_DISPLAY_NAME=...` を実行する
2. システムは管理者グループの存在を確認し、未作成なら作成する
3. システムは初期管理者ユーザーを作成する
4. システムは初期管理者を管理者グループへ所属させる
5. すでに作成済みなら no-op として成功する

### 4.2 初回ログイン

1. ユーザーが Google ログインを開始する
2. Google callback で認証情報を受け取る
3. Google email が bootstrap 済みユーザーと一致する場合は既存ユーザーへ紐付ける
4. 一致しない場合は新規ユーザーを作成する
5. セッションを発行してホームへ遷移する

### 4.3 記事作成から公開まで

1. ユーザーは `/posts/new` で記事を入力する
2. `POST /api/posts` で初回 revision を作成する
3. 初回作成時の status は既定で `wip`
4. 作成者だけが `WIP` を閲覧、編集できる
5. 公開時は `POST /api/posts/:postId/revisions` で `status=published` の revision を追加する
6. 初回 publish 時のみ `article_published` 通知イベントを生成する

### 4.4 グループ限定公開

1. 作成者は `visibility_scope=group` と `visible_group_id` を指定して公開する
2. 対象グループ所属者だけが一覧、詳細、検索、通知からその記事を扱える
3. 公開後に可視範囲が変わった場合、表示と API 応答は常に現在の revision を基準に再評価する

### 4.5 コメントとメンション

1. 閲覧可能な `published` 記事にのみコメントできる
2. コメント保存時に本文からメンションを抽出する
3. `comment_created` 通知は記事作成者へ送る。ただし自分自身への通知は作らない
4. `mention_created` 通知は対象ユーザーへ送る。ただし記事閲覧権限がある場合に限る
5. 同じコメントで同一ユーザーが複数回メンションされても通知は 1 件とする

### 4.6 通知配信

1. イベント発生時に recipient ごとのアプリ内通知を作成する
2. 通知作成後に Slack 配信対象を判定する
3. public 記事の publish は announcement channel 通知を許可する
4. group 記事の publish は DM ベースのみとし、チャンネル一斉投稿はしない
5. Slack identity 未連携ユーザーはアプリ内通知のみとする
6. 通知表示時は現在の閲覧権限を再確認し、権限を失った通知は本文を伏せて表示する

## 5. 画面詳細

### 5.1 `/`

- 最近公開された閲覧可能記事を表示する
- 未読通知件数を表示する
- 自分の未公開 `WIP` への導線を表示する

### 5.2 `/posts`

- 閲覧可能な `published` 記事のみ一覧表示する
- カテゴリ、タグ、作成者で絞り込める
- 自分の `WIP` はこの一覧に含めない

### 5.3 `/posts/new`

- タイトル、本文、カテゴリ、タグ、公開範囲を入力できる
- 初期値は `status=wip`, `visibility_scope=public`
- 保存成功後は `/posts/:postId/edit` へ遷移する

### 5.4 `/posts/:postId`

- 最新 revision の内容を表示する
- コメント一覧、Star 数、自分の Star 状態を表示する
- 権限を失った場合は `404` と同等の扱いにする

### 5.5 `/posts/:postId/edit`

- 現在の最新 revision を初期値として読み込む
- 保存時は `expected_revision_number` を送る
- 競合時は `409 conflict` を返し、再読み込みを促す
- 作成者以外は表示できない

### 5.6 `/search`

- キーワード、カテゴリ、タグで検索できる
- 検索結果は閲覧可能な `published` 記事のみ表示する
- 自分の `WIP` を含む個人検索は MVP 対象外とする

### 5.7 `/notifications`

- 通知一覧を新しい順で表示する
- 表示時に通知対象記事への current access を再評価する
- 一括既読と個別既読をサポートする

### 5.8 `/admin/groups` 系

- グループ一覧、作成、編集、メンバー追加 / 削除を提供する
- すべて管理者グループ所属ユーザーのみアクセス可能とする

### 5.9 `/admin/integrations/slack`

- Slack bot 設定の有効 / 無効
- announcement channel 設定
- 設定更新履歴の確認

## 6. リソース仕様

### 6.1 UserSummary

| field | type | 説明 |
|---|---|---|
| `user_id` | `string` | UUID |
| `display_name` | `string` | 表示名 |
| `avatar_url` | `string|null` | アイコン URL |

### 6.2 GroupSummary

| field | type | 説明 |
|---|---|---|
| `group_id` | `string` | UUID |
| `slug` | `string` | 一意な識別子 |
| `name` | `string` | 表示名 |
| `description` | `string|null` | 説明 |
| `group_type` | `string` | `admin` または `regular` |
| `member_count` | `number` | 現在のメンバー数 |

### 6.3 PostSummary

| field | type | 説明 |
|---|---|---|
| `post_id` | `string` | UUID |
| `title` | `string` | 現在タイトル |
| `category_path` | `string` | 例 `engineering/backend` |
| `tags` | `string[]` | 現在タグ |
| `status` | `string` | `published` のみ返却 |
| `visibility_scope` | `string` | `public` または `group` |
| `visible_group` | `object|null` | group 公開時のみ設定 |
| `author` | `UserSummary` | 作成者 |
| `last_editor` | `UserSummary` | 最新編集者 |
| `revision_number` | `number` | 現在 revision 番号 |
| `star_count` | `number` | 現在 Star 数 |
| `comment_count` | `number` | 現在コメント数 |
| `published_at` | `string` | 初回公開時刻 |
| `updated_at` | `string` | 最新更新時刻 |

### 6.4 PostDetail

`PostSummary` に加えて以下を含む。

| field | type | 説明 |
|---|---|---|
| `body_md` | `string` | Markdown 本文 |
| `body_html` | `string` | サーバー描画済み HTML |
| `change_summary` | `string|null` | 最新更新要約 |
| `starred_by_me` | `boolean` | 自分が Star 済みか |
| `can_edit` | `boolean` | 編集可能か |
| `can_comment` | `boolean` | コメント可能か |

### 6.5 Comment

| field | type | 説明 |
|---|---|---|
| `comment_id` | `string` | UUID |
| `post_id` | `string` | 対象記事 |
| `body_md` | `string` | Markdown 本文 |
| `body_html` | `string` | 描画済み HTML |
| `author` | `UserSummary` | 投稿者 |
| `revision_number` | `number` | 現在 revision 番号 |
| `is_archived` | `boolean` | 削除済み相当か |
| `created_at` | `string` | 作成時刻 |
| `updated_at` | `string` | 最終更新時刻 |

### 6.6 Notification

| field | type | 説明 |
|---|---|---|
| `notification_id` | `string` | UUID |
| `type` | `string` | 通知種別 |
| `is_read` | `boolean` | 既読状態 |
| `actor` | `UserSummary|null` | 発生元ユーザー |
| `post_id` | `string|null` | 対象記事 |
| `comment_id` | `string|null` | 対象コメント |
| `title` | `string` | 表示タイトル |
| `excerpt` | `string|null` | 表示用抜粋 |
| `link_path` | `string|null` | 遷移先 |
| `created_at` | `string` | 通知作成時刻 |
| `access_revoked` | `boolean` | 現在アクセス不可か |

## 7. API 詳細

### 7.1 認証・セッション

#### `GET /api/auth/google/start`

- 認証: 不要
- 入力:
  - `redirect_to?: string`
- 出力:
  - `302` Google OAuth URL へリダイレクト

#### `GET /api/auth/google/callback`

- 認証: 不要
- 入力:
  - `code: string`
  - `state: string`
- 出力:
  - `302` フロントエンド画面へリダイレクト
- 副作用:
  - セッション発行
  - ユーザー作成または既存ユーザー連携

#### `GET /api/session`

- 認証: 必須
- 出力:

```json
{
  "data": {
    "user": {
      "user_id": "uuid",
      "display_name": "Soudai",
      "avatar_url": null
    },
    "roles": {
      "is_admin": true
    }
  }
}
```

#### `DELETE /api/session`

- 認証: 必須
- 出力: `204 No Content`
- 副作用: セッション失効

### 7.2 自分 / ユーザー

#### `GET /api/me`

- 認証: 必須
- 出力:
  - `UserSummary`
  - `primary_email`
  - `slack_linked`
  - `google_linked`

#### `PATCH /api/me`

リクエスト:

```json
{
  "display_name": "Soudai",
  "avatar_url": "https://example.com/avatar.png"
}
```

レスポンス:

```json
{
  "data": {
    "user_id": "uuid",
    "display_name": "Soudai",
    "avatar_url": "https://example.com/avatar.png"
  }
}
```

#### `GET /api/users`

- 認証: 必須
- クエリ:
  - `cursor?: string`
  - `limit?: number`
  - `q?: string`

#### `GET /api/users/:userId`

- 認証: 必須
- 出力:
  - `UserSummary`
  - `public_post_count`
  - `group_post_count` は表示しない

#### `POST /api/me/slack/link`

リクエスト:

```json
{
  "oauth_code": "code-from-slack",
  "redirect_uri": "https://app.example.com/settings/slack"
}
```

レスポンス:

```json
{
  "data": {
    "slack_linked": true,
    "slack_user_id": "U12345678"
  }
}
```

#### `DELETE /api/me/slack/link`

- 認証: 必須
- 出力: `204 No Content`

### 7.3 グループ

#### `GET /api/groups`

- 認証: 必須
- クエリ:
  - `cursor?: string`
  - `limit?: number`
  - `q?: string`
- 出力: `GroupSummary[]`

#### `POST /api/groups`

認証: 管理者必須

```json
{
  "slug": "platform-team",
  "name": "Platform Team",
  "description": "Platform engineers"
}
```

レスポンス:

```json
{
  "data": {
    "group_id": "uuid",
    "slug": "platform-team",
    "name": "Platform Team",
    "description": "Platform engineers",
    "group_type": "regular",
    "member_count": 0
  }
}
```

#### `PATCH /api/groups/:groupId`

認証: 管理者必須

```json
{
  "name": "Platform and SRE Team",
  "description": "Platform and SRE engineers"
}
```

#### `GET /api/groups/:groupId/members`

- 認証: 管理者必須
- 出力:
  - `items: UserSummary[]`

#### `POST /api/groups/:groupId/members`

認証: 管理者必須

```json
{
  "user_id": "uuid"
}
```

- 成功: `204 No Content`

#### `DELETE /api/groups/:groupId/members/:userId`

- 認証: 管理者必須
- 成功: `204 No Content`

### 7.4 記事

#### `GET /api/posts`

- 認証: 必須
- クエリ:
  - `cursor?: string`
  - `limit?: number`
  - `category_path?: string`
  - `tag?: string`
  - `author_id?: string`
  - `status?: string`
  - `author_scope?: "all" | "me"`
- ルール:
  - 既定では閲覧可能な `published` 記事のみ返す
  - `status=wip` は `author_scope=me` の場合のみ指定可能

#### `POST /api/posts`

認証: 必須

```json
{
  "title": "2way-sql 導入メモ",
  "body_md": "# summary",
  "category_path": "engineering/backend",
  "tags": ["postgresql", "2way-sql"],
  "visibility_scope": "public",
  "visible_group_id": null,
  "status": "wip",
  "change_summary": "initial draft"
}
```

レスポンス:

```json
{
  "data": {
    "post_id": "uuid",
    "revision_number": 1,
    "status": "wip",
    "visibility_scope": "public"
  }
}
```

#### `GET /api/posts/:postId`

- 認証: 必須
- 出力: `PostDetail`
- エラー:
  - 権限がない場合は `404 not_found`

#### `POST /api/posts/:postId/revisions`

認証: 作成者必須

```json
{
  "expected_revision_number": 3,
  "title": "2way-sql 導入メモ",
  "body_md": "# updated",
  "category_path": "engineering/backend",
  "tags": ["postgresql", "2way-sql"],
  "visibility_scope": "group",
  "visible_group_id": "uuid",
  "status": "published",
  "change_summary": "publish to backend group"
}
```

ルール:

- `expected_revision_number` が現在値と一致しない場合は `409 conflict`
- `published -> wip` は `409 conflict`
- 初回 `wip -> published` のみ `article_published` 通知を作る

#### `GET /api/posts/:postId/revisions`

- 認証: 対象記事の閲覧権限必須
- 出力:
  - `revision_number`
  - `editor`
  - `status`
  - `visibility_scope`
  - `change_summary`
  - `created_at`

#### `GET /api/posts/:postId/revisions/compare`

- 認証: 対象記事の閲覧権限必須
- クエリ:
  - `from: number`
  - `to: number`
- 出力:
  - `from_revision`
  - `to_revision`
  - `diff_html`

### 7.5 コメント

#### `GET /api/posts/:postId/comments`

- 認証: 対象記事の閲覧権限必須
- 出力: `Comment[]`

#### `POST /api/posts/:postId/comments`

認証: 対象記事へのコメント権限必須

```json
{
  "body_md": "@soudai ここを確認してください"
}
```

ルール:

- `wip` 記事には投稿できない
- 保存後にメンション抽出を行う
- 記事作成者が自分以外なら `comment_created` 通知を作る

#### `POST /api/comments/:commentId/revisions`

認証: コメント投稿者必須

```json
{
  "expected_revision_number": 1,
  "body_md": "updated comment"
}
```

#### `POST /api/comments/:commentId/archive`

- 認証: コメント投稿者または記事作成者
- 成功: `204 No Content`

### 7.6 Star

#### `POST /api/posts/:postId/stars`

- 認証: 対象記事への Star 権限必須
- 成功: `204 No Content`
- ルール:
  - 同一ユーザーの重複 Star は no-op

#### `DELETE /api/posts/:postId/stars`

- 認証: 対象記事への Star 権限必須
- 成功: `204 No Content`

### 7.7 カテゴリ・タグ・検索

#### `GET /api/categories`

- 認証: 必須
- 出力:
  - `path`
  - `post_count`

#### `GET /api/categories/:path/posts`

- 認証: 必須
- 返却対象: 閲覧可能な `published` 記事のみ

#### `GET /api/tags`

- 認証: 必須
- 出力:
  - `tag`
  - `post_count`

#### `GET /api/tags/:tag/posts`

- 認証: 必須
- 返却対象: 閲覧可能な `published` 記事のみ

#### `GET /api/search/posts`

- 認証: 必須
- クエリ:
  - `q?: string`
  - `category_path?: string`
  - `tag?: string`
  - `cursor?: string`
  - `limit?: number`
- 返却対象:
  - 閲覧可能な `published` 記事のみ

### 7.8 通知

#### `GET /api/notifications`

- 認証: 必須
- クエリ:
  - `cursor?: string`
  - `limit?: number`
  - `unread_only?: boolean`
- 出力: `Notification[]`

#### `POST /api/notifications/read`

```json
{
  "notification_ids": ["uuid-1", "uuid-2"]
}
```

または:

```json
{
  "all": true
}
```

- 成功: `204 No Content`

#### `GET /api/notifications/unread-count`

- 認証: 必須
- 出力:

```json
{
  "data": {
    "count": 3
  }
}
```

### 7.9 管理

#### `GET /api/admin/settings/slack`

- 認証: 管理者必須
- 出力:
  - `enabled`
  - `workspace_id`
  - `announcement_channel_id`
  - `updated_at`

#### `PATCH /api/admin/settings/slack`

認証: 管理者必須

```json
{
  "enabled": true,
  "announcement_channel_id": "C12345678"
}
```

#### `GET /api/admin/system/init-status`

- 認証: 管理者必須
- 出力:
  - `admin_group_exists`
  - `bootstrap_admin_exists`
  - `bootstrap_admin_email`
  - `initialized_at`

#### `POST /api/admin/init/bootstrap-admin`

認証: 管理者必須

```json
{
  "email": "admin@example.com",
  "display_name": "Initial Admin"
}
```

- ルール:
  - 既存なら no-op で成功
  - `make init` と同じ業務ロジックを使う

## 8. 通知受信者ルール

| イベント | 受信者 | Slack 配信 |
|---|---|---|
| `article_published` かつ `public` | 公開時点の全アクティブユーザーから投稿者を除いた集合 | announcement channel |
| `article_published` かつ `group` | 対象グループの現行メンバーから投稿者を除いた集合 | Slack DM のみ |
| `comment_created` | 記事作成者。ただし投稿者自身は除外 | Slack DM |
| `mention_created` | メンション対象ユーザー。ただし投稿者自身は除外 | Slack DM |

## 9. 実装上の補足

- 記事本文とコメント本文は Markdown を正本として保存し、HTML はレスポンス生成時または read model で導出する
- 検索は current revision/current tags に対して行う
- 一覧と詳細の権限制御は別実装にせず、同一 SQL 条件を共有する
- Slack 通知は同期 API から切り離せるように、通知作成と配信試行を分離する

## 10. 実装前に確定すべき不足情報と提案

本節は、実装着手時に解釈ぶれが起きやすい未確定項目を明示し、MVP を前進させるための推奨デフォルトを定義する。

### 10.1 認証・セキュリティ

| 項目 | 現状の不足 | 提案（MVP 既定値） |
|---|---|---|
| セッション Cookie 属性 | `Secure` / `HttpOnly` / `SameSite` の明示がない | `HttpOnly=true`, `Secure=true`, `SameSite=Lax`, `Path=/`, `Max-Age=7d` |
| CSRF 実装方式 | 「CSRF 対策を行う」のみで手段未定 | Double Submit Cookie 方式を採用し、状態変更 API で `X-CSRF-Token` ヘッダー必須 |
| Google OAuth state 検証失敗時の扱い | エラー仕様未定 | `400 validation_error` を返し、セッション未発行でログイン画面へ遷移 |

### 10.2 API 契約

| 項目 | 現状の不足 | 提案（MVP 既定値） |
|---|---|---|
| `GET` 系の `meta` 形式 | cursor pagination の `meta` 仕様が未記載 | `{ "next_cursor": string|null, "has_next": boolean, "limit": number }` を共通採用 |
| `PATCH` / `POST` のバリデーション境界 | 文字数・件数上限が未定 | `title<=200`, `change_summary<=500`, `body_md<=200000`, `tags<=20`, `tag<=50` |
| エラー `details` の構造 | フィールドエラー形式が未定 | `validation_error` 時は `details.fields.<field>=[]string` を返す |

### 10.3 コンテンツ仕様

| 項目 | 現状の不足 | 提案（MVP 既定値） |
|---|---|---|
| カテゴリパスの妥当性 | 記法規則が未定 | `^[a-z0-9]+([-/][a-z0-9]+)*$` を採用、保存時は小文字正規化 |
| タグ正規化 | 大小文字・空白の統一ルールが曖昧 | 前後空白除去 + 小文字化 + 重複除去して保存 |
| Markdown の危険要素 | HTML サニタイズ方針が未定 | `body_html` はサーバー側で XSS サニタイズ済み HTML のみ返却 |

### 10.4 通知・Slack 連携

| 項目 | 現状の不足 | 提案（MVP 既定値） |
|---|---|---|
| Slack 送信リトライ | 失敗時リトライ回数・間隔が未定 | 最大 3 回、指数バックオフ（30s, 2m, 10m） |
| `notification_delivery_attempts.status=skipped` の条件 | 判定条件が未定 | `slack disabled` / `recipient not linked` / `access revoked` を `skipped` として記録 |
| 公開通知の受信者スナップショット | 「公開時点」の厳密定義が未定 | publish revision の commit 時点を基準に recipient を確定し保存 |

### 10.5 運用・監査

| 項目 | 現状の不足 | 提案（MVP 既定値） |
|---|---|---|
| 監査ログ保持期間 | 保持期間ポリシーが未定 | `notifications` / `delivery attempts` / `bootstrap events` は最低 1 年保持 |
| 時刻同期 | `occurred_at` の生成責務が未定 | DB サーバー時刻（`now()`）を正とし、アプリ時刻で上書きしない |
| `make init` 失敗時の再実行指針 | 部分失敗時の扱いが未定 | トランザクション単位で実行し、失敗時はロールバック後に再実行可能とする |

### 10.6 追加確認が必要な論点（次アクション）

- 検索エンジンは PostgreSQL FTS のみで開始するか、将来の外部検索基盤（OpenSearch 等）を前提に抽象化するか。
- 権限喪失済み通知の UI 表示文言（`access_revoked=true`）をプロダクト文言として確定するか。
- Slack DM 送信でユーザーが bot を許可していない場合の運用フロー（再連携導線、ヘルプ表示）を決めるか。

## 11. 実装タスク（Phase 0）反映内容

本 PR では、Phase 0（Issue 1 相当）として「実装開始前に固定すべき共通ルール」を仕様へ反映済みとする。

### 11.1 対応済みタスク

- セッション Cookie 属性の既定値を固定（`HttpOnly=true`, `Secure=true`, `SameSite=Lax`, `Max-Age=7d`）
- CSRF 方式を Double Submit Cookie + `X-CSRF-Token` 必須に固定
- cursor pagination 共通 `meta` 形式を固定（`next_cursor`, `has_next`, `limit`）
- `validation_error` の `details.fields` 構造を固定
- 入力バリデーション上限（title, body, tags 等）を固定
- カテゴリ/タグの正規化と Markdown サニタイズ方針を固定

### 11.2 Phase 1 への引き継ぎ条件

- 認証実装時は、`GET /api/auth/google/callback` の state 不正時挙動を `400 validation_error` で統一する
- セッション関連 API は 10.1 の Cookie/CSRF 既定値に必ず準拠する
- API 追加時は 10.2 の `meta` / `error.details` 形式を再利用する
