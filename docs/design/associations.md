# ZAIKO アソシエーション定義書

- バージョン：0.1
- 対応Issue：#3〜#9
- 状態：詳細設計案・レビュー前
- 対象：全26テーブル候補
- 技術：Rails 8.1 / PostgreSQL

## 1. 設計方針

- テーブル同士の関連を外部キーで明確にする。
- アソシエーションの意味を日本語で記載する。
- 店舗をまたぐ不正な関連を禁止する。
- 履歴が存在するデータは原則として不用意に物理削除しない。
- 在庫管理では「食材」「保管場所」「使用ポジション」「棚卸責任ポジション」「棚卸実施者」を区別する。
- `auth_sessions` はPhase 3で実装する。

### 関連の表記

| 表記                | 意味                                     |
| ------------------- | ---------------------------------------- |
| 1対多               | 1件の親に複数件の子が紐付く              |
| 多対多              | 複数の親と子が中間テーブルを通じて紐付く |
| `belongs_to`        | 外部キーを保持する側の関連               |
| `has_many`          | 複数の関連レコードを取得する関連         |
| `has_many :through` | 中間テーブルを通じて取得する関連         |

以下の関連は現在の `tables.md` 案に基づく。カラム構成が変更された場合は本書も更新する。

---

## 2. Phase 1：店舗・認証・スタッフ

| 子テーブル（日本語名）             | 外部キー → 親テーブル       | 関係・説明                                                                       |
| ---------------------------------- | --------------------------- | -------------------------------------------------------------------------------- |
| `staffs`（スタッフ）               | `store_id` → `stores`       | **1対多**。1店舗に複数スタッフが所属する。1人のスタッフは1店舗に所属する         |
| `oauth_identities`（外部認証情報） | `staff_id` → `staffs`       | **1対多**。1人のスタッフは複数の外部認証情報を持てる。1件の認証情報は1人に属する |
| `positions`（ポジション）          | `store_id` → `stores`       | **1対多**。1店舗に複数の担当ポジションを登録できる                               |
| `position_assignments`（担当履歴） | `staff_id` → `staffs`       | **1対多**。1人のスタッフは複数の担当履歴を持てる                                 |
| `position_assignments`（担当履歴） | `position_id` → `positions` | **1対多**。1つのポジションには複数の担当履歴が紐付く                             |
| `position_assignments`（担当履歴） | `assigned_by_id` → `staffs` | **1対多・任意**。1人の管理スタッフが複数の担当登録を行える                       |

### 業務上の関係

`staffs` と `positions` は、`position_assignments` を介した多対多の関係を持つ。

ただし、担当期間があるため、単純な多対多ではなく担当履歴を保持する。

また、担当ポジションは発注承認権限とは別の情報である。

---

## 3. Phase 1：食材・保管場所

| 子テーブル（日本語名）                       | 外部キー → 親テーブル                                             | 関係・説明                                        |
| -------------------------------------------- | ----------------------------------------------------------------- | ------------------------------------------------- |
| `items`（食材）                              | `store_id` → `stores`                                             | **1対多**。1店舗は複数の食材を管理する            |
| `stock_locations`（保管場所）                | `store_id` → `stores`                                             | **1対多**。1店舗には複数の保管場所がある          |
| `position_items`（使用食材）                 | `store_id` → `stores`                                             | 1対多。1店舗に複数の使用食材設定が存在する        |
| `position_items`（使用食材）                 | `(store_id, position_id)` → `positions(store_id, id)`             | 1対多。所属店舗が一致するポジションのみ紐付け可能 |
| `position_items`（使用食材）                 | `(store_id, item_id)` → `items(store_id, id)`                     | 1対多。所属店舗が一致する食材のみ紐付け可能       |
| `stock_location_items`（保管場所別管理食材） | `store_id` → `stores`                                             | 1対多。店舗に属する棚卸対象を管理する             |
| `stock_location_items`（保管場所別管理食材） | `(store_id, stock_location_id)` → `stock_locations(store_id, id)` | 1対多。同一店舗の保管場所のみ関連付ける           |
| `stock_location_items`（保管場所別管理食材） | `(store_id, item_id)` → `items(store_id, id)`                     | 1対多。同一店舗の食材のみ関連付ける               |
| `stock_location_items`（保管場所別管理食材） | `(store_id, responsible_position_id)` → `positions(store_id, id)` | 1対多。同一店舗の棚卸責任ポジションのみ関連付ける |

