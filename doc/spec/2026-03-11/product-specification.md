# 社内ナレッジ共有サービス 仕様書

Status: Draft
Date: 2026-03-11

## 1. 目的

本書は、esa ライクな社内向けブログ / 社内 SNS サービスの実装対象を定義するための仕様書である。

本サービスは以下を目的とする。

- 社内ドキュメント、ノウハウ、日報、設計メモを Markdown で継続的に共有できること
- WIP 段階では個人作業を保護し、公開後は組織内で再利用できること
- コメント、メンション、Star、通知により知識共有の循環を作ること
- Google 認証、グループ制御、Slack 通知により社内運用へ組み込みやすいこと

## 2. 参照ドキュメント

- [ADR Bundle](/home/soudai/takt/saga-blog/doc/adr/2026-03-11/README.md)
- [ADR 001](/home/soudai/takt/saga-blog/doc/adr/2026-03-11/001-application-architecture-baseline.md)
- [ADR 002](/home/soudai/takt/saga-blog/doc/adr/2026-03-11/002-content-visibility-and-group-administration.md)
- [ADR 003](/home/soudai/takt/saga-blog/doc/adr/2026-03-11/003-bootstrap-admin-and-slack-delivery.md)

## 3. スコープ

### 3.1 MVP 対象

- Google ログイン
- 初期管理者 bootstrap
- アプリ内ユーザー管理
- 管理者グループと一般グループ管理
- 記事作成、編集、公開、閲覧
- `WIP` と `公開` の状態管理
- `公開` と `グループ内のみ` の閲覧範囲管理
- Markdown 編集とプレビュー
- カテゴリ、タグ
- コメント、メンション、Star
- 記事検索
- 更新履歴
- アプリ内通知
- Slack bot 通知

### 3.2 将来拡張

- テンプレート機能
- Watch / Follow
- メンバー activity feed
- 外部公開 API
- ストック（個人クリップ）
- 添付ファイル管理
- ピン留め表示（管理者）
- Webhook 連携
- ユーザー招待・サインアップ管理

## 4. プロダクト前提

### 4.1 技術前提

- フロントエンドは `TypeScript + React + Vite`
- UI コンポーネントは `shadcn/ui`
- バックエンドは `Python 3.12 + FastAPI`
- DB は `PostgreSQL 18`
- DB アクセスは `2way-sql`
- マイグレーションは `golang-migrate`
- ローカル実行基盤は `Podman / podman compose`
- 永続化は追記型のイミュータブル設計とし、現在状態は事実から導出する

### 4.2 用語

- `WIP`: 作成途中の記事。作成者のみ閲覧可能
- `公開`: 他ユーザーが閲覧可能な記事状態
- `公開範囲=公開`: 認証済み全ユーザーが閲覧可能
- `公開範囲=グループ内のみ`: 指定グループの所属ユーザーのみ閲覧可能
- `管理者グループ`: グループ管理権限を持つ特別なグループ

## 5. ユーザーロール

| ロール | 説明 |
|---|---|
| 未ログインユーザー | 本システムでは通常利用対象外 |
| 一般ユーザー | 記事投稿、閲覧、コメント、Star、通知受信が可能 |
| 記事作成者 | 自分の記事の WIP 作成、公開、更新が可能 |
| グループメンバー | グループ限定公開記事を閲覧可能 |
| 管理者グループ所属ユーザー | グループ作成、編集、メンバー管理、管理系設定変更が可能 |

## 6. 機能仕様

### 6.1 認証・アカウント

- Google OAuth によるログインを提供する
- ログイン成功時にアプリ内ユーザーと Google identity を紐付ける
- 初回ログイン時に既存の bootstrap admin email と一致する場合は、その管理者アカウントへ連携する
- ユーザーはプロフィール情報を保持できる
- ユーザーは Slack identity を連携できる

### 6.2 初期セットアップ

- `make init` を初期化エントリポイントとする
- `make init` は管理者グループを作成する
- `make init` は `ADMIN_EMAIL` を必須引数として受け付ける
- `make init` は初期管理者ユーザーを作成し、管理者グループへ所属させる
- `make init` は冪等でなければならない

