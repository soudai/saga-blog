# 社内ナレッジ共有サービス DB 論理設計

Status: Draft
Date: 2026-03-11

## 1. 目的

本書は、PostgreSQL 18 上で本サービスを実装するための論理 DB 設計を定義する。正本は append-only な write model とし、current state は SQL で導出する。

## 2. 設計原則

- DB は `PostgreSQL 18` を使う
- マイグレーションは `golang-migrate` の SQL ファイルで管理する
- アプリケーションクエリは `2way-sql` を使う
- 既存事実を `UPDATE` で上書きしない
- 変更可能な業務データは `anchor table + revision/event table` の組で管理する
- current state は view または query で導出する
- 検索など性能要件があるものは rebuild 可能な projection を追加してよい

## 3. スキーマ構成

推奨スキーマ:

- `app_core`: ユーザー、認証、グループ
- `app_content`: 記事、コメント、タグ、Star
- `app_notify`: 通知、Slack 配信
- `app_projection`: current state view / materialized view

MVP では単一スキーマでもよいが、論理的には上記の責務で分割する。

## 4. 共通カラム規約

| カラム | 型 | 説明 |
|---|---|---|
| `*_id` | `uuid` | 主キー |
| `created_at` | `timestamptz` | 作成時刻 |
| `occurred_at` | `timestamptz` | イベント発生時刻 |
| `actor_user_id` | `uuid` | 操作主体 |
| `revision_number` | `integer` | 1 始まりの連番 |

補足:

- Email は大小文字を区別しないため `citext` の利用を推奨する
- タグも大小文字ゆれを避けるため正規化して保存する

## 5. 書き込みモデル

### 5.1 ユーザー系

#### `users`

| カラム | 型 | 制約 | 説明 |
|---|---|---|---|
| `user_id` | `uuid` | PK | ユーザー ID |
| `initial_email` | `citext` | NOT NULL | bootstrap または初回ログイン時のメール |
| `created_reason` | `text` | NOT NULL | `bootstrap` / `google_login` |
| `created_at` | `timestamptz` | NOT NULL | 作成時刻 |

用途:

- ユーザー存在の anchor
- bootstrap admin の事前作成

#### `user_profile_revisions`

| カラム | 型 | 制約 | 説明 |
|---|---|---|---|
| `user_profile_revision_id` | `uuid` | PK | revision ID |
| `user_id` | `uuid` | FK users | 対象ユーザー |
| `revision_number` | `integer` | UNIQUE(user_id, revision_number) | revision |
| `display_name` | `text` | NOT NULL | 表示名 |
| `avatar_url` | `text` | NULL | アイコン URL |
| `actor_user_id` | `uuid` | FK users | 更新者 |
| `created_at` | `timestamptz` | NOT NULL | 作成時刻 |

#### `user_identity_events`

| カラム | 型 | 制約 | 説明 |
|---|---|---|---|
| `user_identity_event_id` | `uuid` | PK | event ID |
| `user_id` | `uuid` | FK users | 対象ユーザー |
| `provider` | `text` | NOT NULL | `google` / `slack` |
| `provider_user_id` | `text` | NOT NULL | provider 側 ID |
| `provider_email` | `citext` | NULL | provider 側 email |
| `event_type` | `text` | NOT NULL | `linked` / `unlinked` |
| `actor_user_id` | `uuid` | FK users | 実行者 |
| `occurred_at` | `timestamptz` | NOT NULL | 発生時刻 |

主なインデックス:

- `(provider, provider_user_id, occurred_at desc)`
- `(user_id, provider, occurred_at desc)`

#### `sessions`

| カラム | 型 | 制約 | 説明 |
|---|---|---|---|
| `session_id` | `uuid` | PK | セッション ID |
| `user_id` | `uuid` | FK users | 対象ユーザー |
| `issued_at` | `timestamptz` | NOT NULL | 発行時刻 |
| `expires_at` | `timestamptz` | NOT NULL | 失効予定時刻 |
| `user_agent` | `text` | NULL | 監査情報 |
| `ip_address` | `inet` | NULL | 監査情報 |

#### `session_revocations`

| カラム | 型 | 制約 | 説明 |
|---|---|---|---|
| `session_revocation_id` | `uuid` | PK | revoke ID |
| `session_id` | `uuid` | FK sessions, UNIQUE | 対象セッション |
| `reason` | `text` | NULL | 失効理由 |
| `revoked_at` | `timestamptz` | NOT NULL | 失効時刻 |

### 5.2 グループ系

#### `groups`

| カラム | 型 | 制約 | 説明 |
|---|---|---|---|
| `group_id` | `uuid` | PK | グループ ID |
| `group_type` | `text` | NOT NULL | `admin` / `regular` |
| `created_by_user_id` | `uuid` | FK users | 作成者 |
| `created_at` | `timestamptz` | NOT NULL | 作成時刻 |

