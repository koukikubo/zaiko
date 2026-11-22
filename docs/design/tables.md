# ZAIKO テーブル定義書

- バージョン：0.2（詳細設計案）
- 対応Issue：#3〜#8、#9
- DB：PostgreSQL
- ORM：Rails 8.1 / Active Record
- 状態：レビュー前・Migration未実装
- 対象：26テーブル候補

## 1. テーブル定義の記載ルール

| 項目     | 意味                                  |
| -------- | ------------------------------------- |
| カラム名 | データベース上の物理名                |
| 日本語名 | 業務上の名称                          |
| 型       | Rails Migrationで使用する型           |
| NULL     | `不可`＝`null: false`、`可`＝NULL許可 |
| DEFAULT  | DBで設定する初期値                    |
| Option   | PK、FK、UNIQUE、INDEX、CHECKなど      |
| 設定元   | 入力・選択・Rails自動設定など         |

### 全テーブル共通の基本カラム

原則として次のカラムを設ける。

| カラム名     | 日本語名 | 型          | NULL | DEFAULT | Option    |
| ------------ | -------- | ----------- | ---- | ------- | --------- |
| `id`         | 主キー   | bigint      | 不可 | DB採番  | PK        |
| `created_at` | 作成日時 | datetime(6) | 不可 | —       | Rails管理 |
| `updated_at` | 更新日時 | datetime(6) | 不可 | —       | Rails管理 |

`audit_events` や `order_approval_events` のような追記型の履歴テーブルでは、更新日時を持たせず作成日時だけで管理する案も検討する。

日時は原則UTCで保存し、アプリ画面では店舗のタイムゾーンに変換する。

### 制約の表記

- `FK → staffs`：`staffs.id` を参照する外部キー
- `UNIQUE(a, b)`：2カラムの組み合わせが一意
- `CHECK(quantity >= 0)`：数量が0以上
- `INDEX(a, b)`：検索向けの複合インデックス
- `RESTRICT`：参照されている場合は削除を拒否する方針

複合UNIQUEやCHECKは、カラム単体ではなくテーブル単位の制約として記載する。

---

# Phase 1：認証・在庫管理

## 2. `stores`（店舗マスタ）

**責務：** 店舗の基本情報・利用状態を管理する。

**作成・更新主体：** システム管理者

| カラム名               | 日本語名             | 型          | NULL | DEFAULT    | Option | 設定元   |
| ---------------------- | -------------------- | ----------- | ---- | ---------- | ------ | -------- |
| `id`                   | 店舗ID               | bigint      | 不可 | DB採番     | PK     | DB       |
| `code`                 | 店舗コード           | string(30)  | 不可 | —          | UNIQUE | 管理者   |
| `name`                 | 店舗名               | string(150) | 不可 | —          | —      | 管理者   |
| `timezone`             | タイムゾーン         | string(50)  | 不可 | Asia/Tokyo | —      | システム |
| `active`               | 利用状態             | boolean     | 不可 | true       | —      | 管理者   |
| `join_code_digest`     | 参加コードのハッシュ | string(255) | 可   | NULL       | —      | Rails    |
| `join_code_expires_at` | 参加コードの期限     | datetime(6) | 可   | NULL       | —      | Rails    |
| `join_code_enabled`    | 自己登録の許可       | boolean     | 不可 | false      | —      | 管理者   |
| `created_at`           | 作成日時             | datetime(6) | 不可 | —          | —      | Rails    |
| `updated_at`           | 更新日時             | datetime(6) | 不可 | —          | —      | Rails    |

### DB制約・INDEX

- `PRIMARY KEY(id)`
- `UNIQUE(code)`
- `CHECK(btrim(code) <> '')`
- `CHECK(btrim(name) <> '')`
- `CHECK(join_code_enabled = false OR (join_code_digest IS NOT NULL AND join_code_expires_at IS NOT NULL))`

### 業務ルール

- 初期段階は1店舗で運用する。
- 将来の複数店舗運用を想定して店舗IDを保持する。
- 利用停止した店舗のスタッフはAPIを利用できない。
- スタッフ・食材・棚卸履歴が存在する店舗は原則物理削除しない。
- タイムゾーンはIANA識別子を使用する。
- 店舗コードの大文字・小文字は登録前に正規化する方針とする。

### 関連

- `has_many :staffs`
- `has_many :positions`
- `has_many :items`
- `has_many :stock_locations`

### Phase 1の店舗運用方針

- Phase 1で運用する店舗は1店舗のみとする。
- 店舗はシステム初期設定時に登録し、一般利用者は作成できない。
- 新規店舗登録API・店舗選択画面はPhase 1では実装しない。
- 初期管理者はシステム導入時に安全な初期設定手順で登録する。
- 一般スタッフは既存店舗へ自己登録する。
- スタッフの新規登録時、所属店舗はRails側で決定する。
- クライアントから任意のstore_idを受け付けない。
- stores.idと各テーブルのstore_idは将来の複数店舗対応のため維持する。
- 将来の複数店舗対応では、店舗の識別・管理・所属ルールを拡張する。

### 店舗参加コード

- 既存店舗のスタッフ自己登録に使用する。
- 個別招待メール・管理者の個別承認は不要とする。
- 参加コードは十分な乱数を使って生成する。
- 原文は保存せず、安全なハッシュ値を保存する。
- 有効期限・無効化・再生成に対応する。
- 不正な試行を防ぐためレート制限を設ける。
- 店舗コードと参加コードは別の情報として扱う。

## 3. `staffs`（スタッフマスタ）

**責務：** スタッフの所属店舗、基本権限、在籍状態を管理する。

**作成主体：**

- 一般スタッフ：既存店舗への自己登録処理
- 初期管理者：システム導入時の初期設定処理
- その他：管理権限を持つスタッフ

**更新主体：**

- 本人：許可されたプロフィール項目
- 店舗管理者：基本権限・利用状態・担当管理
- Rails：認証関連のシステム管理項目

**自己登録時のルール：**

- 既存の1店舗にのみ登録する。
- 所属店舗はRails側で決定する。
- `code` はRailsが店舗内で一意に生成する。
- `role_code` は必ず `staff` とする。
- `status` は `active` とする。
- 一般利用者が `admin` や `manager` を指定することは禁止する。
- Google認証と店舗参加コードの検証後に登録する。
- スタッフ作成とOAuth紐付けは同一トランザクションで行う。

| カラム名       | 日本語名           | 型          | NULL | DEFAULT | Option      | 設定元 |
| -------------- | ------------------ | ----------- | ---- | ------- | ----------- | ------ |
| `id`           | スタッフID         | bigint      | 不可 | DB採番  | PK          | DB     |
| `store_id`     | 所属店舗ID         | bigint      | 不可 | —       | FK → stores | Rails  |
| `code`         | スタッフコード     | string(30)  | 不可 | —       | 複合UNIQUE  | Rails  |
| `name`         | スタッフ名         | string(100) | 不可 | —       | —           | 管理者 |
| `role_code`    | 基本権限           | string(30)  | 不可 | staff   | CHECK       | 管理者 |
| `status`       | 利用状態           | string(30)  | 不可 | active  | CHECK       | 管理者 |
| `retired_at`   | 退職日時           | datetime(6) | 可   | NULL    | —           | Rails  |
| `lock_version` | 更新バージョン     | integer     | 不可 | 0       | 楽観ロック  | Rails  |
| `auth_version` | 認証失効バージョン | integer     | 不可 | 0       | CHECK       | Rails  |
| `created_at`   | 作成日時           | datetime(6) | 不可 | —       | —           | Rails  |
| `updated_at`   | 更新日時           | datetime(6) | 不可 | —       | —           | Rails  |

### DB制約・INDEX

- `PRIMARY KEY(id)`
- `FOREIGN KEY(store_id) REFERENCES stores(id)`
- `UNIQUE(store_id, code)`
- `INDEX(store_id, status)`
- `CHECK(btrim(code) <> '')`
- `CHECK(btrim(name) <> '')`
- `CHECK(role_code IN ('staff', 'manager', 'admin'))`
- `CHECK(status IN ('active', 'suspended', 'retired'))`
- `CHECK(status <> 'retired' OR retired_at IS NOT NULL)`
- `CHECK(auth_version >= 0)`
- `UNIQUE(store_id, id)`：店舗一致を検証する複合FKの参照先

### 基本権限

| role_code | 日本語名     | 主な役割                     |
| --------- | ------------ | ---------------------------- |
| `staff`   | 一般スタッフ | 担当業務・在庫確認           |
| `manager` | 管理スタッフ | スタッフ管理・業務管理の一部 |
| `admin`   | 店舗管理者   | 店舗内の管理操作             |

具体的な操作権限は後続の認可設計で確定する。

### 利用状態

| status      | 日本語名 | API利用          |
| ----------- | -------- | ---------------- |
| `active`    | 有効     | 権限に応じて許可 |
| `suspended` | 利用停止 | 拒否             |
| `retired`   | 退職     | 拒否             |

### 業務ルール

- スタッフは必ず1店舗に所属する。
- スタッフコードは同じ店舗内で一意とする。
- 異なる店舗では同じスタッフコードを使用できる。
- 退職時はstatusをretiredに更新し、retired_atを記録する。
- 退職しても棚卸・発注・監査の履歴を保持する。
- 利用停止・退職済みスタッフのAPIアクセスを拒否する。
- Google認証成功だけでは利用を許可せず、スタッフの有効状態を検証する。
- 基本権限とポジション別発注承認権限を混同しない。
- スタッフの更新競合はlock_versionで検出する。
- 再雇用・復職時の過去の退職日時の扱いは、監査履歴と合わせて別途確定する。

### 関連

- `belongs_to :store`
- `has_many :oauth_identities`
- `has_many :position_assignments`
- `has_many :inventory_counts`（作成者・確定者の関連を区別）
- `has_many :inventory_count_lines`（棚卸実施者・入力者の関連を区別）

## 4. `oauth_identities`（外部認証アカウント）

**責務：** GoogleアカウントとZAIKOのスタッフを安全に紐付ける。

**作成・更新主体：** Railsの認証・アカウント紐付け処理

**実装フェーズ：** Phase 1-A

| カラム名         | 日本語名               | 型          | NULL | DEFAULT | Option      | 設定元          |
| ---------------- | ---------------------- | ----------- | ---- | ------- | ----------- | --------------- |
| `id`             | 外部認証ID             | bigint      | 不可 | DB採番  | PK          | DB              |
| `staff_id`       | スタッフID             | bigint      | 不可 | —       | FK → staffs | Rails           |
| `provider`       | 認証プロバイダー       | string(30)  | 不可 | google  | CHECK       | Rails           |
| `issuer`         | トークン発行元         | string(255) | 不可 | —       | 複合UNIQUE  | Google ID Token |
| `subject`        | Googleユーザー識別子   | string(255) | 不可 | —       | 複合UNIQUE  | Google ID Token |
| `email_snapshot` | 紐付け時メールアドレス | string(255) | 可   | NULL    | —           | Google ID Token |
| `linked_at`      | 紐付け日時             | datetime(6) | 不可 | —       | —           | Rails           |
| `created_at`     | 作成日時               | datetime(6) | 不可 | —       | —           | Rails           |
| `updated_at`     | 更新日時               | datetime(6) | 不可 | —       | —           | Rails           |

### DB制約・INDEX

- `PRIMARY KEY(id)`
- `FOREIGN KEY(staff_id) REFERENCES staffs(id)`
- `UNIQUE(issuer, subject)`
- `UNIQUE(staff_id, provider)`
- `CHECK(provider = 'google')`
- `CHECK(btrim(issuer) <> '')`
- `CHECK(btrim(subject) <> '')`

### 一意性の考え方

`UNIQUE(issuer, subject)` により、同じGoogleアカウントを複数スタッフへ登録できないようにする。

`UNIQUE(staff_id, provider)` により、1スタッフに同じプロバイダーのアカウントを複数登録できないようにする。

将来、複数のGoogleアカウントを1スタッフが利用できる仕様にする場合は、後者の制約を見直す。

### Googleアカウントの識別

- Googleの識別子には検証済みID Tokenの `iss` と `sub` を使用する。
- Googleが許可するissuer表記を検証後、DB登録時には `https://accounts.google.com` に正規化する。
- `email_snapshot` は表示・監査の補助情報であり、ログイン時の識別には使用しない。
- メールアドレスが一致しただけでは、既存スタッフに自動紐付けしない。

### 初回紐付けの原則

Phase 1では、既存の1店舗へスタッフ本人が自己登録する。

1. スタッフ本人がGoogle認証を行う。
2. RailsがGoogle ID Tokenを検証する。
3. 検証済みのissuer・subjectで登録済みアカウントを検索する。
4. 未登録の場合、スタッフ名と店舗参加コードを受け付ける。
5. Railsが対象の既存店舗と参加コードを検証する。
6. 同一DBトランザクションでstaffsとoauth_identitiesを作成する。
7. 登録成功後、ZAIKO API用の認証情報を発行する。
8. 一般スタッフとしてログイン状態へ遷移する。

**制約：**

- 個別招待メールは使用しない。
- 管理者による個別承認は不要とする。
- Google認証だけで既存店舗に参加できない。
- 参加コードを知らない利用者は登録できない。
- 管理者権限を自己登録で取得できない。
- メールアドレス一致による自動紐付けは禁止する。
- 登録済みのGoogleアカウントは新規作成せず通常ログインする。
- 紐付け時には監査イベントを記録する。

### セキュリティ要件

- Google ID Tokenの署名を検証する。
- issuer、audience、有効期限を検証する。
- state・nonce・PKCEを認証フローに応じて検証する。
- 未登録スタッフの利用を禁止する。
- suspended・retiredのスタッフの利用を禁止する。
- 所属店舗が無効化されている場合も利用を禁止する。
- Googleのアクセストークン・リフレッシュトークンを本テーブルに保存しない。
- アカウント紐付け・変更・解除は監査対象とする。

### 削除・変更方針

- スタッフ退職時にGoogleアカウントの紐付け履歴を無条件に削除しない。
- アカウント変更時は管理者の承認と監査記録を必要とする。
- 認証情報の変更・解除後に古い認証情報が再利用されないことを保証する。
- 再紐付けの方法はIssue #4で確定する。

### アソシエーション

- `belongs_to :staff`
- `staffs` 側は `has_many :oauth_identities` を基本とする。
- 現行の1プロバイダー1アカウント制約はDBのUNIQUEで保証する。

### Phase 3との境界