### 6.3 グループ管理

- グループはアプリ内で管理する
- Google Workspace のグループとは連携しない
- 管理者グループ所属ユーザーのみ、グループ作成、編集、メンバー追加、メンバー削除を実行できる
- グループには少なくとも `name`, `slug`, `description` を持たせる

### 6.4 記事管理

- 記事は Markdown で作成、編集できる
- 記事はタイトル、本文、カテゴリ、タグ、状態、公開範囲を持つ
- 記事状態は `WIP` と `公開` のみとする
- 公開範囲は `公開` と `グループ内のみ` のみとする
- `WIP` 記事は作成者のみ閲覧可能とする
- `公開 + 公開` 記事は認証済みユーザー全員が閲覧可能とする
- `公開 + グループ内のみ` 記事は作成者と対象グループ所属ユーザーのみ閲覧可能とする
- 記事編集時は既存データを上書きせず、新しい revision または event を追加する
- 記事一覧、詳細、検索、通知、履歴のすべてで同じ閲覧制御を適用する

### 6.5 Markdown エディタ

- エディタは本文入力、プレビュー表示、保存、公開操作を提供する
- Markdown の見出し、リスト、コードブロック、リンク、引用、画像参照に対応する
- メンション記法を本文中またはコメント中で解釈できる

### 6.6 カテゴリ・タグ

- 記事はカテゴリを 1 つ持つ
- 記事は複数タグを持てる
- カテゴリ一覧、タグ一覧、カテゴリ別一覧、タグ別一覧を提供する
- 検索条件にカテゴリ、タグを含められる

### 6.7 コメント

- 閲覧可能な記事に対してコメント投稿できる
- コメントは投稿者、本文、作成日時、更新日時を持つ
- コメントの編集と削除は append-only の事実として扱う
- コメント投稿時、本文中のメンションを解析する

### 6.8 メンション

- メンション対象はアプリ内ユーザーとする
- メンション通知は、対象ユーザーが記事閲覧権限を持つ場合のみ作成する
- 権限がないユーザーに対するメンションは通知しない

### 6.9 Star

- ユーザーは閲覧可能な記事に Star を付与できる
- ユーザーは自分の Star を解除できる
- Star 件数を記事一覧と記事詳細で表示する

### 6.10 更新履歴

- 記事の revision を保持する
- 少なくとも `誰が`, `いつ`, `何を更新したか` を確認できる
- 必要に応じて revision 間 diff を表示する

### 6.11 検索

- 記事タイトル、本文、カテゴリ、タグを検索対象とする
- 検索結果は閲覧権限を満たす記事のみ返す
- 少なくともキーワード、カテゴリ、タグで絞り込みできる

### 6.12 通知

- 通知チャネルは `アプリ内通知` と `Slack 通知`
- 通知イベントは `記事公開`, `コメント`, `メンション`
- すべての通知イベントはまずアプリ内通知として保存する
- Slack 送信はその後に非同期または同期で実施する
- Slack 送信失敗でアプリ内通知を失敗扱いにしない
- 権限のないユーザーへ通知しない

### 6.13 Slack bot 通知

- Slack bot と Slack Web API を使って通知する
- `公開 + 公開` の記事公開は、設定された announcement channel へ通知可能とする
- `公開 + グループ内のみ` の記事公開は、既定ではチャンネル一斉通知しない
- コメント通知、メンション通知は recipient 単位で送る
- Slack identity が未連携のユーザーにはアプリ内通知のみ送る

## 7. 機能一覧

