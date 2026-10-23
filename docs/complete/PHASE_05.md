# Phase 5 完全仕様案: 外部媒体・ISOと物理PCの導入・保守

[文書一覧](../DOCUMENT_INDEX.md) | [担当工程](../ROADMAP.md#phase-5-外部媒体isoと物理pcの導入保守基盤)

状態は採用未承認の`B13`。[案の適用範囲](PHASE_01.md#案の適用範囲)を参照する。担当要件は`REQ-002`・`REQ-003`・`REQ-005`・`REQ-006`・`REQ-007`・`REQ-008`・`REQ-009`・`REQ-010`・`REQ-011`・`REQ-012`。既存の[外部媒体手順](../BASIC_SPEC.md#外部媒体による導入と保守)と[起動確認](../BASIC_SPEC.md#内蔵osの起動確認)は変更しない。

## 契約と参照

| 契約ID | 担当 |
|---|---|
| `P05-ISO` | 固定資材からのISO生成と起動 |
| `P05-MEDIA` | 操作選択・対象検査・OS導入の開始 |
| `P05-HANDOFF` | Live環境と対象OSの引継ぎ・中断・取り外し |
| `P05-FIRMWARE` | 機種別操作の選定と結果検証 |

共通参照（全フェーズB13）は、`P02-MODEL`・[識別](PHASE_02.md#p02-model)、`P02-STORE`・[記録](PHASE_02.md#p02-store)、`P02-EXEC`・[外部実行](PHASE_02.md#p02-exec)、`P03-FS`・[パス](PHASE_03.md#p03-fs)、`P04-SELF`・[本体管理](PHASE_04.md#p04-self)、`P04-BACKUP`・[保全](PHASE_04.md#p04-backup)、`P04-APPLY`・[適用](PHASE_04.md#p04-apply)。構成機能の完成はPhase 6〜8と全体受入へ接続する。

## P05-ISO

既存の[生成定義](../BASIC_SPEC.md#isoの作成)、[論理構成](../BASIC_SPEC.md#外部媒体の論理構成)、[媒体マニフェスト](../BASIC_SPEC.md#媒体マニフェスト)を正本とし、recipeの未定義部分を具体化する案とする。

| 入力案 | 必須条件 |
|---|---|
| ベース資材 | 正確な公式取得先・版・ハッシュ・署名検証情報と対応する例外ID。浮動参照不可 |
| 本体一式 | 固定版の署名付き本体・マニフェスト・ビルド情報。当該CPUと版が一致 |
| 生成環境 | 固定digestの環境識別、ツール一覧と各固定版。未登録・未検証の構成要素は使用不可 |
| 出力先 | 未使用の作業ディレクトリ。生成済み正式成果物を上書きしない |

採用案では生成開始時に全入力の識別集合を固定し、生成後にISO内の本体・メタデータを再取得して比較する。ISO生成の上限は3,600秒、再試行なし。途中失敗のISOをマニフェストへ登録せず、正式配布物に昇格させない。

起動用unitから選択処理を開始するまで、本体起動・検証に外部通信を使わない。[起動と取得の保護](../SECURITY_POLICY.md#usb起動と取得)は参照のみとし、独立した安全規定を作らない。Subiquityの開始抑止・実行時設定の受渡しは、採用するベースISOとの実際の起動契約を確認するまで`D05-01`として保留する。

## P05-MEDIA

採用案の選択処理は、媒体の信頼確認、対象候補の列挙、状態の検査、操作の選択、必要入力、構成選択、保全・計画確認、適用前再照合の順とする。[状態別分岐](../POLICIES.md#usb起動時の分岐)を勝手に拡張せず、取り込みは明示された操作として扱う。

ディスク一覧には実行環境を支える機器と候補機器を分離して表示し、除外根拠・安定識別・使用状態を返す。消去確認は[既存の表示・入力](../BASIC_SPEC.md#ディスク消去の確認表示)へ接続する。追加の候補は、確認入力の有効期間を計画と同じにし、対象が変われば失効させる方式とする。

OS導入は対応版Subiquityへ確認済み設定を渡す案とし、本書で独自のインストーラーへ切り替えない。上限は3,600秒、再試行0回。導入途中の失敗時は実ディスク・作成済み資源・導入記録を照合し、自動的な全消去・再導入を行わない。ストレージ配置・管理ユーザー・ネットワーク入力の完全スキーマは`D05-02`の未決事項。

## P05-HANDOFF

引継ぎ情報の採用案は、媒体操作ID、計画ID、対象ディスク・対象ルート識別、対象側操作ID、固定本体ハッシュ、実施工程と必要な再起動・検証の集合とする。Live環境の本体導入情報やlockfileを対象の記録へコピーしない。

| 段階 | 続行条件・結果案 |
|---|---|
| 準備 | Live側の対象識別・媒体側排他と、対象側の処理開始条件を照合 |
| 対象OS内の処理 | 対象用のExecutionContextを作り、対象の共通ロックを使用。Live側は同じ対象への別変更を開始しない |
| 再起動前 | 実施位置・保存先・固定計画を照合し、対象側に必要な継続情報を同期 |
| 取り外し前 | 関連書込み・同期・対象側処理の終了を確認し、媒体を使用するマウント・処理を照合して取り外し可否を表示 |
| 媒体非接続での再起動後 | 内蔵OSの対象同一性・実配置・操作記録・サービス結果を取得。再起動要求だけでは完了にしない |

再起動後の確認待機上限は600秒、状態照合間隔5秒を候補とする。時間超過は未確認であり、同じ変更を再投入しない。安全な取り外しの前提が不成立なら取り外し可能と表示しない。

Liveからサービス・パッケージ処理へ移る実行方式、次回起動時の継続処理の入口、媒体と対象側の排他引継ぎ、結果の回収と保持先は`D05-03`で比較・決定する。chroot実行だけで稼働中systemdへの操作や次回起動検証を完了したとは扱わない。

引渡しの排他切替候補は、Live側のpreboot工程を終了/同期→継続情報を対象へ確定→媒体参照解消と取外し→内蔵起動後workerのguard再取得→同じplan/操作対応と実位置の照合、の一方向とする。再取得前は対象サービスの変更を開始しない。target_started/target_verifiedはpreboot工程の位置であり、内蔵OSのsystemd稼働や全サービス検証済みを意味しない。HandoffRecord.target_boot_idは内蔵起動後だけ非null。旧guardの所有権をコピーせず、記録に中断が残る場合は通常setupを起動せず継続照合専用の入口へ渡す。推奨は内蔵OSの一時継続unitで、具体的unit/権限は採用待ちとする。

## P05-FIRMWARE

候補の照合キーはメーカー・機種・ファームウェア版・起動方式・操作インターフェースの組とする。項目ごとに検出、現在値、予定値、公式呼出し、読み戻し、再起動要否を結合して計画する。操作上限30秒、再試行0回を候補とする。

実際の機種・公式手段・版・項目の値域・検証資材は未選定。新しいメーカーCLIを使用可能にせず、[既存の操作制約](../SECURITY_POLICY.md#ファームウェア操作)と[試験環境の未確定条件](../SECURITY_POLICY.md#検証環境の未確定条件)を維持する。実機操作をVM結果で置き換えない。

機種契約の候補FirmwareContractはmanufacturer・model・firmware_version・boot_mode・item_id・interface_ref・read・write・verify・allowed_values・reboot_requiredを全キーとする。識別は完全一致、interface_refはMaterialRef、read/verifyは採用済み構造化コマンド参照、writeだけ変更不可時null、allowed_valuesは空でないstring[]、reboot_requiredはboolまたは未確認null。読取りと変更の許可・試験境界はセキュリティポリシーへ照合する。第一候補は対応行のある公式手段だけを呼ぶ方式、対応行のない機器は手動案内。未知機器を近似モデルの行へ割り当てる案は採らない。

read/verify/writeの候補参照はCommandSpecの固定テンプレートと期待出力形式・期待終了のContractRefへ一意に結合する。検査で得た値がallowed_valuesにない場合はFIRMWARE_VALUE、不明な出力はFIRMWARE_UNKNOWNとし、変更を開始しない。write=nullは変更非対応の確証、read不能は確認不能として分ける。変更要求0でもverify不一致ならfailed。reboot_required=nullは再起動不要の意味でなく、完了に必要な追加確認を表示する。値の読取り・変更・再起動後の読戻し・復電実動作を別evidence_refで追跡し、実機試験の許可境界は担当ポリシーに維持する。

## 内製境界・API・互換性

配置候補は`src/usb/`と`src/platform/firmware/`。選択・対象識別の調整と引継ぎ制御を内製し、OS導入・機種別操作は選定済み例外だけへ委ねる案。媒体生成・署名そのものの新規内製化は今回決めない。

| 宣言案 | 契約 |
|---|---|
| `media_plan(selection: &MediaSelection, targets: &[TargetEvidence], identity: PlanIdentity) -> Result<PlanDraft, ContractError>` | 事前生成した計画識別と検証した選択から未封印計画を所有して返す。変更・hash実行なし、MediaRequestは封印と計画確認後に構築 |
| `handoff(ctx: &mut ExecutionContext, request: &MediaRequest) -> ChangeResult<HandoffRecord>` | preboot位置・継続情報・対象結果を保持。開始後失敗でもpartialを返す |
| `removal_status(record: &HandoffRecord) -> Result<RemovalResult, ContractError>` | 書込み・マウント・関連処理を観測して照合。取外し済み・内蔵起動済みと捏造しない |

型・エラーはP02-MODEL。引継ぎpayloadはD05-03で全項目を確定する必要がある。難易度は高。初期モジュールの新設案で旧実装はなし。将来の抽出では、媒体構造・handoff形式・CLI・対象記録の互換性を同じ契約テストで比較し、旧形式の除去前に媒体と対象の復旧経路を確認する。

## データ契約の具体化案

`P05-ISO`の生成入力と`P05-HANDOFF`の工程を具体化する審査用候補。実際のベースISO・ツールの個別採用値は[例外台帳](../DEPENDENCY_EXCEPTIONS.md)だけで管理する。

| 型 | 全キー・値域の候補 |
|---|---|
| RecipeInput | schema_version: 1、architecture: 既存CPU値、base_ref: MaterialRef、tools_ref: MaterialRef[]、environment: P01-ENVのEnvironmentBinding、binary_sha256・binary_manifest_sha256: string、recipe_revision: 1以上integer、ram_required_bytes・ram_reserve_bytes: 正整数。MaterialRefは[型の詳細](#型とapiの詳細案)で定義 |
| MediaSelection | action: install・adopt・update・backup・recover、target: TargetEvidence、selection: P02-STATEのSelectionまたはnull、backup_destination: stringまたはnull、confirmation: ConfirmationInputまたはnull |
| HandoffRecord | schema_version: 1、media_operation_id・target_operation_id: OperationId、plan_id: PlanId、target: TargetEvidence、binary_sha256: string、stage: 下記段階、target_receipt: OperationReceiptまたはnull |
| RemovalResult | media_operation_id: OperationId、writes_complete・sync_complete・target_stopped・media_unmounted: booleanまたはnull、removable: boolean、checks: Check[] |
| FirmwareResult | item_id: string、status: success・failed・unsupported・unknown、before・after: stringまたはnull、reboot_required: booleanまたはnull、evidence_ref: stringまたはnull |

MediaSelectionは操作別に入力を検証する候補で、install/adoptは初回Selectionが必須、継続管理のupdate/backup/recoverでは保存済み構成を参照する。backupだけ保存先の選択を必須とし、更新内保全の保存先は親計画の担当入力で検証する。対象の再同定で前提が変わればconfirmationも破棄し、同じ文字列の再利用で新対象を確認済みとしない。操作の列挙は媒体内の入力モデル案であり、新しい共通CLIの採用ではない。

HandoffRecord.stageはprepared・target_started・target_verified・ready_for_removal・boot_pending・complete・failed・unknownを候補とする。target_started以降で応答がなくなった場合はunknownで、対象側の実適用・記録を取得するまで次の適用へ進まない。target_receipt=nullを対象未実行の証明にしない。再起動で対象側operation_idを新規生成せず、同じ親計画の継続であることを照合する。

RemovalResultの4条件は、確認できたtrue/falseと確認不能nullを区別する。removable=trueには全条件trueと必須checksの成功が必要。これは媒体を取り外せる判定であり、取り外した事実や次の内蔵OS起動成功を意味しない。FirmwareResultは値の読み戻しと復電の実動作を別evidenceへ分け、未確認のreboot_requiredをfalseで埋めない。

OSインストールのストレージ・ユーザー・ネットワーク設定、媒体選択画面と起動unitの正式入口、stage別の保存先・排他の取得と解除は`D05-01`〜`D05-03`で確定する必要がある。単なる工程表を、Subiquityやsystemdの実契約・動作証拠の代替にしない。

## 接続と責務の照合

取り外しと媒体なしの起動確認を循環した前提にしない。採用候補の順序は、Live側の書込み工程の検証、内蔵OS側への継続情報の同期、対象のマウント・関連処理の終了確認、媒体の取り外し、内蔵OSの起動・継続処理、最終検証と結果確定とする。target_verifiedは取り外し前に実施した工程の検証に限り、内包サービスや媒体なし起動の検証完了ではない。

内蔵OS側の継続処理は同期済みの同じ対象・計画・操作対応を読み、共通排他を再取得して実状態を照合する候補。Live側のロックハンドルやプロセスIDを起動後の所有証拠へ転用しない。継続処理が追加の選択・破壊的確認を必要とする場合は、既に受けた確認で代行せず停止する。起動用unit・継続情報の全形式・信頼・解除条件は`D05-03`のまま未確定とする。

媒体なしの起動確認は対象内蔵OSで結果を保持・表示し、結果回収のためだけに媒体の常時接続・再接続を要求しない。物理PCをSSH保守へ切り替える代替経路も設けない。[P08-MASTER](PHASE_08.md#p08-master)の構成全体の完了はこの最終結果を照合し、RemovalResult.removableを構築完了として使用しない。

## 受入ケース案

| ケースID | 区分・前提 | 期待結果・実施方法 |
|---|---|---|
| `A05-01` | 正常。固定した本体一式からISO生成 | ISO内の本体ハッシュと元の本体が一致。別ビルドなし |
| `A05-02` | 異常。資材不足、署名不正、CPU不一致 | 生成または起動を停止。未検証版へのfallbackなし |
| `A05-03` | 正常。外部通信なしで媒体起動 | 本体・選択入口まで到達。必須選択前の対象書込み0 |
| `A05-04` | 異常。未選択、取り込み未確認、実行中機器を指定 | 変更開始なし。対象候補・除外理由を照合 |
| `A05-05` | 異常。Liveと対象で同じパス、OS導入途中中断 | 対象識別を保持し、Live側の記録を対象へ転用しない |
| `A05-06` | 異常。書込み中・マウント中・終了確認不能 | 取り外し不可または未確認。成功通知で打ち切らない |
| `A05-07` | 正常。VMの媒体を外して起動・再起動 | 内蔵OS・通常サービスが媒体に依存せず復帰 |
| `A05-08` | 境界。再起動待機期限、前提の変化 | 期限超過・対象変更は照合待ち。無確認再投入なし |
| `A05-09` | 異常。未対応機種、Secure Boot未確認 | 対応済み表示なし。禁止された設定変更なし |

## 追加の受入ケース案

`P05-HANDOFF`の段階と結果を検証する設計。媒体起動・取り外し・起動試験は専用VMのテスト用ディスクで実施し、実ホストの設定変更を伴わせない。

| ケースID | 区分・前提と入力 | 期待結果・照合方法 |
|---|---|---|
| `A05-10` | 正常。取り外し前の全条件true、内蔵OS起動は未実施 | removable=trueでも全体completeにしない。起動確認の証拠を別に要求 |
| `A05-11` | 異常。sync_complete=null、他条件true | removable=false。未確認を同期済みと補完しない |
| `A05-12` | 正常。取り外し後に同じ計画の継続、媒体非接続で起動・再起動 | 対象内蔵OSの結果へ結合し、媒体・Live側PID/lockfileへ依存しない |
| `A05-13` | 異常。target_startedで応答欠損、target_receipt=null | 未実行へ戻さずunknown。再導入・再消去0回 |
| `A05-14` | 異常。継続時に別ディスクへ交換、または新しい必須確認が発生 | 対象・前提を再照合して停止。過去の確認文字列で代行しない |

## 型とAPIの詳細案

資材の採用情報と仕様へのContractRefを混同しない候補。ISOとCloudで共通の資材識別型MaterialRef/ReleaseArtifactRefの宣言正本は本節、実装配置候補はsrc/core/model.rsとする。Coreや共通実行器からusb/profilesへ依存させない。版・ライセンス・承認の現在情報は例外台帳が正本で、型の全フィールドがあるだけでは使用可能にしない。

```rust
pub struct MaterialRef {
    exception_id: String, artifact_ref: String, sha256: String,
    bytes: u64, architecture: String,
}
pub enum MediaAction { Install, Adopt, Update, Backup, Recover }
pub struct ConfirmationInput { plan_response: String, erase_response: Option<String> }
pub struct Confirmation {
    plan_id: PlanId, target_disk_id: String, shown_digest: String,
    plan_response: String, erase_response: Option<String>, confirmed_at: UtcMillis,
}
pub struct MediaSelection {
    action: MediaAction, target: TargetEvidence, selection: Option<Selection>,
    backup_destination: Option<String>, confirmation: Option<ConfirmationInput>,
}
pub struct RecipeInput {
    schema_version: u32, architecture: String, base_ref: MaterialRef,
    tools_ref: Vec<MaterialRef>, environment: EnvironmentBinding,
    binary_sha256: String, binary_manifest_sha256: String, recipe_revision: u32,
    ram_required_bytes: u64, ram_reserve_bytes: u64,
}
pub struct MediaRequest {
    schema_version: u32, media_id: String, action: MediaAction, target: TargetEvidence,
    selection: Option<Selection>, install: Option<InstallPayload>, plan: Plan,
    backup: Option<BackupSet>, confirmation: Option<Confirmation>,
}
pub struct ReleaseArtifactRef {
    release_version: String, binary_manifest_sha256: String, path: String,
    sha256: String, bytes: u64, architecture: Option<String>,
}
pub enum HandoffStage {
    Prepared, TargetStarted, TargetVerified, ReadyForRemoval,
    BootPending, Complete, Failed, Unknown,
}
pub struct HandoffRecord {
    schema_version: u32, media_operation_id: OperationId, target_operation_id: OperationId,
    plan_id: PlanId, target: TargetEvidence, binary_sha256: String, stage: HandoffStage,
    target_receipt: Option<OperationReceipt>, media_id: String, request_sha256: String,
    recipe_sha256: String, target_boot_id: Option<String>, removal: Option<RemovalResult>,
    last_confirmed_stage: HandoffStage,
}
pub struct BootProof {
    media_id: String, plan_id: PlanId, target_operation_id: OperationId,
    disk_id: String, root_identity: String, boot_id: String, observed_at: UtcMillis,
    media_absent: bool, required_services: std::collections::BTreeMap<String, String>,
    checks: Vec<Check>,
}
pub struct FirmwareResult {
    item_id: String, status: String, before: Option<String>, after: Option<String>,
    reboot_required: Option<bool>, evidence_ref: Option<String>,
}
pub struct FirmwareContract {
    manufacturer: String, model: String, firmware_version: String, boot_mode: String,
    item_id: String, interface_ref: MaterialRef, read: ContractRef, write: Option<ContractRef>,
    verify: ContractRef, allowed_values: Vec<String>, reboot_required: Option<bool>,
}
pub struct RemovalResult {
    media_operation_id: OperationId, writes_complete: Option<bool>,
    sync_complete: Option<bool>, target_stopped: Option<bool>,
    media_unmounted: Option<bool>, removable: bool, checks: Vec<Check>,
}
```

MaterialRef.bytesは外部integerの上限をさらに適用し0を拒否する。architectureは既存CPU値、artifact_refは台帳で特定した不変成果物への参照で、任意の取得先へ展開するパスではない。RecipeInputのtools_refには使う実行物を各1件含め、まとめた「ツール一式」の自己申告を受理しない。生成環境のイメージと各実行物の両方を照合する。

ServerSeed自身の本体/manifest/署名を外部例外IDへ偽装しないため、独自成果物の参照候補をReleaseArtifactRefとして分ける。全項目はrelease_version・binary_manifest_sha256・path・sha256・bytes・architecture。前四つは既存の版/hash/配布ファイル名制約、bytesは正の共通整数、architectureは本体なら既存CPU値、それ以外の共通情報はnullとする。独自成果物は署名付きリリース集合との対応、MaterialRefは個別例外台帳との対応をそれぞれ検証する。どちらも名前一致だけを真正性根拠にせず、第三者資材へReleaseArtifactRefを割り当てて台帳照合を省略しない。

RemovalResult.checksはP02-CLIの検証済みCheckだけを保持し、Valueのまま渡さない。必須IDはP05-HANDOFF/writes_complete・sync_complete・target_stopped・media_unmountedの4件で、同名boolのtrue/false/nullをそれぞれpass/fail/unknownへ対応させる。Option<bool>のNoneをfalseと同じ根拠へ潰さず、表示では「未確認」とする。HandoffRecordの実行前情報とtarget_receiptは異なる段階で確定し、target_started後の欠損を未実行へ戻す生成関数を設けない候補とする。4件の欠損・重複・boolとの矛盾をMEDIA_BINDINGで拒否する。

| ケースID | 固定入力 | 期待結果 |
|---|---|---|
| `A05-37` | media_unmounted=nullなのに対応Check=pass、必須4件中1件欠損 | MEDIA_BINDING、removableを生成しない。Value経由の型検査省略・不明の成功化なし |

## 状態と中断の詳細案

MediaSelection/ConfirmationInput/Confirmation/RecipeInput/MediaRequestは各全集合表へ対応する型候補。生成器は未確認入力と検証済み確認を別型で保持し、MediaSelection.confirmation=Noneの検査を省略しない。RecipeInputの整数型に入るだけではRAM根拠・資材/環境の採用条件を満たさない。A05-18/32/41へnull条件の相互違反と未封印型の使用を含め、validatorなしの構築を配送可能と認定しない。

`P05-HANDOFF`の通常順はprepared→target_started→target_verified→ready_for_removal→boot_pending→completeの候補。各遷移は直前段階の記録・対象同一性と次表の根拠を要求し、段階名を直接書き換えて飛び越さない。

| 遷移または事象 | 必須根拠・結果 |
|---|---|
| prepared→target_started | 対象側runningと排他の照合。Live側のPIDを代用しない |
| target_started→target_verified | 取り外し前工程の実結果・記録一致。起動後サービスの検証はまだ含めない |
| target_verified→ready_for_removal | 継続情報の同期とRemovalResultの全条件成立 |
| ready_for_removal→boot_pending | 利用者の取り外しと内蔵OS起動への引渡し。ソフトウェアの許可表示だけで取り外した事実を作らない |
| boot_pending→complete | 同一対象の媒体なし起動・再起動、構成別検証、必須記録確定 |
| started以降で応答欠損 | unknown。対象実状態の照合まで新しい適用を開始しない |
| 検証失敗の確証 | failed。元工程と復旧結果を分離して保持 |

LiveのルートがUSB上のSquashFS等を参照している場合はmedia_unmounted=trueと判定できない。取り外し可能にする方式の第一候補を、必要資材を固定したRAM上の実行環境と対象内蔵OSの継続処理の組合せとして提示する。メモリー容量、ブート設定、RAMへの複製・検証、全参照解除の成立性は`D05-01`・`D05-03`で確認するまで未達であり、単なるumount要求成功を根拠にしない。

boot_pendingの確認結果は内蔵OSへ保持し、結果回収だけのために媒体の再接続を必須にしない。failed/unknownからcompleteへの直接遷移は認めず、明示的な復旧計画・実状態照合と未実施検証を必要とする候補とする。

## 依存と互換性の詳細案

ISOの生成環境、実行するLive環境、導入後の内蔵OSを三つの識別済み環境として扱う候補。ツール版・生成資材はMaterialRef、仕様への参照はContractRefとし、互いを代用しない。

| 境界 | 互換性と未達時の扱い |
|---|---|
| ベースISOと生成ツール | EX-002・EX-008の個別固定参照をrecipeへ結合。浮動版と未選定のunitで生成しない |
| Subiquity受渡し | 実行時だけ確認済み/autoinstall.yamlを生成。無確認の先行インストールを抑止する外部契約が成立しない版は使用不可 |
| Liveから対象へ | 同じplanと対象operation対応を保持するが、install情報・PID・lockのコピーなし |
| 媒体と内蔵OSの形式差 | 読めるhandoff形式版・本体版・対応CPUを先に照合。未知版を古い意味へfallbackしない |
| 機種別firmware | メーカー/機種/版/項目/公式手段を個別に固定。未登録機種は自動変更なし |

新しいISOはP11-RELEASEの固定した共通本体から生成する。古い媒体で新しい管理形式を検出した場合は変更せず停止し、媒体の正規交換へ案内する。独自にmedia内本体だけを更新して互換にする方式を追加しない。

引継ぎ制御をcrateへ抽出する将来案は、媒体起動・取り外し・起動後継続を同じ隔離VMで比較する。メモリーだけの模擬結果は媒体参照解除・ブート互換の証明にせず、旧形式の除去前に既存媒体からの復旧可能範囲を確認する。

## 受入条件の充足照合案

ISOは固定資材/生成/起動、MEDIAは選択/導入入力、HANDOFFは段階/取り外し/内蔵OS、FIRMWAREは機種別外部契約と試験許可へ対応する。MaterialRefの存在と台帳の使用可能判定を別に確認する。

| ケースID | 区分・固定入力 | 期待結果と確認 |
|---|---|---|
| `A05-15` | 異常。RecipeInputのbase_refへ仕様のContractRefだけを指定 | 資材型・台帳対応不正で生成停止。仕様リンクをベースISOとして扱わない |
| `A05-16` | 異常。LiveのrootがUSBを参照、全対象書込みは終了 | media_unmounted未成立、removable=false。umount要求だけで安全取り外し表示なし |
| `A05-17` | 正常・異常。各handoff遷移と、target_startedからcompleteへの飛越し | 必須証拠がある隣接遷移だけ受理。媒体なし起動・再起動を省略しない |

必要なSubiquity/ブート設定、RAM実行環境、継続unitと全handoff保存形式が未確定なら媒体保守は未充足。通常CIのISO内容検査、専用VMでの起動/取り外し、実機の機種別動作はそれぞれ別の証拠とし、実機試験の許可境界が未決のまま実行しない。

## 入力と保存形式の詳細案

P05-MEDIAの入力をMediaRequestとして閉じる候補。対象OSへ渡す導入値は[InstallPayload](PHASE_10.md#型とapiの詳細案)を参照し、物理経路用にstorage/network/user/bootの第二定義を作らない。

共通payloadの検証は[InstallEntry](PHASE_10.md#入力と保存形式の詳細案)をMediaへ束縛して呼び、物理PCにCloud用の再接続先・台帳登録を要求しない。導入資材と実行能力は媒体のrecipeから照合し、Cloud用のinstaller選定を媒体へ流用しない。

| 型 | 全キーと制約 |
|---|---|
| MediaRequest | schema_version: 1、media_id: hex32string、action: MediaSelectionのaction、target: TargetEvidence、selection: Selectionまたはnull、install: InstallPayloadまたはnull、plan: Plan、backup: BackupSetまたはnull、confirmation: Confirmationまたはnull |
| Confirmation | plan_id: PlanId、target_disk_id: string、shown_digest: hex64string、plan_response: string、erase_response: stringまたはnull、confirmed_at: UtcMillis。入力二値はConfirmationInputと同じ名前で、下記の計画確認と追加消去確認を区別 |
| HandoffRecordの補完 | 既存キーにmedia_id: MediaRequestの識別子、request_sha256: hex64string、recipe_sha256: hex64string、target_boot_id: stringまたはnull、removal: RemovalResultまたはnull、last_confirmed_stage: 通常経路のHandoffStageを追加する案 |
| 受渡し保存先 | 媒体側・対象側に同じmedia_id/plan_idのレコードを置き、対象側は候補/var/lib/serverseed/media-handoff.json。書込み先は外部requestから受け取らず固定生成する |

installはaction=installだけ非null、既存OSへのadopt/update/backup/recoverではnull。selectionはinstall/adoptだけ必須、継続管理のupdate/backup/recoverはnullとして保存済み構成とplan.profileの一致を要求する。非nullのselectionはplan/target/profileがplanと一致する必要がある。backupが必要な操作は保全計画参照を省略できず、action=backupはbackup非null。確認は利用者の当該計画への応答で、USB接続や--yesで生成しない。MediaSelectionは画面入力、MediaRequestは検証後の配送値とし、確認文字列の収集結果を検証済みConfirmationに変換してから渡す。

MediaRequestの検証候補順序は形式/版→actionごとのnull条件→target/Plan/Selectionの同一性→保存構成→必須の保全/確認→InstallPayload→操作前提とする。install/adoptはsetup計画、update/backup/recoverはそれぞれ当該操作の計画へだけ結合し、adoptにpartition工程を含めない。Confirmationのshown_digestはPlan.digestと完全一致、target_disk_idは再同定した対象、confirmed_atは計画作成以降かつ期限内を要求する。確認文字列の一致だけで別action・別計画への同意へ転用しない。失敗はMEDIA_INPUTまたはMEDIA_BINDING、選択不足は共通CLIの停止へ対応付け、Subiquity・保存・対象変更を呼ばない。

保守操作は[P02-MODELのMaintenanceAction](PHASE_02.md#入力と保存形式の詳細案)とし、一般CLIのActionとは区別する。MediaActionのinstall/adoptはSetup、updateはUpdate、backupはBackup、recoverはRecoverへ一対一変換する。Plan/operation/historyのcommandは順にsetup/update/backup/recoverを保存する候補で、backupは媒体またはマスターの必須選択からだけ生成する内部操作名である。一般CLIにserverseed backupを追加せず、parse_argsの未知コマンド拒否を維持する。独立backupはupdateを実行せず、共通のbegin/finishとP04-BACKUPを使用する。

| ケースID | 固定入力 | 期待結果 |
|---|---|---|
| `A05-38` | 媒体でbackup選択、一般CLIでbackup指定、backup要求にupdate計画 | 独立保全/未知コマンド/結合拒否を区別。更新への置換・一般CLIの暗黙拡張なし |

確認入力の具体候補ConfirmationInputはplan_responseとerase_responseの二キーだけ。短い確認コードは当該Plan.digestの先頭12桁とし、plan_responseは表示したAPPLYとspace一つとコードの完全一致を要求する。消去を含むPlanだけ追加でERASEとspace一つと同じコードをerase_responseへ入力させ、消去なしはnull。コードは[基本仕様の許す生成確認コード](../BASIC_SPEC.md#ディスク消去の確認表示)の具体化候補であり、シリアル等の必須表示は省略しない。大文字小文字変換・trim・--yes・媒体接続から応答を生成しない。Planが変われば両確認を失効させ、コード一致だけを計画の同一性・認証の証明にしない。

last_confirmed_stageは同期済みの直前の通常段階を保持する候補で、failed/unknownを入れない。stageが通常段階なら同値、failed/unknownでは到達済み位置を残す。応答欠損時にpreparedへ戻して同じsetupを再実行しない。復旧計画は固定request hash・対象・実適用位置を照合して未実施の検証と記録確定だけへ接続し、元の失敗/不明記録を通常成功へ無条件上書きしない。

request_sha256はMediaRequestの全キーをP02-CLIのJSON順/escape・末尾LFなしで固定したbyte列、recipe_sha256はRecipeInput全体の同じ表現へのSHA-256とする候補。どちらも媒体file全体のhashやPlan.digestと代用しない。継続情報はmedia_id・両operation_id・plan_id・二hash・対象の固定識別を一組で照合する。既存media-handoff.jsonが別の未完了操作を指す場合はMEDIA_PENDINGで新規操作を停止し、単一保存先へ上書きしない。完了済みでも履歴との対応を保全してから新しい計画の保存を判断し、fileがあるだけで再開処理を起動しない。

media_idは一回の媒体操作セッションを識別する16byte由来のhex32候補とし、P02の乱数/一意性条件で入口が生成する。ISO版・volume label・USB製品serialそのものではなく、同じISOの別操作へ再利用しない。実媒体のdevice/FS/起動参照集合は別の観測証拠へ束縛し、同一labelや同一ISO hashを理由に他のUSB/namespace参照を解除済みと扱わない。再開では保存済みの同じmedia_idを使い、新規IDで未完了を隠さない。A05-21/30/33/36で同一ISO二媒体、label重複、再開時ID差替えを照合する。

ISO recipeのtools_refはMaterialRefの識別集合を重複なしで固定し、baseと全実行物、同梱本体の実hashをrecipeへ結合する。RAM起動を第一候補とし、採用版で測定した必要量と予約量のchecked合計が観測した使用可能RAM以下の場合だけ起動条件を通す。測定根拠なし・取得不能・overflowは起動保留。boot後のlatest取得や、ツール参照名toolsへの別名置換は行わない。容量条件の成功だけで媒体参照解消を認定しない。

| ケースID | 固定入力 | 期待結果 |
|---|---|---|
| `A05-18` | selectionはdisk A、InstallPayloadはdisk B、confirmationは旧plan | 対象照合失敗。Subiquity起動・ディスク消去・確認の自動生成なし |

## 実行と復旧の確定候補

P05-HANDOFFの候補実行は、媒体上で選択・確認→Live制御下のpreboot導入/検証→対象側継続情報の同期→preboot処理終了と媒体依存解消→取り外し→内蔵起動後の継続worker→再起動確認の順とする。preboot中に行うのは対象へ継続unit/入力を配置する準備であり、そのunitで稼働中systemdの処理を実行済みとはしない。Liveを通常運用の実行環境へ残さない。

Subiquityの[公式入力](https://canonical-subiquity.readthedocs-hosted.com/en/latest/reference/autoinstall-reference.html)へ渡す生成物はautoinstall/version=1を明示し、承認済みrecipeからだけ生成する候補。installer自身の更新を無効にする案と、late-commandを構造化argvへ限定する案を審査する。公式文書の対応構文の確認と採用ベースISOでの実行試験を区別する。任意の利用者shell断片を差し込まない。

prebootの親はその工程のguardを保持し、工程終了・同期後に引き渡す。内蔵起動後の継続workerは自身でguardを再取得して同じ対象/計画/操作を照合し、媒体側のハンドルを継承したと扱わない。Liveと内蔵側がそれぞれ独立してsetupを始める設計にはしない。対象boot_idが変わったら旧PID/lockの観測を新OSへ流用せず、ディスク・root・操作記録の結合を再確認する。

RAM起動案ではSquashFS・loop・overlay・swap・開いたFDを含む媒体参照が解消していることを確認する。対象処理終了、同期、参照解消、unmount確認の全成立前に取り外しを案内しない。実RAM容量不足、媒体参照の解除不能、対象側結果不明は取り外し不可。取り外し後の確認結果は内蔵OSに保存し、USBの再接続を完了条件にしない。

取り外し前のtarget_stoppedはprebootの変更子処理の終了確認を指す候補で、起動後の継続unitや選択画面まで実行済み/停止済みと推測しない。起動確認は最初の内蔵bootと再起動後の異なるboot_idに、それぞれ同じdisk/rootと必要サービス検査を対応付ける。一つのbootの二回の観測を起動/再起動の二証拠として使わない。記録先・観測器の実方式はD05-03で採用判断する。

| ケースID | 固定入力 | 期待結果 |
|---|---|---|
| `A05-19` | late処理応答欠損、USB上のloopだけ残存、対象は内蔵起動成功 | 再setupなし、取り外し判定は未成立。内蔵起動の証拠だけで媒体依存を消したとしない |

## 依存と受入の確定候補

P05-ISOはrecipe生成と外部生成処理を分ける候補。入力型はRecipeInput/MaterialRefを使い、生成器が独自にbase/tool版を選ばない。P05-MEDIA/HANDOFFはMediaRequestと対象側OperationReceiptを受け、媒体の存在を通常運用の依存へ残さない。

| 受入区分 | 必須にするfixture/比較 |
|---|---|
| 純粋処理 | 不整合MediaRequest、確認/plan/diskの組、handoffの隣接/飛越し遷移。生成byte列と拒否位置を比較 |
| ISO内容 | 固定recipeと全MaterialRef、CPU、本体hash、manifest/署名の実byte対応。ISO名一致だけ不可 |
| 隔離起動 | 採用VMのUEFI/BIOS、対応CPU、空/既存disk、外部通信なし、RAM不足、loop残存 |
| 導入と通常運用 | 媒体を外した状態で同一内蔵OSの初回起動と再起動、三構成の該当検証。Live成功で代用しない |
| 機種固有 | firmware読取りと許可設定のメーカー/機種/版別契約。VMでは実機の復電・Secure Bootを成功認定しない |

入力生成器/引渡し評価器は同一package内の内製module、Subiquity・ISO・firmware操作は個別例外。内製部分をcrateへ抽出する際も外部実行物を同梱crateとして取り込まない。新旧には同じrecipe/MediaRequestを渡し、生成設定・拒否・引渡し記録を比較する。ISO生成方法やboot構成が変わったら内容/起動/媒体なし検証を再実施し、純粋比較だけでは旧結果を再利用しない。

| ケースID | 固定入力 | 期待結果 |
|---|---|---|
| `A05-20` | 入力生成器は同出力だがbase ISO/boot設定だけ変更、VM起動pass | 旧取り外し/実機結果を自動流用しない。該当ISOの内容・導入・内蔵起動を再検証 |
| `A05-21` | toolsという別名、RAM必要量の根拠なし、必要量+予約量が観測RAMを1byte超過 | recipe不正または起動保留。本体検証を省略して通常Liveへ切替えない |
| `A05-22` | 継続updateでselection=nullかつ保存構成一致、初回installでnull、継続時に別profile | 前者だけ構成継承の検査を通過。再選択の強制・nullからの構成推測なし |
| `A05-23` | preboot検証済みだが内蔵workerのguard未取得、旧PIDだけ一致 | 起動後のサービス変更0件、継続照合待ち。preboot成功を内蔵OS運転成功にしない |
| `A05-24` | 同じmodelでもfirmware版違い、write=null、読み戻し値が予定値と不一致 | 自動変更不可または検証失敗。別版流用・設定済みと復電成功の混同なし |
| `A05-25` | 独自本体へ架空EX ID、第三者loaderへReleaseArtifactRef、共通manifestへCPU値 | 参照種別/リリース集合不一致で拒否。独自成果物と第三者例外の確認を代用しない |
| `A05-26` | action=adoptにpartition工程、shown_digest相違、confirmed_atがplan作成前 | MEDIA_INPUT/MEDIA_BINDINGで変更0件。確認文字列が同じでも消去や別操作へ転用しない |
| `A05-27` | 継続unit配置済みだがpreboot未終了、次に内蔵起動後guard取得失敗 | 取り外し案内または継続処理をそれぞれ停止。unit配置を実行済みとせず二重setupなし |
| `A05-28` | write終了0だがverifyは旧値、readは未知値、reboot_required=null | 順にfailed/変更前停止/追加確認待ち。要求成功・値の読戻し・復電成功を別に記録 |
| `A05-29` | 媒体preboot後/内蔵guard再取得前、SSH accepted後/running前、OS partition.started片側のみ | 各担当記録と実状態を照合し、新しい変更開始0回。媒体・SSH・一時環境のguardを相互コピーしない |
| `A05-30` | target_verified後の応答欠損、last_confirmed_stageはtarget_verified | 到達済み位置を保持してunknown。同じsetupをpreparedから再実行しない |
| `A05-31` | 同じboot_idで二回のサービス検証、preboot子は終了したがloop参照あり | 再起動確認/媒体解除は未成立。一部証拠でcomplete/removableへ進めない |
| `A05-32` | 消去ありでAPPLYだけ、ERASEだけ、旧Planの同じ短いコード | 二確認不足または全Plan照合不一致で変更0件。生成確認コードを表示義務・計画全体の照合の代用にしない |
| `A05-33` | 同じplan_idでmedia_id/recipeだけ変更、既存handoffは別の未完了操作 | 対応不一致またはMEDIA_PENDINGで停止。継続処理の誤起動・既存記録上書き・再setupなし |

## 未決事項と成立判定

### Subiquity入力生成の対応表

必須選択と計画確認の生成順候補は、PlanId/作成時刻の取得→操作/対象/構成/導入値の選択→MediaSelection→media_planのPlanDraft→P03の固定byte生成/外部hash証拠/封印→digestの表示→APPLY/必要時ERASE入力→Confirmation→MediaRequestとする。計画前のMediaSelection.confirmationはNoneだけを受け、古い確認を計画生成に使わない。継続管理でSelection=nullの場合もPlanIdentityが親IDを供給する。A05-41には未封印DraftをConfirmation/MediaRequestへ渡す拒否と、封印器以外のPlan生成を含める。

StorageInput.erase_confirmed_for_planはこの候補では消去対象を親計画へ結合するIDであり、入力済み確認の証拠ではない。計画中に親IDを設定できるが、実消去はMediaRequest.Confirmationの両応答・対象・全digest・期限の照合後だけ。短いコードを得るため、確認済みpayloadを先に要求する循環を作らない。対象/手順/実資材が変われば封印/選択/確認へ戻す。

| ケースID | 固定入力 | 期待結果 |
|---|---|---|
| `A05-41` | 未確認の導入値から計画生成、旧確認付きMediaSelection、親IDだけ一致しERASE未入力 | 計画生成だけ可/古い確認を拒否/消去0回。ID欄を実同意へ転用せず、digestと確認の相互待ちなし |

生成器候補generate_autoinstallは検証済みMediaRequest/RecipeInput/資材カタログを借用し、UTF-8 YAML byte列と入力hashを所有して返す純粋処理とする。action=installだけを受け、既存OSのadopt/update/backup/recoverには生成しない。全キーを次表から生成し、利用者から任意YAML/early/late shell断片を受け取らない。外部ツールのschemaとServerSeed側の未知キー拒否を別に照合する。

| 外部キー | 候補の生成規則 |
|---|---|
| autoinstall.version / interactive-sections / refresh-installer | 1 / 空配列 / update=false。必須選択はServerSeed側で完了済み、installerでの最新版探索なし |
| source | recipeの固定ベースISO内install-sources対応ID一件、search_drivers=false。対象OS/CPU/配布条件を照合し、実IDを推測しない |
| storage | action-based config。disk一件に固定したserialと実device対応、ptable=gpt、preserve=false。UEFIはESP/root、BIOSはbios_boot/rootを生成し、寸法は共通InstallPayloadそのもの |
| network | netplan version=2、ethernets一件。match.macaddress、set-name、mtuを固定、dhcp4またはstatic addresses/routes/nameserversを相互排他で生成。別interfaceの自動補完なし |
| ssh | install-server=true、authorized-keysは検証済み公開鍵集合、allow-pwは認証保護条件に従う。未確認のまま既存認証を削除しない |
| user-data | 空objectだけの比較候補。identityによる秘密password/hashの埋込みを使わず、対象ユーザーの準備を固定の対象側処理へ分離。cloud-initを利用者の別の開始経路にしない |
| early-commands | 空配列。選択/消去確認前に対象を変更するscriptを置かない |
| late-commands | 検証済み同一本体の固定内部入口に、同期済みhandoffを照合させる一つのargv配列。動的入力は記録を読ませ、shell文字列へ連結しない |
| shutdown | [下記の終了制御の成立条件](#installer終了制御の成立条件)を先に確認する。未定義のnone等を生成せず、未成立ならD05-01/02の未達として実行しない |

storageのformat/mountはESP=fat32と/boot/efi、root=ext4と/だけ、bios_bootはformat/mountなし。disk選択の自由なmatch/最大容量指定を使わない。採用storage版がoffsetを保証しない場合、共通の厳密start_bytes契約と両立する証拠を必要とする。[Curtin storage](https://curtin.readthedocs.io/en/latest/topics/storage.html)の版1と版2でoffsetの意味が異なるため、version=1へoffsetを書いただけで厳密配置を保証しない。採用版/生成schema/寸法fixtureが一致しない場合は実行前保留する。

対象ユーザー生成/公開鍵登録/昇格、late入口の設置と安全な起動はP10-OSの対象準備契約とP05-HANDOFFへ対応付ける。identity省略と空user-dataの対応は[Subiquity公式入力](https://canonical-subiquity.readthedocs-hosted.com/en/latest/reference/autoinstall-reference.html)への比較案で、採用ベースISOでの成立・cloud-initの許可用途との適合判断は未完了。生成器がファイルを出せることを、実行可能な完全なinstaller連携の証明にしない。

| ケースID | 固定入力 | 期待結果 |
|---|---|---|
| `A05-34` | adoptへ生成要求、storage v1でoffsetだけ指定、source ID未固定、late-commandへ文字列 | 生成/適用前拒否。既存OS消去・無視される寸法の保証・shell評価・既定source選択なし |

### installer終了制御の成立条件

[Subiquity公式資料](https://canonical-subiquity.readthedocs-hosted.com/en/latest/reference/autoinstall-reference.html#shutdown)に記載されたshutdownはreboot/poweroffであり、未再起動のまま終了する値は確認できていない。late-commandsはinstaller側の環境で動くため、対象内の入口を直接指定するだけでは対象OS内実行にならない。固定した実行側入口と対象rootへの束縛を必要とする。

本案はshutdownキーだけによる制御返却を実現済みとしない。ServerSeedが取り外し前の検証/同期/参照解除を行える監督・停止方式を固定版で特定するまで生成・導入を保留する。late処理の無限待機、installerを意図的に失敗させる方式、未確認のservice停止を成功経路へ追加しない。制御返却を必要としない別方式へ変える場合も、媒体なし起動と既存の引渡し要求への対応を別途審査する。

公式設定には導入後の更新既定値もあるため、refresh-installer.update=falseだけで全資材が固定されたとは扱わない。installerの実取得・package更新・driver/OEM/kernelの選定をrecipeの全変更集合へ対応付け、浮動する取得や未固定の生成結果が残ればD05-02の未達とする。新しい更新無効化キーを創作しない。

| ケースID | 固定入力 | 期待結果 |
|---|---|---|
| `A05-42` | shutdownへnone、late入口は対象内にしかない、installer更新だけ無効で対象packageは浮動 | 生成/外部契約不成立で導入開始0件。存在しない制御返却・Live側の対象処理・固定資材の捏造なし |

### 内蔵OS継続unitと二回の起動証拠

resume_mediaの入力にはP02のOperationBudget/ResumeBudgetEvidenceを必須とする候補。型/算定は[P02の再起動期限](PHASE_02.md#再起動をまたぐ期限の照合)だけを参照する。記録照合と期限判定に成功してから所有プロセス欄/継続位置を同期し、ExecutionContext.deadlineへ同じ残期限を移す。600秒の起動確認待機は元の親期限の内側で、再起動/prepare/verifyごとに全予算を再付与しない。clock契約未選定/予算記録欠損/期限到達なら内蔵consoleの読取り診断だけで、準備unitを成功扱いにして通常サービスを開始しない。A05-08/35/40には二boot間の長時間停止と予算証拠不成立を含める。

HandoffRecord/BootProof/FirmwareContract/Resultの宣言は本書の各全キー表へ対応する候補。last_confirmed_stageへFailed/Unknownを入れる入力はMEDIA_BINDING、BootProofのrequired_servicesは固定unit集合の欠損/余剰を拒否する。boot1/boot2は別のboot IDで同じ操作/対象へ束縛し、BootProofをHandoffRecord.target_receiptの代わりにしない。A05-31/35/40で型・段階・二bootの照合を行う。

resume_mediaの候補入力は、同期済みHandoffRecord/MediaRequest/RecipeInputと、今回のTargetEvidence/boot/process証拠、および上記の予算証拠。型付き境界は`resume_media<'g>(guard: &'g OperationGuard, saved: &HandoffRecord, request: &MediaRequest, recipe: &RecipeInput, observed: &TargetEvidence, budget: &OperationBudget, clock: &ResumeBudgetEvidence) -> Result<ExecutionContext<'g>, BeginFailure>`とする。ID/hash/版/CPU→予定したprebootから内蔵bootへの変化→実配置→intent/operation/資源journal→未完了step/期限の順に照合し、同じ対象operation_id/plan_idを再使用する専用再開である。

旧bootのrunningを通常beginで上書きしない。再開の所有プロセス欄だけを実boot/PID/startへ同期して結合し、元のstarted_at、完了済みstep、history、元失敗を改変しない。prepare終了時には位置を同期してguardを解放し、verifyが同じ再開照合を経て再取得する。隙間に通常変更を許可せず、pending記録を新規beginの停止条件として維持する。再開前に期限の有効性を証明できなければ対象変更/記録修復を行わず、読み取り診断と内蔵consoleへの不足表示だけにする。

| ケースID | 固定入力 | 期待結果 |
|---|---|---|
| `A05-40` | 予定した新bootで同じ操作を再開、別request hash、prepare後verify前に通常begin | 正常だけ所有欄/位置を結合、他はMEDIA_BINDING/未完了停止。新操作ID・開始時刻の上書き・二重setupなし |

継続unit候補の固定名はserverseed-media-continue.service、配置は対象の/etc/systemd/system/同名、実行本体は検証済み/usr/local/bin/serverseedとする。unitはType=oneshot、RemainAfterExit=yes、Restart=no、UMask=0077、WorkingDirectory=/、StandardInput=null、After=local-fs.target、WantedBy=multi-user.target。ExecStartは`/usr/local/bin/serverseed __internal media-continue --stage prepare --record /var/lib/serverseed/media-handoff.json`の固定argv。外部からrecord pathや追加actionを選ばせない。ログには固定診断IDだけを出し、要求/設定/鍵をjournalへ流さない。

同居機能の起動はこのunitの準備検査後に行う。pending記録がある場合は通常setup/updateを並行起動せず、継続workerが新しいguardとPID/bootを取得し、既存operation_id/plan_id/request/recipe/実配置と実対象を照合する専用resume_media契約へ渡す。これは一般beginによる中断上書きではない。recordがない正常運用では対象変更なしで終了、completeなら記録不変で通常起動へ渡す。unknown/failed/別対象/未対応版は通常変更を始めず復旧待ち。ConditionPathExistsだけで実行権限/採用済み計画を証明しない。

prepareと通常サービスの実起動検証を同じoneshotで待ち合わせない。選択構成が管理する全通常unitにはRequires/After=serverseed-media-continue.serviceを生成し、prepare失敗時は起動させない。prepareは対象照合と許可された残りの構成準備までで終了し、通常unitのactiveを待たない。boot証拠の採取は別のserverseed-media-boot-verify.service（Type=oneshot、Restart=no、WantedBy=multi-user.target）から同じ内部入口の--stage verifyだけで行う。verifyはprepareと全通常unitをAfter/Wantsで列挙し、Requiresは付けず、起動失敗した通常unitも失敗として採取できるようにする。共通のUMask/cwd/stdin/秘匿化はprepareと同じ。

verifyは新たなsetupを開始せず、操作別の照合guard下でboot証拠/継続位置だけを確定する。失敗・欠損・非稼働ならhandoff.completeにせずconsoleへ固定診断を表示する。prepareを成功に見せるためにサービス検査を省略せず、verifyから通常unitを再起動しない。両unitと依存辺を生成時に循環検査し、boot1/boot2後もcompleteの読み取りだけで正常終了できる状態を維持する。

| ケースID | 固定入力 | 期待結果 |
|---|---|---|
| `A05-39` | prepare失敗、通常unit一件起動失敗、prepareが通常unit active待ちを要求 | 順に通常起動抑止/verifyで失敗保持/生成契約拒否。oneshotとサービスの循環待ち・失敗の完了化なし |

BootProof候補の全キーはmedia_id、plan_id、target_operation_id、disk_id、root_identity、boot_id、observed_at、media_absent、required_services、checks。required_servicesは当該構成の固定unit名→実状態のmap、media_absentは採用観測器で確認したbool。boot1/boot2を対象側operations/OperationId/media-boot-1.jsonとmedia-boot-2.jsonへ同期する。boot2はboot1と異なるboot_id・同じdisk/root/操作を要求し、通常サービスと媒体非依存の全必須Checkを再検証する。同一bootの二観測やreboot要求の受付は二回目の証拠にしない。

media_absentを観測不能な場合はBootProofを構築せず、HandoffRecordをunknownと確認済み位置へ対応付ける候補。falseは媒体接続の確証がある場合だけとし、観測器失敗を「接続あり」または「取り外し済み」へ変換しない。required_servicesの欠損も空mapの起動成功にせず、同じ対象の不足Checkを保持する。A05-11/31/35へ媒体検出不能とサービス集合欠損を加え、内蔵起動の事実だけでcompleteへ進めない。

boot1後の再起動は確認済み計画と再起動許可を照合し、許可がなければ内蔵consoleで利用者に案内する。USBの再接続、SSH保守への切替、allow_rebootの書換えを要求しない。再起動前に継続位置を同期し、boot2と構成検証/必須記録の一致後だけhandoff.completeを確定する。実行期限が再起動を跨いで判定不能なら未実施の変更は開始せず、照合/表示だけ行う。boot証拠の取得不能を現在時刻や旧PIDで補完しない。

| ケースID | 固定入力 | 期待結果 |
|---|---|---|
| `A05-35` | unit再起動、complete記録、boot1と同じboot_idのboot2、再起動許可なし | 再setupなし/記録不変/二回目拒否/console案内。媒体再接続・guardコピー・自動許可変更なし |

### RAM実行と媒体解除の測定契約

RAM起動の候補測定入力は固定ISO hash/recipe hash、CPU、仮想firmware、RAM量、操作/構成、固定fixtureとする。必要量は同じISOの起動/選択/導入/preboot検証/取外し準備の全工程における最大使用量を測り、ram_required_bytesへ結合する。予約量の候補算定はmax(512 MiB, ceil(必要量/4))、整数bytesのchecked演算。メモリーavailableは起動前の割当可能量と有効cgroup制限の小さい方。測定対象にない構成・boot条件へ値を自動流用しない。

媒体解除の観測集合候補は、全mount namespaceのmountinfo、SquashFS/overlayのlower/upper/work参照、loop backing、swap、関連プロセスの開いたFDとmemory map、起動/継続資材の実配置。媒体のdevice/FSと依存グラフを辿り、参照が一つでも残る/読めない/採取中に世代が変わる場合はmedia_unmountedをtrueにしない。見える一つのnamespaceだけのumount、未取得PID/FDの省略、ファイル名の文字列一致だけでは解除を認定しない。

解除順候補は新しい媒体参照の開始を止める→対象書込み終了/同期→必要資材のRAM/内蔵配置を再検証→関連処理終了→参照グラフを葉から解除→mount/loop/swap/FD/map集合を再採取→全条件照合。解除不能なら媒体を外さず理由を表示する。取り外した後の動作はVMで実際に仮想媒体を切断して確認し、切断直前の観測だけで検証を終えない。採用ベースISOのRAM起動設定、観測APIの固定版、namespace/FDを漏れなく読める実現条件はD05-01/03の方式審査へ残す。

| ケースID | 固定入力 | 期待結果 |
|---|---|---|
| `A05-36` | RAMが必要量+予約量の直前/同時、別namespaceにloop/FD参照、PID情報が読めない | 容量条件と参照解除を独立判定。情報欠損・一namespaceの成功でremovableへ進めず、実切断後の検証を要求 |

| 判断ID | 必要な具体化・採用判断 | 影響 |
|---|---|---|
| `D05-01` | [RAM測定](#ram実行と媒体解除の測定契約)に算定/参照解除案あり。固定ベースISOとboot設定、全参照の観測方法・測定条件/根拠が残る | ISO生成・媒体取外し |
| `D05-02` | [Subiquity生成対応](#subiquity入力生成の対応表)に案あり。[終了制御](#installer終了制御の成立条件)はshutdownだけでは不成立。固定版での制御/資材更新、storage v2/ユーザーhelper、cloud-init用途制限との両立を具体化して採用判断 | 物理OS導入 |
| `D05-03` | [内蔵継続unit](#内蔵os継続unitと二回の起動証拠)に保存/再開/二回起動の案あり。実起動主体・同期/引渡し・権限の成立根拠と採用判断が残る | 媒体導入・保守 |
| `D05-04` | FirmwareContractに案あり。実機の対応行/公式操作/値域とセキュリティの試験境界は未確定。VM専用規定を実機許可へ変更しない | firmware・復電受入 |

各候補の所有者判断と共通契約の承認、非機能要求の確定が必要。本文案の作成は媒体・OS導入・実機対応の成立を示さず、製品試験は未実施。
