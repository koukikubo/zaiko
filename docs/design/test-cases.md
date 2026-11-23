# ZAIKO テストケース設計書

- バージョン：0.1
- 対応Issue：#4
- 対象：認証・スタッフ・ポジション管理
- 状態：設計中・RSpec未実装

## 1. テスト設計方針

- TDD（Red → Green → Refactor）を採用する。
- Model Spec：バリデーション・関連・DB制約を検証する。
- Service Spec：複数の処理にまたがる業務ルールを検証する。
- Request Spec：API認証・認可・レスポンスを検証する。
- 正常系、異常系、境界値、他店舗アクセスを確認する。
- Google認証の外部通信はテスト時に適切に置き換える。
- API認証方式が未確定の項目は、仕様確定後に詳細化する。

## 2. Google認証・スタッフ認可

| ID       | 分類    | テスト内容                                         | 期待結果                 |
| -------- | ------- | -------------------------------------------------- | ------------------------ |
| AUTH-001 | Request | 登録済み・有効なスタッフがGoogle認証に成功         | ログイン成功             |
| AUTH-002 | Request | 未登録のGoogleアカウントで認証                     | 利用拒否                 |
| AUTH-003 | Request | メールアドレスのみ一致する未紐付けアカウントで認証 | 自動紐付けせず拒否       |
| AUTH-004 | Service | Google ID Tokenの署名が不正                        | 認証失敗                 |
| AUTH-005 | Service | Google ID Tokenの有効期限切れ                      | 認証失敗                 |
| AUTH-006 | Service | issuerが許可値以外                                 | 認証失敗                 |
| AUTH-007 | Service | audienceがアプリの許可値以外                       | 認証失敗                 |
| AUTH-008 | Request | 利用停止中のスタッフがAPIを呼び出す                | アクセス拒否             |
| AUTH-009 | Request | 退職済みスタッフがAPIを呼び出す                    | アクセス拒否             |
| AUTH-010 | Request | 認証情報なしで保護対象APIを呼び出す                | 401 Unauthorized         |
| AUTH-011 | Request | 認証済みだが操作権限がない                         | 403 Forbidden            |
| AUTH-012 | Request | 他店舗のデータを指定する                           | 権限に応じてアクセス拒否 |

### 補足

- Googleアカウントの照合には検証済みの `iss` と `sub` を用いる。
- Google認証成功とZAIKOの業務利用許可は別の判定とする。
- 401と403の使い分け、他店舗データの存在を隠すための404利用方針はAPI設計で確定する。
- 初回紐付け手順・PKCE・nonce等の検証条件を確定後、テストケースを追加する。

## 3. 店舗マスタ

| ID        | 分類     | テスト内容                           | 期待結果 |
| --------- | -------- | ------------------------------------ | -------- |
| STORE-001 | Model    | 正常な店舗情報を登録                 | 保存成功 |
| STORE-002 | Model/DB | 店舗コードの重複                     | 登録失敗 |
| STORE-003 | Model/DB | 店舗名が未入力                       | 登録失敗 |
| STORE-004 | Model/DB | 店舗コードが未入力                   | 登録失敗 |
| STORE-005 | Service  | 利用停止した店舗のスタッフがアクセス | 利用拒否 |

## 4. スタッフマスタ

| ID        | 分類     | テスト内容                           | 期待結果               |
| --------- | -------- | ------------------------------------ | ---------------------- |
| STAFF-001 | Model    | 有効なスタッフを登録                 | 保存成功               |
| STAFF-002 | Model/DB | 同一店舗でスタッフコードが重複       | 登録失敗               |
| STAFF-003 | Model    | 異なる店舗で同じスタッフコードを使用 | 登録成功               |
| STAFF-004 | Model/DB | 存在しない店舗IDを指定               | 登録失敗               |
| STAFF-005 | Model/DB | 許可していないrole_codeを指定        | 登録失敗               |
| STAFF-006 | Model/DB | 許可していないstatusを指定           | 登録失敗               |
| STAFF-007 | Service  | スタッフを退職扱いにする             | 履歴を保持して利用停止 |
| STAFF-008 | Request  | 一般スタッフが管理者専用操作を実行   | アクセス拒否           |

## 5. 外部認証アカウント

| ID        | 分類     | テスト内容                               | 期待結果         |
| --------- | -------- | ---------------------------------------- | ---------------- |
| OAUTH-001 | Model    | 有効なスタッフにGoogleアカウントを紐付け | 保存成功         |
| OAUTH-002 | Model/DB | 同じissuerとsubjectを重複登録            | 登録失敗         |
| OAUTH-003 | Model/DB | 存在しないstaff_idを指定                 | 登録失敗         |
| OAUTH-004 | Service  | 未許可のスタッフにアカウントを紐付け     | 拒否             |
| OAUTH-005 | Service  | メールアドレスだけ一致する別アカウント   | 自動紐付けしない |

## 6. ポジションマスタ・担当履歴

| ID      | 分類     | テスト内容                             | 期待結果                   |
| ------- | -------- | -------------------------------------- | -------------------------- |
| POS-001 | Model    | ポジションを登録                       | 保存成功                   |
| POS-002 | Model/DB | 同一店舗でポジションコードが重複       | 登録失敗                   |
| POS-003 | Model    | 異なる店舗で同じポジションコードを使用 | 登録成功                   |
| POS-004 | Model    | スタッフにポジションを割り当て         | 担当履歴を保存             |
| POS-005 | Model/DB | 担当終了日が開始日より前               | 登録失敗                   |
| POS-006 | Service  | 他店舗のポジションをスタッフに割り当て | 拒否                       |
| POS-007 | Service  | ポジションを変更                       | 過去の担当履歴を保持       |
| POS-008 | Service  | 同じ期間に複数ポジションを担当         | 仕様確定後に期待結果を定義 |
| POS-009 | Service  | 同一ポジションの担当期間が重複         | 仕様確定後に期待結果を定義 |

## 7. Phase 1認証方式の未確定テスト

以下はAPI認証方式の決定後に具体化する。

| ID          | 検証項目           | 確認事項                           |
| ----------- | ------------------ | ---------------------------------- |
| SESSION-001 | 認証情報の期限     | 期限切れでAPI利用できない          |
| SESSION-002 | ログアウト         | アプリ側の認証情報削除と残存リスク |
| SESSION-003 | スタッフの利用停止 | 既存ログイン状態でもAPIを拒否する  |
| SESSION-004 | トークン漏えい     | 有効期限と想定する防御策           |
| SESSION-005 | 認証情報の保存     | 端末で適切に保護される             |