#### `group_revisions`

| カラム | 型 | 制約 | 説明 |
|---|---|---|---|
| `group_revision_id` | `uuid` | PK | revision ID |
| `group_id` | `uuid` | FK groups | 対象グループ |
| `revision_number` | `integer` | UNIQUE(group_id, revision_number) | revision |
| `slug` | `citext` | NOT NULL | 識別子 |
| `name` | `text` | NOT NULL | 表示名 |
| `description` | `text` | NULL | 説明 |
| `actor_user_id` | `uuid` | FK users | 更新者 |
| `created_at` | `timestamptz` | NOT NULL | 作成時刻 |

補足:

- current slug の一意性は current view を使ったアプリケーション制御で担保する

#### `group_membership_events`

| カラム | 型 | 制約 | 説明 |
|---|---|---|---|
| `group_membership_event_id` | `uuid` | PK | event ID |
| `group_id` | `uuid` | FK groups | 対象グループ |
| `user_id` | `uuid` | FK users | 対象ユーザー |
| `event_type` | `text` | NOT NULL | `member_added` / `member_removed` |
| `actor_user_id` | `uuid` | FK users | 実行者 |
| `occurred_at` | `timestamptz` | NOT NULL | 発生時刻 |

主なインデックス:

- `(group_id, user_id, occurred_at desc)`
- `(user_id, occurred_at desc)`

### 5.3 記事系

#### `articles`

| カラム | 型 | 制約 | 説明 |
|---|---|---|---|
| `article_id` | `uuid` | PK | 記事 ID |
| `created_by_user_id` | `uuid` | FK users | 作成者 |
| `created_at` | `timestamptz` | NOT NULL | 作成時刻 |

#### `article_revisions`

| カラム | 型 | 制約 | 説明 |
|---|---|---|---|
| `article_revision_id` | `uuid` | PK | revision ID |
| `article_id` | `uuid` | FK articles | 対象記事 |
| `revision_number` | `integer` | UNIQUE(article_id, revision_number) | revision |
| `editor_user_id` | `uuid` | FK users | 編集者 |
| `title` | `text` | NOT NULL | タイトル |
| `body_md` | `text` | NOT NULL | Markdown 本文 |
| `category_path` | `text` | NOT NULL | カテゴリパス |
| `status` | `text` | NOT NULL | `wip` / `published` |
| `visibility_scope` | `text` | NOT NULL | `public` / `group` |
| `visible_group_id` | `uuid` | FK groups, NULL | group 公開時の対象 |
| `change_summary` | `text` | NULL | 更新要約 |
| `created_at` | `timestamptz` | NOT NULL | 作成時刻 |

制約:

- `visibility_scope = 'group'` の場合 `visible_group_id IS NOT NULL`
- `visibility_scope = 'public'` の場合 `visible_group_id IS NULL`

主なインデックス:

- `(article_id, revision_number desc)`
- `(status, created_at desc)`
- `(visible_group_id, created_at desc)`

#### `article_revision_tags`

| カラム | 型 | 制約 | 説明 |
|---|---|---|---|
| `article_revision_id` | `uuid` | FK article_revisions | 対象 revision |
| `tag_name` | `citext` | NOT NULL | タグ名 |

主キー:

- `(article_revision_id, tag_name)`

### 5.4 コメント / Star / メンション

#### `comments`

| カラム | 型 | 制約 | 説明 |
|---|---|---|---|
| `comment_id` | `uuid` | PK | コメント ID |
| `article_id` | `uuid` | FK articles | 対象記事 |
| `created_by_user_id` | `uuid` | FK users | 投稿者 |
| `created_at` | `timestamptz` | NOT NULL | 作成時刻 |

#### `comment_revisions`

| カラム | 型 | 制約 | 説明 |
|---|---|---|---|
| `comment_revision_id` | `uuid` | PK | revision ID |
| `comment_id` | `uuid` | FK comments | 対象コメント |
| `revision_number` | `integer` | UNIQUE(comment_id, revision_number) | revision |
| `editor_user_id` | `uuid` | FK users | 編集者 |
| `body_md` | `text` | NOT NULL | Markdown 本文 |
| `created_at` | `timestamptz` | NOT NULL | 作成時刻 |

#### `comment_archive_events`

| カラム | 型 | 制約 | 説明 |
|---|---|---|---|
| `comment_archive_event_id` | `uuid` | PK | event ID |
| `comment_id` | `uuid` | FK comments, UNIQUE | 対象コメント |
| `actor_user_id` | `uuid` | FK users | 実行者 |
| `occurred_at` | `timestamptz` | NOT NULL | 発生時刻 |