- `auth_sessions` はPhase 3で実装する。
- Phase 1のGoogle認証とAPI認可は省略しない。
- 端末別の強制ログアウトはPhase 3で実装する。

## 5. `positions`（ポジションマスタ）

**責務：** 揚場・焼場・造り場などの担当区分を管理する。

**作成・更新主体：** 店舗管理者

**実装フェーズ：** Phase 1

| カラム名       | 日本語名         | 型          | NULL | DEFAULT | Option      | 設定元 |
| -------------- | ---------------- | ----------- | ---- | ------- | ----------- | ------ |
| `id`           | ポジションID     | bigint      | 不可 | DB採番  | PK          | DB     |
| `store_id`     | 店舗ID           | bigint      | 不可 | —       | FK → stores | Rails  |
| `code`         | ポジションコード | string(30)  | 不可 | —       | 複合UNIQUE  | 管理者 |
| `name`         | ポジション名     | string(100) | 不可 | —       | —           | 管理者 |
| `sort_order`   | 表示順           | integer     | 不可 | 0       | CHECK       | 管理者 |
| `active`       | 有効状態         | boolean     | 不可 | true    | —           | 管理者 |
| `lock_version` | 更新バージョン   | integer     | 不可 | 0       | 楽観ロック  | Rails  |
| `created_at`   | 作成日時         | datetime(6) | 不可 | —       | —           | Rails  |
| `updated_at`   | 更新日時         | datetime(6) | 不可 | —       | —           | Rails  |

### DB制約・INDEX

- `PRIMARY KEY(id)`
- `FOREIGN KEY(store_id) REFERENCES stores(id)`
- `UNIQUE(store_id, code)`
- `UNIQUE(store_id, id)`：店舗一致を保証する複合FKの参照先
- `CHECK(btrim(code) <> '')`
- `CHECK(btrim(name) <> '')`
- `CHECK(sort_order >= 0)`
- `INDEX(store_id, active, sort_order)`

### 業務ルール

- ポジションは必ず1店舗に所属する。
- 同一店舗内のポジションコードは重複させない。
- 異なる店舗では同じポジションコードを使用できる。
- 1つのポジションには複数スタッフを割り当てられる。
- 1人のスタッフが複数ポジションを兼務できる。
- 過去の担当履歴があるポジションは原則物理削除しない。
- `active = false` のポジションには新規担当を割り当てない。
- 無効化しても過去の担当履歴は保持する。
- ポジションの無効化に伴う既存担当の終了処理はServiceで管理する。
- ポジションの変更競合は `lock_version` で検出する。
- 担当ポジションと発注承認権限は別の情報として扱う。

### アソシエーション

- `belongs_to :store`
- `has_many :position_assignments`
- `has_many :staffs, through: :position_assignments`
- `has_many :position_items`

※履歴があるため、スタッフ取得時には担当期間を条件に含める。

## 6. `position_assignments`（ポジション担当履歴）

**責務：** スタッフが担当したポジションと、その担当期間を記録する。

**作成・更新主体：** 店舗管理者

**実装フェーズ：** Phase 1

| カラム名         | 日本語名       | 型          | NULL | DEFAULT | Option             | 設定元 |
| ---------------- | -------------- | ----------- | ---- | ------- | ------------------ | ------ |
| `id`             | 担当履歴ID     | bigint      | 不可 | DB採番  | PK                 | DB     |
| `store_id`       | 店舗ID         | bigint      | 不可 | —       | FK → stores        | Rails  |
| `staff_id`       | スタッフID     | bigint      | 不可 | —       | 複合FK → staffs    | 管理者 |
| `position_id`    | ポジションID   | bigint      | 不可 | —       | 複合FK → positions | 管理者 |
| `started_on`     | 担当開始日     | date        | 不可 | —       | CHECK・EXCLUDE     | 管理者 |
| `ended_on`       | 担当終了日     | date        | 可   | NULL    | CHECK・EXCLUDE     | 管理者 |
| `assigned_by_id` | 登録スタッフID | bigint      | 可   | NULL    | 複合FK → staffs    | Rails  |
| `lock_version`   | 更新バージョン | integer     | 不可 | 0       | 楽観ロック         | Rails  |
| `created_at`     | 作成日時       | datetime(6) | 不可 | —       | —                  | Rails  |
| `updated_at`     | 更新日時       | datetime(6) | 不可 | —       | —                  | Rails  |

### DB制約・INDEX

- `PRIMARY KEY(id)`
- `FOREIGN KEY(store_id) REFERENCES stores(id)`
- `FOREIGN KEY(store_id, staff_id) REFERENCES staffs(store_id, id)`
- `FOREIGN KEY(store_id, position_id) REFERENCES positions(store_id, id)`
- `FOREIGN KEY(store_id, assigned_by_id) REFERENCES staffs(store_id, id)`
- `CHECK(ended_on IS NULL OR ended_on >= started_on)`
- `INDEX(store_id, started_on)`
- `INDEX(staff_id, started_on)`

複合FKの参照先となる `staffs(store_id, id)` と
`positions(store_id, id)` には、
対応するUNIQUE制約を定義する。

### 担当期間のルール

- `started_on` は担当開始日とする。
- `ended_on` は担当終了日とし、その日も担当期間に含む。
- `ended_on = NULL` は終了日未定を意味する。
- 1人のスタッフが同時期に複数ポジションを担当することを許可する。
- 1つのポジションを同時期に複数スタッフが担当することを許可する。
- 同一スタッフ・同一ポジションで期間が重複する登録は禁止する。
- 担当期間が終了した後は、同じポジションに再び担当登録できる。
- 担当履歴は原則物理削除せず、期間を修正する場合は監査履歴を残す。

### 担当期間の重複防止

PostgreSQLの `daterange` と `EXCLUDE` 制約を使用する。

判定対象：

- `staff_id`
- `position_id`
- `daterange(started_on, ended_on, '[]')`

同一スタッフ・同一ポジションの期間が重なる場合は、
DB制約で登録・更新を拒否する。

実装時には `btree_gist` 拡張とGiSTインデックスを使用する。
Migrationの具体的な記述は実装Issueで確定する。

### 店舗境界

- スタッフ・ポジション・登録者は同一店舗に所属すること。
- `store_id` は対象スタッフの店舗からRails側で設定する。
- クライアントから指定された店舗IDは信用しない。
- Railsの認可に加えて、複合FKで店舗間の不正な関連を防ぐ。
- 別店舗への異動は単純な `store_id` 更新では扱わず、スタッフ所属設計と合わせて別途検討する。

### 無効化・退職時の扱い

- 退職済みスタッフへの新規担当登録は禁止する。
- 無効化されたポジションへの新規担当登録は禁止する。
- 退職後も過去の担当履歴は保持する。
- 退職時に未終了の担当期間を終了する処理をServiceで行う。
- 登録者IDは通常Railsで設定する。既存データの移行等は例外としてNULLを許可する。

### 代理棚卸

- 担当ポジションと実際の棚卸実施者は別に管理する。
- `position_assignments` の存在だけで代理棚卸権限を与えない。
- 代理棚卸を許可する条件は認可設計で判断する。
- 実際に数えたスタッフと日時は `inventory_count_lines` に記録する。

### アソシエーション

- `belongs_to :store`
- `belongs_to :staff`
- `belongs_to :position`
- `belongs_to :assigned_by, class_name: "Staff", optional: true`

スタッフ側：

- `has_many :position_assignments`
- `has_many :positions, through: :position_assignments`

ポジション側：

- `has_many :position_assignments`
- `has_many :staffs, through: :position_assignments`

※Railsの関連定義とDBの複合FKは別途設定する。

## 7. `items`（食材・商品マスタ）

**責務：** 店舗で使用する食材・商品の基本情報と在庫基準単位を管理する。

**作成・更新主体：** 店舗管理者

**実装フェーズ：** Phase 1

| カラム名       | 日本語名       | 型          | NULL | DEFAULT | Option      | 設定元 |
| -------------- | -------------- | ----------- | ---- | ------- | ----------- | ------ |
| `id`           | 食材ID         | bigint      | 不可 | DB採番  | PK          | DB     |
| `store_id`     | 店舗ID         | bigint      | 不可 | —       | FK → stores | Rails  |
| `code`         | 食材コード     | string(50)  | 不可 | —       | 複合UNIQUE  | 管理者 |
| `name`         | 食材名         | string(150) | 不可 | —       | INDEX       | 管理者 |
| `base_unit`    | 在庫基準単位   | string(20)  | 不可 | —       | CHECK       | 管理者 |
| `sort_order`   | 表示順         | integer     | 不可 | 0       | CHECK       | 管理者 |
| `active`       | 有効状態       | boolean     | 不可 | true    | —           | 管理者 |
| `memo`         | 備考           | text        | 可   | NULL    | —           | 管理者 |
| `lock_version` | 更新バージョン | integer     | 不可 | 0       | 楽観ロック  | Rails  |
| `created_at`   | 作成日時       | datetime(6) | 不可 | —       | —           | Rails  |
| `updated_at`   | 更新日時       | datetime(6) | 不可 | —       | —           | Rails  |

### DB制約・INDEX

- `PRIMARY KEY(id)`
- `FOREIGN KEY(store_id) REFERENCES stores(id)`
- `UNIQUE(store_id, code)`
- `UNIQUE(store_id, id)`：複合FKの参照先
- `INDEX(store_id, active, sort_order)`
- `INDEX(store_id, name)`
- `CHECK(btrim(code) <> '')`
- `CHECK(btrim(name) <> '')`
- `CHECK(sort_order >= 0)`
- `CHECK(base_unit IN ('g', 'kg', 'ml', 'L', '個', '本', '袋', '缶', '箱'))`

名前の部分一致検索が必要な場合は、
通常のB-treeインデックスとは別に、
pg_trgm等の導入を実装時に検討する。

### 在庫基準単位

各食材に対して、在庫数量を記録する際の基準単位を
1つ設定する。

初期対応単位：

| 区分 | 単位               |
| ---- | ------------------ |
| 重量 | g、kg              |
| 容量 | ml、L              |
| 個数 | 個、本、袋、缶、箱 |

- 同一食材の在庫数量は、登録されたbase_unitで管理する。
- 数量カラムは原則decimal(12,3)を使用する。
- 重量・容量では小数第3位まで扱える。
- 個、本、袋、缶、箱などの個数単位は、
  Phase 1では整数数量のみ許可する。
- 単位に応じた数量入力の検証は棚卸のServiceで行う。
- 単位換算には浮動小数点演算を使用しない。

換算例：

- 1kg = 1000g
- 1L = 1000ml
- 1ケース12本の場合は12本に換算する。

重量と容量の相互換算は、密度などの情報がない限り
自動的には行わない。

発注単位と在庫基準単位の換算係数は、
Phase 2のsupplier_itemsで扱う。

### 在庫管理対象の考え方

- itemsは食材そのものを管理する。
- position_itemsは食材を使用するポジションを管理する。
- stock_location_itemsは保管場所ごとの棚卸対象を管理する。
- items自体には在庫数量を保存しない。
- items自体に棚卸責任ポジションを持たせない。
- 棚卸対象の有効・無効はstock_location_itemsで管理する。

### 食材の名称変更・無効化

- 名称の変更は許可する。
- 食材コードは原則変更しない。
- 棚卸履歴・発注履歴が存在する食材は物理削除しない。
- active=falseとして新規業務の選択対象から除外する。
- 無効化後も過去の棚卸・発注履歴は保持する。
- 基準単位は、数量履歴が存在する場合は原則変更禁止とする。
- 基準単位を変更する必要がある場合は、
  既存数量の換算・監査・履歴整合性を別途検討する。
- 過去の名称や単位を当時の内容で表示する必要がある場合は、
  棚卸・発注の履歴側にスナップショットを保存する。
- 名称変更・単位変更・無効化は監査対象とする。

### 店舗境界

- 食材は必ず1店舗に所属する。
- Phase 1では既存の1店舗をRails側で設定する。
- クライアントが任意のstore_idを指定できない。
- 将来の複数店舗運用では店舗ごとに同じ食材コードを使用可能。
- 食材と保管場所・ポジションの関連は同一店舗に限定する。

### アソシエーション

- `belongs_to :store`
- `has_many :position_items`
- `has_many :positions, through: :position_items`
- `has_many :stock_location_items`
- `has_many :stock_locations, through: :stock_location_items`
- `has_many :inventory_count_lines, through: :stock_location_items`

※履歴を参照する関連では、active=falseの食材も取得できる必要がある。

### 削除方針

- 履歴や関連データが存在する食材の物理削除は禁止する。
- 関連先のFKは原則RESTRICTとする。
- 通常の削除操作はactive=falseへの変更として扱う。

## 8. `position_items`（ポジション別使用食材）

**責務：** 各ポジションが使用する食材を管理する。

**作成・更新主体：** 店舗管理者

**実装フェーズ：** Phase 1

| カラム名       | 日本語名               | 型          | NULL | DEFAULT | Option             | 設定元 |
| -------------- | ---------------------- | ----------- | ---- | ------- | ------------------ | ------ |
| `id`           | ポジション別使用食材ID | bigint      | 不可 | DB採番  | PK                 | DB     |
| `store_id`     | 店舗ID                 | bigint      | 不可 | —       | FK → stores        | Rails  |
| `position_id`  | ポジションID           | bigint      | 不可 | —       | 複合FK → positions | 管理者 |
| `item_id`      | 食材ID                 | bigint      | 不可 | —       | 複合FK → items     | 管理者 |
| `active`       | 使用状態               | boolean     | 不可 | true    | —                  | 管理者 |
| `lock_version` | 更新バージョン         | integer     | 不可 | 0       | 楽観ロック         | Rails  |
| `created_at`   | 作成日時               | datetime(6) | 不可 | —       | —                  | Rails  |
| `updated_at`   | 更新日時               | datetime(6) | 不可 | —       | —                  | Rails  |

### DB制約・INDEX

- `PRIMARY KEY(id)`
- `FOREIGN KEY(store_id) REFERENCES stores(id)`
- `FOREIGN KEY(store_id, position_id) REFERENCES positions(store_id, id)`
- `FOREIGN KEY(store_id, item_id) REFERENCES items(store_id, id)`
- `UNIQUE(store_id, position_id, item_id)`
- `INDEX(store_id, item_id, active)`
- `CHECK(lock_version >= 0)`

複合FKの参照先：

- `positions(store_id, id)`
- `items(store_id, id)`

参照先には対応するUNIQUE制約を定義する。

### 業務ルール