`auth_sessions` テーブルを用いた端末別強制失効は
Phase 3のIssue #9で検証する。

## 8. 未確定事項

- [ ] Phase 1のAPI認証方式と認証情報の有効期限
- [ ] Googleアカウントの安全な初回紐付け方法
- [ ] ログアウト時の仕様とセッション失効の限界
- [ ] ポジションを複数同時担当できるか
- [ ] 担当期間の重複をどう扱うか
- [ ] 管理者・一般スタッフの権限範囲
- [ ] 店舗間の不正な紐付けを防止するDB制約

## 9. 完了条件

- [ ] 正常系・異常系・認可テストが定義されている
- [ ] 各テーブルの主要制約にテストケースがある
- [ ] 認証の仕様が決まっている
- [ ] 未確定テストの期待結果が確定している
- [ ] RailsのRSpec実装に着手できる

本書はテスト設計書であり、RSpecコードは未実装。

## 10. 店舗・スタッフ追加テスト

| ID        | 分類     | テスト内容                          | 期待結果         |
| --------- | -------- | ----------------------------------- | ---------------- |
| STORE-006 | Model/DB | 店舗コードに空文字を指定            | 登録失敗         |
| STORE-007 | Model/DB | 店舗コードに空白のみ指定            | 登録失敗         |
| STORE-008 | Request  | 利用停止した店舗のAPIにアクセス     | 拒否             |
| STAFF-009 | Model/DB | role_codeに不正値を指定             | DB制約違反       |
| STAFF-010 | Model/DB | statusに不正値を指定                | DB制約違反       |
| STAFF-011 | Model/DB | retired_atがNULLのままretiredへ変更 | DB制約違反       |
| STAFF-012 | Request  | 別店舗のスタッフ情報を更新          | 拒否             |
| STAFF-013 | Model    | 古いlock_versionで更新              | StaleObjectError |
| STAFF-014 | Request  | suspendedのスタッフでAPIにアクセス  | 拒否             |
| STAFF-015 | Request  | retiredのスタッフでAPIにアクセス    | 拒否             |

## 11. Googleアカウント紐付け詳細テスト

| ID        | 分類     | テスト内容                                       | 期待結果                |
| --------- | -------- | ------------------------------------------------ | ----------------------- |
| OAUTH-006 | Model/DB | 同じissuer・subjectで別スタッフに登録            | UNIQUE制約違反          |
| OAUTH-007 | Model/DB | 同じスタッフにGoogleアカウントを2件登録          | UNIQUE制約違反          |
| OAUTH-008 | Service  | 正規のGoogleアカウントを許可済みスタッフに紐付け | 成功                    |
| OAUTH-009 | Service  | メールアドレスだけ一致するアカウントで紐付け     | 拒否                    |
| OAUTH-010 | Service  | 不正な署名のID Tokenを使用                       | 認証失敗                |
| OAUTH-011 | Service  | 期限切れのID Tokenを使用                         | 認証失敗                |
| OAUTH-012 | Service  | audienceが異なるID Tokenを使用                   | 認証失敗                |
| OAUTH-013 | Service  | issuerが許可値以外                               | 認証失敗                |
| OAUTH-014 | Service  | 使用済み認証コードの再利用                       | 認証失敗                |
| OAUTH-015 | Service  | 初回紐付けを同時に2回実施                        | 1件のみ成功             |
| OAUTH-016 | Request  | 未登録アカウントで保護対象APIにアクセス          | 拒否                    |
| OAUTH-017 | Request  | 退職済みスタッフのGoogleアカウントでアクセス     | 拒否                    |
| OAUTH-018 | Request  | 停止中の店舗のスタッフとしてアクセス             | 拒否                    |
| OAUTH-019 | Service  | 許可のないアカウント紐付け変更                   | 拒否                    |
| OAUTH-020 | Service  | Googleアカウントのメールアドレス変更             | subjectによる識別を維持 |

### 未確定テスト

- 初回紐付け方式に応じた承認・招待トークンの検証
- PKCEの検証失敗
- state・nonceの検証失敗
- 紐付け解除・再紐付け
- Phase 1のAPI認証トークン発行・期限切れ・ログアウト

## 12. ポジション・担当履歴の追加テスト

| ID      | 分類     | テスト内容                               | 期待結果                           |
| ------- | -------- | ---------------------------------------- | ---------------------------------- |
| POS-010 | Model/DB | 同一店舗でポジションコードが重複         | 登録失敗                           |
| POS-011 | Model/DB | ポジション名が空白のみ                   | 登録失敗                           |
| POS-012 | Service  | 無効化されたポジションへ新規担当登録     | 拒否                               |
| POS-013 | Service  | 同一スタッフが揚場と焼場を兼務           | 登録成功                           |
| POS-014 | Service  | 同じ焼場に複数スタッフを登録             | 登録成功                           |
| POS-015 | DB       | 同一スタッフ・同一ポジションで期間が重複 | 排他制約違反                       |
| POS-016 | DB       | 同一担当の終了日の翌日から再登録         | 登録成功                           |
| POS-017 | DB       | 担当終了日が開始日より前                 | CHECK制約違反                      |
| POS-018 | DB       | ended_onがNULLの担当と重複登録           | 排他制約違反                       |
| POS-019 | DB       | 別店舗のスタッフとポジションを関連付け   | 複合FK制約違反                     |
| POS-020 | Service  | 退職済みスタッフへの新規担当登録         | 拒否                               |
| POS-021 | Service  | スタッフ退職時に未終了の担当期間を処理   | 担当終了・履歴保持                 |
| POS-022 | Request  | 他店舗の担当履歴を参照・変更             | アクセス拒否                       |
| POS-023 | Service  | ポジションの担当だけで発注承認を試行     | 承認権限がなければ拒否             |
| POS-024 | Service  | 代理棚卸を実施                           | 実施者を責任ポジションとは別に記録 |
| POS-025 | Model    | 古いlock_versionでポジションを更新       | 更新競合を検出                     |

## 13. 既存店舗へのスタッフ自己登録テスト