### 業務上の関係

`positions` と `items` は `position_items` を介した多対多。

`items` と `stock_locations` は `stock_location_items` を介した多対多。

**食材の使用ポジションと棚卸責任ポジションは異なる。**

例：

- 料理酒は揚場・焼場・造り場で使用する。
- 共通倉庫の料理酒の棚卸責任は焼場。
- 揚場の棚にある料理酒の棚卸責任は揚場。
- それぞれの保管場所の数量を独立して記録する。

`stock_location_items` は在庫数量を保持せず、棚卸対象と責任ポジションを管理する。

### 保管場所の設計方針

- stock_locationsは物理的な在庫管理区画を表す。
- 保管場所自体に責任ポジションを固定しない。
- 保管場所と食材はstock_location_itemsを介して多対多で関連する。
- 同じ食材が複数保管場所に存在することを許可する。
- 1つの保管場所に複数ポジションが使用する食材を配置できる。
- 棚卸責任はstock_location_itemsのresponsible_position_idで管理する。
- 実際の棚卸実施者はinventory_count_linesで管理する。
- 同じ物理在庫を複数の保管場所として重複計上しない。
- 在庫集計は使用ポジションではなく、保管場所×食材を基準とする。

### 保管場所別管理食材の設計方針

- stock_location_itemsは保管場所×食材を一意に管理する。
- 同じ食材が複数保管場所に存在することを許可する。
- 棚卸責任は保管場所×食材ごとに設定する。
- 使用ポジションと棚卸責任ポジションは独立する。
- 棚卸責任ポジションと実際の棚卸実施者も独立する。
- 在庫集計ではposition_itemsの件数を乗算しない。
- 物理在庫の数量は棚卸記録から取得する。
- 責任変更前の履歴は棚卸明細のスナップショットで保持する。
- 同一店舗の関連は複合FKで保証する。
- 棚卸記録側の重複防止と履歴確定条件はIssue #6で設計する。

---

## 4. Phase 1：日次棚卸・在庫確認

| 子テーブル（日本語名）             | 外部キー → 親テーブル                   | 関係・説明                                               |
| ---------------------------------- | --------------------------------------- | -------------------------------------------------------- |
| `inventory_counts`（日次在庫確認） | `store_id` → `stores`                   | **1対多**。1店舗で複数回の棚卸を実施できる               |
| `inventory_counts`（日次在庫確認） | `stock_location_id` → `stock_locations` | **1対多**。1つの保管場所について複数日の棚卸を記録できる |
| `inventory_counts`（日次在庫確認） | `opened_by_id` → `staffs`               | **1対多**。1人のスタッフが複数の棚卸を作成できる         |
| `inventory_counts`（日次在庫確認） | `confirmed_by_id` → `staffs`            | **1対多・任意**。1人のスタッフが複数の棚卸を確定できる   |

| `inventory_count_lines`（棚卸明細） | `store_id` → `stores` | 1対多。同一店舗の棚卸明細を管理する |
| `inventory_count_lines`（棚卸明細） | `(store_id, stock_location_id, inventory_count_id)` → `inventory_counts(store_id, stock_location_id, id)` | 1対多。棚卸ヘッダーと保管場所の一致を保証する |
| `inventory_count_lines`（棚卸明細） | `(store_id, stock_location_id, stock_location_item_id)` → `stock_location_items(store_id, stock_location_id, id)` | 1対多。別の保管場所の食材混入を防ぐ |
| `inventory_count_lines`（棚卸明細） | `(store_id, counted_by_id)` → `staffs(store_id, id)` | 1対多・任意。実際の棚卸実施者を記録する |
| `inventory_count_lines`（棚卸明細） | `(store_id, entered_by_id)` → `staffs(store_id, id)` | 1対多・任意。システム入力者を記録する |
| `inventory_count_lines`（棚卸明細） | `(store_id, responsible_position_id_snapshot)` → `positions(store_id, id)` | 1対多。棚卸当時の責任ポジションを保持する |
| `inventory_count_lines`（棚卸明細） | `(store_id, source_line_id)` → `inventory_count_lines(store_id, id)` | 自己参照・任意。訂正・引き継ぎ元の明細を記録する |

### 設計上の注意

棚卸の作成者・確定者・実施者・入力者を区別する。