- 1ポジションは複数の食材を使用できる。
- 1食材は複数のポジションから使用できる。
- 同一店舗・同一ポジション・同一食材の組み合わせは1件とする。
- 食材を複数ポジションに紐付けてもitemsは重複作成しない。
- position_itemsには在庫数量を保存しない。
- 発注責任ポジションを管理しない。
- 棚卸責任ポジションを管理しない。
- ポジションへのスタッフ割当はposition_assignmentsで管理する。
- 使用対象の変更は監査対象とする。

### 有効・無効の扱い

- `active=true` は使用中を表す。
- `active=false` は現在の使用対象から除外された状態を表す。
- 無効化しても関連レコードは原則削除しない。
- 同じ組み合わせの再登録では新規レコードを作らず、既存レコードを再有効化する。
- 新規登録・再有効化時はitemsとpositionsの両方がactiveであることを確認する。
- itemsまたはpositionsが無効化された場合、通常の新規選択候補から除外する。
- 過去の棚卸・発注記録の参照は維持する。

### 店舗境界

- position_itemsは必ず1店舗に所属する。
- position_idとitem_idは同じstore_idに所属しなければならない。
- Railsは認証済みスタッフの所属店舗からstore_idを決定する。
- クライアント指定のstore_idは信用しない。
- 他店舗の食材とポジションを組み合わせた登録を禁止する。
- Railsの認可とPostgreSQLの複合FKで店舗境界を保護する。

### 在庫照会との関係

- position_itemsはポジションごとの使用食材の抽出に利用する。
- 物理在庫はstock_location_itemsで特定した保管場所×食材単位で管理する。
- 1食材を複数ポジションが使用しても物理在庫を重複集計しない。
- ポジション別在庫照会では、使用食材の抽出と物理在庫数量の集計を区別する。
- 在庫の最新数量は後続の棚卸記録から取得する。
- 未棚卸と数量0は区別する。

### アソシエーション

- `belongs_to :store`
- `belongs_to :position`
- `belongs_to :item`

positions側：

- `has_many :position_items`
- `has_many :items, through: :position_items`

items側：

- `has_many :position_items`
- `has_many :positions, through: :position_items`

Railsの関連定義とDBの複合FK制約は別途設定する。

### 削除方針

- 通常の削除はactive=falseへの変更とする。
- 関連先の物理削除は原則RESTRICTとする。
- 既存の関連・履歴を維持する。

## 9. `stock_locations`（在庫保管場所）

**責務：** 共通倉庫・冷蔵庫・冷凍庫・各ポジションの棚など、食材の物理的な保管場所を管理する。

**作成・更新主体：** 店舗管理者

**実装フェーズ：** Phase 1

| カラム名        | 日本語名       | 型          | NULL | DEFAULT | Option            | 設定元 |
| --------------- | -------------- | ----------- | ---- | ------- | ----------------- | ------ |
| `id`            | 保管場所ID     | bigint      | 不可 | DB採番  | PK                | DB     |
| `store_id`      | 店舗ID         | bigint      | 不可 | —       | FK → stores       | Rails  |
| `code`          | 保管場所コード | string(50)  | 不可 | —       | 複合UNIQUE・CHECK | 管理者 |
| `name`          | 保管場所名     | string(100) | 不可 | —       | CHECK             | 管理者 |
| `location_kind` | 保管場所区分   | string(30)  | 不可 | —       | CHECK             | 管理者 |
| `sort_order`    | 表示順         | integer     | 不可 | 0       | CHECK             | 管理者 |
| `active`        | 有効状態       | boolean     | 不可 | true    | —                 | 管理者 |
| `memo`          | 備考           | text        | 可   | NULL    | —                 | 管理者 |
| `lock_version`  | 更新バージョン | integer     | 不可 | 0       | 楽観ロック        | Rails  |
| `created_at`    | 作成日時       | datetime(6) | 不可 | —       | —                 | Rails  |
| `updated_at`    | 更新日時       | datetime(6) | 不可 | —       | —                 | Rails  |

### DB制約・INDEX

- `PRIMARY KEY(id)`
- `FOREIGN KEY(store_id) REFERENCES stores(id)`
- `UNIQUE(store_id, code)`
- `UNIQUE(store_id, id)`：複合FKの参照先
- `CHECK(code ~ '^[A-Z0-9][A-Z0-9_-]*$')`
- `CHECK(btrim(name) <> '')`
- `CHECK(location_kind IN ('warehouse', 'refrigerator', 'freezer', 'shelf', 'other'))`
- `CHECK(sort_order >= 0)`
- `CHECK(lock_version >= 0)`
- `INDEX(store_id, active, sort_order)`
- `INDEX(store_id, name)`

保管場所コードは英大文字・数字・ハイフン・アンダースコアに限定する。
小文字で入力された場合の大文字への正規化はRailsで行う。

### 保管場所区分

| コード         | 意味   |
| -------------- | ------ |
| `warehouse`    | 倉庫   |
| `refrigerator` | 冷蔵庫 |
| `freezer`      | 冷凍庫 |
| `shelf`        | 棚     |
| `other`        | その他 |

保管場所区分は検索・表示のための分類とし、
在庫数量や棚卸責任ポジションを決定する属性としては使用しない。

### 保管場所コード

- 店舗内で一意とする。
- 異なる店舗間では同じコードを使用できる。
- コードは管理者が設定する。
- コード例：LOC001、LOC002、LOC003。
- 名称は変更可能とする。
- 過去の業務履歴で参照されるコードは原則変更しない。
- コードの再利用は行わない。

### 物理在庫の管理単位

- 1つの保管場所は、在庫を独立して数える物理的な区画を表す。
- 同じ食材は複数保管場所に存在してよい。
- 保管場所自体には在庫数量を保存しない。
- 食材との関係はstock_location_itemsで管理する。
- 各保管場所の最新確定数量をもとに店舗全体の在庫を集計する。
- 同じ物理在庫を別々の保管場所として二重登録しない。
- Phase 1では親子階層を設けず、集計対象となる最小の管理区画を登録する。

例：

共通倉庫に料理酒5L、焼場冷蔵庫に料理酒1.5Lある場合は、
それぞれ独立した保管場所で管理する。

店舗全体の料理酒は6.5Lとする。

### ポジションとの関係

- stock_locationsにposition_idを持たせない。
- 1つの保管場所に複数ポジションが使用する食材を登録できる。
- 食材ごとの棚卸責任はstock_location_itemsで管理する。
- 実際に棚卸を行ったスタッフは棚卸記録側で管理する。
- 保管場所の区分とポジションの担当関係は独立する。

### 有効・無効の扱い

- active=trueは業務で使用可能な保管場所を表す。
- active=falseは新規の棚卸対象として使用しない状態を表す。
- 無効化しても過去の棚卸履歴は保持する。
- active=falseの保管場所では新規棚卸を開始しない。
- 既存の未確定棚卸がある場合も保管場所の無効化を許可する。
- 無効化時は未確定棚卸の存在を利用者に通知する。
- 入力済みの棚卸明細は自動削除しない。
- 無効化前に開始したdraftは、無効化後も編集・確定できる。
- active=falseの保管場所では新しい棚卸を開始できない。
- 過去の確定済み棚卸と訂正履歴は引き続き参照できる。
- 保管場所の再有効化も許可する。
- 有効なstock_location_itemsが残っていても保管場所の無効化を許可する。
- 無効化によって関連食材や棚卸履歴を自動削除しない。
- 無効化前に作成したdraftは編集・確定できる。
- 無効化後は新規棚卸を開始できない。
- 過去の確定済み棚卸の訂正は引き続き許可する。
- 無効化された保管場所の残存在庫は履歴・在庫照会で確認できる。
- 無効化と関連設定の変更はServiceで管理する。
- 無効化後の再有効化は許可するが、関連する食材の有効状態を再検証する。

未確定棚卸の継続・確定・変更履歴のルールは
inventory-rules.mdに従う。

### 店舗境界

- 保管場所は必ず1店舗に所属する。
- Phase 1ではRailsが運用店舗のstore_idを設定する。
- クライアント指定のstore_idは信用しない。
- 他店舗の食材を保管場所へ紐付けない。
- stock_location_itemsで複合FKを使用して店舗一致を保証する。
- 保管場所へのアクセスは認証済みスタッフの所属店舗で制限する。

### 表示・検索

- 保管場所名で検索できる。
- 保管場所コードで検索できる。
- 保管場所区分で絞り込める。
- active状態で絞り込める。
- sort_order、idの昇順で表示する。
- 棚卸表の保管場所別出力に利用する。
- 食材ごとの表示順はstock_location_itemsで管理する。

### 履歴・監査

- 保管場所名の変更は監査対象とする。
- 保管場所区分の変更は監査対象とする。
- 保管場所の無効化・再有効化は監査対象とする。
- 過去の棚卸は保管場所IDを参照して保持する。
- 過去の名称を当時の内容で表示する必要がある場合は、
  Issue #6で棚卸記録側のスナップショットを設計する。

### アソシエーション

- `belongs_to :store`
- `has_many :stock_location_items`
- `has_many :items, through: :stock_location_items`
- `has_many :inventory_counts`

### 削除方針

- 通常の削除はactive=falseへの変更とする。
- 関連データ・履歴がある場合の物理削除は禁止する。
- 関連先のFKは原則RESTRICTとする。

## 10. `stock_location_items`（保管場所別管理食材）

**責務：** 保管場所ごとの食材と、その棚卸責任ポジションを管理する。

**作成・更新主体：** 店舗管理者

**実装フェーズ：** Phase 1

**対応Issue：** #5

| カラム名                  | 日本語名             | 型          | NULL | DEFAULT | Option                   | 設定元 |
| ------------------------- | -------------------- | ----------- | ---- | ------- | ------------------------ | ------ |
| `id`                      | 保管場所別食材ID     | bigint      | 不可 | DB採番  | PK                       | DB     |
| `store_id`                | 店舗ID               | bigint      | 不可 | —       | FK → stores              | Rails  |
| `stock_location_id`       | 保管場所ID           | bigint      | 不可 | —       | 複合FK → stock_locations | 管理者 |
| `item_id`                 | 食材ID               | bigint      | 不可 | —       | 複合FK → items           | 管理者 |
| `responsible_position_id` | 棚卸責任ポジションID | bigint      | 不可 | —       | 複合FK → positions       | 管理者 |
| `active`                  | 棚卸対象状態         | boolean     | 不可 | true    | —                        | 管理者 |
| `sort_order`              | 棚卸表示順           | integer     | 不可 | 0       | CHECK                    | 管理者 |
| `lock_version`            | 更新バージョン       | integer     | 不可 | 0       | CHECK                    | Rails  |
| `created_at`              | 作成日時             | datetime(6) | 不可 | —       | —                        | Rails  |
| `updated_at`              | 更新日時             | datetime(6) | 不可 | —       | —                        | Rails  |

### DB制約・INDEX

- `PRIMARY KEY(id)`
- `FOREIGN KEY(store_id) REFERENCES stores(id)`
- `FOREIGN KEY(store_id, stock_location_id) REFERENCES stock_locations(store_id, id)`
- `FOREIGN KEY(store_id, item_id) REFERENCES items(store_id, id)`
- `FOREIGN KEY(store_id, responsible_position_id) REFERENCES positions(store_id, id)`
- `UNIQUE(store_id, stock_location_id, item_id)`
- `UNIQUE(store_id, id)`：後続テーブルの複合FK参照先
- `UNIQUE(store_id, stock_location_id, id)`：棚卸明細の複合FK参照先
- `CHECK(sort_order >= 0)`
- `CHECK(lock_version >= 0)`
- `INDEX(store_id, stock_location_id, active, sort_order)`
- `INDEX(store_id, item_id, active)`
- `INDEX(store_id, responsible_position_id, active)`

複合FKの参照先には、以下のUNIQUE制約を定義する。

- `stock_locations(store_id, id)`
- `items(store_id, id)`
- `positions(store_id, id)`

### 採用理由

本テーブルを採用する理由：

1. 1食材が複数保管場所に存在できる。
2. 1保管場所で複数食材を管理できる。
3. 同じ保管場所でも食材ごとに棚卸責任を変更できる。
4. 使用ポジションと棚卸責任を分離できる。
5. 物理在庫の集計単位を明確にできる。

代替案：

- stock_locationsにresponsible_position_idを配置する案
  - 同じ保管場所内で食材ごとに責任者を分けられないため不採用。
- position_itemsにresponsible_position_idを配置する案
  - 使用ポジションと棚卸責任が混在し、
    保管場所ごとの責任を表現できないため不採用。

### 業務ルール

- 1つの保管場所に複数食材を登録できる。
- 同じ食材を複数保管場所に登録できる。
- 同一店舗・同一保管場所・同一食材の組み合わせは1件とする。
- 同じ組み合わせに複数の棚卸責任ポジションを設定しない。
- 棚卸責任ポジションは必ず設定する。
- 1つのポジションが複数食材の棚卸責任を持てる。
- 使用ポジションと棚卸責任ポジションは別の情報とする。
- position_itemsに存在することを棚卸責任の必須条件とはしない。
- 担当スタッフと実際の棚卸実施者は別々に管理する。
- 本テーブルに在庫数量を保存しない。

### 登録時の検証

以下をすべて確認する。

1. 操作スタッフが認証済みである。
2. 操作スタッフが登録権限を持つ。
3. 店舗が有効状態である。
4. 保管場所が有効状態である。
5. 食材が有効状態である。
6. 棚卸責任ポジションが有効状態である。
7. 保管場所・食材・ポジションの店舗が一致する。
8. 同一保管場所・食材の登録が存在しない。

登録時のstore_idはRailsが設定する。
クライアントが指定したstore_idは信用しない。

### 棚卸責任と実施者

- responsible_position_idは通常の棚卸責任ポジションを示す。
- 実際の棚卸実施者はinventory_count_linesで管理する。
- 担当ポジションに所属するスタッフによる棚卸を基本とする。
- 後輩や別スタッフによる代理棚卸を考慮する。
- 代理棚卸の認可条件はIssue #6で確定する。
- 棚卸を実施したスタッフIDと日時は履歴に保存する。
- ポジションの担当履歴はposition_assignmentsで管理する。

### 責任ポジションの変更

棚卸責任ポジションの変更は許可する。

例：

共通倉庫の料理酒について、
棚卸責任を焼場から揚場へ変更する。

変更時のルール：