| ID         | 分類    | テスト内容                         | 期待結果                           |
| ---------- | ------- | ---------------------------------- | ---------------------------------- |
| SIGNUP-001 | Request | 有効なGoogle認証と参加コードで登録 | 既存店舗にstaffを作成              |
| SIGNUP-002 | Request | 正常登録後にAPIへアクセス          | staff権限で成功                    |
| SIGNUP-003 | Request | 不正な参加コードを入力             | 登録拒否                           |
| SIGNUP-004 | Request | 期限切れの参加コードを入力         | 登録拒否                           |
| SIGNUP-005 | Request | 無効化された参加コードを入力       | 登録拒否                           |
| SIGNUP-006 | Request | 参加コードなしで登録               | 登録拒否                           |
| SIGNUP-007 | Request | 登録時にadmin権限を指定            | 権限昇格を拒否                     |
| SIGNUP-008 | Request | 任意のstore_idを送信               | サーバー指定店舗以外へ登録できない |
| SIGNUP-009 | Service | 同じGoogleアカウントを重複登録     | 重複作成を拒否                     |
| SIGNUP-010 | Service | Google ID Tokenが不正              | 登録拒否                           |
| SIGNUP-011 | DB      | 登録途中にエラー発生               | 全体ロールバック                   |
| SIGNUP-012 | Request | 停止中の店舗に登録                 | 登録拒否                           |
| SIGNUP-013 | Request | 参加コードを連続誤入力             | レート制限                         |
| SIGNUP-014 | Service | 参加コードを再生成                 | 古いコードは無効                   |
| SIGNUP-015 | Service | スタッフコード生成が競合           | 一意性を維持                       |
| SIGNUP-016 | Request | 登録済みアカウントで再ログイン     | 既存スタッフとしてログイン         |

## 14. Phase 1 API認証テスト

| ID        | 分類    | テスト内容                                | 期待結果                     |
| --------- | ------- | ----------------------------------------- | ---------------------------- |
| TOKEN-001 | Service | 有効スタッフのGoogle認証が成功            | ZAIKOトークン発行            |
| TOKEN-002 | Request | 有効なトークンでAPIを呼び出す             | 成功                         |
| TOKEN-003 | Request | 期限切れトークン                          | 401                          |
| TOKEN-004 | Request | 改ざんされたトークン                      | 401                          |
| TOKEN-005 | Request | 用途が異なるトークン                      | 401                          |
| TOKEN-006 | Request | auth_versionが一致しない                  | 401                          |
| TOKEN-007 | Request | suspendedスタッフがアクセス               | 401                          |
| TOKEN-008 | Request | retiredスタッフがアクセス                 | 401                          |
| TOKEN-009 | Request | 停止した店舗のスタッフがアクセス          | 401                          |
| TOKEN-010 | Request | 別店舗のデータを参照                      | 原則404                      |
| TOKEN-011 | Service | ログアウト                                | auth_versionを更新           |
| TOKEN-012 | Request | ログアウト前のトークンを再利用            | 401                          |
| TOKEN-013 | Service | トークン期限切れ後に再ログイン            | 新規トークンを発行           |
| TOKEN-014 | Request | 権限不足の操作を実行                      | 403                          |
| TOKEN-015 | Request | OAuth紐付けが解除されたトークンでアクセス | 401                          |
| TOKEN-016 | Service | 複数端末でログイン後にログアウト          | 全端末の旧トークンを拒否     |
| TOKEN-017 | Service | ログアウト時にサーバー通信が失敗          | サーバー失効を成功扱いしない |

## 15. ポジション別発注承認権限テスト

**実装フェーズ：** Phase 2

| ID           | 分類    | テスト内容                                   | 期待結果         |
| ------------ | ------- | -------------------------------------------- | ---------------- |
| APPROVAL-001 | Service | 承認権限があるポジションの発注を承認         | 成功             |
| APPROVAL-002 | Service | 担当外ポジションの明示的な承認権限で承認     | 成功             |
| APPROVAL-003 | Service | 担当ポジションのみで承認権限がない           | 拒否             |
| APPROVAL-004 | Service | 自分が作成した発注を承認                     | 拒否             |
| APPROVAL-005 | Service | adminが自分の発注を承認                      | 拒否             |
| APPROVAL-006 | Service | 失効済み承認権限で承認                       | 拒否             |
| APPROVAL-007 | Service | 期限切れの承認権限で承認                     | 拒否             |
| APPROVAL-008 | Service | 開始日時前の承認権限で承認                   | 拒否             |
| APPROVAL-009 | Service | 他店舗の発注を承認                           | 拒否             |
| APPROVAL-010 | DB      | 同一スタッフ・ポジションで未失効の期間が重複 | EXCLUDE制約違反  |
| APPROVAL-011 | DB      | 同一期間でも別ポジションなら登録             | 成功             |
| APPROVAL-012 | DB      | 同一期間でも別スタッフなら登録               | 成功             |
| APPROVAL-013 | DB      | 他店舗のスタッフを承認対象に指定             | 複合FK制約違反   |
| APPROVAL-014 | DB      | 他店舗のスタッフを権限付与者に指定           | 複合FK制約違反   |
| APPROVAL-015 | Service | 承認権限を失効                               | 以降の承認を拒否 |
| APPROVAL-016 | Service | 失効後に新たな権限を付与                     | 登録成功         |
| APPROVAL-017 | Request | 一般スタッフが承認権限を付与                 | 403              |
| APPROVAL-018 | Service | 発注の承認を実行                             | 監査履歴を登録   |
| APPROVAL-019 | Service | 承認権限の付与・失効を実行                   | 監査履歴を登録   |
| APPROVAL-020 | Service | 退職済みスタッフが発注を承認                 | 拒否             |

## 16. 食材マスタ詳細テスト

**対応Issue：** #5

**実装フェーズ：** Phase 1

| ID       | 分類    | テスト内容                         | 期待結果           |
| -------- | ------- | ---------------------------------- | ------------------ |
| ITEM-001 | Model   | 正常な食材情報を登録               | 成功               |
| ITEM-002 | DB      | 同一店舗で食材コードが重複         | UNIQUE制約違反     |
| ITEM-003 | Model   | 異なる店舗で同じ食材コードを使用   | 成功               |
| ITEM-004 | DB      | 食材コードが空白のみ               | CHECK制約違反      |
| ITEM-005 | DB      | 食材名が空白のみ                   | CHECK制約違反      |
| ITEM-006 | DB      | 不正なbase_unitを指定              | CHECK制約違反      |
| ITEM-007 | DB      | sort_orderに負数を指定             | CHECK制約違反      |
| ITEM-008 | Model   | 食材名を変更                       | 成功               |
| ITEM-009 | Service | 食材を無効化                       | active=falseへ変更 |
| ITEM-010 | Service | 無効化した食材の過去履歴を参照     | 履歴を保持         |
| ITEM-011 | Service | 数量履歴のある食材の基準単位を変更 | 拒否               |
| ITEM-012 | Request | 他店舗の食材を更新                 | 拒否               |
| ITEM-013 | Model   | 古いlock_versionで更新             | 更新競合を検出     |
| ITEM-014 | Service | 複数ポジションで同じ食材を使用     | 食材マスタは1件    |
| ITEM-015 | Service | 在庫数量をポジション別に集計       | 二重計上しない     |
| ITEM-016 | Service | kgからgへ数量換算                  | 正確に換算         |
| ITEM-017 | Service | Lからmlへ数量換算                  | 正確に換算         |
| ITEM-018 | Service | 単位の異なる数量を不正に加算       | 拒否               |
| ITEM-019 | Service | 個数単位で小数数量を登録           | 拒否               |
| ITEM-020 | Service | 在庫数量のない食材を表示           | 未棚卸として扱う   |

