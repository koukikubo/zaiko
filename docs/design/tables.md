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

**責務：** 店舗の基本情報を管理する。

| カラム名   | 日本語名     | 型          | NULL | DEFAULT    | Option |
| ---------- | ------------ | ----------- | ---- | ---------- | ------ |
| `code`     | 店舗コード   | string(30)  | 不可 | —          | UNIQUE |
| `name`     | 店舗名       | string(150) | 不可 | —          |        |
| `timezone` | タイムゾーン | string(50)  | 不可 | Asia/Tokyo |        |
| `active`   | 利用状態     | boolean     | 不可 | true       |        |

**制約・備考**

- `code` は店舗識別に使用し、一意とする。
- 他の業務データから参照されるため、通常は物理削除しない。
- 初期運用は1店舗だが、将来の複数店舗に対応できる構造とする。

## 3. `staffs`（スタッフマスタ）

**責務：** スタッフの在籍状態・基本権限を管理する。

| カラム名       | 日本語名       | 型          | NULL | DEFAULT | Option      |
| -------------- | -------------- | ----------- | ---- | ------- | ----------- |
| `store_id`     | 所属店舗ID     | bigint      | 不可 | —       | FK → stores |
| `code`         | スタッフコード | string(30)  | 不可 | —       |             |
| `name`         | スタッフ名     | string(100) | 不可 | —       |             |
| `role_code`    | 基本権限       | string(30)  | 不可 | staff   | CHECK候補   |
| `status`       | 在籍・利用状態 | string(30)  | 不可 | active  | CHECK候補   |
| `retired_at`   | 退職日時       | datetime    | 可   | —       |             |
| `lock_version` | 更新バージョン | integer     | 不可 | 0       | 楽観ロック  |

**制約・備考**

- `UNIQUE(store_id, code)`
- `role_code` の候補：`staff / manager / admin`
- `status` の候補：`active / suspended / retired`
- 退職したスタッフも棚卸・承認履歴のために残す。
- Google認証の情報は `oauth_identities` へ分離する。

## 4. `oauth_identities`（外部認証アカウント）

**責務：** GoogleアカウントとZAIKOのスタッフを紐付ける。

| カラム名         | 日本語名               | 型          | NULL | DEFAULT | Option      |
| ---------------- | ---------------------- | ----------- | ---- | ------- | ----------- |
| `staff_id`       | スタッフID             | bigint      | 不可 | —       | FK → staffs |
| `provider`       | 認証プロバイダー       | string(30)  | 不可 | google  |             |
| `issuer`         | トークン発行元         | string(255) | 不可 | —       |             |
| `subject`        | 外部ユーザー識別子     | string(255) | 不可 | —       |             |
| `email_snapshot` | 紐付け時メールアドレス | string(255) | 可   | —       |             |
| `linked_at`      | 紐付け日時             | datetime    | 不可 | —       |             |

**制約・備考**

- `UNIQUE(issuer, subject)`
- Googleアカウントの識別にはメールアドレスではなく、検証済みID Tokenの `iss` と `sub` を使用する。
- メールアドレスが一致しただけでは自動的にスタッフへ紐付けない。
- Googleのアクセストークンやリフレッシュトークンを安易にDB保存しない。
- Phase 1では独立した `auth_sessions` テーブルを作らない。

## 5. `positions`（ポジションマスタ）

**責務：** 揚場・焼場・造り場などの担当区分を管理する。

| カラム名     | 日本語名         | 型          | NULL | DEFAULT | Option      |
| ------------ | ---------------- | ----------- | ---- | ------- | ----------- |
| `store_id`   | 店舗ID           | bigint      | 不可 | —       | FK → stores |
| `code`       | ポジションコード | string(30)  | 不可 | —       |             |
| `name`       | ポジション名     | string(100) | 不可 | —       |             |
| `sort_order` | 表示順           | integer     | 不可 | 0       |             |
| `active`     | 有効状態         | boolean     | 不可 | true    |             |

**制約・備考**

- `UNIQUE(store_id, code)`
- 担当履歴を保持するため、使用済みポジションは原則無効化で対応する。

## 6. `position_assignments`（ポジション担当履歴）

**責務：** 誰が、いつ、どのポジションを担当したか記録する。

| カラム名         | 日本語名       | 型     | NULL | DEFAULT | Option         |
| ---------------- | -------------- | ------ | ---- | ------- | -------------- |
| `staff_id`       | スタッフID     | bigint | 不可 | —       | FK → staffs    |
| `position_id`    | ポジションID   | bigint | 不可 | —       | FK → positions |
| `started_on`     | 担当開始日     | date   | 不可 | —       |                |
| `ended_on`       | 担当終了日     | date   | 可   | —       |                |
| `assigned_by_id` | 登録スタッフID | bigint | 可   | —       | FK → staffs    |

**制約・備考**

- `CHECK(ended_on IS NULL OR ended_on >= started_on)`
- 担当期間の重複許可・禁止ルールはIssue #4で確定する。
- 担当履歴と発注承認権限は別の情報として扱う。

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
- 必要性の最終確認はIssue #5で行う。

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

## 15. `position_approval_authorizations`（ポジション別承認権限）

**責務：** スタッフが発注を承認できるポジションと有効期間を管理する。

| カラム名        | 日本語名             | 型       | NULL | DEFAULT | Option         |
| --------------- | -------------------- | -------- | ---- | ------- | -------------- |
| `staff_id`      | 承認可能スタッフID   | bigint   | 不可 | —       | FK → staffs    |
| `position_id`   | 承認対象ポジションID | bigint   | 不可 | —       | FK → positions |
| `granted_by_id` | 権限付与者ID         | bigint   | 不可 | —       | FK → staffs    |
| `starts_at`     | 権限開始日時         | datetime | 不可 | —       |                |
| `ends_at`       | 権限終了日時         | datetime | 可   | —       |                |
| `revoked_at`    | 権限失効日時         | datetime | 可   | —       |                |
| `memo`          | 付与理由等           | text     | 可   | —       |                |

**制約・備考**

- 開始・終了日時の前後関係を検証する。
- 同じスタッフ・ポジションに対する有効な権限の重複防止方法を決める。
- 現在の担当ポジションと承認権限は分離する。

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