- stock_location_itemsのresponsible_position_idを更新する。
- 変更操作をaudit_eventsへ記録する。
- 変更前後のポジションIDを監査履歴に残す。
- 過去の確定棚卸の責任ポジションは変更しない。
- 棚卸記録に当時の責任ポジションIDをスナップショット保存する。
- 既存の未確定棚卸がある場合、責任変更と入力済み明細の整合性を検証する。

スナップショットの記録タイミングと、
棚卸途中の責任変更への対応はIssue #6で確定する。

### 在庫集計と二重計上防止

- 在庫管理の物理的な単位は保管場所×食材とする。
- 同じ組み合わせはUNIQUE制約で重複登録を禁止する。
- position_itemsの件数を在庫集計に使用しない。
- 物理在庫は保管場所別に独立して数える。
- 店舗全体の在庫は、対象食材の各保管場所における
  最新確定数量を集計する。
- 未確認数量と実測0を区別する。
- 同じ物理区画を重複した保管場所として登録しない。
- 集計対象となる保管場所は互いに重複しない管理区画とする。

例：

共通倉庫の料理酒：5L
焼場冷蔵庫の料理酒：1.5L

店舗全体の料理酒：6.5L

料理酒の使用ポジションが2つ以上でも、
その数によって在庫数量を増加させない。

なお、本テーブルのUNIQUE制約だけでは、
棚卸記録の重複や物理区画の重複まで保証できない。

棚卸明細の一意性、確定処理、在庫算出方式は
Issue #6で別途定義する。

### 有効・無効の扱い

- active=trueは棚卸対象として有効な状態を表す。
- active=falseは新規棚卸対象から除外する状態を表す。
- 無効化しても過去の棚卸記録は保持する。
- 無効化済みの同一組み合わせを再登録する場合は、
  新規作成せず既存レコードを再有効化する。
- 再有効化時は保管場所・食材・ポジションの
  有効状態を再検証する。
- 棚卸がdraft中でも管理対象の変更を許可する。
- 食材を追加した場合、対象draftに未確認明細を追加する。
- 対象外にする場合は、入力済み数量を利用者へ確認する。
- 対象外となるdraft明細は変更前情報を監査記録に保存してから除外する。
- 過去のconfirmed明細は変更・削除しない。
- 在庫が残る食材を無効化した場合、残数量を履歴・在庫照会で追跡可能にする。
- 責任ポジションの変更では、記録済みの実施者と実施日時を変更しない。
- 最新確定数量が残っている対象を無効化する場合は、
  在庫移動・ロス・棚卸訂正等で数量の扱いを整理する。
- active=falseを理由に過去の確定在庫履歴を削除しない。

残存在庫の扱いと無効化条件の詳細は、
Issue #6および後続の在庫移動設計で確定する。

### 店舗境界

- 保管場所・食材・責任ポジションは同じ店舗に所属する。
- Railsの認可に加えて複合FKで店舗一致を保証する。
- 他店舗の食材を保管場所へ登録することを禁止する。
- 他店舗のポジションを棚卸責任として指定できない。
- 他店舗の保管場所別食材情報を参照・変更できない。

### 履歴・監査

以下の操作を監査対象とする。

- 保管場所別食材の登録
- 棚卸責任ポジションの変更
- 棚卸対象の無効化
- 棚卸対象の再有効化

過去の棚卸履歴では、
記録当時の責任ポジション・実施者・数量を保持する。

### アソシエーション

- `belongs_to :store`
- `belongs_to :stock_location`
- `belongs_to :item`
- `belongs_to :responsible_position, class_name: "Position"`
- `has_many :inventory_count_lines`

stock_locations側：

- `has_many :stock_location_items`
- `has_many :items, through: :stock_location_items`

items側：

- `has_many :stock_location_items`
- `has_many :stock_locations, through: :stock_location_items`

positions側：

- `has_many :responsible_stock_location_items,
class_name: "StockLocationItem",
foreign_key: :responsible_position_id`

※複合FKを含む具体的なRails関連設定は実装Issueで確認する。

### 削除方針

- 通常の削除はactive=falseへの変更とする。
- 棚卸履歴を持つ関連レコードは物理削除しない。
- 参照先の物理削除は原則RESTRICTとする。

### Phase 2の発注責任との境界

発注を集約する主担当ポジションは、
棚卸責任ポジションと独立して管理する。

候補の比較：

- itemsに発注責任を持たせる案
  - 食材ごとに一律の担当を設定する場合は単純。
  - 仕入先・発注条件による違いを表現しづらい。
- position_itemsに発注責任を持たせる案
  - 食材を使用するポジションと発注担当が混在する。
- Phase 2の発注関連テーブルで管理する案
  - 仕入先や発注単位に応じた担当を表現しやすい。

採用方針：

Phase 1では発注責任を本テーブルへ追加しない。
Phase 2ではsupplier_items等の発注設定と、
purchase_orders.ordering_position_idの役割を整理し、
主担当ポジションの保存先を確定する。

### 最新確定在庫の取得ルール

- 在庫照会は対象営業日（as_of）を指定できる。
- 在庫の集計単位はstock_location_item_idとする。
- 対象営業日以前の確定済み棚卸明細から、
  保管場所×食材ごとに最新の1件を取得する。
- 下書きの棚卸数量は在庫集計に含めない。
- 同じ食材が複数ポジションで使用されていても、
  position_itemsとのJOINによって数量を重複加算しない。
- 最新確定数量が存在しない場合は未確認とし、
  数量0として扱わない。
- 実測数量0は確認済みの数量0として扱う。
- 各保管場所の確認日が異なる場合、
  店舗合計は「各場所の最新確認値の合計」と明示する。
- この合計をリアルタイムの実在庫と同一視しない。
- 最終確認日時と確認スタッフを表示できるようにする。
- 使用する数量の基準単位はitems.base_unitとする。

最新記録の抽出順序、訂正履歴の優先順位、
同一営業日における再棚卸はIssue #6で確定する。

## 11. `inventory_counts`（日次在庫確認）

**責務：** 店舗・営業日・保管場所ごとの棚卸作業、確定状態、再棚卸履歴を管理する。

**対応Issue：** #6

**実装フェーズ：** Phase 1

| カラム名            | 日本語名         | 型          | NULL | DEFAULT | Option                   | 設定元           |
| ------------------- | ---------------- | ----------- | ---- | ------- | ------------------------ | ---------------- |
| `id`                | 棚卸ID           | bigint      | 不可 | DB採番  | PK                       | DB               |
| `store_id`          | 店舗ID           | bigint      | 不可 | —       | FK → stores              | Rails            |
| `stock_location_id` | 保管場所ID       | bigint      | 不可 | —       | 複合FK → stock_locations | Rails            |
| `business_date`     | 営業日           | date        | 不可 | —       | 複合UNIQUE               | スタッフ・Rails  |
| `revision_no`       | 棚卸版番号       | integer     | 不可 | 1       | 複合UNIQUE・CHECK        | Rails            |
| `revision_reason`   | 再棚卸・訂正理由 | text        | 可   | NULL    | CHECK                    | スタッフ（任意） |
| `status`            | 棚卸状態         | string(30)  | 不可 | draft   | CHECK                    | Rails            |
| `opened_by_id`      | 作成スタッフID   | bigint      | 不可 | —       | 複合FK → staffs          | Rails            |
| `confirmed_by_id`   | 確定スタッフID   | bigint      | 可   | NULL    | 複合FK → staffs          | Rails            |
| `confirmed_at`      | 確定日時         | datetime(6) | 可   | NULL    | CHECK                    | Rails            |
| `lock_version`      | 更新バージョン   | integer     | 不可 | 0       | CHECK・楽観ロック        | Rails            |
| `created_at`        | 作成日時         | datetime(6) | 不可 | —       | —                        | Rails            |
| `updated_at`        | 更新日時         | datetime(6) | 不可 | —       | —                        | Rails            |

### DB制約・INDEX

- `PRIMARY KEY(id)`
- `FOREIGN KEY(store_id) REFERENCES stores(id)`
- `FOREIGN KEY(store_id, stock_location_id) REFERENCES stock_locations(store_id, id)`
- `FOREIGN KEY(store_id, opened_by_id) REFERENCES staffs(store_id, id)`
- `FOREIGN KEY(store_id, confirmed_by_id) REFERENCES staffs(store_id, id)`
- `UNIQUE(store_id, id)`：後続テーブルの複合FK参照先
- `UNIQUE(store_id, stock_location_id, business_date, revision_no)`
- `CHECK(revision_no >= 1)`
- `CHECK(lock_version >= 0)`
- `CHECK(status IN ('draft', 'confirmed'))`
- `CHECK (revision_reason IS NULL OR btrim(revision_reason) <> '')`
- `CHECK((status = 'draft' AND confirmed_by_id IS NULL AND confirmed_at IS NULL) OR (status = 'confirmed' AND confirmed_by_id IS NOT NULL AND confirmed_at IS NOT NULL))`
- `INDEX(store_id, business_date, status)`

次の部分UNIQUEインデックスを追加する。

`UNIQUE(store_id, stock_location_id, business_date) WHERE status = 'draft'`

- `UNIQUE(store_id, stock_location_id, id)`：棚卸明細の複合FK参照先

これにより、同一店舗・同一営業日・同一保管場所に、
未確定の棚卸が複数存在することを防止する。

最新確定版の参照に備えて、次の部分インデックスを追加する。

`INDEX(store_id, stock_location_id, business_date DESC, revision_no DESC) WHERE status = 'confirmed'`

### 棚卸の管理単位

- 基本単位は店舗×営業日×保管場所とする。
- 責任ポジション単位でヘッダーを分割しない。
- 1つのヘッダーに複数の食材明細を持つ。
- 1つのヘッダーを複数スタッフで分担できる。
- 実際に食材を数えたスタッフは明細側で管理する。
- 食材ごとの棚卸責任ポジションはstock_location_itemsを参照する。

### 棚卸の版管理

- 最初の棚卸はrevision_no=1とする。
- 確定後の再棚卸・訂正では新しい版を作成する。
- 第2版以降はrevision_noを1ずつ増やす。
- revision_reasonは任意入力とする。
- 通常の再棚卸はdraftとして開始できる。
- 確定済み棚卸の画面編集は許可する。
- 確定後の編集内容を保存した場合、新しいconfirmed版を自動作成する。
- 確定後の編集に管理者承認は必要としない。
- 過去のconfirmedレコードは直接更新・削除しない。
- 新しい版には前回確定版の明細を引き継ぐ。
- 変更していない明細も含めて完全なスナップショットを保存する。
- 変更前後の数量・変更スタッフ・変更日時を追跡できるようにする。
- 過去の各版は履歴画面から参照できる。
- 同時更新では古い内容による上書きを防止する。
- 未確定のdraftが存在する場合は、その内容を確認してから編集結果を反映する。
- draftと確定後編集の内容を無条件に上書き・破棄しない。

### 下書きと確定後訂正の競合

- 同じ営業日・保管場所にdraftが存在する場合、別操作による確定版の即時作成を拒否する。
- 利用者には既存のdraftを編集・確定するよう案内する。
- 確定版の画面編集では、編集開始時のrevision_noを保持する。
- 保存時に現在の最新確定revision_noと比較する。
- 編集開始後に別スタッフが新しい版を確定していた場合、競合として保存を拒否する。
- 競合時は最新データを再取得し、変更内容を確認できるようにする。
- 同じ棚卸明細への同時更新はlock_versionで検出する。

### 最新確定版の選択

- 店舗・保管場所・営業日ごとに確定版を取得する。
- 同じ営業日に複数の確定版がある場合、revision_noが最大のものを採用する。
- draftは在庫照会の集計対象としない。
- 過去の確定版は履歴として保存するが、最新数量の集計では重複加算しない。
- 対象営業日以前の最新確定版を在庫照会に使用する。
- 異なる営業日の比較ではbusiness_dateを優先し、同一営業日内ではrevision_noを使用する。
- 再棚卸の確定日時だけを理由に、後日の営業日の棚卸を上書きしない。

### 状態遷移

許可する操作：

- 新規作成 → draft
- draft → draft（編集・保存）
- draft → confirmed（確定）
- confirmedの画面編集 → 新しいconfirmed版を作成
- confirmedから手動再棚卸 → 新しいdraft版を作成

禁止する操作：

- 既存のconfirmedレコードをdraftに戻す
- 既存のconfirmedレコードの数量を直接上書きする
- 既存のconfirmedレコードを物理削除する

### 確定後の編集

- 確定済み棚卸は画面から編集可能とする。
- 編集時に追加の承認は要求しない。
- 編集内容の保存時に新しいrevision_noの確定版を自動作成する。
- 新しい確定版のconfirmed_by_idには保存操作を行ったスタッフを設定する。
- confirmed_atには保存時の日時を設定する。
- 訂正理由の入力は任意とする。
- 編集前の確定版は履歴として保持する。
- 訂正後の数量は最新確定在庫として取得する。
- 保存処理はDBトランザクション内で実行する。
- 競合が発生した場合は自動上書きせず、再取得を促す。

### 確定条件

確定時はServiceで以下を検証する。

1. 操作スタッフが認証済みかつ有効である。
2. 通常の確定では棚卸ヘッダーがdraftである。
3. 確定後編集では元のconfirmedを直接変更せず、新しいconfirmed版を作成する。
4. 対象保管場所が同一店舗に所属する。
5. 必須の棚卸明細がそろっている。
6. 必要な実測数量がすべて入力されている。
7. 実測数量に負数・不正な精度が含まれない。
8. 記録元に応じた実施者・実施日時・入力者の記録条件を満たしている。
9. 棚卸開始後に対象が変化した場合、整合性を再検証する。
10. 他の処理による確定・訂正との競合がない。
11. 店舗・営業日・保管場所の版番号が重複しない。

未確認NULLは確定できない。
実測数量0は有効な確認済み数量として扱う。

確定処理はDBトランザクション内で実行する。
確定者・確定日時・ステータスの更新を一括して確定する。

### 同時更新・競合防止

- lock_versionによる楽観ロックを使用する。
- 確定時は棚卸ヘッダーを行ロックして状態を再確認する。
- 明細更新時も同じヘッダーの状態とロックを確認する。
- 確定処理中に明細が変更されないよう、同じロック規約を使用する。
- 新しいrevision_noの採番はトランザクションで行う。
- 採番時は対象stock_locationsの行ロックで同時作成を直列化する。
- UNIQUE制約による重複拒否を最終防衛線とする。
- 競合時は処理をロールバックし、利用者へ再取得を促す。

### 店舗境界

