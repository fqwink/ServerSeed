# Phase 8 完全仕様案: 物理マスター構築と内包機能の統合

[文書一覧](../DOCUMENT_INDEX.md) | [担当工程](../ROADMAP.md#phase-8-物理マスター構築内包機能の統合)

案版は`B13`、採用判断は未承認。[案の適用範囲](PHASE_01.md#案の適用範囲)の対象とする。要件は`REQ-001`・`REQ-002`・`REQ-004`・`REQ-005`・`REQ-006`・`REQ-007`・`REQ-008`・`REQ-009`・`REQ-011`・`REQ-012`。既存の[マスター管理構成](../master/specification.md)を統合・削除せず参照する。

## 契約と参照

| 契約ID | 本書の所有範囲 |
|---|---|
| `P08-MASTER` | 初期構築・対象台帳・マスターCLIの候補 |
| `P08-COHOST` | 三機能の同居・共有変更・準備状態 |
| `P08-BACKUP` | マスター固有の保全と喪失時復旧 |

共通参照（全フェーズB13）は`P02-MODEL`・[型](PHASE_02.md#p02-model)、`P02-CLI`・[CLI](PHASE_02.md#p02-cli)、`P02-STORE`・[記録](PHASE_02.md#p02-store)、`P05-MEDIA`・[導入](PHASE_05.md#p05-media)、`P05-HANDOFF`・[引継ぎ](PHASE_05.md#p05-handoff)、`P06-APP`・[サーバ](PHASE_06.md#p06-app)、`P07-RUN`・[採掘](PHASE_07.md#p07-run)、`P04-BACKUP`・[保全](PHASE_04.md#p04-backup)。SSH自動処理はP09-SSH・P09-REMOTEに分離する。

## P08-MASTER

採用案では外部媒体でのマスター選択後、内包するサーバ・採掘の必要能力をすべて検査する。一機能の非対応を無視した縮退構成は追加しない。内蔵OSへ同じ共通処理と各構成機能を適用し、マスター固有の設定を行う。

### 台帳の候補

| 項目 | 型・必須性・制約案 |
|---|---|
| 台帳の版 | 必須integerの候補版1。配置候補は`/etc/serverseed/master/targets.yml`。個別構成仕様への正式反映は別途判断 |
| 対象ID | 必須string、`[a-z][a-z0-9-]{0,63}`。表示名と接続先を識別子の代わりにしない |
| 接続先 | host・port・管理ユーザー・接続用の参照情報を必須。秘密情報の値は台帳のフィールドにしない |
| 信頼の対応 | 対象IDとホスト鍵識別・対象の安定識別を結合。保存・更新・削除は信頼ポリシーへ照合 |
| 管理操作 | 台帳登録・検査・実行の操作を分ける。対象登録だけで遠隔の構成適用を開始しない |
| 件数 | 1マスター1,000対象を上限候補。未登録0件は許容し、遠隔準備と通信成功を分ける |

CLIの候補は、対象登録・一覧・解除と、対象・操作の対話選択を分ける方式。具体的なコマンド名・フラグはD08-01で比較するため、本案で`master`コマンドを採用済みとはしない。通常運用時の選択・計画・実行入口は既存の[自動処理フロー](../master/specification.md#自動処理の流れ)へ接続する。

## P08-COHOST

採用案では管理・サーバ・採掘それぞれの「設定済み」「基盤適用済み」「検証済み」「利用可能」を別々に照合する。SSH管理先0件は接続成功の代わりにせず、対象未登録という準備状態として表示する。これらは内部の機能別結果であり、共通の総合状態名を増やさない。

| 共有する資源・操作 | 計画に必須とする具体化案 |
|---|---|
| CPU・GPU・メモリー | 管理・サーバに必要な予約と採掘の上限、測定根拠、競合時の停止対象 |
| ストレージ | データ・保全・ログの容量、資源所有者、同一機器での保存先混同の検査 |
| ネットワーク | 管理経路・アプリ・poolへの影響、変更前後の同一性・復旧確認 |
| ホスト再起動・driver変更 | 全機能の保全・停止・復帰と実行中SSH操作の位置・記録照合 |

資源割当を無根拠に固定せず、管理者入力へ丸投げもしない。測定方式、許容範囲、必須性、不足時の停止と評価式をD08-02で決定する。採掘側の許可を他機能へ広げない既存の[同居運用条件](../POLICIES.md#マスター管理構成の運用)を維持する。

構築の処理順案は共通OS・記録基盤、サーバ機能、採掘機能、管理機能、全機能照合とする。機能別に開始・完了・失敗位置を保存し、途中失敗で作成済み資源を無条件除去しない。再実行は残存資源と記録から再計画する。同居構成の全体完了は、媒体なしの起動・再起動を含む必須項目の照合後に限る。

資源審査の追加候補CapacityProfileはprofile_id・os_architecture_ref・workload_ref・minimum_cpu_reserved・minimum_memory_reserved_bytes・server_requirement_ref・mining_requirement_ref・measurement_refを全キーとする。各数値は正整数、参照は固定した測定条件/担当要求への参照。CPU総量はOSで観測した割当可能量と有効制限の小さい方を論理CPU単位へ切捨て、取得不能/0は不成立とする案。各予約/割当は最低要求以上かつ既存の総量式を満たす必要がある。推奨は固定負荷プロファイルを使う測定で、値を入力しただけの資源保証はしない。最低値そのものは未決で、採用前の任意設定へ委ねない。

## P08-BACKUP

共通の取得・保存・復元手段はP04-BACKUPを再利用し、本案では保全対象を対象台帳・信頼情報の参照・計画と遠隔操作の対応・実行中/未確認状態の証拠へ対応付ける。秘密情報の保存可否・方式は[担当ポリシー](../SECURITY_POLICY.md#秘密情報)の決定を必要とし、本案で新しい鍵保管規則を作らない。

マスター喪失時は新しい物理マスターを外部媒体で構築し、検証した保全物を復元し、各管理先の同一性と操作状態を読み取って再照合する案。同じ対象への変更を旧マスターも再開しないよう、排他・管理主体の切替を確認する。復元した台帳だけで未確認操作を再投入しない。

旧マスターの失効・切替方式、喪失した秘密情報、同じMachineIdを継承する条件、保全物からの台帳解除・復元はD08-03で審査する。冗長化・複数マスターの運用契約を本案へ追加しない。

喪失時切替の審査候補ControlTransferはold_master_id・new_master_id・registry_sha256・target_ids・revocation_evidence_refs・reconciliation_refsを全キーとし、IDはMachineId、対象/参照は重複なしarray。第一候補は新IDと対象側での旧制御権失効を明示する方式、旧ID自動継承は採らない。失効は台帳書換えでは証明できず、対象の認証/権限の実効性を担当安全審査へ対応付ける。対象一件でも失効/未確認要求の照合が欠ければその依存範囲の変更は保留する。失われた秘密鍵の復元を保全物の存在だけで保証しない。

ControlTransferの候補ではtarget_idsをASCII昇順に固定し、二つの証拠参照配列は同件数・同位置の対象IDに対応させる。各参照先にもold/new主体・対象・台帳hashを持つ照合資料を要求し、配列の位置一致だけで正しさを認定しない。新旧MachineIdが同じ、台帳hashが復元対象と違う、証拠が別対象、pending要求の結果一件でも未確認ならMASTER_TRANSFERで当該依存範囲の変更を保留する。旧主体失効の実方式はセキュリティポリシーで決めるまで未成立とし、照合器が失効処理を自動実行しない。

## 内製API・差し替え案

配置候補は`src/profiles/master/registry.rs`・`cohost.rs`・`backup.rs`。台帳・統合制御・対応照合は内製、サーバと採掘の処理は各担当モジュールを呼び出す。Coreがマスター専用型へ依存する循環を作らない。

| 宣言案 | 所有権・動作 |
|---|---|
| `master_plan(registry: &TargetRegistry, server: &[AppPlan], mining: PlanDraft, capacity: &CapacityInput, target: &TargetEvidence, identity: PlanIdentity) -> Result<PlanDraft, ContractError>` | 同じ親identityへ三機能と共有資源を結合した未封印計画を所有して返す。子Planの再帰内包・hash実行なし |
| `apply_master(ctx: &mut ExecutionContext, plan: &Plan) -> ChangeResult<MasterProgress>` | MasterProgressはreceipt: OperationReceiptとfeatures: FeatureReadiness[]の二項目。開始後の部分位置と三機能の結果を保持 |
| `reconcile_master(saved: &ControlTransfer, observed: &[ReconcileResult]) -> Result<Vec<Check>, ContractError>` | 固定した対象別証拠と観測結果を照合。失効・修復・再投入を実行しない |

型はP02-MODEL。台帳・機能別結果の全フィールド、権限と保存形式はD08-01〜03の未達。新設候補で旧実装なし、難易度は統合・喪失時復旧が高。将来の差し替えは台帳版・対象ID・遠隔操作対応の互換性を維持し、旧マスターとの同時変更防止と切戻し可能性を先に確定する。

## データ契約の具体化案

master_planはprofile=Masterの同じ親identityで生成したmining Draftを消費し、serverのAppPlan資源/担当step、registry/capacityのstepと平坦に合成する候補。子Draft全体をpayloadへ入れず、target/identity/資源ID/path/step ID/参照版の衝突はPLAN_PAYLOAD。別IDの子計画を親IDへ書換えて取り込まない。封印は結合後の一回だけ、完了済み子Planのdigestを親digestとして流用しない。A08-01/15には別親のmining Draft、同pathのserver/mining競合、平坦合成後の循環を含める。

`P08-MASTER`の対象台帳、`P08-COHOST`の機能別結果と資源配分入力を具体化する候補。配置先は同節の候補を使用し、構成別仕様への正式反映・実ファイルの生成は行わない。

| 型 | 全キー・制約の候補 |
|---|---|
| TargetRegistry | schema_version: 1、master_machine_id: MachineId、targets: RegistryEntry[]。件数上限はP08-MASTER |
| RegistryEntry | target_id: 同節の識別子、host: string、port: 1〜65,535のinteger、user: string、credential_ref・trust_ref: string、expected_machine_id: MachineIdまたはnull |
| FeatureReadiness | feature: server・mining・master、configured・applied・verified: boolean、status: ready・failed・unknown、checks: Check[] |
| MasterReadiness | machine_id: MachineId、features: FeatureReadiness[]、target_count: 0以上integer、ready: boolean |
| CapacityInput | cpu_total・cpu_reserved・cpu_server・cpu_mining: 0以上integer、memory_total_bytes・memory_reserved_bytes・memory_server_bytes・memory_mining_bytes: 0以上integer、evidence: TargetEvidence |

target_idの重複と、同じ検証済み対象を異なるIDで同時管理する登録を拒否する候補。expected_machine_id=nullは未導入時にだけ許容し、実接続後に得た識別情報を無確認で台帳へ書き戻さない。host・userは入力長と文字種・接続先形式を`P09-SSH`の正式契約へ照合し、任意のSSH設定断片を受け付けない。credential_ref・trust_refは所有者管理下の安全な参照で、値やファイル内容を台帳へ格納する意味ではない。

重複検査候補はtarget_id、非nullのexpected_machine_id、同じhost/port/userの登録を別々に照合する。接続設定の文字列が異なる場合も、同じ実対象と確認した証拠があれば二重登録を拒否する。一方、DNS名の相違だけで別機器と確定しない。登録前の純粋検査は名前解決やSSHを実行せず、初回同定で既存対象と判明した場合は変更配送前にREGISTRY_DUPLICATEで停止する。

遠隔管理先としてマスター自身を登録しない候補とする。expected_machine_id/既知の実対象がmaster_machine_idと一致する登録はREGISTRY_DUPLICATE、未同定の別名が接続後に自身と判明した場合もSubmit前に停止する。ローカルの共通guardを保持したまま自身の遠隔guardを待つ構成、SSHを使って物理マスターの媒体保守条件を迂回する経路を作らない。A08-12/26/28で別DNS/別資格の同一マスターを照合する。

MasterReadiness.featuresは三機能のFeatureReadinessをちょうど1件ずつ含める。configured/appliedだけでreadyにせず、verifiedと必須検査成功が必要。MasterReadiness.readyは全三機能readyのときだけtrue。target_count=0でも対象登録入口の準備は検証できるが、接続・遠隔実行が成功した証拠にはしない。

FeatureReadinessの候補集約は、必須failが一件でもあればstatus=failed、failなしで必須unknown/欠損またはconfigured/applied/verifiedの未成立があればunknown、それ以外だけreadyとする。資源配分のresources.boundedと台帳整合の検査はmaster機能のchecksへ必須で結合し、三つのverified boolだけで全体readyを作らない。featuresはserver/mining/masterの固定順で返す。capacityの総量式、最低要求、測定根拠、実効制限の四条件を別Checkにし、一条件のpassを他条件へ複製しない。純粋集約器は測定・サービス操作・記録保存を行わない。

readyの候補判定では、P02-CLIのrequired_checksから求めた当該機能の必須集合とchecksの対応も必要とする。server/miningは各担当の適用済み資源・運転/データの検査、masterは台帳と資源四条件を含み、担当集合が未登録ならunknownとする。checks=[]、他機能のpassだけ、全件required=falseを根拠に空集合のreadyを生成しない。A08-24/27/29へこの三入力を加え、boolが全trueでもready=falseとなることを照合する。

configured/applied/verifiedは工程の検証済み事実で、verified=trueならapplied/configuredもtrue、applied=trueならconfigured=trueを要求する候補。矛盾するboolの組はREADINESS_INPUTで拒否し、unknownへ丸めて正常入力にしない。boolが全trueでも担当必須Checkのfailを優先し、status/readyは集計器が再算定する。呼出元からのready=trueをそのまま採用しない。

純粋集計API候補はmaster_readiness(registry: &TargetRegistry, features: Vec<FeatureReadiness>) -> Result<MasterReadiness, ContractError>。machine_id/target_countは検証済み台帳から求め、別の入力値で対象数を偽装しない。三機能集合の欠損/重複・上記bool矛盾はREADINESS_INPUT。資源四条件のCheck.idはP08-COHOST/resources.total、resources.minimum、resources.measured、resources.enforced（後三つにも同じ接頭辞）とし、全必須passのときだけresources.boundedをpassにする。欠損はunknown、確定failはfail、元の各根拠は消さない。各機能の他の必須集合は担当契約へ参照し、未知集合を空の成功へしない。

資源配分の評価候補は、`cpu_reserved + cpu_server + cpu_mining <= cpu_total`と、bytes単位で同形のメモリー式の両方が成立すること。整数加算はchecked arithmetic、計算不能・観測不足は失敗。予約・割当の入力の生成方法、最低必要量、検出CPUとスケジューラ制限の対応、systemdへの適用方法は`D08-02`の採用事項で、ゼロ予約を既定値として通さない。式の成立だけで実負荷時の管理経路やアプリ性能を保証しない。

マスター保全では、台帳版・master_machine_id・保全時点の未確認要求を同じ保全物へ対応付ける候補。失われたマスターの識別継承と旧マスターの制御権失効を確認できない場合、readinessがreadyでも遠隔変更を再開しない。切替の具体手段と秘密情報の復旧は`D08-03`およびセキュリティポリシーの決定待ちである。

## 接続と責務の照合

マスター自身の変更は[P05-MEDIA](PHASE_05.md#p05-media)から、Cloud先の変更は[P09-REMOTE](PHASE_09.md#p09-remote)から開始する。TargetRegistry.master_machine_idとRegistryEntry.expected_machine_idを取り違えず、マスターの管理状態・共通ロックを遠隔対象の管理記録へコピーしない。

同じ物理マスター上の三機能は一つの親計画・共通排他で調整する候補。対象台帳の変更、遠隔操作の要求・結果保存と、マスター自身の保守が共有する資源を照合する。台帳の解除を対象OSの削除・構成解除・実行中操作の取消へ拡張せず、未確認要求が残る対象を無条件に解除しない。台帳側と対象側の排他の全取得順序は`D08-01`・`D09-02`の未達に残す。

喪失時の復元は[P04-BACKUP](PHASE_04.md#データ契約の具体化案)の同じ保全物・形式・検証方法へ対応付ける。新しい物理マスターの構築成功と旧制御主体の失効確認は別の必須条件である。復元後も未確認要求を読み取り照合するまで遠隔変更を停止し、冗長化や同時二台運用を復元の副作用として導入しない。

## 受入ケース案

| ケースID | 区分・前提 | 期待結果・確認 |
|---|---|---|
| `A08-01` | 正常。対応物理PCを外部媒体から構築 | 同一内蔵OSへ三機能を適用し、構成値を分裂させない |
| `A08-02` | 異常。採掘またはサーバの必要能力不足 | 変更開始前停止。機能を省いてマスター成功としない |
| `A08-03` | 正常。管理対象0件、登録後未接続 | 機能の準備と実接続の結果を区別 |
| `A08-04` | 異常。採掘側だけが再起動許可 | ホスト再起動なし。全機能の影響・許可条件を照合 |
| `A08-05` | 異常。実行中SSHあり、機能間の共有変更競合 | 記録保全と位置を照合し、両立不能なら開始しない |
| `A08-06` | 異常。内包機能の途中失敗・必須記録欠損 | 部分成功と全体未完了を分ける。無条件除去なし |
| `A08-07` | 正常。媒体を外して起動・再起動 | 内包機能と管理入口が内蔵OSだけで準備可能 |
| `A08-08` | 異常。保全物復元後に対象・ホスト鍵が不一致 | 操作を再投入せず、未確認の範囲を提示 |

## 追加の受入ケース案

`P08-MASTER`・`P08-COHOST`・`P08-BACKUP`の照合に対する設計。台帳・機能状態・容量・復元情報は所有者管理下の模擬fixtureを使用する。

| ケースID | 区分・前提と入力 | 期待結果・確認方法 |
|---|---|---|
| `A08-09` | 異常。FeatureReadinessがserver/miningの2件だけ、またはmaster重複 | READINESS_INPUTでMasterReadinessを生成しない。欠損機能の補完・縮退成功なし |
| `A08-10` | 正常。三機能の必須検査成功、target_count=0 | 機能の準備状態はready候補、遠隔接続・実行の成功証拠は0件として分離 |
| `A08-11` | 境界。cpu_total=8、予約2/server2/mining4とmining5、加算overflow | 式は前者だけ成立。後者とoverflowは拒否。前者も最低資源・実負荷の未達を充足扱いにしない |
| `A08-12` | 異常。同じ検証済み機器を別target_idへ登録、未確認要求ありで解除 | 重複登録・無確認解除を拒否。対象OSの削除を行わない |
| `A08-13` | 異常。復元した台帳は正しいが旧マスターの失効が未確認 | 遠隔変更を再開しない。構築成功と制御権の切替を別に判定 |

## 型とAPIの詳細案

`P08-MASTER`の台帳値と、選択済み対象の操作を分ける候補。登録による暗黙接続、一覧表示による接続・変更は行わない。

```rust
pub struct TargetId(String);
pub struct RegistryEntry {
    target_id: TargetId, host: String, port: u16, user: String,
    credential_ref: String, trust_ref: String, expected_machine_id: Option<MachineId>,
}
pub struct TargetRegistry {
    schema_version: u32, master_machine_id: MachineId, targets: Vec<RegistryEntry>,
}
pub struct FeatureReadiness {
    feature: String, configured: bool, applied: bool, verified: bool,
    status: String, checks: Vec<Check>,
}
pub struct MasterReadiness {
    machine_id: MachineId, features: Vec<FeatureReadiness>, target_count: u32, ready: bool,
}
pub struct MasterProgress { receipt: OperationReceipt, features: Vec<FeatureReadiness> }
pub struct CapacityInput {
    cpu_total: u32, cpu_reserved: u32, cpu_server: u32, cpu_mining: u32,
    memory_total_bytes: u64, memory_reserved_bytes: u64,
    memory_server_bytes: u64, memory_mining_bytes: u64, evidence: TargetEvidence,
}
pub struct CapacityProfile {
    profile_id: String, os_architecture_ref: String, workload_ref: String,
    minimum_cpu_reserved: u32, minimum_memory_reserved_bytes: u64,
    server_requirement_ref: String, mining_requirement_ref: String, measurement_ref: String,
}
pub struct RegistrySnapshot {
    registry: TargetRegistry, source_bytes: Vec<u8>, sha256: String, verification_ref: String,
}
pub enum RegistryEdit { Register(RegistryEntry), Remove(TargetId) }
pub struct RegistryChange {
    before_sha256: String, after: TargetRegistry, affected: Vec<TargetId>,
}
pub fn validate_registry(input: &Value) -> Result<TargetRegistry, ContractError>;
pub fn registry_change(current: &RegistrySnapshot, edit: RegistryEdit, pending: &[TargetId],
    known_machines: &std::collections::BTreeMap<String, MachineId>)
    -> Result<RegistryChange, ContractError>;
```

validate_registry・registry_changeはI/Oなし。RegistrySnapshotは収集器が読んだ元byte列・その検証済みhash・同じbyte列を解析した台帳と検証参照を保持し、任意hexだけで構築しない。registry_changeはそのsha256をbefore_sha256へ保持し、自身でhash実行物を起動しない。known_machinesは検証済みのtarget_id→MachineIdの観測集合で、台帳/今回登録のID以外を拒否する。expected_machine_idと観測済みIDのいずれからも同一実体の別名登録を検出する。pending対象の解除・同じIDの置換を拒否し、未確認のhostを名前だけで別機器と認定しない。台帳保存・対象側操作は別の実行契約とする。

マスターCLIの具体候補は`serverseed master targets list`、`serverseed master targets add`、`serverseed master targets remove`、`serverseed master run`、`serverseed master reconcile`の5入口。add/remove/runは対話入力・必要確認を要求し、--yesで対象・操作・構成を補完しない。list/reconcileは照合だけ。新構文の採用・全オプションと結果の確定は`D08-01`で行い、共通CLIが今すでに受理するとは扱わない。

CLI候補の共通フラグはP02-CLIの字句/値域へ照合し、--outputは本書で保存入口を追加しないため拒否する。--dry-runの台帳変更/runは計画だけ、list/reconcileは読取りだけとし、要求の配送・保存・取消を開始しない。target-file/request-fileの指定値は選択候補の入力であり、必須の対話選択・計画確認を代行しない。登録後の初回信頼確認は別工程。初期構築が作る台帳の0件と、台帳欠損/読取り不能を区別し、後者を空一覧へ補完しない。推奨は五入口のまま、一括変更/複数対象の暗黙実行は追加しない。

## 状態と中断の詳細案

`P08-COHOST`はserver→mining→masterの順に機能別の開始・適用・検証位置を保持し、ホスト全体のreadyは三機能の成功後だけ算定する候補。対象0件という事実は管理入口の準備と遠隔成功を分離する。

| 事象 | 停止と再開候補 |
|---|---|
| 開始前に一機能の能力不足 | 変更開始なし。二機能だけのmasterへ縮退しない |
| server成功後にmining失敗 | serverの実適用位置を保持、master工程not_started、全体ready=false |
| 各機能成功、媒体なし起動が未確認 | 構築未完了。機能検証だけで完了にしない |
| 遠隔要求が実行中・不明でホスト変更要求 | 台帳と対象側記録の照合まで新しいホスト変更を開始しない |
| 未確認要求ありで台帳解除 | 拒否。対象操作の取消・OS削除を代行しない |
| マスター喪失後に保全物復元 | 旧主体の失効と全未確認要求の読取り照合まで遠隔変更を保留 |
| 一機能の復旧成功 | その機能の結果だけ更新。全機能と統合の再検証なしでreadyにしない |

台帳更新と遠隔開始を競合させない候補順序は、マスター内の共通ガード取得→対象登録と未確認要求の照合→要求の固定・記録→対象側ガード取得→受理・実行。ネットワーク待ちの間にどのマスター側資源を保持するか、親停止時の寿命と永続照合は`D08-01`・`D09-02`でOS方式を確定する。順序表だけで分散トランザクション成立とは扱わない。

## 依存と互換性の詳細案

マスターはCore・server・miningを呼び、Coreからmasterへ依存させない候補。台帳と遠隔要求の形式変更では、マスターの版と対象の版の組ごとに読取り・配送・中断照合の可否を固定し、不明な相手へ変更を送らない。

| 対象 | 互換性の必須照合候補 |
|---|---|
| 台帳 | target_id・接続設定・信頼参照・expected_machine_idの意味、形式版と登録解除の保護 |
| 同居 | 同じAppPlan・MiningSample/制御契約、共通ガードと機能別結果。master専用の複製なし |
| 資源配分 | 実測CPU/メモリー、予約とserver/mining割当、systemdの実適用値・管理経路の負荷試験 |
| マスター保全 | 台帳版・主体ID・未確認要求・対象側結果の対応。秘密情報の非平文条件は担当ポリシー |
| 喪失時復元 | 新物理マスターの媒体構築、旧主体の失効、全対象の再同定、未確認要求照合 |

CPU割当の単位候補は論理CPU相当の整数、memoryはbytes。予約・server必要量の最小値を採用せずに計算式だけ通す実装は禁止条件の解消にならない。実測方法・負荷条件・不足判定の数値は`D08-02`で所有者へ提示して確定する。ゼロ値・未指定を自動算定と扱わない。

将来のcrate差し替えはregistry・cohost・remoteの順に呼出元を確認し、移行中に旧新主体が同じ対象へ変更を送れないことを先に検証する。旧主体の失効が未確認なら切替成功としない。冗長化・複数マスターへの拡張は今回の候補に含めない。

## 受入条件の充足照合案

MASTERは台帳/CLI/対象管理、COHOSTは三機能・共有資源・親操作、BACKUPは主体失効と未確認要求の復元照合へ対応する。対象が0件であっても、全機能の存在・準備状態を検証する。

| ケースID | 区分・固定入力 | 期待結果と確認 |
|---|---|---|
| `A08-14` | 異常。台帳編集前hashと保存直前hashが相違 | 競合として保存停止。新しい登録を消して上書きしない |
| `A08-15` | 異常。serverはverified、mining失敗、master未開始 | 全体ready=false、三機能それぞれの実位置を保持。serverの無条件削除なし |
| `A08-16` | 異常。新マスターはready、旧主体失効済みだが1要求の対象結果未照合 | 当該依存範囲の遠隔変更は保留。readinessだけで再開しない |

資源予約の最低値・実測条件、マスターCLI全構文/表示、旧主体の失効手段と秘密情報の復元は所有者の判断が必要。これらを管理者入力という自由文字列へ逃がさず、指定先・型・範囲・確認・不成立時停止まで確定してから仕様充足を判定する。マスターの物理媒体構築条件とCloud管理先の条件は混同しない。

## 入力と保存形式の詳細案

P08-MASTERのCLI候補を以下へ閉じる。共通フラグ・出力・終了分類はP02-CLI、ID制約は本書の台帳定義を使用する。ここで候補構文を定義しても共通CLIへ採用されたものとはしない。

| 構文候補 | 入力・返却 |
|---|---|
| master targets list | 追加引数なし。TargetRegistryを読取り、target_id順の配列を返す。接続・修復・保存なし |
| master targets add --target-file PATH | 新規一件のRegistryEntryを読み、既存台帳hashと比較して登録計画を作る。機器ID未検証なら登録準備状態として区別 |
| master targets remove --target-id ID | 台帳からの登録解除計画だけ。実対象の本体・OS・データを除去しない |
| master run --target-id ID --request-file PATH | P09-REMOTEのSubmitを一対象へ配送。install_osだけPhase 10を使用。対象集合・自動列挙・一括消去なし |
| master reconcile --target-id ID --request-id ID | P09のQueryだけ。計画・消去・変更の再投入なし |

RegistryEntryの全キー候補はtarget_id・host・port・user・credential_ref・trust_ref・expected_machine_id。portは1〜65535、hostはDNS名またはIPの単一文字列、制御文字・空白・先頭hyphen・user@hostを拒否する。userはPhase 10のUserInputの名前制約へ参照。credential_ref/trust_refは保護された登録済み参照IDだけで、秘密値・任意pathを受け取らない。

マスターCLIのdataはlist={registry_sha256,targets}、台帳変更={before_sha256,after_sha256,changed_target_ids}、run=DeliveryRecord、reconcile=ReconcileResultとする候補。list/reconcileの返却先はstdoutだけとし、--outputを拒否する前述の五入口へ統一する。結果保存用の別入口・既定ログ・明示成果物保存を追加しない。登録時の接続・信頼の初回確認は別手順として計画し、listへ通信を混入させない。

台帳編集候補の検査順序は登録済みマスター自身の識別→台帳版/hash→編集一件の形式→同ID/同実対象重複→pending対象→利用者の選択/計画確認→保存直前hashとする。Registerに既存target_idを渡して更新へ変換せずREGISTRY_DUPLICATE、解除対象不在はREGISTRY_MISSING。台帳変更dataのhashは確定した前後byte列、changed_target_idsは当該一件だけ。dry-runではafterの予定値を表示するだけで保存しない。reconcileのSSHによる問合せはP09のQueryであり、listのローカル読取りと区別する。

| ケースID | 固定入力 | 期待結果 |
|---|---|---|
| `A08-17` | hostに-user@host、秘密鍵本文をcredential_ref、未照合requestありの解除 | 入力または依存照合で拒否。SSH起動・台帳保存・実対象削除なし |

## 実行と復旧の確定候補

P08-COHOSTの一操作内で各機能を直列に適用し、同じ共通guardを維持する候補。台帳変更もローカル変更としてguard・operation・履歴確定を使用する。対象側の遠隔guardは別対象の排他であり、マスターのローカルguardを遠隔へコピーしない。

台帳の変更前後hashを計画へ固定し、保存直前に前hashを再照合する。対象に未照合の遠隔要求がある間はその登録解除・接続先/鍵参照変更を停止する。台帳保存の成功で遠隔側の旧主体失効や未確認操作の終了を推測しない。

CapacityInputのcpu_*は論理CPU相当数の整数とし、既存の算定式・受入入力と同じ単位を使う。メモリーは同型のbyte整数式、ストレージはP04の操作別必要容量へ照合する。cgroup等の強制制限の実現性も検証対象とし、入力値の和だけで保証しない。最低資源・負荷プロファイルは所有者が対象機器で測定して決める未達である。

CapacityInputのtotalを含む全数値は符号/範囲検証後にchecked加算し、予約+server+miningがtotal以下であることをCPU/memoryで独立に照合する。CapacityProfileの参照は同じtarget/OS/architecture/負荷の測定資料へ完全一致させる。測定値・最低要求・実効制限の取得不能を0へ補完せず、型上0が許される割当でも三機能の最低要求は別に満たす。論理CPU数のu32への変換overflowもCAPACITY_INPUTとする。

| ケースID | 固定入力 | 期待結果 |
|---|---|---|
| `A08-34` | 各CPU値はu32内だが和overflow、memory合計が1byte超過、別負荷profileの測定 | CAPACITY_INPUTまたは資源unknown。wrap/単位混用/別負荷の結果再利用なし |

マスター喪失後は旧master_idの失効証拠と全pending要求の対象側照合を先に確定する。新しい物理マスターには新MachineIdを発行し、旧IDを自動継承しない候補。同じ対象への管理権限の切替は信頼ポリシーの審査対象で、旧主体を失効できないなら遠隔変更を再開しない。旧マスターを残した二台運用へ移行しない。

| ケースID | 固定入力 | 期待結果 |
|---|---|---|
| `A08-18` | 台帳変更直前に別hash、機能適用中に遠隔要求未照合、新マスターへ旧MachineIdをそのまま復元 | それぞれ競合/依存停止。同じID・入力配分だけで制御権や資源保証を認定しない |

## 依存と受入の確定候補

P08-MASTERは台帳/対象選択、COHOSTは三機能の集約、BACKUPは管理情報と未照合要求の保全を内製する候補。サーバ/採掘機能はP06/P07の同じAPIを呼び、マスター専用の第二実装を作らない。

| 契約検査ID候補 | 必須条件 |
|---|---|
| P08-MASTER/registry.consistent | 台帳の版・hash・対象ID一意・同一実機重複なし |
| P08-COHOST/server.ready | サーバのconfigured/applied/verifiedと担当必須検査の全成立 |
| P08-COHOST/mining.ready | 採掘の同条件。対象機器が未対応なら全体ready不可 |
| P08-COHOST/resources.bounded | 実測資源・予約・上限・強制制御が一致 |
| P08-BACKUP/control.reconciled | 旧主体失効と未照合要求の対象側結果を確認 |

上記はローカルreadinessの集合で、遠隔0件でSSH成功を作らない。構築/台帳変更の返却はP04のOperationReceipt/ChangeFailureを使い、機能別失敗を一boolへ消さない。復元時にはregistry・trust参照・remote記録・保全物の版を対応付ける。

将来crate化するなら台帳validatorとreadiness集計だけを抽出候補とし、P06/P07/P09実行器は再利用する。新旧で同じ台帳・三機能結果・pending列を渡し、全検査/未達/解除拒否を一致させる。単一マスターという要件を多重化実装へ変更する権限は実装者にない。

| ケースID | 固定入力 | 期待結果 |
|---|---|---|
| `A08-19` | 三機能readyでもresources.boundedがunknown、旧主体失効は未確認 | 全体準備と制御再開を拒否。集計器差替え後も省略・成功化しない |
| `A08-20` | 台帳欠損、run --dry-run、request-fileはあるが対話選択なし | 欠損は未確認、dry-run配送/保存0回。ファイル指定だけで選択成立にしない |
| `A08-21` | 総量式は成立、最低予約を1単位下回る、制限総量は取得不能 | resources.boundedを通さない。入力の和・論理CPU数だけで実効容量を保証しない |
| `A08-22` | 新ID構築済み、台帳は正しいが一対象の旧認証が有効、別要求の結果不明 | 対象別の制御切替未完了。新マスターのreadyで遠隔変更を再開しない |
| `A08-23` | list/reconcileへ--output、既存IDのRegister、存在しないIDのRemove | CLI_OPTION_SCOPE/REGISTRY_DUPLICATE/REGISTRY_MISSING。保存・暗黙置換・登録解除の偽成功なし |
| `A08-24` | 三機能のverified=trueだが最低資源Checkがunknown、次にfail | master.statusはunknown/failed、全体ready=false。総量式passや機能boolで不足を隠さない |
| `A08-25` | 二対象の失効証拠を逆順、old/new ID同一、未確認要求一件残存 | MASTER_TRANSFERで保留。別対象の証拠・新構築成功を制御権移行の代用にしない |
| `A08-26` | 別target_idで同endpoint、別DNS名だが同じ検証済みMachineId | 登録/配送前にREGISTRY_DUPLICATE。登録検査から暗黙接続・既存台帳書換えなし |
| `A08-27` | verified=true/applied=false、全bool trueだが必須Check fail | 前者READINESS_INPUT、後者failed。入力のready/status自己申告で通過しない |
| `A08-28` | DNS名は別だがknown_machinesが同じ実体、Snapshotのbyte列とhash証拠が不一致 | 重複登録/スナップショット不成立で保存0件。純粋編集器の暗黙接続・hash実行・任意hash採用なし |
| `A08-29` | 台帳は二対象なのに結果は0件、resources.minimumだけ欠損、bounded自己申告pass | 台帳から対象数2を算定し資源判定はunknown。集合/根拠を再計算し偽のreadyを受理しない |

## 未決事項と成立判定

### マスターCLIの解析と応答全集合

新構文の型付き候補MasterInvocationはList/Register/Remove/Run/Reconcileのenumとし、Registerはtarget_fileの絶対PathBuf、RemoveはTargetId、RunはTargetIdとrequest_fileの絶対PathBuf、ReconcileはTargetIdとOperationIdを保持する。共通フラグ/上限/UTF-8/helpの解析はP02-CLIを再利用し、当該五入口の許可optionだけを追加する。全optionは一回だけ、余分な語・未定義のmaster操作はCLI_COMMAND/CLI_OPTION_SCOPE。検証済みfile入力を持つことと利用者選択の成立を別にする。

| 応答 | data候補の全集合と停止時 |
|---|---|
| list | registry_sha256: hex64、targets: RegistryEntry[]。一意なtarget_id昇順。不在/読取り不能は空一覧でなく開始前エラー |
| add/remove | before_sha256・after_sha256: hex64、changed_target_ids: TargetId[]。一件だけ。dry-runのafterは予定byte列のhash、実保存済みと表示しない |
| run | delivery: DeliveryRecord、request_may_have_arrived: bool。配送前エラーは共通error_stage/prerequisites、配送開始後は下記の既知位置を保持 |
| reconcile | ReconcileResult。記録不在/不明の根拠を保持し、成功したQueryを元操作成功へ変換しない |
| help/解析/未実装 | P02-CLIの専用data。未知の語や入力file内容をmessageへ連結しない |

対話候補は一覧表示→対象IDの明示選択→操作の選択→初回構成/保存先/導入値の必須入力→固定Plan提示→計画確認/追加消去確認→直前再照合の順。target-file/request-fileは選択候補にすぎず、--yesでこの選択列を省略しない。初回構成は三択を提示し、Cloudでmaster選択時は非対応理由で停止してserverへ変更しない。更新では保存済み構成を継承する。

各必須選択の待機候補は受付から300秒、未入力/EOF/対話不能/取消なら変更前blocked。入力は一行4 KiB以下、改行終端を除く完全一致で候補ID/操作名を照合し、trim/大小文字変換/近似選択なし。入力行は秘密値を受けない。入力エラーの再提示は同じ工程で最大三回、上限後は停止する。再提示で計画期限を延長せず、失効した計画は新しいIDと確認へ戻す。製品上の選択を開発の「承認」と混同しない。

固定診断候補はMASTER_INPUT/MASTER_SELECTION/MASTER_TIMEOUT/MASTER_NOT_READY/REGISTRY_DUPLICATE/REGISTRY_MISSING/MASTER_TRANSFER。表示は分類だけを固定し、認証/秘密値/生出力を含めない。終了分類はP02-CLIへ照合し、構文/設定は2、選択/準備不足は3、確定不整合は4、実行後の失敗は対象操作の分類を保持する。readinessの成功を配送の成功にしない。

| ケースID | 固定入力 | 期待結果 |
|---|---|---|
| `A08-30` | 五構文と余分option、fileだけ指定し無応答、Cloud master選択、dry-run台帳変更 | 解析/選択停止/適用拒否/予定hashだけを区別。自動選択・構成代替・配送/保存・送達成功の捏造なし |

MasterInvocationの宣言候補は次のとおり。共通要求型へ置くが、定義の担当は本節に統一する。pub enumのvariantを構築できることを検証済み入力の証明にせず、dispatch前にP02の解析/型検証を要求する。

```rust
pub enum MasterInvocation {
    List,
    Register { target_file: std::path::PathBuf },
    Remove { target: TargetId },
    Run { target: TargetId, request_file: std::path::PathBuf },
    Reconcile { target: TargetId, request_id: OperationId },
}
pub enum MasterData {
    List(MasterListData), Registry(MasterRegistryData),
    Run(MasterRunData), Reconcile(ReconcileResult),
}
pub struct MasterListData { registry_sha256: String, targets: Vec<RegistryEntry> }
pub struct MasterRegistryData {
    before_sha256: String, after_sha256: String, changed_target_ids: Vec<TargetId>,
}
pub struct MasterRunData { delivery: DeliveryRecord, request_may_have_arrived: bool }
```

各MasterDataをEnvelope.commandのmaster targets list/add/remove、master run、master reconcileへ対応させ、外部へvariant tagを追加しない。RemoteFailureは内部結果であり、生のContractErrorをdataへ直列化しない。配送開始後はそのdeliveryを保持し、未受理/到達不明でdeliveryがない場合だけ、検証済み要求のID/plan/hashからstatus=unknown・last_confirmed_status=null・operation_id=null・receipt=nullのDeliveryRecordを作る。これは受理の証拠を生成する処理ではない。errorは共通Diagnosticへ分類し、request_may_have_arrivedはP09の根拠に従い不明もtrue。開始後を開始前ErrorDataへ置換しない。

| ケースID | 固定入力 | 期待結果 |
|---|---|---|
| `A08-33` | 送信途中EOFでRemoteFailure.deliveryなし、target/hashは既知 | runのunknownと到達可能性trueを返し、共通errorsへ分類。受理成功・自動再送・生内部エラー公開なし |

### 同居資源の測定と割当算定

WorkloadProfile候補の全キーはprofile_id、対象機器/OS/driverの固定参照、全アプリ/サービス/imageの固定集合、採掘方式/機器/algorithmの参照、管理対象数、要求操作列、測定環境、測定時間/間隔、期待応答上限とする。空のアプリ集合を未選定アプリの負荷へ代用せず、最大1,000対象という台帳上限を1,000同時操作の保証にしない。初期の遠隔変更は直列一対象だけ。

測定方式候補は隔離環境で同じprofileを五回実行し、一回900秒、1秒ごとに管理/サーバ/採掘のcgroup CPU使用量とmemory.peak、管理CLI/Queryの応答時間を採取する。中央値でピークを隠さず、五回の最大値を算定へ使う。CPUは測定区間のusage_usec差/経過usecをmillicpuへ切上げ、逆行/不足/overflowは測定不成立。管理応答の候補上限は各1秒、失敗/未確認0件とする。実採掘・実機試験は安全ポリシーの許可境界が確定するまで実施しない。

最低CPU候補は各管理/サーバの最大millicpuに5/4を乗じ、1,000で割って論理CPU整数へ切上げ、最低1とする。最低memory候補は各最大memory.peakの5/4をbyte整数へ切上げ、さらに256 MiBを加える。採掘は残量の中で確認済みthreads/必要memoryを満たす範囲だけ。数値は測定方式の提案で、機器/負荷未指定のまま実最低値を生成しない。実際のCapacityProfileには測定元と算定結果を固定して採用する。

CPUの割当可能集合はonline、現在のCpus_allowed_list、有効cpusetの共通部分とする候補。論理CPU IDを昇順に管理→サーバ→採掘へ重複なく分配し、各必要数と総量式を照合する。systemdのAllowedCPUs/MemoryMaxと、Dockerの実効cgroupを読み戻し、全対象サービスが正しい親/範囲にあることを検証する。CPUQuotaやMemoryLowだけで専有・優先/実負荷応答を保証しない。SMT/共有memory/GPU/IOの干渉は同じ負荷試験へ含める。

管理/サーバ/採掘の一件でも有効制限不明、最低要求未選定、GPUの未確認共有、負荷profile不一致ならresources.enforced/measured/minimumをpassにせずready=false。資源不足を理由に一機能を削る縮退masterへ変えない。実際の最低値/対象負荷と強制方式の採用はD08-02に残す。

制限の読み戻しはunitの設定値だけでなく、全管理対象PID/containerの実所属と親からの実効値へ対応付ける候補。Docker daemonのunitとcontainerのcgroupが同じ親であると仮定しない。CPU online/許可集合・親制限・container再作成で所属が変われば旧測定/強制結果を失効させる。AllowedCPUsは他の全処理からの専有を単独では保証しないため、OS/対象外処理との共有を同じ負荷条件へ含め、未測定の干渉を予約済みの保証へ変換しない。

| ケースID | 固定入力 | 期待結果 |
|---|---|---|
| `A08-36` | unit設定は一致するがcontainerは別親、測定後CPU offline、対象外負荷が予約CPUを使用 | 実効範囲/測定条件を再照合しreadyを保留。設定値だけの強制確認・CPU専有/応答保証の捏造なし |

| ケースID | 固定入力 | 期待結果 |
|---|---|---|
| `A08-31` | CPUピーク1,001 millicpu、memory測定欠損、割当集合重複、四回成功/一回応答上限超過 | CPU候補は2、他は未成立/失敗を保持。平均値でピーク隠蔽・ゼロ予約・未測定機器の資源保証なし |

### 単一制御主体の移行候補

```rust
pub struct ControlTransfer {
    old_master_id: MachineId, new_master_id: MachineId, registry_sha256: String,
    target_ids: Vec<TargetId>, revocation_evidence_refs: Vec<String>,
    reconciliation_refs: Vec<String>,
}
pub struct ActiveController {
    schema_version: u32, master_machine_id: MachineId, credential_fingerprint: String,
    generation: u64, transfer_id: Option<OperationId>, target_machine_id: MachineId,
}
```

ControlTransfer/ActiveControllerは本書の同名全キーだけへ変換する型候補。generationは1以上かつ共通外部整数範囲内、更新はchecked旧値+1、初期だけtransfer_id=Noneを許容する。ControlTransferの平行配列は全件数と対象別資料を照合し、一対象でも未取得なら全対象完了にしない。型validatorと実認証/権限の失効確認を分け、reconcile_masterはこの型から失効処理を実行しない。A08-25/32に世代overflow/初期以外のnull/別対象の失効資料を含める。

マスター固有の保全方法候補master-v1を次の一組へ固定する。P04の暗号container/保存/隔離復元を再利用し、configuration-v1へ固有データを混入しない。

| 項目 | master-v1の候補契約 |
|---|---|
| 対象 | TargetRegistryの元byte列/hash、公開の信頼資料、資格の安全な参照IDと用途、全remote要求の元byte列/確認済み配送位置、対応する操作別intent/snapshot/照合証拠 |
| 除外 | 秘密鍵/token/passphrase、OSのPID/lock/監督unit、現在の実行権限。旧MachineIdは証拠として保持し、新マスターのIDへ上書きしない |
| 取得 | 共通guard下で新規台帳編集/Submit/Cancel送信を停止し、採取集合を列挙・固定する。進行中の対象処理が同時点のsnapshotになったとは保証せず、最後の確認位置とunknownを保持する |
| 隔離検証 | 全byte/hash/IDの対応、台帳validator、remote記録validator、参照の閉包、秘密値不在を検査。対象接続/Submit/失効/新主体登録は0回 |
| 復元確認TTL | 候補3,600秒。形式/参照/暗号/鍵/台帳hash変化で無効。TTLは復元可能な形式の確認期限であり、対象の最新状態・制御再開の許可ではない |
| 実復元 | 新物理マスターの隔離された証拠領域へ読み取り資料として配置し、通常のoperation/lock/ActiveControllerへコピーしない。対象全件の要求索引を照合してからControlTransferへ進む |

バックアップ時点より後の要求が保存物にない可能性を前提にし、既知pendingだけでなく各対象の旧master_idに束縛された要求索引を読み取る。索引取得/全件照合が不能なら当該対象の変更は保留する。失われた資格は独立した回復資格で扱い、保全物から架空の秘密値を復元しない。構成検証と制御権移行を別工程に保つ。

| ケースID | 固定入力 | 期待結果 |
|---|---|---|
| `A08-35` | 保存後の追加要求が対象に存在、保全物に旧PID/秘密鍵、復元TTL内だが失効不明 | 全件索引照合/保全形式拒否/制御再開保留。既知pendingだけの照合・旧実行権限コピー・鍵の架空復元なし |

喪失時復旧の第一比較候補は、対象側の固定入口がActiveControllerを一件だけ照合する方式。全キーはschema_version: 1、master_machine_id: MachineId、credential_fingerprint: string、generation: 正整数、transfer_id: OperationIdまたは初期時null、target_machine_id: MachineId。対象側のroot管理領域に保持し、SSHで認証された資格のfingerprintと主体IDの双方を確認する。要求内のmaster_machine_idだけで権限を決めない。初回enrollment/更新の権限は担当安全ポリシーの別の所有者管理資格へ照合する候補。

| 工程 | 必須の照合・変更範囲候補 |
|---|---|
| 新物理マスター準備 | 外部媒体構築、新MachineId、隔離復元/台帳検証。旧ID・lock/PID・秘密鍵の無条件継承なし。まだ通常の遠隔変更を送らない |
| 対象別の照合 | 独立した回復資格で接続信頼/実対象/現ActiveControllerを読み、旧要求のoperation/実状態を回収。未確認要求は再投入しない |
| 旧主体の停止 | 旧資格による新規Submitの受付を止め、進行中要求の完了/保全停止を照合。Queryの必要性と新変更の禁止を分ける |
| 一件の制御切替 | 対象の共通排他の下で旧generation/hashを再検査し、旧/新資格・対象・transfer IDを固定した新ActiveControllerへ原子的に置換/同期。新generationは旧+1のchecked値 |
| 認証の失効確認 | 旧資格の実効認証/変更権限の失効、新資格の固定入口照合、pending全件の結果を別に検証。台帳値だけで失効済みにしない |
| 再開 | 対象別の失効/照合資料をControlTransferへ結合し、所有者の再開判断後だけ当該対象の新Planを許可 |

この比較候補は一時的な二制御主体を許可しない。新資格登録時に旧資格も残る場合でも、固定入口の実効制御権は一件だけであること、迂回したsudo/旧wrapperがないことを審査する。SSH資格そのものの失効も別に確認する。回復資格未設定/旧主体の追加権限不明/対象SSH不通なら変更を再開しない。外側の回復経路と具体的な認証/昇格の許可は未採用で、通常のrestoreによる自動登録を行わない。

切替途中の応答欠損ではActiveControllerの実世代/hashと対象記録をQueryで取得し、旧状態へ書き戻したり、generationを再加算したりしない。結果が対象ごとに異なる場合はその依存範囲を保留し、全台切替成功と表示しない。旧秘密鍵の漏えい/喪失から、既に取得された情報を回収できるとは保証しない。

| ケースID | 固定入力 | 期待結果 |
|---|---|---|
| `A08-32` | 切替同期後に応答欠損、旧資格のsudo迂回あり、旧要求不明、新旧同ID | 実世代照合/権限失効未達/要求照合待ち/拒否。世代二重更新・旧ID復元・二台からの変更再開なし |

| 判断ID | 決定・確認が必要な事項 | 影響 |
|---|---|---|
| `D08-01` | [マスターCLI](#マスターcliの解析と応答全集合)と台帳に案あり。全応答子型・初回信頼/登録の実手順と採用判断が残る | 初期設定・対象管理 |
| `D08-02` | [同居測定と算定](#同居資源の測定と割当算定)に案あり。機器/負荷/許容遅延、強制制御の成立根拠と測定された最低値は未確定 | 同居・更新 |
| `D08-03` | [単一主体移行](#単一制御主体の移行候補)に世代/旧資格失効/照合案あり。復旧資格・失効の実効性と固有保全の復元方法を安全ポリシーに対応させ採用判断 | マスター喪失・復旧 |

同居評価の値・操作上限・試験条件と共通契約が未確定で、完全仕様は未成立。SSH操作の実装・製品の構築・受入試験は今回実施しない。