#### `mention_events`

| カラム | 型 | 制約 | 説明 |
|---|---|---|---|
| `mention_event_id` | `uuid` | PK | event ID |
| `source_type` | `text` | NOT NULL | `article_revision` / `comment_revision` |
| `source_id` | `uuid` | NOT NULL | source の ID |
| `article_id` | `uuid` | FK articles | 関連記事 |
| `comment_id` | `uuid` | FK comments, NULL | 関連コメント |
| `mentioned_user_id` | `uuid` | FK users | 対象ユーザー |
| `created_by_user_id` | `uuid` | FK users | メンション実行者 |
| `created_at` | `timestamptz` | NOT NULL | 作成時刻 |

ユニーク制約:

- `(source_type, source_id, mentioned_user_id)`

#### `article_star_events`

| カラム | 型 | 制約 | 説明 |
|---|---|---|---|
| `article_star_event_id` | `uuid` | PK | event ID |
| `article_id` | `uuid` | FK articles | 対象記事 |
| `user_id` | `uuid` | FK users | 実行ユーザー |
| `event_type` | `text` | NOT NULL | `starred` / `unstarred` |
| `occurred_at` | `timestamptz` | NOT NULL | 発生時刻 |

主なインデックス:

- `(article_id, user_id, occurred_at desc)`
- `(article_id, occurred_at desc)`

### 5.5 通知系

#### `notifications`

| カラム | 型 | 制約 | 説明 |
|---|---|---|---|
| `notification_id` | `uuid` | PK | 通知 ID |
| `recipient_user_id` | `uuid` | FK users | 受信者 |
| `type` | `text` | NOT NULL | 通知種別 |
| `actor_user_id` | `uuid` | FK users, NULL | 発生元ユーザー |
| `article_id` | `uuid` | FK articles, NULL | 関連記事 |
| `comment_id` | `uuid` | FK comments, NULL | 関連コメント |
| `source_event_type` | `text` | NOT NULL | `article_revision`, `comment_revision`, `mention_event` |
| `source_event_id` | `uuid` | NOT NULL | 元イベント ID |
| `payload_json` | `jsonb` | NOT NULL | 表示用スナップショット |
| `created_at` | `timestamptz` | NOT NULL | 作成時刻 |

主なインデックス:

- `(recipient_user_id, created_at desc)`
- `(article_id, created_at desc)`

#### `notification_read_events`

| カラム | 型 | 制約 | 説明 |
|---|---|---|---|
| `notification_read_event_id` | `uuid` | PK | event ID |
| `notification_id` | `uuid` | FK notifications | 対象通知 |
| `read_by_user_id` | `uuid` | FK users | 読んだユーザー |
| `occurred_at` | `timestamptz` | NOT NULL | 既読時刻 |

主なインデックス:

- `(notification_id, occurred_at desc)`
- `(read_by_user_id, occurred_at desc)`

#### `notification_delivery_attempts`

| カラム | 型 | 制約 | 説明 |
|---|---|---|---|
| `notification_delivery_attempt_id` | `uuid` | PK | attempt ID |
| `notification_id` | `uuid` | FK notifications | 対象通知 |
| `channel` | `text` | NOT NULL | `slack_dm` / `slack_channel` |
| `destination` | `text` | NOT NULL | Slack channel ID または user ID |
| `status` | `text` | NOT NULL | `pending` / `sent` / `failed` / `skipped` |
| `provider_message_id` | `text` | NULL | 外部 message ID |
| `error_code` | `text` | NULL | エラーコード |
| `error_message` | `text` | NULL | エラー詳細 |
| `attempted_at` | `timestamptz` | NOT NULL | 実行時刻 |

#### `slack_configuration_revisions`

| カラム | 型 | 制約 | 説明 |
|---|---|---|---|
| `slack_configuration_revision_id` | `uuid` | PK | revision ID |
| `revision_number` | `integer` | UNIQUE | revision |
| `enabled` | `boolean` | NOT NULL | 通知有効フラグ |
| `workspace_id` | `text` | NULL | Slack workspace |
| `announcement_channel_id` | `text` | NULL | public publish 通知先 |
| `bot_user_id` | `text` | NULL | bot ユーザー ID |
| `bot_token_secret_ref` | `text` | NULL | secret 参照名 |
| `actor_user_id` | `uuid` | FK users | 更新者 |
| `created_at` | `timestamptz` | NOT NULL | 作成時刻 |

### 5.6 bootstrap 監査

#### `bootstrap_events`

| カラム | 型 | 制約 | 説明 |
|---|---|---|---|
| `bootstrap_event_id` | `uuid` | PK | event ID |
| `bootstrap_key` | `text` | NOT NULL | 例 `initial_admin` |
| `admin_user_id` | `uuid` | FK users | 作成された管理者 |
| `admin_group_id` | `uuid` | FK groups | 対象管理者グループ |
| `email` | `citext` | NOT NULL | bootstrap email |
| `created_at` | `timestamptz` | NOT NULL | 実行時刻 |

