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

**責務：** 店舗が扱う食材・商品を一元管理する。

| カラム名     | 日本語名     | 型          | NULL | DEFAULT | Option      |
| ------------ | ------------ | ----------- | ---- | ------- | ----------- |
| `store_id`   | 店舗ID       | bigint      | 不可 | —       | FK → stores |
| `code`       | 食材コード   | string(50)  | 不可 | —       |             |
| `name`       | 食材名       | string(150) | 不可 | —       | INDEX候補   |
| `base_unit`  | 在庫基準単位 | string(20)  | 不可 | —       |             |
| `sort_order` | 表示順       | integer     | 不可 | 0       |             |
| `active`     | 有効状態     | boolean     | 不可 | true    |             |
| `memo`       | 備考         | text        | 可   | —       |             |

**制約・備考**

- `UNIQUE(store_id, code)`
- 共通倉庫と各ポジションの棚に同じ料理酒が存在しても、食材マスタは1件にする。
- 在庫数量はこのテーブルへ直接保持しない。
- 発注単位と在庫基準単位が異なる場合の換算は、後続の仕入先別商品で扱う。

## 8. `position_items`（ポジション別使用食材）

**責務：** どのポジションが、どの食材を使用するか管理する。

| カラム名      | 日本語名     | 型      | NULL | DEFAULT | Option         |
| ------------- | ------------ | ------- | ---- | ------- | -------------- |
| `position_id` | ポジションID | bigint  | 不可 | —       | FK → positions |
| `item_id`     | 食材ID       | bigint  | 不可 | —       | FK → items     |
| `active`      | 有効状態     | boolean | 不可 | true    |                |

**制約・備考**

- `UNIQUE(position_id, item_id)`
- 1つの食材は複数ポジションから使用可能。
- 発注担当・棚卸責任は管理しない。
- Phase 1で採用する。詳細なカラム・制約はIssue #5で確定する。

**採用判断：Phase 1で採用する。**

- 複数ポジションで同じ食材を使用するため、食材と使用ポジションを多対多で管理する。
- ポジションごとの食材一覧・検索・棚卸画面の表示対象を整理する目的で使用する。
- 発注担当や棚卸責任ポジションは管理しない。
- 在庫数量を保持せず、使用ポジション数による二重計上を防止する。
- 作成・更新は店舗管理者が行う。
- 詳細なカラム・制約はIssue #5で確定する。

## 9. `stock_locations`（在庫保管場所）

**責務：** 倉庫・冷蔵庫・各ポジションの棚など物理的な保管場所を管理する。

| カラム名        | 日本語名       | 型          | NULL | DEFAULT | Option      |
| --------------- | -------------- | ----------- | ---- | ------- | ----------- |
| `store_id`      | 店舗ID         | bigint      | 不可 | —       | FK → stores |
| `code`          | 保管場所コード | string(50)  | 不可 | —       |             |
| `name`          | 保管場所名     | string(100) | 不可 | —       |             |
| `location_kind` | 保管場所区分   | string(30)  | 可   | —       |             |
| `sort_order`    | 表示順         | integer     | 不可 | 0       |             |
| `active`        | 有効状態       | boolean     | 不可 | true    |             |

**制約・備考**

- `UNIQUE(store_id, code)`
- 保管場所そのものに棚卸責任ポジションを固定しない。
- 同じ保管場所でも食材ごとに責任ポジションが異なるため。

## 10. `stock_location_items`（保管場所別管理食材）

**責務：** 食材の保管場所と、通常の棚卸責任ポジションを管理する。

| カラム名                  | 日本語名             | 型      | NULL | DEFAULT | Option               |
| ------------------------- | -------------------- | ------- | ---- | ------- | -------------------- |
| `stock_location_id`       | 保管場所ID           | bigint  | 不可 | —       | FK → stock_locations |
| `item_id`                 | 食材ID               | bigint  | 不可 | —       | FK → items           |
| `responsible_position_id` | 棚卸責任ポジションID | bigint  | 不可 | —       | FK → positions       |
| `active`                  | 棚卸対象状態         | boolean | 不可 | true    |                      |
| `sort_order`              | 棚卸表示順           | integer | 不可 | 0       |                      |

**制約・備考**

- `UNIQUE(stock_location_id, item_id)`
- 責任ポジションと実際の棚卸実施者を区別する。
- 食材と保管場所の店舗が一致することを検証する。
- 責任ポジションの変更後も過去の棚卸履歴が正しく読めるよう、記録時点の責任関係を残す方法を検討する。
- このテーブルは在庫数量を保持しない。

## 11. `inventory_counts`（日次在庫確認）

**責務：** 棚卸作業の実施日・進捗・確定情報を管理する。