特に、後輩が先輩に代わって棚卸する場合でも、実際に数えたスタッフを保存する。

下書き中は実施者や数量が未入力の場合があるため、NULL許可と棚卸確定時の必須条件を区別して設計する。

棚卸明細の保管場所と、親となる棚卸の保管場所が一致することを保証する。

同一営業日・同一保管場所の棚卸はrevision_noで版管理する。
確定済みの過去版は保持し、最新確定版だけを在庫集計に使用する。
詳細はinventory-rules.mdを参照する。

---

## 5. Phase 1：ロス・監査

| 子テーブル（日本語名） | 外部キー → 親テーブル | 関係・説明 |
| ---------------------- | --------------------- | ---------- |

| `loss_records`（ロス記録） | `store_id` → `stores` | 1対多。店舗に属するロス実績を管理する |
| `loss_records`（ロス記録） | `(store_id, stock_location_item_id)` → `stock_location_items(store_id, id)` | 1対多。同一店舗の保管場所別食材を参照する |
| `loss_records`（ロス記録） | `(store_id, reported_by_id)` → `staffs(store_id, id)` | 1対多。実際のロス報告者を記録する |
| `loss_records`（ロス記録） | `(store_id, entered_by_id)` → `staffs(store_id, id)` | 1対多。システムへの登録者を記録する |
| `loss_records`（ロス記録） | `(store_id, voided_by_id)` → `staffs(store_id, id)` | 1対多・任意。取消者を記録する |
| `loss_records`（ロス記録） | `(store_id, replaces_loss_record_id)` → `loss_records(store_id, id)` | 自己参照・任意。訂正元記録を保持する |
| `audit_events`（監査履歴） | `store_id` → `stores` | **1対多**。1店舗に複数の操作履歴がある |
| `audit_events`（監査履歴） | `(store_id, actor_id)` → `staffs(store_id, id)` | **1対多・任意**。スタッフ操作時は必須、システム処理時はNULL |

### 監査対象の扱い

- `audit_events`は`(store_id, actor_id)`から`staffs(store_id, id)`へ複合FKを持つ。`actor_id`はシステム操作時のみNULLとする。
- `actor_kind='staff'`ではactor_idを必須とし、`actor_kind='system'`ではactor_sourceで処理を特定する。
- `target_type`・`target_id`は固定コードによる論理参照であり、通常のDB外部キーでは参照先の存在を保証できない。
- Serviceは操作時点で対象レコードの存在・店舗一致・許可する操作種別を検証する。
- 棚卸訂正の`target_id`は新確定版のID、旧確定版のIDは`before_data`へ保存する。数量比較は前後の`inventory_count_lines`から行う。
- ロス訂正の`target_id`は訂正後ロスID、訂正元ロスIDは`before_data`から参照する。
- 下書き明細の削除監査では、削除前に対象を検証し、必要な識別情報を`before_data`に残す。
- 監査イベントは追記専用。対象が退職・無効化されても監査履歴は保持する。
---

## 6. Phase 2：承認・在庫移動

| 子テーブル（日本語名）                         | 外部キー → 親テーブル                  | 関係・説明                                                     |
| ---------------------------------------------- | -------------------------------------- | -------------------------------------------------------------- |
| `position_approval_authorizations`（承認権限） | `staff_id` → `staffs`                  | **1対多**。1人のスタッフが複数ポジションの発注承認権限を持てる |
| `position_approval_authorizations`（承認権限） | `position_id` → `positions`            | **1対多**。1つのポジションに複数の承認可能スタッフを登録できる |
| `position_approval_authorizations`（承認権限） | `granted_by_id` → `staffs`             | **1対多**。1人の管理者が複数の権限を付与できる                 |
| `stock_transfers`（在庫移動）                  | `store_id` → `stores`                  | **1対多**。1店舗で複数の在庫移動を実施できる                   |
| `stock_transfers`（在庫移動）                  | `item_id` → `items`                    | **1対多**。1種類の食材を複数回移動できる                       |
| `stock_transfers`（在庫移動）                  | `from_location_id` → `stock_locations` | **1対多**。1つの保管場所が複数の移動元になる                   |
| `stock_transfers`（在庫移動）                  | `to_location_id` → `stock_locations`   | **1対多**。1つの保管場所が複数の移動先になる                   |
| `stock_transfers`（在庫移動）                  | `moved_by_id` → `staffs`               | **1対多**。1人のスタッフが複数の在庫移動を実施できる           |

