# Phase 10 完全仕様案: OS未導入Cloudへの導入・復旧

[文書一覧](../DOCUMENT_INDEX.md) | [担当工程](../ROADMAP.md#phase-10-os未導入のcloudへの導入復旧)

案版`B4`の契約候補で、採用未承認。[案の適用範囲](PHASE_01.md#案の適用範囲)を適用する。対象要件は`REQ-002`・`REQ-003`・`REQ-005`・`REQ-006`・`REQ-007`・`REQ-008`・`REQ-009`・`REQ-011`・`REQ-012`。基本仕様の[実行環境と対象OS](../BASIC_SPEC.md#cloudの実行環境と対象os)を正本とし、OSがない対象を扱う要求を削除しない。

## 契約と共通参照

| 契約ID | 対象 |
|---|---|
| `P10-BOOTSTRAP` | 一時実行環境の能力と導入資材の照合 |
| `P10-OS` | 対象ディスクへのOS導入・起動構成 |
| `P10-RECOVER` | 起動不能・導入中断時の照合と復旧 |

参照（全フェーズB4）は`P02-MODEL`・[対象識別](PHASE_02.md#p02-model)、`P02-STATE`・[判定](PHASE_02.md#p02-state)、`P02-EXEC`・[実行上限](PHASE_02.md#p02-exec)、`P04-BACKUP`・[保全](PHASE_04.md#p04-backup)、`P05-HANDOFF`・[対象OSへの引継ぎ](PHASE_05.md#p05-handoff)、`P09-SSH`・[接続](PHASE_09.md#p09-ssh)、`P09-REMOTE`・[配送](PHASE_09.md#p09-remote)、`P09-RECONNECT`・[照合](PHASE_09.md#p09-reconnect)。媒体に固有の起動をCloudへ必須化しない。

## P10-BOOTSTRAP

採用案では接続後に能力行を作り、必要な情報・実行物を段階別に確認する。SSHの成立は検査の入口であり、OS導入済み・root権限・必要ツールの存在は推測しない。

| 能力行 | 必須とする確認案 |
|---|---|
| 実行環境 | 実際のOS・CPU・権限・使用中の機器・ルートとマウント。対象OSの情報とは別に保持 |
| 資材の検証 | 検証済みOS資材・実行物の版・供給元・署名/ハッシュ・例外ID、実行時依存 |
| ストレージ | 対象の安定識別、実行環境を支える機器の除外、既存内容・容量・保存先との分離 |
| 起動能力 | ブート方式・loader配置・firmware操作の可否、起動先切替と失敗時経路 |
| 管理経路 | 導入後の管理ユーザー・鍵・ネットワーク・接続先と予定される識別変更 |

必要物が不足する場合は、許可された入力から自動搬入できる範囲を計画に分ける。利用者に対象VPSで追加の手動セットアップを一律要求する案ではない。搬入できない・権限が足りない・資材の採用条件が未確定なら、変更前に不足と必要な準備をマスターへ返す。

導入機構の第一候補は、対応する一時環境からUbuntu公式の導入基盤を起動し、確認済み設定で対象へ適用する方式。別候補は公式root filesystem資材の展開と起動構成だが、追加の責務・復旧・資材検証が必要になる。どちらも採用・動作確認済みではない。対応する実行環境、実行物、資材、ブート能力は`D10-01`で比較・決定し、Ubuntu以外の実行環境を根拠なく使用可能・一律対象外にしない。

導入方式の選定入力候補InstallerBindingはkind: A/B、execution_contract: ContractRef、materials: MaterialRef[]、execution_environment_ref: string、supported_architectures・supported_boot_modes: string[]を全キーとする。能力行はこの一件と実環境を照合し、方式ごとに固定工程と外部実行/期待結果を対応付ける。第一候補はA、Aの公式対応/非対話契約が成立しない場合は自動でBへ移らず所有者へ選定を戻す。Bを採るなら展開/FS/loader/network/userの全契約を別に揃える。SSH以外の入口を増やさず、実行環境の必要能力と対象OSの対応を別に判定する。

## P10-OS

1. 能力行と対象の状態・安定識別・保全条件を照合する。既存OSへの保守要求をOS再導入へ置き換えない。
2. 対象ディスク、導入方式、必要な資材、ネットワークと起動・再接続、データ影響を一つの計画へ結合する。
3. 必須選択とディスク消去の確認をマスターで受ける。実行環境を支えるディスクを選べないことを再検査する。
4. 対象の排他・実行記録を確立して、確認済みの対象だけへOSを導入する。
5. ブート構成・管理経路を検査し、必要な再起動と一時環境からの引継ぎを行う。
6. P09-RECONNECTで対象の同一性・OS起動・SSH経路を照合し、既存OS向けの本体導入・構成適用へ接続する。

OS導入1回の上限は3,600秒、自動再試行0回、導入後再接続はP09-RECONNECTの候補値を使う案。消去・展開・loader配置・再起動のどこまで実施したかを個別に記録する。タイムアウトがディスク未変更を意味するとはしない。

ストレージレイアウト、パーティション・filesystem・mount・loaderの全入力、ネットワーク生成、OS導入前の操作記録と排他、再起動後に必要な信頼根拠はD10-02で確定する。SSH先の既存ルートを書換えるだけの方式を暗黙採用しない。

## P10-RECOVER

| 状態・事象 | 採用案の分岐 |
|---|---|
| SSH実行環境が利用不能 | 対象変更なし。接続経路の準備が必要と表示し、事業者APIを呼ばない |
| 一時環境は利用可能、対象が起動不能 | 対象・保全・実ディスク・導入位置を照合して起動修復/設定復元の計画を返す |
| OS導入中断 | 記録と実ディスクを突き合わせ、適用済み・確認不能を分ける。全消去の自動再試行なし |
| 起動先・ホスト鍵の予期しない変化 | 同一性と信頼の照合待ち。別対象へ接続成功したことを復旧成功にしない |
| OS再導入が必要 | データ復元と別の計画・選択・保全確認を必要とする。通常recoverの無確認処理に含めない |

OS再導入だけでデータを復元したと報告しない。保全物があっても、対象データ・形式版・復元先・復元確認の有効性をP04-BACKUPへ照合する。

起動後照合の候補BootExpectationはtarget_disk_id・expected_root_ref・os_id・os_version・architecture・endpoint_ref・host_trust_ref・request_id・plan_idを全キーとする。refは固定した対象/台帳/信頼資料、IDは共通型で、OS/CPUは基本仕様の対象条件へ照合する。推奨は再起動前に期待集合を固定し、実応答と一項目ずつ照合する方式。ホスト鍵の予定変更も担当ポリシーで独立に信頼を確認する。partition以前の不成立は変更なし、partition以後の不明は実位置照合、SSH喪失は自動修復不可として分ける。空ディスクへの導入中断に旧OS切戻しを提案しない。

## 内製API・移行案

配置候補は`src/cloud/capability.rs`・`install.rs`・`recover.rs`。能力判定・計画・対象照合を内製し、選定したOS導入基盤を外部契約として呼び出す。OSインストーラー・SSH・ブートローダーそのものの内製化を本案の条件にしない。

| 宣言案 | 入出力と失敗 |
|---|---|
| `capabilities(session: &mut RemoteSession) -> Result<Value, ContractError>` | 環境と対象を分けた観測結果。必要能力の不明を許可にしない |
| `os_plan(input: &Value, capabilities: &Value) -> Result<Value, ContractError>` | 導入・再接続・復旧可能範囲を結合した計画。変更なし |
| `install_os(ctx: &mut ExecutionContext, plan: &Value) -> Result<Value, ContractError>` | 工程別の実適用位置と結果を返す。再送・全消去を自動判断しない |

共通型はP02-MODEL、RemoteSessionはP09-SSH。難易度は高、旧実装なし。将来の差し替えでは導入資材・対象識別・記録版と接続先変更の互換性を評価し、旧実装除去前に起動不能時も照合可能な経路を保持する。初回導入の失敗を、存在しない旧OSへの切戻し成功として表示しない。

## データ契約の具体化案

`P10-BOOTSTRAP`の能力判定と`P10-OS`の工程結果を具体化する候補。接続先の識別は`P09-SSH`、配送は`P09-REMOTE`、管理対象の状態は`P02-STATE`へ参照し、Cloud用の別の総合状態を新設しない。

| 型 | 全キー・値域の候補 |
|---|---|
| BootstrapCapability | target: TargetEvidence、installer_ref: ContractRefまたはnull、checks: Check[]、installable: boolean |
| InstallInput | target: TargetEvidence、storage_ref・network_ref・user_ref・boot_ref: ContractRef、materials: MaterialRef[]、selection: Selection、plan_id: PlanId |
| MaterialRef | 共通の取得済み資材型は[P05-ISOの型定義](PHASE_05.md#型とapiの詳細案)を参照。Cloud用のOS資材も同じ識別・台帳照合を使用 |
| InstallProgress | plan_id: PlanId、operation_id: OperationId、target: TargetEvidence、stages: InstallStage[]、reconnect: ReconcileResultまたはnull |
| InstallStage | name: 下記固定工程名、status: not_started・started・verified・failed・unknown、evidence_ref: stringまたはnull、error: Diagnosticまたはnull |

能力検査の必須ID候補は`P10-BOOTSTRAP/execution`・`privilege`・`materials`・`storage`・`boot`・`management_path`。後5件にも同じ契約ID接頭辞を付ける。全件がpass、installer_refが採用済み、対象が同じ場合だけinstallable=true。unknown・fail・未採用をnot_applicableへ変更して集合を縮小しない。既存OSの有無は別の状態判定へ照合し、SSHの接続成功をこの集合のpassとして使わない。

InstallInputの各refは完全な入力契約への参照であって、契約本文の代わりではない。ストレージ・ネットワーク・ユーザー・ブートの具体値は、採用した方式の確定済みスキーマへ別に照合する必要がある。未選定のインストーラー、版、実行環境、必須値をこの参照表だけで確定したとは扱わない。

固定工程順の候補はtarget_recheck・preserve・partition・filesystem・os_expand・boot_configure・management_configure・boot_verify・reconnect・handoff。各工程の実適用位置をInstallStageで記録し、後続not_startedを省略しない。対象方式で工程が不要な場合も、承認済み契約に理由を対応付けた検証結果を持たせ、実装者の判断で飛ばさない。各工程の予算は親の残期限を超えず、消去・展開・起動を再送しない。

新OSからの再接続は、予定されたOS・CPU・ディスク/ルートの対応と管理経路、ホスト鍵の信頼根拠を検査する。同じIP・接続成功・復元したmachine-idのどれか一つだけでは同一対象と判定しない。再起動後の証拠が不足すればInstallProgressは未完了で、Phase 9の構成変更を開始しない。

OS導入前の永続記録・排他、各工程に必要なコマンドと資材、工程別の復旧可否は`D10-01`〜`D10-03`に残る。候補の工程名・MaterialRefの存在を、資材の使用許可やroot filesystem方式の採用に読み替えない。

## 接続と責務の照合

[P05-HANDOFF](PHASE_05.md#p05-handoff)から再利用するのは対象OS側の継続・識別・結果照合の責務だけであり、媒体操作ID・USBのマウント・取り外しをCloudの必須入力にしない。Cloudの前後の対応はP09-REMOTEのrequest_idと対象operation_idへ結合する。媒体用HandoffRecordを全項目そのまま配送する流用は認めない候補とする。

対象OSがまだない段階は通常の`/var/lib/serverseed/`と実行中OSの共通ロックだけでは排他・記録を代替できない。実行環境を支えるOSに作った記録と、後から起動する対象OSの記録を区別し、両者とマスターに保持する証拠の確定・継続方式を`D10-02`で審査する。対象ディスクを消去する前にこの方式が成立しなければ、installableをtrueにしない。

一時環境のOSが通常運用の対象外であることと、その環境からUbuntuを導入できる能力は別に判定する。必要な標準機能・実行物・権限・記録先が未確認なら保留し、「SSHさえあれば必ず導入可能」とも「Ubuntu以外は一律利用不可」とも補完しない。選定済みインストーラーの外部契約が揃った範囲だけで能力行を定義する。

P09-RECONNECTの同一性・信頼・対象OSの起動確認を終えてから本体導入へ移る。OS起動成功、構成適用、バックアップ復元、アプリ検証は別のstepとして[P04の結果](PHASE_04.md#データ契約の具体化案)へ接続し、OS導入の成功一つで全保守完了にしない。

## 受入ケース案

| ケースID | 区分・前提 | 期待結果・検証 |
|---|---|---|
| `A10-01` | 正常。SSH一時環境と空のテストディスク | 必須選択後にOS導入、対象OS起動と再接続、その後の構成適用へ接続 |
| `A10-02` | 異常。実行環境はあるが権限・資材・loader能力不足 | 変更前停止。接続成立だけで導入可能としない |
| `A10-03` | 異常。実行中ルート・保存先・別機器を導入対象に指定 | 誤対象への消去・書込み0回 |
| `A10-04` | 異常。消去後・展開後・loader配置後の中断 | 実位置を照合し、自動再消去・未実行扱いなし |
| `A10-05` | 正常・異常。予定された接続変更と予期しない鍵変更 | 前者も同一性確認、後者は照合待ち。鍵の無確認置換なし |
| `A10-06` | 異常。新OS起動不能、SSH自体が停止 | 保持した証拠と利用可能経路に基づく案内。事業者操作の自動化なし |
| `A10-07` | 境界。OS導入・再接続の上限超過 | 適用位置と未確認を表示し、成功・未実行を推測しない |
| `A10-08` | 異常。保全物破損、復元先不一致 | データ復元停止。OS起動成功をデータ復旧へ転用しない |

## 追加の受入ケース案

`P10-BOOTSTRAP`・`P10-OS`・`P10-RECOVER`の能力と引継ぎを検証する設計。実際の導入試験は方式・資材・隔離条件を承認した専用VMでのみ行う。

| ケースID | 区分・前提と入力 | 期待結果・検証方法 |
|---|---|---|
| `A10-09` | 異常。SSH接続成功、必須能力1件unknownまたはinstaller未採用 | installable=false。対象ディスクへの書込み0回 |
| `A10-10` | 異常。空ディスクだがOS導入前の排他・永続記録を確立不能 | 消去前停止。一時OSのoperationだけで対象排他を成立としない |
| `A10-11` | 異常。Cloud要求へ媒体操作ID・USBマウントを必須とする入力 | 経路契約の不一致。USB準備をCloudの追加条件にしない |
| `A10-12` | 異常。os_expand後中断、後続not_started、同じIPに別OSから応答 | 工程位置を保持し、同一対象・信頼の照合完了までhandoffなし |
| `A10-13` | 正常・異常。OS起動は成功、バックアップ復元は失敗 | OS段階の結果とデータ復元失敗を分離。構成・保守全体の成功なし |

## 型とAPIの詳細案

`P10-OS`のContractRefは仕様識別であり、導入値そのものではない。InstallInputへ次のpayloadを結合する採用候補を置く。ストレージ方式・OS導入機構の採用は`D10-01`・`D10-02`で別に確認する。

```rust
pub struct StorageInput {
    disk_id: String, expected_bytes: u64, erase_confirmed_for_plan: PlanId,
    layout: String, sector_bytes: u32, partitions: Vec<PartitionInput>,
}
pub struct PartitionInput {
    number: u32, start_bytes: u64, size_bytes: u64,
    purpose: String, filesystem: String, mountpoint: Option<String>,
}
pub struct NetworkInput {
    interface_id: String, method: String, addresses: Vec<String>,
    gateway: Option<String>, dns: Vec<String>, mac_address: String, mtu: u32,
}
pub struct UserInput { name: String, public_keys: Vec<String>, privilege_method: String }
pub struct BootInput { mode: String, loader_material: MaterialRef, next_endpoint: String }
pub struct InstallPayload {
    storage: StorageInput, network: NetworkInput, user: UserInput, boot: BootInput,
}
pub fn validate_install_payload(input: &Value, capability: &BootstrapCapability)
    -> Result<InstallPayload, ContractError>;
```

許容候補はlayout=`whole_disk_uefi`または`whole_disk_bios`、network.method=`dhcp`または`static`。UEFI/BIOSは対象の確認済み方式と一致させ、未確認を既定方式へ補完しない。dhcpはaddresses/dnsが空でgateway=null、staticはアドレス1件以上・DNS1件以上とし、IP族・prefix・gatewayの到達条件は方式別外部契約で検証する。DHCP成功を導入後の接続先固定の証明にしない。

UserInputは秘密値を含めず、管理ユーザー名・公開鍵・選定した昇格方式の識別だけを保持する。対象ユーザー名の候補範囲は`[a-z_][a-z0-9_-]{0,31}`、rootは拒否、公開鍵は空集合不可。鍵の形式検証・昇格の実現性・ネットワーク生成・レイアウト内のパーティション寸法は未決として明示し、String型を自由入力の許可にしない。検証不能なら消去前停止。物理媒体の導入入力へ共通化する範囲も、Subiquityの正式契約へ照合してから決定する。

## 状態と中断の詳細案

`P10-OS`の各InstallStageはnot_started→started→verifiedだけを通常経路とする候補。失敗確証はfailed、応答欠損・実位置不明はunknown。同じ工程をnot_startedへ書き戻して再実行しない。

| 工程境界 | 失敗・中断後の必須照合 |
|---|---|
| target_recheck/preserve | 対象と保存先・保全物。消去前に保全根拠が揃わなければ停止 |
| partition/filesystem | 実テーブル・FSと作成位置。元の空状態や旧OSへ戻ったと推測しない |
| os_expand | 展開した資材・対象root・未完了範囲。展開コマンドを自動再実行しない |
| boot_configure/management_configure | loader・起動先・ユーザー・ネットワークの実設定。実行環境のSSH設定を代用しない |
| boot_verify | 書込み済み設定の検査と、実際の対象OS起動を区別。設定検査だけでreconnect成功にしない |
| reconnect | 接続先・ホスト鍵・ディスク/ルート・OS/CPUの対応。IP一致だけで成功にしない |
| handoff | 旧一時環境の記録と対象内蔵OSの記録・操作の結合。二つの独立変更を同時に走らせない |

全工程を消去後の対象ディスクだけへ記録すると、最初の消去・記録先障害を照合できない。候補はマスターの固定要求・工程記録と、対象一時環境の監督記録、導入後内蔵OSへの引渡しの三者照合とする。前二者の不一致時は消去を開始せず、開始後の欠損はunknownで止める。永続媒体と排他の具体手段は`D10-02`で決定し、メモリー上の値だけで成立したとはしない。

通常のrecoverは起動修復・設定復元の承認済み範囲だけ。partition以降の再導入が必要なら新しい選択・保全・消去確認を必要とし、前回の確認文字列を自動再利用しない。

## 依存と互換性の詳細案

OS導入方式の審査対象を次の二案へ具体化する。現時点でいずれも未採用。既存OSのないSSH対象を除外せず、実行環境の能力によって各案の可否を個別判定する。

| 案 | 要求される外部境界 | 不成立なら停止する条件 |
|---|---|---|
| A: Ubuntu公式導入基盤 | 実行環境で動く対応版インストーラー、資材の署名/版、非対話入力、対象分離、loader/ネットワークの出力契約 | 公式に対応する起動・実行方式、監督記録、必要権限を確認できない |
| B: 公式root filesystem展開 | 固定rootfsと展開方式、partition/FS/loaderの個別実行物、ユーザー/SSH/ネットワーク生成と起動検証 | 個別成果物・ABI・ブート互換、完全な入力と復旧範囲を確定できない |

比較の基準は、必要な事前準備、SSH実行環境のOS/CPU、導入後の対応OS/CPU、初回信頼、データ保全、停止・再接続の確実性、必要例外と実証可能性。SSH以外の事業者API・cloud-init・ISO接続を一律必須にしない。利用者による対象VPSでの手動セットアップを既定工程へ追加せず、成立不能な能力不足を具体的にマスターへ返す。

P05のMaterialRefと対象OS継続の責務は再利用するが、媒体起動・取り外し・media_operation_idは使用しない。P09のinstall_osへ配送した要求は本書だけで検証し、install_bodyからの自動昇格はしない。将来のインストーラー差し替えは同じ空ディスク/使用中ディスク/中断位置の試験を適用し、生成OS・起動・管理経路と復旧限界を比較する。旧OSがない場合の「切戻し」を捏造しない。

## 受入条件の充足照合案

BOOTSTRAPは接続と能力を分離し、OSは具体payloadと全工程、RECOVERは実位置・起動/再接続・保全物へ対応する。方式A/Bのどちらにも未確定契約があるため、参照だけを渡してインストーラーの入力を実装側に補完させない。

| ケースID | 区分・固定入力 | 期待結果と確認 |
|---|---|---|
| `A10-14` | 異常。InstallInputの契約refはあるがstorage/network/user/bootの実payloadなし | 変更前拒否。契約リンクを実際の導入値へ変換しない |
| `A10-15` | 異常。UEFI layout指定、検出boot方式はBIOSまたはunknown | layout/boot不一致で消去前停止。既定方式へのfallbackなし |
| `A10-16` | 異常。マスターにはpartition開始、対象一時環境の記録欠損 | 対象を未消去と推測しない。再消去0回、実ディスクと残存証拠の照合待ち |

パーティション寸法・FS/loader・ネットワーク/公開鍵・昇格の全契約、導入前の永続記録と排他、実対象への信頼の根拠を確定するまで完全仕様未成立とする。空ディスクへのOS導入、OS起動、本体導入、構成適用、データ復元を別結果にし、どれか一つで全経路の成功を代表させない。

## 入力と保存形式の詳細案

P10-OSのInstallPayloadについて、先述の型に対応するレイアウト・ネットワークの全集合を次の採用候補へ閉じる。物理経路からも同じ値を参照するが、導入機構は各経路で別に採用・検証する。

| 入力 | 具体化する候補 |
|---|---|
| StorageInput | 六キーの宣言を使用し、sector_bytesは512または4096、partitionsはPartitionInput[]。expected_bytesと実disk容量・sectorが完全一致しない場合は停止 |
| PartitionInput | number: 1以上u32、start_bytes・size_bytes: u64、purpose: esp/bios_boot/root、filesystem: fat32/ext4/none、mountpoint: /boot/efi・/・null。開始/寸法は1 MiB境界、重複・終端超過・予約領域との重複を拒否 |
| whole_disk_uefi | GPT、espは先頭1 MiB後から512 MiB・fat32、rootはその直後から末尾予約1 MiBを除く残りの1 MiB切捨て・ext4。ESP/root以外なし |
| whole_disk_bios | GPT、bios_bootは先頭1 MiB後から2 MiB・none、rootはその直後から同じ末尾予約を除く残り・ext4。採用loaderのBIOS/GPT対応が未検証なら使用不可 |
| root容量 | 候補最小20 GiB。実OS・三構成・保全領域の必要容量を別に満たすこと。rootの最小値だけでmaster適合としない |
| NetworkInput | 八キーの宣言を使用し、mac_addressは小文字colon区切り6octet、mtuは1280〜9000。interface_idだけで再同定せずMACと対応 |
| static | 初期候補はIPv4のみ。addressは1件のCIDR、gatewayは同一subnetの単一IPv4、dnsは1〜3件の一意IPv4。network/broadcast/unspecified/multicastを拒否。IPv6は方式別仕様待ち |
| BootInput | modeとloader_materialとnext_endpointの既存集合を維持。next_endpointは登録済みRegistryEntry参照だけ。任意shell・loader設定断片を受理しない |

レイアウト検査の候補計算はA=1,048,576 bytes、末端E=floor(expected_bytes/A)*A-A、UEFIのroot開始S=513*A、BIOSはS=3*A、root寸法=E-S。partition番号/順序はESPまたはbios_bootを1、rootを2とし、purpose/filesystem/mountpointは表の組だけを許容する。E<=S・算定overflow・寸法不一致を拒否する。UserInput.public_keysは制御文字/複数行/空/重複を拒否し、受理鍵形式と昇格方式の安全条件は担当ポリシーの採用待ち。自由なauthorized_keysオプションやshell断片を受けない。

これらは寸法・対応範囲の提案値であり、Ubuntu公式資材・Subiquity・Cloud導入方式との適合は未確認。未対応のIPv6環境をIPv4へ勝手に変更せず導入前停止とする。公開鍵・昇格方法・loader実行引数は個別外部契約とセキュリティ審査が成立するまで未達として残す。

| ケースID | 固定入力 | 期待結果 |
|---|---|---|
| `A10-17` | 512/4096sector、末尾予約への1 MiB侵入、同じinterface名でMAC変更、IPv6-only | 前者は各寸法検査、他は消去前停止。余剰partition・別interface・IPv4代替の自動選択なし |

## 実行と復旧の確定候補

P10-BOOTSTRAPの事前記録は消去対象外の永続領域を対象一時環境から使用できることを候補条件とする。マスター側の要求・工程記録も固定し、双方のrequest/plan/disk/digest一致を確認してからpartitionを開始する。SSHだけを入口とし、事業者API・ISO接続・cloud-initを追加の必須入口にしない。一時環境の能力不足は実行不可として説明する。

永続領域が消去対象内、RAMだけ、保存元の実機器不明、他プロセスとの対象排他不成立なら消去前停止。新しく領域を作るため先に対象を消去する循環を認めない。SSHに接続できるすべての環境へ導入可能とは保証しない。

| 保存・応答の位置 | 中断後の照合 |
|---|---|
| 両者prepared、partition開始前 | 再観測したdiskと記録の一致、対象排他、確認期限を検査。新規変更の自動投入なし |
| 片側started・他方prepared | startedを優先して実ディスク照合。未消去と扱わない |
| partition以後の記録欠損 | unknown。再partition・再format・再展開を禁止し、残存表/FS/資材を読む |
| OS設定済み・再起動応答欠損 | 新OS側の信頼・同一disk/root/OSを照合。旧IPの疎通だけではhandoff不可 |
| 起動不能・SSH利用不能 | 自動化の実行経路なしとして停止。保全資料と手動で必要な復旧条件を提示 |

初期案の導入機構A/Bの選定は未完了。Aは公式installerの対応可能な無人導入、Bは公式rootfsとloader構築の個別統合であり、互換な一方式として混在させない。bootloader・network・ユーザーの実生成と検証が確定するまでstageの名前だけで実行可能にしない。再導入には新planと消去確認が必要で、旧確認を再利用しない。

| ケースID | 固定入力 | 期待結果 |
|---|---|---|
| `A10-18` | journal保存領域が同じ消去disk、片側のみpartition.started、再起動後SSH不能 | 前者は消去0回、後者は実位置照合/経路なしで停止。自動再消去・事業者API代行なし |

## 依存と受入の確定候補

P10の内製範囲はBootstrapCapabilityの評価、InstallPayload検証、工程記録/引渡しの調整とする候補。OS展開・partition/FS/loader・ネットワーク生成は外部契約の個別採用対象であり、Rust stdにない機能を未定義の自作crateへ丸投げしない。

| 採用審査 | 決定に必要な比較 |
|---|---|
| installer A/B | 対応CPU/UEFI/BIOS、公式資材の取得・署名、root/boot/network/user生成、SSH切断後の監督、全実行物とargv、同じpayloadからの期待結果 |
| 永続記録/排他 | 消去外の記録と対象diskの所有、親/SSH切断・再起動・記録片側欠損で再消去しない根拠 |
| 新OSの信頼 | 初回公開鍵/host keyの独立した照合、期待endpointと実disk/root/OS/CPU、旧主体/一時環境の失効 |
| 復旧限界 | SSHが残る場合の読取り/修復と、SSH喪失後に自動化できない範囲。事業者APIや物理媒体をCloud必須経路へ追加しない |

必須検査ID候補はP10-BOOTSTRAP/capabilities.complete、P10-OS/storage.bound、P10-OS/journal.durable、P10-OS/payload.valid、P10-RECOVER/new_os.bound。全passだけで導入後構成まで完了にはせず、P09と各構成の受入へ接続する。

将来のcrate差し替えは評価器と工程記録器を候補とし、installerの変更は外部依存の再採用として別評価する。同じ空/既存diskと中断点fixtureでstage・実書込み・partial・復旧不能範囲を比較し、OS起動後だけの比較で消去前安全性を認定しない。

| ケースID | 固定入力 | 期待結果 |
|---|---|---|
| `A10-19` | installer変更で成功OSは同じ、消去前journal方式だけ未検証 | 互換認定不可。中断・再接続・再消去防止の受入を再設計し未達を保持 |
| `A10-20` | 方式Aは非対話契約不成立、方式Bの資材だけ存在 | installable=false、所有者の選定待ち。Bの自動選択・対象VPSでの追加手入力待ちなし |
| `A10-21` | root寸法がE-Sと1sector違う、partition番号重複、公開鍵へ改行/options | payload拒否、消去前停止。外部installerの補正へ任せない |
| `A10-22` | 同じIP/OSでroot対応不一致、予定の鍵変更だが独立信頼未確認、空disk導入後SSH喪失 | handoff不可/照合待ち/経路なしを分ける。旧OSへの架空切戻し・自動再消去なし |

## 未決事項と成立判定

| 判断ID | 残る具体化・採用判断 | 影響 |
|---|---|---|
| `D10-01` | InstallerBindingを方式/環境単位で固定しAを第一候補とする。所有者は実資材・対応CPU/boot・非対話/監督の外部契約を比較して採用判断。A不成立時のB自動選択はしない | 導入方式全体 |
| `D10-02` | 単一の六/八キー型と明示レイアウト算定を推奨。所有者はIPv4限定候補の要求適合、鍵/昇格・loader/network生成、永続記録/排他の実方式を決定。型の存在だけで消去可能としない | OS導入と中断照合 |
| `D10-03` | BootExpectationの事前固定と工程別復旧限界を推奨。所有者は新OSの信頼確立/失効確認と隔離VMの証拠条件を決定。SSH喪失時に事業者操作や未確認の再導入で補完しない | 初回起動・復旧 |

契約・共通依存の採用承認、実行環境の根拠・非機能要求の確定が残る。本文案はOS未導入環境の実装・受入・対応済みを示さず、完全仕様は未成立。