- store_idは認証済みスタッフの店舗からRails側で決定する。
- クライアント指定のstore_idは信用しない。
- 保管場所、作成者、確定者は同一店舗に所属する。
- 複合FKによって他店舗の関連付けを防止する。
- 他店舗の棚卸一覧・詳細の参照と更新を拒否する。

### 監査・削除方針

- 棚卸の作成・確定・再棚卸を監査対象とする。
- 確定済み棚卸は物理削除しない。
- 確定済み棚卸の修正は新しい版として保存する。
- 再棚卸・訂正理由が入力された場合は保持する。
- 訂正理由の有無にかかわらず、変更者・変更日時・変更前後の数量を追跡可能とする。
- どの版が有効な最新確定版かを照会時に判定する。

### アソシエーション

- `belongs_to :store`
- `belongs_to :stock_location`
- `belongs_to :opened_by, class_name: "Staff"`
- `belongs_to :confirmed_by, class_name: "Staff", optional: true`
- `has_many :inventory_count_lines`

stock_locations側：

- `has_many :inventory_counts`

staffs側：

- `has_many :opened_inventory_counts, class_name: "InventoryCount", foreign_key: :opened_by_id`
- `has_many :confirmed_inventory_counts, class_name: "InventoryCount", foreign_key: :confirmed_by_id`

### 設計判断

採用：同一営業日内の複数確定版をrevision_noで管理する。

理由：

- 確定済みの実績を直接上書きしない。
- 再棚卸・訂正履歴を保持できる。
- 最新確定版だけを選択して集計できる。
- 新しい訂正専用テーブルを初期段階で増やさずに済む。

留意点：

- 明細は各版に完全なスナップショットとして保持する。
- 版の作成時には前回確定版が存在することを検証する。
- 初版のrevision_noは1とする。
- 新しい版番号は現在の最大番号+1とし、連番の重複を禁止する。
- 物理削除をしないため、版番号を再利用しない。

## 12. `inventory_count_lines`（日次在庫確認明細）

**責務：** 食材ごとの実測数量、品質状態、棚卸実施者、入力者、棚卸当時の情報を記録する。

**対応Issue：** #6

**対応要件：** FR-03、FR-04

**実装フェーズ：** Phase 1

| カラム名                           | 日本語名                 | 型            | NULL | DEFAULT | Option        | 設定元        |
| ---------------------------------- | ------------------------ | ------------- | ---- | ------- | ------------- | ------------- |
| `id`                               | 棚卸明細ID               | bigint        | 不可 | DB採番  | PK            | DB            |
| `store_id`                         | 店舗ID                   | bigint        | 不可 | —       | FK            | Rails         |
| `stock_location_id`                | 保管場所ID               | bigint        | 不可 | —       | 複合FK        | Rails         |
| `inventory_count_id`               | 棚卸ヘッダーID           | bigint        | 不可 | —       | 複合FK        | Rails         |
| `stock_location_item_id`           | 保管場所別食材ID         | bigint        | 不可 | —       | 複合FK        | Rails         |
| `measured_quantity`                | 実測数量                 | decimal(12,3) | 可   | NULL    | CHECK         | 入力者        |
| `item_name_snapshot`               | 棚卸時の食材名           | string(150)   | 不可 | —       | CHECK         | Rails         |
| `base_unit_snapshot`               | 棚卸時の基準単位         | string(20)    | 不可 | —       | CHECK         | Rails         |
| `quality_status`                   | 品質状態                 | string(30)    | 可   | NULL    | CHECK         | 入力者        |
| `quality_note`                     | 品質メモ                 | text          | 可   | NULL    | —             | 入力者        |
| `count_source`                     | 記録元                   | string(30)    | 可   | NULL    | CHECK         | Rails         |
| `source_note`                      | 記録元の補足             | text          | 可   | NULL    | —             | 入力者        |
| `source_line_id`                   | 元となった棚卸明細ID     | bigint        | 可   | NULL    | 複合FK・CHECK | Rails         |
| `counted_by_id`                    | 実際に数えたスタッフ     | bigint        | 可   | NULL    | 複合FK        | Rails         |
| `counted_at`                       | 実際に数えた日時         | datetime(6)   | 可   | NULL    | —             | Rails・入力者 |
| `entered_by_id`                    | システム入力スタッフ     | bigint        | 可   | NULL    | 複合FK        | Rails         |
| `entered_at`                       | システム入力日時         | datetime(6)   | 可   | NULL    | CHECK         | Rails         |
| `responsible_position_id_snapshot` | 棚卸時の責任ポジションID | bigint        | 不可 | —       | 複合FK        | Rails         |
| `lock_version`                     | 更新バージョン           | integer       | 不可 | 0       | CHECK         | Rails         |
| `created_at`                       | 作成日時                 | datetime(6)   | 不可 | —       | —             | Rails         |
| `updated_at`                       | 更新日時                 | datetime(6)   | 不可 | —       | —             | Rails         |

### DB制約・INDEX

- `PRIMARY KEY(id)`
- `FOREIGN KEY(store_id) REFERENCES stores(id)`
- `FOREIGN KEY(store_id, stock_location_id, inventory_count_id) REFERENCES inventory_counts(store_id, stock_location_id, id)`
- `FOREIGN KEY(store_id, stock_location_id, stock_location_item_id) REFERENCES stock_location_items(store_id, stock_location_id, id)`
- `FOREIGN KEY(store_id, counted_by_id) REFERENCES staffs(store_id, id)`
- `FOREIGN KEY(store_id, entered_by_id) REFERENCES staffs(store_id, id)`
- `FOREIGN KEY(store_id, responsible_position_id_snapshot) REFERENCES positions(store_id, id)`
- `FOREIGN KEY(store_id, source_line_id) REFERENCES inventory_count_lines(store_id, id)`
- `UNIQUE(store_id, id)`
- `UNIQUE(inventory_count_id, stock_location_item_id)`
- `CHECK(measured_quantity >= 0)`
- `CHECK(lock_version >= 0)`
- `CHECK(btrim(item_name_snapshot) <> '')`
- `CHECK(btrim(base_unit_snapshot) <> '')`
- `CHECK(quality_status IS NULL OR quality_status IN ('good', 'caution', 'discard_candidate'))`
- `CHECK(count_source IS NULL OR count_source IN ('direct', 'paper', 'carried_forward', 'corrected'))`
- `CHECK(measured_quantity IS NULL OR (count_source IS NOT NULL AND entered_by_id IS NOT NULL AND entered_at IS NOT NULL))`
- `CHECK(count_source IS DISTINCT FROM 'direct' OR (counted_by_id IS NOT NULL AND counted_at IS NOT NULL))`
- `CHECK ((source_line_id IS NULL AND (count_source IS NULL OR count_source IN ('direct', 'paper')))OR(source_line_id IS NOT NULL AND count_source IS NOT NULL AND count_source IN ('carried_forward', 'corrected')))`
- `INDEX(store_id, stock_location_item_id, inventory_count_id)`
- `INDEX(store_id, counted_by_id, counted_at)`

### 保管場所・店舗の一致

棚卸ヘッダーと棚卸明細は、同じ保管場所に所属する。

そのため、明細にstore_idとstock_location_idを保持し、
以下の2つの複合FKで整合性を保証する。

- 棚卸ヘッダーの店舗・保管場所・ID
- 保管場所別食材の店舗・保管場所・ID

単純なinventory_count_idとstock_location_item_idの
FKだけでは、異なる保管場所の食材を
同じ棚卸に登録することをDB制約だけで防げない。

重複するstore_idとstock_location_idは
クライアントから受け取らずRails側で設定する。

### 棚卸数量

- 数量の基準単位はitems.base_unitとする。
- measured_quantityはdecimal(12,3)で保存する。
- NULLは未確認を表す。
- 0は実測済みの数量0を表す。
- 負数を禁止する。
- 個、本、袋、缶、箱は整数数量のみ許可する。
- 不正な小数精度は保存前にServiceで拒否する。
- 下書き中は数量未入力を許可する。
- 確定時は必要な明細すべてに数量が必要となる。

PostgreSQLのnumeric型は小数桁を丸める場合があるため、
入力精度の検証はDB保存前にも行う。

### 品質状態

| コード              | 意味     |
| ------------------- | -------- |
| `good`              | 良好     |
| `caution`           | 要注意   |
| `discard_candidate` | 廃棄候補 |

- 品質状態はNULLを許可する。
- 品質メモは自由記述とする。
- 廃棄候補であることだけでロス記録を自動作成しない。
- ロス実績はloss_recordsで別途記録する。

### 記録元

| コード            | 意味                         |
| ----------------- | ---------------------------- |
| `direct`          | システム上で実測して入力     |
| `paper`           | 紙の棚卸結果を転記           |
| `carried_forward` | 前版の数量を変更せず引き継ぐ |
| `corrected`       | 前版の数量や内容を訂正       |

- directの場合はcounted_by_idとcounted_atを必須とする。
- paperの場合は実際の実施者・日時が不明でも登録可能とする。
- paperで実施者または日時が不明な場合はsource_noteを必須とする。
- carried_forwardとcorrectedではsource_line_idを必須とする。
- source_line_idの参照先は同一店舗とする。
- 元明細は同じ保管場所別食材の過去版でなければならない。
- source_line_idが参照する版の前後関係はServiceで検証する。
- copiedされた数量を新たな実測として扱わない。
- correctedの訂正理由は任意とし、入力された場合は棚卸ヘッダーのrevision_reasonに保持する。
- 訂正理由の有無にかかわらず、訂正前後の明細と変更者・変更日時を追跡できるようにする。

### 実施者と入力者

- counted_by_idは実際に食材を数えたスタッフを表す。
- counted_atは実際の計測日時を表す。
- entered_by_idはシステムへ入力したスタッフを表す。
- entered_atはシステムへの入力日時を表す。
- 実施者と入力者が異なっていてもよい。
- paperで実施者が不明な場合はcounted_by_idをNULLにできる。
- confirmedの明細ではentered_by_idとentered_atを必須とする。
- 有効な同一店舗スタッフは、担当ポジションにかかわらず代理棚卸できる。
- 実施者と入力者は別々に記録する。
- 他店舗のスタッフによる操作は禁止する。

### 履歴スナップショット

- item_name_snapshotには記録当時の食材名を保持する。
- base_unit_snapshotには記録当時の基準単位を保持する。
- responsible_position_id_snapshotには記録当時の責任ポジションIDを保持する。
- 新規計測では実測情報を登録した時点でスナップショットを確定する。
- 再棚卸で前版を引き継ぐ場合、元の実測日時と実施者は保持する。
- 引き継ぎ後に新たに計測した場合は、実測情報とスナップショットを更新する。
- 確定済み明細のスナップショットは変更しない。
- 現在のマスタ情報が変更されても過去の確定記録に影響させない。

### 版管理と二重計上防止

- 同じ棚卸ヘッダーに同じstock_location_item_idを複数登録しない。
- 初版と訂正版は異なるinventory_count_idを持つ。
- 訂正版は各対象食材を含む完全なスナップショットとする。
- 最新確定版の数量だけを在庫照会に使用する。
- 過去版は履歴として保持し、数量を重複加算しない。
- 同じ食材を使用するポジション数で数量を増やさない。

### 同時更新

- 明細の更新にはlock_versionによる楽観ロックを使用する。
- 明細の作成・更新時は棚卸ヘッダーの状態も確認する。
- ヘッダーと明細の変更は共通のトランザクション規約に従う。
- 確定処理中の明細更新は許可しない。
- 確定済みヘッダーの明細を直接更新しない。
- 競合時は処理を中断し、利用者に再取得を促す。

### アソシエーション

- `belongs_to :store`
- `belongs_to :stock_location`
- `belongs_to :inventory_count`
- `belongs_to :stock_location_item`
- `belongs_to :counted_by, class_name: "Staff", optional: true`
- `belongs_to :entered_by, class_name: "Staff", optional: true`
- `belongs_to :responsible_position_snapshot, class_name: "Position", foreign_key: :responsible_position_id_snapshot`
- `belongs_to :source_line, class_name: "InventoryCountLine", optional: true`

棚卸ヘッダー側：

- `has_many :inventory_count_lines`

保管場所別食材側：

- `has_many :inventory_count_lines`

### 削除・監査方針

- 確定済み明細は物理削除しない。
- 確定済み明細は直接編集しない。
- 訂正は新しい棚卸版で管理する。
- 実測者・入力者・数量変更を追跡できるようにする。
- 重要な変更操作はaudit_eventsへ保存する。

## 13. `loss_records`（ロス記録）

**責務：** 食材の廃棄・破損・腐敗などのロス実績と訂正履歴を管理する。

**対応Issue：** #6

**対応要件：** FR-05、FR-04

**実装フェーズ：** Phase 1

| カラム名                  | 日本語名               | 型            | NULL | DEFAULT  | Option     | 設定元        |
| ------------------------- | ---------------------- | ------------- | ---- | -------- | ---------- | ------------- |
| `id`                      | ロス記録ID             | bigint        | 不可 | DB採番   | PK         | DB            |
| `store_id`                | 店舗ID                 | bigint        | 不可 | —        | FK         | Rails         |
| `stock_location_item_id`  | 保管場所別食材ID       | bigint        | 不可 | —        | 複合FK     | 入力者        |
| `business_date`           | 対象営業日             | date          | 不可 | —        | INDEX      | 入力者・Rails |
| `quantity`                | ロス数量               | decimal(12,3) | 不可 | —        | CHECK      | 入力者        |
| `reason_code`             | ロス理由               | string(30)    | 不可 | —        | CHECK      | 入力者        |
| `occurred_at`             | ロス発生日時           | datetime(6)   | 不可 | —        | INDEX      | 入力者        |
| `reported_by_id`          | ロス報告スタッフID     | bigint        | 不可 | —        | 複合FK     | 入力者        |
| `entered_by_id`           | システム登録スタッフID | bigint        | 不可 | —        | 複合FK     | Rails         |
| `item_name_snapshot`      | 記録時の食材名         | string(150)   | 不可 | —        | CHECK      | Rails         |
| `base_unit_snapshot`      | 記録時の基準単位       | string(20)    | 不可 | —        | CHECK      | Rails         |
| `memo`                    | 備考                   | text          | 可   | NULL     | —          | 入力者        |
| `status`                  | 記録状態               | string(20)    | 不可 | recorded | CHECK      | Rails         |
| `voided_at`               | 取消日時               | datetime(6)   | 可   | NULL     | CHECK      | Rails         |
| `voided_by_id`            | 取消スタッフID         | bigint        | 可   | NULL     | 複合FK     | Rails         |
| `void_reason`             | 取消理由               | text          | 可   | NULL     | CHECK      | 入力者        |
| `replaces_loss_record_id` | 訂正元ロス記録ID       | bigint        | 可   | NULL     | 自己参照FK | Rails         |
| `request_key`             | 重複送信防止キー       | uuid          | 不可 | —        | 複合UNIQUE | クライアント  |
| `lock_version`            | 更新バージョン         | integer       | 不可 | 0        | CHECK      | Rails         |
| `created_at`              | 登録日時               | datetime(6)   | 不可 | —        | —          | Rails         |
| `updated_at`              | 更新日時               | datetime(6)   | 不可 | —        | —          | Rails         |