| ID | 機能名 | 優先度 | 説明 |
|---|---|---|---|
| F-01 | Google ログイン | Must | Google OAuth で認証する |
| F-02 | 初期管理者作成 | Must | `make init` で初期管理者を作成する |
| F-03 | グループ管理 | Must | 管理者グループのみがグループ管理できる |
| F-04 | 記事作成 / 編集 | Must | Markdown で記事を作成、更新する |
| F-05 | WIP 保存 | Must | WIP は作成者のみ閲覧できる |
| F-06 | 公開範囲制御 | Must | `公開` / `グループ内のみ` を制御する |
| F-07 | 記事一覧 / 詳細 | Must | 権限制御込みで表示する |
| F-08 | コメント | Must | 閲覧可能記事へコメントできる |
| F-09 | メンション | Must | コメントや本文からメンション通知を作る |
| F-10 | Star | Must | 記事へ Star 付与 / 解除できる |
| F-11 | 検索 | Must | タイトル、本文、カテゴリ、タグを検索する |
| F-12 | 更新履歴 | Must | revision と差分を参照できる |
| F-13 | アプリ内通知 | Must | 未読管理を含めて通知を表示する |
| F-14 | Slack bot 通知 | Must | 権限制御込みで Slack に通知する |
| F-15 | カテゴリ / タグ | Must | 記事分類と絞り込みに使う |
| F-16 | テンプレート | Should | 投稿テンプレートを提供する |
| F-17 | Watch | Could | 記事購読通知を提供する |

## 8. ページ一覧

| 画面ID | パス | ページ名 | 主な利用者 | 説明 |
|---|---|---|---|---|
| P-01 | `/login` | ログイン | 全ユーザー | Google ログイン開始 |
| P-02 | `/auth/google/callback` | 認証コールバック | 全ユーザー | ログイン完了処理 |
| P-03 | `/` | ホーム | ログイン済みユーザー | 最近更新、通知、導線表示 |
| P-04 | `/posts` | 記事一覧 | ログイン済みユーザー | 閲覧可能記事の一覧 |
| P-05 | `/posts/new` | 新規記事作成 | ログイン済みユーザー | 新規 WIP 作成 |
| P-06 | `/posts/:postId` | 記事詳細 | 権限を持つユーザー | 記事本文、コメント、Star |
| P-07 | `/posts/:postId/edit` | 記事編集 | 記事作成者 | revision 追加、公開操作 |
| P-08 | `/posts/:postId/revisions` | 記事履歴 | 権限を持つユーザー | revision 一覧と diff |
| P-09 | `/search` | 検索 | ログイン済みユーザー | キーワード、カテゴリ、タグ検索 |
| P-10 | `/categories/:path` | カテゴリ別一覧 | ログイン済みユーザー | カテゴリ配下の記事一覧 |
| P-11 | `/tags` | タグ一覧 | ログイン済みユーザー | タグ集計表示 |
| P-12 | `/tags/:tag` | タグ別一覧 | ログイン済みユーザー | タグ別の記事一覧 |
| P-13 | `/notifications` | 通知一覧 | ログイン済みユーザー | 通知閲覧、既読化 |
| P-14 | `/me` | マイページ | ログイン済みユーザー | 自分の投稿、プロフィール |
| P-15 | `/me/wips` | 自分の WIP 一覧 | ログイン済みユーザー | 自分だけが見られる WIP 一覧 |
| P-16 | `/settings/profile` | プロフィール設定 | ログイン済みユーザー | 表示名などの更新 |
| P-17 | `/settings/slack` | Slack 連携設定 | ログイン済みユーザー | Slack identity 連携 |
| P-18 | `/members` | メンバー一覧 | ログイン済みユーザー | ユーザー検索と一覧 |
| P-19 | `/members/:userId` | メンバー詳細 | ログイン済みユーザー | ユーザー情報と公開投稿 |
| P-20 | `/admin/groups` | グループ一覧 | 管理者 | グループ管理 |
| P-21 | `/admin/groups/new` | グループ作成 | 管理者 | 新規グループ追加 |
| P-22 | `/admin/groups/:groupId` | グループ詳細 | 管理者 | グループ更新 |
| P-23 | `/admin/groups/:groupId/members` | グループメンバー管理 | 管理者 | メンバー追加 / 削除 |
| P-24 | `/admin/integrations/slack` | Slack 管理設定 | 管理者 | Bot 設定、通知先設定 |
| P-25 | `/admin/system/init-status` | 初期化状態確認 | 管理者 | bootstrap 状態確認 |

