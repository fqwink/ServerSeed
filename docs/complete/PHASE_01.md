# Phase 1 完全仕様案: 開発・検証基盤とCLI骨格

[文書一覧](../DOCUMENT_INDEX.md) | [担当工程](../ROADMAP.md#phase-1-開発検証基盤とcli骨格)

## 案の適用範囲

本書およびPhase 2〜11の本文は、所有者から承認された「本文案の作成」に基づく採用審査用の案とする。現在の案版は全フェーズ`B7`。契約内容の採用承認・承認済み版はなし。作成作業の承認を、各候補の採用、仕様充足、実装着手または受入完了へ転用しない。案の対象はファイル・節・契約IDとGitコミットで識別し、承認後の版は別途所有者が確定する。

照合元は`9583140b7e539e8db2c382a73a88d8cb4a44246d`の基本仕様・方針・両ポリシー・構成別仕様とする。適用する憲章の最終変更コミットは`ece87b2fc4f7821e3723734c13d6b0e420b96cd8`。既存定義は各リンク先が正本であり、本案の候補を優先させない。本文中の「採用案」は採用された場合の動作を記述する。未決事項がある範囲は完全仕様未成立のままとする。

フェーズ共通の契約は担当ファイルで一度だけ具体化し、参照側では契約ID・節・上記の現在の案版を指定する。`B7`間の参照は設計上の接続であり、憲章第4条が要求する承認済み版への参照はまだ成立していない。基本仕様・構成別仕様の確定本文の移動・削除、製品契約の切替、基本要求の変更は今回行わない。フェーズ本文内の競合する候補定義は同じ担当契約へ集約し、別定義を残さない。

## 要件と境界

担当は`REQ-003`・`REQ-004`・`REQ-006`・`REQ-010`・`REQ-011`・`REQ-012`。要求の本文は[受入と追跡](../BASIC_SPEC.md#受入と追跡)を参照する。Phase 0は[文書整理工程](../ROADMAP.md#phase-0-基本仕様文書体系要件の対応整理)のまま維持し、別の製品実装フェーズにしない。

| 契約ID | 本書の担当 | 基本仕様の根拠 |
|---|---|---|
| `P01-ENV` | ビルド環境の照合と停止 | [ツールチェーン](../BASIC_SPEC.md#rustツールチェーン)、[CI環境](../BASIC_SPEC.md#ci実行環境) |
| `P01-CLI` | 引数解析・ヘルプ・未実装停止の骨格 | [CLI](../BASIC_SPEC.md#cli)、[起動応答](../BASIC_SPEC.md#起動時の応答) |
| `P01-CHECK` | 検査入口・対象範囲・結果の対応 | [仕様充足検査](../BASIC_SPEC.md#仕様充足の検査)、[依存検査](../BASIC_SPEC.md#依存関係の検査) |

共通JSONの未定義部分は[Phase 2](PHASE_02.md#p02-cli)の`P02-CLI`・B7を参照する。Phase 1の実装前にも、その使用部分の採用承認と仕様充足が必要となる。実装順序と契約の策定依存を混同せず、未定義のJSONを骨格実装で先に決めない。

## P01-ENV

採用案は、ローカルとCIで同じ検査入口を使用し、環境検査を全検査の最初に一度行う方式とする。採用Rust版・edition・最低対応版・ターゲットは基本仕様の正本を参照し、別の値を本書へ複製しない。

| 入力・条件 | 判定と出力案 |
|---|---|
| リポジトリルート、対象ターゲット、実行する検査名 | 三つとも必須。ルート外の作業ディレクトリを拒否 |
| `rustc -Vv`・`cargo -V`・rustfmt・Clippyの識別結果 | 固定版との不一致・取得不能は環境検査失敗。自動導入・自動更新なし |
| manifest・lockfile | 欠損、解析不能、禁止依存または解決差分で停止。検査中に再生成しない |
| 対象CPUの実行検証環境 | ビルド環境と分けて記録。クロスビルドだけなら実行試験は未実施 |

通常検査の開始からの上限は30分、外部コマンドの制御は`P02-EXEC`・B7へ接続する。検査のタイムアウトを試験成功へ変換しない。最終CI環境の固定参照・Actionは`EX-007`〜`EX-009`の個別採用条件が揃うまで使用可能としない。

P01-ENVが照合する固定環境の審査入力をEnvironmentBindingとして具体化する候補。全キーはexception_ids: string[]、image_digest: string、tool_versions: 実行物ID→版のobject、targets: string[]、approval_refs: string[]。例外IDから個別採用情報を参照し、版・digestを本書に複製しない。対象CPUの実行証拠は別に要求する。台帳の状態が未確定、同一実行物の欠損/二重登録、イメージ内の実版との不一致、承認参照不在のいずれかで環境検査を失敗にする。推奨は固定イメージ方式で、浮動runnerだけを採用根拠にする案は除外する。

環境照合の候補順序は、要求形式→commitの実在と検査ツリー一致→EnvironmentBinding→全実行物の版→要求targetの標準ライブラリ/リンカー→manifest/lock→実行用CPU環境とする。image_digestはsha256:と小文字hex64桁、各集合は非空・重複なし。要求targetがtargetsにない場合はCHECK_ENV。実行用CPUがない場合はビルド可否と実行不可を別結果にし、実行を要求したjobはnot_run、suite全体は未完了とする。クロスビルドの成功をenvironmentの全条件成功へ広げない。

文書専用検査ではtarget=documentを使い、Rustのtarget・manifest・lockfile・CPU実行環境を要求しない候補とする。検査器自身の実行環境は識別して記録し、製品をビルドした証拠とは分ける。purpose=document_draftとtarget=documentの組だけに適用し、implementation_gate/product_acceptanceでdocumentを指定したらCHECK_INPUT。EnvironmentBindingの各tool_versionsキーは個別採用情報の実行物IDへ一対一で対応させ、空の版・未列挙の呼出し先・浮動参照をCHECK_DEPENDENCYにする。

## P01-CLI

引数を`std::env::args_os`で取得し、サブコマンド名とオプション名をASCIIとして照合する採用案とする。パスの非UTF-8対応範囲は`P02-CLI`で審査する。OSや設定の検査より先に構文を検証し、ヘルプ表示だけで管理領域を読み書きしない。

1. 共通フラグ、サブコマンド、操作別引数は[P02-CLIの字句と検証順序](PHASE_02.md#字句と検証順序案)だけで解析する。骨格側に別の重複・値域検査順を持たせない。
2. helpの成立・必須入力との優先関係も同節を参照する。骨格は成功・エラーの分類と出力を受け取り、再解釈しない。
3. `version`は実行本体の組込み情報と、読める対象情報を分離して出す。配置前の本体にも`applied`を捏造しない。
4. 構文妥当でも機能未実装なら、基本仕様の未実装停止へ接続する。未実装の`inspect`・`doctor`を空の成功結果として返さない。
5. 終了区分と共通JSONの枠は[既存定義](../BASIC_SPEC.md#共通json出力)を使い、要素形式は`P02-CLI`の採用案で具体化する。

### 内部API案

配置案は`src/cli/args.rs`と`src/cli/dispatch.rs`。初期は同じCargoパッケージの内部モジュールとし、crate新設はしない。

| APIの担当 | 宣言・所有権・戻り値・失敗の定義先 |
|---|---|
| 引数解析の骨格 | [P02-CLIのparse_args](PHASE_02.md#cliの型とapi案) |
| 担当契約への振分け | [P02-CLIのdispatch](PHASE_02.md#cliの型とapi案) |

`Invocation`のコマンド別フィールドと受理順序は`P02-CLI`、自動化入口の追加構文は各担当フェーズの未決事項へ分離する。これらの型の宣言が未確定な間はAPI案も未充足とする。

骨格のdispatch候補は、妥当なHelpを純粋な表示へ渡し、RunはActionとplan分岐の組に対する実装登録を一度だけ照合する。未登録の組はCLI_UNIMPLEMENTEDで停止し、近いコマンドへ転送しない。plan=trueの未実装を実行済み計画へ、versionの未実装を固定の製品版文字列へ置換しない。登録済みの骨格と後続機能も同じCommandResponseの検証・render・emitを通すため、骨格独自の成功JSONは生成しない。

## P01-CHECK

検査名と実行入口は[CI/CD](../BASIC_SPEC.md#cicd)に従う。採用案では各入口が対象コミット、環境識別、対象契約、実行コマンド、成否、未実施・適用外の理由を返す。検査結果の保存先は実行者の明示指定とし、製品の管理ファイルを保存先に使わない。

`run-all`は選択した検査を順に実行し、最初の失敗で後続を未実施として列挙する。まだ実装していない検査入口は終了0の代用品を置かず、検査失敗として扱う。Phase 1で不要なISO生成・VM試験を実行済みと表示しない。全初期ジョブの成立確認は`P11-ACCEPT`へ接続する。

仕様充足検査の入力案は、対象要件・契約ID・担当節・承認済みコミット・適用項目・受入ケースの集合とする。空の対象、承認済み版の欠損、同一契約IDの重複、未知要件、未承認案への依存を失敗とする。文書案の構文検査と実装着手判定を別の検査目的として識別する。

依存宣言のTOML解析、Markdownの節解決と意味上の充足判定の自動化範囲は`D01-02`の未決事項。正規表現による文字列一致だけを依存グラフ・意味整合性の検査と認める案にはしない。

## データ契約の具体化案

`P01-CHECK`の検査要求を`SuiteRequest`、結果を`SuiteResult`として具体化する候補。表のキーはすべて必須、明記した箇所以外のnull、未知キー・型変換を拒否する。これは製品CLIのEnvelopeとは別の開発用形式で、製品状態・終了コードの定義には流用しない。

| 型・キー | 候補の型・制約 |
|---|---|
| SuiteRequest.schema_version | integer、1だけ |
| SuiteRequest.commit | 検査対象の完全なGitコミットID。対象ツリーとの一致が必須 |
| SuiteRequest.purpose | string、document_draft・implementation_gate・product_acceptanceのいずれか |
| SuiteRequest.target | 基本仕様のRustターゲットstring。文書検査だけはstringのdocument |
| SuiteRequest.jobs | string[]。既存CIジョブ名から重複なし・1件以上。run-allでは基本仕様のジョブ掲載順で実行 |
| SuiteRequest.contracts | ContractRef[]。1件以上・重複なし。実装・製品目的では採用済み参照だけ。定義は[P02-MODEL](PHASE_02.md#p02-model) |
| SuiteResult.schema_version・commit・purpose・target | 要求と同じ値。対象や目的の置換は禁止 |
| SuiteResult.jobs | JobResult[]。要求された全ジョブを順序どおり一度ずつ含む |
| SuiteResult.environment | JobResult。name=environmentの一件。環境・依存の事前検査を保持し、要求jobsには混ぜない |
| SuiteResult.complete | boolean。environmentと全必須ジョブがpassのときだけtrue |
| JobResult | 全キーはname: string、status: pass・fail・not_run、commands: string[][]、evidence: string[]、reason: stringまたはnull |

JobResultのcommandsは実行物と引数の配列を実行順に保持し、再実行可能な記述とする。未実行はcommands・evidenceを空配列、reasonを必須とする。passはreason=null、commandsと検査対象・実結果へのevidenceがそれぞれ1件以上必要で、空の成功を認めない。failも失敗位置へのevidenceとreasonを必須とする。証拠は所有者管理下の相対参照であり、秘密値や外部ログ本文を格納しない。reasonの固定値は`CHECK_ENV`・`CHECK_INPUT`・`CHECK_DEPENDENCY`・`CHECK_EXECUTION`・`CHECK_EVIDENCE`とする。

ジョブ要求の候補集合は基本仕様のジョブ名だけとし、environmentをjobsへ指定する入力はCHECK_INPUT。個別入口は対応する一ジョブ、run-allは指定集合を基本仕様の掲載順へ並べてから実行する。結果のnameはその要求順と完全一致させる。environmentがnot_runの場合も全要求ジョブをnot_runにし、complete=false。failのcommandsが空でも、起動前検査の失敗証拠があれば受理するが、未実行の外部コマンドをcommandsへ記載しない。

検証順序は要求形式、対象コミット、検査目的、環境、依存、各ジョブ、結果集合の完全性。環境失敗なら全ジョブをnot_runとし、環境検査自体の失敗をreasonへ対応付ける。各ジョブの予算と親の残り予算の小さい方を使用する案で、最初のfail後は後続をnot_runとする。集計器の宣言は[型付きAPI](#型とapiの詳細案)に集約する。検査の実行・保存を行わない純粋処理とする。

TOML・Markdownの内製解析範囲、全検査項目の機械判定可能範囲、実行ラッパーへの引数受渡しは`D01-02`に残る。このデータ契約だけで検査器を実装可能としない。

## 標準機能の充足調査

確認日は2026-10-19。以下は[採用版のRust標準ライブラリ公式資料](https://doc.rust-lang.org/std/index.html)に対するAPI資料の確認であり、ホスト上でのコンパイル・両CPUでの実行結果ではない。標準ライブラリ以外の追加依存を選定・使用した根拠にはしない。

| 確認対象 | 確認できた範囲と不足 | 担当案 |
|---|---|---|
| 引数・コレクション・文字列 | `env`・`collections`・文字列の基本処理を再利用。製品固有のCLI文法・診断は別に必要 | `P01-CLI`、低難易度 |
| 構造化形式 | 公式モジュール一覧にYAML・JSON・TOMLの専用パーサーはない。ファイルI/Oだけでは構文・スキーマ検証を満たさない | `P02-PARSE`、制限付き形式でも中難易度。完全互換は高難易度 |
| 外部起動 | [Command](https://doc.rust-lang.org/std/process/struct.Command.html)の引数渡しを再利用。製品の出力上限・時間制限・子孫管理は別契約 | `P02-EXEC`、高難易度 |
| 子の終了 | [Child](https://doc.rust-lang.org/std/process/struct.Child.html)の終了照合を利用候補とする。単一の子の強制終了だけでは子孫・保全・排他契約を満たさない | `D02-04`で停止方式の成立性を審査 |
| ファイル | [File](https://doc.rust-lang.org/std/fs/struct.File.html)の新規作成・同期・非待機ロックは利用候補。対象ルートの安全な解決・複数記録の確定は別に必要 | `P02-STORE`・`P03-FS`、中〜高難易度 |
| 経過時間 | [Instant](https://doc.rust-lang.org/std/time/struct.Instant.html)を制限判定に使用する案。保存時刻との相互代用はしない | `P02-MODEL` |
| SSH・TLS・暗号・圧縮 | `net`等の基本APIだけで各プロトコル・検証済み形式を充足したとは認定しない | `D02-06`、`D04-02`、`D09-01`の調査・例外境界へ対応 |

既存内製コードは現在存在しないため、再利用実装の評価対象なし。再利用するのは本案内の共通契約であり、実装済み機能とは区別する。JSON・YAMLの資料照合は[P02-PARSE・B7](PHASE_02.md#解析の範囲と一次資料)、限定TOML・Markdownの資料対応は[入力集合](#入力と保存形式の詳細案)を参照する。資料対応の記載と、採用版での境界ケースの実行試験を区別し、後者は未実施とする。

## 接続と責務の照合

`P01-CLI`のparse_args・dispatchの宣言と所有権は[P02-CLIのAPI](PHASE_02.md#cliの型とapi案)を具体化先とし、骨格側はその受入部分を使用する。引数上限・UTF-8・help優先順を骨格用の別ルールへ再定義しない。案版・コミットが不一致なら、実装順序を理由に先行採用しない。

SuiteResultは[P11-ACCEPT](PHASE_11.md#データ契約の具体化案)の証拠候補にできるが、収集した[VerifiedEvidence](PHASE_11.md#入力と保存形式の詳細案)との照合を必要とする。purpose=document_draftの成功をproduct_acceptanceの実出力へ変換しない。文書形式の成功と意味上の充足を区別し、機械判定できない項目は所有者確認待ちとして残す。後続フェーズの未実装ジョブをrun-allから黙って消さず、要求範囲とnot_runの理由を保持する。

検査の親上限には環境確認・子コマンド・結果集計を含める候補。ISOの生成上限より短い検査要求を受けた場合、親の残り予算を優先して停止し、ISO契約自体の上限や公開条件を緩めない。Phase 1の骨格検査とPhase 11の全ジョブ受入は要求範囲が異なり、片方の成功で他方を完了にしない。

## 受入ケース案

| ケースID | 区分・前提と入力 | 期待結果・確認方法 |
|---|---|---|
| `A01-01` | 正常。管理領域のない環境で`--help` | ヘルプ成功。実行前後のファイル一覧・内容が不変 |
| `A01-02` | 異常。未知オプション、必須値なし、引数なし | 基本仕様の入力エラー。変更処理の呼出し0回 |
| `A01-03` | 境界。同じオプションを2回指定 | 重複エラー候補。後勝ち・先勝ちで黙認しない |
| `A01-04` | 異常。妥当な未実装コマンドと`--json` | 未実装停止。JSONの枠・終了区分が既存契約へ一致 |
| `A01-05` | 正常。配置前の`version --json` | 本体情報と未適用情報を区別。OSや構成の偽装なし |
| `A01-06` | 異常。固定版不一致、lockfile欠損、禁止依存 | 環境・依存検査失敗。取得・更新・自動修復なし |
| `A01-07` | 境界。検査上限到達、検査入口欠損 | 失敗と未実施を識別。後続成功の捏造なし |
| `A01-08` | 異常。B7契約だけを根拠に実装着手判定を要求 | 承認済み版なしとして失敗。文書検査成功と分離 |

## 追加の受入ケース案

`P01-CHECK`の形式・目的・集計に対する設計。入力に妥当なコミット・環境の固定fixtureを使用し、実装やCIを今回実行した結果ではない。

| ケースID | 区分・固定する前提と入力 | 期待結果・確認方法 |
|---|---|---|
| `A01-09` | 正常。jobs=[format,lint]、両JobResultがpassで証拠あり | 要求順の2行、complete=true。製品の受入結果は生成しない |
| `A01-10` | 異常。format=fail、lintをpassとして入力 | 後続未実施という規則への不一致を検出。架空のlint成功を採用しない |
| `A01-11` | 異常。document_draftの結果をproduct_acceptanceへ結合 | 目的不一致で拒否。試験結果の成功へ転用しない |
| `A01-12` | 境界。jobsが0件・同名2件、結果の欠損・余分な行 | CHECK_INPUTまたはCHECK_EVIDENCE。空集計・部分集計をcompleteにしない |
| `A01-13` | 異常。対象コミットは同じ、検査後に環境または成果物を変更 | 証拠適用範囲の不一致。前の結果を再利用せず該当検査を要求 |

## 型とAPIの詳細案

`P01-CHECK`のValueによる未検証入力と、検証後の検査要求・結果を分ける候補。以下は宣言案で、実行体ではない。共通のContractRef・ContractErrorは`P02-MODEL`を使用する。

```rust
pub enum CheckPurpose { DocumentDraft, ImplementationGate, ProductAcceptance }
pub enum CheckTarget { Document, RustTarget(String) }
pub enum JobStatus { Pass, Fail, NotRun }
pub struct SuiteRequest {
    schema_version: u32, commit: String, purpose: CheckPurpose,
    target: CheckTarget, jobs: Vec<String>, contracts: Vec<ContractRef>,
}
pub struct JobResult {
    name: String, status: JobStatus, commands: Vec<Vec<String>>,
    evidence: Vec<String>, reason: Option<String>,
}
pub struct SuiteResult {
    schema_version: u32, commit: String, purpose: CheckPurpose,
    target: CheckTarget, environment: JobResult, jobs: Vec<JobResult>, complete: bool,
}
pub fn validate_suite(input: &Value) -> Result<SuiteRequest, ContractError>;
pub fn aggregate_suite(input: &SuiteRequest, environment: JobResult, jobs: Vec<JobResult>)
    -> Result<SuiteResult, ContractError>;
```

validate_suiteは形式と組合せを検証し、ファイル・Git・環境を読まない。commitの実在・ツリー一致は実行前の環境検査で照合し、形式の成功を実在確認にしない。aggregate_suiteはenvironmentとjobsを所有し、環境の一件性・先行成功と、要求jobsの順序・件数・証拠の対応を検証する。環境失敗時のjobsはすべてnot_runとし、環境の成功を要求jobの結果へ複製しない。入力と異なるpurpose/targetへ変更するAPIを設けない。フィールドは非公開で、検証なしの構築・completeだけの書換えを許可しない候補とする。

## 状態と中断の詳細案

`P01-CHECK`の検査実行について、実行器の内部位置を次の候補へ固定する。JobStatusとは別の工程位置であり、製品状態名ではない。

| 現在位置・事象 | 次の位置・結果 |
|---|---|
| received、要求不正 | rejected。ジョブ実行0件、CHECK_INPUT |
| received、要求形式が妥当 | checking_environment。まだジョブを実行しない |
| checking_environment、環境・依存不成立 | stopped。全ジョブnot_run、失敗根拠を保持 |
| checking_environment成功 | running。要求順の最初のジョブだけ開始 |
| running、ジョブpass | 次の要求ジョブへ。最終ジョブ後だけaggregating |
| running、fail・取消・期限到達 | stopped。実行中ジョブの失敗位置を保持、後続not_run |
| aggregating、証拠・集合が一致 | finished。全件passのときだけcomplete=true |
| aggregating、欠損・不一致・出力失敗 | stopped。成功扱いせずCHECK_EVIDENCE |

どの位置からのプロセス強制終了でも自動再開はしない。再実行時は新しい検査要求・環境照合を必要とし、前回の途中出力を最終SuiteResultとして使用しない。検査出力の保存失敗と製品ファイルの復旧を混同せず、再試行のための製品状態変更は0件。対象コミット・環境が変わった証拠は新しい検査へ自動再利用しない候補とする。

## 依存と互換性の詳細案

`P01-ENV`・`P01-CHECK`の依存候補を次へ限定する。検査用Ruby等をこの文書審査で使うことと、将来の製品・CI実行体の採用は別である。

| 責務 | 再利用・内製・外部境界 | 差し替え前の照合 |
|---|---|---|
| CLI骨格 | P02-CLIの同一parse_args/dispatch | 引数列・正規化名・診断・終了区分・非変更性 |
| 検査要求・集計 | 本書のvalidator/aggregatorを内製 | purpose・target・jobs・証拠の完全一致 |
| TOMLとMarkdownの解析 | Cargo manifestの全依存区分・見出し解決を内製する候補 | 未対応構文は検査失敗。コメント・無効featureで禁止依存を隠せないこと |
| ビルドと静的検査 | EX-007の採用版ツールを使用 | 版・target・lockfile・終了結果を固定 |
| CI・VM・ISO実行基盤 | EX-008・EX-009の個別情報が必須 | 実行環境・権限・資材・公開範囲の一致 |

Markdown/TOMLの全標準互換を宣言せず、実際に管理する構文集合・不正時出力・fixturesを`D01-02`で確定する。自動検査の意味判定限界は所有者確認へ残すが、機械判定できないことを成功と表記する方式は採用しない。初期は同一パッケージ内に置き、crate抽出時はSuiteRequest/ResultとP02-CLIの双方を同じ契約テストで比較し、呼出元・lockfile切替と旧コード除去を段階分離する候補とする。

検査器の結果をScanResult候補へ閉じる。全キーはentries・diagnostics・unresolvedで、entriesは検証済み依存辺または解決済み文書参照の配列、diagnosticsはcode/path/byte_offsetの配列、unresolvedは機械判定対象外のpath/理由の配列。空のunresolvedだけを自動合格の必要条件とし、非空なら所有者確認待ちでcomplete=false。例えば未対応の複数行TOML文字列を含むmanifestはCHECK_INPUT、`[target.'cfg(unix)'.build-dependencies]`のgit宣言はCHECK_DEPENDENCY、閉じたfence内の架空リンクは文書参照集合へ加えない。限定言語の残る全字句・escapeの確定はD01-02に維持し、この三例だけで文法完成とはしない。

ScanResultの安定順候補はentriesを入力path・対象ID・参照先、diagnosticsをpath・byte_offset・code、unresolvedをpath・理由のUTF-8バイト順とする。重複は同じ全フィールドだけを一件へまとめる。diagnosticsが非空でもcomplete=falseとし、unresolvedが空という条件だけで構文不正や禁止依存を成功にしない。異なる入力位置の違反は消さず、先頭の正常な参照だけを全体結果として返さない。

純粋検査器へ渡すファイル集合の候補キーは、区切りを/にしたUTF-8のリポジトリ相対path。空・絶対path・空成分・.・..・制御文字・同じ正規形の重複をCHECK_INPUTで拒否する。本文は元byte列を保持し、byte_offsetは当該本文の0始まり位置、EOFは本文長。収集器が別に得た実体/リンク/所有元の証拠を依存検査へ渡し、map内のpathだけでpath依存の実体が安全とは判定しない。欠損本文を空文字へ補完せず、検査器からファイル取得を開始しない。

## 受入条件の充足照合案

P01-ENVは固定環境・実行/クロスビルドの区分、P01-CLIはP02-CLIの入力と出力、P01-CHECKはSuiteRequest/Result・実行位置・依存表へ対応付ける。下記は追加の受入設計で、製品試験の成功記録ではない。

| ケースID | 区分・固定入力 | 期待結果と確認 |
|---|---|---|
| `A01-14` | 異常。正しい長さの架空commit、形式検査だけ成功 | 環境検査で実在/対象ツリー不一致を検出。ジョブ実行0件、全not_run |
| `A01-15` | 異常。jobsが2件、1件目実行中に取消 | 実行中の停止位置と失敗証拠を保持、2件目not_run。部分SuiteResultをcompleteにしない |
| `A01-16` | 境界。ジョブは全passだが証拠1件のcommitだけ相違 | aggregatingで停止、CHECK_EVIDENCE。以前の結果の自動再利用なし |

仕様充足の未達はCLI/型の採用、TOML・Markdown対応構文/検査項目、標準/外部実行基盤の個別採用条件。要求・型・工程・例外・受入を結び付けたことと、それらの採用済み版があることを分ける。Phase 1だけ先に実装する場合もP02-CLIの使用契約の成立が必要で、先行フェーズという番号を免除に使わない。

## 入力と保存形式の詳細案

P01-CHECKの解析器は次の限定言語を採用候補とする。未対応を無視するのではなくCHECK_INPUTで停止し、依存なしという結果に変換しない。

| 入力 | 受理する集合・拒否条件 |
|---|---|
| Cargo manifest | UTF-8、1 MiB以内。bare/quoted key、dotted key、table、array-of-table、string、bool、integer、array、inline tableとコメントを解析する。多重定義、再定義、浮動小数・日時・未対応escapeは拒否 |
| 依存の探索 | dependencies・dev-dependencies・build-dependencies、target配下の同名区分、workspace.dependencies、workspace members/exclude、patch・replaceと各依存のworkspace継承を評価。optionalや未選択targetも検査対象 |
| 依存解決 | workspace memberのpath参照を正規化してリポジトリ内へ限定。循環・欠損・registry/git/source指定は拒否。Cargo.lockのpackage/source/checksumと全memberを対応付ける |
| Markdown | UTF-8のATX見出し、閉じたbacktick/tilde fence、inline link、reference linkと定義を解析。fence内をリンクとして解釈しない。HTML・見出し内の複雑なinline構文は自動判定対象外として報告 |
| 節参照 | 見出しの表示文字を小文字化し、空白をhyphenへ変換、句読点を除き重複を出現順に-1以降で識別。生成結果と実際のGitHub節参照が異なる場合はCHECK_EVIDENCE。対応不能を成功にしない |

SuiteRequest.commitは40桁の小文字hexを候補とし、SHA-256形式のGitリポジトリは未対応で拒否する。結果のevidenceは所有者指定の証拠ルートに対する相対参照だけとし、絶対パス・親への逸脱・外部URL・symlinkを拒否する。参照先のdigestと対象commitを照合する情報はPhase 11のEvidenceRefへ結合し、単なる文字列の存在でpassにしない。

限定TOMLの字句候補は[TOML 1.0.0](https://toml.io/en/v1.0.0)の該当生成規則を参照する。bare/単行basic/単行literal key、単行basic/literal string、bool、符号付き64bit整数、array、inline table、table、array-of-tableだけを受理し、整数の基数・underscore・escape・dotted keyの意味は同版へ一致させる。float・日時・複数行stringはCHECK_INPUT。BOM・不正UTF-8・多重定義・不正escapeを拒否する。arrayの改行/末尾commaは同版どおり、inline table内の改行/末尾commaは拒否する。暗黙tableと明示tableの結合も同版の再定義条件で評価し、文字列検索で依存節を探さない。文書上限・深さ・要素数・文字列長・時間/論理予算はP02-PARSEを再利用し、TOMLの専用検査器がJSON/YAMLの受理範囲を変更することはない。

Markdown候補は[GFM](https://github.github.com/gfm/)のblock/inline構造を先に識別する。fenceは先頭0〜3space、backtickまたはtildeが3個以上、終端は同じ文字で開始長以上とし、閉じていないfenceをCHECK_INPUTにする。code span・fence内のリンク文字列は参照外。リンク先の括弧・escape・reference labelの対応は同仕様の規則を使い、raw HTML・独自アンカー・解釈不能な見出しをunresolvedへ残す。参照pathは文書所在から解決し、percent escapeの不正・リポジトリ外への逸脱を拒否する。GitHubの節IDはGFM文法そのものとは別のホスティング契約として照合し、Unicodeや重複見出しの未確認を推測で通さない。

| ケースID | 固定入力 | 期待結果 |
|---|---|---|
| `A01-17` | 未選択targetのbuild-dependenciesにgit依存、reference linkの参照先欠損 | それぞれCHECK_DEPENDENCY/CHECK_INPUT。通常targetだけの検査・リンク無視による成功なし |

## 実行と復旧の確定候補

P01-CHECKの30分予算は要求受付時に開始し、環境確認・解析・ジョブ・証拠保存・集計を含める。各ジョブの開始前に親の残期限からP02-EXECの停止待ちと結果保存の予算を予約し、残量不足ならnot_runで停止する候補。ジョブ自身が指定した上限を親の追加時間として扱わない。

保存手順は明示された出力先と同じFSにcreate_newで一時物を作り、全結果を検証・同期してから新規の最終名へ確定する。既存結果を上書きしない。途中出力はcomplete=falseの診断資料として区別し、正式SuiteResultへ昇格しない。保存操作もP02-STOREの対象同一性・同期契約へ従い、通常のrenameに上書き防止を期待しない。

再実行では同じcommitでもrun単位の新しい証拠を生成する。前回のpassをコピーして後続だけ実行する方式は初期対象外とする。取消後にジョブの子孫停止を確認できなければCHECK_EXECUTION、後続は全not_run。操作終了と証拠の保存完了を別に確認する。

| ケースID | 固定入力 | 期待結果 |
|---|---|---|
| `A01-18` | 最終ジョブpass後に証拠保存失敗、同名最終ファイルが先に出現 | finished/completeとして提出しない。既存ファイル不変、同ジョブの自動再実行なし |

## 依存と受入の確定候補

P01-CHECKの内製境界をdependency_scanとdocument_scanの二つの純粋処理へ分ける候補。前者はmanifest/lockの解析木と取得済みファイル集合だけを受け、後者はMarkdown全文と索引だけを受ける。どちらもCargo実行・ネットワーク・ファイル修正を行わない。全標準対応ではなく[入力集合](#入力と保存形式の詳細案)への完全一致を契約にする。

| 内製処理 | 受渡し・失敗・差し替え時の比較 |
|---|---|
| dependency_scan | 入力は相対path→UTF-8本文のmap、結果はmember/依存辺/禁止根拠の集合。構文未対応はCHECK_INPUT、辺やsource不一致はCHECK_DEPENDENCY。pathと理由を安定順で比較 |
| document_scan | 入力は同mapと文書索引path、結果は文書/見出し/解決済み参照/機械判定対象外項目。未解決はCHECK_INPUT、機械判定対象外を自動充足にしない |
| 実行wrapper | P02-EXECだけを使用。環境指紋とコマンド列/終了結果をJobResultへ結合し、純粋検査器の成功をジョブ実行に変換しない |

初期は同一package内の別module。抽出候補は上記二処理のみで、CLI・OS実行器をまとめてcrate化しない。抽出の受入では同じfixturesに対する辺集合/順序/失敗位置/出力を完全一致させ、全呼出元を切り替えた後だけ旧moduleを除去する。manifest/lockの変化は別の承認対象とし、旧実装が存在しない今回に比較成功を記録しない。

| ケースID | 固定入力 | 期待結果 |
|---|---|---|
| `A01-19` | 同じ依存木を異なるtable順で入力、未対応構文をoptional依存へ追加 | 同じ辺集合/診断順。未対応側は失敗し、辺を省いた空成功を返さない |
| `A01-20` | 骨格と通常CLIへ同じhelp/重複/値不足の引数列を渡す | P02-CLIと同じInvocationまたは固定エラー。骨格専用の例外・優先順なし |
| `A01-21` | environment=fail、全要求job=not_run、環境失敗の証拠あり | environmentのCHECK_ENVを保持しcomplete=false。空commandsの未実施jobへ失敗実行を捏造しない |
| `A01-22` | image digestは一致、内包する実行物一件の版または個別承認だけ欠損 | CHECK_ENVまたはCHECK_DEPENDENCY、job開始0件。イメージ一致で内包物の不足を隠さない |
| `A01-23` | 要求targetがBindingにない、またはクロスビルド可能だが対象CPUの実行環境なし | 前者CHECK_ENV。後者は実行jobをnot_runとしてsuite未完了。ビルド結果と実行結果を分離 |
| `A01-24` | Run(Update,plan=true)だけ未登録、通常Updateは登録済み | CLI_UNIMPLEMENTEDで停止。通常Updateへの転送0回、構造化応答はP02-CLIと同じ検証を通す |
| `A01-25` | 文書の機械判定対象外項目一件、fence内の架空リンク、未選択targetのgit依存 | 前者はunresolvedありで未完了。fenceは参照外、gitはCHECK_DEPENDENCY。空成功への縮約なし |
| `A01-26` | document_draft/documentでCargoファイルなし、product_acceptance/document | 前者は文書検査環境だけ照合、後者CHECK_INPUT。製品依存やCPU実行の成功を捏造しない |
| `A01-27` | unresolvedは空だが禁止依存diagnosticあり、入力table順だけ逆転 | 前者はcomplete=false。後者は同じ安定順の全診断を返す |
| `A01-28` | TOMLのinline table末尾comma、array末尾comma、quoted keyに依存節名 | 順に拒否/受理/構文木で判定。依存文字列の単純一致で節を取り違えない |
| `A01-29` | 4backtick fence内に3backtickと架空リンク、raw HTMLアンカー参照 | 短いfenceで終了せずリンクは参照外。HTMLはunresolvedとし意味判定を自動成功にしない |
| `A01-30` | jobsへenvironmentを混入、environment=not_run、起動前に失敗したjob | 混入はCHECK_INPUT、未実施環境では全job未実施。起動前失敗には実行コマンドを捏造せず証拠を要求 |
| `A01-31` | 入力mapにa/../b、本文欠損、path依存の実体証拠なし | 入力不正または依存確認待ち。空本文・リポジトリ内実体を捏造せず、純粋検査器の追加I/Oなし |

## 未決事項と成立判定

| 判断ID | 必要な判断・根拠 | 依存範囲 |
|---|---|---|
| `D01-01` | 推奨案はP02-CLIの単一解析契約と本書の親予算を採用。所有者は参照版・A01-20・予算算定を一組で決定する。採用待ちとし、骨格側の独自補完は残さない | `P01-CLI`・`P01-ENV`と後続CLI |
| `D01-02` | environmentを持つ検査形式・TOML 1.0.0の限定集合/GFM構造参照・安定診断順を推奨。残る採用判断には未対応構文停止、GitHub節IDの照合根拠、意味判定対象外の確認待ちを含める。全標準互換は選択肢に含めない | `P01-CHECK`、全フェーズの自動検査 |
| `D01-03` | 固定イメージ方式を推奨しEnvironmentBindingへ対応付ける。残る根拠はEX-007〜009の個別版・条件・承認と両CPU実行環境。これらの採用は本案作成承認の対象外 | 実際のビルド・検査・配布 |

難易度の判断方法は[段階的な内製化](../PRINCIPLES.md#段階的な内製化)、新設・抽出・差し替え・更新の工程は[実施手順](../../AGENTS.md#内製化と差し替えの実施手順)へ照合する。初回は新設案のため旧実装・新旧比較結果は存在しない。将来の抽出候補も、担当契約と呼び出し元・移行・契約テストを先に確定し、今回はcrate名・版を確定しない。

本文案・受入設計は作成済みだが、各判断ID、`P02-CLI`の使用部分、例外採用情報と非機能要求の不足が残る。完全仕様の採用承認・仕様充足は未成立。製品実装・受入試験は未着手・未実施とする。