---

## 7. Phase 2：仕入先・発注・納品

| 子テーブル（日本語名）              | 外部キー → 親テーブル                             | 関係・説明                                                    |
| ----------------------------------- | ------------------------------------------------- | ------------------------------------------------------------- |
| `suppliers`（仕入先）               | `store_id` → `stores`                             | **1対多**。1店舗が複数の仕入先と取引する                      |
| `supplier_items`（仕入先別商品）    | `supplier_id` → `suppliers`                       | **1対多**。1社が複数の商品を扱う                              |
| `supplier_items`（仕入先別商品）    | `item_id` → `items`                               | **1対多**。1つの食材を複数の仕入先から購入できる              |
| `purchase_orders`（発注書）         | `store_id` → `stores`                             | **1対多**。1店舗で複数の発注書を作成する                      |
| `purchase_orders`（発注書）         | `supplier_id` → `suppliers`                       | **1対多**。1社の仕入先に複数回発注できる                      |
| `purchase_orders`（発注書）         | `ordering_position_id` → `positions`              | **1対多**。1つのポジションが複数の発注書を取りまとめる        |
| `purchase_orders`（発注書）         | `requested_by_id` → `staffs`                      | **1対多**。1人のスタッフが複数の発注書を作成する              |
| `purchase_order_lines`（発注明細）  | `purchase_order_id` → `purchase_orders`           | **1対多**。1件の発注書に複数の商品明細が含まれる              |
| `purchase_order_lines`（発注明細）  | `supplier_item_id` → `supplier_items`             | **1対多**。1つの取扱商品を複数回発注できる                    |
| `purchase_order_lines`（発注明細）  | `requested_by_position_id` → `positions`          | **1対多・任意**。どのポジションから追加発注依頼があったか示す |
| `order_approval_events`（承認履歴） | `purchase_order_id` → `purchase_orders`           | **1対多**。1件の発注に複数の承認・差戻し履歴を持てる          |
| `order_approval_events`（承認履歴） | `actor_id` → `staffs`                             | **1対多**。1人のスタッフが複数の承認操作を行える              |
| `receipts`（納品・検品）            | `purchase_order_id` → `purchase_orders`           | **1対多**。1件の発注書に複数回の分納実績を紐付けられる        |
| `receipts`（納品・検品）            | `received_by_id` → `staffs`                       | **1対多**。1人のスタッフが複数回の検品を行える                |
| `receipt_lines`（検品明細）         | `receipt_id` → `receipts`                         | **1対多**。1回の納品に複数の商品明細が含まれる                |
| `receipt_lines`（検品明細）         | `purchase_order_line_id` → `purchase_order_lines` | **1対多**。1件の発注明細を複数回に分けて納品できる            |
| `receipt_lines`（検品明細）         | `stock_location_id` → `stock_locations`           | **1対多**。1つの保管場所に複数の検品明細から入庫できる        |

### 発注集約に関する注意

みりん・醤油・料理酒など、複数ポジションで使用する食材の発注は代表ポジションが取りまとめる。

ただし、発注明細1件に複数ポジションの依頼を集約する場合、単独の `requested_by_position_id` では依頼元をすべて保存できない。

複数の依頼元と数量の履歴を残す必要がある場合、Phase 2で別の依頼明細構造を検討する。

---

## 8. Phase 3：繁忙期・納品ルール・認証セッション

| 子テーブル（日本語名）                  | 外部キー → 親テーブル       | 関係・説明                                                 |
| --------------------------------------- | --------------------------- | ---------------------------------------------------------- |
| `supplier_closures`（仕入先休業日）     | `supplier_id` → `suppliers` | **1対多**。1社に複数の休業日を登録できる                   |
| `supplier_delivery_rules`（納品ルール） | `supplier_id` → `suppliers` | **1対多**。1社に複数の納品条件を登録できる                 |
| `busy_periods`（繁忙期）                | `store_id` → `stores`       | **1対多**。1店舗で複数年の繁忙期を管理できる               |
| `auth_sessions`（認証セッション）       | `staff_id` → `staffs`       | **1対多**。1人のスタッフが複数端末のセッションを保持できる |

`busy_periods` は、発注履歴や棚卸履歴の業務日付と期間照合して分析する。直接の外部キーを必須とはしない。