## 17. ポジション別使用食材テスト

**対応Issue：** #5

**実装フェーズ：** Phase 1

| ID                | 分類    | テスト内容                           | 期待結果                 |
| ----------------- | ------- | ------------------------------------ | ------------------------ |
| POSITION-ITEM-001 | Model   | 有効なポジションと食材を紐付け       | 成功                     |
| POSITION-ITEM-002 | DB      | 同一店舗・ポジション・食材を重複登録 | UNIQUE制約違反           |
| POSITION-ITEM-003 | Model   | 同じ食材を別ポジションに紐付け       | 成功                     |
| POSITION-ITEM-004 | Model   | 同じポジションに複数食材を紐付け     | 成功                     |
| POSITION-ITEM-005 | DB      | 他店舗の食材を紐付け                 | 複合FK制約違反           |
| POSITION-ITEM-006 | DB      | 他店舗のポジションを紐付け           | 複合FK制約違反           |
| POSITION-ITEM-007 | Service | 無効な食材を新規紐付け               | 拒否                     |
| POSITION-ITEM-008 | Service | 無効なポジションを新規紐付け         | 拒否                     |
| POSITION-ITEM-009 | Service | 使用対象を無効化                     | active=false             |
| POSITION-ITEM-010 | Service | 無効化済みの組み合わせを再有効化     | 既存レコードを更新       |
| POSITION-ITEM-011 | Model   | 古いlock_versionで更新               | 更新競合を検出           |
| POSITION-ITEM-012 | Service | 複数ポジションで同じ食材を使用       | 食材マスタは1件          |
| POSITION-ITEM-013 | Service | 複数ポジションに紐付く食材を在庫集計 | 物理在庫を二重計上しない |
| POSITION-ITEM-014 | Request | 他店舗の関連情報を参照・変更         | 拒否                     |
| POSITION-ITEM-015 | Service | 使用食材の設定を変更                 | 監査履歴を登録           |

## 18. 保管場所マスタ詳細テスト

**対応Issue：** #5

**実装フェーズ：** Phase 1

| ID           | 分類    | テスト内容                             | 期待結果                 |
| ------------ | ------- | -------------------------------------- | ------------------------ |
| LOCATION-001 | Model   | 正常な保管場所を登録                   | 成功                     |
| LOCATION-002 | DB      | 同一店舗で保管場所コードが重複         | UNIQUE制約違反           |
| LOCATION-003 | DB      | 異なる店舗で同じ保管場所コードを登録   | 成功                     |
| LOCATION-004 | DB      | 保管場所名が空白のみ                   | CHECK制約違反            |
| LOCATION-005 | DB      | 不正なlocation_kindを指定              | CHECK制約違反            |
| LOCATION-006 | DB      | 不正な保管場所コードを指定             | CHECK制約違反            |
| LOCATION-007 | DB      | sort_orderに負数を指定                 | CHECK制約違反            |
| LOCATION-008 | Model   | 保管場所名を変更                       | 成功                     |
| LOCATION-009 | Service | 保管場所を無効化                       | active=false             |
| LOCATION-010 | Service | 無効な保管場所で新規棚卸を開始         | 拒否                     |
| LOCATION-011 | Service | 未確定棚卸がある保管場所を無効化       | 拒否                     |
| LOCATION-012 | Service | 保管場所を再有効化                     | 関連設定を検証して成功   |
| LOCATION-013 | Request | 他店舗の保管場所を参照                 | 拒否                     |
| LOCATION-014 | Request | 他店舗の保管場所を更新                 | 拒否                     |
| LOCATION-015 | Model   | 古いlock_versionで更新                 | 更新競合を検出           |
| LOCATION-016 | Service | 保管場所情報を変更                     | 監査履歴を登録           |
| LOCATION-017 | Service | 同じ食材を複数保管場所に配置           | 成功                     |
| LOCATION-018 | Service | 複数ポジションが使用する同一食材を集計 | 物理在庫を二重計上しない |
| LOCATION-019 | Service | 無効化した保管場所の過去棚卸を参照     | 履歴を保持               |
| LOCATION-020 | Request | 保管場所コード・名称・区分で検索       | 条件に一致する結果を返す |

## 19. 保管場所別管理食材テスト

**対応Issue：** #5

**実装フェーズ：** Phase 1