## 9. API 一覧

### 9.1 認証・セッション

| ID | Method | Path | 説明 |
|---|---|---|---|
| A-01 | `GET` | `/api/auth/google/start` | Google OAuth 開始 |
| A-02 | `GET` | `/api/auth/google/callback` | Google OAuth コールバック |
| A-03 | `GET` | `/api/session` | 現在のログインユーザー取得 |
| A-04 | `DELETE` | `/api/session` | ログアウト |

### 9.2 自分 / ユーザー

| ID | Method | Path | 説明 |
|---|---|---|---|
| A-10 | `GET` | `/api/me` | 自分のプロフィール取得 |
| A-11 | `PATCH` | `/api/me` | 自分のプロフィール更新 |
| A-12 | `GET` | `/api/users` | ユーザー一覧取得 |
| A-13 | `GET` | `/api/users/:userId` | ユーザー詳細取得 |
| A-14 | `POST` | `/api/me/slack/link` | Slack identity 連携 |
| A-15 | `DELETE` | `/api/me/slack/link` | Slack identity 連携解除 |

### 9.3 グループ

| ID | Method | Path | 説明 |
|---|---|---|---|
| A-20 | `GET` | `/api/groups` | グループ一覧取得 |
| A-21 | `POST` | `/api/groups` | グループ作成 |
| A-22 | `GET` | `/api/groups/:groupId` | グループ詳細取得 |
| A-23 | `PATCH` | `/api/groups/:groupId` | グループ更新 |
| A-24 | `GET` | `/api/groups/:groupId/members` | メンバー一覧取得 |
| A-25 | `POST` | `/api/groups/:groupId/members` | メンバー追加 |
| A-26 | `DELETE` | `/api/groups/:groupId/members/:userId` | メンバー削除 |

### 9.4 記事

| ID | Method | Path | 説明 |
|---|---|---|---|
| A-30 | `GET` | `/api/posts` | 閲覧可能記事一覧取得 |
| A-31 | `POST` | `/api/posts` | 記事作成 |
| A-32 | `GET` | `/api/posts/:postId` | 記事詳細取得 |
| A-33 | `POST` | `/api/posts/:postId/revisions` | 記事 revision 追加 |
| A-34 | `GET` | `/api/posts/:postId/revisions` | revision 一覧取得 |
| A-35 | `GET` | `/api/posts/:postId/revisions/compare` | revision diff 取得 |

### 9.5 コメント・メンション・Star

| ID | Method | Path | 説明 |
|---|---|---|---|
| A-40 | `GET` | `/api/posts/:postId/comments` | コメント一覧取得 |
| A-41 | `POST` | `/api/posts/:postId/comments` | コメント投稿 |
| A-42 | `POST` | `/api/comments/:commentId/revisions` | コメント更新 |
| A-43 | `POST` | `/api/comments/:commentId/archive` | コメント削除相当の event 追加 |
| A-44 | `POST` | `/api/posts/:postId/stars` | Star 付与 |
| A-45 | `DELETE` | `/api/posts/:postId/stars` | Star 解除 |

### 9.6 カテゴリ・タグ・検索

| ID | Method | Path | 説明 |
|---|---|---|---|
| A-50 | `GET` | `/api/categories` | カテゴリ一覧取得 |
| A-51 | `GET` | `/api/categories/:path/posts` | カテゴリ別記事一覧 |
| A-52 | `GET` | `/api/tags` | タグ一覧取得 |
| A-53 | `GET` | `/api/tags/:tag/posts` | タグ別記事一覧 |
| A-54 | `GET` | `/api/search/posts` | 記事検索 |

### 9.7 通知

| ID | Method | Path | 説明 |
|---|---|---|---|
| A-60 | `GET` | `/api/notifications` | 通知一覧取得 |
| A-61 | `POST` | `/api/notifications/read` | 通知既読化 |
| A-62 | `GET` | `/api/notifications/unread-count` | 未読件数取得 |

