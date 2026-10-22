# Phase 6 完全仕様案: サーバ構成

[文書一覧](../DOCUMENT_INDEX.md) | [担当工程](../ROADMAP.md#phase-6-サーバ構成)

本文は`B9`の採用未承認案。[案の適用範囲](PHASE_01.md#案の適用範囲)を適用する。対象は`REQ-001`・`REQ-002`・`REQ-004`・`REQ-007`・`REQ-008`・`REQ-009`・`REQ-011`・`REQ-012`。[構成固有仕様](../server/specification.md)は個別に維持し、本案で既存本文を移動しない。

## 契約と共通参照

| 契約ID | 対象 |
|---|---|
| `P06-CONFIG` | サーバ設定・アプリ入力の未定義部分の候補 |
| `P06-APP` | アプリの導入・更新・検証・復旧 |
| `P06-DATA` | アプリ固有の整合性・保全・復元検証 |

参照する契約（全フェーズB9）は`P02-PARSE`・[解析](PHASE_02.md#p02-parse)、`P02-EXEC`・[外部実行](PHASE_02.md#p02-exec)、`P02-STORE`・[記録](PHASE_02.md#p02-store)、`P04-UPDATE`・[更新](PHASE_04.md#p04-update)、`P04-APPLY`・[資源適用](PHASE_04.md#p04-apply)、`P04-BACKUP`・[保全](PHASE_04.md#p04-backup)。共通データ型は`P02-MODEL`とし、サーバ用の別状態基盤を作らない。

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

P06-DATAの採用審査はアプリ名/正確な版/Compose全サービス/digest/データ形式/取得・復元・検査コマンド/隔離先/TTLを一行で対応付ける方式を推奨する。各コマンドは実行物・argv・必要権限・許可副作用・出力物・期待終了を契約参照から一意に求める。停止取得なら元稼働状態を記録してから停止し、復帰は元が稼働していたサービスだけ。Dockerの共有network/volumeは所有範囲を別に確認し、project名一致だけで削除を許可しない。行の欠損/重複・復元先不明なら更新前blocked。

データ方法の照合キー候補はapp名・アプリ版・Compose全サービスのdigest集合・data_kind・保存形式版とする。BackupSet.methodはこの一意行のApplicationDataMethod.contractへ完全一致させ、capture/restore/verifyのどれかだけを別行から継ぎ合わせない。停止取得ではstop前の全サービス状態を記録し、一部停止失敗ならcaptureへ進まず、許可範囲内の復帰結果を個別に残す。隔離先が元データ領域・同一DBへ解決する場合、またはその同一性が不明ならDATA_ISOLATIONで復元確認を停止する。復帰不能・復元検査失敗はdata_verified=trueにしない。

## 内製単位・API案

配置候補は`src/profiles/server/config.rs`・`apps.rs`・`data.rs`。構成入力とDocker呼出しの調整は内製、Docker/Composeそのものは個別採用条件が必要な外部基盤。共通処理への依存方向を逆転させず、Coreからサーバ固有設定を直接読む案にはしない。

| 宣言案 | 返却・副作用・失敗 |
|---|---|
| `plan_server(input: &ServerInput, target: &TargetEvidence, compose: &[ComposeEvidence]) -> Result<Vec<AppPlan>, ContractError>` | 検証済み入力と取得済みCompose証拠から計画を所有して返す。変更・通信なし |
| アプリ適用の型付き境界 | [依存と受入](#依存と受入の確定候補)のAppPlan[]→ChangeResultに集約。開始後の部分位置を保持 |
| `check_app(plan: &AppPlan, context: CommandContext<'_>) -> Result<HealthResult, ContractError>` | 許可された検査だけ。自動修復なし、ReadOnly時は個別の非変更契約が必須 |

ComposeEvidence候補の全項目はname・project_name・compose_sha256・service_names・image_digests・contract。project_nameの生成/同定条件は[Composeの範囲](#composeの範囲とデータ方法行)だけを参照する。service_namesとimage_digestsのキーは同じ空でない集合、nameはAppConfigと一致、contractは採用済みのCompose観測契約とする。plan_serverは取得済み証拠しか使わず、浮動image解決やpullを行わない。apply_serverの結果列は入力AppPlan順と一対一に対応し、先行失敗後も後続をnot_startedのOperationReceiptとして保持する。HealthResultがpassでも必須のデータ検査/記録が不足すればAppResultを全体成功へ集約しない。

公開型はP02-MODELを参照し、リクエスト・結果の厳密な全集合はD06-01〜03に含める。難易度は構成解析が中、アプリ固有のデータ互換・復旧が高。新設案で旧実装は存在しない。将来のcrate抽出にはP06-DATAと共通保全の境界、呼出元、共通契約テストと移行順序を先に確定する。

## データ契約の具体化案

`P06-CONFIG`・`P06-APP`の内部入力と検査結果を具体化する候補。外部のserver.yml・apps.ymlの正式キー全集合・版は`D06-01`に残し、下表をそのまま保存形式へ流用しない。

| 型 | 全キー・制約の候補 |
|---|---|
| CommandInput | executable: 絶対パスstring、args: string[]、timeout_seconds: 同フェーズの操作上限内integer。秘密情報の値を引数に含める契約は追加しない |
| HealthInput | kind: http・command、http: HttpInputまたはnull、command: CommandInputまたはnull、interval_seconds: 1〜300、attempts: 1〜10のinteger |
| HttpInput | url: string、expected_status: 200〜299のinteger[]、timeout_seconds: 1〜300のinteger、body_limit_bytes: 0〜1,048,576のinteger |
| AppPlan | name・project_name・compose_file・compose_sha256: string、service_names: string[]、image_digests: サービス名→digestのobject、operation: install・update・recover、health: HealthInput、backup_method: ContractRefまたはnull、resources: P03-RESOURCEの資源集合 |
| HealthResult | name: string、status: pass・fail・unknown、attempts_used: 0以上integer、checks: Check[]、error: Diagnosticまたはnull |
| AppResult | name: string、receipt: OperationReceipt、health: HealthResult、data_verified: booleanまたはnull |

HealthInputはkindに対応する片方だけを非nullとし、期待状態コードは空・重複を拒否する。初回検査をattempts=1として数え、失敗した検査の終了後からinterval_seconds待って次の検査を行う。最終回後の待機・勝手な方式変更・無限再試行をしない。各試行はアプリ全体と親操作の残期限にも制限され、次の試行の予算がなければ開始しない。body_limit_bytesは受信する本文の厳密上限。0は空本文だけを許容し、本文を読まずに成功とする指定ではない。上限超過は失敗、最終応答の完了と期待status一致を両方要求する。

CommandInputの候補は構造化された呼出しパラメーターであり、任意実行の許可ではない。executable/argsをApplicationDataMethodまたはhealthの承認済み実行契約の全テンプレートへ一致させ、P02-EXECのmaterial・working_directory・environment・stdin・停止予約をその契約から束縛して初めてCommandSpecを生成する。契約にない引数、実行物の別名、NUL、暗黙shell、未固定のcwd/環境があればAPP_COMMANDで起動しない。timeout_secondsは正整数、healthではHttpInputと同じ時間範囲、取得/復元では当該操作の上限以下。msへのchecked変換後にも親の残期限と停止予約を照合する。

HTTP URL・HTTPS・応答の解析・redirect・成功条件の外部契約は`D06-02`で正式資料と照合するまで未達。この表の値域をHTTP実装・取得基盤の採用と扱わない。command方式の非ゼロ・timeout・取得不能はP02-EXECの結果を保持し、自動修復は行わない。

既存backup.commandの単一stringは、全体を一つの絶対実行パスとして解釈しargs=[]へ対応付ける候補とする。空白・引用符・shell記号で分割せず、実際のパスとして検証できないものは拒否する。引数を持つ新しい構造化表現への変更は、元形式の識別・変換・切戻し条件を含む`D06-01`の別途採用事項である。

AppPlanのimage_digestsとcompose_sha256は計画前に固定し、検査直前の実配置と照合する。コンテナの起動成功だけでHealthResultをpassにせず、保全・データ検査が必要な場合にdata_verified=nullをtrueへ補完しない。外部アプリのバージョン別の整合性取得・復元検査は`D06-03`へ対応付ける。

service_namesはComposeEvidenceの非空・一意な全サービス集合をASCII順に保持する候補。image_digestsのキー集合と完全一致させ、停止前観測/停止/復帰の各結果もこの集合に対応付ける。AppPlanにないサービスをproject名だけで停止・削除しない。image/composeの一つでも変われば方法照合と計画確認へ戻し、以前のdata_verifiedを再利用しない。

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

## 型とAPIの詳細案

`P06-CONFIG`の方式別入力をenumにする候補。CommandInput・HttpInputの全フィールドと範囲は[データ契約](#データ契約の具体化案)へ一度だけ定義し、下記では同名structとして使用する。

```rust
pub enum HealthMethod { Http(HttpInput), Command(CommandInput) }
pub struct HttpInput {
    url: String, expected_status: Vec<u16>, timeout_seconds: u32, body_limit_bytes: u64,
}
pub struct CommandInput {
    executable: std::path::PathBuf, args: Vec<String>, timeout_seconds: u32,
}
pub struct HealthInput { method: HealthMethod, interval_seconds: u32, attempts: u8 }
pub enum HealthStatus { Pass, Fail, Unknown }
pub struct HealthResult {
    name: String, status: HealthStatus, attempts_used: u8,
    checks: Vec<Check>, error: Option<Diagnostic>,
}
pub struct ServerConfig { schema_version: u32, operation_timeout_seconds: u32 }
pub struct AppConfig {
    name: String, enabled: bool, compose_file: std::path::PathBuf,
    healthcheck: HealthInput, backup: AppBackup,
}
pub struct AppBackup { required: bool, method: Option<ApplicationDataMethod> }
pub struct ApplicationDataMethod {
    contract: ContractRef, data_kind: String, consistency: String,
    stop: Option<CommandInput>, capture: CommandInput, restore: CommandInput,
    verify: CommandInput, restart: Option<CommandInput>,
    restore_ttl_seconds: u32, isolated_target_ref: String,
}
pub struct ComposeEvidence {
    name: String, project_name: String, compose_sha256: String, service_names: Vec<String>,
    image_digests: std::collections::BTreeMap<String, String>, contract: ContractRef,
}
pub struct AppPlan {
    name: String, project_name: String, compose_file: std::path::PathBuf,
    compose_sha256: String, service_names: Vec<String>,
    image_digests: std::collections::BTreeMap<String, String>, operation: String,
    health: HealthInput, backup_method: Option<ContractRef>, resources: ResourceSet,
}
pub struct AppResult {
    name: String, receipt: OperationReceipt, health: HealthResult, data_verified: Option<bool>,
}
pub struct ServerInput { config: ServerConfig, apps: Vec<AppConfig> }
pub fn validate_server(config: &Value, apps: Option<&Value>)
    -> Result<ServerInput, ContractError>;
```

外部HealthInputのhttp/commandの片方が非nullという形式から一つのenumへ変換し、両方・両方なしを拒否する。HealthResult.attempts_usedは要求のattempts以下、開始前停止は0、開始済みで結果不明の試行も1回へ数える。検査結果不明を再試行未消費へ戻さない。ServerConfigはserver.yml候補のキー全集合とし、未知キー・null・未指定の補完なし。apps欠損は既存どおり空集合へ対応するが、読取り不能を欠損にしない。

HealthResultのchecks/errorは[共通CLI型](PHASE_02.md#検査観測の固定集合)を使い、契約内部のContractErrorを外部Diagnosticの別形式として出さない。passはattempts_used>=1、error=None、全必須Check=passを必要とする。failは確定した不一致とDiagnostic、unknownは取得不能/停止不明のDiagnosticを必要とする。検査集合欠損をpassにせず、実開始した失敗試行数を保持する。初回pass後の余分な試行は行わない。

| ケースID | 固定入力 | 期待結果 |
|---|---|---|
| `A06-32` | attempts_used=0でpass、errorに未知キー、必須Check一件unknown | 応答validatorで拒否またはunknownを保持。Value経由の未検証出力・未実施成功なし |

AppPlanのimage_digestsのRust表現は`BTreeMap<String, String>`。キーはCompose内のサービス名、値は固定したイメージdigest。サービス集合と完全一致が必要で、サービスが複数でも一つのdigestで代表しない。

AppResult.nameとhealth.nameは対応するAppPlan.nameへ完全一致させる。OperationReceiptは親全stepを持つため、アプリ固有stepの対応はPlanStep.contractと担当アプリ名で照合し、他アプリの成功receiptを転用しない。data_verified=Noneは当該操作でデータ検証を要求しない明示根拠がある場合だけで、必要な検証が未実施ならSome(false)と未達Checkを保持する。判定不能はCheckのunknownへ残す。

| ケースID | 固定入力 | 期待結果 |
|---|---|---|
| `A06-33` | app aへbのhealth、必須データ検証未実施なのにdata_verified=null | APP_BINDINGで応答拒否。正常な別アプリ結果・nullで検証不足を隠さない |

## 状態と中断の詳細案

`P06-APP`のアプリ別工程候補はchecked→preserved→applying→checking_health→verified→recorded。必須保全が不要なアプリでも、その適用外の承認済み根拠を持つ検査結果が必要。名前順で処理し、先行失敗後のアプリはnot_startedとして保持する。

| 失敗・事象 | 動作候補 |
|---|---|
| fixed image/compose/例外条件が不足 | checked以前で停止。pull/up/backupを開始しない |
| 保全コマンド非ゼロ・保存検証失敗 | applyingへ進まない。元稼働状態への復帰結果は別に記録 |
| Compose変更中の失敗・応答欠損 | 実配置・サービス状態・記録を照合。upの自動再実行0回 |
| ヘルスチェック非成立 | failまたはunknownを保持し、サーバの自動修復へ変換しない |
| ヘルス試行中に親期限到達 | 次試行・待機を開始せず停止。実行済み回数を保持 |
| health成功、データ復元確認失敗 | 全体未完了。旧imageの起動だけでデータ復旧成功にしない |
| 検証済み、必須記録だけ失敗 | verifiedとrecordedを分離。全体成功なし |

read-only確認の可否が不明なcommandは、verify時にはunknownで実行しない。更新時でも計画外の副作用がある検査へ置き換えない。復旧は元のimage集合・compose・データ形式の互換性を照合した明示計画だけとし、可逆性が確認できないアプリの自動ダウングレードを行わない。

## 依存と互換性の詳細案

サーバ機能はEX-003のDocker/Composeとアプリ別の採用情報へ依存する候補。Composeのパーサー・エンジンをRustへ取り込まず、ServerSeedは構成・計画・結果の調整だけを内製する。

| 境界 | 固定・検証の候補 |
|---|---|
| アプリ全サービス | compose内容の固定ハッシュとimage_digestsの全サービス集合。浮動tagだけでは不可 |
| command health/backup | 絶対実行物・args・上限・read-only可否・副作用の個別契約。shell文字列なし |
| HTTP health | [RFC 9110](https://datatracker.ietf.org/doc/html/rfc9110)を照合元とし、方式・URL・応答完了・状態・redirect・TLSと上限を個別確定 |
| データ形式 | アプリ版・保存形式・整合性取得・復元方法・有効期限を組で固定 |
| 設定移行 | 旧backup.commandは全体を一つのパスとして扱い、構造化argsを追加するなら識別版・変換・切戻しを明示 |

HTTPの第一候補は固定GET、redirectなし、userinfo/fragmentなし、明示した接続先だけへの要求。最終応答と取得完了を必要とし、1xxだけ・ヘッダーだけの到着を成功にしない。これは製品側の制限案で、HTTP標準自体の制限という意味ではない。TLS/URL/転送実装と全応答上限は`D06-02`で未達のまま、inspectには通信を追加しない。

応答量の候補では全status/header/trailerの元byte合計を64 KiB以内、暫定1xxは16応答以内とし、どちらも最終応答前から累積する。本文量はHTTPの転送フレーミングを除いた受信内容byteで計数する。Accept-Encodingはidentityを要求し、Content-Encodingが未指定/identity以外ならHEALTH_ENCODINGで失敗、自動展開しない。[RFC 9110](https://www.rfc-editor.org/rfc/rfc9110.html)の表現符号化と転送完了の区別へ照合する候補で、HTTP全実装の選定ではない。Content-Lengthだけを実量の証明にせず、終了不正・途中EOF・矛盾したframing・trailer量超過もHEALTH_RESPONSE。生header/bodyは診断へ転記せず、全条件を満たした最終応答だけ期待statusへ照合する。

新設・差し替えの比較では空アプリ・多サービス・部分失敗・保全/復元失敗を同じ入力で照合する。マスターからも同じモジュールを呼び、別のDocker制御を複製しない。

## 受入条件の充足照合案

CONFIGはserver.ymlとapps入力、APPは全サービスの固定とhealth、DATAは対象固有の保全/復元へ対応する。HTTPの方式候補は[依存境界](#依存と互換性の詳細案)に集約し、実行adapterの採用がない間は方式名だけで実装着手可能としない。

| ケースID | 区分・固定入力 | 期待結果と確認 |
|---|---|---|
| `A06-14` | 異常。Composeがweb/dbの2サービス、image_digestsはwebだけ | 集合不一致で適用停止。片方のimage固定を全体の固定にしない |
| `A06-15` | 境界。最終health試行を開始後、応答欠損 | attempts_usedは実開始を含む。unknownを0回へ戻して余分な試行をしない |
| `A06-16` | 異常。HTTPは1xxだけ、またはredirectで別URLへ誘導 | 最終応答・固定要求条件が不成立。候補方式の自動追従なし |

アプリ別の復元方法・データ形式・期限と、Docker/Compose/アプリの個別採用情報は未達として残る。構文・設定型の充足と、実行・データ復旧の充足を分け、空アプリ集合の成功を任意アプリの対応済みへ一般化しない。

## 入力と保存形式の詳細案

P06-CONFIGの新形式候補はapps.ymlのrootをschema_version: 1とapps: AppConfig[]の二キーとする。server.ymlは先述のschema_versionとoperation_timeout_secondsだけ。既存apps例は旧形式の資料として維持し、無承認の変換はしない。

| 型 | 全キー・型・必須性 |
|---|---|
| AppConfig | name: 既存名前制約、enabled: bool、compose_file: 絶対path、healthcheck: HealthInput、backup: AppBackup。全キー必須 |
| AppBackup | required: bool、method: ApplicationDataMethodまたはnull。required=trueならmethod必須。falseでも重要データ等の保護条件を免除しない |
| ApplicationDataMethod | contract: ContractRef、data_kind: application/database、consistency: stopped/native_dump、stop: CommandInputまたはnull、capture・restore・verify: CommandInput、restart: CommandInputまたはnull、restore_ttl_seconds: u32、isolated_target_ref: string |
| 条件 | stoppedはstop/restart必須、native_dumpは両方null。各コマンドの実対象・非変更範囲・期待終了/成果物をcontractへ対応付け、単に終了0だけで整合性を認定しない |
| 不在・null | apps.ymlの不在は既存仕様どおりアプリなし。読取り不能はエラー。新形式のapps=[]は明示的な空集合、apps=null・backup省略は形式エラー |

旧healthcheck.typeから新kind、旧backup.commandからcaptureへの自動変換は行わない。旧一文字列を一実行pathとする規則だけではrestore/verify/TTL/隔離先を作れないため、これらを所有者が指定した新形式を別計画として検証する。enabled=falseも構文検査対象とし、無効な危険設定を隠さない。無効化を既存コンテナ・データの自動除去と解釈しない。

validate_serverの候補順序はserverの形式/版/全値→apps不在か読取り失敗か→apps形式/版→各AppConfigを掲載順に検証→名前重複→機能適用条件とする。null/空文字/欠損を相互に補完しない。掲載順は入力エラーの位置だけへ使用し、実行順は検証完了後のアプリ名ASCII順。enabled=falseでも全health/backup値を検証するが、正常に無効と指定されたアプリから適用stepや停止/除去stepを生成しない。apps=[]の正常判定とDocker基盤・サーバ構成全体の受入は別に確認する。

AppPlanのresourcesはP03-RESOURCE、実適用はP04-APPLY、データの保存はP04-BACKUPへ渡す。backup.method=nullで重要データ変更を伴う計画は、データ保護条件を別に満たせない限りblocked。形式のnull許容を保全免除にしない。restore_ttl_secondsはP04-BACKUPと同じ正整数範囲へ照合する。

| ケースID | 固定入力 | 期待結果 |
|---|---|---|
| `A06-17` | 旧backup.commandだけの設定、新形式でstoppedだがrestart=null、enabled=falseに未知キー | 必須契約不足または形式不正。shell分割・暗黙の復元手順・無効項目の黙認なし |

## 実行と復旧の確定候補

P06-APPのアプリ工程をchecked→preserved→applying→checking_health→verified→recordedの候補へ固定する。failed/unknownでは次アプリを開始しない。AppResultは元のAppPlanと同じcompose/image/data method参照を保持し、検査時に別のlatestへ置き換えない。

ヘルス検査は最初の要求開始をattempts_usedへ確定してから起動する。timeout/応答欠損も消費した一回であり、別プロセスの再開で0へ戻さない。次回は前試行終了時刻からintervalを待ち、親期限の残量がなければ未実施。HTTP 1xxだけでは完結としない。候補のGETはredirect非追従、userinfo/fragment拒否、本文上限を1 byte超えたら失敗とし、body_limit=0も同じ厳密上限として非空本文を拒否する。

試行集約候補は、応答完了かつ期待status/本文上限の全成立時に直ちにpassで終了し、以後の試行を開始しない方式。確定failは残回数/期限があれば再検査し、全試行failならfail。停止確認不能・試行状態不明ではunknownを返し、先行試行が残ったまま次を重ねない。親期限が試行間で尽きた場合も未完了を保持する。HTTP adapterのheader量/終了条件とcommandの非変更契約はD06-02に残し、curlのbody_limit=0を無制限指定へ変換しない。

停止取得の保全では、元の稼働有無を記録してからstopし、停止確認後だけcaptureする。元が停止中なら終了時に勝手に起動しない。capture後の復帰失敗と保全内容の成否は別結果で、復帰不能のまま更新を開始しない。DBの復元検証は稼働DBへ接続せず、methodで固定した隔離先だけへ適用する。

検査失敗後に旧imageへ戻せる条件は旧image/composeとデータ形式の互換が成立し、復元先・保全物・復帰条件を確認できる場合に限る。不可逆migration後のimage切替を復旧成功にしない。復旧も利用者の当該計画への確認範囲を必要とする。

| ケースID | 固定入力 | 期待結果 |
|---|---|---|
| `A06-18` | 初回HTTP開始後中断、再開時attempts=1、capture元サービスは元から停止 | 検査回数を巻き戻さない。サービスを起動せず、残予算/試行条件を照合 |

## 依存と受入の確定候補

P06-CONFIGはAppConfig/ServerConfigの検証、P06-APPは共通計画/適用への調整、P06-DATAは個別データ方法の契約照合を担当する候補。Docker/Compose・HTTP/TLS・DB実行物を新しいRust依存へ置換せず、例外採用と内製判断を分ける。

| 境界 | 比較・受入条件 |
|---|---|
| config→plan | AppConfigと固定compose/image集合からAppPlanを所有して返す。enabled=falseでも形式検査を行い、既存データの除去を計画しない |
| apply→result | apply_serverの型付き入力はAppPlan[]、結果はAppResult[]を包むChangeResult。開始後の全失敗にpartial必須 |
| health | HealthInputと残予算からHealthResult。command非変更性・HTTP応答完了・試行数は同じ基準、失敗後の方式切替なし |
| data | ApplicationDataMethodとBackupSetを結合し、同じapp/版/形式/隔離先の復元確認がなければdata_verified=trueにしない |

同じAppPlanをstandalone serverとmasterのサーバ機能へ入力し、対象・制約が同じなら同じ変更/検査/失敗を返すことを受入条件とする。難易度はschema/計画調整が中、データ互換/復元が高。後者を共通file-copyへ吸収して内製完了にしない。

将来のcrate差し替えでは設定validator・健康判定・実行adapterを個別に比較し、共通保存やOS実行器を重複実装しない。API変更時は全呼出元・旧形式の取扱い・lockfile切替と除去条件を同じ変更案へ追跡する。

| ケースID | 固定入力 | 期待結果 |
|---|---|---|
| `A06-19` | 同じAppPlanをserver/masterへ入力、DB復元fixtureだけ別版 | サーバ部分の結果は同条件で一致。別版の復元を成功へせず、同居全体も未完了 |
| `A06-20` | enabled=falseだが型不正、旧工程名validated、旧backup.commandだけ | 形式/工程不一致で拒否。無効項目の黙認・第二状態列・暗黙の移行なし |
| `A06-21` | body_limit_bytes=0で本文0/1byte、期待status=199/200/300 | 空本文と2xx入力だけ候補条件を通過。超過本文・1xx/3xx期待は拒否、ヘッダー到着だけで成功にしない |
| `A06-22` | 同じapp版にデータ方法二行、共有volumeの所有不明、元は停止済み | 前二件は開始前停止。保全後に元停止サービスを勝手に起動・共有データを除去しない |
| `A06-23` | 入力順z,aで両方妥当、enabled=falseのa、次にaへ不正health追加 | 妥当時の適用はzのみ。不正時はaも検証拒否。入力順・実行順・無効化の意味を区別 |
| `A06-24` | app a失敗後にbが未開始、Compose証拠のサービス一件欠損 | bのnot_startedを保持。証拠不足時は計画前停止、pull/自動修復/結果列の省略なし |
| `A06-25` | captureとrestoreが別方法行、二サービス中一方のstop失敗、隔離参照が本番DBへ解決 | 方法不一致/取得前停止/DATA_ISOLATIONを区別。capture・本番復元を実行せず元失敗を保持 |
| `A06-26` | Compose全三サービスのうちPlanは二件、未計画サービスが同projectに存在 | 計画不成立/対象外不変。共有projectを理由に停止・削除対象を拡張しない |
| `A06-27` | attempts=3で初回pass、別入力では初回停止不明、全三回は確定fail | 順に一回でpass/一回でunknown/三回でfail。余分な試行・重複実行なし |
| `A06-28` | 正しい絶対実行pathに未承認引数、cwd契約なし、health時間が範囲外 | APP_COMMAND/値域検査で起動0回。構造化配列というだけで任意実行・環境継承を許可しない |
| `A06-29` | 1xxを17件、累積header量超過、gzip本文、期待2xxだが途中EOF | 応答上限/HEALTH_ENCODING/HEALTH_RESPONSEで失敗。本文だけの上限・status一致でpassにしない |

## 未決事項と成立判定

### health URLと外部adapter

HttpInput.urlの候補はASCII http/httpsだけとし、単一のDNS名/IPv4/角括弧IPv6、port省略時80/443、/始まりpathと任意queryを受理する。userinfo/fragment/空白/制御文字/不正percent escape/percent復号後の制御文字を拒否し、hostnameやpathをshellへ渡さない。採用済みアプリのhealth契約が許可するorigin/path範囲へ一致させ、任意の外部URLを設定できるという意味にしない。HTTPSの証明書/名前検証は維持する。

HTTP実行器の第一候補はP04で個別審査する同じcurl実行物。全argv候補は固定path、--disable、--globoff、--proto、=http,https、--request、GET、--connect-timeout、当該試行の残通常秒、--max-time、同じ値、--retry、0、--silent、--show-error、--header、Accept-Encoding: identity、--dump-header、/dev/stderr、--write-out、固定のstderr用完了印とresponse_code、--url、検証済みURL。--location/--compressed/--insecureなし、環境LANG=C/LC_ALL=Cだけ、cwd=/、stdin空。/dev/stderrは当該子の取得pipeへの既知経路としてのみ使い、任意file出力は指定しない。

stdoutは本文だけ、stderrはheader/trailerと固定完了印/エラーの一時取得へ分ける候補。通常captureの各1 MiB上限を維持し、追加で本文をbody_limit、HTTP header/trailerを64 KiBへ計数する。body_limit=0の際も一byte受信で超過を検出し、curlの0=無制限optionへ変換しない。ヘッダー到着順と1xx/最終応答/trailerを構造化して検査し、stderr末尾の固定完了印も一件だけ要求する。未知の診断混入・不完全応答・非ゼロ・停止不明をpassにしない。本文/header/URLの生出力はレポートへ出さない。

上記は個別固定版での比較契約であり、curl出力の版別文法、HTTP/1.1/2のframingとtrailerの表現、/dev/stderr経路の保護、ReadOnly契約への適合をfixtureで照合する必要がある。採用版で完了印と応答を一意に分離できなければこのadapterは不成立で、自由な外部HTTP clientへ置換しない。P04の配布取得とは異なり、healthは保存fileを作らない。

| ケースID | 固定入力 | 期待結果 |
|---|---|---|
| `A06-30` | userinfo/encoded改行URL、body_limit=0へ一byte、header上限、完了印なし/重複、TLS名前不一致 | URL/量/完了/TLSを個別拒否。file保存・curlの無制限0・生応答公開・別client自動選択なし |

### Composeの範囲とデータ方法行

ComposeEvidence/AppPlanへproject_nameを必須追加する候補。値はss-とアプリ名の元UTF-8 byte列のSHA-256全64桁。既存の大文字/小文字を区別するnameをlowercaseへ潰さず、hashの外部取得証拠を計画前に照合する。異なるnameで同じproject名または既存の別所有projectへ対応したらAPP_PROJECTで停止。plan_server自身はhash実行/接続をしない。

docker argvの共通prefix候補は固定path、--config、保護された採用済みDocker設定directory、compose、--ansi、never、--parallel、1、--project-name、project_name、--project-directory、固定composeの親、--env-file、/dev/null、--file、固定compose path。環境はLANG=C/LC_ALL=C/DOCKER_HOST=unix:///var/run/docker.sockだけ、cwdは固定親directory、stdin空。ユーザーのHOME/COMPOSE_*やリモートdaemonを継承しない。

| 操作 | prefix後のargvと照合候補 |
|---|---|
| 設定観測 | config、--format、json、--no-interpolate、--no-env-resolution。未展開変数・env_file/include/extendsの未固定参照は拒否し、暗黙読取り/取得しない |
| 稼働観測 | ps、--all、--format、json。全サービス名と固定image/所有label/稼働状態を対応付け、同project名だけで所有認定しない |
| 資材準備 | pull、--、固定した全サービス列。変更計画内だけ、digestと取得後の実imageを再照合 |
| 適用 | up、--detach、--no-build、--pull、never、--no-deps、--、計画対象サービス列。未計画のbuild/pull/依存起動なし |
| 停止/復帰 | stopまたはstart、--、元状態/保全計画で確定したサービス列。down/rm/prune/remove-orphans/volume削除なし |

[Compose config](https://docs.docker.com/reference/cli/docker/compose/config/)と[up](https://docs.docker.com/reference/cli/docker/compose/up/)へ照合した候補。終了0でも全サービス/health/data/journalの照合を必要とする。compose内部のhooks・privileged・host共有・bind mount等もアプリの採用範囲へ含め、引数配列で呼ぶだけで安全としない。固定版でoption/出力/副作用が異なる場合は不成立で、別版へ自動切替しない。

アプリ別データ方法行の必須キー候補はapp_name・exact_version・compose_sha256・image_digests・data_format_version・method・source_entries・isolated_target_ref・expected_checks・restore_ttl_seconds・compatibility_ref。methodはApplicationDataMethod一件だけ、source_entriesはBackupEntryの明示集合、expected_checksは契約付き検査IDの非空集合とする。行の新設は対象アプリ/版の所有者指定と外部依存の個別採用を必要とする。現時点の対応行は0件で、任意DBを「ファイルコピー」で自動的に対応済みにしない。

| ケースID | 固定入力 | 期待結果 |
|---|---|---|
| `A06-31` | nameがApp/app、未固定env_file、未計画service、image版だけ異なるデータ方法行 | caseを保つ別projectとし、未固定/範囲外/方法不一致は拒否。環境継承・依存の暗黙起動・任意DB対応の認定なし |

| 判断ID | 所有者の採用判断・調査対象 | 依存範囲 |
|---|---|---|
| `D06-01` | AppConfigと工程名に案あり。新形式/旧形式の移行、全子型閉包と採用判断が残る。enabled=falseを停止/除去へ変換しない | サーバ入力 |
| `D06-02` | [health adapter](#health-urlと外部adapter)にURL/argv案あり。固定版のHTTP構文/出力分離・終了観測とcommandの非変更性の根拠が残る | health判定 |
| `D06-03` | [Composeとデータ方法行](#composeの範囲とデータ方法行)に形式/argv案あり。実アプリ未指定、採用済み方法行0件。Docker/アプリ版・隔離/復元/TTLの個別確定が必要 | アプリ導入・更新・保全 |

既存構成別仕様の未確定部分は維持する。共通契約と候補の採用承認、非機能要求の不足が残るため、本案だけでサーバ機能を完全仕様成立・実装着手可能とはしない。
