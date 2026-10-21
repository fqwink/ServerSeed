# Phase 6 完全仕様案: サーバ構成

[文書一覧](../DOCUMENT_INDEX.md) | [担当工程](../ROADMAP.md#phase-6-サーバ構成)

本文は`B1`の採用未承認案。[案の適用範囲](PHASE_01.md#案の適用範囲)を適用する。対象は`REQ-001`・`REQ-002`・`REQ-004`・`REQ-007`・`REQ-008`・`REQ-009`・`REQ-011`・`REQ-012`。[構成固有仕様](../server/specification.md)は個別に維持し、本案で既存本文を移動しない。

## 契約と共通参照

| 契約ID | 対象 |
|---|---|
| `P06-CONFIG` | サーバ設定・アプリ入力の未定義部分の候補 |
| `P06-APP` | アプリの導入・更新・検証・復旧 |
| `P06-DATA` | アプリ固有の整合性・保全・復元検証 |

参照する契約（全フェーズB1）は`P02-PARSE`・[解析](PHASE_02.md#p02-parse)、`P02-EXEC`・[外部実行](PHASE_02.md#p02-exec)、`P02-STORE`・[記録](PHASE_02.md#p02-store)、`P04-UPDATE`・[更新](PHASE_04.md#p04-update)、`P04-APPLY`・[資源適用](PHASE_04.md#p04-apply)、`P04-BACKUP`・[保全](PHASE_04.md#p04-backup)。共通データ型は`P02-MODEL`とし、サーバ用の別状態基盤を作らない。

## P06-CONFIG

既存の[設定配置](../server/specification.md#構成固有設定)と[apps.ymlの定義](../server/specification.md#appsyml)は正本参照とする。以下は未定義フィールドを採用する場合の候補であり、実際に未知キーを受理できるよう変更したものではない。

| 候補 | 型・必須性・検証案 |
|---|---|
| server設定の形式版 | `schema_version`を必須integerとし候補版1。未知版を拒否 |
| アプリ実行の時間上限 | `operation_timeout_seconds`、必須integer、1〜3,600。未指定は補完せずエラーとする案 |
| ヘルスチェック | アプリ別の方式・実行入力・timeout・interval・試行回数を必須。httpとcommandの相互排他を検査 |
| command入力 | 実行ファイルの絶対パスと引数arrayへ構造化。既存の単一string例からの正確な変換規則はD06-01 |
| アプリ集合 | 最大100件、同名重複拒否。名前の許容文字、欠損ファイル・空配列の扱いは既存定義を使う |

HTTP確認は対象が返す状態コードと応答完了を判定する案とし、期待状態コードの許容値、HTTPSの扱い、本文の取得上限を`D06-02`で確定する。URLをシェルへ渡さず、command方式へ黙って切り替えない。ヘルスチェックの具体数値はアプリ定義に必須入力として持たせ、未定義の既定値で動作させない。

## P06-APP

採用案の実行単位はアプリ一件とする。対象OS・構成・Docker/Composeの固定採用情報、compose定義の識別と管理範囲、バックアップ・復元確認、必要容量、共有資源への影響を計画に結合する。

1. 入力・対象・選択構成と外部基盤の使用条件を照合する。
2. Compose定義とイメージの固定参照を取得し、前後の差分と復旧範囲を計画する。
3. P06-DATAの保全を確認し、P04-APPLYのpackage・systemd資源契約へ接続する。
4. 確認済みのアプリだけを更新・起動し、指定したヘルスチェックを実施する。
5. 検証・必須記録を確定する。失敗時は後続を停止し、保全した資材と実状態を照合して復旧計画を返す。

複数アプリは名前のASCII順で直列に処理する候補。失敗後の残りアプリを成功扱いにしない。全体の上限はP04-UPDATEと、各アプリの上限の小さい方に制限する。各コマンドの自動再試行は0回、ヘルスチェックの再検査だけは明示した回数・間隔に限定する。

イメージ版変更・データ形式変更・compose設定変更を分けて記録する。旧イメージの存在だけでデータの切戻しを保証しない。再起動・自動修復の許可条件は[サーバ運用](../POLICIES.md#サーバ構成)へ参照し、実装都合で初期対象を広げない。

## P06-DATA

アプリ別の保全契約案は、データ種別、整合性取得方式、事前停止の要否、取得コマンドの構造化引数、保存物の識別・版、復元方法、隔離確認先、確認の期待結果と有効期限を一組にする。共通の転送・保存・結果確定はP04-BACKUPを使う。

停止して取得する方式では、停止前の稼働状態と再起動条件を記録し、バックアップ失敗時のサービス復帰と元の要求の失敗を区別する。データベースの単なるファイルコピーを、整合したダンプ・復元確認として扱わない。外部アプリ固有のコマンド、版・例外・ライセンスは`D06-03`で個別評価する。

## 内製単位・API案

配置候補は`src/profiles/server/config.rs`・`apps.rs`・`data.rs`。構成入力とDocker呼出しの調整は内製、Docker/Composeそのものは個別採用条件が必要な外部基盤。共通処理への依存方向を逆転させず、Coreからサーバ固有設定を直接読む案にはしない。

| 宣言案 | 返却・副作用・失敗 |
|---|---|
| `server_plan(config: &Value, target: &TargetEvidence) -> Result<Value, ContractError>` | アプリ別の対象・差分・保全・復旧計画。変更なし |
| `apply_server(ctx: &mut ExecutionContext, plan: &Value) -> Result<Value, ContractError>` | アプリ別の適用・検証位置を返し、親操作へ結果を結合 |
| `check_app(app: &Value, ctx: &ExecutionContext) -> Result<Value, ContractError>` | 指定した検査の観測だけ。自動修復や更新なし |

公開型はP02-MODELを参照し、リクエスト・結果の厳密な全集合はD06-01〜03に含める。難易度は構成解析が中、アプリ固有のデータ互換・復旧が高。新設案で旧実装は存在しない。将来のcrate抽出にはP06-DATAと共通保全の境界、呼出元、共通契約テストと移行順序を先に確定する。

## データ契約の具体化案

`P06-CONFIG`・`P06-APP`の内部入力と検査結果を具体化する候補。外部のserver.yml・apps.ymlの正式キー全集合・版は`D06-01`に残し、下表をそのまま保存形式へ流用しない。

| 型 | 全キー・制約の候補 |
|---|---|
| CommandInput | executable: 絶対パスstring、args: string[]、timeout_seconds: 同フェーズの操作上限内integer。秘密情報の値を引数に含める契約は追加しない |
| HealthInput | kind: http・command、http: HttpInputまたはnull、command: CommandInputまたはnull、interval_seconds: 1〜300、attempts: 1〜10のinteger |
| HttpInput | url: string、expected_status: 100〜599のinteger[]、timeout_seconds: 1〜300のinteger、body_limit_bytes: 0〜1,048,576のinteger |
| AppPlan | name・compose_file・compose_sha256・image_digest: string、operation: install・update・recover、health: HealthInput、backup_method: ContractRefまたはnull、resources: P03-RESOURCEの資源集合 |
| HealthResult | name: string、status: pass・fail・unknown、attempts_used: 0以上integer、checks: Check[]、error: Diagnosticまたはnull |
| AppResult | name: string、receipt: OperationReceipt、health: HealthResult、data_verified: booleanまたはnull |

HealthInputはkindに対応する片方だけを非nullとし、期待状態コードは空・重複を拒否する。初回検査をattempts=1として数え、失敗した検査の終了後からinterval_seconds待って次の検査を行う。最終回後の待機・勝手な方式変更・無限再試行をしない。各試行はアプリ全体と親操作の残期限にも制限され、次の試行の予算がなければ開始しない。本文上限0は本文を不要とする候補で、応答完了の判定契約自体を省略しない。

HTTP URL・HTTPS・応答の解析・redirect・成功条件の外部契約は`D06-02`で正式資料と照合するまで未達。この表の値域をHTTP実装・取得基盤の採用と扱わない。command方式の非ゼロ・timeout・取得不能はP02-EXECの結果を保持し、自動修復は行わない。

既存backup.commandの単一stringは、全体を一つの絶対実行パスとして解釈しargs=[]へ対応付ける候補とする。空白・引用符・shell記号で分割せず、実際のパスとして検証できないものは拒否する。引数を持つ新しい構造化表現への変更は、元形式の識別・変換・切戻し条件を含む`D06-01`の別途採用事項である。

AppPlanのimage_digestとcompose_sha256は計画前に固定し、検査直前の実配置と照合する。コンテナの起動成功だけでHealthResultをpassにせず、保全・データ検査が必要な場合にdata_verified=nullをtrueへ補完しない。外部アプリのバージョン別の整合性取得・復元検査は`D06-03`へ対応付ける。

## 接続と責務の照合

check_appの名前は「読取り結果の評価」を意味し、[verifyの非変更条件](../POLICIES.md#コマンド別の変更可否)に反する任意コマンド実行の許可ではない。healthcheck.type=commandが読取り専用であることを個別の実行契約で確認できなければ、verify/doctorから呼ばずunknownとする候補。明示された更新操作内でも計画外の変更を許可しない。

HTTP検査の通信と[inspectの通信禁止](../POLICIES.md#読み取り専用検査)を区別する。inspectから到達性確認のHTTP要求を追加せず、アプリ検査の担当入口と実行可否が確定するまで`D06-02`を未達とする。取得基盤の選定は`D02-06`・`D04-01`の境界に接続し、標準net APIだけでHTTPSの充足を宣言しない。

マスターでの呼出しは[P08-COHOST](PHASE_08.md#p08-cohost)の同じAppPlanとAppResultを使う。masterという構成値とserverという機能所有者を混同せず、共通ロックをアプリごとに解除して並行更新へ移らない。保全の失敗後に元の稼働状態へ復帰させた場合も、バックアップ要求・更新要求の失敗は消さない。

## 受入ケース案

| ケースID | 区分・前提 | 期待結果と検証手段 |
|---|---|---|
| `A06-01` | 正常。apps欠損、空配列 | 既存定義どおりアプリなし。採掘・管理機能は追加しない |
| `A06-02` | 異常。アプリ重複・相対パス・不正型 | 変更前に入力エラー。先勝ち・shell評価なし |
| `A06-03` | 正常。固定版アプリの導入・再実行 | 指定資源を照合し、不要な書換え・再起動なし |
| `A06-04` | 異常。基盤未採用、image参照不定、保全未確認 | 適用開始なし。候補の自動選定をしない |
| `A06-05` | 異常。healthcheck失敗・時間超過 | 元操作の失敗と復旧可能範囲を表示。無許可自動修復なし |
| `A06-06` | 境界。アプリ件数上限、試行回数の最終回 | 上限内だけを処理し、実行回数・待機・終了を照合 |
| `A06-07` | 異常。backup失敗、形式非互換、restore期限切れ | 更新停止。旧imageだけで復旧成功にしない |
| `A06-08` | 正常。マスターから同じサーバ機能を呼出し | 同じ設定・処理・受入を使用。別のサーバ実装なし |

## 追加の受入ケース案

`P06-CONFIG`・`P06-APP`・`P06-DATA`の形式と非変更境界に対する設計。ネットワーク・外部コマンドは模擬結果、データ復元は隔離資材を使用する。

| ケースID | 区分・前提と入力 | 期待結果・確認方法 |
|---|---|---|
| `A06-09` | 異常。HealthInput.kind=httpでcommandも非null | 入力拒否。都合のよい方式を選んで実行しない |
| `A06-10` | 境界。attempts=1/10/11、最終試行が失敗 | 範囲内だけ検査。最終回後の待機・11回目の実行なし |
| `A06-11` | 異常。backup.commandに空白・shell記号を含むstring | 全体を一つのパスとして扱い、分割・展開0回。実在パスでなければ拒否 |
| `A06-12` | 異常。読取り専用と確認できないhealth commandをverifyから要求 | unknownとして実行しない。inspectでのHTTP通信も0回 |
| `A06-13` | 異常。保全後にcompose/image変更、データ復元の版不一致 | 計画前提を再照合し停止。ヘルスチェック成功だけでデータ検証済みにしない |

## 未決事項と成立判定

| 判断ID | 所有者の採用判断・調査対象 | 依存範囲 |
|---|---|---|
| `D06-01` | server.yml全集合、appsのcommand表現・版・移行とAPI | 設定・実行入力 |
| `D06-02` | HTTP/commandの完全な検査契約、入力範囲と出力上限 | ヘルスチェック・成功判定 |
| `D06-03` | Docker/Compose・アプリの個別採用と固定資材、データの確認・復旧 | 導入・更新・保全 |

既存構成別仕様の未確定部分は維持する。共通契約と候補の採用承認、非機能要求の不足が残るため、本案だけでサーバ機能を完全仕様成立・実装着手可能とはしない。
