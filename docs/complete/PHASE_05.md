# Phase 5 完全仕様案: 外部媒体・ISOと物理PCの導入・保守

[文書一覧](../DOCUMENT_INDEX.md) | [担当工程](../ROADMAP.md#phase-5-外部媒体isoと物理pcの導入保守基盤)

状態は採用未承認の`B3`。[案の適用範囲](PHASE_01.md#案の適用範囲)を参照する。担当要件は`REQ-002`・`REQ-003`・`REQ-005`・`REQ-006`・`REQ-007`・`REQ-008`・`REQ-009`・`REQ-010`・`REQ-011`・`REQ-012`。既存の[外部媒体手順](../BASIC_SPEC.md#外部媒体による導入と保守)と[起動確認](../BASIC_SPEC.md#内蔵osの起動確認)は変更しない。

## 契約と参照

| 契約ID | 担当 |
|---|---|
| `P05-ISO` | 固定資材からのISO生成と起動 |
| `P05-MEDIA` | 操作選択・対象検査・OS導入の開始 |
| `P05-HANDOFF` | Live環境と対象OSの引継ぎ・中断・取り外し |
| `P05-FIRMWARE` | 機種別操作の選定と結果検証 |

共通参照（全フェーズB3）は、`P02-MODEL`・[識別](PHASE_02.md#p02-model)、`P02-STORE`・[記録](PHASE_02.md#p02-store)、`P02-EXEC`・[外部実行](PHASE_02.md#p02-exec)、`P03-FS`・[パス](PHASE_03.md#p03-fs)、`P04-SELF`・[本体管理](PHASE_04.md#p04-self)、`P04-BACKUP`・[保全](PHASE_04.md#p04-backup)、`P04-APPLY`・[適用](PHASE_04.md#p04-apply)。構成機能の完成はPhase 6〜8と全体受入へ接続する。

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

## P05-FIRMWARE

候補の照合キーはメーカー・機種・ファームウェア版・起動方式・操作インターフェースの組とする。項目ごとに検出、現在値、予定値、公式呼出し、読み戻し、再起動要否を結合して計画する。操作上限30秒、再試行0回を候補とする。

実際の機種・公式手段・版・項目の値域・検証資材は未選定。新しいメーカーCLIを使用可能にせず、[既存の操作制約](../SECURITY_POLICY.md#ファームウェア操作)と[試験環境の未確定条件](../SECURITY_POLICY.md#検証環境の未確定条件)を維持する。実機操作をVM結果で置き換えない。

## 内製境界・API・互換性

配置候補は`src/usb/`と`src/platform/firmware/`。選択・対象識別の調整と引継ぎ制御を内製し、OS導入・機種別操作は選定済み例外だけへ委ねる案。媒体生成・署名そのものの新規内製化は今回決めない。

| 宣言案 | 契約 |
|---|---|
| `media_plan(input: &Value, targets: &Value) -> Result<Value, ContractError>` | 必須選択と対象証拠を検証した計画を所有して返す。計画段階の変更なし |
| `handoff(ctx: &mut ExecutionContext, plan: &Value) -> Result<Value, ContractError>` | 引継ぎ位置・対象側結果を返す。応答欠損は未確認 |
| `removal_status(operation: &Value) -> Result<Value, ContractError>` | 書込み・マウント・関連処理の照合結果。取外し完了を捏造しない |

型・エラーはP02-MODEL。引継ぎpayloadはD05-03で全項目を確定する必要がある。難易度は高。初期モジュールの新設案で旧実装はなし。将来の抽出では、媒体構造・handoff形式・CLI・対象記録の互換性を同じ契約テストで比較し、旧形式の除去前に媒体と対象の復旧経路を確認する。

## データ契約の具体化案

`P05-ISO`の生成入力と`P05-HANDOFF`の工程を具体化する審査用候補。実際のベースISO・ツールの個別採用値は[例外台帳](../DEPENDENCY_EXCEPTIONS.md)だけで管理する。

| 型 | 全キー・値域の候補 |
|---|---|
| RecipeInput | schema_version: 1、architecture: 既存CPU値、base_ref: MaterialRef、tools_ref: MaterialRef[]、binary_sha256・binary_manifest_sha256: string、recipe_revision: 1以上integer。MaterialRefは[型の詳細](#型とapiの詳細案)で定義 |
| MediaSelection | action: install・adopt・update・backup・recover、target: TargetEvidence、selection: P02-STATEのSelectionまたはnull、backup_destination: stringまたはnull、confirmation: stringまたはnull |
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

資材の採用情報と仕様へのContractRefを混同しない候補。ISOとCloudで共通の取得済み資材の識別型MaterialRefを本節へ集約する。版・ライセンス・承認の現在情報は例外台帳が正本で、型の全フィールドがあるだけでは使用可能にしない。

```rust
pub struct MaterialRef {
    exception_id: String, artifact_ref: String, sha256: String,
    bytes: u64, architecture: String,
}
pub enum MediaAction { Install, Adopt, Update, Backup, Recover }
pub enum HandoffStage {
    Prepared, TargetStarted, TargetVerified, ReadyForRemoval,
    BootPending, Complete, Failed, Unknown,
}
pub struct RemovalResult {
    media_operation_id: OperationId, writes_complete: Option<bool>,
    sync_complete: Option<bool>, target_stopped: Option<bool>,
    media_unmounted: Option<bool>, removable: bool, checks: Vec<Value>,
}
```

MaterialRef.bytesは外部integerの上限をさらに適用し0を拒否する。architectureは既存CPU値、artifact_refは台帳で特定した不変成果物への参照で、任意の取得先へ展開するパスではない。RecipeInputのtools_refには使う実行物を各1件含め、まとめた「ツール一式」の自己申告を受理しない。生成環境のイメージと各実行物の両方を照合する。

RemovalResult.checksはP02-CLIのCheck形式へvalidatorで照合するValueだけを保持する。Option<bool>のNoneをfalseと同じ根拠へ潰さず、表示では「未確認」とする。HandoffRecordの実行前情報とtarget_receiptは異なる段階で確定し、target_started後の欠損を未実行へ戻す生成関数を設けない候補とする。

## 状態と中断の詳細案

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

| 型 | 全キーと制約 |
|---|---|
| MediaRequest | schema_version: 1、media_id: hex32string、action: MediaSelectionのaction、target: TargetEvidence、selection: Selection、install: InstallPayloadまたはnull、plan: Plan、backup: BackupSetまたはnull、confirmation: Confirmationまたはnull |
| Confirmation | plan_id: PlanId、target_disk_id: string、shown_digest: hex64string、response: string、confirmed_at: UtcMillis。消去の確認文字列は基本仕様を参照し、別の短縮確認へ変えない |
| HandoffRecordの補完 | 既存キーにrequest_sha256: hex64string、recipe_sha256: hex64string、target_boot_id: stringまたはnull、removal: RemovalResultまたはnullを追加する案 |
| 受渡し保存先 | 媒体側・対象側に同じmedia_id/plan_idのレコードを置き、対象側は候補/var/lib/serverseed/media-handoff.json。書込み先は外部requestから受け取らず固定生成する |

installはaction=installだけ非null、既存OSへのadopt/update/backup/recoverではnull。backupが必要な操作は保全計画への参照を省略できず、action=backupはbackup非null。selectionのplan/target/profileとplan本体が完全一致しない場合は実行前に拒否する。確認は利用者のその計画への応答であり、USBの接続や--yesで生成しない。

ISO recipeのtools配列はMaterialRefの識別集合を重複なしで固定し、baseと全tools、同梱本体の実hashをrecipeへ結合する。boot後にlatestを取得してrecipeだけ固定という扱いにはしない。RAM起動方式の採用前に必須RAM量と退避領域を測定する必要があり、本候補の形式だけで媒体取り外し可能とはしない。

| ケースID | 固定入力 | 期待結果 |
|---|---|---|
| `A05-18` | selectionはdisk A、InstallPayloadはdisk B、confirmationは旧plan | 対象照合失敗。Subiquity起動・ディスク消去・確認の自動生成なし |

## 実行と復旧の確定候補

P05-HANDOFFの候補実行は、媒体上で選択・確認→対象側記録準備→内蔵側の継続処理→媒体依存解消→取り外し案内→内蔵起動→再起動確認の順とする。Liveを通常運用の実行環境へ残さない。

Subiquityの[公式入力](https://canonical-subiquity.readthedocs-hosted.com/en/latest/reference/autoinstall-reference.html)へ渡す生成物はautoinstall/version=1を明示し、承認済みrecipeからだけ生成する候補。installer自身の更新を無効にする案と、late-commandを構造化argvへ限定する案を審査する。公式文書の対応構文の確認と採用ベースISOでの実行試験を区別する。任意の利用者shell断片を差し込まない。

対象側の継続処理がguardを所有し、媒体側は対象の同じ操作を照合するだけとする。Liveと内蔵側がそれぞれ独立してsetupを始める設計にはしない。対象boot_idが変わったら旧PID/lockの観測を新OSへ流用せず、ディスク・root・操作記録の結合を再確認する。

RAM起動案ではSquashFS・loop・overlay・swap・開いたFDを含む媒体参照が解消していることを確認する。対象処理終了、同期、参照解消、unmount確認の全成立前に取り外しを案内しない。実RAM容量不足、媒体参照の解除不能、対象側結果不明は取り外し不可。取り外し後の確認結果は内蔵OSに保存し、USBの再接続を完了条件にしない。

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

## 未決事項と成立判定

| 判断ID | 必要な具体化・採用判断 | 影響 |
|---|---|---|
| `D05-01` | ベースISO・生成ツール・起動unit・Subiquity契約と個別例外 | ISOの実生成・起動 |
| `D05-02` | ストレージ・ユーザー・ネットワーク入力と導入失敗位置 | OS導入と取り込み |
| `D05-03` | Liveと対象の実行方式・記録・排他・再起動・取外しの全契約 | 全媒体保守 |
| `D05-04` | 機種別の公式操作手段・試験環境境界・対応証拠 | ファームウェアと実機受入 |

各候補の所有者判断と共通契約の承認、非機能要求の確定が必要。本文案の作成は媒体・OS導入・実機対応の成立を示さず、製品試験は未実施。