| ID                      | 分類    | テスト内容                           | 期待結果                     |
| ----------------------- | ------- | ------------------------------------ | ---------------------------- |
| STOCK-LOCATION-ITEM-001 | Model   | 正常な保管場所別食材を登録           | 成功                         |
| STOCK-LOCATION-ITEM-002 | DB      | 同一店舗・保管場所・食材を重複登録   | UNIQUE制約違反               |
| STOCK-LOCATION-ITEM-003 | Model   | 同じ食材を別の保管場所に登録         | 成功                         |
| STOCK-LOCATION-ITEM-004 | Model   | 同じ保管場所に複数食材を登録         | 成功                         |
| STOCK-LOCATION-ITEM-005 | DB      | 他店舗の食材を登録                   | 複合FK制約違反               |
| STOCK-LOCATION-ITEM-006 | DB      | 他店舗の保管場所を登録               | 複合FK制約違反               |
| STOCK-LOCATION-ITEM-007 | DB      | 他店舗の責任ポジションを登録         | 複合FK制約違反               |
| STOCK-LOCATION-ITEM-008 | DB      | 責任ポジションがNULL                 | NOT NULL制約違反             |
| STOCK-LOCATION-ITEM-009 | Service | 無効な食材を登録                     | 拒否                         |
| STOCK-LOCATION-ITEM-010 | Service | 無効な保管場所を登録                 | 拒否                         |
| STOCK-LOCATION-ITEM-011 | Service | 無効な責任ポジションを登録           | 拒否                         |
| STOCK-LOCATION-ITEM-012 | Service | 棚卸責任ポジションを変更             | 成功・監査履歴作成           |
| STOCK-LOCATION-ITEM-013 | Service | 棚卸対象を無効化                     | 条件を満たす場合active=false |
| STOCK-LOCATION-ITEM-014 | Service | 無効化済みの組み合わせを再有効化     | 既存レコードを更新           |
| STOCK-LOCATION-ITEM-015 | Model   | 古いlock_versionで更新               | 更新競合を検出               |
| STOCK-LOCATION-ITEM-016 | Service | 同じ食材を複数ポジションが使用       | 物理在庫は重複しない         |
| STOCK-LOCATION-ITEM-017 | Service | 同じ食材を複数保管場所で棚卸         | 保管場所別に数量を管理       |
| STOCK-LOCATION-ITEM-018 | Service | 複数保管場所の最新確定数量を集計     | 同じ実在庫を二重計上しない   |
| STOCK-LOCATION-ITEM-019 | Service | 責任ポジション変更後に過去棚卸を参照 | 当時の責任ポジションを保持   |
| STOCK-LOCATION-ITEM-020 | Request | 他店舗の保管場所別食材を参照・更新   | 拒否                         |
| STOCK-LOCATION-ITEM-021 | Service | 未確定棚卸がある対象を無効化         | 拒否                         |
| STOCK-LOCATION-ITEM-022 | Service | 在庫が残る対象を無効化               | 在庫整理なしでは拒否         |
| STOCK-LOCATION-ITEM-023 | Service | 責任ポジション変更と棚卸実施者の記録 | 両方を区別して保持           |
| STOCK-LOCATION-ITEM-024 | DB      | sort_orderに負数を指定               | CHECK制約違反                |

## 20. 在庫照会・帳票出力のデータ対応テスト

**対応Issue：** #5

**対応要件：** FR-01、FR-02

**実装フェーズ：** Phase 1

| ID         | 分類    | テスト内容                         | 期待結果                     |
| ---------- | ------- | ---------------------------------- | ---------------------------- |
| OUTPUT-001 | Service | 同一食材を複数ポジションが使用     | 数量を重複加算しない         |
| OUTPUT-002 | Service | 保管場所ごとの最新確定数量を取得   | 各場所で最新1件のみ採用      |
| OUTPUT-003 | Service | 下書きの数量が存在する             | 集計から除外                 |
| OUTPUT-004 | Service | 確定履歴が存在しない               | 未確認として表示             |
| OUTPUT-005 | Service | 確定済み数量が0                    | 実測0として表示              |
| OUTPUT-006 | Service | 過去の対象営業日を指定             | 対象日以前の情報を取得       |
| OUTPUT-007 | PDF     | 棚卸記入用を出力                   | 前回数量を初期状態で非表示   |
| OUTPUT-008 | PDF     | 前回数量の表示を指定               | 前回数量を表示               |
| OUTPUT-009 | CSV     | 食材名・数量・単位等を出力         | 定義した列を出力             |
| OUTPUT-010 | CSV     | 数式として解釈され得る文字列を出力 | CSV Injectionを防止          |
| OUTPUT-011 | Request | 他店舗の帳票を出力                 | 拒否                         |
| OUTPUT-012 | Service | 名称・コード・場所で検索           | 指定条件で抽出               |
| OUTPUT-013 | Service | 保管場所ごとの確認日が異なる       | 各場所の最新確認値として集計 |
| OUTPUT-014 | Service | 最新数量が未確認の保管場所を含む   | 未確認対象を区別して表示     |
| OUTPUT-015 | PDF     | 責任ポジション別に棚卸表を出力     | 対象の食材・場所のみ出力     |
| OUTPUT-016 | CSV     | 日本語の食材名を出力               | 文字化けせず出力             |

## 21. 日次棚卸ヘッダー詳細テスト

**対応Issue：** #6

**対応要件：** FR-03、FR-04

| ID        | 分類    | テスト内容                             | 期待結果                          |
| --------- | ------- | -------------------------------------- | --------------------------------- |
| COUNT-001 | Model   | 初回棚卸を作成                         | draft・revision_no=1              |
| COUNT-002 | DB      | 同一営業日・保管場所・版番号を重複登録 | UNIQUE制約違反                    |
| COUNT-003 | DB      | 同一営業日・保管場所にdraftを重複作成  | 部分UNIQUE制約違反                |
| COUNT-004 | DB      | 不正なstatusを指定                     | CHECK制約違反                     |
| COUNT-005 | DB      | revision_noに0以下を指定               | CHECK制約違反                     |
| COUNT-006 | Model   | 第2版以降で訂正理由を未設定            | 登録成功・変更履歴を保持          |
| COUNT-007 | Service | draftを編集                            | 成功                              |
| COUNT-008 | Service | 必須数量が未入力の状態で確定           | 拒否                              |
| COUNT-009 | Service | 実測数量0で確定                        | 成功                              |
| COUNT-010 | Service | draftを確定                            | confirmed・確定者・確定日時を保存 |
| COUNT-011 | Service | 既存のconfirmedレコードを直接更新      | 拒否・既存レコードは不変          |
| COUNT-012 | Service | 確定済み棚卸を再棚卸                   | 新しい版を作成                    |
| COUNT-013 | Service | 再棚卸を確定                           | 前版を残し最新版を採用            |
| COUNT-014 | Service | 同日の複数確定版から在庫照会           | 最大確定revision_noのみ採用       |
| COUNT-015 | Service | 再棚卸がdraftの間に在庫照会            | 直前の確定版を採用                |
| COUNT-016 | Model   | 古いlock_versionで更新                 | 更新競合を検出                    |
| COUNT-017 | Service | 2人が同時に確定                        | 一方のみ成功                      |
| COUNT-018 | Service | 2人が同時に再棚卸版を作成              | 版番号を重複させない              |
| COUNT-019 | DB      | 他店舗の保管場所を指定                 | 複合FK制約違反                    |
| COUNT-020 | DB      | 他店舗の作成者・確定者を指定           | 複合FK制約違反                    |
| COUNT-021 | Request | 他店舗の棚卸を参照・編集               | 拒否                              |
| COUNT-022 | Service | 複数スタッフが同じ棚卸を分担           | 同じヘッダーに明細を保存          |
| COUNT-023 | Service | 過去の営業日の訂正版を作成             | 後日の棚卸を最新として維持        |
| COUNT-024 | Service | 確定途中にエラーが発生                 | 全変更をロールバック              |
| COUNT-025 | Service | 確定後に棚卸履歴を参照                 | 全版の数量・確定者・日時を確認    |

