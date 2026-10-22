# Phase 2 完全仕様案: 共通契約と構成選択

[文書一覧](../DOCUMENT_INDEX.md) | [担当工程](../ROADMAP.md#phase-2-共通契約と構成選択)

案版`B11`・採用未承認。[案の適用範囲](PHASE_01.md#案の適用範囲)に従う。既存の設定値・状態・終了コード・記録例は基本仕様が正本であり、本書では未定義部分の候補を扱う。共通型・CLI・解析・状態・記録・外部実行の具体化と受入設計を収録する。契約内容の採用や既存仕様の切替を含めない。

## 要件と契約の対応

対象要件は`REQ-001`・`REQ-002`・`REQ-003`・`REQ-004`・`REQ-006`・`REQ-007`・`REQ-008`・`REQ-009`・`REQ-011`・`REQ-012`。

| 契約ID | 担当と参照元 |
|---|---|
| `P02-MODEL` | 識別・時刻・エラーの共通型。[共通管理記録](../BASIC_SPEC.md#共通管理記録の未確定契約) |
| `P02-CLI` | 共通入力・診断・出力。[共通CLI](../BASIC_SPEC.md#共通cliの未確定契約) |
| `P02-PARSE` | 形式解析・スキーマ検証。[構成定義の解析](../BASIC_SPEC.md#構成定義の解析) |
| `P02-STATE` | 対象・三構成・状態判定。[状態判定の優先順位](../BASIC_SPEC.md#状態判定の優先順位) |
| `P02-STORE` | 排他・記録・原子性。[共通基盤](../BASIC_SPEC.md#共通基盤と状態管理) |
| `P02-EXEC` | 外部実行の制御。[外部コマンド](../BASIC_SPEC.md#外部コマンドの未確定契約) |

`P01-CLI`・`P01-CHECK`はB11の[Phase 1](PHASE_01.md#要件と境界)を参照する。Phase 2を使用する後続フェーズは必要な本書の契約IDと節を個別に参照し、共通値を再定義しない。

## P02-MODEL

採用案の共通型を以下に置く。配置候補は`src/core/model.rs`。検証済みの対象・識別・実行型は生成関数で検証し、フィールドを外部から直接変更させない。汎用の`Value`は構文上の値を表し、対象・設定・計画の検証済み型の代わりにしない。

| 型 | 内容・生成条件案 |
|---|---|
| `TargetEvidence` | 実行環境ID、対象ディスクの安定識別子、対象ルート、対象OS識別、観測時点。未取得値は`None`と理由を持ち、実行環境の値で補完しない |
| `PlanId`・`OperationId` | 接頭辞`plan-`・`op-`と32桁の小文字16進数。16バイトのOS乱数を取得し、一意性を確認。乱数取得・照合不能なら失敗 |
| `MachineId` | ServerSeed固有の32桁小文字16進数。構成初回確定時に生成し、OSのmachine-idとは別に保持。復元時の継承は`P04-BACKUP`で審査 |
| `ContractError` | `code: String`、`message: String`、`field: Option<String>`。コードは契約別の固定識別子、表示文は秘匿化済み。内部エラーをそのまま転記しない |
| `Value` | 宣言・数値の表現・所有権は[P02-PARSEの型とAPI](#解析の型とapi案)に集約。担当スキーマの検証前に適用可能とは扱わない |
| `ExecutionContext` | 対象証拠を持つOperationGuardを借用し、計画/操作ID・期限・取消状態を所有する。型の具体化は本書の型とAPIの詳細案を参照。単なるJSONの読み込みだけでは生成しない |

時刻保存はUTCのRFC3339、秒未満3桁、末尾`Z`へ正規化する採用案とする。期限判定は経過時間と絶対期限を分離し、時計後退・再起動後の期限不明は有効と推測しない。乱数の取得手段・OS識別情報との結合は`D02-01`の調査対象であり、識別子生成を実装可能と判定しない。

ID生成の純粋境界候補は`encode_id(kind: IdKind, entropy: &[u8; 16], existing: &[String]) -> Result<String, ContractError>`。IdKindはPlan/Operation/Machineで、表記は上表だけを使用する。入力entropyを保存・出力せず、生成後のIDだけを返す。既存集合の取得不能を空集合に置換せず、衝突では一度でMODEL_ID_COLLISION。OS取得器と表記器を分離する案を推奨し、時刻/PIDの連結を乱数の代替にする案は採らない。プロセスの照合キー候補はboot_id・pid・process_start_ticksの三つで、同一採取前後にキーが変わればMODEL_CHANGED、読めない場合はMODEL_UNREADABLE。PIDだけの一致を根拠にしない。

プロセス取得の候補元は、同じ対象起動世代の/proc/sys/kernel/random/boot_idと/proc/PID/stat。[Linux proc資料](https://www.kernel.org/doc/html/latest/filesystems/proc.html)と[kernelのboot_id](https://www.kernel.org/doc/html/latest/admin-guide/sysctl/kernel.html)へ照合する。boot_idはUUIDの小文字hex8-4-4-4-12と末尾LF一つ、statは最大4 KiBの元byte列として読む候補。statは先頭pidと括弧付きcommを区別し、最後の閉じ括弧の後を第3フィールド以降として数え、第22フィールドstarttimeを非負十進u64へ検証する。commの空白/閉じ括弧/非UTF-8を通常の空白区切りで誤解析しない。PID不一致・欠損・上限超過はMODEL_UNREADABLE。PIDの存在/不在と読み取り不能を区別し、別PID namespace・procの差替え・起動世代不明を同一主体にしない。収集前後でboot/PID/startが一致して初めて採用する。tickをUTC秒へ換算して照合する方式ではない。乱数源・取得期限・対象環境での成立確認はD02-01に残す。

### 共通参照型の候補

ContractRefの全キーはfile・anchor・contract_id・draft・commit・adopted。前5項目は空でないstring、adoptedはboolean。fileは文書索引が案内するリポジトリ相対パス、anchorはその文書の節、contract_idはその節の契約ID、draftは現在の案版、commitは参照を評価する確定ツリーの完全なコミットIDとする。採用審査用と採用済みの参照をadoptedで区別し、値だけで承認の存在を証明せず所有者の承認根拠と照合する。作成途中の文書には架空のコミットを埋めず、提出コミットができた時点の確認資料で参照集合を束縛する。

ContractRefの検証候補は、fileのUTF-8相対パスと索引所属→commitの完全ID/実在→当該ツリー内のfile/anchor/contract_id→draft一致→adoptedと所有者の採用根拠、の順。絶対path・空成分・親逸脱・別ツリーの同名節を拒否する。失敗コードはMODEL_REFERENCEとし、adopted=falseは文書案目的でだけ受理、実装着手目的では拒否する。参照先の自己申告だけでadopted=trueを構築しない。型検証器は取得済みの文書/承認資料を借用し、Git取得・ネットワーク・ファイル修復を行わない。

以下の追加データ契約は、特記がなければ表に列挙したキーを全集合とする候補。すべて必須、null許容は明記した箇所だけ、未知キー・欠損・重複・暗黙の型変換を拒否する。objectの空集合と未知集合を区別し、arrayの順序・件数は担当節へ照合する。stringは解析の文字列上限内、integerはi64の範囲内とし、担当の狭い範囲が優先する。外部入力を検証した後の内部値にも同じ制約を適用する。これは各担当データの未記載項目を補完する包括スキーマではない。

## P02-CLI

### 入力案

担当は`REQ-004`・`REQ-006`・`REQ-007`・`REQ-008`・`REQ-011`・`REQ-012`の共通入出力。既存の[CLI](../BASIC_SPEC.md#cli)へ接続する採用候補であり、媒体・マスター・独立バックアップの新しいコマンドを追加しない。引数解析・診断集約・整形を内製し、初期配置は`src/cli/`内のモジュールとする。旧実装・再利用できる内製CLIはなく、独立crate・外部crateを追加しない。

2026-10-20に採用Rust版の[OsStr::to_str](https://doc.rust-lang.org/std/ffi/struct.OsStr.html#method.to_str)と[Write](https://doc.rust-lang.org/std/io/trait.Write.html)を照合した。前者の変換不能と後者の部分書込み・失敗を考慮するが、下記の入力制限・出力方式はServerSeedの候補であり、標準ライブラリが要求する製品仕様ではない。実行試験は未実施。

#### 字句と検証順序案

1. `args_os`から実行ファイル名を除いた列を受ける。上限は128引数、1引数4,096バイト、合計32,768バイト。計数はUbuntu上の引数の原バイト数、区切り・終端NULを除く。上限超過を先に検出する。
2. 全引数をUTF-8として検証する。非UTF-8はパスを含め`CLI_ENCODING`で拒否し、損失変換・置換文字で同一視しない。空文字は`CLI_VALUE`。この段階は対象OSの情報取得前の入力列の解析だけを行う。
3. 左から字句を分類する。長いオプションだけを受け、大小文字を区別する。値付きは`--name=value`と`--name value`を同一視し、`=`は最初の一つだけが区切り。後者の直後が欠損・`--`・`--`始まりなら値不足。boolへの値、短縮形、連結短縮形、未知オプションは拒否する。
4. オプションはコマンドの前・間・後で受理し、全体で同名を一回だけ許可する。同じ値・bool・`--help`も重複は拒否。単独の`--`はオプション終端で、以降はコマンド語だけとして照合する。既存操作に自由位置引数はなく、余分な語を拒否する。引用符・glob・環境変数・`~`を再解釈しない。
5. コマンド語列と下表の許可オプション・値域を照合する。引数順で最初の字句エラーを一件返し、続いてコマンド不正、操作に不許可のオプション、値域不正をこの順で判定する。同段階では元の引数位置順とする。
6. 構文と値域が妥当な`--help`は、必須オプションの欠損・組合せ検査より先に対応ヘルプを返す。設定・OS・実装有無を検査しない。`self --help`・`plan --help`・引数が`--help`だけの場合も許可するが、未知語・不許可オプション・値不足・値不正・重複をhelpで隠さない。
7. help以外は必須入力・相互条件を下表の順に検査する。その後だけ担当機能へ渡し、明示設定の検査、必要権限、対象・状態・前提、必須選択・確認の順で照合する。前段の停止後に変更・採取・保存へ進まない。未実装の担当機能は`CLI_UNIMPLEMENTED`で停止し、架空の観測・計画・成功を返さない。

構文検査はファイル・環境を読まない。後段の設定や権限を必要としないhelp・組込みversion情報を、その検査の失敗で隠さない。複数の後段失敗は[結果集約](#結果と終了区分案)で扱い、引数段階の失敗と混在させない。

#### 操作別の入力案

字句としては全操作で既存の[共通オプション](../BASIC_SPEC.md#共通オプション)と`--help`を認識し、操作ごとの許可・用途は下表と値域表へ照合する。下表は追加の操作別入力を列挙する。共通フラグを認識しただけで変更や保存を許可しない。

| 操作 | 操作別オプション | 必須性・関係の候補 |
|---|---|---|
| `inspect` | なし | 追加の対象・通信先を受けない |
| `setup` | なし | 構成・対象の選択を新規CLI引数へ独断で置換しない |
| `update` | `--source`・`--release-version` | 両方あり、または両方なし。片方だけは入力エラー |
| `recover` | `--self`・`--keyring` | keyringはselfと併用時だけ。初回中断・保存済み鍵なしで必要となる条件は既存recoverへ照合 |
| `verify` | なし | 検査範囲の新しい指定は追加しない |
| `report` | なし | 直近操作だけ。ID指定・検索オプションを追加しない |
| `status` | なし | 読み取りのみ |
| `version` | なし | 収録情報と適用済み情報を分離 |
| `doctor` | `--transaction` | 指定時は`--json`必須。操作ID・親ロック・権限の実検査は既存doctorの契約 |
| `plan` | 対象操作と同じ | 対象語が必須。下記の7通りだけ |
| `support-bundle` | なし | `--output`必須。dry-runでも入力を省略しない |
| `self install` | `--source`・`--release-version`・`--keyring` | 3項目すべて必須。sourceはローカルディレクトリ |
| `self update` | `--source`・`--release-version` | 2項目とも必須。鍵差し替えの入力は受けない |
| `self uninstall` | なし | 新しい取得元・鍵を指定しない |

`plan`の対象語列は`setup`、`update`、`recover`、`recover --self`、`self install`、`self update`、`self uninstall`。ここで`--self`は語ではなくrecover専用フラグ。`plan plan`や読み取りコマンドの計画は拒否する。`--dry-run`はplanと併記しても再帰的な計画を作らず、保存禁止を維持する。

| 値・共通フラグ | 解釈・候補の値域 |
|---|---|
| `--config` | UTF-8の絶対パス。未指定は既存の基本設定配置。明示先の欠損・非通常ファイル・読取り不能は入力エラー |
| `--log-level` | 既存4値のみ。CLI指定、基本設定、基本設定も未作成の場合の[既定値](../POLICIES.md#基本設定の既定値)の順。必要な設定検査をCLIで隠さない |
| `--output` | UTF-8の絶対パス。report・plan・support-bundleだけで保存先として解釈し、他操作の指定は`CLI_OPTION_SCOPE`。保存形式は下記 |
| `--source` | ローカルはUTF-8の絶対パス。HTTPSを許す操作では`https://`の基点URLも候補とし、通信先・URL全構文・取得制限の検証はP04-SELFの未達へ接続。http等へfallbackしない |
| `--release-version` | 先頭`v`なしのSemantic Versioning。正確な版の照合は[既存本体入力](../BASIC_SPEC.md#self)。文法は[SemVer 2.0.0](https://semver.org/spec/v2.0.0.html)の第2・9〜11項へ対応し、trim・大小文字変換をしない |
| `--keyring` | UTF-8の絶対パス。読取り・信頼の確認は担当処理。鍵の値をCLIへ受けない |
| `--transaction` | P02-MODELのOperationId表記だけ。文字列の一致を実行主体・排他の証明にしない |
| `--json`・`--no-color` | 出力表示だけを選ぶ。JSONは常に無色 |
| `--yes` | 読取りコマンドでは効果なし。変更時は既存の通常確認だけを省略し、選択・破壊的確認・入口・権限を省略しない |
| `--dry-run` | 読取りコマンドは同じ観測、support-bundleは採取計画だけ、変更操作はplan相当。ファイル・一時領域・ログの作成なし |

パスはNUL、U+0001〜U+001F、U+007Fを拒否し、相対パスの補完、symlink解決、存在検査を引数解析で行わない。後段の安全な解決・交換検出は`P03-FS`等の担当契約へ接続する。明示設定がなく構成未作成の既定ファイル欠損だけは`setup_required`として扱う。設定スキーマ・構成選択の入力を拡張しない。

### 出力案

共通JSONのトップレベル10項目は[既存の枠](../BASIC_SPEC.md#共通json出力)を維持する。以下の全集合はCLI側の形の候補であり、ネストする担当契約の未確定スキーマを確定済みとはしない。記載のないキー・型、省略・独自補完を許可しない。

#### 共通の型と必須項目案

| 型 | 全フィールド・必須性の候補 |
|---|---|
| Envelope | 10キーすべて必須。schema_versionはintegerの1、command・summaryはstring、result・stateは既存列挙、started_at・finished_atはP02-MODELの時刻string、dataはobject、warnings・errorsはDiagnosticのarray |
| Diagnostic | code・messageは必須string、fieldは必須のstringまたはnull。入力値・秘密値・外部出力をmessageへ連結しない。fieldは引数名またはJSON Pointer、全体エラーはnull |
| Check | idは必須の契約IDとローカル検査名の`契約ID/検査名`、requiredはboolean、statusはpass・fail・unknown・not_applicable、code・fieldはstringまたはnull、messageは秘匿化済みstring。全6キー必須。passはcodeがnull、それ以外の理由はcodeとmessageへ明示 |
| Observation | idはstring、statusはobserved・unknown・not_applicable、valueはstringまたはnull、reasonはstringまたはnull。全4キー必須。observedはvalueあり・reasonなし、他はvalueなし・reasonあり。値の空文字と未取得を区別 |
| OperationRef | operation_idはP02-MODELのstring、statusは既存の操作状態、commandは既存操作名string。全3キー必須。完全な操作記録の代わりではなく、読取った記録への参照 |
| RevisionRange | min・maxとも必須integer、[スキーマ登録](#入力と保存形式の詳細案)のrevision範囲内、min <= max。マニフェストが要求する各範囲をさらに照合 |
| Artifact | pathは絶対パスstring、formatはtext・json・tar.gz、statusはplanned・saved。全3キー必須。保存失敗・部分成果物をsavedにしない |

配列は要素0件と確認した場合もキーを省略せず空配列。取得不能・集合未確定を0件へ変換しない。任意とする値もキーは存在させnullを使う。Checkの集合・必須性・各Observationのidと取得元は担当検査の仕様へ対応付け、集合未確定を空配列の成功で埋めない。requiredかつnot_applicableは担当正本に明示した適用外の根拠がある場合だけ許可し、確認不能ならunknownとする。CheckはidのASCII順、同一idの重複は禁止。Observationもid順・重複禁止。診断は検証段階順、同段階はfieldのUTF-8順（nullが先）、次にcode順。同じcodeとfieldは最初の一件へ集約し、異なる失敗を消さない。

総合状態を判定できないときは既存`UNKNOWN`を用い、nullや新しい状態名に置換しない。構成・経路のフィールドへUNKNOWNを入れず、検証できないAppliedInfoは下記のnullと理由で扱う。未観測の非UTF-8パスはObservationをunknown、valueをnull、reasonを`CLI_PATH_ENCODING`とし、別パスへ損失変換しない。状態への影響は当該情報の必須性から判定する。

#### コマンド別data案

| コマンド・応答 | 必須キーと型の全集合 |
|---|---|
| `status` | checks: Check[]、pending_operation: OperationRefまたはnull、profile: 既存構成stringまたはnull、setup_required: boolean |
| `verify` | scope: stringのarray、checks: Check[]。scopeは担当契約IDを重複なしASCII順で示す |
| `inspect` | execution_environment: Observation[]、target: Observation[]、checks: Check[]。ローカルで観測した事実だけ。外部到達性の未確認も理由を持つ |
| `doctor` | installation_status: 既存5値、setup_required: boolean、checks: Check[]、transaction_id: OperationIdまたはnull。指定した診断のIDを他操作へ置換しない |
| `report` | operation: OperationRefまたはnull、record: 操作記録objectまたはnull、checks: Check[]、missing_records: string[]、artifact: Artifactまたはnull。recordの全形式はP02-STOREの未達。履歴・検査・復旧案の対応も同契約へ接続 |
| `version` | binary: 下記BinaryInfo、applied: 下記AppliedInfoまたはnull、observations: Observation[]、checks: Check[] |
| `plan` | requested_command: 対象操作名string、plan: 計画objectまたはnull、checks: Check[]、artifact: Artifactまたはnull。計画はP03-PLANと各操作の担当正本。nullを実行可能計画にしない |
| `setup`・`update`・`recover`・`self install`・`self update`・`self uninstall` | mode: planまたはexecute、plan: 計画objectまたはnull、operation: OperationRefまたはnull、checks: Check[]、requires_reboot: boolean。dry-runはmode=plan、operation=null。未確定の変更記録はP02-STOREとPhase 4へ接続 |
| `support-bundle` | collection_status: planned・complete・failed、items: 下記BundleItem[]、excluded: Diagnostic[]、artifact: Artifactまたはnull。収集・形式未成立をcompleteにしない |
| 妥当な`--help` | help_for: 正規化したコマンド名string、usage: string、commands: string[]、options: string[]。選択範囲に有効な候補だけをASCII順で表示。通常コマンドのdataとは排他的 |
| 担当dataを確定できない開始前エラー、未実装停止、整形エラーの代替応答 | error_stage: arguments・configuration・prerequisites・unimplemented・render、help_for: stringまたはnull。実行していない担当機能のdataを捏造しない。開始後の失敗は通常のdataで適用済み範囲を保持 |

`command`は通常は正規化した操作名、planは対象を含む`plan self update`等とする。helpは要求範囲名（rootは`serverseed`）。入力が不正なら認識できた最長の有効な範囲名を用い、全く認識不能なら`serverseed`。生の不正語を出力名にしない。recoverのself分岐はcommand=`recover`、planの対象名も`recover`で、分岐の事実は担当計画の契約で識別する。versionは構文が妥当なら後段の設定エラーでも通常dataに読めたbinaryを保持し、errorsと終了区分で失敗を示す。

BinaryInfoの全6キーは既存の`serverseed_version: string`、`architecture: string`、`config_revision: integer`、`migration_revision: integer`、`compatible_config_revision: RevisionRange`、`compatible_migration_revision: RevisionRange`。revisionはスキーマ登録の共通範囲、architectureは既存の本体CPU値。AppliedInfoは`serverseed_version: string`、`config_revision: integer`、`migration_revision: integer`、`environment: string`、`profile: string`の全5キーを候補とし、構成値・経路は基本設定へ照合する。構成未作成または適用情報が検証不能ならapplied=nullと理由を分け、欠損revisionを0で埋めない。OS・kernelの読める版はobservationsへ置く。自己申告を署名検証済みの証拠にしない。

BundleItemの全4キーは`name: string`、`status: planned|collected|excluded|failed`、`bytes: integerまたはnull`、`reason: stringまたはnull`。bytesは0以上i64::MAX以下、collectedだけ実採取バイト数を持つ。plannedは未実施、excluded・failedは理由必須。nameは採取項目の固定識別子で、外部パスや秘密値を格納しない。

担当計画・操作記録・検査ID一覧・観測項目一覧の全集合は未確定で、上表はその本文を再定義しない。未知キーを通す汎用payloadを実装者に決めさせず、`D02-02`と各担当の成立判定に残す。上記dataの形だけで全CLIの完全仕様成立としない。

#### 結果と終了区分案

既存の[数値と意味](../BASIC_SPEC.md#終了コード)を変更せず、結果分類を次の候補へ対応付ける。下表は上から優先し、最初に該当する区分を選ぶ。収集できた別の失敗はerrors・checksから消さない。

| 条件 | result・終了コード | 扱い |
|---|---|---|
| 引数・必要な設定・要求操作に必須のroot権限の不正 | failed・2 | 対象変更の開始なし。transaction診断も含む |
| 担当機能未実装 | blocked・3 | `CLI_UNIMPLEMENTED`。空の成功を返さない |
| 復旧が必要だが経路・保全根拠を確認不能 | blocked・6 | 単なる未観測とは区別。復旧済みとしない |
| 開始した本体導入・更新・削除、またはsetupが失敗 | failed・5 | 実適用位置と記録の照合を必要とする。復旧の成功で元失敗を消さない |
| 必須の検査がfail、導入状態inconsistent、記録の不整合 | failed・4 | verify・doctor等の不一致。既知のfailはunknownより優先 |
| 必須情報unknown、許可条件不成立、選択・確認なし、実行不能計画 | blocked・3 | 理由をerrorsへ。判定に任意のunknownだけならwarnings |
| 内部整形・保存・一般I/O等の失敗 | failed・1 | 他の区分に該当しない失敗。成功にしない |
| 処理成功かつ実施した変更後の再起動が必要 | success・10 | errors空。planで将来の再起動を予測しただけなら10にしない |
| 上記なし | success・0 | errors空が必須。警告だけなら成功可 |

`status`は対象状態そのものがADOPTABLE等であるだけではfailにしない。必要な根拠不足は3、矛盾・破損の確証は4。`verify`は必須failがあれば4、必須unknownだけなら3。任意fail・unknownはwarningsとして、必須集合を勝手に縮小しない。`inspect`は取得できない任意情報と外部到達性をwarningsへ置き、状態判定に必要な情報のunknownは3。不対応対象を対応済みとして返さない。

`doctor`の構成未作成・正常なremoved・absentだけでは失敗にしない。setup_requiredを表示し、必要な診断のunknownは3、不整合は4。installation_status自体を判定不能なら新しい値を追加せず、prerequisitesのエラー応答で3とする。transaction診断の権限不足は2、親・ロック等の証拠不足は3、配置との確定不一致は4。`report`は操作未実施を確認できた場合のoperation=nullは許可するが、必須記録を読めない場合は3、破損は4。`version`は実行本体を必ず表示し、未構成はapplied=nullでも0、構成済みだが適用情報不明なら3、不整合なら4とする案。

エラー時は少なくとも一つerrorsを持つ。引数エラーの固定コードは`CLI_LIMIT`・`CLI_ENCODING`・`CLI_VALUE`・`CLI_OPTION`・`CLI_DUPLICATE`・`CLI_COMMAND`・`CLI_OPTION_SCOPE`・`CLI_REQUIRED`・`CLI_COMBINATION`、設定は`CLI_CONFIG`、root不足は`CLI_PRIVILEGE`。前段の検証順に従い、他契約のエラーはその固定コードを参照する。診断文は固定の日本語、summaryは`command: result`のASCII表記とし、原入力を転記しない。

CLI_VALUEは空値・値不足・boolへの値・値域不正、CLI_OPTIONは未知または短縮オプション、CLI_COMMANDは未知・不足のコマンド語または余分な位置引数へ割り当てる。その他は同名重複、操作への不許可、必須オプション欠損、組合せ不正へそれぞれ割り当てる。固定messageは順にLIMIT「引数上限を超えています」、ENCODING「UTF-8以外の引数は使用できません」、VALUE「引数の値が不正です」、OPTION「未対応のオプションです」、DUPLICATE「オプションが重複しています」、COMMAND「コマンドの指定が不正です」、OPTION_SCOPE「この操作では指定できません」、REQUIRED「必須オプションが不足しています」、COMBINATION「オプションの組合せが不正です」、CONFIG「設定を検証できません」、PRIVILEGE「必要な権限がありません」、UNIMPLEMENTED「対象機能は未実装です」、RENDER「応答を整形できません」、OUTPUT「出力を完了できません」、PATH_ENCODING「パスをUTF-8で表現できません」。接頭辞はすべて`CLI_`。固定stderrは`serverseed: CLI_OUTPUT\n`、代替応答自体を生成不能な場合は`serverseed: CLI_RENDER\n`とし、外部エラー文を付加しない。

helpのresultはsuccess・0、stateは未観測のUNKNOWN、warnings・errorsは空。`--json`とhelpは上記help専用dataを用い、必須入力の不足だけならエラーにしない。引数解析失敗時も、値として消費されていない独立した`--json`がオプション終端より前にある場合はJSON、それ以外は人間向け表示。上限超過時は入力上限内の範囲だけで表示モードを判定し、超過部分を追加走査しない。エラー後の表示モード判定だけの走査はI/O・追加実行を伴わず、`--config=--json`や`-- --json`をJSON要求にしない。

#### JSONと標準出力案

出力文法は[RFC 8259](https://www.rfc-editor.org/rfc/rfc8259)へ照合する。[Value](#解析の型とapi案)からUTF-8の一オブジェクトと末尾LFだけを返し、BOM・余分な空白・ANSI色・進捗・二つ目のJSON文書をstdoutへ混ぜない。全objectキーはUTF-8バイト辞書順、arrayは上記の契約順で、現在時刻等を除く同じ値は同じバイト列とする。署名対象の正規化形式としては採用しない。

文字列では引用符・逆斜線を`\"`・`\\`、U+0000〜U+001Fを小文字16進の`\u00xx`でescapeし、他のUnicode scalarはUTF-8のまま出す。`/`はescapeしない。stringを再解釈・Unicode正規化しない。integerは十進・先頭0なし、0は`0`。NumberはP02-PARSEの検証済み字句を保持し、手作りのValueでも同じ数値文法を再検証する。floatへの変換・丸め、NaN・Infinity、不正な数値字句の素通しをしない。

整形後上限は末尾LF込み1,048,576バイト、Valueの深さ・ノード・文字列上限は[P02-PARSE](#解析の入力と上限)と同じ候補を使用し、整形前に全体を検証する。超過・不正な応答は`CLI_RENDER`の固定診断へ置換し、stdoutに元データを一部だけ出して成功にしない。この小さな代替応答の生成も不能ならstdoutなし、固定stderr、終了1。応答全体の実割当・実使用量の保証は`D02-02`・`D02-03`の実証待ち。

人間向け表示は同じCommandResponseから、1行目summary、2行目`state: STATE`、以後dataを同じキー順のJSON表記で示し、続いてwarning・errorの順に`LEVEL CODE FIELD: MESSAGE`の一件一行とする。field=nullは`-`、改行等は上記escapeを用いる。errorsに入力不足がある場合は対応する`serverseed ... --help`の案内を末尾へ付ける。helpだけはusageとcommands・optionsの順で一項目一行。色は任意で意味を持たせず、`--no-color`時と非TTY時は付けない。

整形はI/Oなしで完了後、stdoutの単一writerで`write_all`、`flush`を行う。write_all内の短いwrite・Interruptedの処理と、文書全体の再送は区別する。両方の完了前に出力送達成功と報告しない。書込みが完了不能・BrokenPipe・flush失敗なら`CLI_OUTPUT`、プロセス終了1と固定stderr。既に送ったstdoutを取り消せるとせず、文書全体の再送・二つ目のエラーJSONの追記をしない。表示失敗の終了1は応答内の終了分類より優先するが、対象側の適用済み位置・確定済み操作記録を巻き戻さず、変更操作も再実行しない。stderrも失敗した場合は追加の出力を試みず終了1とする。標準の同期I/Oだけで停止しない出力先へ硬い期限を保証する方式は未確定で、実行上限の成立を主張しない。

emitの成功はwriterが要求したbyte列を受け取りflushに成功した事実に限定する候補。[Write](https://doc.rust-lang.org/std/io/trait.Write.html)の契約だけで、pipeの相手がJSONを解析したこと、terminalで利用者が閲覧したこと、通常fileが永続媒体へ同期されたことは認定しない。成果物保存は別の同期/確定契約、SSHの受付はP09の受理記録を必要とする。空writeが継続するwriter・無期限blockingの成立性はD02-02へ残し、threadを分けただけで期限を保証したとしない。

#### CLIの型とAPI案

以下は実装予定の宣言であって、実装コードではない。P01-CLIの関数を具体化する。型の値域は上記表を唯一の定義とし、生成時に検証して内部フィールドを呼出元から変更させない。

```rust
pub enum Action { Inspect, Setup, Update, Recover, Verify, Report,
    Status, Version, Doctor, SupportBundle, SelfInstall, SelfUpdate, SelfUninstall }
pub enum OutputMode { Human, Json }
pub struct CommonOptions {
    json: bool, dry_run: bool, yes: bool, no_color: bool,
    config: Option<std::path::PathBuf>, log_level: Option<String>,
    output: Option<std::path::PathBuf>,
}
pub struct OperationOptions {
    source: Option<String>, release_version: Option<String>,
    keyring: Option<std::path::PathBuf>, transaction: Option<String>, self_only: bool,
}
pub enum Request {
    Run { action: Action, plan: bool },
    Master(MasterInvocation),
    Help { scope: String },
}
pub struct Invocation { request: Request, common: CommonOptions, operation: OperationOptions }
pub struct CliInputError {
    diagnostic: ContractError, argument_index: Option<usize>,
    scope: String, output_mode: OutputMode,
}
pub enum ExitClass { Success, General, Input, Blocked, Verification, Installation, Recovery, Reboot }
pub struct CommandResponse { envelope: Envelope, exit: ExitClass }
pub fn parse_args(args: &[std::ffi::OsString]) -> Result<Invocation, CliInputError>;
pub fn dispatch(input: &Invocation) -> Result<CommandResponse, ContractError>;
pub fn validate_response(response: &CommandResponse) -> Result<(), ContractError>;
pub fn render(response: &CommandResponse, json: bool) -> Result<Vec<u8>, ContractError>;
pub fn emit(bytes: &[u8], writer: &mut dyn std::io::Write) -> Result<(), ContractError>;
```

Invocationは入力を所有し、source等の不要なOptionはNone、self_onlyはrecover以外でfalse。indexは実行名を除いた0始まり。Runのplanは`plan`要求でtrue、dry-runの実効modeもplanとする。helpでは指定値を保持しても使わない。パーサー・render・validate_responseはファイル・通信・ログ・時計に触れず、emitだけが指定writerへ書く。dispatchは時刻・観測を取得し担当契約へ委譲、helpでは管理対象を読まない。時刻取得の実現方式はP02-MODELの未達に接続する。

CommandResponseの生成・検証は、上表のdata形と担当契約の承認済みスキーマを両方照合してから許可する。入出力境界のValueはvalidate_documentへ渡し、検証済みEnvelopeへ変換した後だけCommandResponseを構築する。矛盾するresult・ExitClass、非空errorsの成功、未定義payloadを拒否する。ExitClassは[結果表](#結果と終了区分案)の対応で数値化し、型名を製品状態に転用しない。dispatchの内部エラーも固定診断の応答に変換してrenderへ渡し、生の内部エラーを表示しない。開始後の失敗は担当の通常dataと部分結果を保持し、開始前専用Errorへ置換しない。

CommandResponseの構築候補では、担当dataの検証→Checkの全必須集合照合→診断の並替え/重複集約→結果表の適用→Envelope検証→整形、の順を固定する。required=falseのfailをerrorsへ残しながらsuccessにする矛盾は作らず、任意失敗はwarningsだけ、必須失敗はerrorsへ対応付ける。OperationReceiptが確定していてもstdout失敗時のプロセス終了は1となり、親操作の保存結果とCLI送達結果を別項目として報告する。代替エラーJSONの追記・変更再実行・保存済み成功の取消は行わない。

内製の難易度候補は字句・診断整形が低、型検証・出力保全が中。stdにはServerSeed固有の文法・スキーマ・集約器がないため、この責務だけを内製する。将来のcrate抽出・差し替えでは同じ入力バイト列・応答・失敗writerを新旧へ与え、stdout・stderr・終了値・副作用を比較する。CLI・設定・保存形式の変更と旧コード除去は別の提示・承認が必要。今回は新設案だけで、新旧比較対象・切替・除去はない。

### 調査成果物案

圧縮境界の推奨候補は、秘匿化済みの固定member集合を圧縮adapterへ渡す方式。member名は固定採取IDに拡張子を付けた単一相対名だけで、重複・絶対path・区切り・親逸脱・link・deviceを拒否する。生成前に全memberのbytes/hashと採取結果を固定し、生成後の展開検査でmember集合・内容の完全一致と余分なentry不在を要求する。これは復号ではなく非暗号アーカイブの展開照合。暗号化・任意archive抽出は対象外。内製圧縮形式案は相互運用検証の負担が増えるため第二候補とし、OS実行物案も個別採用前は実行しない。HTTPSの取得契約はP04-SELFだけで定義する。

bundle検証の候補では、生成tar.gz自体を17 MiB以下、展開時の通常file合計を採取上限以下へ制限し、各memberの宣言量と実量を生成前の集合へ一致させる。展開は保護された検証用領域だけで行い、入力名を任意pathへ解決しない。圧縮物の切詰め・破損・余分なgzipストリーム・未登録の拡張entry・展開量超過はBUNDLE_ARCHIVEで正式保存しない。採取・生成・検証・確定の全工程を既存の30秒へ算入する。正確なtar/gzip版・全argv・形式の相互運用契約はD02-06で採用審査し、上限の追記だけで実行可能とはしない。

`support-bundle`の`--output`は既存の保存先ディレクトリを指定する候補を維持し、その直下の[既存ファイル名形式](../BASIC_SPEC.md#support-bundle)へ一つのtar.gzを保存する。保存先ディレクトリ自体を新規成果物と取り違えず、生成するアーカイブのパスが既存なら停止する。ファイル名の識別子はP02-MODELのMachineId、時刻はUTCの`YYYYMMDDTHHMMSSmmmZ`。MachineId・時刻が確認不能なら代用値を作らず停止する。

採取上限は100ファイル、合計16 MiB、1ファイル1 MiB、全工程30秒を候補とする。直近履歴は最大100行とし、不完全な末尾行を黙って成功扱いにしない。秘匿化の成立性を確認できない項目は[秘密情報](../SECURITY_POLICY.md#秘密情報)へ照合して除外し、excludedへ理由を出す。上限超過・必須採取の失敗はcollection_status=failed、errorsあり、正式成果物をsavedにしない。除外だけを理由に非公開内容を診断へ転記しない。

dry-runはcollection_status=planned、itemsもplannedまたはexcluded、artifactは予定値だけ。保存・一時ファイル・子コマンドの再帰呼出しをせず、同じ読取り用データ取得処理から計画を作る。通常の採取も各コマンドのstdoutを連結せず、担当処理の構造化結果を収録する。tar.gzの形式・検証・必須項目集合・部分成果物の安全な後始末は`D02-06`の未達で、外部tarを使用可能とせず、完成したbundleを生成できるとも扱わない。

report・planのoutputは新規の単一ファイルパスを指定する候補とし、通常のstdoutと同じ表現を保存する。JSONは同じ応答から作り、保存物自身のartifactはplanned、保存後のstdoutだけsavedとする。保存物が自分の保存成功を証明する循環を作らない。report・planも既存ファイルの上書き・親ディレクトリ作成なし。dry-run時はArtifactをplannedとして表示するだけ。排他的な新規作成、同期、部分保存と中断時の扱いはP02-STORE・P03-FSの未達へ接続し、CLI案だけで成立済みとしない。

### CLIの受入ケース案

いずれも設計であり、製品テストは未実施。時刻・対象観測・担当payloadは承認済みfixtureを必要とし、未確定の集合をテスト内で独自に確定しない。純粋な解析・整形、失敗writer、隔離FSでのI/O、実対象での担当試験を区別する。

| ケースID | 区分・入力 | 期待結果・確認方法 |
|---|---|---|
| `A02-41` | 正常。14操作とplanの7分岐 | 表の各構文を解析し、正規化名・action・planと必須入力を照合。未実装は3で変更なし |
| `A02-42` | 正常。共通オプションを前・間・後、値を空白/等号形式 | 同じInvocation。値内の2個目の等号は保持 |
| `A02-43` | 異常。短縮形、未知語・オプション、boolへの値、余分な位置引数 | 最初の該当段階で入力2。設定・OS読取り0回 |
| `A02-44` | 異常。同値の同名オプション・helpの重複 | CLI_DUPLICATE。先勝ち・後勝ちなし |
| `A02-45` | 正常・異常。self install --help、未知オプション併記、値不足 | 必須欠損だけならhelp成功。未知・値不足は2、設定を読まない |
| `A02-46` | 境界。128/129引数、4,096/4,097バイト、合計32,768/32,769 | 各上限内を次段階へ、超過はCLI_LIMIT。Unicodeは原バイトで計数 |
| `A02-47` | 異常。非UTF-8パス、空引数、制御文字を含むパス | CLI_ENCODINGまたはCLI_VALUE。置換文字・shell展開・I/Oなし |
| `A02-48` | 正常・異常。単独--、その後の--json、値中の--json | オプション終端・値を誤認しない。`-- --json`は入力2で人間向け |
| `A02-49` | 異常。updateの片側指定、selfの必須欠損、keyring誤用 | CLI_COMBINATIONまたはCLI_REQUIRED。取得・変更なし |
| `A02-50` | 異常。plan plan、plan status、不許可output、transactionのみ | 入力2。doctor transactionはjsonが必須 |
| `A02-51` | 正常・異常。明示config欠損と未構成の既定config欠損 | 前者2、後者setup_required。勝手な設定生成なし |
| `A02-52` | 正常・異常。CLI/config/既定ログ値、不正configとCLI併記 | 優先順を照合し、不正設定をCLI指定で隠さない |
| `A02-53` | 正常。helpとjson、構文エラーと独立json | 各専用data・正規化command。1JSON+LF、未知語の生値なし |
| `A02-54` | 正常・異常。各操作のdata、欠損・未知キー・不正型 | 表と担当スキーマを照合。不明payloadを空objectで通さない |
| `A02-55` | 正常・異常。未構成versionと読取り不能applied | binaryを保持。未構成だけは0、不明3、不整合4 |
| `A02-56` | 正常・異常。doctor absent/removed/pending/inconsistent | 未構成と破損を分離。transactionの証拠・権限を別検査 |
| `A02-57` | 異常。必須failとunknown、任意unknown、記録破損 | 集約表どおり、既知failをunknownで隠さない。errorsありで0なし |
| `A02-58` | 正常・異常。planの再起動予測、変更成功の再起動、元操作失敗 | 予測だけは0、成功後必要なら10、元失敗を10へ上書きしない |
| `A02-59` | 正常。診断の順序入替・同じcode/fieldの重複 | 検証段階・field・code順、重複だけを一件化 |
| `A02-60` | 正常・異常。引用符・逆斜線・制御文字・日本語・非UTF-8観測パス | 指定escapeとUTF-8。観測不能はnullと理由、別パスへ変換なし |
| `A02-61` | 正常・異常。1.0・1e400・不正Number・integer端 | 数値字句保持、丸め・NaNなし。不正応答はCLI_RENDER |
| `A02-62` | 境界。整形後1,048,576/1,048,577バイト、深さ・ノード上限 | LF込み上限判定。超過をstdoutへ部分表示しない |
| `A02-63` | 正常。同じ応答を人間向け/JSON・無色で整形 | 同じ事実・診断と所定の順序。時計・FS・通信へ触れない |
| `A02-64` | 異常。短いwrite、Interrupted、BrokenPipe、flush失敗 | write_all/flushの結果を照合。失敗は1、全体再送・二つ目のJSONなし |
| `A02-65` | 異常。変更記録確定後にstdout失敗、stderrも失敗 | 適用済み記録を保持。再適用・無承認巻戻し・無限出力なし |
| `A02-66` | 正常・異常。bundle保存先ディレクトリと既存アーカイブ | directoryを既存成果物と誤認せず、同名archiveは上書きせず停止 |
| `A02-67` | 境界。bundleの件数・個別量・総量・30秒、秘密項目 | 各上限と除外理由を照合。必須失敗をcompleteにしない |
| `A02-68` | 正常・異常。全操作のdry-run・yesと、許可/不許可操作へのoutput | 許可範囲だけ計画し、変更・成果物・ログの保存0。不許可outputは2。必須選択をyesで代行しない |
| `A02-69` | 異常。report/plan保存失敗・親欠損・競合・部分保存 | savedを偽装せず、管理状態を変更しない。担当保存契約未成立なら保留 |
| `A02-70` | 異常。未定義検査集合・時刻取得失敗・出力先の無期限停止 | 成功・実行上限を保証しない。未達を担当判断へ追跡し、実証なしで閉じない |

## P02-PARSE

### 解析の範囲と一次資料

担当要件は`REQ-004`・`REQ-007`・`REQ-008`・`REQ-011`・`REQ-012`。配置案は`src/core/format/`とし、字句・構文・値への変換を内製する。標準機能と再利用対象の確認は[調査表](PHASE_01.md#標準機能の充足調査)を参照する。既存内製パーサー・旧実装はなく、crateの新設・抽出・差し替えは行わない。

照合日は2026-10-20。JSONは[RFC 8259の第2〜9節](https://www.rfc-editor.org/rfc/rfc8259)、YAMLは[YAML 1.2.2の第5〜8章・第10章](https://yaml.org/spec/1.2.2/)へ照合する。本節の数値上限、限定構文、型解決、エラーはServerSeedの採用候補であり、外部標準がその値や制限を要求するとの主張ではない。YAML全構文・標準スキーマとの完全互換は提供範囲に含めない。

解析は借用した1文書のバイト列から所有する値を返す。形式の自動推定、別形式へのフォールバック、ファイル読書き、通信、外部コマンド、環境変数の展開、設定の適用、ログ保存は行わない。呼出元がJSONかYAMLかを選ぶ。同時呼出しは入力・予算・途中状態を共有せず、失敗後の部分値を返さない。署名検証に用いる原バイト列を解析・再生成した値で置き換えない。

### 解析の型とAPI案

次は実装予定の宣言であり、実装コードではない。値型は`src/core/model.rs`、解析型・関数は`src/core/format/mod.rs`、形式固有の処理は同ディレクトリの`json.rs`・`yaml.rs`へ置く候補とする。依存方向は各パーサーから共通値型へ向け、構成別モジュールを参照しない。

```rust
pub(crate) enum Value {
    Null,
    Bool(bool),
    Integer(i64),
    Number(String),
    String(String),
    Array(Vec<Value>),
    Object(std::collections::BTreeMap<String, Value>),
}

pub(crate) enum ParseErrorCode {
    InputLimit, Utf8, Bom, Empty, Syntax, Trailing,
    Escape, Unicode, DuplicateKey, IntegerRange,
    Unsupported, Indent, MissingValue, YamlKeyLimit,
    DepthLimit, ElementLimit, StringLimit, MemoryLimit,
    Timeout, Clock,
}

pub(crate) struct ParseError {
    pub(crate) code: ParseErrorCode,
    pub(crate) byte_offset: usize,
    pub(crate) line: usize,
    pub(crate) column: usize,
}

pub(crate) fn parse_json(input: &[u8]) -> Result<Value, ParseError>;
pub(crate) fn parse_yaml(input: &[u8]) -> Result<Value, ParseError>;
pub(crate) fn parse_acceptance_json(input: &[u8]) -> Result<Value, ParseError>;
```

入力の寿命に返却値を結び付けず、文字列・数値の字面・配列・objectを返却値が所有する。parse成功の`Value`には本節の構文・上限を適用するが、手作成・変更された値を自動的に検証済みとはしない。objectのキーは復号後のUTF-8バイト列で比較し、Unicode正規化・大小文字変換をしない。ソース上の順序をobjectの意味へ含めず、配列の順序は保存する。

### 解析の入力と上限

既定値として緩和できない固定候補を次に定める。MiBは1,048,576 bytes、KiBは1,024 bytesとする。設定・CLIから解析上限を変更する入口は追加しない。

| 対象 | 算定と境界案 |
|---|---|
| 入力 | 借用slice全体が1 MiB以下。空白・改行・コメント・escapeの元表現も含む。超過は構文解析前に拒否 |
| 文字コード | UTF-8だけ。先頭BOMを拒否し、不正UTF-8を置換文字へ修復しない。空白だけ・YAMLのコメントだけは文書なしとして拒否 |
| 深さ | root値の深さ1、配列要素・objectの値は親+1。最大32。キーは深さを増やさない。空collectionも1段として数える |
| 要素数 | rootを含む全Valueと、objectの各キーをそれぞれ1件として合計10,000以下。空collectionは1件。コメント・空白・区切りは数えない |
| 文字列 | 復号したUTF-8の長さが1個64 KiB以下。キーも対象。元のescapeの長さは入力上限へ、復号後の長さは本上限へ算入 |
| 数値字面 | Numberへ保持する字面は入力の該当範囲そのもの。単独の追加上限は設けず、入力・下記予算の双方で制限 |
| YAMLキー | 区切りcolonを除くキーのソース表現が1,024 Unicode scalar values以下。引用符・escapeの各文字も数え、ASCIIの単純キーにも適用 |
| 時間 | 関数入口から最終結果確定前までの経過が1秒未満。1秒以上を検出した呼出しは成功を返さない。I/Oは含まない |
| 計算上のメモリー予算 | 下記Mが32 MiB以下。実ヒープ・RSSの上限保証とは区別する |

計算上の予算は`M = I + S + R + 128*N + 1024*D + 8192` bytesとする。Iは入力slice長、Sは生成した全キー・文字列の復号後長の累計、Rは保持する数値字面長の累計、Nは上表の要素数、Dは現在までの最大深さ。いずれも削除・巻戻しで予算を戻さず、checked arithmeticの失敗もMemoryLimitとする。ノード・文字列の生成前に予定値を検査し、超過分を構築してから捨てる方式にしない。

Mは型やallocatorに左右されない論理計数であり、BTreeMapの内部割当、Vec/Stringの余剰capacity、allocator管理領域、実行stack、Rust実行基盤を実測した値ではない。実使用量の32 MiB保証に読み替えず、実ヒープ・stackの計測方法、実割当失敗時の保全と対象環境での検証は`D02-03`の未達へ残す。通常の入力上限が先に働く場合に、メモリー超過入力が存在したと捏造しない。予算計算の境界は計数器単体の受入で照合する。

時間計測には[Instant](https://doc.rust-lang.org/std/time/struct.Instant.html)を用いる案とする。入口、標準機能による全文UTF-8確認の前後、各トークンの開始前、内製処理で最大4,096入力bytesを走査するごと、値の返却前に期限を検査する。内製の文字列比較等も4,096 bytes以下の区間で検査する。経過時間の算定不能・逆転検出はClock。成功直前の検査も省略せず、タイムアウト後の自動再試行は0回とする。標準機能の一括処理内やOSによる休止・割当待ちを強制中断する方式ではなく、1秒で必ず制御が戻るリアルタイム保証はしない。

parse_acceptance_jsonだけは[P11の受入カタログ/期待集合](PHASE_11.md#受入カタログの閉包と更新)専用の固定profileを使う候補。入力16 MiB以下、ノード2,000,000以下、同じM式で512 MiB以下、成功直前経過15秒未満へ置換し、構文/深さ/文字列/位置/checked演算/検査間隔/割当失敗規約は通常JSONと同じ。YAMLや一般設定・配送・manifestへこのprofileを使わない。呼出元指定の任意上限値/緩和optionを設けず、用途を検証するP11だけから呼ぶ。数値は論理予算でありRSS/強制返却期限を保証しない。

### JSONの受理契約案

JSONではobject・array・string・number・小文字のtrue/false/nullを1値だけ受け付ける。値の前後と構造記号間の空白はASCII space・HT・LF・CRのみ。末尾の空白後に内容が残ればTrailingとする。コメント、末尾comma、単引用文字列、引用のないキー、欠損値を受理しない。

文字列は二重引用とし、rawのU+0000〜U+001Fを拒否する。escapeは引用符、backslash、slash、b・f・n・r・t、uと4桁hexの組だけを受け付ける。hexの大小は問わない。high surrogateは直後のu escapeのlow surrogateと対になる場合だけ1 scalarへ復号し、単独・順序逆転・範囲不正はUnicodeとする。escapeの不明種別・桁不足・非hexはEscape。raw UTF-8とescapeの同じ復号結果は同一のキーとして照合する。

numberは全体が`-?(0|[1-9][0-9]*)(\.[0-9]+)?([eE][+-]?[0-9]+)?`に一致する字面だけを受け付ける。この記法は文法の説明で、外部の正規表現エンジンを導入する意味ではない。先頭plus・不要な先頭zero・小数点や指数の桁欠損・NaN・InfinityはSyntax。小数部も指数部もなければi64の範囲へ検査してIntegerとし、範囲外はIntegerRange。`-0`はInteger(0)とする。小数部または指数部があればNumberに元の字面を保持し、f64への変換・丸め・指数展開をしない。例えば`1e400`の構文受理は、下流で有限の計算値として使用できるとの認定ではない。

数値の終了は空白・comma・閉じ括弧・EOFに限り、妥当な数値の直後にそれ以外のbyteが続く場合も数値のSyntaxとする。`01`をInteger(0)と余分なrootへ分離せず、offset=1で拒否する。数値以外のrootが完成した後に別内容がある場合はTrailingを使用する。

### YAMLの構造契約案

YAMLの限定形式は次の全条件を満たす1文書とする。ここで行とは元入力のLFまたはCRLFで区切る行であり、最終行の改行なしも認める。単独CR、rawのHT、rawの制御文字を拒否する。通常のraw文字はU+0020〜U+007E、U+00A0〜U+D7FF、U+E000〜U+FFFD、U+10000〜U+10FFFFとし、改行は上記の区切りだけを使う。

1. rootの有効行は字下げ0。block mapping・block sequence、1行scalar、空collectionを扱う。空白行・コメントだけの行は構造を変えない。
2. blockの子は親の基準位置からspace2個だけ深くし、兄弟は同じ位置へ揃える。深さ飛越し、奇数の字下げ、既存の親位置へ一致しないdedent、scalar後の子行はIndentとする。
3. mappingは1行のstringキーとcolonで始める。colonの直前にspaceを置かず、直後は1個以上のspaceまたは行末。キーは`[A-Za-z_][A-Za-z0-9_-]*`のplain、または下記の引用文字列とし、空の引用キーは許容する。複合キー・explicit keyは対象外。
4. sequenceは`-`とspace1個の後に値を書くか、`-`だけの行の次へ子を置く。inlineのmapping開始`- key: value`も認め、そのmapping基準位置はsequenceの基準+2とする。後続のキーはその位置、キーの子はさらに+2。inlineでさらにsequenceを始める省略形は認めない。
5. colon・単独の`-`の後に値がなければ、次の有効行に上記位置の子を必須とする。子がない場合はMissingValueとし、暗黙のnullや空文字へ変換しない。nullは明示する。
6. flowの例外は空配列`[]`と空object`{}`の完全一致だけとする。非空flow、括弧内のspace・コメントはUnsupported。配列・objectを表す必要がある場合はblockを使う。
7. 引用外の`#`は行頭または直前がspaceの場合だけ行末コメントを開始する。値とコメントの間の末尾spaceを除き、scalar内容の内部spaceを保存する。`abc#def`内の`#`と引用内の`#`は内容。
8. directive、文書区切り`---`・`...`、タグ、anchor・alias、block scalar、引用scalarの改行継続、merge keyを受理しない。構文として現れた場合はUnsupported。引用文字列中の同じ記号は内容であり、objectキーの復号結果`<<`だけは引用の有無を問わず拒否する。plain scalarの複数行折畳はせず、その後の子行は手順2のIndentとする。

同じblock位置にmappingとsequenceの項目を混在させた場合はSyntax、完成したroot scalarの後の別の有効行はTrailingとする。quoteが閉じないまま改行へ達した場合はその改行位置でUnsupported、EOFへ達した場合はEOFでSyntaxとする。

### YAMLのscalarと型案

| 表記 | 復号・型・拒否条件 |
|---|---|
| single quote | 同じ行の対応するquoteで閉じ、内部のquote2個を1個へ復号。backslashはそのまま保持 |
| double quote | 同じ行で閉じる。JSONの単純escapeとu4桁、およびU8桁hexを認める。u/Uの値は単独でUnicode scalarの範囲にあることを要求し、surrogate値を拒否。YAMLではJSON型のsurrogate対を結合しない。その他のYAML 1.2.2で有効なescapeはUnsupported、不明なescape種別・hex不正・桁不足はEscape |
| plainの範囲 | 空でなく1行。先頭は`! & * [ ] { } # , : ?`、quote、backtick、percent、at-sign、block scalar記号を禁止。先頭hyphenは次がspaceまたは行末ならsequence扱い。内部の`[ ] { } ,`とquote・backtickを禁止。colonは直後が非spaceの内容文字のときだけ許容し、URLの`http://`を保持。引用が必要な値を自動補正しない |
| 明示型 | 引用しない完全一致の`true`・`false`はBool、`null`はNull。JSONと同じnumber文法は上記のInteger/Number規則へ接続 |
| その他のplain | String。`yes`・`no`・`True`・`~`・日付・`01`・`+1`・`0x10`・`.nan`は暗黙変換しない。引用した数値・true/nullもString |

scalarに関係する構文の優先順は、引用、空collection、禁止された構文、明示型、その他plainとする。quoteの閉じ後はspaceとコメントまたは行末だけを認め、別の値の連結を拒否する。キーは常にStringとし、値用の暗黙型解決を適用しない。復号したキーの重複は、1個目と同じ値でもDuplicateKeyとする。

### 解析のエラーと位置案

ParseErrorは1個だけ返す。入力上限、全文UTF-8、先頭BOM、YAMLの全文文字・改行条件、左からの構文・値構築、末尾の順で確認する。前段の失敗が後段より優先し、後段のエラーを得るために解析を続けない。各確認点で期限を先に照合し、同時に検出する制限はMemoryLimit、DepthLimit、ElementLimit、StringLimit、YamlKeyLimitの順とする。文法から分類が一意なものは下表の専用分類を使い、それ以外だけSyntaxとする。

| codeの値 | ParseErrorCode・検出対象・位置 |
|---|---|
| `parse.input_limit` | InputLimit。入力上限を初めて超えるbyteの位置 |
| `parse.utf8` | Utf8。[from_utf8](https://doc.rust-lang.org/std/str/fn.from_utf8.html)のvalid_up_toに対応する位置 |
| `parse.bom` | Bom。先頭の位置0 |
| `parse.empty` | Empty。有効値なし。EOF |
| `parse.syntax` | Syntax。要求された構文に合わない最初のbyte。必要なbyteなしならEOF |
| `parse.trailing` | Trailing。root後に残る最初の非空白内容。YAMLの許可されたコメントは除く |
| `parse.escape`・`parse.unicode` | Escape/Unicode。問題のescapeのbackslash。rawの不正UTF-8は前段のUtf8 |
| `parse.duplicate_key` | DuplicateKey。2個目のキーの開始位置。quote込みのソース開始 |
| `parse.integer_range` | IntegerRange。符号を含む数値の開始位置 |
| `parse.unsupported` | Unsupported。対象外構文の開始位置。merge keyならキーの開始位置 |
| `parse.indent` | Indent。異常な有効行の最初の非spaceの位置 |
| `parse.missing_value` | MissingValue。子を持たないcolonまたは単独hyphenの位置 |
| `parse.yaml_key_limit` | YamlKeyLimit。長過ぎるキーの開始位置 |
| `parse.depth_limit`・`parse.element_limit` | DepthLimit/ElementLimit。超過する値またはキーの開始位置 |
| `parse.string_limit` | StringLimit。超過するstringの開始位置。quoteを含む |
| `parse.memory_limit` | MemoryLimit。予定計数が超過する値・キー・文字列の開始位置。入口の算定なら位置0 |
| `parse.timeout`・`parse.clock` | Timeout/Clock。検査時点の走査位置。走査前は0、返却前はEOF |

byte_offsetは原入力の先頭を0とするusizeで、EOFは入力長。line・columnは1始まりとし、columnはその行の先頭からのbyte数+1である。LFで次行へ進み、CRLFを1改行として扱う。JSONの単独CRも1改行とし、YAMLの不正な単独CRではその位置を指す。不正UTF-8の位置も有効UTF-8への変換後の文字数で数えない。位置を求めるために別の予算なし全文コピーを作らない。

呼出元のContractErrorへの変換ではcodeをそのまま固定文字列に対応付け、messageは固定の分類名と数値の位置だけ、fieldはNoneとする。キー名・値・入力断片・ファイル内容・パスをエラーへ含めない。CLIのトップレベル枠・終了区分の選択は[P02-CLI](#p02-cli)と操作別契約が担当し、パーサーが終了コードを返したりprocessを終了させたりしない。

### スキーマ検証との境界案

解析は未知キーを消去せず、null・空文字・型・配列順序をそのまま返す。必須項目、未知キー、設定版、許容値・既定値・互換性は担当スキーマが後段で検証し、型の不一致をstringからinteger/boolへ再変換しない。構文として妥当な値を、既存の設定・マニフェストがすべて受理するとは扱わない。

スキーマ検証APIは下記の型付きvalidate_document候補だけを使用する。形式名・版・全キー・検証順序と各移行の採用はD02-03に残し、構文解析の詳細化だけでスキーマ充足とは判定しない。JSONの出力byteとescapeは[P02-CLI](#p02-cli)を参照する。

スキーマ選択の候補APIは`validate_document(kind: SchemaKind, value: &Value) -> Result<ValidatedDocument, ContractError>`に一本化し、文字列IDの汎用validateを並行実装しない。SchemaKindは本書の登録表の各形式、ValidatedDocumentはその対応する検証済み型を一つだけ保持するenum。呼出元が保存先/用途からkindを選び、入力の自己申告だけで別形式へ切り替えない。root型の不一致はSCHEMA_TYPE、宣言する版が未対応ならSCHEMA_VERSION。JSON Pointerではキーの~を~0、/を~1へescapeし、配列indexは0始まり十進、rootは空文字とする。キーの内容をmessageへ転記せず、fieldだけに検証位置を持つ。全形式の型宣言・移行表の残る不足はD02-03に対応付ける。

### 解析の内製化と差し替え案

標準機能で再利用する範囲はUTF-8確認、文字・整数の基本操作、集合、所有する文字列と経過時間。字句・構文・上限計数・位置・エラーの評価は新規内製で、外部crateや外部パーサー実行物を必要とする案にはしない。難易度は限定解析でも中、Unicode・時間・実割当を含む敵対入力の保全検証は高として評価し、未実証を低難易度へ移さない。

将来の抽出・差し替えは本節の宣言・Valueの意味・エラーと位置・受理集合を共通契約とし、[解析の受入ケース](#解析の受入ケース案)を新旧へ同じ入力で適用する。受理集合や型の変更、保存済み設定の再解析、共通CLIへの変換が変わる場合は互換変更と決めつけず、移行と復旧可能範囲を別途審査する。初回のため旧実装・旧crate・除去対象はなく、新旧比較結果も存在しない。

### 解析の受入ケース案

表内の`\n`・`\r`・`\t`は、特に「字面」と記した箇所以外ではテスト入力のLF・CR・HTを表す。JSON/YAMLのescapeを示すbackslashは原入力の文字とする。反復表記は指定した文字列を正確な回数だけ連結する入力生成規則であり、外部fixtureのコピーではない。全ケースは文書上の設計で、内製パーサーの実行結果ではない。

| ケースID | 前提・入力案 | 期待結果・合否の照合 |
|---|---|---|
| `A02-11` | JSON `{"b":[true,null],"a":-0}` | objectにa=Integer(0)、b=Array(Bool(true),Null)。入力と外部状態は不変 |
| `A02-12` | JSON `null`、`[]`、`{}`を別呼出し | Null、空Array、空Objectを区別。rootの型をobjectへ補完しない |
| `A02-13` | JSON `{}`の後にspaceを入力上限ちょうどまで追加。同じ入力へspace1 byte追加 | 前者成功、後者InputLimit・offset=入力上限。コメントや空白も数える |
| `A02-14` | JSONで`[`を31個、`0`、`]`を31個。括弧を各32個に変更 | 前者の最大深さは32、後者はDepthLimit。閉じ括弧を追加して回避不可 |
| `A02-15` | JSON配列にnullを9,999個、次に10,000個。別入力で一意なstringキーとnull値を4,999組・5,000組持つobject | 配列のroot込み10,000件とobjectの9,999件は成功。それぞれ超過側はElementLimit |
| `A02-16` | JSON引用内にASCII aを文字列上限ちょうど・+1 bytes | 前者成功、後者StringLimit。キーにも同じ上限を適用 |
| `A02-17` | JSON引用内にu0061 escapeを文字列上限の回数だけ反復 | 復号後の長さで受理し、入力長との二重の計数を照合 |
| `A02-18` | JSONのi64最小・最大値と、その範囲を1超える正負の値 | 範囲内はInteger、範囲外はIntegerRange・数値開始位置。丸めなし |
| `A02-19` | JSON `1.00`、`1e400`、`-0.0` | Numberへ各字面を保持。f64化やIntegerへの縮約なし |
| `A02-20` | JSON `01`、`+1`、`1.`、`1e`、`NaN`、`Infinity` | いずれもSyntax。順にoffset=1・0・2・2・0・0を照合 |
| `A02-21` | JSONのquote内にuD834・uDD1Eの連続escape、単独uD834、単独uDD1E | 対だけ1 scalarへ復号。単独はUnicode・backslash位置 |
| `A02-22` | JSON `{"a":0,"\u0061":1}` | DuplicateKey・offset=7、line=1、column=8。値一致でも同じ失敗 |
| `A02-23` | JSON `"é"`の後にspaceとx。別入力でNFC/NFDの異なるキー | 前者Trailing・offset=5、column=6。後者の異なるUTF-8列を自動統合しない |
| `A02-24` | BOM付きJSON、byte FF、空入力、spaceだけ | Bom・Utf8・Emptyを区別。いずれも部分値・外部出力なし |
| `A02-25` | 不正構文の後に不正UTF-8。別入力で入力上限超過とBOMを併存 | 前者Utf8、後者InputLimit。前段検証の優先順を照合 |
| `A02-26` | JSON `[0,]`、`/*x*/null`、`null null`、raw LFを含む引用文字列、qの未知escapeを含む引用文字列 | 順にSyntax(offset=3)、Syntax(0)、Trailing(5)、Syntax(raw LF位置)、Escape(backslash位置) |
| `A02-27` | YAML `apps:\n  - name: example\n    enabled: true\n`。同じ内容をCRLF化 | 同じObject/Array/String/Bool。inline mappingの継続位置が一致 |
| `A02-28` | YAML `apps: []`、`mapping: {}`。別入力で`[ ]`・`[1]`・`{a: 1}` | 空collectionだけ受理。他はUnsupportedでblockへ黙って変換しない |
| `A02-29` | YAML `a: yes\nb: "true"\nc: null\nd: 01\ne: 1.5\nf: 2026-10-20` | a/b/d/fはString、cはNull、eはNumber。日付・8進・boolの推測なし |
| `A02-30` | YAML `a: http://127.0.0.1/x#part # note\nb: 'it''s'` | URL内のcolon・hashを保持し末尾コメントを除外。bはString("it's") |
| `A02-31` | YAML double quoteのu0041、U0001D11E、uD834、qの各escape。single quote内のbackslash | 順にA・U+1D11Eへ復号、Unicode、Escape。single quoteのbackslashは不変 |
| `A02-32` | YAMLのodd indent・子の深さ飛越し・raw HT・単独CR・scalar後の子 | 順にIndent・Indent・Syntax・Syntax・Indent。raw HT/CRは全文文字検査で先に拒否 |
| `A02-33` | YAML `a:\nb: 1`、単独hyphen、空文書、コメントだけ | 前2件はMissingValue、後2件はEmpty。省略値をnullへ変換しない |
| `A02-34` | YAML tag・anchor・alias・merge key・directive・文書区切り・block scalar | Unsupported。引用内の同記号は内容として保持し、キー`<<`だけは拒否 |
| `A02-35` | YAML `a: 0\n"a": 1`。quoted keyのソース長1,024・1,025 scalar values | 重複はDuplicateKey・offset=5、line=2、column=1。長さ超過はYamlKeyLimit |
| `A02-36` | 空string・null・未知キー・型の違う値を構文解析 | 値を保持して成功。担当スキーマの拒否・既定値検証は別で、適用開始なし |
| `A02-37` | 計数器のMが予算ちょうど・+1、算定の整数overflowを単体で与える | ちょうどを許容、超過/overflowはMemoryLimit。実RSS保証の試験とはしない |
| `A02-38` | 差し替え可能な試験時計で、返却前の経過を1秒未満・ちょうど・超過にする | 未満だけ成功、残りはTimeout。実時間のsleepで境界を曖昧にしない |
| `A02-39` | 時計の逆転・計測不能、同時2呼出し、エラーに秘密値を含む入力 | Clock・予算独立を照合。エラーは分類と位置だけで入力断片を含めない |
| `A02-40` | 現存する基本仕様とサーバ構成のYAML形式例、基本仕様のJSON形式例 | 解析値の型・配列・文字列を期待木と照合。通常パーサーの成功を内製試験結果にしない |

境界ケースの計数器・試験時計は、将来のunit testで内部へ注入する試験条件であり、製品のCLI・設定で上限や時刻を改変する入口ではない。位置と返却値の一致、部分値の不在、外部状態の不変を全ケースの合否へ含める。実割当・時間制約の対象OS・両CPUでの実証と、担当スキーマによる統合試験は未実施とする。

スキーマ検証の候補順序は、root型→形式名/版→未知キー（UTF-8順）→必須キー欠損（同順）→各値の型/範囲→相互条件→参照先の版/互換性。最初の一件を返し、未知キーはSCHEMA_UNKNOWN、欠損はSCHEMA_REQUIRED、型はSCHEMA_TYPE、範囲はSCHEMA_RANGE、相互条件はSCHEMA_CONDITION、未知版はSCHEMA_VERSIONとする。fieldはJSON Pointer、messageへ原値を含めない。推奨は形式ごとのvalidatorで検証済み型を返す方式。汎用Valueのまま適用器へ渡す案は採らない。実ヒープ/stackの上限保証は、この形式検査とは別の未達である。

## P02-STATE

候補APIは`detect_state(target: &TargetEvidence, records: &ValidatedRecords) -> Result<StateDecision, ContractError>`とし、[既存の判定順](../BASIC_SPEC.md#状態判定の優先順位)を変えない。ValidatedRecordsは本書の各記録validatorを通過した値と、個別記録のabsent/unknown/invalidを区別する集合。invalidを記録なしへ変換せず、採取した実状態と合わせて最初に該当するruleを返す。状態スナップショットを入力の事実より優先しない。

対象総合状態と読取りコマンドの成否は別判定とする。前回operation=runningなら基本仕様どおりRECOVERABLEだが、記録・対象を正常に観測できたinspect/statusは実行中の事実だけで終了失敗にしない。新しい変更はoperation.clear不成立で停止する。各ruleの根拠・不成立・未取得をCheckへ保持し、呼出元がRECOVERABLEをMANAGEDへ補正する経路を設けない。

初回の構成選択結果は、利用者の対話結果、対象証拠、計画IDに結合する値としてメモリー上で保持する案。設定ファイルだけで選択成立とはしない。変更開始直前に再照合し、保存は最終構成検証後にP02-STOREへ渡す。保存された構成の値と三構成の適用範囲は[各正本](../POLICIES.md#構成選択と継続管理)へ参照する。

構成リビジョンの旧二択形式移行、初回選択結果を媒体・SSHへ渡す形式は`D02-03`と各入口の契約へ残す。既存の版番号を変更・互換認定しない。

### 状態入力の閉じた型

記録ごとの採取結果を以下へ閉じる候補。Presentだけが担当validatorの正常型を保持する。Invalidは取得したが形式不正、Unknownは取得/検証不能、Absentは不在の確証であり、互換でない。状態判定器が不正入力の記録を削除・空集合化しない。

```rust
pub enum RecordObservation<T> {
    Present(T), Absent { evidence_ref: String },
    Unknown { code: String, evidence_ref: Option<String> },
    Invalid { diagnostic: Diagnostic, evidence_ref: String },
}
pub struct StorageFacts {
    partitions_absent: bool, os_absent: bool, bootloader_absent: bool,
    verification_ref: String,
}
pub struct ValidatedRecords {
    target: TargetEvidence, storage: RecordObservation<StorageFacts>,
    config: RecordObservation<ConfigDocument>, policy: RecordObservation<PolicyDocument>,
    managed: RecordObservation<ManagedRecord>, machine_id: RecordObservation<MachineId>,
    install: RecordObservation<InstallRecord>, operation: RecordObservation<OperationRecord>,
    history: RecordObservation<Vec<HistoryRecord>>, version: RecordObservation<VersionRecord>,
    last_success: RecordObservation<LastResultRecord>, last_failure: RecordObservation<LastResultRecord>,
    transaction: RecordObservation<SelfTransaction>, snapshot: RecordObservation<StateSnapshot>,
    checks: Vec<Check>,
}
pub struct StateDecision {
    state: String, rule: u8, profile: Option<Profile>, setup_required: bool,
    evidence: TargetEvidence, checks: Vec<Check>,
}
```

ValidatedRecords.targetはdetect_stateのtargetと同じ採取世代へ照合する。checksには実配置本体/起動設定/選択機能の整合確認を保持し、未取得をpassへ補完しない。EMPTYの判定にはstorageの三不在と管理記録不在の確証を必要とする。OS非対応/取得不能の優先、構成未作成の許容、破損/前回未完了の扱いは基本仕様の七規則だけを適用する。キャッシュ単独で不足を補わない。A02-01/72にAbsent/Unknown/Invalidの独立入力、空diskだがbootloaderあり、別targetの記録集合を含める。

## P02-STORE

### 排他と確定案

変更の親プロセスが既存の共通ロックを一度取得し、子処理は借用する`ExecutionContext`を使う案。`File::try_lock`を候補とし、競合時は待ち時間0で停止する。ロックファイルをunlink・renameして所有者をすり替えない。読み取り専用ではファイルもロックも作成しない。

プロセス識別にはPIDだけでなくOS起動識別子とプロセス開始情報を対応付ける案。子への引き継ぎ方式・子だけ残った場合のロック寿命は`D02-04`で確定するまで実装しない。

管理ファイルの置換は[原子性](../BASIC_SPEC.md#原子性と書き込み)に従い、親ディレクトリの同期と書き込み先の同一性確認も候補に含める。履歴は操作IDごとの重複を照合し、部分的な追記を成功記録としない。追記型履歴と複数ファイル確定の具体方式は`D02-05`で審査し、通常のrenameだけでトランザクション成立を保証しない。

### 記録とAPI案

完了時の必須集合候補はoperation・同一操作のhistory行・結果に対応するlast-successまたはlast-failure。構成の版を実際に変更した場合だけversion/managed/configの該当更新を追加し、self単独で適用済みrevisionを書換えない。stateは再生成可能な補助記録で、生成を計画した場合はその保存成功も必要とする。失敗操作でlast-successを削除せず、逆も同じ。履歴確定後の補助記録不足は元の結果を作り直さず、hashと実状態を照合する復旧計画へ渡す。推奨は全候補の事前直列化と既存履歴を保つ置換方式である。

`version`は既存の製品バージョン情報のJSONオブジェクト、`last-success`・`last-failure`は操作ID・終了時刻・結果・対象IDを持つJSONオブジェクトとする候補。現在状態の置換と製品の追記型実行履歴を分ける。管理権限・秘密情報は[セキュリティポリシー](../SECURITY_POLICY.md#ファイル権限)へ参照し、新たな権限規定を本書で作らない。

| 宣言案 | 制約 |
|---|---|
| `acquire(target: &TargetEvidence) -> Result<OperationGuard, ContractError>` | guardはClone不可、対象・所有プロセスと結合。取得失敗後の変更なし |
| `begin<'g>(guard: &'g OperationGuard, plan: &Plan, start: OperationStart) -> Result<ExecutionContext<'g>, BeginFailure>` | 事前生成済みIDと対象・期限・前提を再照合し、intent→running→beginイベントの同期後だけ返す。contextはguardより長生きできない |
| `finish(ctx: &mut ExecutionContext, result: &OperationCompletion) -> ChangeResult<RecordReceipt>` | 必須記録の確定に失敗したら確認済み段階のRecordReceiptをpartialへ保持。失敗記録も書けなければstderrと残存証拠を使い復旧待ち |

OperationCompletion候補の全項目はstatus（success/failed/interrupted）、finished_at（UtcMillis）、steps（StepReceipt[]）、required_records（固定保存先IDの重複なし集合）。finishは全stepと計画の一対一対応・statusの妥当性→必要記録集合→事前直列化→履歴→終了記録→補助記録の順で照合する。successは全必須stepのverifiedが必要。同じoperationの完結行が既存ならbyte完全一致だけ再追加なしで照合し、一文字でも違えばSTORE_CONFLICTとして元行を保持する。履歴は論理追記のまま、改行未完結の末尾を切除して続行しない。

required_recordsは呼出元が成功条件を減らす入力ではなく、親Planの操作/変更層と上記必須集合から保存器が再算定する集合の照合値とする候補。過不足・重複をSTORE_REQUIREDで拒否する。interruptedは失敗側の結果記録へ結合するが、failedへ偽装せず元statusを保持する。finish前の実適用済みstepを消して再提出することはできず、失敗時のpartialも同じPlanの全stepに対応させる。

ログの採用案は、操作ID・対象ID・時刻・level・イベントID・秘匿化済みメッセージをJSONLへ記録する方式。保持・ローテーションは10 MiB×5世代を候補とするが、必須操作記録をログ整理で消去しない。履歴容量の保全・退避と操作上限の関係は`D02-05`の未決事項。

## P02-EXEC

通常実行器の型宣言候補を次へ閉じる。専用stream/file-sink/二入力pipeのadapterは本capture型と別で、通常の量制限を解除しない。

```rust
pub enum CommandMaterial { External(MaterialRef), Release(ReleaseArtifactRef) }
pub struct CommandSpec {
    material: CommandMaterial, executable: std::path::PathBuf,
    working_directory: std::path::PathBuf, args: Vec<std::ffi::OsString>,
    environment: std::collections::BTreeMap<std::ffi::OsString, std::ffi::OsString>,
    stdin: Vec<u8>, timeout_ms: u32, grace_ms: u32,
}
pub enum CommandContext<'a> {
    ReadOnly { target: &'a TargetEvidence, deadline: std::time::Instant },
    Change { guard: &'a OperationGuard, plan_id: &'a PlanId,
        operation_id: &'a OperationId, deadline: std::time::Instant, cancelled: &'a bool },
}
pub struct CommandOutcome {
    status: String, exit_code: Option<i32>, signal: Option<i32>,
    stdout_bytes: u64, stderr_bytes: u64, elapsed_ms: u64, stopped: bool,
}
pub struct CommandOutput { outcome: CommandOutcome, stdout: Vec<u8>, stderr: Vec<u8> }
```

Change文脈は有効なExecutionContextを借用する内部構築器からだけ作る候補。guardより短い借用で各ID/期限/取消を無変更で参照し、JSONから直接構築しない。ReadOnlyからChangeへ昇格する変換を設けない。CommandOutcome.statusの固定値と数値/null条件は本書の結果契約を適用する。A02-93/98へ文脈のID差替えとstatus/exit_code矛盾を含め、起動前拒否と起動試行後の観測結果を区別する。

配置案は`src/platform/command.rs`。`Command`の引数配列と`Instant`を再利用し、呼び出しごとに操作契約の制限値を必須入力とする。外部実行物の版・パス・引数・例外IDの照合なしに起動しない。

| 入力 | 採用案の型・範囲 |
|---|---|
| 実行物・引数 | 絶対パス、`Vec<OsString>`。シェル文字列を受けない。環境変数は必要な固定集合だけ |
| 時間 | timeout_msは1〜3,600,000、終了待ちgrace_msは1〜30,000。未指定・0・範囲外を拒否 |
| 出力 | stdout・stderr各1 MiBまで保持。並行して読み、超過時は失敗へ移る。末尾だけ捨てて成功にしない |
| 再試行 | 共通実行器は自動再試行0回。呼出元だけが回数・待機・再照合を定義できる |
| 取消 | 起動前は起動しない。起動後は対象契約の保全・停止へ移り、単に接続を閉じたことを終了とみなさない |

候補APIは`run_command(spec: &CommandSpec, context: CommandContext<'_>) -> Result<CommandOutput, ContractError>`へ一本化する。CommandSpecの全項目はmaterial（CommandMaterial）、executable・working_directory（PathBuf）、args（Vec<OsString>）、environment（BTreeMap<OsString, OsString>）、stdin（Vec<u8>）、timeout_ms/grace_ms（u32）。CommandMaterialはExternal(MaterialRef)またはRelease(ReleaseArtifactRef)のenum候補で、個別例外実行物と検証済み同一本体の起動を区別する。値域は本節の表とstdin契約へ照合する。CommandContextはReadOnly（対象証拠と期限を借用）またはChange（ExecutionContextを借用）のenumとし、前者は個別の非変更実行契約だけを許可する。読取りのためにguardや管理記録を新規作成しない。

起動前検証の候補順序は資材/実行物の固定対応→引数テンプレート→作業directory→環境集合→stdin量→期限/実行文脈。working_directoryは当該実行契約が指定する既存の絶対directoryだけで、CLI起動場所を継承しない。環境はenv_clear後に宣言集合だけ設定し、PATHによる実行物探索・HOMEのユーザー設定・proxy・loader変数を暗黙継承しない。実行物/引数/環境名/値のNUL、環境名の空と=を拒否する。引数の許可集合・個数・量は担当実行契約へ完全一致させ、CLIの引数上限を別用途の共通実行器へ無言で転用しない。不足はEXEC_SPECでspawn前に返す。

CommandOutputの全項目はoutcome（CommandOutcome）、stdout/stderr（Vec<u8>）。起動前の検証失敗だけErrを返し、起動試行後のspawn_failed・timeout・出力超過・停止不明もOk内の観測結果として保持する。Okをコマンド成功とは解釈せず、終了0・必要出力・停止確認を担当処理が別に照合する。入力1 MiB、出力各1 MiBの上限は解除しない。子孫停止・ブロッキングI/Oの中断と親停止時の保全はD02-04に残し、Child::kill・新しい外部ツール・FFIで補完済みとはしない。

CommandOutcomeの候補優先順は、起動不成立ならspawn_failed、起動後に全停止を確認不能ならstop_unknown、それ以外は最初の取消/timeout/出力超過/I/O失敗、最後に通常終了またはsignalとする。同時検出ではcancelled→timed_out→output_limit→io_failedの順とし、追加原因は担当Checkへ保持する。停止要求後にexit_code=0を得ても元原因をexitedへ戻さない。stdout/stderrの生byte列は一時メモリーの観測値で、公開診断・ログ・bundleへそのまま転記しない。秘匿化は担当セキュリティポリシーを適用する。

期限算定の候補起点はrun_command受付時。timeout_msは通常処理と停止/回収を含む全予算、grace_msはその内側の停止予約とし、timeout_ms<=grace_msはEXEC_SPEC。外側の期限と受付時+timeoutの早い方を最終期限、そこからgraceをchecked減算した時点を通常処理の期限にする。起動前に通常処理の残量がなければspawnしない。通常期限以後は新しいstdin配送/処理を開始せず停止・回収へ移り、停止のため親期限を延長しない。elapsed_msは受付から最後に観測した位置までの経過を切り捨てた整数msで、応答不能中の終了値を生成しない。OSが処理を戻さない場合の外部観測/保全はD02-04の成立条件に残す。

## データ契約の具体化案

### 対象証拠と状態結果

`P02-MODEL`のTargetEvidenceを次のメモリー上の値へ具体化する候補。OS固有の証拠取得自体は`D02-01`・`D03-02`で調査する。

| 型 | 全キー・値域の候補 |
|---|---|
| EvidenceField | status: observed・absent・unknown、value: stringまたはnull、reason: 固定コードstringまたはnull。observedはvalueのみ、absent/unknownはreasonのみ |
| TargetEvidence | execution_id・disk_id・root_id・os_id・os_version・boot_idはEvidenceField、observed_atは時刻string。各値を同じ採取世代へ対応付ける |
| StateDecision | state: 既存総合状態、rule: integerの1〜7、profile: 既存構成またはnull、setup_required: boolean、evidence: TargetEvidence、checks: P02-CLIのCheck[] |
| Selection | profile: 既存構成、plan_id: PlanId、target: TargetEvidence、selected_at: 時刻、origin: media・master。選択画面が生成し、設定読込みだけでは生成しない |

証拠のabsentは取得に成功して不在を確認した場合だけで、OS情報が未取得の場合はunknown。EvidenceFieldのreason候補は`MODEL_NOT_PRESENT`・`MODEL_UNREADABLE`・`MODEL_UNSUPPORTED`・`MODEL_CHANGED`。StateDecision.ruleは[既存判定順](../BASIC_SPEC.md#状態判定の優先順位)の最初の該当番号とし、stateと一致しない組を拒否する。MANAGEDのprofile=nullや、未選択値をmaster/serverへ補完する結果を生成しない。Selectionは権限証明でも永続的な承認トークンでもなく、再検査で対象・計画が変われば破棄する。

TargetEvidenceの全フィールドがobservedであることを状態判定の一律前提にしない。空ディスクではrootやOSの不在をabsentとして証明する必要があり、unknownと区別したうえで既存判定順の各条件へ照合する。観測世代の一致だけで未取得の値を補完せず、disk_id等の取得・必須性が未確定なら対応する判断IDを残す。

TargetEvidenceの比較では、元の証拠を保存する完全比較と、観測時刻を除いた対象同一性比較を分ける候補。後者の比較対象は担当契約が定める必須証拠集合であり、呼出元が欠損項目を任意除外しない。observedは値の完全一致、unknownは一致未確認、absentはその工程が不在を許す場合だけ不在事実として照合する。observed_atの相違だけで別対象にせず、逆に同時刻だけで同一対象にしない。boot/実行環境/rootの予定変化は媒体/OS導入の引渡し契約で別に証明し、この比較の対象集合を減らして通さない。

識別子のparseは長さ・接頭辞・小文字hexを完全一致で確認し、空白除去や大文字変換をしない。生成側は候補と照合できる既存IDの集合を受け、衝突時は`MODEL_ID_COLLISION`として停止する。無限に乱数を引き直さず、乱数源の成立性・照合範囲が不足する限り`D02-01`は未解消。時刻の候補表現は`YYYY-MM-DDTHH:MM:SS.mmmZ`、年1970〜9999、有効な暦日、秒0〜59。過剰な小数桁を受理して丸めず、生成時だけナノ秒をミリ秒へ切り捨てる。経過時間はInstantで別に評価する。

### 記録確定と実行結果

`P02-STORE`には、記録候補を事前にすべて直列化・検査してから、同じ親操作の下で確定する方式を提案する。準備失敗は対象変更前に停止する。実資源の検証後に履歴を確定し、その操作IDを参照する終了記録を最後に確定する。履歴だけ確定・終了記録未確定の中断は、実適用状態と履歴の一致を復旧側が確認するまで成功としない。

追記型history.jsonlの論理的な追記を、既存バイト列と新しい完結行を含む一時ファイルの同期・renameで実現する候補とする。既存行の改変・並替え・欠落は禁止。同じ操作IDの同じ確定行は再追加せず、異なる結果なら競合として停止する。これで複数ファイル全体の原子性が成立したとはしない。全履歴のコピー容量、保持・退避、親ディレクトリ同期、クラッシュ時の照合手順は`D02-05`に残す。

| 型 | 全キー・値域の候補 |
|---|---|
| RecordReceipt | operation_id: OperationId、stage: prepared・history_committed・complete、record_paths: string[]、verified: boolean。completeと全必須保存の成功時だけverified=true |
| CommandOutcome | status: exited・spawn_failed・signaled・timed_out・cancelled・output_limit・io_failed・stop_unknown、exit_code: integerまたはnull、signal: integerまたはnull、stdout_bytes・stderr_bytes・elapsed_ms: 0以上integer、stopped: boolean |

```rust
pub enum RecordStage { Prepared, HistoryCommitted, Complete }
pub struct RecordReceipt {
    operation_id: OperationId, stage: RecordStage, record_paths: Vec<String>, verified: bool,
}
pub struct OperationCompletion {
    status: OperationStatus, finished_at: UtcMillis,
    steps: Vec<StepReceipt>, required_records: Vec<String>,
}
pub struct AppendBinding {
    path_id: String, record_id: String, offset: u64, bytes: u64, sha256: String,
}
pub struct RecordIntent {
    schema_version: u32, operation_id: OperationId, plan_id: PlanId, target: TargetEvidence,
    required_paths: Vec<String>, before_hashes: std::collections::BTreeMap<String, Option<String>>,
    planned_after_hashes: std::collections::BTreeMap<String, Option<String>>,
    observed_after_hashes: std::collections::BTreeMap<String, Option<String>>,
    append_bindings: Vec<AppendBinding>, complete: bool,
}
```

OperationCompletion.statusにRunningを渡したらSTORE_REQUIREDで拒否する。record_pathsは実在/同期を照合した固定保存先IDの一意ASCII順集合であり、任意pathや「予定しただけ」の保存先を含めない。RecordStage::Completeとverified=trueはintent.complete/全必須snapshot/担当記録の確認後だけ許可する。hash mapのnullとキー不在は別で、未確認値をnullへ変換しない。AppendBindingのoffset+bytesはchecked演算し、bytes=0と範囲外を拒否する。

| ケースID | 固定入力 | 期待結果 |
|---|---|---|
| `A02-120` | Completion.status=running、予定pathだけのcomplete receipt、offset+bytes overflow | STORE_REQUIRED/STORE_RECONCILE。未終了・未保存の成功化、整数wrapによる行照合なし |

CommandOutcomeはプロセスの観測結果で、操作全体の成功ではない。exit_codeはexited時のみ、signalはsignaled時のみ数値を持つ。出力量は保持できた量と総観測量を混同せず、本候補のbytesは上限検出までの観測量。上限を1 byte超えた時点で失敗し、以降の無制限収集は行わない。標準入力も1 MiB以下の固定byte列を候補とし、空入力は明示的なEOF、書込み失敗後の再配送はしない。起動・入力配送・出力回収・終了確認をすべて同じ期限へ算入する。

2026-10-22に採用版の[SystemTime](https://doc.rust-lang.org/std/time/struct.SystemTime.html)、[File](https://doc.rust-lang.org/std/fs/struct.File.html)、[CommandExt](https://doc.rust-lang.org/std/os/unix/process/trait.CommandExt.html)を再照合した。壁時計の非単調性、明示同期、プロセス群設定と実際の子孫終了確認を区別する。プロセス群を作るだけで子孫の停止・親停止時の排他維持を証明せず、`D02-04`は未達のまま。追加データのRust完全宣言・生成関数・OS取得実証も各判断IDの採用対象である。

## 接続と責務の照合

CLIのCheckは検査実施時の構造で、[状態キャッシュ](../BASIC_SPEC.md#状態スナップショット)のchecks[].resultとは別の既存契約である。CLIのrequired・not_applicableをキャッシュへそのまま書かず、保存用のpass・warn・fail・unknownへの対応が確定するまで`D02-02`・`D02-05`を未達にする。読取りコマンドは変換・診断をメモリー上だけで行う。

TargetEvidence.disk_id/root_idは[P03-FS](PHASE_03.md#p03-fs)の実対象固定と照合する。OS・ブートIDは観測時点の識別で、SSHのホスト鍵やMachineIdと単独で同一視しない。media/masterというSelection.originは入口の区分であり、config.ymlのusb/cloudを変更する理由にならない。保存するenvironmentは基本仕様の導入経路を使う。

保存記録の終了状態と[P04-UPDATE](PHASE_04.md#データ契約の具体化案)のStepReceiptを照合し、`history_committed`だけを操作完了にしない。RecordReceiptは実際に確認できた保存先と段階だけを返し、復旧が必要な中断記録を通常のfinish呼出しで上書きしない。必須履歴の容量・同期・原子置換が成立しない場合、変更開始を保留する。

外部実行のtimeoutは実行開始からの親予算に制限され、出力・終了待ちにも残予算を割り当てる候補。停止確認不能はstopped=false、status=stop_unknownとし、ロックを解放しただけで新変更を許可しない。stdだけによる停止・同期I/Oの期限保証の未達を外部ツールの暗黙採用で埋めない。

## 受入ケース案

| ケースID | 区分・入力 | 期待結果と確認方法 |
|---|---|---|
| `A02-01` | 正常・異常。整合対象、対応外OS、読み取り不能 | 基本仕様の優先順と根拠を照合。キャッシュのみでMANAGEDとしない |
| `A02-02` | 異常。未選択、対話不能、--yes、構成不一致 | 選択・保存を代行せず停止。ファイル差分なし |
| `A02-03` | 境界。入力1 MiB・1 MiB+1、深さ32・33 | 境界内を構文検査、超過を拒否。設定適用0回 |
| `A02-04` | 異常。重複キー、型不正、未知キー、YAML alias | 誤った値の自動補完・コード評価なし。位置と分類を返す |
| `A02-05` | 異常。重複オプション、無効config、CLIログ値 | 入力エラー。CLI値による不正設定の隠蔽なし |
| `A02-06` | 正常。読み取り専用で中断記録を検出 | メモリー上の診断だけ。管理ディレクトリ・ログの不変 |
| `A02-07` | 異常。別プロセスのロック、PID再利用 | 新変更なし。PID一致だけで所有者・完了を認定しない |
| `A02-08` | 異常。記録途中の容量不足・同期失敗 | 成功を返さず、実適用位置の照合待ち。無承認巻戻しなし |
| `A02-09` | 境界。時間・出力量の上限と直後の超過 | 所定の失敗分類。子孫終了と記録確定を別に照合 |
| `A02-10` | 異常。秘密項目混在、既存保存先、dry-run | 秘匿化不能項目を採取せず、上書き・保存の迂回なし |

## 追加の受入ケース案

`P02-MODEL`・`P02-STATE`・`P02-STORE`・`P02-EXEC`の追加設計。OS取得器・記録器の障害は隔離fixtureで注入し、未確定の取得・停止方式を実装済みとしない。

| ケースID | 区分・前提と入力 | 期待結果・照合対象 |
|---|---|---|
| `A02-71` | 境界。規定桁のplan/op ID、末尾を大文字化、空白追加、生成時衝突 | 規定表記だけ受理。衝突はMODEL_ID_COLLISIONで停止、再生成ループなし |
| `A02-72` | 異常。同じdevice pathでdisk_id交換、OS情報unknown、偽のMANAGEDキャッシュ | MODEL_CHANGEDと取得不能を保持。キャッシュで対象・対応OSの不足を補完せず、基本仕様のUNKNOWN判定へ照合 |
| `A02-73` | 境界。時刻の小数3桁/4桁、秒59/60、時計後退・再起動 | 候補形式外を拒否。期限不明を有効としない。壁時計と経過時間の比較を分離 |
| `A02-74` | 異常。履歴確定直後・終了記録前で中断、同じ操作IDに異なる結果 | completeを返さず復旧照合。同じIDの履歴を重複追加・上書きしない |
| `A02-75` | 境界。stdin/stdout/stderrの上限+1 byte、子だけ残存、終了確認不能 | 超過を成功にせず、stop_unknown/stopped=falseを保持。再配送・次の変更開始をしない |

## 型とAPIの詳細案

`P02-MODEL`の共有宣言候補。Valueからの構築は担当validatorだけに限定し、整数のRust型に入ることと表の狭い許容範囲を満たすことを区別する。時刻・ID・対象の証拠は別型で保持する。

```rust
pub struct PlanId(String);
pub struct OperationId(String);
pub struct MachineId(String);
pub struct UtcMillis(String);
pub enum Profile { Master, Mining, Server }
pub enum EntryOrigin { Media, Master }
pub enum IdKind { Plan, Operation, Machine }
pub fn parse_plan_id(input: &str) -> Result<PlanId, ContractError>;
pub fn parse_operation_id(input: &str) -> Result<OperationId, ContractError>;
pub fn parse_machine_id(input: &str) -> Result<MachineId, ContractError>;
pub enum MaintenanceAction { Setup, Update, Backup, Recover, SelfInstall, SelfUpdate, SelfUninstall }
pub enum EvidenceField {
    Observed(String), Absent { reason: String }, Unknown { reason: String },
}
pub struct TargetEvidence {
    execution_id: EvidenceField, disk_id: EvidenceField, root_id: EvidenceField,
    os_id: EvidenceField, os_version: EvidenceField, boot_id: EvidenceField,
    observed_at: UtcMillis,
}
pub struct ContractRef {
    file: String, anchor: String, contract_id: String,
    draft: String, commit: String, adopted: bool,
}
pub struct ContractError { code: String, message: String, field: Option<String> }
pub struct Selection {
    profile: Profile, plan_id: PlanId, target: TargetEvidence,
    selected_at: UtcMillis, origin: EntryOrigin,
}
pub struct ChangeFailure<T> {
    error: ContractError, partial: Option<T>, persisted_evidence: Vec<String>,
}
pub type ChangeResult<T> = Result<T, ChangeFailure<T>>;
pub struct OperationGuard {
    file: std::fs::File, target: TargetEvidence,
    boot_id: String, pid: u32, process_start_ticks: u64,
}
pub struct ExecutionContext<'g> {
    guard: &'g OperationGuard, plan_id: PlanId, operation_id: OperationId,
    deadline: std::time::Instant, cancelled: bool,
}
pub struct OperationStart {
    operation_id: OperationId, plan_id: PlanId, target: TargetEvidence, started_at: UtcMillis,
    deadline: std::time::Instant,
}
pub enum BeginStage { Validated, IntentSynced, RunningSynced, EventSynced, Unknown }
pub struct BeginFailure {
    error: ContractError, operation_id: OperationId, stage: BeginStage,
    last_confirmed_stage: BeginStage, persisted_evidence: Vec<String>,
}
pub struct StartEvidence {
    target: TargetEvidence, existing_operation_ids: Vec<OperationId>,
    previous_operation: Option<OperationRecord>, inventory_complete: bool,
    verification_ref: String, budget: OperationBudget,
}
pub fn prepare_start(guard: &OperationGuard, plan: &Plan,
    operation_id: OperationId, started_at: UtcMillis, deadline: std::time::Instant,
    evidence: &StartEvidence)
    -> Result<OperationStart, ContractError>;
```

EvidenceFieldの外部形式は[既存候補の3キー](#対象証拠と状態結果)へ一意に変換する。enumの内部表現を新しいJSON形式として流出させない。IDのparseは表記検査だけ、生成はOS乱数・衝突照合を伴う別工程。TargetEvidenceをCloneしても観測時点の証拠の複製にすぎず、再検査済みにはならない。

ChangeFailureは開始後の部分結果を消さない共通の返却規約候補。開始前だけpartial=Noneを許容し、実適用または開始記録後はSomeを必須とする。persisted_evidenceは実在を確認した記録参照だけで、保存不能時は空を許すがerrorへ記録失敗を保持する。プロセス強制終了でこの返却自体がない場合は永続記録と実状態の照合へ戻る。ExecutionContext・OperationGuardの実所有権とOS上の寿命は`D02-04`のまま別途確定が必要であり、宣言から推測しない。

## 状態と中断の詳細案

ID表記器のString返却は上記の各parse関数で対応型へ変換する候補。parseは表記だけを確認し、乱数源・既存集合・一意性の証拠を生成しない。Plan/Operation/Machineの誤った種別はMODEL_IDで拒否する。A02-71/82には三種の相互誤用と、正常表記でも未使用確認がない開始要求を含める。

`P02-STORE`の確定候補は、排他取得→running同期→実適用→実状態検証→履歴確定→終了記録確定→補助記録照合→completeとする。各境界の中断判定を以下へ固定する案で、複数ファイル全体が原子的だとは主張しない。

| 観測した位置 | 復旧側の扱い |
|---|---|
| runningを保存できず実適用なし | 開始失敗。変更を開始しない |
| runningあり、実適用の位置不明 | 中断・照合待ち。未適用と推測しない |
| 実状態は期待どおり、履歴なし | 適用済みだが記録未完了。同じ変更を再適用しない |
| 履歴あり、operationがrunning | 履歴のID・結果と実状態を照合する復旧計画。通常finishで上書きしない |
| 終了記録あり、last-success等が不一致 | 必須記録集合未完了。completeを返さない |
| 全記録一致、応答のみ欠損 | 読取りで同じ確定結果を回収。適用は再送しない |

必須記録の全集合・同期失敗時の保存場所は`D02-05`に残し、上表があるだけで確定処理を実装可能としない。状態キャッシュへ写すCheckの候補対応はpass→pass、必須fail→fail、任意fail→warn、unknown→unknown、根拠のあるnot_applicable→warn。キャッシュのmessageに非適用理由を残し、必須検査の欠損をwarnへ落とさない。キャッシュは真実の代替ではなく、読取り時には書き戻さない。

`P02-EXEC`は起動前・起動済み・停止要求済み・終了確認済みを区別する。起動後のtimeout/出力超過/取消は一度だけ停止要求へ移り、子孫・I/O回収の終了を確認できない場合はstop_unknownとする。Childの破棄は終了確認ではないことを[公式資料](https://doc.rust-lang.org/std/process/struct.Child.html)へ照合した。以後の変更は保留し、ガードの破棄だけを排他安全性の証明にしない。

## 依存と互換性の詳細案

共通モジュール間の依存候補はcli→model/format/state/store、state→model、store→model/platform、platform-command→model。formatはprofilesやstoreへ依存せず、解析だけでOS変更・保存を開始しない。循環依存をcrate分割で隠さない。

| 担当 | 標準で使う範囲 | 残る内製・外部境界 |
|---|---|---|
| MODEL | String・Option・Result・SystemTime・Instant | 識別生成・暦日変換・期限判定は内製。OS乱数/起動情報の取得契約はD02-01 |
| PARSE/CLI | UTF-8・整数・BTreeMap・Vec・Write | 文法・スキーマ・診断・出力保全は内製。外部parser crateなし |
| STORE | File・OpenOptions・同期・非待機ロック | 複数記録照合・親停止・対象固定は別に成立性確認 |
| EXEC | Command・Child・Instant | 子孫停止・入出力の期限・監督単位はD02-04。FFIを迂回採用しない |
| bundle圧縮・HTTPS | 標準機能だけで充足認定しない | 外部OSツールの個別例外候補は台帳で審査。独断でtar/curlを使用しない |

スキーマ識別候補は`共通契約ID/形式名/版`。config・policy・managed・install・state・operation・history・self_transaction・version・last_success・last_failure・CLI envelopeを別の形式名として管理し、同じschema_versionの数字だけで互換としない。各キー全集合は担当節へ対応付け、旧二択のprofileを含む記録は互換根拠がない限り書換え停止とする。

将来の内製crate差し替えは、parseの受理集合・型・エラー位置、CLIのbyte列/終了値、stateの優先順、storeの障害位置、execの終了/残存プロセスを比較対象にする。新旧へ同じfixtureを与え、差異があれば仕様採用・移行を先に判断する。内部のstruct配置だけを理由に保存版を更新せず、外部表現が変わるときは版・移行・切戻しの全契約を確定する。

## 受入条件の充足照合案

MODELは[型とAPI](#型とapiの詳細案)、CLI/PARSEは各詳細節、STATEは既存判定順、STORE/EXECは[中断表](#状態と中断の詳細案)と[依存境界](#依存と互換性の詳細案)へ対応付ける。スキーマ名の列挙だけでは各キー全集合と移行を充足しない。

| ケースID | 区分・固定入力 | 期待結果と確認 |
|---|---|---|
| `A02-76` | 異常。実適用後のChangeFailureでpartial=None | 返却契約違反を検出。実適用範囲を不在として後続へ渡さない |
| `A02-77` | 異常。history確定、operation=running、補助記録は前操作 | complete不可。3記録と実状態の照合を要求、履歴再追加・再適用0回 |
| `A02-78` | 正常・異常。必須fail/任意fail/根拠あり非適用のCheckをキャッシュへ変換 | 候補対応のfail/warn/warnを照合。必須集合の欠損はwarnにせず、読取りコマンドの保存0件 |

採用前に残る判断を、型/ID/時刻、CLI全検査と出力上限、全形式スキーマと移行、実行・排他寿命、複数記録確定、圧縮/取得の6判断へ対応付ける。特にstd::fs/processのAPIが存在することは、安全なroot固定・子孫停止・全I/Oの期限保証の実現根拠ではない。計測・反証試験の設計と実結果を分離し、未実施を自動検査の成功で閉じない。

## 入力と保存形式の詳細案

P02-PARSEの後段へ渡すスキーマ登録を以下の候補へ固定する。既存キーの値域はリンク先の定義だけを使用し、本表は未定義の必須性・型を補う。全objectで未知キー・重複・型強制変換を拒否する。日付はUtcMillis、版は基本仕様の製品版契約、revisionは0以上のu32とし、未対応版を自動変換しない。

| 登録対象 | 完全なキー集合の所在と補完案 |
|---|---|
| config.yml | [基本設定](../BASIC_SPEC.md#configyml)の集合だけ。root.serverseed以外なし。省略可能項目だけポリシー既定値を使用。config_revisionは対応表へ完全一致 |
| policy.yml | [共通設定](../BASIC_SPEC.md#policyyml)の集合だけ。root.policy以外なし。省略時とnullを区別しnullは型エラー |
| managed | [マーカー](../BASIC_SPEC.md#管理マーカー)の7キーだけ。managedはbool、machine_idはMachineId、created_atはUtcMillis、両revisionはu32、profileは三構成enum。null不可 |
| machine-id | MachineIdのASCII32桁とLF一つだけ。JSON・空白・OS machine-idの暗黙流用なし |
| version | serverseed_version: 製品版string、config_revision・migration_revision: u32、profile: 構成enum、operation_id: OperationId、updated_at: UtcMillis、last_successful_operation: OperationIdまたは初回成功前null、last_successful_operation_at: UtcMillisまたは同条件null。全キー必須。既存の製品バージョン管理の必須情報を保持 |
| install.json | [導入情報](../BASIC_SPEC.md#本体導入と構成管理)の6キーだけ。architectureは既存2値、binary_sha256は小文字hex64桁、installed_atはUtcMillis。removedでも旧値を保持 |
| state.json | [キャッシュ](../BASIC_SPEC.md#状態スナップショット)のキー集合。os/boot/serverseedの各子集合も例のキーで閉じる。未取得の観測stringはnull、revision/profileのnull条件は正本に従う。checksはid/result/messageの全キー必須 |
| operation.json | [操作状態](../BASIC_SPEC.md#操作状態)のキーにboot_id: string、process_start_ticks: u64、target: TargetEvidenceを追加する案。pidは1以上u32。runningだけfinished_at=null、lock_pathは固定値 |
| last-success・last-failure | operation_id・plan_id・target・finished_at・result・history_sha256を全キーとする。last-successはsuccess、last-failureはfailedまたはinterrupted。hashは該当履歴行のLF込みbyte列。初回未作成は許容、破損は不在扱い不可 |
| history.jsonl | [履歴](../BASIC_SPEC.md#実行履歴)の集合にtarget: TargetEvidence、plan_id: PlanIdを追加する案。両versionとprofileだけ未導入時null可。resultはsuccess/failed/interrupted、完結したJSONとLFを一行として扱う |
| self-transaction/state.json | [本体取引](../BASIC_SPEC.md#本体の置換と復旧)の8キーだけ。phaseは同節の工程へ対応付ける。before_sha256/previous_pathは初回導入時null、after_sha256/staged_pathはuninstall時null。それ以外のhashはhex64桁、パスは固定生成先だけ |
| ログ・成果物 | [ログ](../BASIC_SPEC.md#ログ形式)・P02-CLIのreport/plan/bundleを担当形式とする。状態記録をログ形式で代用しない |

追加するoperation/historyのキーは採用前の既存記録へ書かない。形式の版変更と旧形式の移行可否はP04-UPDATEの互換表へ結合する。チェック集合はos.supported、architecture.supported、target.identity、records.consistent、profile.selected、operation.clear、dependency.approved、plan.currentを共通候補とし、各入口は担当契約の検査を加える。適用外は理由必須、欠損をpassへしない。実行環境だけのversion/helpには対象検査を要求しない。

versionのlast_successful_operationと時刻は両方nullまたは両方非nullだけを許す候補。成功確定時だけ同じoperation/finished_atへ更新し、失敗・中断では直前の成功値を保持する。operation_id/updated_atは現在の版記録を更新した操作/時刻であり、最後の成功を常に意味するわけではない。初回成功前の架空ID・時刻を生成せず、last-successと不一致なら記録照合へ戻す。

operationの候補検証ではstatus=runningだけfinished_at=null、終端三状態はUtcMillisを必須とし、開始/終了時刻の読取り不能を現在時刻で埋めない。historyの同じoperation_id・command・target・profile・result・開始/終了時刻を照合し、operationのstatusとhistoryのresultの名前の違いを別結果としない。last-resultのhistory_sha256は保存済みの当該一行とLFへ結合し、history全fileのhashを入れない。時計補正で壁時計が後退した場合も経過時間を差算せず、実行期限の根拠は単調時計へ分離する。

| ケースID | 固定入力 | 期待結果 |
|---|---|---|
| `A02-79` | managed.profileとconfigが不一致、operationに未知キー、last-successの履歴hash不一致 | 構文成功後も整合検査失敗。キャッシュ補完・不在扱い・成功記録なし |

## 実行と復旧の確定候補

P02-EXECの第一候補はOSの一時serviceで変更操作全体を監督する方式とする。CLI配送側ではなく対象側の同一本体workerを変更の親とし、このworkerが共通guardを取得し、operation開始から必須記録の確定まで保持する。常駐daemonや独立agentは新設しない候補。内部入口の具体構文・呼出し権限は公開CLIと分離して採用審査する。

[systemd-run 255](https://www.freedesktop.org/software/systemd/man/255/systemd-run.html)の一時serviceはmanagerが親となる。[KillMode](https://www.freedesktop.org/software/systemd/man/255/systemd.kill.html)のcontrol-groupを候補にし、単一Childのkillやscopeへの置換を互換とはしない。unit名はOperationIdへ固定し、Type=exec、KillMode=control-group、SendSIGKILL=yes、TimeoutStopSecは呼出元grace、RuntimeMaxSecは停止・記録予約を除いた残時間へ対応付ける。未採用のOS機能を使用可能にしたものではない。

| 中断位置 | 対象側の処理と次回の判定 |
|---|---|
| worker開始前 | 変更0件。開始不成立の証拠がある場合だけ未実行 |
| guard取得後・begin失敗 | 対象変更0件。記録不足とエラーを返し、既存記録を削除しない |
| 実適用後・history未確定 | runningと資源journalを保持。復旧照合まで後続変更不可 |
| history確定・operation未確定 | 同じ操作行のhashと実状態を照合。履歴の再追加なし |
| operation終了・補助記録未確定 | complete不可。last-result/version/stateの必要集合を照合して是正計画を提示 |
| 親異常終了・子孫残存 | guardが解放されても新規操作を開始しない。残存unit・operation・実状態を照合する |

履歴は候補上限64 MiB、各行64 KiB、超過予定ならbegin前停止とし、自動削除・退避で空けない。各記録の一時物・旧物・履歴全コピーの必要容量と親同期を事前計算する。記録容量の見積と書込み成功を区別する。ブロックしたFSや停止不能な子孫を時間内終了と偽らずstop_unknownへ移し、結果を失った要求の再適用を禁止する。

利用者指定先の新規成果物確定候補は、同じFSの保護された親へcreate_newで一時file作成→全byte書込み/sync_all→新規最終名へのhard_link→親directory同期とする。[hard_link](https://doc.rust-lang.org/std/fs/fn.hard_link.html)の既存宛先での失敗を利用し、存在確認後のrenameで上書きしない。root/親の固定はP03-FSが別に必要。link非対応・別FS・既存宛先・同期不能ならsavedを返さず、確認済み一時物と確定位置を保持する。最終名作成後の同期失敗を未作成と報告せず、無条件の削除/再生成はしない。管理記録の置換は従来のrename契約と区別する。採用FSでの実証はD02-05に残す。

最終名の親同期成功後だけ、同じinode/device/既知一時名/nlinkを再照合して一時名をunlinkし、親を再同期する候補とする。最終名を後始末の削除対象にしない。一時名が別物へ変わったら削除0回、後始末失敗はARTIFACT_CLEANUPと最終名確定済みの部分結果を保持する。再実行は同名の新規生成を行わず、固定byte列/hash/最終名の一致と残存一時物の照合から始める。最終名の作成前失敗と作成後失敗を同じ「保存なし」にしない。後始末と二回の親同期も元の工程期限/容量へ含める。

| ケースID | 固定入力 | 期待結果 |
|---|---|---|
| `A02-119` | 最終名同期後に一時名unlink失敗、次に一時名が第三者fileへ交換 | 確定位置と失敗を保持、再生成/最終名削除なし。交換後のfileは削除せず照合待ち |

| ケースID | 固定入力 | 期待結果 |
|---|---|---|
| `A02-80` | worker強制終了直後に子孫残存、次のCLIがlock取得成功、履歴64 MiB超過予定 | ロック取得だけでは続行不可。残存/中断照合または容量不足でbegin/適用0件 |

## 依存と受入の確定候補

P02-CLIのCheck.idは既存の契約ID/local形式を維持する。[共通集合](#入力と保存形式の詳細案)の省略名はP02-STATE/os.supported、P02-STATE/architecture.supported、P02-MODEL/target.identity、P02-STORE/records.consistent、P02-STATE/profile.selected、P02-STORE/operation.clear、P01-CHECK/dependency.approved、P03-PLAN/plan.currentへ対応付ける候補。これを裸のdot形式で直列化しない。

status/inspect/doctorは対象に適用するOS/CPU/同一性/記録整合を必須とする。profile.selectedは構成管理の証跡がある対象だけ必須、未管理と確認した対象ではrequired=false・not_applicableと未構成理由を返す。operation.clearは新規変更の開始条件であり、読取り時は実行中/中断の事実を保持する任意項目とする。実行中である事実だけで読取り禁止・不整合としない。記録破損・対象取得不能は別の必須検査で検出する。

planと変更操作は適用可能な八検査と担当step検査を要求する。self単独で構成未作成の場合のprofile.selectedは上記と同じ非適用根拠を必要とし、選択済みと捏造しない。version/helpは対象の必須集合を持たず、versionの対象取得不能は理由付きで分離する。reportは記録整合、support-bundleは記録整合と収録/秘匿化検査、verifyは対象構成の全必須検査を結合する。取得不能を非適用へ変更しない。

スキーマvalidatorは登録対象enumとValueを受け、検証済み設定/記録のenumを返す候補。未知登録型を汎用Valueのまま適用器へ通さない。解析器からOS観測器・保存器への依存を作らず、初期はparse/model/store/execを別moduleとして保つ。型を後でcrateへ抽出する場合、JSON/YAMLの成功木・失敗位置・既存記録互換を同じfixtureで新旧比較し、形式移行を単なるimport変更へ混入しない。

外部コマンドの通常captureは従来の1 MiB上限を維持する。大きい配布物は新しい専用file-sink契約の採用が必要で、取得先・新規File・最大byte数・OS強制上限・同期/hash検査を固定する。単にstdout制限を解除する実装は禁止候補。Fileの[同期/排他API](https://doc.rust-lang.org/std/fs/struct.File.html)の存在だけでパス固定・複数記録確定・I/O停止可能性を充足としない。u64で表す外部整数もP02-PARSEのi64上限を超える値は拒否する。

| ケースID | 固定入力 | 期待結果 |
|---|---|---|
| `A02-81` | 必須check一件欠損、裸dotのid、通常captureへ128 MiB本体、u64::MAXの記録値 | それぞれ契約不一致/上限で拒否。大容量取得を通常capture成功へ変換しない |
| `A02-82` | 固定16byteのID生成、既存集合取得不能、同じpidでboot_idまたは開始tickだけ変化 | 表記は決定的、集合不明は生成停止、プロセス不一致。時刻/PIDへの代替と再生成ループなし |
| `A02-83` | 正常な未管理OSのstatus、変更実行中のinspect、u32上限を超えるBinaryInfo revision | 前二件は未構成/実行中を保持して読取り可否を判定。後者は型検証拒否、保存形式と別の範囲を受理しない |
| `A02-84` | 正しい版に未知キー・必須欠損・型不正を併存、次に未知版へ変更 | 前者SCHEMA_UNKNOWN、後者SCHEMA_VERSIONを先に返す。キー削除や暗黙変換なし |
| `A02-85` | guardを先にdropするcontext、監督unit不明なのにlockだけ取得成功 | 前者は所有権契約不成立、後者は変更開始不可。Rustの借用成立をOS子孫停止の証明にしない |
| `A02-86` | failed操作の履歴は確定、last-failure同期失敗、以前のlast-successあり | complete不可、以前のlast-successを保持。同じ失敗行の追記・元結果の成功化なし |
| `A02-87` | 固定memberに../名または同名linkを混入、生成物へ一件の余剰entry | 圧縮前または照合で拒否し正式保存なし。別用途の抽出・秘密項目の追加採取なし |
| `A02-88` | 参照commitだけ別ツリー、またはadopted=trueだが所有者の採用根拠なし | MODEL_REFERENCE。現在の同名節やbool自己申告で参照を成立させず、取得/修復0回 |
| `A02-89` | 任意fail一件と必須pass集合、次に必須fail追加、最後に確定後stdout失敗 | 順に警告成功/検証失敗/送達終了1。errors非空のsuccessと確定操作の再適用なし |
| `A02-90` | kind=ConfigへInstall形式、キーa/b~cの型不正、rootがarray | 別kindへ再解釈せず拒否。fieldは/a~1b~0c、root型不正は空PointerとSCHEMA_TYPE |
| `A02-91` | 対象記録は整合しoperation=running、inspect/status/updateをそれぞれ要求 | 総合状態は全てRECOVERABLE。読取りは観測結果を返し、updateは開始0回。状態と終了分類を混同しない |
| `A02-92` | 同じoperationの既存完結行と同一/一byte相違、別途未完結末尾 | 同一は追記0回で照合、相違はSTORE_CONFLICT、未完結は復旧待ち。自動切除・partial喪失なし |
| `A02-93` | ReadOnly文脈から変更コマンド要求、起動後timeout、spawn失敗 | 前者は起動前Errで管理領域不変。後二件はCommandOutputに区分と取得済み出力を保持し、Okを成功へ変換しない |
| `A02-94` | 解析成功Valueをvalidatorなしで適用、開始後失敗をContractErrorだけへ縮約、同名の旧APIを追加 | 型付き境界/ChangeResult契約の違反。適用器へ通さず、部分結果・公開契約を一意に照合 |
| `A02-95` | PIDだけ一致しboot/startが相違、proc読取り権限なし | 主体不一致/取得不能を区別。PID存在だけで所有/停止完了を認定しない |
| `A02-96` | writer flush成功だが受信側未解析、SSH受付記録なし | byte書込みと相手の受理/永続化を別判定。操作の再送・成功範囲拡大なし |
| `A02-97` | required_recordsからhistory除去、interrupted結果、最後の成功値あり | 集合不一致を拒否。中断はlast-failureへ元statusで記録し、成功値は保持 |
| `A02-98` | timeout後exit=0、出力超過後停止未確認、取消とtimeout同時 | 順にtimed_out/stop_unknown/cancelled。後から得た通常終了で失敗原因を消さない |
| `A02-99` | 一時file同期後に最終名が先に出現、link成功後の親同期失敗 | 既存宛先不変、後者は作成済み/未同期として保存失敗。上書きrename・無確認削除なし |
| `A02-100` | versionの最後の成功二値が片方null、失敗操作で両値を更新 | 形式/照合失敗。直前の成功参照を消さず、版更新操作と成功操作を混同しない |
| `A02-101` | statのcommに空白と閉じ括弧、別PID、starttime欠損、boot_id前後不一致 | 第22フィールドを正しく特定し、別PID/欠損/世代変化は不採用。commを表示せず時刻/PIDだけで主体を認定しない |
| `A02-102` | runningに終了時刻、failedにnull、last-resultへhistory全fileのhash | 記録形式または同一操作の照合で拒否。終端時刻・一行hashを推測で補完しない |
| `A02-103` | 実行物は固定だがcwd未指定、環境名に=、親のproxy/loader変数だけ存在 | 前二件はEXEC_SPECで起動0回。親変数を継承せず宣言集合だけ渡し、PATH探索・shell補完なし |
| `A02-104` | timeout=grace、親残量が停止予約以下、通常期限到達後のstdin未配送 | 起動前停止または停止/回収へ移行。猶予の外付け延長・期限後の新規配送・未観測の終了捏造なし |
| `A02-105` | 生成bundleへ余剰gzipストリーム、link entry、宣言量より長いmember | BUNDLE_ARCHIVEで正式保存なし。展開量を制限し、アーカイブ検証を復号・任意pathへの復元と混同しない |
| `A02-106` | 観測時刻だけ相違、必須disk証拠unknown、起動後rootの予定変化 | 時刻だけは別対象にせず、未知diskは一致未確認。予定変化は担当引渡しで検証し、必須集合縮小で通さない |

## 未決事項と成立判定

### 共通スキーマの型対応

validate_documentの登録対象を次の13形式へ閉じる候補とする。machine-idはJSONでないためparse_machine_idのASCII検証だけを使い、ログ/他フェーズの形式は各担当validatorへ渡す。未登録型を本enumの汎用Valueへ取り込まない。各structの外部キー・null・値域は[登録表](#入力と保存形式の詳細案)、出力は[P02-CLI](#p02-cli)だけを参照し、型から別のJSON名を生成しない。

```rust
pub enum SchemaKind {
    Config, Policy, Managed, Install, State, Operation, History,
    SelfTransaction, Version, LastSuccess, LastFailure, CliEnvelope, Event,
}
pub enum ValidatedDocument {
    Config(ConfigDocument), Policy(PolicyDocument), Managed(ManagedRecord),
    Install(InstallRecord), State(StateSnapshot), Operation(OperationRecord),
    History(HistoryRecord), SelfTransaction(SelfTransaction), Version(VersionRecord),
    LastSuccess(LastResultRecord), LastFailure(LastResultRecord),
    CliEnvelope(Envelope), Event(EventRecord),
}
pub enum Environment { Usb, Cloud }
pub enum OperationStatus { Running, Success, Failed, Interrupted }
pub struct ConfigDocument { serverseed: BaseConfig }
pub struct BaseConfig {
    config_revision: u32, environment: Environment, profile: Profile,
    log_level: String, allow_reboot: bool, update_channel: String,
}
pub struct PolicyDocument { policy: CommonPolicy }
pub struct CommonPolicy {
    require_backup_before_update: bool, require_ssh_recovery_path: bool,
    allow_password_auth_disable: bool, allow_unattended_security_updates: bool,
    allow_destructive_disk_setup: bool,
}
pub struct ManagedRecord {
    managed: bool, machine_id: MachineId, created_at: UtcMillis,
    serverseed_version: String, config_revision: u32,
    migration_revision: u32, profile: Profile,
}
pub struct InstallRecord {
    schema_version: u32, status: String, serverseed_version: String,
    architecture: String, binary_sha256: String, installed_at: UtcMillis,
}
pub struct OperationRecord {
    schema_version: u32, operation_id: OperationId, command: String,
    status: OperationStatus, started_at: UtcMillis, finished_at: Option<UtcMillis>,
    pid: u32, plan_id: PlanId, lock_path: std::path::PathBuf,
    boot_id: String, process_start_ticks: u64, target: TargetEvidence,
}
pub struct HistoryRecord {
    operation_id: OperationId, command: String, started_at: UtcMillis,
    finished_at: UtcMillis, result: OperationStatus, profile: Option<Profile>,
    serverseed_version_before: Option<String>, serverseed_version_after: Option<String>,
    target: TargetEvidence, plan_id: PlanId,
}
pub struct VersionRecord {
    serverseed_version: String, config_revision: u32, migration_revision: u32,
    profile: Profile, operation_id: OperationId, updated_at: UtcMillis,
    last_successful_operation: Option<OperationId>, last_successful_operation_at: Option<UtcMillis>,
}
pub struct LastResultRecord {
    operation_id: OperationId, plan_id: PlanId, target: TargetEvidence,
    finished_at: UtcMillis, result: OperationStatus, history_sha256: String,
}
pub struct SelfTransaction {
    schema_version: u32, operation_id: OperationId, command: String, phase: String,
    before_sha256: Option<String>, after_sha256: Option<String>,
    staged_path: Option<std::path::PathBuf>, previous_path: Option<std::path::PathBuf>,
}
```

OperationStatusのRunningはHistory/LastResultで拒否し、LastSuccessはSuccessだけ、LastFailureはFailed/Interruptedだけを受理する。Policy省略値の補完は既存の担当ポリシーから一回だけ行い、nullは省略にしない。config_revisionの対応値・旧形式からの移行はP04-UPDATEの採用対象で、enumの追加だけで未知revisionを受理しない。StateSnapshot/Envelope/EventRecordの子型は[検査・観測の固定集合](#検査観測の固定集合)と[記録集合と保存順序](#記録集合と保存順序)で閉じる。

| ケースID | 固定入力 | 期待結果 |
|---|---|---|
| `A02-107` | kind=LastSuccessにfailed、Historyにrunning、Policyにnull、MachineIdをJSONとして投入 | 型/形式不一致で拒否。別kindへの切替・null既定化・終端結果の補完なし |

### 検査観測の固定集合

P02-CLIの型を下記へ具体化する候補。status/reasonの値域は同契約を参照する。キャッシュ用SnapshotCheckだけは既存のresultキーを保持し、CLI Checkと相互代用しない。

```rust
pub struct Diagnostic { code: String, message: String, field: Option<String> }
pub struct Check {
    id: String, required: bool, status: String,
    code: Option<String>, field: Option<String>, message: String,
}
pub struct Observation {
    id: String, status: String, value: Option<String>, reason: Option<String>,
}
pub struct Envelope {
    schema_version: u32, command: String, result: String,
    started_at: UtcMillis, finished_at: UtcMillis, state: String, summary: String,
    data: CommandData, warnings: Vec<Diagnostic>, errors: Vec<Diagnostic>,
}
pub struct SnapshotCheck { id: String, result: String, message: String }
pub struct SnapshotOs { id: Option<String>, version_id: Option<String>, kernel: Option<String> }
pub struct SnapshotBoot { mode: Option<String>, secure_boot: Option<String> }
pub struct SnapshotServerseed {
    installed: bool, profile: Option<Profile>, version: Option<String>,
    config_revision: Option<u32>, migration_revision: Option<u32>,
}
pub struct StateSnapshot {
    schema_version: u32, state: String, detected_at: UtcMillis,
    machine_id: Option<MachineId>, os: SnapshotOs, boot: SnapshotBoot,
    serverseed: SnapshotServerseed, checks: Vec<SnapshotCheck>,
}
```

CommandDataは[P02-CLIのコマンド別data](#コマンド別data案)へ次の型で一対一対応させる。外部JSONへenumのvariant名・追加のtagキーを出さず、Envelope.commandとmodeで選択する。Help/Errorだけは専用形、MasterDataは[P08の単一定義](PHASE_08.md#マスターcliの解析と応答全集合)を参照する。共通モデルへ置く型の担当正本と、Rustモジュールの依存方向を混同しない。

```rust
pub enum CommandData {
    Status(StatusData), Verify(VerifyData), Inspect(InspectData), Doctor(DoctorData),
    Report(ReportData), Version(VersionData), Plan(PlanData), Change(ChangeData),
    Bundle(BundleData), Help(HelpData), Error(ErrorData), Master(MasterData),
}
pub struct OperationRef { operation_id: OperationId, status: OperationStatus, command: String }
pub struct RevisionRange { min: u32, max: u32 }
pub struct Artifact { path: std::path::PathBuf, format: String, status: String }
pub struct BinaryInfo {
    serverseed_version: String, architecture: String, config_revision: u32,
    migration_revision: u32, compatible_config_revision: RevisionRange,
    compatible_migration_revision: RevisionRange,
}
pub struct AppliedInfo {
    serverseed_version: String, config_revision: u32, migration_revision: u32,
    environment: Environment, profile: Profile,
}
pub struct StatusData {
    checks: Vec<Check>, pending_operation: Option<OperationRef>,
    profile: Option<Profile>, setup_required: bool,
}
pub struct VerifyData { scope: Vec<String>, checks: Vec<Check> }
pub struct InspectData {
    execution_environment: Vec<Observation>, target: Vec<Observation>, checks: Vec<Check>,
}
pub struct DoctorData {
    installation_status: String, setup_required: bool,
    checks: Vec<Check>, transaction_id: Option<OperationId>,
}
pub struct ReportData {
    operation: Option<OperationRef>, record: Option<OperationRecord>, checks: Vec<Check>,
    missing_records: Vec<String>, artifact: Option<Artifact>,
}
pub struct VersionData {
    binary: BinaryInfo, applied: Option<AppliedInfo>,
    observations: Vec<Observation>, checks: Vec<Check>,
}
pub struct PlanData {
    requested_command: String, plan: Option<Plan>, checks: Vec<Check>, artifact: Option<Artifact>,
}
pub struct ChangeData {
    mode: String, plan: Option<Plan>, operation: Option<OperationRef>,
    checks: Vec<Check>, requires_reboot: bool,
}
pub struct BundleItem { name: String, status: String, bytes: Option<u64>, reason: Option<String> }
pub struct BundleData {
    collection_status: String, items: Vec<BundleItem>, excluded: Vec<Diagnostic>, artifact: Option<Artifact>,
}
pub struct HelpData { help_for: String, usage: String, commands: Vec<String>, options: Vec<String> }
pub struct ErrorData { error_stage: String, help_for: Option<String> }
```

Stringで表す列挙値も上表の許容集合だけへ検証し、任意文字列を通さない。Plan/OperationRecordは担当型を参照し再宣言しない。Planは非再帰payloadを必要とし、CommandDataをPlanへ内包しない。reportのoperation/recordは両方nullまたは同一ID/command/statusの組だけを受理する。型宣言は採用承認・外部観測器の成立根拠とは別で、残件の担当はD02-02とする。

| ケースID | 固定入力 | 期待結果 |
|---|---|---|
| `A02-114` | statusにVersionData、異なるoperation/record ID、変更開始後をErrorDataへ変換 | 全てCLI_RENDER。未知キーや汎用Valueへ退避して受理せず、適用済み部分結果を保持 |

キャッシュのos/kernel/boot等がnullの場合は根拠をchecksへ残す。内蔵OSのmodeはuefi/bios、Secure Bootの表現は既存キャッシュのenabled/disabled/unknownを維持する。unknownは状態を特定できない観測、nullは項目自体を取得できない場合へ区別し、unknownをnullに書き換える移行は本案で行わない。

inspectのObservation集合候補は次表。execution_environmentとtargetの両配列に、それぞれ指定したIDを一回ずつ含める。取得不能はunknownと固定reason、非適用は担当条件に根拠のある場合だけ。IDが存在してもvalueの自己申告だけで収集成功にしない。

| 配列・ID | 取得対象と表現 |
|---|---|
| execution_environment: execution.id / os.id / os.version / kernel.release / cpu.architecture / boot.id / systemd.state | 実行側の識別済みOS、uname相当のkernel/CPU、proc起動ID、OS managerの実状態。対象側の値を流用しない |
| target: target.disk / target.root / os.id / os.version / boot.mode / boot.secure_boot | 対象disk/ルートと対象内os-release、対象の起動構成。空diskでroot/OS不在を確認した場合だけnot_applicable |
| target: storage.filesystems / network.interfaces / ssh.configuration / docker.installation / serverseed.installation | 各採取器の検証済み結果の証拠参照ID。全一覧を自由文字列へ連結せず、対象別の収集資料へ対応付ける |
| target: network.reachability | inspectでは常にunknown・reason=INSPECT_NO_NETWORK。外部接続を実施しない |
| version: os.id / os.version / kernel.release / applied.source | 対象側で読めた版と適用記録の検証参照。未構成のapplied.sourceだけ根拠付きnot_applicable |

root/OS以外でも適用外を使う場合は担当契約の明示条件が必要で、未採取・コマンド欠損・権限不足をnot_applicableにしない。ReadOnlyの観測器は管理領域を作成せず、検査中に情報が変わればMODEL_CHANGED。idに動的パス・秘密値を使わない。

### 記録集合と保存順序

容量予約には各操作のrequest/intent/events、最終snapshot、資源journal、既存historyの全コピー、各置換時の旧物/一時物をFS別に含める候補。寿命が重ならない一時物は実行順に求めた同時保持最大量へ算入し、同一実体の二重算入も無根拠な除外もしない。未知の最大量/保存先/参照閉包はSTORE_CAPACITYでbegin前停止。記録16 MiB予約を既存history等の必要量へ置き換えず追加し、空き容量はguard下で再取得する。開始後に予約を超えても過去証拠の自動削除/別FS退避はしない。A02-110へ複数FS/最終snapshot二重保持/旧historyコピー不足を含める。

操作別snapshotは各可変記録の最終確定byte列にだけ一回生成する候補。running版operationのsnapshotを終端版で上書きせず、beginの証拠は操作別eventsとintentへ残す。finishでは各最終記録を同期→その同一byte列の新規snapshotを同期→全hash照合→operation.end→intent.completeの依存順を守る。historyの論理追記行と操作別eventsの行はAppendBindingへ対応し、可変記録のsnapshotと混同しない。中断後の同名snapshotはbyte完全一致の照合だけを許し、差異はSTORE_CONFLICT。A02-115/116へrunning snapshot残存/最終snapshot未同期/同名別byteを追加して照合する。

OperationStart.deadlineは今回の同一起動世代の外側実行器が固定した最終期限を渡す内部値で、JSON/保存記録には出さない候補。prepare_startは値を保持し、beginは受付時に残期限を検査してExecutionContextへ無延長で移す。begin受付から新しい全予算を作らず、開始同期/イベントの時間も元期限へ算入する。到達済みなら開始失敗、保存を開始した後の超過/応答不明はBeginFailureの確認済み段階を保持する。A02-104/123にprepare後beginまでの待機で期限到達した入力を含める。

開始識別の候補順は、OS取得器によるOperationId/開始時刻の生成→監督unitと要求への束縛→対象guard取得→prepare_start→beginとする。prepare_startは取得済み値を照合するだけでID生成/時計/保存を行わず、親Planとguardの対象一致、ID未使用、未完了操作なしを検証する。begin内部でIDを新規生成しない。OperationStartは非Clone、消費は一回だけで、intentとrunningとイベントは同じID/開始時刻へ結合する。

ID未使用/前操作の検査入力はStartEvidenceからだけ受ける。収集器がguard下で対象の全必要記録と監督単位を照合して構築し、inventory_completeの自己申告だけでは生成できない。existing_operation_idsは既存の操作/履歴/対応記録のID集合、previous_operationは不在の確証だけNoneとし、読取り不能をNoneへ補完しない。別target、集合欠損/重複、同じIDの既存操作、running/interrupted/未確認の前操作はSTORE_RECONCILE。今回の開始要求への予約と、既に開始した同じIDの記録を区別する。純粋prepare_startから不足資料の読取り/復旧を開始しない。A02-123にはこの入力欠損/別対象も含める。

初回の固定管理directory/lockの確保は基本仕様が許す準備だけに限定し、既存記録を初期化しない。intent保存後のrunning/イベント失敗では開始記録の残存を保持し、資源変更0件でも同じIDを新規beginへ再利用しない。resume_media等は別の再開契約で照合する。開始記録後の失敗は残存intent/開始位置を保持した開始失敗として報告し、対象適用開始後のChangeFailureと区別する。

| ケースID | 固定入力 | 期待結果 |
|---|---|---|
| `A02-123` | ID生成前のintent要求、intent同期後running失敗、同じOperationStartの再消費 | 前者は型/工程不成立、後二件は資源変更0件で残存記録照合。begin内の別ID生成・既存記録削除・ID再利用なし |

EventRecordの候補キーはoperation_id: OperationId、target_id: MachineIdまたは未構成時null、at: UtcMillis、level: debug/info/warn/error、event_id: 固定ASCII識別子、message: 秘匿化済みstring。すべて必須、未知キー拒否。schema_versionを持たないこの形式は保存先のEvent登録で識別し、他JSONLへ流用しない。対応する非公開フィールドstructをEventRecordとする。保存を許可された変更操作だけが出力し、ReadOnlyの診断をこの記録生成のため保存しない。

```rust
pub struct EventRecord {
    operation_id: OperationId, target_id: Option<MachineId>, at: UtcMillis,
    level: String, event_id: String, message: String,
}
```

target_id=Noneは初回MachineId未生成の確証がある段階だけ許可し、生成後は同じ操作の固定IDへ一致させる。messageは固定分類と非秘密の検証済み情報だけから作り、生の外部診断を流さない。levelの未知値・改行を含むevent_id・別操作IDはEVENT_INPUT。行のJSON escapeは共通serializerに従い、message内のLFを物理的なJSONL区切りへ変換しない。

| ケースID | 固定入力 | 期待結果 |
|---|---|---|
| `A02-121` | MachineId生成後にtarget_id=null、event_idにLF、messageに許可済みLF | 前二件EVENT_INPUT、後者はescapeした一行。識別欠損・ログ行注入・生診断の転記なし |

必須保存集合の算定候補は下表の和集合。固定保存先IDは[管理ファイル一覧](../BASIC_SPEC.md#管理ファイル一覧)と各担当保存先から一意に解決し、外部入力のpathを受けない。開始前にRecordIntentを対象の/var/lib/serverseed/operations/OperationId/record-intent.jsonへ同期する候補とする。全キーはschema_version: 1、operation_id、plan_id、target、required_paths: 一意な固定保存先ID配列、before_hashes・planned_after_hashes・observed_after_hashes: 保存先IDからhex64またはnullへのobject、append_bindings: AppendBinding[]、complete: bool。hashのnullは初回不在または計画済み除去の確証だけで、未確認をnullにしない。開始時complete=false、observed_after_hashesとappend_bindingsは空、まだ生成不能な終了時刻等はplanned_after_hashesへ入れない。終了時に実hashをobserved_after_hashesへ固定し、全required_pathsの照合と同期後だけcomplete=trueへ置換する。後述のintent.completeはこのboolの確定を意味し、未定義の別ファイルやstageではない。

AppendBindingの全キー候補はpath_id: 固定履歴/イベント保存先ID、record_id: 当該操作内の一意行ID、offset: u64、bytes: u64、sha256: hex64。LFを含む確定行の開始位置/長さ/hashを保存し、重複/範囲外/部分末尾を拒否する。排他中に取得した実offsetだけを登録し、未確認の仮offsetを確定値にしない。共有追記先はrequired_pathsの照合対象だが、hash objectではなくこの行単位形式を使う。intent自身は別途同期/完了を照合し、自己参照のhash対象にはしない。

共有eventsのoffsetはローテーションで永続しないため、当該操作が必須とするbegin/endの完結行をoperations/OperationId/events.jsonlへ同一byte列で保存/同期する。必須eventsのAppendBinding.path_idはこの操作別保存先を指し、共有events.jsonlのoffsetを指さない。beginはintentとrunning保存後・最初の対象変更前、endは上記finish位置で確定する。共有ログにも同じevent IDで出すが、共有ログだけの存在を必須証拠としない。共有ログの保存失敗も操作失敗であり、操作別成功によって隠さない。

10 MiB×5世代の整理対象は共有ログだけとし、ロック下で新規beginの前または全完了後に行う。実行中の世代移動は行わず、容量超過で予約を確保できなければ開始前停止する。historyは既定の非ローテーション方式を保持する。操作別events/intent/snapshotは世代整理の対象外で、完了済みの重複eventは同じbyte列だけ再利用する。

| ケースID | 固定入力 | 期待結果 |
|---|---|---|
| `A02-116` | 共有ログ5世代を交換、begin未保存、同じevent IDで異なるbyte列 | 過去操作は操作別eventsで照合。未保存なら対象変更0件、別内容はSTORE_CONFLICT。古い共有offsetへ誤結合しない |

| 条件 | 常に必要な記録・追加条件 |
|---|---|
| 全変更 | intentの独立照合と、operation、当該history一行、対応するlast-result、eventsのoperation.begin/operation.end二イベント。失敗側も同じ対応を保持。intent自身はrequired_pathsの自己参照対象外 |
| self install/update/uninstall | install情報とself取引、release/previous資材。管理対象ならserverseed_versionを更新するmanaged/version。構成revisionの変更なし |
| 初回setup/adopt | config、managed、machine-id、version、選択機能の設定/検証。machine-id生成後の再生成なし |
| 通常update/recover | 実際に変更する版/設定と担当journal/receipt。変更しない版記録は不変。復旧による元結果の改変なし |
| バックアップ工程 | P04-BACKUPの保存物検証と復元証拠、同じ親のstep。子は親historyを個別確定しない |
| media/remote/monitor/registry | 各担当が要求する対応記録も追加。共通operationだけで代用しない |
| state/report/text logを計画した操作 | 計画した保存物を追加し、生成失敗を成功へしない。ReadOnlyでは集合自体を生成しない |

必須量は新しい全byte列+同FSの一時物+保持する旧物+親directory更新をFSごとにchecked合算する。空き容量にはこの和と16 MiBの記録予約を必要とする候補。historyの一行/全体上限は既存候補を再利用する。予約量は実容量・永続化の保証でなく、各write/sync失敗も検出する。begin前の不足は変更0件、開始後は元intentと実配置を保持し、保存先を別diskへ自動変更しない。

finishの確定順は全step検証→終了集合の直列化→history→operation終端→last-result→変更済み版/補助記録照合→operation.end→intent.complete。各置換にfile同期と親同期を要求し、intent.complete前は全体completeを返さない。intent自身は同期/形式/ID/completeを別途照合し、自己hashをrequired_pathsやhash objectへ含める循環を作らない。共有の追記履歴/イベントは当該操作の行byte列とoffset/長さを照合し、後続操作で変わる共有file全体のhashを永続条件にしない。クラッシュ後はintentの集合・実体hash・history行・資源journalを照合する。completeという自己申告だけを信用せず、不一致はSTORE_RECONCILE。eventsの再追加防止キーはoperation_id/event_idで、同キー別内容はSTORE_CONFLICT。ログ整理ではintent/historyを削除しない。

過去操作と現行記録の照合を区別する候補とする。complete前は、required_pathsの変更後実体とobserved_after_hashesを現在位置で一致させる。各可変管理記録の確定byte列は、同じ操作のoperations/OperationId/records/固定保存先ID.snapshotへcreate_newで保存/同期し、同hashを照合する。snapshotは元の形式/byte列を無変更で保持し、拡張子からJSONへ変換しない。保存先IDは英数字・dot・hyphenだけの登録済みIDとし、path入力を連結しない。削除済み項目は除去証拠を持つ担当receiptを保存し、空ファイルを代用品にしない。snapshot保存も必須容量とfinish集合へ含め、intent自体のsnapshotは作らない。

complete後の過去照合はこの不変snapshot・intent・担当journal・当該追記行の対応を対象とし、後続操作のoperation/state/version/last-resultへ古いhashの一致を要求しない。現行管理記録の整合性は最新操作の集合から別途検査する。snapshotなしの旧intentを自動補完せずSTORE_RECONCILE。snapshotの存在は改ざん耐性・現在の実資源状態の保証ではない。保存量が上限に達すれば新規beginを停止し、過去証拠の自動削除は行わない。

| ケースID | 固定入力 | 期待結果 |
|---|---|---|
| `A02-115` | 操作A完了後にBがversion/operationを更新、Aのsnapshot破損、Aが未完了 | 正常なAは過去証拠から照合。破損は拒否。未完了Aは現在実体との照合を必要とし、Bの開始を許可しない |

| ケースID | 固定入力 | 期待結果 |
|---|---|---|
| `A02-110` | self更新がconfig revision変更を要求、operation.end保存失敗、FS別容量の一つだけ不足 | 集合/保護不一致、complete不可、開始前停止を区別。同じhistory再追記・別保存先への自動退避なし |

### 変更workerの固定起動契約

P02-EXECの監督候補は、対象OSのsystem managerに一操作一serviceを生成する方式へ具体化する。unit名はserverseed-OperationId.service、要求保存先はoperations/OperationId/request.json。所有者/種別/RootBinding/元byte列hashを再検証した要求だけをworkerが読む。内部入口は検証済み同一本体の`__internal worker --request 固定保存先`。一般CLIの任意引数としては受理せず、root・system manager起動・同一要求/対象/版・単一実行の確認後だけbeginへ進む候補。名前が内部用であることだけを認証にしない。

起動argv候補は採用済みsystemd-runの絶対path、--unit=上記名、--service-type=exec、下表の--property=KEY=VALUEを各一引数、--、本体の絶対path、__internal、worker、--request、固定保存先の順。--scope/--pipe/--pty/--collect/--waitは付けず、SSH/stdin/端末の寿命へ結合しない。起動器のstdinは空、cwd=/。起動器とworkerが直接起動する外部コマンドは[Command::env_clear](https://doc.rust-lang.org/std/process/struct.Command.html#method.env_clear)後にLANG=C.UTF-8/LC_ALL=C.UTF-8と担当実行契約の明示項目だけを設定し、親の全環境を継承しない。

system managerがworkerへ渡す環境は別境界で、Environment/UnsetEnvironmentだけを「二変数だけ」の保証にしない。[systemd 255の環境構築](https://www.freedesktop.org/software/systemd/man/255/systemd.exec.html)へ照合する。固定環境の採用行に、manager/設定/PAM/自動付与の出所、許可する全変数名・値または照合式を固定する。LD_PRELOAD/LD_LIBRARY_PATH等の実行前に作用する変数は起動前のmanager設定審査で禁止し、worker起動後の削除で保護したことにしない。許可集合未定・設定変更・審査不能ならworker起動0件、起動後の実集合不一致ならbegin前停止。値をログへ列挙せず、不一致の分類だけを返す。実集合と固定版での成立性はD02-04の採用条件とする。

| ケースID | 固定入力 | 期待結果 |
|---|---|---|
| `A02-117` | managerに未審査LD_PRELOAD、通常子に親の代理設定、worker自動付与変数が採用行にない | 起動前拒否/子への非継承/begin前停止を区別。UnsetEnvironmentや起動後削除だけの安全認定なし |

| property | 候補値・条件 |
|---|---|
| KillMode / SendSIGKILL / Restart / Delegate | control-group / yes / no / no |
| RuntimeMaxSec / TimeoutStopSec | 親の通常処理残予算 / graceのms精度値。既定の二重加算をせず、終了記録予約も通常予算から除外 |
| WorkingDirectory / UMask / StandardInput | / / 0077 / null |
| OOMPolicy / MemoryAccounting / TasksAccounting | stop / yes / yes。OOMを正常終了へ変更しない |

実行者はsystemctl showの固定property集合Id/ActiveState/SubState/Result/MainPID/ControlGroup/ExecMainCode/ExecMainStatusを採取し、unit名とoperation記録を照合する。停止要求は同じunitへのsystemctl stop一回だけで、観測は再試行できてもstart/stopを重ねない。managerのjob受付、unitの終了、cgroup内全子孫の終了、実資源と必須記録を別に確認する。scope外に起動する管理対象サービスは担当StepReceiptで追跡し、worker停止だけでそのサービスの変更を巻き戻したとしない。

worker異常終了・OOM・期限到達・manager照会不能では保存済みrunning/intentを保全する。ロックがOSにより解放されても、残存unit/cgroupと未完了operationの照合が通らなければ次のbeginを禁止する。カーネルの中断不能I/Oや監督不能を時間内停止と保証せずstop_unknown、外側の制御器にも応答不能なら次回照合対象とする。systemd 255のAPI資料と、実際のUbuntu固定版/能力/停止試験は区別し、個別採用まで本候補を実行しない。

| ケースID | 固定入力 | 期待結果 |
|---|---|---|
| `A02-111` | SSH切断後もworker稼働、worker死亡後に子孫残存、同名unitが既存、OOM | Query/未完了照合/起動拒否/異常結果を区別。第二worker起動・lock解放だけの続行・期限保証の偽装なし |

### 割当失敗と測定の判定

P02-PARSEのMemoryLimitは論理計数Mの超過だけを意味する候補で、allocatorの全失敗を同じErrで回収できるという約束をしない。Vec/Stringを拡張する前はchecked計数とtry_reserveを行い、予約失敗をparse.memory_limitへ結合する。BTreeMap挿入・標準機能内部等の回復不能な割当失敗でプロセスが異常終了した場合は呼出元のworker異常として扱い、成功Valueを生成しない。解析完了・全validator通過前は対象への変更を開始しないため、解析失敗の後処理に復旧書込みを必要としない。

測定候補は採用CPU/ツールチェーンのrelease build、同じ入力byte列・深さ・ノード数を固定し、単一解析呼出しを一プロセスで実行する方式。通常/最大長/最大深さ/最大ノード/Unicode escape/重複キー/長い共通prefixを独立に測る。記録する値はM、入口から返却までの単調経過、最大RSS、stack設定、終了/シグナル。外側のRSSや時間をMへ足して別の「32 MiB」と呼ばない。成功要件は契約内の同じ値/エラー/位置であり、実RSSの製品目標値は未採用の非機能要求として別審査する。

CLIの出力は整形完了前のstdout書込み0回を維持する。出力writerが戻らない場合は制御が戻る硬い期限を標準Write APIの契約に加えず、外側が観測できた場合だけCLI_OUTPUT/送達不明を記録する。既に対象操作が確定している場合もその記録を保持する。同期I/Oのリアルタイム保証を必要とする要求は未成立としてD02-02/04へ残し、thread化やOS timeout設定だけで解消したとしない。

| ケースID | 固定入力 | 期待結果 |
|---|---|---|
| `A02-112` | try_reserve失敗、allocator abort、返却しないwriter、論理Mは上限内だがRSS大 | 解析エラー/worker異常/送達不明/別測定値を区別。部分値・対象変更・32 MiB実保証の捏造なし |

### bundle生成adapterの比較契約

OS方式の比較候補をGNU tar→GNU gzipの二段へ固定する。入力は秘匿化済みContentBlobをコピーして固定した専用stagingだけで、各member名はASCII採取IDと.json/.txt/.ymlのいずれか、100bytes以内・先頭hyphen禁止とする。元設定のdirectoryを再帰アーカイブしない。環境はLANG=C/LC_ALL=C/TZ=UTCだけでTAR_OPTIONS/GZIPを継承せず、cwdは当該staging、stdinは空。実行物の正確な版は例外台帳で未選定のまま。

| 工程 | argvテンプレート・成功判定候補 |
|---|---|
| tar生成 | tarの固定path、--create、--format=ustar、--blocking-factor=1、--numeric-owner、--owner=0、--group=0、--mtime=@0、--file=固定tar一時path、--、検証済みmember名の昇順列。通常file以外をstagingへ登録しない |
| gzip生成 | gzipの固定path、--no-name、-6、--stdout、--、固定tar一時path。stdoutは専用file-sink、stderrは通常capture、最終17 MiB以下 |
| 展開検査 | gzipの固定path、--decompress、--stdout、--、固定gzip一時path。通常stdout captureへ流さず、非公開検証用sinkへ上限付き出力。生成前tarの実byte列とも照合 |
| member検査 | 512byteのustar header/サイズ/型/終端を照合し、固定member集合とhash/実量を一対一照合。通常file以外・リンク・prefix逸脱・拡張entry・余剰非zeroを拒否。実FSへのtar抽出なし |

展開量の上限は合計file実量+各header/padding+終端1,024bytesのchecked和とし、無制限展開しない。tar/gzip各終了0だけではarchive検証成功としない。[RFC 1952](https://www.rfc-editor.org/rfc/rfc1952)は複数memberを許すため、gzipコマンドの展開成功だけでは候補の「一stream」条件を証明できない。一memberの境界確認は次の内製純粋検査器を第一候補とし、単純なmagic検索・gzip -t・展開結果一致だけで判定しない。

```rust
pub struct BundleGzipProof { compressed_bytes: u64, tar_bytes: u64, crc32: u32 }
pub fn verify_bundle_gzip(compressed: &[u8], expected_tar: &[u8], deadline: std::time::Instant)
    -> Result<BundleGzipProof, ContractError>;
```

本検査はbundle専用で、汎用gzipの全header拡張を受理しない。入力は17 MiB以下、headerはID1/ID2=1f/8b、CM=8、FLG=0、MTIME=0、XFL=0、OS=3の10bytesへ限定する生成profile案とする。採用gzipがこのprofileを生成することは別途固定版で確認する。FNAME/FEXTRA/FCOMMENT/FHCRC・reserved flag付きはBUNDLE_ARCHIVE。

DEFLATE本体は[RFC 1951の3.1.1/3.2.3〜3.2.7](https://www.rfc-editor.org/rfc/rfc1951)のbit順、stored/fixed/dynamicの全三blockを復号する。reserved BTYPE、LEN/NLEN不一致、不正Huffman表/参照、予約symbol、出力済み範囲を超すdistance、終端欠損を拒否する。expected_tarも17 MiB以下かつ算定済みtar上限以下を要求し、各出力byteを同位置で照合して一byteも超えない。BFINAL=1のstored blockは宣言LEN分の終了、fixed/dynamic blockはend-of-block symbolの終了をもって最終block終端とする。その後だけbyte境界へ進み、未使用padding bitは本生成profileでは0を要求する。続く8bytesを唯一のCRC32/ISIZE trailerとしてlittle-endianで読む。CRC32はRFC 1952の多項式/初期値/反転、ISIZEは実展開長mod 2^32へ一致させ、その直後が入力EOFでなければ空member・zero paddingも含め拒否する。

内製復号は32 KiBの参照windowと固定Huffman表上限を使い、処理中4,096入力/出力bytesごとと返却前に呼出元から渡したbundleの同じ30秒deadlineを照合する。関数受付から新たな30秒を作らない。短い入力、演算overflow、tar差異、期限超過はBUNDLE_ARCHIVEで証拠生成なし。proofは生成tar照合/ustar検査/保存確定の代わりではない。第三者復号crate・サンプル実装の持込みは行わず、仕様から内製する。固定gzipとの相互fixtureと独立CRC確認は採用審査に残す。

| ケースID | 固定入力 | 期待結果 |
|---|---|---|
| `A02-122` | final stored block、expected_tar17 MiB+1、受付時点で親deadline到達 | 正常storedはLEN終端で検証、他は開始前/期限拒否。存在しないEOB要求・無制限比較入力・予算の再起算なし |

| ケースID | 固定入力 | 期待結果 |
|---|---|---|
| `A02-118` | stored/fixed/dynamic各正常、CRC/ISIZE不一致、空第二member、非zero padding、後方参照範囲外 | 正常一memberだけproof。残りはBUNDLE_ARCHIVE、FS展開・余剰の無視・外部復号成功だけの認定なし |

| ケースID | 固定入力 | 期待結果 |
|---|---|---|
| `A02-113` | TAR_OPTIONS継承、memberがhyphen開始、同内容になる空の第二gzip member | 入力/環境拒否または一stream確認未達。通常fileへ任意抽出・終了0だけのBUNDLE成功なし |

### コマンドごとの検査閉包

担当検査の閉包は、各PlanStepのcontractとcheck. stepのcheck_ids/expected_refs/execution_refsを同じ承認済み検査登録へ照合して求める候補。検査登録はid/source/適用条件/期待資料/採取契約を持つ固定集合で、参照だけの契約に検査が未定義ならunknownを残す。契約IDだけから架空の検査名を生成しない。参照の巡回はP01の六値閉包で止め、検査の実行依存循環は拒否する。A02-108/114では未知check、欠けた担当登録、同IDで異なる必須性を注入し、応答生成前の拒否を照合する。

必須検査算定器の型付き境界候補を次へ閉じる。commandは共通CLIまたは媒体/マスターが検証した内部操作名だけ、state/profileは検証済み状態結果、plan_contractsはPlanの参照集合を借用する。状態不明を「未管理」へ変換しない。

```rust
pub enum CheckApplicability { Applicable, NotApplicable(ContractRef), Unknown }
pub struct CheckRequirement {
    id: String, required: bool, applicability: CheckApplicability,
    source: ContractRef,
}
pub fn required_checks(command: &str, state: &StateDecision,
    profile: Option<&Profile>, plan_contracts: &[ContractRef])
    -> Result<Vec<CheckRequirement>, ContractError>;
```

返却順はCheck.idのASCII順、同じID/同じ義務は一件、異なるrequired/適用根拠の衝突はCLI_RENDER。NotApplicableは正本の明示条件を参照し、Unknownは検査結果unknownを要求する。実行・観測・ファイル取得は行わない。A02-108で状態UNKNOWN/未構成確証/非適用根拠欠損を区別する。

beginの失敗はBeginFailureへ必ずID/段階/保存証拠を返す候補。last_confirmed_stageはUnknown以外、stage=Unknownなら最後に確認した段階を保持し、同期要求後の応答不明を前段階の未保存と断定しない。対象の実資源変更はcontext返却後だけなので、begin中の失敗で適用済みstepを捏造しない。呼出元は残存intent/runningを読み取り照合し、新規begin・削除で埋め合わせない。context成立後の実処理は既定のChangeResultを使い、BeginFailureを汎用エラーへ縮約しない。

| ケースID | 固定入力 | 期待結果 |
|---|---|---|
| `A02-124` | running同期要求で応答不明、intentだけ同期の確証、beginイベント失敗 | stage=Unknown/last=IntentSynced等の実証範囲を返す。実適用0件だが残存を保持、再begin・空の失敗結果なし |

必須検査を再算定する候補関数はrequired_checks(command, state, profile, plan_contracts)。入力は検証済み値だけで、結果はCheck.id/required/適用根拠の順序付き集合。状態をまだ判定できない段階では「未管理」を仮定せずunknownを要求する。共通八検査のIDは[既定の対応](#依存と受入の確定候補)を使い、下表は必要な部分集合と追加分を定義する。

| コマンド群 | 検査の期待集合 |
|---|---|
| inspect / status | OS・CPU・target.identity・records.consistent、条件付きprofile.selected、任意operation.clear。inspectの通信禁止はP02-CLI/inspect.local_onlyを必須追加 |
| doctor | 上記の共通対象集合とP04-SELF/installation.consistent、P02-PARSE/config.valid、P02-STORE/paths.protected、P01-CHECK/dependency.approved。未構成と確証のある設定不在だけconfig.validを根拠付き非適用 |
| doctor --transaction | doctor集合にP04-SELF/transaction.boundを追加。親guard/operation/候補hashの全対応が必要 |
| report | P02-STORE/records.consistent、P02-STORE/record.set。操作なしの確証がある場合だけrecord.setを非適用 |
| version / help | helpは空。versionはP02-MODEL/binary.identity、構成済みならP02-STORE/records.consistent。対象観測の未取得は個別診断へ保持 |
| plan / 変更操作 | 適用する共通八検査とPlan.contracts全件の担当検査閉包。依存/保全/署名/容量/復旧経路の担当検査を除外しない |
| verify | 対象の共通集合と、選択構成が要求するP04/P06/P07/P08の検査。未定義の個別能力行はunknownとして充足不可 |
| support-bundle | P02-STORE/records.consistent、P02-CLI/bundle.scope、bundle.redacted、bundle.archive、bundle.saved。dry-runはarchive/savedを未実施として予定表示し、成功済みにしない |

担当契約に複数の適用stepがある場合はCheckを重複生成せず、各stepの証拠を照合して一件へ集約する。全件passだけpass、一件failならfail、failなしで欠損/unknownならunknown、全件が明示非適用の場合だけnot_applicable。集約前の結果はStepReceiptへ保持する。必須集合に含まれない追加CheckもIDと担当根拠を必要とし、未知IDはCLI_RENDER。判定前に集合の過不足を検出し、空の配列だけでsuccessを構築しない。

| ケースID | 固定入力 | 期待結果 |
|---|---|---|
| `A02-108` | doctorのpaths.protected欠損、同じ契約の二stepがpass/fail、未構成証拠なしでprofile非適用 | 集合欠損/集約fail/unknownを区別。必須検査の縮小・重複ID・推測非適用なし |

### 再起動をまたぐ期限の照合

[Instant](https://doc.rust-lang.org/std/time/struct.Instant.html)の値を保存して別bootへ復元せず、[SystemTime](https://doc.rust-lang.org/std/time/struct.SystemTime.html)の差だけで経過を保証しない候補。操作開始前に以下の不変予算をoperations/OperationId/budget.jsonへ同期し、intentの必須集合へ追加する。StartEvidence.budgetは同じID/計画/元開始時刻へ束縛する。保存失敗ならbegin/資源変更を開始しない。上限は各親操作の担当契約から固定し、利用者が再開時に増やせない。

```rust
pub struct OperationBudget {
    schema_version: u32, operation_id: OperationId, plan_id: PlanId,
    started_at: UtcMillis, total_ms: u64, clock_contract: ContractRef,
}
pub struct ResumeBudgetEvidence {
    budget: OperationBudget, boot_id: String, observed_instant: std::time::Instant,
    elapsed_lower_ms: u64, elapsed_upper_ms: u64, verification_ref: String,
}
pub fn resume_deadline(budget: &OperationBudget, evidence: &ResumeBudgetEvidence)
    -> Result<std::time::Instant, ContractError>;
```

OperationBudgetの上記六キーだけを保存し、ResumeBudgetEvidenceは再収集した内部証拠で保存用Instant表現を設けない。clock_contractは停止/休止/電源断/時刻補正を含む元開始からの経過区間を保証する採用済み取得契約。収集器が同じ操作/対象と現在bootに対応する下限/上限を確認してから証拠を構築する。参照の存在や単なる時計の同期表示を保証にしない。純粋resume_deadlineは入力budgetと証拠内budgetの六値完全一致、0<=lower<=upper<totalを検査し、observed_instant+(total-upper)をchecked算定する。区間/対応/取得契約不明はMODEL_DEADLINE、再起動時点から全予算を再付与しない。

同一boot内も外側deadlineを延長せず、再開時は算定期限と呼出元期限の早い方を使用する。期限不明/超過は読取り照合だけとし、記録修復/未実施変更は開始しない。新しい復旧計画が必要なら別の明示選択/確認/操作として扱い、旧開始時刻を改変しない。採用できるclock_contractの実現根拠はD02-01/04と担当入口の未達に残す。A02-73/104に上限=total、区間逆転、電源断分不明、時刻後退を含める。

budget保存はguard下の開始準備で一回だけ行う候補。同じID/全byte列/hashに束縛された今回の予約と、既にintent/runningがある開始済みIDを区別する。予約存在だけを衝突や開始成功へ変換せず、別内容/出所不明の既存budgetはSTORE_CONFLICTで保持する。StartEvidenceのID集合が今回予約を除外する場合も、その対応根拠をverification_refへ必須とする。A02-123/124へ予約後中断と別要求の予算流用を含める。

### OS取得器の入力と停止

P02-MODELの乱数源の第一候補は、Linuxの/dev/randomから16bytesを一回分だけ取得する方式とする。[Linux random(4)](https://man7.org/linux/man-pages/man4/random.4.html)へ照合した候補で、初期化前urandomへのfallbackはしない。対象は採用済みLinux実行環境だけ。OpenOptionsExtのO_NOFOLLOW（Linux amd64/arm64では0x20000）を指定し、開いた実体がcharacter device、major=1/minor=8であることをMetadataExtのrdevから照合する。定数は当該OS ABIの確認対象で、他OSへ流用しない。symlink・通常file・別device・namespaceの確認不能はMODEL_RANDOM_SOURCE。

取得は変更前の観測workerで実施し、受付から5秒、停止予約1秒を候補とする。read_exactで短いread/Interruptedを同じ一回の16byte取得へ結合し、EOF・期限到達・OS読取り不能はMODEL_RANDOM。実際に停止を確認できなければstop_unknown、変更開始0件。乱数sourceが読めない場合に時刻/PID/既存IDから作らない。照合する既存集合は計画生成元の全保存計画/operation/history/remote/media/monitor IDで、MachineIdは対象記録と登録済み台帳を含む。集合未取得は失敗とし、衝突は既定の一回停止を維持する。

boot/PID/starttimeの採取も一組5秒内、各proc入力の量を先の上限へ制限し、読取り前後のboot_idを照合する。UTCはSystemTimeからUNIX_EPOCH以後の秒/ナノ秒を取得し、グレゴリオ暦の400年/100年/4年規則でUtcMillisへ変換する。範囲外・duration_since失敗はMODEL_CLOCK。壁時計が戻ってもelapsedの減算へ転用しない。OSのソース仕様を調べたことと、採用環境での停止/同一性が成立した実証は別とする。

| ケースID | 固定入力 | 期待結果 |
|---|---|---|
| `A02-109` | /dev/randomを通常file/別deviceへ置換、16bytes中の短いread/EOF、未取得の既存集合 | source拒否/同一取得の継続/取得失敗を区別。代替乱数・再抽選・変更開始なし |

| 判断ID | 採用前に必要な判断・調査 | 影響 |
|---|---|---|
| `D02-01` | [OS取得器](#os取得器の入力と停止)に乱数/時刻/boot取得の案あり。固定OS版の実行契約・失敗観測の成立根拠と採用判断が残る。MachineId復旧はD08-03 | 全対象照合 |
| `D02-02` | [CommandData全子型](#検査観測の固定集合)と[P08のマスター応答](PHASE_08.md#マスターcliの解析と応答全集合)に宣言/対応案あり。型・全検査集合の採用判断と同期writerの応答不能時境界は未確定 | 全CLI |
| `D02-03` | [型対応](#共通スキーマの型対応)と[割当判定](#割当失敗と測定の判定)に案あり。三構成への形式移行・全子型の閉包、実ヒープ/stackの測定条件と成立根拠が残る | 設定・状態・マニフェスト |
| `D02-04` | [worker起動](#変更workerの固定起動契約)に案あり。固定版での排他/監督・環境の全条件、停止不能I/Oと親停止後寿命の根拠が残る | 全変更・中断 |
| `D02-05` | [記録集合](#記録集合と保存順序)に保存順序・完了形式の案あり。FS別の容量/同期/障害照合の成立根拠と採用判断が残る | 全変更の確定 |
| `D02-06` | [bundle adapterと一member検査](#bundle生成adapterの比較契約)に全argv/復号境界/CRC/EOF案あり。固定tar/gzip版の生成profile、内製検査の相互fixture/予算と部分保存方式の採用判断が残る。HTTPSはD04-01 | 調査成果物 |

本書の内製候補は形式解析・対象判定・識別・記録調整・実行制御。モジュール新設案とし、初期からcrateへ抽出しない。難易度は[調査表](PHASE_01.md#標準機能の充足調査)へ対応付ける。旧実装は存在せず、差し替え時の共通ケースは上表を拡張して契約ごとに固定する。移行・抽出・旧コード除去の具体的変更は今回の対象外とする。

候補値の採用、公開型の完全な宣言、外部実行物の個別情報、非機能要求・試験条件の確定が残るため、Phase 2と依存範囲は完全仕様未成立。本文案の存在を実装着手可能の根拠にしない。