### DB制約・INDEX

- `PRIMARY KEY(id)`
- `FOREIGN KEY(store_id) REFERENCES stores(id)`
- `FOREIGN KEY(store_id, stock_location_item_id) REFERENCES stock_location_items(store_id, id)`
- `FOREIGN KEY(store_id, reported_by_id) REFERENCES staffs(store_id, id)`
- `FOREIGN KEY(store_id, entered_by_id) REFERENCES staffs(store_id, id)`
- `FOREIGN KEY(store_id, voided_by_id) REFERENCES staffs(store_id, id)`
- `FOREIGN KEY(store_id, replaces_loss_record_id) REFERENCES loss_records(store_id, id)`
- `UNIQUE(store_id, id)`
- `UNIQUE(store_id, request_key)`
- `UNIQUE(replaces_loss_record_id)`
- `CHECK(quantity > 0)`
- `CHECK(lock_version >= 0)`
- `CHECK(btrim(item_name_snapshot) <> '')`
- `CHECK(btrim(base_unit_snapshot) <> '')`
- `CHECK(reason_code IN ('disposal', 'damage', 'spoilage', 'other'))`
- `CHECK(reason_code <> 'other' OR (memo IS NOT NULL AND btrim(memo) <> ''))`
- `CHECK(status IN ('recorded', 'voided'))`
- `CHECK(replaces_loss_record_id IS NULL OR replaces_loss_record_id <> id)`

取消状態の整合性には次のCHECKを使用する。

```sql
CHECK (
  (
    status = 'recorded'
    AND voided_at IS NULL
    AND voided_by_id IS NULL
    AND void_reason IS NULL
  )
  OR
  (
    status = 'voided'
    AND voided_at IS NOT NULL
    AND voided_by_id IS NOT NULL
    AND void_reason IS NOT NULL
    AND btrim(void_reason) <> ''
  )
)
```

検索用INDEX：

- `INDEX(store_id, business_date, status)`
- `INDEX(store_id, stock_location_item_id, business_date)`
- `INDEX(store_id, reported_by_id, occurred_at)`

### ロス理由

| コード     | 日本語表示 |
| ---------- | ---------- |
| `disposal` | 廃棄       |
| `damage`   | 破損       |
| `spoilage` | 腐敗       |
| `other`    | その他     |

- その他を選択した場合は備考を必須とする。
- ロス理由と品質状態は別々に管理する。
- 品質状態が廃棄候補であるだけではロス実績に含めない。
- 実際に廃棄・破損等が発生したときにロス記録を作成する。

### 営業日と発生日時

- business_dateはロスを帰属させる店舗営業日とする。
- occurred_atは実際にロスが発生した日時とする。
- created_atはシステムに登録した日時とする。
- 発生日時と登録日時が異なることを許可する。
- 過去営業日のロスを後から登録することを許可する。
- 店舗の営業日は暦日と一致するとは限らない。
- business_dateの確定方法は店舗の営業日ルールに従う。
- occurred_atとbusiness_dateの整合性をServiceで検証する。
- ロス集計では営業日と登録日時を混同しない。

### 数量と基準単位

- quantityはitems.base_unitに対応する数量を保存する。
- quantityはdecimal(12,3)とする。
- 数量0および負数は禁止する。
- 個数単位は整数のみ許可する。
- 入力精度はDB保存前にServiceで検証する。
- 登録時の食材名と基準単位をスナップショット保存する。
- 基準単位の異なる数量は直接加算しない。
- ロス記録の登録によって棚卸明細の実測数量を変更しない。

### 登録時の検証

- 操作スタッフが認証済みである。
- 操作スタッフが登録権限を持つ。
- 店舗・保管場所別食材・報告者が同一店舗に所属する。
- 対象食材が有効な状態である。
- 数量と基準単位が整合している。
- 発生日時と営業日の対応が妥当である。
- ロス理由が許可された値である。
- request_keyによる重複送信を防止する。

### 二重登録の防止

- 1回のロス登録操作に1つのrequest_keyを割り当てる。
- 通信失敗による同一操作の再送では同じrequest_keyを使用する。
- UNIQUE(store_id, request_key)で重複登録を防ぐ。
- 再送時は既存登録を確認し、二重加算しない。
- 別々に発生したロスは同じ食材・同じ営業日でも登録可能とする。
- 異なるrequest_keyで登録された業務上の重複は、確認・取消操作で解消する。

### ロス記録の訂正・取消

- 登録済みの数量・理由・発生日時を直接書き換えない。
- 誤登録を取り消す場合はstatusをvoidedに変更する。
- 取消時はvoided_by_id、voided_at、void_reasonを必須とする。
- 取り消した記録も物理削除しない。
- 訂正する場合は元記録を取り消し、新しいロス記録を作成する。
- 新しい記録のreplaces_loss_record_idに訂正元IDを保存する。
- 訂正元と訂正後は同じ店舗・保管場所別食材に限定する。
- 1つの訂正元に対して複数の訂正後レコードを作成しない。
- 取り消しと訂正版の登録は同一トランザクションで行う。
- 訂正版は新しいrequest_keyを使用する。
- 訂正済みレコードをさらに訂正する場合も同じ方式を使用する。
- 取消済みレコードを再度取消できない。
- 重要操作はaudit_eventsへ保存する。

### 在庫照会との関係

- ロス記録は実測在庫を自動的に減算するデータではない。
- 最新確定在庫は棚卸の実測数量から取得する。
- ロスが発生しても過去の棚卸数量は変更しない。
- ロスの登録・取消・訂正は、ロス実績の集計結果に反映する。
- 通常のロス集計ではstatus=recordedのみ採用する。
- status=voidedは履歴照会の対象とし、通常の集計から除外する。
- 推定使用量の算定に利用する場合は、棚卸の計測区間に発生したロスのみを対象とする。

### 推定使用量との関係

将来の推定使用量では、次の数量を区別する。

- 期首の確定棚卸数量
- 期末の確定棚卸数量
- 期間中の入庫数量
- 期間中の保管場所間移動
- 期間中のロス数量

ロス発生後に実測した期末棚卸数量には、
すでに物理的な数量減少が反映されている。

したがって、ロス数量を期末実測数量から
再度差し引いてはならない。

概念上の算定式：

推定業務使用量 = 期首数量 + 入庫数量 + 移動入数量 - 移動出数量 - 期末数量 - ロス数量

- 移動出数量 - 期末数量 - ロス数量

ただし、この式は以下を前提とする。

- 期首と期末が独立した実測値である。
- 入出庫とロスの対象期間が一致している。
- ロスが期末実測に反映されている。
- 他の在庫増減や計測差異を別途考慮している。
- ロスの訂正・取消が反映済みである。

必要なデータがそろわない場合は、
推定業務使用量を確定値として表示しない。

Phase 1ではロス実績の記録を主目的とし、
推定使用量の本格算定は後続の発注支援設計で扱う。

### 店舗境界と認可

- store_idは認証済みスタッフの所属店舗から決定する。
- クライアント指定のstore_idは信用しない。
- 他店舗の保管場所別食材を登録できない。
- 他店舗のスタッフを報告者・入力者・取消者に指定できない。
- 他店舗のロス記録の参照・訂正・取消は禁止する。
- 取消・訂正は許可された権限を持つスタッフのみ実行する。

### 同時更新・トランザクション

- lock_versionによる楽観ロックを使用する。
- 取消前に現在のstatusを確認する。
- 二重取消・同時訂正を拒否する。
- 訂正元の取消、訂正版の登録、監査記録を
  同一トランザクションで処理する。
- 途中で失敗した場合は全体をロールバックする。
- request_keyのUNIQUE制約を重複送信防止の最終防衛線とする。

### アソシエーション

- `belongs_to :store`
- `belongs_to :stock_location_item`
- `belongs_to :reported_by, class_name: "Staff"`
- `belongs_to :entered_by, class_name: "Staff"`
- `belongs_to :voided_by, class_name: "Staff", optional: true`
- `belongs_to :replaces_loss_record, class_name: "LossRecord", optional: true`

stock_location_items側：

- `has_many :loss_records`

staffs側：

- `has_many :reported_loss_records, class_name: "LossRecord", foreign_key: :reported_by_id`
- `has_many :entered_loss_records, class_name: "LossRecord", foreign_key: :entered_by_id`
- `has_many :voided_loss_records, class_name: "LossRecord", foreign_key: :voided_by_id`

### 削除方針

- ロス記録は物理削除しない。
- 訂正・取消履歴を保持する。
- 関連先の物理削除は原則RESTRICTとする。

## 14. `audit_events`（操作・変更監査履歴）

**責務：** 在庫訂正・権限変更等の重要操作を追跡する。

| カラム名      | 日本語名                 | 型          | NULL | DEFAULT | Option      |
| ------------- | ------------------------ | ----------- | ---- | ------- | ----------- |
| `store_id`    | 店舗ID                   | bigint      | 不可 | —       | FK → stores |
| `actor_id`    | 操作スタッフID           | bigint      | 可   | —       | FK → staffs |
| `action`      | 操作種別                 | string(50)  | 不可 | —       | INDEX候補   |
| `target_type` | 対象テーブル・モデル種別 | string(100) | 不可 | —       |             |
| `target_id`   | 対象レコードID           | bigint      | 不可 | —       |             |
| `before_data` | 変更前データ             | jsonb       | 可   | —       |             |
| `after_data`  | 変更後データ             | jsonb       | 可   | —       |             |
| `reason`      | 操作・訂正理由           | text        | 可   | —       |             |
| `request_id`  | リクエスト識別子         | string(100) | 可   | —       |             |
| `created_at`  | 記録日時                 | datetime    | 不可 | —       |             |

**制約・備考**

- `INDEX(store_id, created_at)`
- `INDEX(target_type, target_id)`
- 追記型として扱い、通常の更新・削除を許可しない。
- パスワード・認証トークン・不要な個人情報を保存しない。
- `target_type` と `target_id` は通常のFKでは参照整合性を保証できないため、対象特定と履歴保持の方式をIssue #7で検討する。

---

# Phase 2：発注・承認・納品

以下の定義は**拡張時の整合性を確認するための暫定案**。Phase 2の詳細設計で確定する。

## 15. `position_approval_authorizations`（ポジション別発注承認権限）

**責務：** スタッフが発注を承認できるポジションと、その権限の有効期間を管理する。

**作成・更新主体：** 店舗管理者・権限を持つ管理スタッフ

**実装フェーズ：** Phase 2

| カラム名        | 日本語名             | 型          | NULL | DEFAULT | Option             | 設定元 |
| --------------- | -------------------- | ----------- | ---- | ------- | ------------------ | ------ |
| `id`            | 承認権限ID           | bigint      | 不可 | DB採番  | PK                 | DB     |
| `store_id`      | 店舗ID               | bigint      | 不可 | —       | FK → stores        | Rails  |
| `staff_id`      | 承認対象スタッフID   | bigint      | 不可 | —       | 複合FK → staffs    | 管理者 |
| `position_id`   | 承認対象ポジションID | bigint      | 不可 | —       | 複合FK → positions | 管理者 |
| `granted_by_id` | 権限付与者ID         | bigint      | 不可 | —       | 複合FK → staffs    | Rails  |
| `starts_at`     | 権限開始日時         | datetime(6) | 不可 | —       | CHECK・EXCLUDE     | 管理者 |
| `ends_at`       | 権限終了日時         | datetime(6) | 可   | NULL    | CHECK・EXCLUDE     | 管理者 |
| `revoked_at`    | 権限失効日時         | datetime(6) | 可   | NULL    | —                  | Rails  |
| `memo`          | 付与理由             | text        | 可   | NULL    | —                  | 管理者 |
| `created_at`    | 作成日時             | datetime(6) | 不可 | —       | —                  | Rails  |
| `updated_at`    | 更新日時             | datetime(6) | 不可 | —       | —                  | Rails  |

### DB制約・INDEX

- `PRIMARY KEY(id)`
- `FOREIGN KEY(store_id) REFERENCES stores(id)`
- `FOREIGN KEY(store_id, staff_id) REFERENCES staffs(store_id, id)`
- `FOREIGN KEY(store_id, position_id) REFERENCES positions(store_id, id)`
- `FOREIGN KEY(store_id, granted_by_id) REFERENCES staffs(store_id, id)`
- `CHECK(ends_at IS NULL OR ends_at > starts_at)`
- `INDEX(store_id, staff_id, position_id)`

複合FKの参照先には、以下のUNIQUE制約を設ける。

- `staffs(store_id, id)`
- `positions(store_id, id)`

参照先の削除は原則RESTRICTとする。

### 有効期間のルール

- `starts_at` は権限開始日時。
- `ends_at` は権限終了日時。
- `ends_at = NULL` は終了日時未定。
- 開始日時を含み、終了日時は含まない。
- `revoked_at = NULL` は未失効を意味する。
- `revoked_at` が設定された権限は失効済みとする。
- 失効日時はRailsがサーバー時刻で記録する。
- 失効済みの権限レコードは履歴として保持する。
- 失効済みレコードの再有効化はせず、新しい権限を登録する。

### 期間の重複防止

同一スタッフ・同一ポジションに対して、
未失効の権限の有効期間が重複することを禁止する。

PostgreSQLの `tsrange` と `EXCLUDE` 制約を採用する。

比較対象：

- `staff_id`
- `position_id`
- `tsrange(starts_at, ends_at, '[)')`

条件：

- `revoked_at IS NULL` の権限を対象とする。

実装では `btree_gist` 拡張を使用する。

失効済みの権限は重複制約の対象外とするが、
監査履歴は削除しない。

日時は原則UTCで保存する。
Railsのdatetime(6)をPostgreSQLのtimestamp without time zone
として保存する前提でtsrangeを使用する。