## 22. 日次棚卸明細詳細テスト

**対応Issue：** #6

**対応要件：** FR-03、FR-04

| ID       | 分類    | テスト内容                       | 期待結果               |
| -------- | ------- | -------------------------------- | ---------------------- |
| LINE-001 | Model   | 正常な棚卸明細を作成             | 成功                   |
| LINE-002 | DB      | 同一棚卸に同じ食材を重複登録     | UNIQUE制約違反         |
| LINE-003 | DB      | 他店舗の食材を登録               | 複合FK制約違反         |
| LINE-004 | DB      | 別保管場所の食材を登録           | 複合FK制約違反         |
| LINE-005 | DB      | 他店舗の実施者を設定             | 複合FK制約違反         |
| LINE-006 | DB      | 他店舗の責任ポジションを設定     | 複合FK制約違反         |
| LINE-007 | DB      | 実測数量に負数を指定             | CHECK制約違反          |
| LINE-008 | Model   | 実測数量NULLで下書き保存         | 成功・未確認           |
| LINE-009 | Model   | 実測数量0を保存                  | 成功・実測0            |
| LINE-010 | Service | 個数単位に小数を入力             | 拒否                   |
| LINE-011 | Service | 不正な小数精度を入力             | 保存前に拒否           |
| LINE-012 | DB      | 不正な品質状態を入力             | CHECK制約違反          |
| LINE-013 | Service | 直接計測で実施者が未設定         | 拒否                   |
| LINE-014 | Service | 紙の棚卸で実施者が不明           | 補足と入力者を記録     |
| LINE-015 | Service | 実施者と入力者が異なる           | 両方のスタッフを保存   |
| LINE-016 | Service | 確定時に入力者・入力日時が欠落   | 拒否                   |
| LINE-017 | Service | 責任ポジションを変更             | 過去の確定明細は不変   |
| LINE-018 | Service | 食材名を変更                     | 過去の名称を表示       |
| LINE-019 | Service | 前版の明細を引き継ぐ             | 元明細との関係を保持   |
| LINE-020 | Service | 訂正後の最新版を確定             | 前版も保持             |
| LINE-021 | Service | 引き継ぎ数量を再実測と誤認しない | 元の実測日時を保持     |
| LINE-022 | Service | 既存の確定済み明細を直接更新     | 拒否・既存明細は不変   |
| LINE-023 | Model   | 古いlock_versionで更新           | 更新競合を検出         |
| LINE-024 | Service | 2人が同じ明細を更新              | 競合を検出             |
| LINE-025 | Service | 確定と明細更新が同時発生         | 不整合なく一方を拒否   |
| LINE-026 | Request | 他店舗の明細を参照               | 拒否                   |
| LINE-027 | Service | 同じ棚卸で複数スタッフが作業     | 明細ごとに実施者を保持 |
| LINE-028 | DB      | 別店舗の元明細を参照             | 複合FK制約違反         |
| LINE-029 | Service | 他食材の明細を引き継ぎ元に指定   | 拒否                   |
| LINE-030 | Service | 訂正版で全対象食材を保持         | 完全スナップショット   |

## 23. ロス記録詳細テスト

**対応Issue：** #6

**対応要件：** FR-04、FR-05

| ID       | 分類    | テスト内容                         | 期待結果                   |
| -------- | ------- | ---------------------------------- | -------------------------- |
| LOSS-001 | Model   | 正常なロス記録を登録               | 成功                       |
| LOSS-002 | DB      | ロス数量が0                        | CHECK制約違反              |
| LOSS-003 | DB      | ロス数量が負数                     | CHECK制約違反              |
| LOSS-004 | Service | 不正な小数精度を入力               | 拒否                       |
| LOSS-005 | Service | 個数単位で小数を入力               | 拒否                       |
| LOSS-006 | DB      | 不正なロス理由を登録               | CHECK制約違反              |
| LOSS-007 | DB      | その他を選択し備考なし             | CHECK制約違反              |
| LOSS-008 | DB      | 他店舗の食材を指定                 | 複合FK制約違反             |
| LOSS-009 | DB      | 他店舗の報告者を指定               | 複合FK制約違反             |
| LOSS-010 | DB      | 他店舗の登録者を指定               | 複合FK制約違反             |
| LOSS-011 | Model   | 同じ食材で別のロスを登録           | 成功                       |
| LOSS-012 | DB      | 同じrequest_keyを再登録            | UNIQUE制約違反             |
| LOSS-013 | Service | 同じ操作を再送                     | ロス数量を二重加算しない   |
| LOSS-014 | Service | ロスを取消                         | voided・取消者・理由を保存 |
| LOSS-015 | DB      | 理由なしで取消                     | CHECK制約違反              |
| LOSS-016 | Service | 取消済みを再度取消                 | 拒否                       |
| LOSS-017 | Service | ロス数量を訂正                     | 旧記録取消・新記録登録     |
| LOSS-018 | Service | 訂正元と訂正後が別食材             | 拒否                       |
| LOSS-019 | DB      | 同じ訂正元を複数回使用             | UNIQUE制約違反             |
| LOSS-020 | Service | 訂正途中にエラーが発生             | 全変更をロールバック       |
| LOSS-021 | Model   | 古いlock_versionで取消             | 更新競合を検出             |
| LOSS-022 | Service | 2人が同時に訂正                    | 一方のみ成功               |
| LOSS-023 | Service | 取消済みロスを集計                 | 通常集計から除外           |
| LOSS-024 | Service | ロス登録後に在庫照会               | 最新実測数量を変更しない   |
| LOSS-025 | Service | ロス発生前後の棚卸を比較           | 二重控除しない             |
| LOSS-026 | Service | 計測期間外のロスを集計             | 推定使用量から除外         |
| LOSS-027 | Request | 他店舗のロスを参照                 | 拒否                       |
| LOSS-028 | Request | 退職・利用停止スタッフがロスを取消 | 拒否                       |
| LOSS-029 | Service | 食材名変更後に過去ロスを表示       | 記録当時の名称を表示       |
| LOSS-030 | Service | 訂正履歴を表示                     | 元記録と訂正後を追跡可能   |

## 24. 棚卸認可・履歴照会・対象変更テスト

**対応Issue：** #6

**対応要件：** FR-03、FR-04、FR-05