### 9.8 管理

| ID | Method | Path | 説明 |
|---|---|---|---|
| A-70 | `GET` | `/api/admin/settings/slack` | Slack 管理設定取得 |
| A-71 | `PATCH` | `/api/admin/settings/slack` | Slack 管理設定更新 |
| A-72 | `GET` | `/api/admin/system/init-status` | 初期化状態取得 |
| A-73 | `POST` | `/api/admin/init/bootstrap-admin` | bootstrap 実行または再実行 |

## 10. 権限制御ルール

- すべての API と画面は認証前提とする
- `WIP` 記事は作成者のみ閲覧可能とする
- `WIP` 記事は他者への通知対象にしない
- `公開 + グループ内のみ` の記事は対象グループ所属者と作成者のみ閲覧可能とする
- コメント、Star、通知、検索、履歴は記事本体と同一の権限制御を使う
- 管理系 API と管理画面は管理者グループ所属ユーザーのみ利用可能とする

## 11. データ設計方針

- 事実を上書きせず append-only で保存する
- 現在の記事状態は最新 revision または event 群から導出する
- コメント編集や削除も event として記録する
- Star の付与 / 解除も event または append-only log として扱う
- グループメンバー追加 / 削除も履歴が残る形で保存する
- 通知配信結果も監査可能な形で記録する

## 12. 非機能要件

- 権限制御の不整合を許容しない
- DB クエリは `2way-sql` へ集約する
- マイグレーションは `golang-migrate` の SQL ベースで管理する
- `make init` は安全に再実行できる
- 主要ユースケースに対してテストを用意する
- 少なくとも以下をテスト対象とする
  - 認証
  - WIP 可視性
  - グループ限定公開
  - グループ管理権限
  - コメント / メンション
  - Star
  - 検索
  - 通知
  - 初期管理者 bootstrap

## 13. 非対象

- 未ログイン公開サイト
- Google Workspace グループとの同期
- 外部向け公開 API
- メール通知
- 公開 Webhook 提供
- Watch / テンプレートの MVP 実装

## 14. 今後の設計入力

本仕様書をもとに、次の設計ドキュメントを作成する。

- 画面遷移図
- API request / response 詳細仕様
- DB 論理設計
- 通知配信シーケンス
- 権限制御マトリクス


## 15. Knowledge 参考で追加した機能要件

`support-project/knowledge` の公開実装（画面構成・Control/API構成）を参考に、MVP 後に優先検討すべき機能を整理する。

### 15.1 追加機能要件（機能）

| ID | 機能名 | 優先度 | 追加要件 |
|---|---|---|---|
| F-18 | ストック（個人クリップ） | Should | 記事を個人の保存リストへ追加 / 解除し、後で再参照できるようにする |
| F-19 | 添付ファイル管理 | Should | 記事 / コメントに紐づく添付ファイルのアップロード、参照、権限制御を提供する |
| F-20 | 記事リアクション拡張 | Could | Star に加えて Like 系リアクションを将来拡張可能なイベント構造で保持する |
| F-21 | テンプレート管理（管理者） | Should | 管理者が投稿テンプレートを作成・更新・無効化できる |
| F-22 | ピン留め（運用告知） | Could | 重要記事を上位表示するピン留め設定を管理者が操作できる |
| F-23 | Webhook 通知 | Could | 記事公開・コメント作成などのイベントを外部システムへ Webhook 配信できる |
| F-24 | ユーザー招待 / サインアップ管理 | Could | 招待制または承認制サインアップの有効化・承認ワークフローを提供する |
| F-25 | 下書き自動保存 / 復元 | Should | 編集途中の本文を自動保存し、編集中断後に復元できる |
| F-26 | 記事アンケート | Could | 記事にアンケートを埋め込み、回答収集と集計を行える |
| F-27 | プレゼン表示 | Could | Markdown 記事をスライド形式で閲覧できる |