### 発注承認の業務ルール

- 1人のスタッフは複数ポジションの承認権限を持てる。
- 1つのポジションに複数の承認者を設定できる。
- 担当していないポジションでも、明示的に権限が付与されていれば承認できる。
- position_assignmentsの登録だけでは承認できない。
- 承認権限の付与・失効はmanagerまたはadminが行う。
- 権限付与者自身の所属店舗と、対象スタッフ・ポジションの店舗が一致する必要がある。
- 承認スタッフはactive状態でなければならない。
- 店舗がactive状態でなければ承認できない。
- adminであることだけを理由に、全ポジションの発注承認権限を自動付与しない。
- 発注作成者による自己承認は、adminを含めて禁止する。

### 発注承認時の検証条件

承認処理では以下をすべて確認する。

1. 認証済みスタッフが有効である。
2. 所属店舗が有効である。
3. 発注書の店舗とスタッフの所属店舗が一致する。
4. 発注書の対象ポジションに承認権限が存在する。
5. starts_atが現在日時以前である。
6. ends_atがNULL、または現在日時より後である。
7. revoked_atがNULLである。
8. 発注作成者と承認者が異なる。
9. 発注書が承認可能な状態である。

### 監査

以下の操作をaudit_eventsへ記録する。

- 承認権限の付与
- 承認権限の変更
- 承認権限の失効
- 発注の承認操作

権限変更の監査では、実行者・対象者・対象ポジション・
変更日時・変更前後の情報を追跡可能にする。

### Phase 1との境界

- 本テーブルのMigration・Model・Service・APIはPhase 2で実装する。
- Phase 1ではポジション担当履歴のみを管理する。
- 発注承認権限はposition_assignmentsから自動生成しない。
- Phase 1の認証基盤とスタッフの有効状態はPhase 2でも利用する。

### アソシエーション

- `belongs_to :store`
- `belongs_to :staff`
- `belongs_to :position`
- `belongs_to :granted_by, class_name: "Staff"`

Rails Modelでの複合FKと関連定義の具体的な設定は、
Phase 2の実装Issueで確認する。

## 16. `stock_transfers`（在庫移動履歴）

**責務：** 食材を保管場所間で移動した履歴を記録する。

| カラム名           | 日本語名           | 型            | NULL | DEFAULT | Option               |
| ------------------ | ------------------ | ------------- | ---- | ------- | -------------------- |
| `store_id`         | 店舗ID             | bigint        | 不可 | —       | FK → stores          |
| `item_id`          | 食材ID             | bigint        | 不可 | —       | FK → items           |
| `from_location_id` | 移動元保管場所ID   | bigint        | 不可 | —       | FK → stock_locations |
| `to_location_id`   | 移動先保管場所ID   | bigint        | 不可 | —       | FK → stock_locations |
| `quantity`         | 移動数量           | decimal(12,3) | 不可 | —       | CHECK > 0            |
| `moved_by_id`      | 移動実施スタッフID | bigint        | 不可 | —       | FK → staffs          |
| `moved_at`         | 移動日時           | datetime      | 不可 | —       |                      |
| `memo`             | 備考               | text          | 可   | —       |                      |

**制約・備考**

- `CHECK(quantity > 0)`
- `CHECK(from_location_id <> to_location_id)`
- 同一店舗内の保管場所移動を原則とする。
- 移動前後で店舗全体の在庫が増減しないようにする。

## 17. `suppliers`（仕入先マスタ）

**責務：** 食材の仕入先と連絡先を管理する。

| カラム名       | 日本語名     | 型          | NULL | DEFAULT | Option      |
| -------------- | ------------ | ----------- | ---- | ------- | ----------- |
| `store_id`     | 店舗ID       | bigint      | 不可 | —       | FK → stores |
| `code`         | 仕入先コード | string(50)  | 不可 | —       |             |
| `name`         | 仕入先名     | string(150) | 不可 | —       |             |
| `fax_number`   | FAX番号      | string(30)  | 可   | —       |             |
| `phone_number` | 電話番号     | string(30)  | 可   | —       |             |
| `active`       | 取引有効状態 | boolean     | 不可 | true    |             |
| `memo`         | 備考         | text        | 可   | —       |             |

**制約・備考**

- `UNIQUE(store_id, code)`
- 過去の発注履歴を維持するため、取引停止は原則無効化とする。

## 18. `supplier_items`（仕入先別取扱商品）

**責務：** 仕入先ごとの取扱商品・発注単位・単位換算を管理する。

| カラム名                  | 日本語名                  | 型            | NULL | DEFAULT | Option         |
| ------------------------- | ------------------------- | ------------- | ---- | ------- | -------------- |
| `supplier_id`             | 仕入先ID                  | bigint        | 不可 | —       | FK → suppliers |
| `item_id`                 | 食材ID                    | bigint        | 不可 | —       | FK → items     |
| `supplier_item_code`      | 仕入先商品コード          | string(100)   | 可   | —       |                |
| `order_unit`              | 発注単位                  | string(30)    | 不可 | —       |                |
| `base_qty_per_order_unit` | 1発注単位あたりの基準数量 | decimal(12,3) | 不可 | —       | CHECK > 0      |
| `active`                  | 取扱状態                  | boolean       | 不可 | true    |                |

**制約・備考**

- `CHECK(base_qty_per_order_unit > 0)`
- 1社から同じ食材を異なる荷姿で仕入れるケースを考慮する。
- 荷姿の一意性・コードの重複条件はPhase 2で確定する。
- 発注時には換算数量をスナップショット保存する。

## 19. `purchase_orders`（発注書）

**責務：** 発注先・作成者・取りまとめポジション・承認状態を管理する。

| カラム名               | 日本語名             | 型         | NULL | DEFAULT | Option         |
| ---------------------- | -------------------- | ---------- | ---- | ------- | -------------- |
| `store_id`             | 店舗ID               | bigint     | 不可 | —       | FK → stores    |
| `supplier_id`          | 仕入先ID             | bigint     | 不可 | —       | FK → suppliers |
| `ordering_position_id` | 発注担当ポジションID | bigint     | 不可 | —       | FK → positions |
| `requested_by_id`      | 発注作成スタッフID   | bigint     | 不可 | —       | FK → staffs    |
| `status`               | 発注状態             | string(30) | 不可 | draft   | CHECK候補      |
| `expected_delivery_on` | 納品予定日           | date       | 可   | —       |                |
| `sent_at`              | FAX送信済み日時      | datetime   | 可   | —       |                |
| `fax_number_snapshot`  | 発注時FAX番号        | string(30) | 可   | —       |                |
| `lock_version`         | 更新バージョン       | integer    | 不可 | 0       | 楽観ロック     |

**制約・備考**

- 発注・承認・送信・納品の状態遷移を定義する。
- 異なるポジションからの追加発注依頼を集約する業務を考慮する。
- 発注確定時点では在庫数量を増加させない。
- 過去の送信先を保持するためFAX番号のスナップショットを検討する。

## 20. `purchase_order_lines`（発注明細）

**責務：** 発注書に含まれる商品・数量を管理する。

| カラム名                   | 日本語名             | 型            | NULL | DEFAULT | Option               |
| -------------------------- | -------------------- | ------------- | ---- | ------- | -------------------- |
| `purchase_order_id`        | 発注書ID             | bigint        | 不可 | —       | FK → purchase_orders |
| `supplier_item_id`         | 仕入先別商品ID       | bigint        | 不可 | —       | FK → supplier_items  |
| `requested_by_position_id` | 依頼元ポジションID   | bigint        | 可   | —       | FK → positions       |
| `ordered_quantity`         | 発注数量             | decimal(12,3) | 不可 | —       | CHECK > 0            |
| `order_unit_snapshot`      | 発注時単位           | string(30)    | 不可 | —       |                      |
| `base_qty_snapshot`        | 発注時の単位換算数量 | decimal(12,3) | 不可 | —       | CHECK > 0            |
| `unit_price`               | 単価                 | decimal(12,2) | 可   | —       |                      |
| `memo`                     | 明細備考             | text          | 可   | —       |                      |

**制約・備考**

- `CHECK(ordered_quantity > 0)`
- 依頼元が複数の場合の集約方法はPhase 2で検討する。
- 数量・単位換算情報は発注後にマスタ変更されても変化しないよう保持する。

## 21. `order_approval_events`（発注承認履歴）

**責務：** 承認・差戻し等の操作履歴を保存する。

| カラム名            | 日本語名             | 型         | NULL | DEFAULT | Option               |
| ------------------- | -------------------- | ---------- | ---- | ------- | -------------------- |
| `purchase_order_id` | 発注書ID             | bigint     | 不可 | —       | FK → purchase_orders |
| `actor_id`          | 操作スタッフID       | bigint     | 不可 | —       | FK → staffs          |
| `action`            | 操作種別             | string(30) | 不可 | —       |                      |
| `comment`           | コメント・差戻し理由 | text       | 可   | —       |                      |
| `created_at`        | 操作日時             | datetime   | 不可 | —       |                      |

**制約・備考**

- 承認・差戻し履歴を追記して保持する。
- 自己承認を原則禁止する。
- スタッフの有効な承認権限を承認処理時に確認する。

## 22. `receipts`（納品・検品記録）

**責務：** 発注書に対する納品実績と検品状態を管理する。

| カラム名            | 日本語名           | 型         | NULL | DEFAULT | Option               |
| ------------------- | ------------------ | ---------- | ---- | ------- | -------------------- |
| `purchase_order_id` | 発注書ID           | bigint     | 不可 | —       | FK → purchase_orders |
| `received_by_id`    | 検品担当スタッフID | bigint     | 不可 | —       | FK → staffs          |
| `received_at`       | 納品日時           | datetime   | 不可 | —       |                      |
| `status`            | 検品状態           | string(30) | 不可 | draft   |                      |
| `confirmed_at`      | 検品確定日時       | datetime   | 可   | —       |                      |
| `lock_version`      | 更新バージョン     | integer    | 不可 | 0       | 楽観ロック           |

**制約・備考**

- 1件の発注に対して複数回の納品を記録できる。
- 確定済み検品だけを入庫実績に反映する。
- 二重確定や二重計上を防止する。

## 23. `receipt_lines`（納品・検品明細）

**責務：** 商品ごとの受入数量・不受入数量を記録する。

| カラム名                 | 日本語名         | 型            | NULL | DEFAULT | Option                    |
| ------------------------ | ---------------- | ------------- | ---- | ------- | ------------------------- |
| `receipt_id`             | 納品記録ID       | bigint        | 不可 | —       | FK → receipts             |
| `purchase_order_line_id` | 発注明細ID       | bigint        | 不可 | —       | FK → purchase_order_lines |
| `stock_location_id`      | 入庫先保管場所ID | bigint        | 不可 | —       | FK → stock_locations      |
| `accepted_quantity`      | 受入数量         | decimal(12,3) | 不可 | 0       | CHECK >= 0                |
| `rejected_quantity`      | 不受入数量       | decimal(12,3) | 不可 | 0       | CHECK >= 0                |
| `memo`                   | 検品メモ         | text          | 可   | —       |                           |

**制約・備考**

- `CHECK(accepted_quantity >= 0)`
- `CHECK(rejected_quantity >= 0)`
- 発注数量と累計受入数量の整合性を検証する。
- 受入数は発注単位から基準単位へ換算して集計する。
- 同じ発注明細を複数回に分けて納品できる。

---

# Phase 3：分析・追加セキュリティ

以下の定義は暫定案。Phase 3の詳細設計で確定する。

## 24. `supplier_closures`（仕入先休業日）

**責務：** 仕入先の営業・休業情報を管理する。

| カラム名      | 日本語名 | 型          | NULL | DEFAULT | Option         |
| ------------- | -------- | ----------- | ---- | ------- | -------------- |
| `supplier_id` | 仕入先ID | bigint      | 不可 | —       | FK → suppliers |
| `closed_on`   | 休業日   | date        | 不可 | —       |                |
| `reason`      | 休業理由 | string(150) | 可   | —       |                |

**制約・備考**

- `UNIQUE(supplier_id, closed_on)`
- GW、お盆、正月などの休業日を登録する。
- 発注段階で必要になればPhase 2に前倒しする。

## 25. `supplier_delivery_rules`（仕入先納品ルール）

**責務：** 仕入先の発注締切・納品条件を管理する。

| カラム名      | 日本語名     | 型      | NULL | DEFAULT | Option         |
| ------------- | ------------ | ------- | ---- | ------- | -------------- |
| `supplier_id` | 仕入先ID     | bigint  | 不可 | —       | FK → suppliers |
| `weekday`     | 曜日         | integer | 不可 | —       | CHECK 0〜6     |
| `cutoff_time` | 発注締切時刻 | time    | 可   | —       |                |
| `lead_days`   | 納品所要日数 | integer | 不可 | 0       | CHECK >= 0     |
| `active`      | 有効状態     | boolean | 不可 | true    |                |
| `memo`        | 備考         | text    | 可   | —       |                |

**制約・備考**

- `CHECK(weekday BETWEEN 0 AND 6)`
- `CHECK(lead_days >= 0)`
- 複数の納品便や締切がある場合、曜日単位のUNIQUE制約は再検討する。

## 26. `busy_periods`（繁忙期管理）

**責務：** GW・お盆・年末年始などの分析対象期間を管理する。

| カラム名    | 日本語名 | 型          | NULL | DEFAULT | Option      |
| ----------- | -------- | ----------- | ---- | ------- | ----------- |
| `store_id`  | 店舗ID   | bigint      | 不可 | —       | FK → stores |
| `name`      | 繁忙期名 | string(100) | 不可 | —       |             |
| `starts_on` | 開始日   | date        | 不可 | —       |             |
| `ends_on`   | 終了日   | date        | 不可 | —       |             |
| `memo`      | 備考     | text        | 可   | —       |             |

**制約・備考**

- `CHECK(ends_on >= starts_on)`
- 発注履歴・在庫実績と日付範囲で照合する。
- 年度や期間ごとの実績比較を想定する。

## 27. `auth_sessions`（認証セッション）

**責務：** Googleログイン後のZAIKO独自セッションを端末ごとに管理する。

**Phase 3で実装する。Phase 1では作成しない。**