以下は「1営業日・1保管場所に1つの棚卸ヘッダー」を作る案。

| カラム名            | 日本語名       | 型         | NULL | DEFAULT | Option               |
| ------------------- | -------------- | ---------- | ---- | ------- | -------------------- |
| `store_id`          | 店舗ID         | bigint     | 不可 | —       | FK → stores          |
| `stock_location_id` | 保管場所ID     | bigint     | 不可 | —       | FK → stock_locations |
| `business_date`     | 営業日         | date       | 不可 | —       | INDEX                |
| `status`            | 棚卸状態       | string(30) | 不可 | draft   | CHECK候補            |
| `opened_by_id`      | 作成スタッフID | bigint     | 不可 | —       | FK → staffs          |
| `confirmed_by_id`   | 確定スタッフID | bigint     | 可   | —       | FK → staffs          |
| `confirmed_at`      | 確定日時       | datetime   | 可   | —       |                      |
| `lock_version`      | 更新バージョン | integer    | 不可 | 0       | 楽観ロック           |

**制約・備考**

- 暫定：`UNIQUE(store_id, stock_location_id, business_date)`
- ステータス候補：`draft / confirmed`
- 再棚卸・複数回実施を認める場合は一意性を再設計する。
- 確定時に、必要な明細がすべて入力されていることをServiceで検証する。
- 1つの保管場所を複数スタッフ・ポジションが分担して棚卸できる構造とする。

## 12. `inventory_count_lines`（日次在庫確認明細）

**責務：** 食材の実測数量・品質・棚卸実施者と日時を記録する。

| カラム名                           | 日本語名                 | 型            | NULL | DEFAULT | Option                    |
| ---------------------------------- | ------------------------ | ------------- | ---- | ------- | ------------------------- |
| `inventory_count_id`               | 棚卸ID                   | bigint        | 不可 | —       | FK → inventory_counts     |
| `stock_location_item_id`           | 保管場所別食材ID         | bigint        | 不可 | —       | FK → stock_location_items |
| `measured_quantity`                | 実測数量                 | decimal(12,3) | 可   | —       | CHECK候補                 |
| `quality_status`                   | 食材状態                 | string(30)    | 可   | —       |                           |
| `quality_note`                     | 食材状態メモ             | text          | 可   | —       |                           |
| `counted_by_id`                    | 実際の棚卸スタッフID     | bigint        | 可   | —       | FK → staffs               |
| `counted_at`                       | 実際の棚卸日時           | datetime      | 可   | —       |                           |
| `entered_by_id`                    | システム入力スタッフID   | bigint        | 可   | —       | FK → staffs               |
| `responsible_position_id_snapshot` | 棚卸時の責任ポジションID | bigint        | 可   | —       | FK → positions            |
| `lock_version`                     | 更新バージョン           | integer       | 不可 | 0       | 楽観ロック                |

**制約・備考**

- `UNIQUE(inventory_count_id, stock_location_item_id)`
- `CHECK(measured_quantity >= 0)` を検討。
- 下書き中は数量NULLを許可し、**未確認と数量0を区別**する。
- 確定時には必要な数量・実施者・日時が記録されていることを検証する。
- 棚卸ヘッダーの保管場所と明細の保管場所別食材が一致するよう検証する。
- 紙で数えた人と入力した人が異なる場合を想定する。
- `responsible_position_id_snapshot` は責任ポジション変更後の履歴整合性を目的とする候補であり、最終的な保持方法はIssue #6で確定する。

## 13. `loss_records`（ロス記録）

**責務：** 食材の廃棄・破損・腐敗等の実績を管理する。

| カラム名                 | 日本語名         | 型            | NULL | DEFAULT | Option                    |
| ------------------------ | ---------------- | ------------- | ---- | ------- | ------------------------- |
| `store_id`               | 店舗ID           | bigint        | 不可 | —       | FK → stores               |
| `stock_location_item_id` | 保管場所別食材ID | bigint        | 不可 | —       | FK → stock_location_items |
| `quantity`               | ロス数量         | decimal(12,3) | 不可 | —       | CHECK > 0                 |
| `reason_code`            | ロス理由コード   | string(30)    | 不可 | —       |                           |
| `occurred_at`            | ロス発生日時     | datetime      | 不可 | —       |                           |
| `reported_by_id`         | 登録スタッフID   | bigint        | 不可 | —       | FK → staffs               |
| `memo`                   | 備考             | text          | 可   | —       |                           |

**制約・備考**

- `CHECK(quantity > 0)`
- ロス理由の候補：廃棄、破損、腐敗、その他。
- ロスと棚卸結果から使用量を算出する場合は、二重控除しない計算方法を定義する。

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