| ID       | 分類    | テスト内容                         | 期待結果                             |
| -------- | ------- | ---------------------------------- | ------------------------------------ |
| RULE-001 | Request | staffが棚卸を作成                  | 許可                                 |
| RULE-002 | Request | 同じポジションの代理棚卸           | 許可                                 |
| RULE-003 | Request | 別ポジションの代理棚卸             | 許可                                 |
| RULE-004 | Request | staffが棚卸を確定                  | 許可                                 |
| RULE-005 | Request | managerが棚卸を確定                | 許可                                 |
| RULE-006 | Request | staffが確定後の数量を訂正          | 許可                                 |
| RULE-007 | Service | 棚卸途中で食材を追加               | 未確認明細を追加                     |
| RULE-008 | Service | 棚卸途中で食材を対象外に変更       | 入力済み数量の扱いを確認し履歴を保持 |
| RULE-009 | Service | 棚卸途中で責任ポジションを変更     | 過去の実施者・日時を保持             |
| RULE-010 | Service | 確定後の数量を編集・保存           | 新しい確定版を自動作成               |
| RULE-011 | Service | 訂正理由を入力しないで保存         | 成功                                 |
| RULE-012 | Service | 訂正後の在庫照会                   | 最新確定版の数量を取得               |
| RULE-013 | Service | 訂正後に過去版を参照               | 元の数量を確認可能                   |
| RULE-014 | Request | 同一店舗の他ポジションの履歴を参照 | 許可                                 |
| RULE-015 | Request | 他店舗の履歴を参照                 | 拒否                                 |
| RULE-016 | Request | 棚卸履歴一覧を取得                 | 直近30日を初期表示                   |
| RULE-017 | Request | 営業日・保管場所で履歴を検索       | 条件に一致する履歴を表示             |
| RULE-018 | Request | 棚卸履歴詳細を取得                 | 数量・実施者・日時等を表示           |
| RULE-019 | Request | スタッフがロスを登録               | 許可                                 |
| RULE-020 | Request | スタッフがロスを訂正・取消         | 許可                                 |
| RULE-021 | Service | 未確認明細を含む棚卸を確定         | 拒否                                 |
| RULE-022 | Service | 同時に同じ確定版を訂正             | 競合を検出し二重更新を防止           |
| RULE-023 | Service | ロス登録後に実測在庫を取得         | ロス数量を再減算しない               |
| RULE-024 | Service | 保存途中にエラー発生               | 新旧版が不整合にならずロールバック   |

## 25. 在庫訂正・監査履歴・整合性テスト

**対応Issue：** #7

**対応要件：** FR-03、FR-04、FR-05

| ID        | 分類    | テスト内容                            | 期待結果                                                                                          |
| --------- | ------- | ------------------------------------- | ------------------------------------------------------------------------------------------------- |
| AUDIT-001 | DB      | 同一店舗のスタッフでイベント作成      | 成功                                                                                              |
| AUDIT-002 | DB      | 別店舗のactor_idを指定                | 複合FK制約違反                                                                                    |
| AUDIT-003 | DB      | actor_kind=staffかつactor_id=NULL     | CHECK制約違反                                                                                     |
| AUDIT-004 | DB      | actor_kind=systemでactor_sourceを指定 | 成功                                                                                              |
| AUDIT-005 | DB      | actor_kind=systemでactor_sourceなし   | CHECK制約違反                                                                                     |
| AUDIT-006 | DB      | 許可されないactor_kindを指定          | CHECK制約違反                                                                                     |
| AUDIT-007 | DB      | 許可されないactionを指定              | CHECK制約違反                                                                                     |
| AUDIT-008 | DB      | 許可されないtarget_typeを指定         | CHECK制約違反                                                                                     |
| AUDIT-009 | DB      | target_id=0または負数                 | CHECK制約違反                                                                                     |
| AUDIT-010 | DB      | before_dataにJSON配列を指定           | CHECK制約違反                                                                                     |
| AUDIT-011 | DB      | after_dataにJSON配列を指定            | CHECK制約違反                                                                                     |
| AUDIT-012 | DB      | reasonに空白のみを指定                | CHECK制約違反                                                                                     |
| AUDIT-013 | Service | 棚卸初回確定と監査を記録              | 確定版と対応イベントを保存                                                                        |
| AUDIT-014 | Service | 確定棚卸を訂正                        | 新版のIDと旧版IDを監査に保持                                                                      |
| AUDIT-015 | Service | 訂正理由を未入力として保存            | 成功・変更者日時を保存                                                                            |
| AUDIT-016 | Service | 訂正前後の数量を確認                  | 旧版・新版の`inventory_count_lines`を参照して明細ごとに比較可能（監査JSONへ数量明細を複製しない） |
| AUDIT-017 | Service | 存在しないtarget_idを指定             | 保存拒否                                                                                          |
| AUDIT-018 | Service | 別店舗のtarget_idを指定               | 保存拒否                                                                                          |
| AUDIT-019 | Service | actionとtarget_typeの不正な組合せ     | 保存拒否                                                                                          |
| AUDIT-020 | Service | 棚卸訂正で監査INSERT失敗              | 棚卸新版もロールバック                                                                            |
| AUDIT-021 | Service | ロス訂正で監査INSERT失敗              | 元ロス取消と訂正版をロールバック                                                                  |
| AUDIT-022 | Service | 同じ確定棚卸を並行訂正                | 一方のみ成功、イベントも1件                                                                       |
| AUDIT-023 | Service | ロスの二重取消・訂正競合              | 不正な二重処理を拒否                                                                              |
| AUDIT-024 | Service | 訂正前の版を開いたまま別の版が確定    | 古い版の保存を拒否                                                                                |
| AUDIT-025 | Request | 一般スタッフが監査イベントを更新      | APIを提供せず更新不可                                                                             |
| AUDIT-026 | Request | 一般スタッフが監査イベントを削除      | APIを提供せず削除不可                                                                             |
| AUDIT-027 | Request | 他店舗の監査履歴を閲覧                | 拒否                                                                                              |
| AUDIT-028 | Request | 退職済みスタッフの監査履歴を閲覧      | 同一店舗の閲覧権限で過去記録を参照可能                                                            |
| AUDIT-029 | Service | 機密情報を監査JSONに渡す              | 許可リストで除外                                                                                  |
| AUDIT-030 | Service | 訂正後の在庫照会                      | 最新確定版のみ採用                                                                                |
| AUDIT-031 | Service | 訂正後に出力するCSV/PDF用データを取得 | 新しい確定数量・版を使用                                                                          |
| AUDIT-032 | Service | ロス取消後の在庫照会                  | 既存の実測数量を再減算しない                                                                      |
| AUDIT-033 | Model   | 同一request_idで複数イベントを保存    | 成功（UNIQUE制約なし）                                                                            |
| AUDIT-034 | Service | 下書き明細削除と監査を記録            | `target_id`に削除明細ID、`before_data`に棚卸IDと食材IDを保存                                      |
| AUDIT-035 | DB      | 本番用DBロールでUPDATE/DELETE         | 権限が分離された環境では拒否                                                                      |
| AUDIT-036 | Service | スタッフ停止・退職後の新規操作        | 拒否、過去監査イベントは不変                                                                      |