## 6. 読み取りモデル

### 6.1 current profile / identity

| view | 内容 |
|---|---|
| `v_current_user_profiles` | 各ユーザーの最新プロフィール |
| `v_active_google_identities` | 現在有効な Google identity |
| `v_active_slack_identities` | 現在有効な Slack identity |
| `v_active_sessions` | revoke されておらず有効期限内のセッション |

### 6.2 current group / membership

| view | 内容 |
|---|---|
| `v_current_groups` | 各グループの最新属性 |
| `v_active_group_memberships` | 最新イベントが `member_added` の所属関係 |
| `v_admin_users` | admin group に属するユーザー集合 |

### 6.3 current article / comment

| view | 内容 |
|---|---|
| `v_current_articles` | 各記事の最新 revision |
| `v_current_article_tags` | 最新 revision のタグ集合 |
| `v_current_comments` | archive されていない最新コメント |
| `v_current_star_state` | ユーザーごとの現在 Star 状態 |
| `v_article_star_counts` | 記事ごとの現在 Star 数 |

### 6.4 notification / search

| view | 内容 |
|---|---|
| `v_notification_read_state` | 通知ごとの current read 状態 |
| `v_current_notifications` | 通知に read 状態を付与した一覧 |
| `mv_article_search_documents` | current article と current tags から作る検索用 projection |

補足:

- `mv_article_search_documents` は rebuild 可能な派生データであり正本ではない
- 高頻度更新に耐えない場合は materialized view ではなく通常テーブル projection に切り替えてよい

## 7. 主要導出ロジック

### 7.1 記事 current state

各 `article_id` について最大の `revision_number` を持つ `article_revisions` を current state とする。

### 7.2 記事可視性

可視性は `v_current_articles` と `v_active_group_memberships` を使い、以下で判定する。

- `status = 'wip'` かつ `created_by_user_id = :request_user_id`
- `status = 'published'` かつ `visibility_scope = 'public'`
- `status = 'published'` かつ `visibility_scope = 'group'` かつ `visible_group_id` に対する active membership がある
- ただし記事作成者は group membership がなくても自身の記事を閲覧できる

### 7.3 Star current state

`article_id + user_id` ごとに最新の `article_star_events` を取り、その `event_type = 'starred'` のものだけを current star とする。

### 7.4 通知既読状態

`notification_id` ごとに最新の `notification_read_events` が存在すれば既読とする。

### 7.5 管理者判定

`group_type = 'admin'` の group に対する active membership があるユーザーを管理者とする。

## 8. インデックス設計

必須インデックス:

- `article_revisions(article_id, revision_number desc)`
- `article_revisions(status, visibility_scope, created_at desc)`
- `article_revision_tags(tag_name, article_revision_id)`
- `group_membership_events(group_id, user_id, occurred_at desc)`
- `comment_revisions(comment_id, revision_number desc)`
- `article_star_events(article_id, user_id, occurred_at desc)`
- `notifications(recipient_user_id, created_at desc)`
- `notification_read_events(notification_id, occurred_at desc)`
- `user_identity_events(provider, provider_user_id, occurred_at desc)`

検索用:

- `mv_article_search_documents.search_vector` に GIN index
- 必要に応じて `category_path` に btree index

## 9. 整合性ルール

- `published -> wip` の revision は許可しない
- `visibility_scope = 'group'` で存在しない group を参照してはならない
- `comment` は current article が `published` の場合のみ追加できる
- `star` は current article が `published` かつ閲覧可能な場合のみ追加できる
- `mention_event` 生成時に対象記事が閲覧不能なユーザーは通知対象にしない
- `notification` 作成時点で recipient は閲覧権限を持っていなければならない

## 10. 物理設計上の注意

- `UPDATE` を避ける設計だが、projection や materialized view の再構築は許容する
- `payload_json` には通知表示に必要な最小限のスナップショットだけ保存する
- Slack token など秘密情報は DB に平文で持たず、secret reference を保持する
- `body_md` は大きくなり得るため TOAST 前提で扱う

## 11. 初期 migration 順序

1. 拡張機能 `citext` を有効化する
2. anchor tables を作成する
3. revision / event tables を作成する
4. views を作成する
5. 検索 projection を作成する
6. 初期 admin group を作る migration は入れず、`make init` で作成する

## 12. 今後の詳細化ポイント

- `2way-sql` 用 SQL ファイル配置規約
- 検索 projection の更新戦略
- 通知 fan-out の非同期実行方式
- revision diff 生成方式