### 15.2 追加機能要件（UI）

| 画面ID | パス | ページ名 | 主な利用者 | 用途 |
|---|---|---|---|---|
| P-26 | `/stocks` | ストック一覧 | ログイン済みユーザー | 自分が保存した記事一覧を閲覧する |
| P-27 | `/posts/:postId/stocks` | 記事別ストック閲覧 | 記事閲覧権限を持つユーザー | 記事の保存状況を確認する（公開情報のみ） |
| P-28 | `/admin/templates` | テンプレート管理 | 管理者 | 投稿テンプレート一覧 / 作成 / 編集 |
| P-29 | `/admin/pins` | ピン留め管理 | 管理者 | ホーム上位表示の優先記事を設定する |
| P-30 | `/admin/webhooks` | Webhook 管理 | 管理者 | Webhook エンドポイント設定、署名鍵、再送制御 |
| P-31 | `/admin/users/invitations` | 招待 / 承認管理 | 管理者 | 招待発行、承認待ちユーザー処理 |
| P-32 | `/drafts` | 下書き一覧 | ログイン済みユーザー | 自動保存された編集中記事を一覧/復元する |
| P-33 | `/posts/:postId/survey` | 記事アンケート | 記事閲覧権限を持つユーザー | 記事に紐づくアンケート回答と結果表示 |
| P-34 | `/posts/:postId/presentation` | プレゼン表示 | 記事閲覧権限を持つユーザー | 記事本文をスライド表示で閲覧する |

### 15.3 追加機能要件（API）

| ID | Method | Path | 説明 |
|---|---|---|---|
| A-80 | `GET` | `/api/stocks` | 自分のストック一覧取得 |
| A-81 | `POST` | `/api/posts/:postId/stocks` | 記事をストックへ追加 |
| A-82 | `DELETE` | `/api/posts/:postId/stocks` | 記事をストックから削除 |
| A-83 | `POST` | `/api/attachments` | 添付ファイルアップロード |
| A-84 | `GET` | `/api/attachments/:attachmentId` | 添付ファイルメタデータ取得 |
| A-85 | `DELETE` | `/api/attachments/:attachmentId` | 添付ファイル削除（権限付き） |
| A-86 | `GET` | `/api/admin/templates` | テンプレート一覧取得 |
| A-87 | `POST` | `/api/admin/templates` | テンプレート作成 |
| A-88 | `PATCH` | `/api/admin/templates/:templateId` | テンプレート更新 |
| A-89 | `GET` | `/api/admin/pins` | ピン留め設定取得 |
| A-90 | `POST` | `/api/admin/pins` | ピン留め設定作成 / 更新 |
| A-91 | `GET` | `/api/admin/webhooks` | Webhook 設定一覧取得 |
| A-92 | `POST` | `/api/admin/webhooks` | Webhook 設定作成 |
| A-93 | `POST` | `/api/admin/webhooks/:webhookId/test` | Webhook テスト送信 |
| A-94 | `GET` | `/api/admin/invitations` | 招待一覧取得 |
| A-95 | `POST` | `/api/admin/invitations` | 招待作成 |
| A-96 | `GET` | `/api/drafts` | 下書き一覧取得（自分のみ） |
| A-97 | `POST` | `/api/drafts` | 下書き保存（自動保存含む） |
| A-98 | `POST` | `/api/posts/:postId/surveys/:surveyId/answers` | 記事アンケート回答 |
| A-99 | `GET` | `/api/posts/:postId/presentation` | プレゼン表示用データ取得 |

### 15.4 仕様反映時の実装ガイド

- 既存の権限制御原則（`WIP` 秘匿、`group` 公開制御）を新機能にも強制適用する。
- `stocks` / `attachments` は記事閲覧権限を前提に判定し、権限喪失後はアクセス不可とする。
- 管理系 API（templates, pins, webhooks, invitations）は管理者グループ所属ユーザー限定にする。
- Webhook は通知と同様に「作成」と「配信試行」を分離し、失敗時リトライと監査ログを保持する。