## 26. Phase 2 発注業務テスト

**対応要件：** FR-06

**実装フェーズ：** Phase 2

| ID        | 分類    | テスト内容                                                   | 期待結果                                                                                            |
| --------- | ------- | ------------------------------------------------------------ | --------------------------------------------------------------------------------------------------- |
| ORDER-001 | Service | 発注案を途中保存して再開                                     | 保存済みの明細・数量を復元                                                                          |
| ORDER-002 | Service | 複数ポジションの依頼を同じ発注明細に集約                     | 各依頼元を個別に参照可能                                                                            |
| ORDER-003 | Service | 依頼明細ごとにスタッフとポジションを記録                     | 依頼者・依頼元ポジションを保持                                                                      |
| ORDER-004 | Service | 集約後に発注数量を調整                                       | 元の依頼数量は変更されない                                                                          |
| ORDER-005 | Service | 未完了の発注業務一覧を取得                                   | 状態・次の対応者・対応期限を表示                                                                    |
| ORDER-006 | Service | 対応期限を過ぎても発注業務が未完了                           | 対応対象者へ通知対象として登録                                                                      |
| ORDER-007 | Service | 対応期限より前に発注業務が完了                               | 未対応通知の対象にならない                                                                          |
| ORDER-008 | Service | 承認権限がないスタッフが承認                                 | 拒否                                                                                                |
| ORDER-009 | Service | 承認待ちの発注が期限を過ぎる                                 | 別の有効な承認権限者が対応可能、自動承認しない                                                      |
| ORDER-010 | Service | 発注数量の変更理由を選択して保存                             | 選択した理由を依頼または発注明細に保持                                                              |
| ORDER-011 | Service | 前回と異なる仕入単価を登録                                   | 同一仕入先・同一商品・同一発注単位の価格と比較                                                      |
| ORDER-012 | UI      | 仕入価格が前回より上昇                                       | 「上昇」と価格差を表示                                                                              |
| ORDER-013 | UI      | 仕入価格が前回より低下                                       | 「低下」と価格差を表示                                                                              |
| ORDER-014 | Service | 数量または仕入価格の変更理由に不正な値を指定                 | 許可リストで拒否                                                                                    |
| ORDER-015 | Service | 発注申請を行う                                               | 内容版1を不変保存し、期限・送信担当者を必須化                                                       |
| ORDER-016 | Service | 申請済み発注の期限が未設定                                   | 申請を拒否                                                                                          |
| ORDER-017 | Service | 承認期限が仕入先発注期限より後                               | 申請を拒否                                                                                          |
| ORDER-018 | Service | 取りまとめポジションの有効な承認者が承認                     | 発注全体の現行版を承認                                                                              |
| ORDER-019 | Service | 依頼元ポジションの権限だけで発注を承認                       | 取りまとめポジションの承認権限がなければ拒否                                                        |
| ORDER-020 | Service | 発注作成者が承認                                             | 拒否                                                                                                |
| ORDER-021 | Service | 承認履歴を登録                                               | 判断した内容版番号を保存                                                                            |
| ORDER-022 | Service | 承認済み発注の提案を採用                                     | 新しい版を作成し、旧承認を現行版の承認として扱わない                                                |
| ORDER-023 | Service | 未判断提案がある発注を承認                                   | 拒否                                                                                                |
| ORDER-024 | Service | 承認済み版と現在版が異なる発注を送信準備                     | 拒否                                                                                                |
| ORDER-025 | Service | cancel_line提案を採用                                        | 明細・依頼・提案履歴を保持し数量0にせず、判断者・提案理由・判断日時を取消明細に記録して再承認を要求 |
| ORDER-026 | Service | 取消済み明細をPDF・数量集計・納品照合に含める                | いずれからも除外                                                                                    |
| ORDER-027 | Service | 有効明細が0件の発注を送信準備                                | 拒否                                                                                                |
| ORDER-028 | Service | 承認済み発注に未判断提案がある                               | 送信を拒否                                                                                          |
| ORDER-029 | Service | PDF生成からFAX送信記録まで同じ版を維持                       | PDF版と送信済み版が一致                                                                             |
| ORDER-030 | Service | 送信準備中に内容編集・提案を登録                             | 拒否。準備中止後に提案を受け付け、採用時は新内容版の再承認を要求                                    |
| ORDER-031 | Service | 送信準備を中止                                               | 準備版を解除し、送信記録を作らない                                                                  |
| ORDER-032 | Service | 送信済み発注を通常編集                                       | 拒否し、元の送信版を維持                                                                            |
| ORDER-033 | Service | 承認待ちの期限超過通知を判定                                 | 当該発注を承認可能な有効スタッフを通知対象にする                                                    |
| ORDER-034 | Service | 承認済み未送信の期限超過通知を判定                           | `sending_assignee_id` を通知対象にする                                                              |
| ORDER-035 | DB      | 同一サイクル・同一担当への期限超過通知を重複登録             | 一意制約で拒否                                                                                      |
| ORDER-036 | Service | 期限または対応担当者を変更して再判定                         | 新しい期限・担当への通知対象を作成する                                                              |
| ORDER-037 | Service | 期限超過後に承認・送信担当者が対応                           | 自動承認・自動送信せず、手動操作で完了可能                                                          |
| ORDER-038 | Service | 送信準備中止後に新たな提案を登録し、未判断のまま送信         | 送信を拒否                                                                                          |
| ORDER-039 | Service | 承認済み発注の提案を不採用                                   | 内容版と既存承認を維持し、他の未判断提案がなければ送信可能                                          |
| ORDER-040 | Service | 取りまとめポジションに有効な承認者がいない                   | 管理者へ通知し、承認・送信を停止                                                                    |
| ORDER-041 | Service | 承認可能者不在で期限超過通知を判定                           | 店舗の有効な管理者を宛先とし、自動承認しない                                                        |
| ORDER-042 | Service | 承認済み発注を直接編集                                       | 拒否。内容変更は提案採用または差戻し後の再申請で行う                                                |
| ORDER-043 | DB      | 同一発注・同一担当・同一期限で異なる対応サイクルの通知を登録 | 別通知として登録できる                                                                              |
