# Phase 11 完全仕様案: 統合受入と初期リリース

[文書一覧](../DOCUMENT_INDEX.md) | [担当工程](../ROADMAP.md#phase-11-全体の統合受入初期リリース)

採用未承認案`B11`。[案の適用範囲](PHASE_01.md#案の適用範囲)に従う。対象は`REQ-001`〜`REQ-012`。製品の[受入と追跡](../BASIC_SPEC.md#受入と追跡)、[CI/CD](../BASIC_SPEC.md#cicd)、[リリース工程](../BASIC_SPEC.md#リリース工程)を参照し、新しい公開経路や署名鍵の運用規則を本書に作らない。

## 契約と依存範囲

| 契約ID | 所有する範囲 |
|---|---|
| `P11-ACCEPT` | 全要件・経路・構成の受入集合と結果の追跡 |
| `P11-RELEASE` | 固定入力・成果物・試験と公開対象の同一性 |

本フェーズは各契約（全フェーズB11）の`P01-CHECK`・[検査](PHASE_01.md#p01-check)、`P02-MODEL`・[共通型](PHASE_02.md#p02-model)、`P02-CLI`・[CLI](PHASE_02.md#p02-cli)、`P03-PLAN`・[計画](PHASE_03.md#p03-plan)、`P04-SELF`・[本体](PHASE_04.md#p04-self)、`P04-BACKUP`・[保全](PHASE_04.md#p04-backup)、`P05-ISO`・[媒体](PHASE_05.md#p05-iso)、`P06-APP`・[サーバ](PHASE_06.md#p06-app)、`P07-RUN`・[採掘](PHASE_07.md#p07-run)、`P08-COHOST`・[同居](PHASE_08.md#p08-cohost)、`P09-RECONNECT`・[SSH照合](PHASE_09.md#p09-reconnect)、`P10-OS`・[OS導入](PHASE_10.md#p10-os)を統合する。各担当の残る契約も受入対象の依存として追跡する。

## P11-ACCEPT

採用案の受入行は、要件ID、担当契約のファイル・節・ID・承認済みコミット、受入ケースID、入力・前提・期待結果、対象OS・CPU・構成・実行経路、環境識別、対象成果物のハッシュ、実出力・副作用、成否と未達理由を必須とする。

| 検証軸 | 必須とする照合案 |
|---|---|
| 各フェーズ | Phase 1〜11にある全受入ケースと、その依存する共通契約。ケースID/入力/期待結果を固定集合へ列挙し、件数だけで充足にしない |
| 共通本体 | 基本仕様の両CPU・GNUターゲットでビルドと実行を別に確認 |
| 物理経路 | 対応CPU・起動方式の媒体起動、三構成の適用可否、取外し後の内蔵OS起動・再起動 |
| 遠隔経路 | 対応する既存OS、OS未導入、起動不能、権限不足、切断・再接続と要求重複 |
| 構成機能 | 単独server・miningと物理masterの同居。CPU対応の差は各構成の正本へ照合 |
| 保全・復旧 | 独立と更新内のバックアップ、期限・破損・容量不足、実適用位置と復元先の照合 |
| 非機能 | 各契約の上限・停止・保全と、基本仕様で確定する要求水準。未定義は未達 |

受入集合の算定は、適用可能な組合せを一意に列挙し、非適用の理由を正本へ対応付ける案。arm64の本体・ISOがあることを、arm64でmining・masterが使える証明にしない。対応外入力を拒否する試験と、対応する機能の成功試験を分ける。

経路/CPU/構成の期待行の候補は次表へ限定する。各行のケースは担当契約の正常・異常・境界設計から固定し、実結果の有無で減らさない。boot方式はenvironment_refの固定環境に含め、amd64媒体はUEFI/BIOS、arm64媒体はUEFIを別環境の別行にする。

| 入口 | CPU | 成功経路を検証する構成 | 必須の拒否経路 |
|---|---|---|---|
| internal（共通本体） | amd64/arm64 | profile=nullの共通契約 | 非対応CPU/不正入力は別fixture |
| media | amd64 | server/mining/master | 構成選択不能・機器能力不足 |
| media | arm64 | server | mining/masterの選択後適用拒否 |
| ssh_existing | amd64 | server/mining | Cloudへのmaster適用拒否 |
| ssh_existing | arm64 | server | mining/masterの適用拒否 |
| ssh_bootstrap | amd64 | server/mining | 使用中disk・能力不足・master適用拒否 |
| ssh_bootstrap | arm64 | server | mining/master・能力不足の適用拒否 |

純粋評価の同一ケースを全経路へ重複実行するのではなく、各ケースの担当契約・適用行・拒否入力・固定環境への対応をScopeで一度だけ列挙する。ケースIDの欠損、適用行の欠損、同じ一意キーの重複をそれぞれACCEPT_SCOPEとして検出する。非機能の要求値や対象実機が未決なら該当行をnot_runで残し、適用行の存在だけを仕様充足にしない。

未承認案・未確定契約・未採用例外に依存する実装試験は実行可能としない。文書案の構文・参照検査だけは別目的で行えるが、受入行の製品試験結果には置換しない。実機検証の境界は[未確定の試験条件](../SECURITY_POLICY.md#検証環境の未確定条件)を維持し、VM成功だけで実機対応を付けない。

## P11-RELEASE

既存の[成果物名・タグ・工程](../BASIC_SPEC.md#リリース)と[真正性・公開前条件](../SECURITY_POLICY.md#公開前の検証)に、次の候補の確認行を追加する方式とする。

1. 採用承認済み仕様・実装コミット・固定環境・例外台帳の適用版と、リリースの対象範囲を特定する。
2. 対象コミットから各CPUの本体を一度だけ生成し、ビルド・依存・CLI・実行試験を照合する。
3. ビルド情報と本体マニフェストを確定し、署名工程の許可条件を確認して所定の署名処理へ渡す。
4. 同じ本体からISOを生成し、必要なVM・媒体なし起動と経路別の受入を行う。変更・再生成があれば影響する試験を再実施する。
5. 本体・媒体・検証情報の相互参照と必須成果物の集合を照合し、欠損・重複・差し替えを検出する。
6. 承認された公開操作の範囲で候補を配置・再取得して再照合し、一致した同一成果物だけを正式公開する。

各生成・検証操作の上限はP01-ENV・P05-ISOと各担当契約へ接続し、署名・搬入・公開要求の各上限は300秒、再試行0回を候補とする。公開要求の応答欠損は未公開と決めつけず、同じタグの実配置を読み取って一致を確認する。タグ・成果物の強制上書きで合わせない。

既存の「非公開のdraft release」という工程表現について、実際のリポジトリ公開状態・閲覧権限・API動作との一致はD11-02の確認対象とする。文書の表現だけで非公開性を保証せず、条件未達なら公開工程を停止する。

正式な許諾、第三者の同梱・再配布、署名鍵の保管・利用・ローテーションの未確定部分は[一般ポリシー](../POLICIES.md#ソースコードの公開と利用条件)、[署名鍵管理](../SECURITY_POLICY.md#リリース署名鍵)、[例外台帳](../DEPENDENCY_EXCEPTIONS.md)へ残す。本案で条文・鍵運用・例外の現在状態を確定しない。

公開直前の審査集合は、独自成果物の正式許諾版、第三者資材ごとの再配布/表示、例外の有効情報、本番鍵の利用条件、実機/VMの適用境界、公開対象/権限の六区分を一対一に証拠へ対応付ける案を推奨する。各本文は担当ポリシー/台帳にのみ置き、ReleaseEvidenceには参照と判定だけを結合する。未確定が一つでもあればpublishable=falseだが、文書案そのものの作成/構文検査まで製品公開と同一視しない。正式許諾制定・鍵操作・個別例外採用・実機操作は今回の承認範囲外である。

## 検査の入出力・内製単位案

統合受入の集計・成果物間の対応照合は内製候補とする。既存の`ci/`・`release/`という配置境界を維持し、Rustで所有する判断処理と薄い実行ラッパーの役割を分ける。具体的な補助実行体の構成はD11-01で決定するまで追加しない。

| 宣言案 | 入出力・副作用 |
|---|---|
| 受入集計 | [collect_acceptanceの型付き宣言](#型とapiの詳細案)だけを使用。全期待行・実結果・未達を照合 |
| リリース検証 | [verify_releaseの型付き宣言](#データ契約の具体化案)だけを使用。純粋検証で自動公開なし |

共通型はP02-MODEL。受入行・証拠・成果物集合の完全スキーマ、許容する版、結果と実行上限の対応はD11-01に残る。難易度は集計が中、正式公開との統合が高。新設案で旧実装なし。将来の差し替えは同じ固定成果物と欠損・改変fixtureで検証し、署名済みバイト列や公開済みタグを移行目的で書換えない。

## データ契約の具体化案

`P11-ACCEPT`の受入行と`P11-RELEASE`の同一性検査へ渡す証拠を具体化する候補。将来の製品受入の形式であり、今回の文書検査をpassとして記入するための形式ではない。

| 型 | 全キー・値域の候補 |
|---|---|
| AcceptanceRow | requirement_id: 既存要件ID、contract: ContractRef、case_id: 受入ID、architecture: 既存CPU値、profile: 既存構成またはnull、entry: media・ssh_existing・ssh_bootstrap・internal、execution_class: CaseExecutionClass、environment_ref・artifact_sha256・expected_ref: string、result: pass・fail・not_run・not_applicable、actual_ref・reason: stringまたはnull |
| AcceptanceMatrix | schema_version: 1、implementation_commit: 完全コミットID、rows: AcceptanceRow[]、complete: boolean |
| ReleaseEvidence | schema_version: 1、tag・commit・build_environment_ref: string、artifacts: ArtifactEvidence[]、acceptance: AcceptanceMatrix、publishable: boolean |
| ArtifactEvidence | name・sha256: string、bytes: 正のinteger、producer_ref: string、verification_refs: string[] |

受入行の結合キーは要件ID・契約ID・ケースID・CPU・構成・入口・execution_class・environment_refの組で一意にする。契約のfile/anchor/版/commitとexpected_refも別に完全一致を検査する。pass/failはactual_ref必須、not_run/not_applicableはreason必須。not_applicableには担当正本の非適用根拠への参照を要求し、能力未確認・試験困難・未実装を非適用にしない。必須行のnot_run/fail、欠損・重複・承認済み契約や期待結果の不在が一件でもあればcomplete=false。

expected_refは実出力から作らず、採用済みの受入条件と固定した入力・前提への参照とする。actual_refは実行コマンド・対象・前後状態・実出力・成否を特定する所有者管理下の証拠を指す。証拠のバイト列・対象コミット・環境・成果物が変わった場合は、同じケース名でも結果を自動再利用しない。

ReleaseEvidence.artifactsの期待集合は[既存成果物名](../BASIC_SPEC.md#リリース)だけから生成し、各名前をちょうど1件要求する。本体とISOのCPU対応、両マニフェストとbuild情報、署名・checksumの相互参照を既存工程へ照合する。単なるファイル名一致やproducerの申告だけをverification_refsの代わりにしない。

リリース集合の候補検証順序は固定tag/commit/version→必須成果物名の完全一致→各byte数/hash→本体manifestからbuild/各CPU本体→媒体manifestから本体manifest/ISO/収録本体→checksum対象集合→全detached署名の対象→受入証拠→公開条件とする。checksum自身とその署名は既存仕様どおり一覧対象外だが、配布必須集合からは除外しない。余分なassetや重複名もRELEASE_SETで停止し、自動削除で一致させない。各署名の検証成功を別ファイルへ流用せず、再取得後も同じbyte集合と参照グラフを再検証する。

publishable=trueの候補条件は、全必須成果物の一致、必要な実行・VM受入のcomplete、各許諾・例外・署名鍵・公開設定の担当条件の成立を所有者が確認できること。未確定の担当条件はfalseとし、チェック数の多数決で成功にしない。公開可否の評価関数と実際の公開操作は分離し、評価成功だけでタグ作成・署名・uploadを開始しない。

純粋検証の候補APIは`verify_release(scope: &AcceptanceScope, set: &ReleaseSet, evidence: &ReleaseEvidence, catalog: &[VerifiedEvidence]) -> Result<ReleaseCheck, ContractError>`。ReleaseCheckはchecks（Check[]）とpublishable（bool）を所有し、入力側のpublishable=trueは信用せず上記条件から算定する。形式/識別不一致はErr、正しく識別できた対象の条件不足はchecksに根拠を残したpublishable=falseとする。検証と実際の公開adapterを別に保ち、API応答欠損時の実配置は読取り照合へ戻す。

受入集合の全列挙、証拠の保存・改変検出、集計用Rust型と公開APIの版・権限の実証は`D11-01`〜`D11-03`の未達に残す。文書だけの改善でこれらの証拠が存在した扱いにはしない。

## 接続と責務の照合

[P01-CHECK](PHASE_01.md#データ契約の具体化案)の検査目的・コミット・環境と、AcceptanceMatrixの対象を照合する。文書案・構文検査・クロスビルドの結果は、製品実行・VM・実機の代替にせず、各担当が要求する証拠だけを適用する。test署名と本番署名、配布物検査とソース公開審査も混同しない。

媒体の起動試験と[P05-HANDOFF](PHASE_05.md#接続と責務の照合)の取り外し後の起動・再起動は別行にする。マスターでは三機能の結果を統合し、arm64のmining/masterを成功試験の対象へ推測追加しない。一方で、対応外構成を拒否する試験は非適用として消さない。

ReleaseEvidenceは「同じ本体をISOに収録した」証拠と、手動VM試験で使った候補のハッシュを結合する。ISOを再生成してハッシュが変わったら、そのISOの内容・起動・導入検証を再照合する。同じタグ・recipe・ファイル名だけを結果再利用の根拠にしない。公開要求の応答欠損は実際の配置を照合し、強制upload・タグ付け替えで一致させない。

許諾・署名鍵・実機試験境界・個別例外は担当正本の未確定部分を維持する。審査候補の文書化と、正式条文・方式・固定版の採用や実施を区別し、未確定部分を全体の採用・実装移行・正式公開を妨げる判断事項として所有者へ提示する。反復回数や受入設計数をそれらの解消根拠にしない。

## 受入ケース案

| ケースID | 区分・前提 | 期待結果・照合方法 |
|---|---|---|
| `A11-01` | 正常。全必須行と実証拠が対応 | 対象の版・環境・出力・副作用を追跡できる |
| `A11-02` | 異常。未承認契約・未定義非機能・未実施試験 | 受入未完了。件数や別試験の成功で埋めない |
| `A11-03` | 異常。クロスビルドだけ成功、片CPUで未実行 | 実行試験未実施として公開条件を満たさない |
| `A11-04` | 異常。ISO内本体・metadata・署名対象の不一致 | 配布集合の検査失敗。別ビルドで差し替えて続行しない |
| `A11-05` | 異常。必須起動方式不足、媒体なし起動未検証 | 対応済みISOとして公開しない |
| `A11-06` | 異常。test鍵・未確定許諾・再配布条件不足 | 担当ポリシーへ照合して正式公開を停止 |
| `A11-07` | 境界。再取得後の改変、公開要求の応答欠損 | 実配置を照合し、一致確認まで成功・再送を決めない |
| `A11-08` | 異常。公開済み同名成果物・タグの変更要求 | 置換しない。新しい版・生成・検証の工程へ戻す |

## 追加の受入ケース案

`P11-ACCEPT`・`P11-RELEASE`の証拠検査を設計する。署名・成果物・実行結果は試験用fixtureであり、正式リリースや本番鍵の使用を今回許可しない。

| ケースID | 区分・前提と入力 | 期待結果・照合方法 |
|---|---|---|
| `A11-09` | 異常。必須受入行を1件欠損、同じキーの行を2件、actual_refなしpass | complete=false。件数・多数決で不足を隠さない |
| `A11-10` | 異常。expected_refが実出力の自己申告だけ、契約は未採用 | 受入証拠として拒否。文書検査の結果を製品結果へ置換しない |
| `A11-11` | 異常。同じrecipe・版で再生成したISOだけハッシュ変更 | 変更ISOへ古い起動・導入結果を流用しない。publishable=false |
| `A11-12` | 異常。媒体起動はpass、取り外し後の起動・再起動がnot_run | 物理経路の受入未完了。removable=trueでも代替しない |
| `A11-13` | 異常。全ファイル一致だが鍵・許諾・例外・実機境界のいずれか未確定 | publishable=false。対象正本と必要な所有者判断を提示し、公開操作0回 |

## 型とAPIの詳細案

`P11-ACCEPT`は要求される行の集合を実結果から作らない候補。ScopeとExpectedCaseを先に固定し、実結果の欠損・余分な結果を検出する。

```rust
pub struct ExpectedCase {
    requirement_id: String, contract: ContractRef, case_id: String,
    architecture: String, profile: Option<Profile>, entry: String,
    execution_class: CaseExecutionClass, expected_ref: EvidenceRef,
    artifact_ref: EvidenceRef, environment_ref: String, required: bool,
}
pub struct AcceptanceScope {
    schema_version: u32, scope_id: String, specification_commit: String,
    implementation_commit: String, cases: Vec<ExpectedCase>, artifacts: Vec<ExpectedArtifact>,
}
pub struct ActualCase {
    key: ExpectedCase, implementation_commit: String, environment_ref: String,
    artifact_sha256: String, result: String, actual_ref: Option<String>, reason: Option<String>,
    evidence: Vec<EvidenceRef>,
}
pub struct VerifiedEvidence { reference: EvidenceRef, verification_ref: String }
pub struct EvidenceRef {
    evidence_id: String, relative_path: String, sha256: String, bytes: u64,
    commit: String, environment_id: String, scope_id: String,
}
pub struct ExpectedArtifact {
    name: String, architecture: Option<String>, producer_contract: ContractRef, required: bool,
}
pub struct AcceptanceRow {
    requirement_id: String, contract: ContractRef, case_id: String, architecture: String,
    profile: Option<Profile>, entry: String, execution_class: CaseExecutionClass,
    environment_ref: String, artifact_sha256: String,
    expected_ref: String, result: String, actual_ref: Option<String>, reason: Option<String>,
}
pub struct AcceptanceMatrix {
    schema_version: u32, implementation_commit: String, rows: Vec<AcceptanceRow>, complete: bool,
}
pub struct ArtifactEvidence {
    name: String, sha256: String, bytes: u64, producer_ref: String, verification_refs: Vec<String>,
}
pub struct ReleaseEvidence {
    schema_version: u32, tag: String, commit: String, build_environment_ref: String,
    artifacts: Vec<ArtifactEvidence>, acceptance: AcceptanceMatrix, publishable: bool,
}
pub struct ReleaseSet {
    schema_version: u32, tag: String, commit: String, manifest: EvidenceRef,
    artifacts: Vec<EvidenceRef>, acceptance_scope: EvidenceRef, acceptance_result: EvidenceRef,
    draft_release_id: Option<u64>,
}
pub struct ReleaseCheck { checks: Vec<Check>, publishable: bool }
pub struct ReleaseDestination {
    owner: String, repository: String, tag: String, target_commit: String,
    release_id: Option<u64>, asset_ids: std::collections::BTreeMap<String, u64>,
}
pub fn collect_acceptance(scope: &AcceptanceScope, actual: Vec<ActualCase>,
    catalog: &[VerifiedEvidence])
    -> Result<AcceptanceMatrix, ContractError>;
```

resultの許容値は[受入行](#データ契約の具体化案)へ参照する。ExpectedCaseのkeyに含める識別項目だけで結果を結合し、expected_ref・承認済みcommitも別に完全一致を検証する。同じcase_idでもCPU・構成・入口が違えば別行。参照先がない期待値、未知の行、欠損・重複を失敗とし、実結果側からscopeを縮小しない。戻り値のcompleteは関数が算出し、呼出元のtrueを信頼しない。

結合の候補順はrequirement_id・contract.contract_id・case_id・architecture・profile・entry・execution_class・environment_refのUTF-8バイト辞書順で、profile=nullはstringより前とする。残るContractRef全項目・expected_ref・artifact_ref・requiredも固定scopeと完全一致させ、同じ結合キーで内容だけ変えた行を受理しない。Scopeのcase_id集合は担当本文の全受入設計へ照合し、同一ケースの全直積を自動生成しない。各ケースの適用行/拒否行が未決ならscope未成立とし、集計器の成功と分ける。

ExpectedCase.expected_refは検証済みEvidenceRef、AcceptanceRow.expected_refは同じ証拠のevidence_idだけを保持する。ActualCase.keyから既定の識別組を複写し、実結果を照合してrowを作る。参照を丸ごと別の自由stringへ直列化しない。not_runの実artifactが存在しない場合にゼロhashを生成せず、その行のartifact_sha256は期待する成果物hashとしてscopeの証拠から照合し、実取得の有無はactual_ref/evidenceで区別する。未確定の成果物を必要とするscopeは生成せず、document_draftの検査結果で製品scopeを補完しない。

| ケースID | 固定入力 | 期待結果 |
|---|---|---|
| `A11-33` | expected_refのIDだけ一致しhash違い、not_runへ架空の実hash、publishable入力true | ACCEPT_EVIDENCEまたは公開不可。参照名・入力bool・未実施データの捏造による完了なし |

集計内部は上記collect_acceptanceだけを使用し、別のValue返却APIを実装しない。verify_releaseも純粋評価に分離する。集計器の動作試験と、各受入行が指す製品試験の結果は別々に追跡する。

## 状態と中断の詳細案

`P11-RELEASE`の内部工程候補はinputs_fixed→binaries_verified→binary_signed→iso_verified→set_signed→staged→retrieved_verified→published。工程名は公開済み状態の自己申告ではなく、各固定成果物への検証証拠を必要とする。

| 事象 | 停止・再開の扱い |
|---|---|
| 本体・build metadata・manifestを署名後に変更 | 対応する固定・生成・署名・試験へ戻す。古い署名を再利用しない |
| ISOだけ再生成・ハッシュ変更 | 当該ISOの内容・起動・導入・媒体なし起動を再検証。名前一致だけでは再利用不可 |
| 一つの必須成果物不足・upload失敗 | staged未完了。部分集合を正式公開しない |
| upload応答欠損 | 実際のdraft/tag/assetを読取り照合。未配置と推測して上書きしない |
| 再取得したbyte列が不一致 | 公開停止。提出候補を配布先の別物へ合わせない |
| publish応答欠損 | 実際の公開状態と全assetを確認。再度の公開要求を自動送信しない |
| 公開後の欠陥 | 公開物の固定を維持し、新しい版の判断へ。強制upload・tag付替えなし |

集計の停止と、既に実行した署名・配置・公開の実状態を分ける。publishable=falseへ再判定しても公開が取り消された事実にはならず、保護・是正は所有者の承認範囲を別に確認する。秘密情報や非公開報告を公開証拠へ含めない。

## 依存と互換性の詳細案

受入・公開は、全P01〜P10の必要契約と例外台帳の個別情報・許諾・署名鍵・公開権限へ依存する候補。公開APIの戻り値だけで実際の閲覧状態・assetのbyte列を証明しない。

| 審査対象 | 候補の判断と残る根拠 |
|---|---|
| draft配置 | 実リポジトリの閲覧権限とdraft資材の取得権限を別に照合。非公開性を名称から推測しない |
| GitHub操作 | 採用API版・認証権限・tag/release/asset識別・失敗/応答欠損の読取り照合をEX-009へ個別登録 |
| 正式署名 | 試験鍵から分離した本番手順をセキュリティポリシーで確定。CIへの鍵持込みを暗黙承認しない |
| 許諾 | 独自成果物と第三者資材の対象・条文・再配布条件を別々に確認。公開目的を利用許諾へ変換しない |
| 結果再利用 | 同じcommit・環境・入力・成果物・期待契約への適用根拠を確認。変更した軸の試験は再実施 |

リリース確認器をcrateへ抽出する場合も公開操作は別責務に維持し、検査器の単体試験から実upload/署名を呼ばない。成果物集合・相互参照・欠損/改変・未実施結果への判定を同じfixtureで比較する。公開済み資材をテスト用の書換え対象にしない。

## 受入条件の充足照合案

全38契約の審査資料は、契約IDと[13必須項目](../POLICIES.md#完全仕様の必須条件)の掲載順index（1〜13）の組を一意キーとする候補。494組を欠損/重複なく対応付け、各組に定義先のファイル/節、共通参照、受入case ID、未達の判断IDを持たせる。本文の存在・型宣言の存在・参照検査の成功と、内容が成立している判定を別欄にする。要件を定義していない本文へ形式だけで充足印を付けない。

候補がある組、具体化不足、方式根拠不足、所有者入力/採用待ち、適用外の根拠を区別する。適用外は根拠と影響を持つ場合だけ、共通参照も参照先の同じ条件まで辿る。未実装の製品試験はnot_runで別管理し、この文書審査の証拠には含めない。A11-16/25へ一契約一項目欠損、根拠のない適用外、同じ参照の別版、案だけの採用済み判定を含める。最終の仕様充足判定は所有者が行い、494組があるという件数で代替しない。

各フェーズの同名節は担当契約・受入ケース・未達を対応付ける。全体の成立判定では、[必須条件](../POLICIES.md#完全仕様の必須条件)の13項目を次表の不足へ照合する。案の具体化先を示すもので、13項目を満たしたという判定ではない。

| 必須項目 | 具体化先と現在の不足 |
|---|---|
| 要件と担当フェーズ | 基本仕様の受入/追跡→P01〜P11の契約一覧。採用承認済みcommitが全件未成立 |
| 責務と範囲 | 各契約・各依存表。機種/一時実行環境/外部採用範囲はD05-04・D07-01・D10-01等が残る |
| 入力 | 各「入力と保存形式の詳細案」にキー/型/条件を具体化。設定/記録の採用・版の対応、機器別値域とOS生成契約はD02-03・D06-01・D07-02・D10-02等が残る |
| 既定値と検証 | P02-CLI/PARSEと担当validator。ポリシーfalse・全形式の検証順序の採用判断が残る |
| 出力 | P02-CLI、各receipt/遠隔payloadと検査集合の候補あり。担当機器/アプリの観測ID・全エラー対応・実装可能な保存方式の採用はD02-02等が残る |
| 実行と状態 | 各「実行と復旧の確定候補」に対象worker/引渡し/取消/部分確定を記載。P02の停止予約、P05の二確認/媒体識別、P09の論理セッション/最終確認位置は候補あり。内部入口・親停止時の監督・導入方式の完全な実現契約が残る |
| 副作用と保護 | P02-STORE/EXEC・P03-FS・各操作。root固定・排他引継ぎ・媒体解除方式が残る |
| 失敗と復旧 | ChangeFailure、資源journal、各中断表で実位置と記録不一致を具体化。OS固有の確定/停止成立性・復旧可能範囲の採用はD02-05・D04-04・D09-02・D10-03等が残る |
| 判定と数値 | 各候補の境界と算定、TTLの秒/ms、HTTPの全応答量、IPv4/MACの字句は担当本文に候補あり。機器値域・データ方法別の復元期限・同居の最低資源等が未確定 |
| 非機能の検証 | 各受入・上限・依存表。製品共通の目標/測定/合否、OS方式の成立性確認が残る |
| 保存と互換性 | 共通記録/構成設定/遠隔/媒体/証拠の形式候補あり。版の採用、旧形式/三構成の実移行行・主体切替は未確定 |
| 依存と外部契約 | 各依存表と例外台帳。全個別固定版・全引数・許諾・承認根拠が未完了 |
| 受入条件 | 各A01〜A11の設計とVerifiedEvidenceを使う純粋集計の境界。方式未確定のケース、全組合せの固定集合と採用判断が残る |

| ケースID | 区分・固定入力 | 期待結果と確認 |
|---|---|---|
| `A11-14` | 異常。全実結果passだがAcceptanceScopeの期待行を1件削除 | 承認済みscopeとの不一致。縮小した集合を全体completeにしない |
| `A11-15` | 異常。upload応答欠損、同名assetは存在するがhash違い | 配置不一致で停止。自動上書き・再upload0回 |
| `A11-16` | 境界。設計した全受入ケースを列挙、13必須項目の1項目が未確定 | ケース数で充足認定せず未成立。設計の存在を実施済みにしない |

仕様策定の成立と、製品受入の完了は別工程。将来の実装試験の設計を確定することは必要だが、仕様策定中に製品試験の実結果がないことだけを未成立理由にはしない。一方、方式・入力・保護・判定が一意に定まらない現在の未達は、反復回数で解消せず所有者の採用判断と残る具体化へ戻す。

## 入力と保存形式の詳細案

P11-ACCEPTの証拠参照を以下の候補へ閉じる。所有者の判定資料と製品テストの実出力を別に保ち、結果ファイル自身が作った期待集合を使わない。

| 型 | 全キー・型 |
|---|---|
| EvidenceRef | evidence_id: ASCII識別子、relative_path: string、sha256: hex64string、bytes: u64、commit: Git完全ID、environment_id: string、scope_id: string |
| ExpectedCase | requirement_id・contract・case_id・architecture・profile・entry・execution_classは受入行と同じ型、environment_ref: string、required: bool、expected_ref・artifact_ref: EvidenceRef |
| AcceptanceScope | schema_version: 1、scope_id: string、specification_commit・implementation_commit: Git完全ID、cases: ExpectedCase[]、artifacts: ExpectedArtifact[] |
| ExpectedArtifact | name: 必須成果物名、architecture: 既存CPUまたは全体用null、producer_contract: ContractRef、required: bool |
| ActualCaseの補完 | 既存結果にevidence: EvidenceRef[]を結合。passは期待参照と実観測の対応が必要、not_runは実出力を捏造しない |
| ReleaseSet | schema_version: 1、tag: string、commit: Git完全ID、manifest: EvidenceRef、artifacts: EvidenceRef[]、acceptance_scope: EvidenceRef、acceptance_result: EvidenceRef、draft_release_id: u64またはnull |

relative_pathは証拠ルート内に限定し、absolute・親逸脱・symlinkを拒否する。証拠本文はbyte列を読み込んで長さ/hashを確認し、参照文字列の存在だけで実行済みとしない。ExpectedCaseは受入行の識別組とenvironment_refで一意とする。必須ケースの欠損・余剰・目的違い・別commitを集計前に拒否する。ExpectedCase.requiredは対象scopeの固定要求であり、実結果のnot_applicableだけでfalseへ変更しない。非適用は担当正本の根拠とscopeの対応が一致する場合だけ受理する。推奨は期待集合を先に固定する純粋集計器で、実行できた行からのscope生成は採らない。

EvidenceRef.scope_idは固定したscope識別子であり、scope自身のdigestを内部へ埋める循環形式を作らない。scope全byte列のhashは外部の固定入力資料へ保存して照合する。期待証拠のcommitはspecification_commit、実出力のcommitはimplementation_commitへ照合し、受入行のexpected_refは検証済みEvidenceRefのevidence_idへ対応付ける。上記structに出てくるEvidenceRef/ExpectedArtifactは本表の検証済み値を保持する非公開フィールド型とする。

VerifiedEvidenceは収集器が証拠root内の実体・byte数・実hash・commit/環境/scopeへの対応を確認した後だけ生成する候補。verification_refはその確認資料の参照で、任意のEvidenceRefからの無検査変換を設けない。純粋集計器はcatalogのevidence_idを一意に解決し、全フィールド一致を照合するだけで、file読取り・hash実行・ダウンロードを行わない。欠損/重複/別対象の証拠はACCEPT_EVIDENCE。大容量ISO等のbyte列を全てRAMに保持せず、収集器が固定実体を有界読取りで検証した資料へ結合する。実体が変わればVerifiedEvidenceを失効させ再収集する。

必須成果物名の全集合は[初期リリース](../BASIC_SPEC.md#リリース)を一度だけ参照し、ReleaseSet.artifactsと一対一照合する。公開用証拠に秘密値を含めない検査はSECURITY_POLICYが担当する。署名対象byte列は再整形せず、配布本体・ISO内本体・マニフェストの実hashを対応付ける。

| ケースID | 固定入力 | 期待結果 |
|---|---|---|
| `A11-17` | 証拠参照は存在するがbytes/hash違い、別commitのpass、必須成果物一件欠損 | 集計/公開前停止。パス存在や結果行のpassだけで受入完了にしない |

## 実行と復旧の確定候補

P11-RELEASEの公開操作候補は、固定したReleaseSet→draft作成/照合→全asset新規配置→全asset再取得検証→所有者の公開条件確認→公開→実公開状態照合の順とする。現時点でこれらのAPI操作や署名を行わない。

[GitHub公式Releases API](https://docs.github.com/en/rest/releases/releases?apiVersion=2022-11-28)のrelease ID・tag・draft・asset IDを記録する方式を候補とする。draftの閲覧はリポジトリの権限へ依存するため、draftという名称を機密保証にしない。実リポジトリの設定・利用者権限による閲覧確認は公開前の別検証とする。

| 境界 | 再試行と照合 |
|---|---|
| draft作成の応答欠損 | tag/commitと実releaseを読取り照合。自動の二重作成なし |
| asset uploadの応答欠損 | 同名assetのID/サイズ/実bytes/hashを確認。存在だけで成功・強制上書きなし |
| 再取得の不足・不一致 | retrieved_verifiedへ進めず停止。同名の別物へマニフェストを合わせない |
| 公開要求の応答欠損 | draft状態と実閲覧・全assetを読む。公開済みなら実事実を記録し、再公開要求なし |
| 公開後に誤り発見 | 新版または所有者承認の保護措置へ。タグ付替え・同名asset差替えを自動実行しない |

公開先の候補識別ReleaseDestinationはowner・repository・tag・target_commit・release_id・asset_idsを全キーとし、release_idだけ新規作成前null、asset_idsは成果物名→asset IDのobjectとする。owner/repository/tagは固定入力、APIの応答から別リポジトリへ切り替えない。403/404/timeoutだけで未配置を断定せず、権限と実配置の照合が成立するまで再作成しない。draft閲覧は[公式APIの権限説明](https://docs.github.com/en/rest/releases/releases#list-releases)へ対応付け、実リポジトリの閲覧確認を別に行う。

asset照合の候補ではrelease IDを固定して列挙の終端まで確認し、第一pageだけで全集合を確定しない。必要集合より一件でも多い・同名重複・同ID重複・未確認の次pageがある状態ではRELEASE_SET。次pageの取得は同じAPI origin/固定release/宣言済み認証/親予算の範囲だけとし、不明な外部URLへ認証を継承しない。staged以後のasset_idsは必須成果物名と一対一で、実配置のID/名前/量と取得後の実hashを別々に照合する。配置途中の部分mapは進捗として保持してもstagedにしない。照合前後でtag/commit/release/draft/asset集合が変わったらRELEASE_CHANGEDで停止し、二回一致だけを同時変更の排除証明にしない。

本書の300秒は一つの署名/搬入/公開工程の親予算で、API呼出しはその内側の一回60秒・再試行0回。asset取得はP04の取得上限とReleaseSetのサイズへ制限する候補。ISO生成・VM/実機試験は各専用予算を採用してから実行し、通常検査30分へ隠れて延長しない。全体予算・API固定版・認証方式はD11の審査へ残る。

API資料のURL queryにある版番号をHTTPのX-GitHub-Api-Version採用値と同一視しない候補。2026-10-22確認時の[公式asset API](https://docs.github.com/en/rest/releases/assets#get-a-release-asset)は200/302の取得契約を示しており、公開adapterが参照するAPI版・メディア型・認証権限は例外台帳で別に固定する。asset名の比較は固定配布ファイル名の完全一致、CPUは期待集合とmanifestの両方へ照合する。再取得はP04の取得契約へ接続し、未採用redirect方式を公開工程だけで有効化しない。

| ケースID | 固定入力 | 期待結果 |
|---|---|---|
| `A11-18` | publish応答欠損後に実release公開済み、asset一件だけ別hash | 公開事実と集合不一致を別に報告。再publish/差替えなし、是正判断待ち |

## 依存と受入の確定候補

P11-ACCEPTの内製対象は独立した期待集合との結合・証拠同一性検査・充足集計。P11-RELEASEは固定ReleaseSetの公開工程調整。ビルド・ISO・署名・VM・GitHub操作は各担当の個別例外へ接続する候補。

受入の期待行は[基本仕様の要件](../BASIC_SPEC.md#受入と追跡)×担当契約×対象CPU/構成/入口の必要な組を所有者が固定し、case_id単位でExpectedCaseへ列挙する。対応外の構成の拒否試験も含める。全直積を無条件に要求したり、実装できた行だけ抽出したりしない。性能/容量/停止上限の期待値が未定義ならその項目は未充足であり、件数の増加で閉じない。

| 検査ID候補 | 確認資料 |
|---|---|
| P11-ACCEPT/scope.complete | 承認済みscopeと全期待行、仕様/実装コミット、非適用根拠 |
| P11-ACCEPT/evidence.bound | 実行環境・実出力・成果物hashとExpectedCaseへの対応 |
| P11-RELEASE/set.identical | 固定した全必須配布物、ISO内本体、署名/metadata/再取得結果 |
| P11-RELEASE/publication.allowed | 許諾・個別例外・正式鍵・閲覧範囲・所有者の公開判断 |

集計器差替えは欠損/余剰/重複/別commit/非適用の各fixtureで同じ受入行・診断・complete/publishableを要求する。公開adapter変更ではAPI応答だけでなく実asset再取得と閲覧確認を再検証する。将来のcrate抽出で公開操作を純粋集計器へ移さない。

| ケースID | 固定入力 | 期待結果 |
|---|---|---|
| `A11-19` | 全実結果passだが非機能期待値なし、集計器が欠損行を除外、公開adapterの再取得未実施 | 仕様充足/受入/公開をそれぞれ未完了とする。多数決・ケース数で通過しない |
| `A11-20` | 同caseだが別environment、必須行を実結果だけnot_applicable、Phase 11ケース欠損 | 環境別照合・根拠不足/集合欠損を検出。scope縮小やPhase番号誤読による除外なし |
| `A11-21` | release作成応答欠損、照合は権限不明の404、API残予算0 | 未配置と断定せず停止。再作成/上書き/親予算を超えた照合開始なし |
| `A11-22` | 全配布物一致、六区分の一つだけ未確定、文書検査は成功 | publishable=false。文書検査の成功は保持するが正式許諾/署名/公開の成立へ転用しない |
| `A11-23` | amd64媒体のBIOS環境行欠損、arm64 master拒否ケース欠損、全実結果はpass | ACCEPT_SCOPEで未完了。成功行の全passで起動方式・拒否経路の不足を隠さない |
| `A11-24` | 期待集合へ余剰asset一件、checksum署名をmanifestへ流用、入力publishable=true | RELEASE_SET/署名対象不一致を検出。boolを再算定し、削除・署名・公開の副作用0回 |
| `A11-25` | 全契約の案・全受入設計が存在するが一判断IDの採用/成立根拠が不足 | 文書案の検査結果と不足を保持。全フェーズ成立・実装着手可能・製品試験成功へ昇格しない |
| `A11-26` | 同じ結合キーでexpected_ref/requiredだけ改変、入力行順だけ逆転 | 改変を拒否、順序変更だけなら同じ安定順の結果。scope縮小・期待値差替えなし |
| `A11-27` | 文書URLの版だけ固定、実APIヘッダー版なし、asset取得が302 | 公開条件未成立。URLや200だけの実装で取得成功/固定API採用を認定しない |
| `A11-28` | catalogに証拠欠損/同ID重複、参照は同じだが収集後の実体変更 | ACCEPT_EVIDENCEまたは証拠失効で集計/公開不可。純粋関数の暗黙I/O・参照だけの実測認定なし |
| `A11-29` | 第一pageだけ一致し次page未確認、途中uploadの部分map、取得中にrelease ID変更 | RELEASE_SET/RELEASE_CHANGEDで公開停止。部分列挙・名前一致・二回観測を完全配置や排他の証明にしない |

### 受入カタログの閉包と更新

ExpectedCase.artifact_refは当該行が検証する一つの固定成果物を指す候補。期待仕様のexpected_refとは別で、artifact_refのcommitはimplementation_commitへ、実体hash/環境/Scopeは検証済み証拠catalogへ照合する。AcceptanceRow.artifact_sha256はこのsha256を使い、pass/failのActualCaseも同じ実物へ一致させる。not_runは「期待する実在成果物のhash」であって、実行証拠があるとの意味ではない。未生成の本体を必要とする場合は製品Scope構築前の未達として保持し、0のhashや文書hashで補完しない。CaseCatalogからの展開は設計行の生成までとし、成果物の収集/束縛が済む前の行を検証済みExpectedCaseと呼ばない。

| ケースID | 固定入力 | 期待結果 |
|---|---|---|
| `A11-34` | 仕様証拠だけ存在し成果物参照なし、別commitのartifact、not_runで実行証拠なし | 前二件はScope構築拒否。第三は実在する期待成果物への参照だけ保持し、未実施を実測passへ変換しない |

ExpectedCaseDraftの生成入力候補をCaseBindingの固定集合とする。全キーはcase_id・source（ContractRef）・requirement_ids（既存要件IDの空でない一意配列）・contracts（検証対象ContractRefの空でない一意配列）・lanes・expected_ref。lanesの各要素はarchitecture/profile/entry/environment_ref/execution_class/required/applicability_refの七キー。execution_classはdocument/pure/adapter/vm/hardwareの一つで、CPU・構成・入口は本書の対応表へ照合する。applicability_refは適用/拒否/非適用を区別する固定根拠であり、能力未確認や未実施を非適用へ変換しない。

```rust
pub enum CaseExecutionClass { Document, Pure, Adapter, Vm, Hardware }
pub struct CaseLane {
    architecture: String, profile: Option<Profile>, entry: String,
    environment_ref: String, execution_class: CaseExecutionClass,
    required: bool, applicability_ref: ContractRef,
}
pub struct CaseBinding {
    case_id: String, source: ContractRef, requirement_ids: Vec<String>,
    contracts: Vec<ContractRef>,
    lanes: Vec<CaseLane>, expected_ref: EvidenceRef,
}
pub struct CaseCatalog { schema_version: u32, specification_commit: String, bindings: Vec<CaseBinding> }
pub struct ExpectedCaseDraft {
    requirement_id: String, contract: ContractRef, case_id: String, lane: CaseLane,
    expected_ref: EvidenceRef,
}
pub fn validate_case_catalog(input: &Value, sources: &[ContractRef])
    -> Result<CaseCatalog, ContractError>;
pub fn expand_expected_cases(catalog: &CaseCatalog)
    -> Result<Vec<ExpectedCaseDraft>, ContractError>;
pub fn bind_acceptance_scope(drafts: Vec<ExpectedCaseDraft>, artifacts: &[VerifiedEvidence],
    environments: &[VerifiedEnvironment], identity: ScopeIdentity)
    -> Result<AcceptanceScope, ContractError>;
pub struct VerifiedEnvironment {
    environment_id: String, architecture: String, entry: String,
    profiles: Vec<Option<Profile>>, execution_classes: Vec<CaseExecutionClass>,
    evidence: EvidenceRef, verification_ref: String,
}
pub struct ScopeIdentity { scope_id: String, specification_commit: String, implementation_commit: String }
```

CaseCatalogの全外部キーはschema_version: 1・specification_commit・bindings。検証/展開/束縛のAPIは検証済み参照と入力値だけを扱う純粋処理で、ファイル取得・時計取得・実試験を開始しない。sourceは担当受入行を含む節の固定参照、expected_refは期待内容の資料であり実結果ではない。lanesは空不可、同じ七値の重複を拒否し、同じ結合キーで根拠/requiredだけ違う行も競合とする。意味を未確認のままケース名の単語からclass/laneを自動認定しない。

| ケースID | 固定入力 | 期待結果 |
|---|---|---|
| `A11-32` | 同一laneのrequiredだけ相違、空lanes、実結果から作ったexpected_ref | ACCEPT_SCOPE/ACCEPT_EVIDENCE。重複の後勝ち・空直積の完了化・期待と実結果の循環なし |

検証順序は、担当本文に現存する全ケースID集合との完全一致→ID一意性→source/expectation/要件/契約の参照閉包→execution_classと環境の適合→全lane→ExpectedCase展開→結合キー重複と必要経路の不足、とする。展開は各Bindingのrequirement_ids×contracts×lanesだけを使い、全製品の無条件直積を作らない。ケースが複数の要件/契約を扱う場合も省略せず別行へ対応付ける。構文/純粋fixtureの実行成功を同じケースのVM/実機成功へ転写しない。

仕様にケースを追加したときは、case_idをカタログへ追加してlane/期待参照を固定するまでscopeは不成立。改定時は旧新の集合差・影響する契約/資材/環境と証拠失効範囲を照合し、既存のpassを名前だけで再利用しない。上限候補はCaseBinding 4,096件、展開後ExpectedCase 100,000行、入力全体16 MiBとし、P02の通常1 MiB解析と別に予算を宣言する専用の有界収集入力とする。超過時は停止し、自動間引きやページ分割によるscope縮小をしない。

専用入力は[P02-PARSEのparse_acceptance_json](PHASE_02.md#解析の入力と上限)へ一回だけ渡し、次に選択済みのCatalog/Scope validatorへ対応付ける。rootの自己申告で一般設定の用途を切り替えない。構文解析後の展開も同じ15秒の外側予算へ含め、行生成前のchecked乗算/加算で100,000行と論理512 MiBを検査する。入力byte/ノード/展開行の全上限を独立に満たす必要があり、100,000行が常に16 MiBへ収まるという保証ではない。超過時は部分scopeを返さない。

| ケースID | 固定入力 | 期待結果 |
|---|---|---|
| `A11-31` | 1 MiB超/16 MiB内のCatalog、同byte列を通常設定へ、展開100,001行 | 専用profileだけ解析可能、通常設定と行超過は拒否。汎用上限緩和・自動間引きなし |

本項はカタログの全形式・展開/欠損検出の契約案である。[全ケースの実行割当](#全ケースの実行割当案)は設計候補として列挙し、実対象のenvironment_ref/成果物/採用済み期待資料への束縛は未完了とする。実施済みのAcceptanceScopeを捏造しない。D11-01の残りは環境/証拠の成立根拠と固定集合の採用判断を含む。

| ケースID | 固定入力 | 期待結果 |
|---|---|---|
| `A11-30` | 新ケースのBinding欠損、同ID二行、VMケースをpureだけへ結合、展開100,001行 | ACCEPT_SCOPEで停止。反復数・件数一致だけで閉包/VM受入を認定せず、期待集合の自動間引きなし |

## 未決事項と成立判定

### 全ケースの実行割当案

下表を全受入ケースの実行区分/laneの設計上の固定集合とする候補。NNは担当ファイルPHASE_NN.md、番号はA NNの末尾（実IDはA01-01等）、a..bは両端を含む整数列、commaは和集合、+は必要な全profileの和。入力語の推測分類ではなく、各行を明示列挙した集合を使う。同じケースを二行へ重複登録しない。

期待結果は担当ファイル内の同じcase_idの前提/入力/期待結果の全セルが唯一の正本。提出commitでその一行と所属節/担当契約・要件を束縛し、期待資料の元byte列/hashを外側の確認資料で固定する。本表へ期待文を複製せず、テスト実出力から期待値を生成しない。行の一部だけの一致はpassにせず、正常/異常/境界の各入力を全て照合する。

| profile | 実行区分・展開するlaneと環境の設計 |
|---|---|
| D | document。両CPU/internal/profile=null。固定した全文書/manifest corpus、文書/依存scanner専用環境 |
| P | pure。両CPU/internal/profile=null。入力byte/値/試験時計/期待木を固定し、OS実変更なし |
| A | adapter。両CPU/internal/profile=null。採用版Ubuntuの使い捨て環境、固定OS実行物/失敗writer/専用FS/模擬接続先。実hostや本番資格は使用しない |
| U | vm。両CPU/internal/profile=null。専用Ubuntu VMでworker/監督/cgroup/FS障害/再起動を確認。物理ディスクなし |
| V | vm。mediaの対応表と下記担当範囲の交差。amd64 UEFI/BIOSとarm64 UEFIを分離し、担当する起動/導入/媒体切断を検査 |
| R | vm。ssh_existingの対応表と下記担当範囲の交差。固定host鍵/非root資格、既存OSと切断/権限不足を隔離再現 |
| S | vm。ssh_bootstrapの対応表と下記担当範囲の交差。空/使用中テストdisk、一時SSH環境、消去外journalと新OS起動を隔離再現 |
| H | hardware。下記担当範囲内で個別指定した機器。機種/版/安全値と試験許可が未決なら未束縛。VM/模擬値で実機対応を証明しない |
| L | adapter。初期配布基盤の隔離した試験先と試験資格。実API版/権限/閲覧/asset再取得の照合用。本番署名/正式公開を行わない |

全profileは期待する環境の要件であり、実在するenvironment_refではない。CPU/構成/入口ごとの環境、拒否入力と正本のapplicability、成果物を採用・収集してからCaseLaneへ展開する。D/P/A/Uのprofile=nullは共通処理を対象にする区分で、入力fixture内の構成をnullへ変更する意味ではない。V/R/Sの拒否laneは非適用で消さず、拒否が期待結果の必須試験にする。

| NN | ケース番号集合 | 必須profile |
|---|---|---|
| 01 | 01..08,14..16,18,21..23,26,30,32 | P+A |
| 01 | 09..13,17,19..20,24..25,27..29,31,33..36 | D+P |
| 02 | 01..06,10,41..79,81..84,86..103,105..110,113..116,118..122 | P+A |
| 02 | 07..09,80,85,104,111..112,117,123..124 | P+A+U |
| 02 | 11..40 | P |
| 03 | 01..03,05..10,12..17,19..20,22..23,25..31,33..37 | P+A |
| 03 | 04,11,18,21,24,32 | P+A+U |
| 04 | 01..04,07..10,12,14..15,17..18,20,25,29..30,33..34,42..44,46 | P+A+U |
| 04 | 05..06,11,13,16,19,21..24,26..28,31..32,35..41,45,47..49 | P+A |
| 05 | 01..02,15,20..21,25,34,37..38 | P+A |
| 05 | 03..08,10..14,16..19,22..23,26..27,29..33,35..36,39..41 | P+V |
| 05 | 09 | P+V+H |
| 05 | 24,28 | P+H |
| 06 | 01..02,04,06,09..12,14..17,20..23,27..30,32..33,35 | P+A |
| 06 | 03,05,07..08,13,18..19,24..26,31,34 | P+A+V+R |
| 07 | 01..03,05,07,23..24,35,40 | P+A+H |
| 07 | 04,06,08,12..14,18,22,26..34,36..39,41 | P+A+U |
| 07 | 09..11,15..17,19..21,25 | P |
| 08 | 01..02,07,21,31 | P+A+V+H |
| 08 | 05,08,16,22,25,32,35 | P+A+V+R+S |
| 08 | 12..13,18,26,28 | P+A+V |
| 08 | 03..04,06,09..11,14..15,17,19..20,23..24,27,29..30,33..34 | P+A |
| 09 | 03,16,26..27 | P+A+R+S |
| 09 | 01..02,04..15,17..25,28..39 | P+A+R |
| 10 | 11,17,21,28,35 | P+A+S+V |
| 10 | 01..10,12..16,18..20,22..27,29..34 | P+A+S |
| 11 | 04..05,11..12,23 | D+P+A+V |
| 11 | 07,15,18,21,27,29 | D+P+A+L |
| 11 | 01..03,06,08..10,13..14,16..17,19..20,22,24..26,28,30..37 | D+P+A |

| ケースID | 固定入力 | 期待結果 |
|---|---|---|
| `A11-36` | 任意一ケースの割当欠損/二行重複、VM必要なのにPだけ、期待セルの一部を省略 | 設計集合またはScopeを拒否。件数だけの全件認定・実行区分の降格・部分一致のpassなし |

環境の具体的固定情報が揃わないprofileは設計割当のまま未束縛とする。source/要件/担当契約を提出commitに束縛するときは、本表全件と担当行の双方を照合し、複数責務を扱う入力の検査を省略しない。本表は将来の実行設計であり、いずれのケースも今回実行した製品テストではない。採用済み環境/契約/期待資料が揃ったCaseCatalog/Scopeの成立判定とは分離する。

| ケースID | 固定入力 | 期待結果 |
|---|---|---|
| `A11-37` | 全担当IDを展開した設計割当、H/L環境未選定、後から新ケース追加 | 全件の過不足を照合し未選定は未束縛。新ケースの割当までScope不成立。設計509件を試験509件成功へ転用しない |

### 設計行から実行Scopeへの束縛

仕様策定の設計完了条件と製品Scopeの準備条件を分ける候補。CaseCatalogは承認対象の仕様入力/期待値/適用範囲/検証方法を固定する設計資料で、将来の本体/ISOや実試験結果が未生成であることだけでは設計の欠落にしない。ExpectedCaseDraftは未束縛のまま設計として審査できる。ただし適用範囲/入力/期待結果/実行方式が未定義なら設計自体が未完了。

製品受入を開始するScopeだけが実在成果物とVerifiedEnvironmentを必要とし、束縛できない段階は「受入準備未完了」である。これを採用済み仕様の欠陥や試験のfailへ自動変換しない。not_runは構築済みScopeの行へだけ付け、未生成artifactに架空hashを付けない。A11-02/34/35に設計済みだが未生成の成果物と、設計自体が欠ける場合の別判定を含める。

実行profileは下記の担当範囲で絞る候補。CPU/構成/入口の全直積を各ケースへ複製しない。表の「対応集合」はP11-ACCEPTの成功経路表だけを参照する。拒否入力は担当ケースに明示された対象/不正条件だけを別fixtureへ束縛し、成功laneの対象構成を勝手に不正構成へ変えない。

| 担当ケース・profile | 成功対象として束縛できる範囲 |
|---|---|
| Phase 5のV/H | mediaの対応集合。Hは機種/版/操作を個別指定した物理対象だけ |
| Phase 6のV | mediaのserverとmaster。arm64はserverのみ |
| Phase 6のR | ssh_existingのserverのみ。マスター上のserver機能の試験をCloud master導入にしない |
| Phase 7のH | amd64のminingと物理master。arm64の採掘成功を生成しない |
| Phase 8のV/H | amd64の物理master自身。管理先の構成をマスター自身の構成へ混同しない |
| Phase 8のR/S | 表で指定した遠隔照合ケースだけ。制御元は物理master、laneは操作先のserver/miningの対応集合 |
| Phase 9のR/S | 対応する遠隔入口のserver/mining集合。install_bodyとinstall_osの成功工程を混同しない |
| Phase 10のS | ssh_bootstrapのserver/mining対応集合 |
| Phase 10のV | 共通InstallPayload/target-prepareを使うmedia対応集合。Cloud journal/SSH処理を媒体へ要求しない |
| Phase 11のV | media対応集合の成果物/起動/欠損検出。下位の全運転試験を一ケースへ集約しない |

契約が対象としないprofileを割り当てた行はACCEPT_SCOPEで拒否する。sourceの全入力枝と期待結果を確認し、複数責務のケースは別lane/fixtureの証拠を揃える。applicability_refにはその担当節と成功/拒否の適用根拠を結合し、未確認を非適用にしない。A11-23/32/36へP08構築のarm64/server割当、P06のmining割当、P07のarm64成功、Cloud master拒否入力の欠損を含める。

CaseLaneのexecution_classは[カタログ形式](#受入カタログの閉包と更新)へ従い、同じケースでも純粋評価とVM/実機で確認する責務を別laneにできる。CaseBinding直下に重複したclassは置かない。七キー/型を検証し、classと環境証拠が不一致ならACCEPT_SCOPE。

expand_expected_casesの返却はExpectedCaseDraftだけで、artifact/実環境の存在を示さない。bind_acceptance_scopeは固定spec/implementation/scope、検証済み期待資料、VerifiedEnvironment、当該成果物を照合して初めてExpectedCaseを所有するScopeを返す。artifactの一意な対応はケースの担当成果物とlaneから求め、複数候補/未生成/環境未選定はACCEPT_EVIDENCE。I/O・取得・テスト開始は0件。

同じ環境がpure/adapterの両方を実行できても、execution_classはExpectedCase/ActualCase.key/AcceptanceRowへ無変更で保持し、別行として照合する。VerifiedEnvironmentのCPU/入口/構成/実行区分は実環境の確認資料へ束縛し、呼出元が許可集合を自己申告して作らない。別classのpassを転用せず、各実行で出力/副作用と担当期待条件を検証する。この保持もA11-35の合否へ含める。

解析/validator/展開/束縛の予算は専用profile受付時から一つの15秒deadlineへ接続する。純粋処理の公開APIは時計を直接取得せず、実行wrapperが入口/返却前と内部4,096処理単位の計数点へ試験時計を注入する。製品CLIで予算変更は不可。展開完了後に新たな15秒を足さず、100,000行/論理512 MiBも全段階で累計する。未束縛の案行を製品Scope/試験結果へ出さない。

| ケースID | 固定入力 | 期待結果 |
|---|---|---|
| `A11-35` | pureとVMの同ケース、未生成artifact、解析14秒+展開2秒 | laneを分け、未束縛を保持、共通期限で拒否。ExpectedCaseの早期生成・環境名だけの実在認定・予算再起算なし |

| 判断ID | 必要な判断・確認 | 影響 |
|---|---|---|
| `D11-01` | [受入カタログ](#受入カタログの閉包と更新)と[全ケース割当](#全ケースの実行割当案)に設計案あり。[Scope束縛](#設計行から実行scopeへの束縛)の実環境/成果物/採用済み期待資料、検査実行体/予算の成立根拠と採用判断が残る | 仕様充足・統合受入 |
| `D11-02` | ReleaseDestinationと全asset照合に案あり。固定HTTP API版・全操作/権限・draft閲覧条件と公開予算の成立根拠が残る | 配布・公開 |
| `D11-03` | 六審査区分の対応案あり。正式許諾・個別採用情報・鍵/実機条件は担当正本で所有者が別途決定。本文作成承認を公開許可にしない | 正式署名・公開 |

Phase 1〜10の全必須契約・判断の採用承認と仕様充足が未完了であり、本フェーズも完全仕様未成立。初期CI・製品テスト・バイナリ/ISO生成・署名・公開は行っていない。
