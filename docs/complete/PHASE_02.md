# Phase 2 完全仕様案: 共通契約と構成選択

[文書一覧](../DOCUMENT_INDEX.md) | [担当工程](../ROADMAP.md#phase-2-共通契約と構成選択)

案版`B2`・採用未承認。[案の適用範囲](PHASE_01.md#案の適用範囲)に従う。既存の設定値・状態・終了コード・記録例は基本仕様が正本であり、本書では未定義部分の候補を扱う。共通型・CLI・解析・状態・記録・外部実行の具体化と受入設計を収録する。契約内容の採用や既存仕様の切替を含めない。

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

`P01-CLI`・`P01-CHECK`はB2の[Phase 1](PHASE_01.md#要件と境界)を参照する。Phase 2を使用する後続フェーズは必要な本書の契約IDと節を個別に参照し、共通値を再定義しない。

## P02-MODEL

採用案の共通型を以下に置く。配置候補は`src/core/model.rs`。検証済みの対象・識別・実行型は生成関数で検証し、フィールドを外部から直接変更させない。汎用の`Value`は構文上の値を表し、対象・設定・計画の検証済み型の代わりにしない。

| 型 | 内容・生成条件案 |
|---|---|
| `TargetEvidence` | 実行環境ID、対象ディスクの安定識別子、対象ルート、対象OS識別、観測時点。未取得値は`None`と理由を持ち、実行環境の値で補完しない |
| `PlanId`・`OperationId` | 接頭辞`plan-`・`op-`と32桁の小文字16進数。16バイトのOS乱数を取得し、一意性を確認。乱数取得・照合不能なら失敗 |
| `MachineId` | ServerSeed固有の32桁小文字16進数。構成初回確定時に生成し、OSのmachine-idとは別に保持。復元時の継承は`P04-BACKUP`で審査 |
| `ContractError` | `code: String`、`message: String`、`field: Option<String>`。コードは契約別の固定識別子、表示文は秘匿化済み。内部エラーをそのまま転記しない |
| `Value` | 宣言・数値の表現・所有権は[P02-PARSEの型とAPI](#解析の型とapi案)に集約。担当スキーマの検証前に適用可能とは扱わない |
| `ExecutionContext` | 借用する対象証拠・計画・操作、排他所有者、期限、取消状態。単なるJSONの読み込みだけでは生成しない |

時刻保存はUTCのRFC3339、秒未満3桁、末尾`Z`へ正規化する採用案とする。期限判定は経過時間と絶対期限を分離し、時計後退・再起動後の期限不明は有効と推測しない。乱数の取得手段・OS識別情報との結合は`D02-01`の調査対象であり、識別子生成を実装可能と判定しない。

### 共通参照型の候補

ContractRefの全キーはfile・anchor・contract_id・draft・commit・adopted。前5項目は空でないstring、adoptedはboolean。fileは文書索引が案内するリポジトリ相対パス、anchorはその文書の節、contract_idはその節の契約ID、draftは現在の案版、commitは参照を評価する確定ツリーの完全なコミットIDとする。採用審査用と採用済みの参照をadoptedで区別し、値だけで承認の存在を証明せず所有者の承認根拠と照合する。作成途中の文書には架空のコミットを埋めず、提出コミットができた時点の確認資料で参照集合を束縛する。

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
| RevisionRange | min・maxとも必須integer、0以上i64::MAX以下、min <= max。マニフェストが要求する各範囲をさらに照合 |
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

BinaryInfoの全6キーは既存の`serverseed_version: string`、`architecture: string`、`config_revision: integer`、`migration_revision: integer`、`compatible_config_revision: RevisionRange`、`compatible_migration_revision: RevisionRange`。revisionは0以上i64::MAX以下、architectureは既存の本体CPU値。AppliedInfoは`serverseed_version: string`、`config_revision: integer`、`migration_revision: integer`、`environment: string`、`profile: string`の全5キーを候補とし、構成値・経路は基本設定へ照合する。構成未作成または適用情報が検証不能ならapplied=nullと理由を分け、欠損revisionを0で埋めない。OS・kernelの読める版はobservationsへ置く。自己申告を署名検証済みの証拠にしない。

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
    Help { scope: String },
}
pub struct Invocation { request: Request, common: CommonOptions, operation: OperationOptions }
pub struct CliInputError {
    diagnostic: ContractError, argument_index: Option<usize>,
    scope: String, output_mode: OutputMode,
}
pub enum ExitClass { Success, General, Input, Blocked, Verification, Installation, Recovery, Reboot }
pub struct CommandResponse { envelope: Value, exit: ExitClass }
pub fn parse_args(args: &[std::ffi::OsString]) -> Result<Invocation, CliInputError>;
pub fn dispatch(input: &Invocation) -> Result<CommandResponse, ContractError>;
pub fn validate_response(response: &CommandResponse) -> Result<(), ContractError>;
pub fn render(response: &CommandResponse, json: bool) -> Result<Vec<u8>, ContractError>;
pub fn emit(bytes: &[u8], writer: &mut dyn std::io::Write) -> Result<(), ContractError>;
```

Invocationは入力を所有し、source等の不要なOptionはNone、self_onlyはrecover以外でfalse。indexは実行名を除いた0始まり。Runのplanは`plan`要求でtrue、dry-runの実効modeもplanとする。helpでは指定値を保持しても使わない。パーサー・render・validate_responseはファイル・通信・ログ・時計に触れず、emitだけが指定writerへ書く。dispatchは時刻・観測を取得し担当契約へ委譲、helpでは管理対象を読まない。時刻取得の実現方式はP02-MODELの未達に接続する。

CommandResponseの生成・検証は、上表のdata形と担当契約の承認済みスキーマを両方照合してから許可する。Valueのobject以外、矛盾するresult・ExitClass、非空errorsの成功、未定義payloadを拒否する。ExitClassは[結果表](#結果と終了区分案)の対応で数値化し、型名を製品状態に転用しない。dispatchの内部エラーも固定診断の応答に変換してrenderへ渡し、生の内部エラーを表示しない。開始後の実適用情報を失う変換は許可せず、保持方式未確定はP02-STORE・Phase 4へ残す。

内製の難易度候補は字句・診断整形が低、型検証・出力保全が中。stdにはServerSeed固有の文法・スキーマ・集約器がないため、この責務だけを内製する。将来のcrate抽出・差し替えでは同じ入力バイト列・応答・失敗writerを新旧へ与え、stdout・stderr・終了値・副作用を比較する。CLI・設定・保存形式の変更と旧コード除去は別の提示・承認が必要。今回は新設案だけで、新旧比較対象・切替・除去はない。

### 調査成果物案

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

既存候補`validate(value: &Value, schema_id: &str) -> Result<(), ContractError>`のschema_id全集合・版・定義構造・検証順序と、各形式の移行は`D02-03`に残る。今回そのAPIや三構成の設定形式を確定せず、構文解析の詳細化をスキーマ検証まで完成した根拠にしない。JSON生成の既存候補はobjectキーをUTF-8バイト順とする。その他の出力byte・丸め・escapeは[P02-CLI](#p02-cli)の未達へ対応付け、解析成功から補完しない。

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

## P02-STATE

`detect(target: &TargetEvidence, records: &Value) -> Result<Value, ContractError>`は[既存の判定順](../BASIC_SPEC.md#状態判定の優先順位)を変えず、採用案では各判定について根拠の出所・対象・未取得理由を返す。状態スナップショットを入力の事実より優先しない。

初回の構成選択結果は、利用者の対話結果、対象証拠、計画IDに結合する値としてメモリー上で保持する案。設定ファイルだけで選択成立とはしない。変更開始直前に再照合し、保存は最終構成検証後にP02-STOREへ渡す。保存された構成の値と三構成の適用範囲は[各正本](../POLICIES.md#構成選択と継続管理)へ参照する。

構成リビジョンの旧二択形式移行、初回選択結果を媒体・SSHへ渡す形式は`D02-03`と各入口の契約へ残す。既存の版番号を変更・互換認定しない。

## P02-STORE

### 排他と確定案

変更の親プロセスが既存の共通ロックを一度取得し、子処理は借用する`ExecutionContext`を使う案。`File::try_lock`を候補とし、競合時は待ち時間0で停止する。ロックファイルをunlink・renameして所有者をすり替えない。読み取り専用ではファイルもロックも作成しない。

プロセス識別にはPIDだけでなくOS起動識別子とプロセス開始情報を対応付ける案。子への引き継ぎ方式・子だけ残った場合のロック寿命は`D02-04`で確定するまで実装しない。

管理ファイルの置換は[原子性](../BASIC_SPEC.md#原子性と書き込み)に従い、親ディレクトリの同期と書き込み先の同一性確認も候補に含める。履歴は操作IDごとの重複を照合し、部分的な追記を成功記録としない。追記型履歴と複数ファイル確定の具体方式は`D02-05`で審査し、通常のrenameだけでトランザクション成立を保証しない。

### 記録とAPI案

`version`は既存の製品バージョン情報のJSONオブジェクト、`last-success`・`last-failure`は操作ID・終了時刻・結果・対象IDを持つJSONオブジェクトとする候補。現在状態の置換と製品の追記型実行履歴を分ける。管理権限・秘密情報は[セキュリティポリシー](../SECURITY_POLICY.md#ファイル権限)へ参照し、新たな権限規定を本書で作らない。

| 宣言案 | 制約 |
|---|---|
| `acquire(target: &TargetEvidence) -> Result<OperationGuard, ContractError>` | guardはClone不可、対象・所有プロセスと結合。取得失敗後の変更なし |
| `begin(guard: &OperationGuard, plan: &Value) -> Result<ExecutionContext, ContractError>` | 既存の操作状態を記録できた場合のみ返す |
| `finish(ctx: &mut ExecutionContext, result: &Value) -> Result<(), ContractError>` | 必須記録の確定に失敗したら成功を返さない。失敗記録も書けない場合はstderrと残存証拠を使い復旧待ち |

ログの採用案は、操作ID・対象ID・時刻・level・イベントID・秘匿化済みメッセージをJSONLへ記録する方式。保持・ローテーションは10 MiB×5世代を候補とするが、必須操作記録をログ整理で消去しない。履歴容量の保全・退避と操作上限の関係は`D02-05`の未決事項。

## P02-EXEC

配置案は`src/platform/command.rs`。`Command`の引数配列と`Instant`を再利用し、呼び出しごとに操作契約の制限値を必須入力とする。外部実行物の版・パス・引数・例外IDの照合なしに起動しない。

| 入力 | 採用案の型・範囲 |
|---|---|
| 実行物・引数 | 絶対パス、`Vec<OsString>`。シェル文字列を受けない。環境変数は必要な固定集合だけ |
| 時間 | timeout_msは1〜3,600,000、終了待ちgrace_msは1〜30,000。未指定・0・範囲外を拒否 |
| 出力 | stdout・stderr各1 MiBまで保持。並行して読み、超過時は失敗へ移る。末尾だけ捨てて成功にしない |
| 再試行 | 共通実行器は自動再試行0回。呼出元だけが回数・待機・再照合を定義できる |
| 取消 | 起動前は起動しない。起動後は対象契約の保全・停止へ移り、単に接続を閉じたことを終了とみなさない |

`run(spec: &Value, ctx: &ExecutionContext) -> Result<Value, ContractError>`は終了状態・出力バイト数・制限超過・経過時間・停止確認を返す案。標準入力の配送量上限、子孫停止の実現方式、ブロッキングI/Oの中断と親停止時の保全は未確定。`Child::kill`だけでは全条件を満たせず、新しい外部ツール・FFIを暗黙に追加しない。

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

識別子のparseは長さ・接頭辞・小文字hexを完全一致で確認し、空白除去や大文字変換をしない。生成側は候補と照合できる既存IDの集合を受け、衝突時は`MODEL_ID_COLLISION`として停止する。無限に乱数を引き直さず、乱数源の成立性・照合範囲が不足する限り`D02-01`は未解消。時刻の候補表現は`YYYY-MM-DDTHH:MM:SS.mmmZ`、年1970〜9999、有効な暦日、秒0〜59。過剰な小数桁を受理して丸めず、生成時だけナノ秒をミリ秒へ切り捨てる。経過時間はInstantで別に評価する。

### 記録確定と実行結果

`P02-STORE`には、記録候補を事前にすべて直列化・検査してから、同じ親操作の下で確定する方式を提案する。準備失敗は対象変更前に停止する。実資源の検証後に履歴を確定し、その操作IDを参照する終了記録を最後に確定する。履歴だけ確定・終了記録未確定の中断は、実適用状態と履歴の一致を復旧側が確認するまで成功としない。

追記型history.jsonlの論理的な追記を、既存バイト列と新しい完結行を含む一時ファイルの同期・renameで実現する候補とする。既存行の改変・並替え・欠落は禁止。同じ操作IDの同じ確定行は再追加せず、異なる結果なら競合として停止する。これで複数ファイル全体の原子性が成立したとはしない。全履歴のコピー容量、保持・退避、親ディレクトリ同期、クラッシュ時の照合手順は`D02-05`に残す。

| 型 | 全キー・値域の候補 |
|---|---|
| RecordReceipt | operation_id: OperationId、stage: prepared・history_committed・complete、record_paths: string[]、verified: boolean。completeと全必須保存の成功時だけverified=true |
| CommandOutcome | status: exited・spawn_failed・signaled・timed_out・cancelled・output_limit・io_failed・stop_unknown、exit_code: integerまたはnull、signal: integerまたはnull、stdout_bytes・stderr_bytes・elapsed_ms: 0以上integer、stopped: boolean |

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
```

EvidenceFieldの外部形式は[既存候補の3キー](#対象証拠と状態結果)へ一意に変換する。enumの内部表現を新しいJSON形式として流出させない。IDのparseは表記検査だけ、生成はOS乱数・衝突照合を伴う別工程。TargetEvidenceをCloneしても観測時点の証拠の複製にすぎず、再検査済みにはならない。

ChangeFailureは開始後の部分結果を消さない共通の返却規約候補。開始前だけpartial=Noneを許容し、実適用または開始記録後はSomeを必須とする。persisted_evidenceは実在を確認した記録参照だけで、保存不能時は空を許すがerrorへ記録失敗を保持する。プロセス強制終了でこの返却自体がない場合は永続記録と実状態の照合へ戻る。ExecutionContext・OperationGuardの実所有権とOS上の寿命は`D02-04`のまま別途確定が必要であり、宣言から推測しない。

## 状態と中断の詳細案

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

## 未決事項と成立判定

| 判断ID | 採用前に必要な判断・調査 | 影響 |
|---|---|---|
| `D02-01` | 識別子・時刻の正確な型、OS乱数・起動情報の取得と復元時の同一性 | 全変更の対象照合 |
| `D02-02` | [CLI詳細案](#p02-cli)の採用審査、担当検査・観測IDと必須集合、ネストする計画・記録の全スキーマ、時刻・実割当・出力停止時の上限と保存失敗の実証 | 全CLIと骨格。候補の記載を成立・採用済みとしない |
| `D02-03` | [解析の詳細案](#p02-parse)の採用審査、実ヒープ・stackの計測・割当失敗の保全と対象環境での上限実証、担当スキーマ全集合・版・検証順序と三構成の移行契約 | 全設定・状態・マニフェスト |
| `D02-04` | 子孫を含む停止・排他引継ぎ・親停止の実現可能性とOS外部契約 | 全実行・中断復旧 |
| `D02-05` | 複数記録と追記履歴の確定方式、保存形式全集合、保持容量・失敗時照合 | 全変更の成功判定 |
| `D02-06` | tar.gz・HTTPSの内製境界・正式仕様・互換試験。形式解析の候補はP02-PARSEを参照し、その実証不足はD02-03で管理 | 調査成果物・取得処理 |

本書の内製候補は形式解析・対象判定・識別・記録調整・実行制御。モジュール新設案とし、初期からcrateへ抽出しない。難易度は[調査表](PHASE_01.md#標準機能の充足調査)へ対応付ける。旧実装は存在せず、差し替え時の共通ケースは上表を拡張して契約ごとに固定する。移行・抽出・旧コード除去の具体的変更は今回の対象外とする。

候補値の採用、公開型の完全な宣言、外部実行物の個別情報、非機能要求・試験条件の確定が残るため、Phase 2と依存範囲は完全仕様未成立。本文案の存在を実装着手可能の根拠にしない。