| カラム名       | 日本語名                     | 型          | NULL | DEFAULT | Option      |
| -------------- | ---------------------------- | ----------- | ---- | ------- | ----------- |
| `staff_id`     | スタッフID                   | bigint      | 不可 | —       | FK → staffs |
| `token_digest` | セッショントークンのハッシュ | string(255) | 不可 | —       | UNIQUE      |
| `device_label` | 端末識別用表示名             | string(100) | 可   | —       |             |
| `issued_at`    | 発行日時                     | datetime    | 不可 | —       |             |
| `expires_at`   | 有効期限                     | datetime    | 不可 | —       | INDEX候補   |
| `revoked_at`   | 失効日時                     | datetime    | 可   | —       |             |
| `last_used_at` | 最終使用日時                 | datetime    | 可   | —       |             |

**制約・備考**

- `UNIQUE(token_digest)`
- セッショントークンを平文でDB保存しない。
- 端末ごとの失効・全端末の失効に対応する。
- Phase 1で使用する認証方式との移行方法を検討する。
- 対応Issue：#9

---

## 28. 全体の整合性ルール

### 店舗境界

同じ店舗に属する食材・スタッフ・ポジション・保管場所だけを紐付けられるようにする。

通常の単独FKだけでは、異なる店舗のレコードを組み合わせる問題までは防げないため、複合FK等のDB設計とRails側の認可検証を検討する。

### 削除方針

- 履歴のある食材・スタッフ・保管場所は原則として無効化する。
- 棚卸・発注・納品・監査の履歴は保持する。
- 関連する履歴が存在するレコードの不用意な物理削除を禁止する。

### 数量

- 数量は原則 `decimal(12,3)` を候補とする。
- 負数を禁止する。
- 在庫未確認を数量0として扱わない。
- 単位換算の精度と端数処理は詳細設計で確定する。

### 在庫と履歴

- 食材の使用ポジション数で実在庫数量を増やさない。
- 棚卸の責任ポジションと実際の作業者を分離する。
- 重要な変更は監査履歴を残す。
- 確定済み実績を訂正する場合、元の値と変更理由を追跡できるようにする。
- 発注しただけでは在庫を増やさない。検品確定済みの数量を入庫実績とする。

## 29. 詳細設計で確定する事項

- [ ] Phase 1のテーブル・カラム・制約の確定（Issue #4〜#7）
- [ ] 共有食材の棚卸責任と担当交代時の履歴保持
- [ ] 棚卸の作成単位と1日の実施回数
- [ ] 棚卸確定時に必要な入力項目
- [ ] 数量単位と単位換算
- [ ] 店舗をまたぐ不正なFKの防止方法
- [ ] 外部認証とAPI認可の方式
- [ ] Phase 2の発注集約・承認・納品データ構造
- [ ] Phase 3の分析・セッション管理
- [ ] Model Spec・Service Spec・Request Specのテスト観点

**注意：本書は実装前の設計案であり、Migrationが作成されたことやDB構造が確定したことを意味しない。**

## 30. テーブル別の作成・更新主体

各テーブルの作成・更新を行う主体を以下に整理する。
権限の具体的な判定条件は、後続Issueで詳細化する。

| Phase | テーブル                           | 作成主体                                 | 更新主体                            |
| ----- | ---------------------------------- | ---------------------------------------- | ----------------------------------- |
| 1     | `stores`                           | システム管理者                           | システム管理者                      |
| 1     | `staffs`                           | 本人の自己登録処理・初期設定・店舗管理者 | 本人（許可項目）・店舗管理者・Rails |
| 1     | `oauth_identities`                 | 認証・紐付け処理                         | 認証・紐付け管理処理                |
| 1     | `positions`                        | 店舗管理者                               | 店舗管理者                          |
| 1     | `position_assignments`             | 店舗管理者                               | 店舗管理者                          |
| 1     | `items`                            | 店舗管理者                               | 店舗管理者                          |
| 1     | `position_items`                   | 店舗管理者                               | 店舗管理者                          |
| 1     | `stock_locations`                  | 店舗管理者                               | 店舗管理者                          |
| 1     | `stock_location_items`             | 店舗管理者                               | 店舗管理者                          |
| 1     | `inventory_counts`                 | 権限を持つスタッフ                       | 権限を持つスタッフ・確定者          |
| 1     | `inventory_count_lines`            | 棚卸実施・入力スタッフ                   | 棚卸実施・入力スタッフ              |
| 1     | `loss_records`                     | 権限を持つスタッフ                       | 権限を持つスタッフ                  |
| 1     | `audit_events`                     | Railsの監査記録処理                      | 原則更新禁止                        |
| 2     | `position_approval_authorizations` | 店舗管理者                               | 店舗管理者                          |
| 2     | `stock_transfers`                  | 権限を持つスタッフ                       | 原則訂正履歴で対応                  |
| 2     | `suppliers`                        | 店舗管理者                               | 店舗管理者                          |
| 2     | `supplier_items`                   | 店舗管理者                               | 店舗管理者                          |
| 2     | `purchase_orders`                  | 発注担当スタッフ                         | 発注担当者・承認処理                |
| 2     | `purchase_order_lines`             | 発注担当スタッフ                         | 発注確定前の担当者                  |
| 2     | `order_approval_events`            | Railsの承認処理                          | 更新禁止                            |
| 2     | `receipts`                         | 検品担当スタッフ                         | 検品担当者・確定処理                |
| 2     | `receipt_lines`                    | 検品担当スタッフ                         | 検品確定前の担当者                  |
| 3     | `supplier_closures`                | 店舗管理者                               | 店舗管理者                          |
| 3     | `supplier_delivery_rules`          | 店舗管理者                               | 店舗管理者                          |
| 3     | `busy_periods`                     | 店舗管理者                               | 店舗管理者                          |
| 3     | `auth_sessions`                    | Rails認証処理                            | Rails認証・失効処理                 |

### 設計上の注意

- マスタの作成・更新は、管理権限を持つスタッフに限定する。
- Google認証情報の紐付けは、事前に許可されたスタッフに対してのみ実行する。
- 棚卸は原則として担当ポジションのスタッフが行うが、代理実施も想定する。
- 確定済み棚卸は直接上書きせず、訂正理由と監査履歴を残す。
- 発注・納品は確定前後で更新できる範囲を変える。
- 監査・承認イベントは追記型とし、通常の編集・削除を禁止する。
- 他店舗のデータは、作成・更新・参照のいずれも許可しない。
- 各カラムの具体的な設定元（画面入力・Rails自動設定など）はIssue #4〜#7で確定する。

## 31. 既存店舗へのスタッフサインアップ

Phase 1では、システムに登録済みの1店舗に対して、
スタッフ本人がアプリから自己登録する方式を採用する。

### サインアップ

1. スタッフがアプリで新規登録を選択する。
2. Google OpenID Connectで本人認証する。
3. Railsが認証結果を検証する。
4. 検証済みissuer・subjectを照合する。
5. 登録済みなら通常ログインへ誘導する。
6. 未登録ならスタッフ名・店舗参加コードを入力する。
7. Railsが唯一の運用店舗を特定する。
8. 参加コード・有効期限・店舗状態を検証する。
9. DBトランザクション内で以下を登録する。
   - staffs（role_code=staff、status=active）
   - oauth_identities
   - audit_events
10. RailsがZAIKO API用認証トークンを発行する。
11. 一般スタッフとしてログインを完了する。

### 登録ルール

- 店舗作成機能はPhase 1の対象外とする。
- 店舗選択画面を設けない。
- 所属店舗はRailsが決定する。
- クライアント指定のstore_idは信用しない。
- 管理者権限を自己登録で取得できない。
- Googleのメールアドレスだけで既存スタッフに紐付けない。
- 同一Googleアカウントの重複登録を禁止する。
- 停止中の店舗には登録できない。
- 登録完了後は再度サインインを求めず利用開始できる。
- 登録処理失敗時は全体をロールバックする。

### 初期管理者

- 初期管理者はシステム導入時に作成する。
- 初期管理者のGoogleアカウント紐付けも安全な初期設定で実施する。
- 一般向けのサインアップ処理からadminを作成しない。

### 将来の拡張

- storesテーブルとstore_idは維持する。
- Phase 1では1スタッフは1店舗所属とする。
- 複数店舗の登録・選択・兼務は将来の別Issueで設計する。

## 32. Phase 1 API認証・ログアウト設計

**対応Issue：** #4

**実装フェーズ：** Phase 1-A

### 32.1 認証方式

- Google OpenID Connectによって本人認証を行う。
- RailsはGoogleの認証結果をサーバー側で検証する。
- 検証済みのissuer・subjectでoauth_identitiesを照合する。
- RailsはZAIKO API専用の署名付きトークンを発行する。
- Google ID Tokenを業務APIの認証トークンとして使い回さない。
- 署名にはRailsのActiveSupport::MessageVerifierを使用する。
- トークンの有効期限は8時間とする。
- Phase 1ではリフレッシュトークンを発行しない。

### 32.2 APIトークンの内容

署名対象のデータ：

- staff_id
- store_id
- oauth_identity_id
- auth_version

用途は `api_access` とする。

署名対象に氏名・メールアドレス・パスワード等は含めない。

role_codeはトークンに固定せず、
各APIでDBの最新値を取得する。

### 32.3 APIアクセス時の認証

保護対象APIでは、以下を検証する。

1. AuthorizationヘッダーにBearer Tokenが存在する。
2. トークンの署名が正しい。
3. 用途がapi_accessである。
4. 有効期限内である。
5. staff_idに対応するスタッフが存在する。
6. スタッフのstatusがactiveである。
7. 所属店舗のactiveがtrueである。
8. トークンのstore_idと所属店舗が一致する。
9. auth_versionがDBの最新値と一致する。
10. oauth_identity_idが同じスタッフに紐付いている。
11. リクエスト対象の店舗が認可された店舗と一致する。
12. 操作に必要な権限がある。

スタッフID・店舗ID・権限を
クライアントのリクエスト値から決定しない。

### 32.4 ログアウト

1. アプリからRailsのログアウトAPIを呼び出す。
2. Railsが現在のトークンを検証する。
3. staffs.auth_versionを原子的に加算する。
4. 成功時、同じスタッフの発行済みトークンをすべて失効する。
5. アプリ側のSecureStoreからトークンを削除する。
6. ログイン画面へ遷移する。

Phase 1はスタッフ単位の全端末失効とする。

特定端末だけを失効する機能はPhase 3で実装する。

サーバーとの通信に失敗し、失効処理が完了しなかった場合は、
ローカルで削除してもサーバー側の失効は保証しない。

### 32.5 スタッフの利用停止・退職

- suspendedまたはretiredのスタッフはAPIを利用できない。
- 各APIでDBの最新statusを検証する。
- 管理者による強制ログアウトはauth_versionを更新する。
- 店舗が無効化されている場合もAPIを拒否する。
- 変更前から実行中のリクエストまで即時中断することは保証しない。

### 32.6 再ログイン

- トークンの期限切れ時は再認証を求める。
- Google認証に成功した場合でもスタッフと店舗を再検証する。
- 有効なスタッフにのみ新しいトークンを発行する。
- 登録済みスタッフは店舗参加コードを再入力しない。
- 既存のGoogleアカウント紐付けを再利用する。

### 32.7 APIエラー

| 条件                   | HTTPステータス |
| ---------------------- | -------------- |
| トークン未指定         | 401            |
| 署名不正・期限切れ     | 401            |
| スタッフ無効化         | 401            |
| 権限不足               | 403            |
| 他店舗の個別データ参照 | 原則404        |

エラーレスポンスに不要なスタッフ情報・店舗情報を含めない。

### 32.8 セキュリティ対策

- API通信はHTTPSに限定する。
- トークンはExpo SecureStoreに保存する。
- トークン原文をDBやログに保存しない。
- Googleのアクセストークン・リフレッシュトークンも原則保存しない。
- 認証処理と参加コード検証にはレート制限を設ける。
- トークンの署名鍵はサーバー側で安全に管理する。
- 署名鍵の変更時の失効・移行方法を実装時に検証する。

### 32.9 Phase 3との境界

- auth_sessionsはPhase 3（Issue #9）で実装する。
- Phase 1では端末別のセッション記録を保持しない。
- 特定端末だけの強制ログアウトは実装しない。
- 発行済みトークンが漏えいした場合、
  失効処理または有効期限到来まで悪用される可能性がある。
- この残存リスクを受入レビューで評価する。

## 33. Issue #5 在庫照会・帳票出力のデータ対応

**対応要件：** FR-01、FR-02

### 出力項目とデータ取得元

| 出力項目           | データ取得元                                        | 種別   |
| ------------------ | --------------------------------------------------- | ------ |
| 食材コード         | items.code                                          | DB     |
| 食材名             | items.name                                          | DB     |
| 基準単位           | items.base_unit                                     | DB     |
| 保管場所コード     | stock_locations.code                                | DB     |
| 保管場所名         | stock_locations.name                                | DB     |
| 棚卸責任ポジション | positions.name                                      | DB関連 |
| 前回確認数量       | 最新確定済みinventory_count_lines.measured_quantity | 算出   |
| 最終確認日時       | inventory_counts.confirmed_at                       | DB関連 |
| 実施者             | inventory_count_lines.counted_by_id → staffs        | DB関連 |
| 食材状態           | inventory_count_lines.quality_status                | DB関連 |
| 状態メモ           | inventory_count_lines.quality_note                  | DB関連 |
| 実数記入欄         | PDF出力時に空欄を生成                               | 表示用 |
| 店舗合計数量       | 保管場所別の最新確定数量を集計                      | 算出   |

### PDF出力

- 棚卸記入用と実績確認用の帳票を区別する。
- 棚卸記入用は前回数量を初期状態で非表示にする。
- 前回数量の表示・非表示は切り替え可能とする。
- 食材名、基準単位、実数記入欄、状態・メモ欄を設ける。
- 保管場所別と責任ポジション別で出力できる。

### CSV出力

- DBに存在する項目と算出項目を区別する。
- 対象営業日と最終確認日時を区別する。
- 未確認数量は0へ変換せず、空欄等で表現する。
- 日本語の文字化けを防ぐ文字コードを選定する。
- CSV Injection対策を実施する。
- 出力時に店舗・ポジションの認可を検証する。

### 表示順

1. stock_locations.sort_order
2. stock_locations.id
3. stock_location_items.sort_order
4. stock_location_items.id

ポジション別の出力では、
responsible_position_idによる棚卸責任対象と、
position_itemsによる使用食材の検索を区別する。

### Phase 1と後続Issueの境界

本Issueでは出力に必要なデータと取得元を定義する。
実際のPDF・CSV生成処理と棚卸履歴の取得方法は、
後続の実装Issueで具体化する。