`auth_sessions` はPhase 3の追加セキュリティ機能として扱い、Phase 1のMigrationには含めない。

---

## 9. Rails実装時の注意事項

### 複数の外部キーで同じテーブルを参照する

`inventory_counts` は、作成者と確定者の両方で `staffs` を参照する。

この場合、Railsの関連名と実際のModelを区別するため、`class_name` を使用する。

### NULLを許可する関連

下書き中は、棚卸の確定者が存在しない。

このため `confirmed_by_id` などはNULLを許可する設計とし、Railsでは `optional: true` を指定する。

ただし、NULLを許可することと、存在しないスタッフIDへの参照を許可することは異なる。実際のDBでは外部キー制約を設定する。

### 中間テーブル

`position_items` および `stock_location_items` は、独立した属性を持つ中間テーブルとして扱う。

Rails実装では `has_many :through` を使用する方針とする。

### 削除時の動作

棚卸・発注・納品などの履歴に参照されているマスタを不用意に削除しない。

`dependent: :destroy` の適用は各テーブルの履歴保存要件を確認してから判断する。

---

## 10. 後続Issueで確定する事項

| Issue | 決定する内容                               |
| ----- | ------------------------------------------ |
| #4    | 認証・スタッフ・ポジション担当履歴の制約   |
| #5    | 食材・保管場所・棚卸責任ポジションの関連   |
| #6    | 棚卸実施者・入力者・確定者と棚卸明細の関連 |
| #7    | 訂正履歴・監査対象の関連                   |
| #8    | FK・UNIQUE・店舗境界・整合性のレビュー     |
| #9    | Phase 3の認証セッション管理                |

本書は論理設計案であり、Rails ModelやMigrationが実装されたことを意味しない。

### ポジション担当履歴の店舗境界

- `position_assignments.store_id` → `stores.id`
- `(store_id, staff_id)` → `staffs(store_id, id)`
- `(store_id, position_id)` → `positions(store_id, id)`
- `(store_id, assigned_by_id)` → `staffs(store_id, id)`（登録者がいる場合）

スタッフとポジションは、所属店舗が一致する場合のみ関連付けられる。

スタッフとポジションの多対多は、担当期間を持つ
`position_assignments` を介して表現する。

同一スタッフによる複数ポジションの兼務は許可する。
同一スタッフ・同一ポジションの担当期間重複は禁止する。

担当ポジションは発注承認権限とは独立する。

## 11. Issue #4 スタッフ自己登録と認証の関連

### Phase 1の運用

- 運用店舗はシステム初期設定済みの1店舗とする。
- スタッフはアプリから本人が自己登録する。
- スタッフ登録時の所属店舗はRails側で決定する。
- 将来の複数店舗運用を見据え、store_idは維持する。

### スタッフとGoogle認証

- storesは複数のstaffsを持つ。
- staffsはoauth_identitiesと関連する。
- 自己登録ではstaffsとoauth_identitiesを同一トランザクションで作成する。
- UNIQUE(issuer, subject)で同一Googleアカウントの重複を禁止する。
- UNIQUE(staff_id, provider)で同一スタッフへの複数Googleアカウント登録を禁止する。

### 登録時の認可

- Google認証だけでは店舗参加を許可しない。
- Railsが既存店舗の参加コードを検証する。
- 新規スタッフにはstaff権限だけを付与する。
- 管理者権限は自己登録で付与しない。
- スタッフ停止・退職後はAPIアクセスを拒否する。
- 初期管理者はシステム導入時に別の安全な手順で設定する。

### ポジション別発注承認権限

`position_approval_authorizations` は、
スタッフと承認対象ポジションの権限関係を管理する。

関連：

- `store_id` → `stores.id`
- `(store_id, staff_id)` → `staffs(store_id, id)`
- `(store_id, position_id)` → `positions(store_id, id)`
- `(store_id, granted_by_id)` → `staffs(store_id, id)`

業務上の関係：

- 1店舗は複数の発注承認権限を持つ。
- 1人のスタッフは複数ポジションの承認権限を持てる。
- 1ポジションは複数の承認可能スタッフを持てる。
- 権限付与者もスタッフとして記録する。
- 担当ポジションと発注承認権限は独立する。
- 現在の所属店舗が一致しない権限付与は禁止する。

Phase 1では関連構造を設計するのみとし、
本テーブルの実装はPhase 2で行う。
