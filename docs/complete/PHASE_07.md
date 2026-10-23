# Phase 7 完全仕様案: マイニング構成・監視・自動復旧

[文書一覧](../DOCUMENT_INDEX.md) | [担当工程](../ROADMAP.md#phase-7-マイニング構成監視自動復旧)

本書は`B13`の審査用本文で、採用未承認。[案の適用範囲](PHASE_01.md#案の適用範囲)に従う。担当は`REQ-001`・`REQ-002`・`REQ-004`・`REQ-007`・`REQ-008`・`REQ-009`・`REQ-011`・`REQ-012`。[マイニング構成仕様](../mining/specification.md)と各運転ポリシーの既存定義を変更しない。

## 契約と共通参照

| 契約ID | 担当 |
|---|---|
| `P07-DEVICE` | CPU・GPU・ドライバー・センサーの能力判定 |
| `P07-CONFIG` | 採掘設定と監視パラメーターの具体化案 |
| `P07-RUN` | CPU/GPU別の監視・制御・復旧 |
| `P07-UPDATE` | 採掘プログラム・設定の更新と失敗時復旧 |

共通契約（全フェーズB13）は`P02-PARSE`・[解析](PHASE_02.md#p02-parse)、`P02-MODEL`・[型](PHASE_02.md#p02-model)、`P02-STORE`・[排他と記録](PHASE_02.md#p02-store)、`P02-EXEC`・[外部起動](PHASE_02.md#p02-exec)、`P04-UPDATE`・[更新](PHASE_04.md#p04-update)、`P04-BACKUP`・[保全](PHASE_04.md#p04-backup)、`P04-APPLY`・[資源適用](PHASE_04.md#p04-apply)。ホスト共有条件は`P08-COHOST`へ接続する。

## P07-DEVICE

採用案の能力行は、対象ID、CPU/GPU識別、ドライバー・カーネルの識別、採掘ソフトの固定資材、algorithm、検査結果、温度・電力取得の可否を持つ。機種名の一致だけで採掘対応としない。

選択した運転方式ごとに必要な全能力を検査し、非対応・取得不能・未検証を分ける。取得不能値を0℃や正常なhashrateへ置換しない。能力表の版・外部呼出し・GPU電力制御の許容値は`D07-01`で確定する。[EX-004・EX-005](../DEPENDENCY_EXCEPTIONS.md#外部依存の例外一覧)は未確定のまま維持する。

## P07-CONFIG

設定は[採掘サービスと設定](../mining/specification.md#採掘サービスと設定)の担当ファイルへ接続する。本案では次を採用候補とする。既存の実設定の受理範囲を変更したものではない。

| 入力候補 | 型・範囲・必須性 |
|---|---|
| 形式版・方式 | schema_versionは必須integerの候補版1。modeは必須でcpu・gpu・combinedのいずれか |
| pool・algorithm・wallet公開アドレス | 選択エンジンごとに必須。形式の厳密な範囲は対象プログラムとの契約照合待ち |
| CPUスレッド | CPU使用時に必須integer、1〜検出した利用可能論理CPU数。0を自動選択としない |
| GPU集合・電力設定 | GPU使用時に機器ID集合を必須。重複を拒否。電力値の型・範囲は機器能力へ照合 |
| 監視間隔 | 必須integer秒、1〜300。上限や単位なしの値を受けない |
| 起動待機・停止判定 | 必須integer秒、起動待機1〜600、採掘停止の継続時間1〜600 |
| 再試行 | 必須integer回、0〜10。待機秒は1〜600。0回なら自動復旧要求を出さない |
| 温度 | 取得可能機器では停止・再開の値を必須とし、再開値が停止値より小さいことを検査。機器別の値域はD07-01 |
| 許可・時間帯 | サービス復旧・OS再起動・更新対象・実行時間の既存項目へ対応。キー・時刻帯・省略時動作はD07-02 |

数値範囲は運転値の推奨や機器の安全温度を決めるものではなく、候補入力契約である。温度取得不能時の動作・温度差と連続安定時間の条件は、機器資料・ポリシーとの整合を確認して所有者が決定する。無根拠の共通温度をすべての機器へ適用しない。

## P07-RUN

採用案の運転状態はCPU・GPUごとに、準備・起動待ち・稼働・再試行待ち・温度停止・停止・確認不能を内部状態として持つ。これを製品の総合状態名へ転用しない。

1. 構成・機器・実行基盤と起動条件を照合して、該当サービスだけを開始する。
2. 起動待機中の一時的なhashrate不足と、待機後の継続停止を分ける。
3. プロセス、algorithm、接続、hashrate、取得可能な温度を同じ観測区間へ対応付ける。
4. 異常原因を分類し、事前許可と再試行上限、温度の停止・再開条件に沿って当該エンジンだけを制御する。
5. 解消不能時は該当採掘を停止して結果を記録し、管理経路の維持を検査する。

共通の変更操作が実行中・中断照合待ちなら、その操作と競合するサービス変更を開始しない。監視の観測だけは読み取りとして分ける。運転中のサービス復旧を、独立したOS更新・本体更新・バックアップの開始へ拡張しない。

通信断・pool障害・低hashrate・過熱・温度取得不能が同時に起きた場合の優先順、hashrateの閾値と丸め、連続時間のリセット規則は`D07-03`の未決事項。プロセス生存だけで稼働としない既存条件を維持する。

## P07-UPDATE

入口から選択された更新計画に、採掘プログラム・設定とOS・カーネル・ドライバーを分けて列挙する。正常版の保持、候補の採用条件・真正性・互換性、共通保全、計画確認を終えた範囲だけを適用する案とする。

採掘プログラムの置換後は同じ運転入力で起動・接続・hashrate・取得可能な温度を検証する。失敗時は確認済み正常版と対応設定へ復元し、元の更新を失敗として返す。データ形式や設定版の不一致がある場合は無条件復元しない。OS再起動と同居機能への影響は[運用条件](../POLICIES.md#マイニングの自動更新と再起動)を参照する。

サービス起動・停止は各120秒、全更新はP04-UPDATEの上限内、再試行なしを候補とする。運転監視の許可された再試行と、更新処理の再投入は別に数える。

採掘更新の審査単位はminer MaterialRef・設定hash/版・algorithm・driver/kernel能力行・正常版/復元元の組とする候補。第一候補はminer/configurationだけの変更をOS/kernel/driverから別stepへ分ける方式で、共有driverの運転中差替えは行わない。候補の実行前に利用/再配布条件と手数料を台帳へ対応付け、同梱不可の場合の取得経路も別に決定する。模擬試料・fixtureで更新調整を検査し、実採掘や未承認の実機制御を適合証拠にしない。

MiningUpdateBinding候補の全キーはbefore_miner・after_miner（MaterialRef）、before_config_sha256・after_config_sha256、before_config_revision・after_config_revision、capability_ref・restore_method（ContractRef）、engines（Engine[]）。選択外engineを含めず、更新前後の設定と能力を同じalgorithm/driver/kernel条件へ照合する。復元には正常版資材とその設定の組が必要で、新設定を旧minerへ渡すだけの復旧は拒否する。運転中のretry回数と更新stepの再投入回数を別に保持し、更新は常に自動再投入0回。復元の成功は元の更新失敗を変更しない。

## 内製単位・API案

配置候補は`src/profiles/mining/device.rs`・`config.rs`・`monitor.rs`・`update.rs`。検出結果の評価・監視状態機械・更新調整は内製、採掘エンジンとドライバーは外部例外。新規内製の難易度は機器評価・監視が中、機器制御・復旧が高とする候補。

| 宣言案 | 入出力と副作用 |
|---|---|
| `mining_plan(config: &MiningConfig, target: &TargetEvidence, evidence: &CapabilityEvidence, identity: PlanIdentity) -> Result<PlanDraft, ContractError>` | 同じ対象/起動世代へ束縛された検証済み能力と親識別から未封印計画を返す。ID/時刻/hashを生成せず、用途を自動選択しない |
| 監視評価 | 宣言は[型付きevaluate_monitor](#型とapiの詳細案)へ集約。前状態・試料・設定を入力にする純粋処理 |
| `control(ctx: &mut ExecutionContext, request: &ControlRequest) -> ChangeResult<ControlReceipt>` | 許可・競合・対象と同じ要求キーを再照合し、該当サービスだけを変更 |

型はP02-MODEL。サンプル・状態遷移・actionの全スキーマはD07-03で審査する。旧実装はなし。将来のモジュール抽出・crate差し替えでは、fixtureの観測列を同じ順で与え、新旧の判定・待機・副作用・復旧を比較する。未確認の温度・機器契約を互換性の既定値にしない。

## データ契約の具体化案

mining_planの親identityは入口が生成済みの値だけを受ける候補。standaloneではprofile=Mining、物理マスターへ組み込む場合は同じ親identityのprofile=Masterとし、資源owner=miningを変えない。能力/対象/構成の不一致は計画前拒否。生成後の封印はP03だけで行い、採掘生成器から外部hash/適用を開始しない。A07-01/02にidentity欠損、異なる親対象、Master内包時の単一親IDを含める。

`P07-RUN`の純粋評価へ渡す観測列と制御候補を具体化する。機器・minerのAPIからの単位変換と能力表は`P07-DEVICE`が担当し、値がない場合の推測・0埋めをしない。

| 型 | 全キー・値域の候補 |
|---|---|
| MiningSample | engine: cpu・gpu、device_ids: string[]、epoch: string、elapsed_ms: 0以上integer、process_alive・connected: booleanまたはnull、algorithm: stringまたはnull、hashrate_milli: 0以上integerまたはnull、temperature_milli_c: integerまたはnull、temperatures_by_device: device_id→integerまたはnullのobject |
| MonitorState | engine: cpu・gpu、epoch: string、state: preparing・warming・running・retry_wait・thermal_stop・stopped・unknown、entered_ms: 0以上integer、last_sample_ms: integerまたはnull、attempts_used: 0以上integer |
| ControlProposal | engine: cpu・gpu、action: none・start・stop・restart、reason: 固定コードstring、sample_epoch: string、sample_elapsed_ms: 0以上integer、state_before・state_after: 上記内部状態 |

hashrate_milliは1/1,000 hash/s、temperature_milli_cは1/1,000℃の整数を候補とする。変換元の精度・丸めを機器能力表で確認できない場合はnull。機器ごとの温度・電力の上下限を共通の推測値で決めない。epochは監視開始・再起動で変わる採取世代で、異なるepochのelapsed_msを減算しない。同じepochで時刻が後退・重複した試料を連続正常時間へ算入しない。

評価順と全状態の遷移は[状態と中断](#状態と中断の詳細案)へ集約する。最初の成立条件を主因とし、他の異常もchecksに保持する。

起動待機・再試行待機・停止保持を飛び越す概説用の第二遷移表は設けない。温度は機器別ポリシーへ照合し、未知試料・未確定制御の扱いも参照先だけで判断する。

再試行の計数と停止保持は[状態と中断](#状態と中断の詳細案)に集約する。観測再取得・拒否された制御要求を実開始回数へ数えず、プロセス生存だけで回数を0へ戻さない。

evaluateは観測・設定から候補を返すだけで、制御の許可にはならない。controlが同一試料世代、サービスの対象、許可・共通排他と機器状態を再照合する。CPU側のrestart候補をGPU停止やホスト再起動に変換しない。全状態×事象の遷移、温度不明時動作と監視用記録の確定方式はなお未達である。

## 接続と責務の照合

監視のSample・MonitorStateは[P02-STATE](PHASE_02.md#p02-state)の総合管理状態と別の機能内データで、runningという語をMANAGEDの証拠へ転用しない。観測評価だけでは共通operationやログを自動保存せず、許可された運転制御・稼働記録の担当契約に従って記録する。監視記録の未確定形式を共通operation.jsonへ押し込まない。

service単位の復旧も[P02-STORE](PHASE_02.md#p02-store)の排他・未完了照合へ接続する。競合を検出した制御要求は保留し、通常の更新を運転復旧として実行しない。親変更中に子の監視プロセスだけがロックを別取得する構成、プロセス終了だけで未確認の変更を解除する構成を避け、具体的な実現方式は`D02-04`・`D07-03`で確認する。

マスター上では[P08-COHOST](PHASE_08.md#p08-cohost)の予約資源・サーバ影響・遠隔操作を検査する。GPU不調を理由に共有driverやカーネルを運転制御から更新せず、ホスト全体の変更は別の確認済み保守計画へ戻す。採掘ソフトの正常版へ戻せても、OS/driverまで復元したとは報告しない。

## 受入ケース案

| ケースID | 区分・前提 | 期待結果・検証方法 |
|---|---|---|
| `A07-01` | 正常。CPUのみ・GPUのみ・併用 | 選択した機能・設定だけを適用。片方の制御で他方を不要に停止しない |
| `A07-02` | 異常。未対応機器・未確認driver・未採用miner | 適用前停止。別方式・別構成を自動選択しない |
| `A07-03` | 正常。起動直後の低値、その後の接続・稼働 | 起動待機を区別。プロセス生存のみで成功にしない |
| `A07-04` | 異常。pool障害・通信断・停止の継続 | 許可された回数・待機で当該採掘だけ復旧。上限後停止 |
| `A07-05` | 境界。温度停止値・再開値・安定時間 | 閾値と差・継続条件を模擬観測で照合。未定義なら試験保留 |
| `A07-06` | 異常。温度不明、複合障害、操作中の復旧要求 | 確認不能と競合を区別し、無確認の制御なし |
| `A07-07` | 異常。更新後検証失敗・旧設定非互換 | 許可範囲の復旧だけ。元更新の失敗を維持 |
| `A07-08` | 異常。timerから更新開始、採掘側だけ再起動許可 | 入口・ホスト共有条件に照合して開始しない |

## 追加の受入ケース案

`P07-RUN`の観測列を入力にする設計。温度の数値は模擬機器の評価fixtureだけに用い、実機へ推奨・適用する温度ではない。実採掘は行わない。

| ケースID | 区分・前提と入力 | 期待結果・照合方法 |
|---|---|---|
| `A07-09` | 境界。模擬停止値80,000、再開値70,000 milli_c。入力79,999/80,000 | 既知の正常観測が揃う前者は過熱扱いにせず、後者はthermal_stop。接続復旧より過熱を優先 |
| `A07-10` | 異常。同じepochでelapsed_ms=1000の後に999、または別epoch | 連続正常時間を加算しない。unknownとして制御候補none |
| `A07-11` | 異常。温度null、プロセス生存と接続はtrue | running・再開へ補完しない。担当の温度不明時動作が未確定なら保留 |
| `A07-12` | 境界。再試行上限まで受理済み、次に復旧候補、別途ロック競合で拒否 | 上限後はstopped。拒否だけではattempts_usedを増減しない |
| `A07-13` | 異常。CPU側restart、GPUは正常、マスターで遠隔操作中 | GPU停止・ホスト再起動・driver更新を追加せず、共有排他へ照合して競合時は保留 |

## 型とAPIの詳細案

`P07-RUN`では観測の不在と0を型で分け、評価の入力へ前状態を必須にする候補。既存evaluate案のsamples/configだけでは再試行回数・温度停止の継続を再現できないため、状態を引数として明示する。

```rust
pub enum Engine { Cpu, Gpu }
pub enum MonitorPhase { Preparing, Warming, Running, RetryWait, ThermalStop, Stopped, Unknown }
pub enum ControlAction { None, Start, Stop, Restart }
pub struct MiningSample {
    engine: Engine, device_ids: Vec<String>, epoch: String, elapsed_ms: u64,
    process_alive: Option<bool>, connected: Option<bool>, algorithm: Option<String>,
    hashrate_milli: Option<u64>, temperature_milli_c: Option<i64>,
    temperatures_by_device: std::collections::BTreeMap<String, Option<i64>>,
}
pub struct MonitorState {
    engine: Engine, epoch: String, state: MonitorPhase, entered_ms: u64,
    last_sample_ms: Option<u64>, attempts_used: u32,
}
pub struct ControlProposal {
    engine: Engine, action: ControlAction, reason: String, sample_epoch: String,
    sample_elapsed_ms: u64, state_before: MonitorPhase, state_after: MonitorPhase,
}
pub struct ControlRequest {
    target: TargetEvidence, proposal: ControlProposal, config_sha256: String, request_id: String,
}
pub struct TemperaturePolicy { stop_milli_c: Option<i64>, resume_milli_c: Option<i64> }
pub struct MonitorConfig {
    interval_seconds: u32, warmup_seconds: u32, failure_seconds: u32,
    retry_wait_seconds: u32, retry_limit: u32,
    temperature_policies: std::collections::BTreeMap<String, TemperaturePolicy>,
    temperature_stable_seconds: u32, temperature_unknown_action: String, allow_service_recovery: bool,
}
pub struct MaintenanceWindow {
    timezone: String, start_minute: u16, end_minute: u16,
    allow_auto_update: bool, allow_os_reboot: bool, update_targets: Vec<String>,
}
pub struct EngineConfig {
    capability_ref: ContractRef, material_ref: MaterialRef, algorithm: String, pool: String,
    wallet: String, backup_pools: Vec<String>, device_ids: Vec<String>, threads: Option<u32>,
    power_milli_watts: Option<u64>, minimum_hashrate_milli: u64,
}
pub struct MiningConfig {
    schema_version: u32, mode: String, cpu: Option<EngineConfig>, gpu: Option<EngineConfig>,
    monitor: MonitorConfig, maintenance: MaintenanceWindow,
}
pub struct MonitorMemory {
    target: TargetEvidence, config_sha256: String,
    state: MonitorState, failure_since_ms: Option<u64>,
    cool_since_ms: Option<u64>, healthy_since_ms: Option<u64>,
    last_control_receipt: Option<String>, pending_control: Option<String>,
    reserved_retry: Option<String>, stop_latched: bool,
}
pub struct MiningUpdateBinding {
    before_miner: MaterialRef, after_miner: MaterialRef,
    before_config_sha256: String, after_config_sha256: String,
    before_config_revision: u32, after_config_revision: u32,
    capability_ref: ContractRef, restore_method: ContractRef, engines: Vec<Engine>,
}
pub struct ControlReceipt {
    request_id: String, epoch: String, action: ControlAction,
    accepted: bool, started: Option<bool>, elapsed_ms: u64, steps: Vec<StepReceipt>,
}
pub struct Evaluation {
    next: MonitorMemory, proposal: ControlProposal, checks: Vec<Check>,
}
pub fn evaluate_monitor(previous: &MonitorMemory, sample: &MiningSample, config: &MiningConfig)
    -> Result<Evaluation, ContractError>;
pub fn observe_control(previous: &MonitorMemory, request: &ControlRequest, receipt: &ControlReceipt)
    -> Result<MonitorMemory, ContractError>;
```

MonitorState・MiningSample・ControlProposalの全項目は[観測データ](#データ契約の具体化案)へ参照し、engineはEngine、時刻は検証済みu64、温度はi64、hashrateはu64を内部表現にする。外部整数の上限も保持する。evaluate_monitorは純粋処理で、observe_controlだけが検証済み要求・実施receiptを照合して次状態を生成する。last_control_receiptは重複消費防止用ID、pending_controlは未確定の要求ID。提案を出しただけでは回数・開始時刻を確定しない。

MiningConfig/MonitorConfigのString列挙と整数は[入力形式](#入力と保存形式の詳細案)の範囲へ検証し、Rust型の全値域を許可しない。ControlProposal.state_beforeは入力memory.state.stateへ一致させ、state_afterは純粋評価のnextと一致させる。前提が違うproposalをcontrolへ渡さず、機器別温度の集合欠損を総合温度だけで補わない。

| ケースID | 固定入力 | 期待結果 |
|---|---|---|
| `A07-38` | proposalのstate_beforeだけ相違、u32内だが設定の秒上限超過、温度の機器一件欠損 | MONITOR_INPUT/設定拒否/unknown。型変換だけの検証済み認定・総合温度への代替なし |

監視器内部のAPI候補は上記に統一する。MonitorMemoryの保存・再起動後の継承は別の契約が必要で、経過時刻の値だけを永続化して同じepochとして復元しない。

MonitorState.entered_msはengine/epoch/stateの組が変わった試料時刻だけで更新する。同じretry_waitを継続する正常な試料では不変、last_sample_msだけを進める。最初の状態は明示セッション開始時刻、時刻逆行/epoch不一致では既定のunknown処理へ移し、旧epochのentered_msを新しい待機計算へ使わない。failure_since/cool_since/healthy_sinceはそれぞれの連続条件の開始時刻であってentered_msの別名ではない。checksは検証済み共通Checkだけを返す。

| ケースID | 固定入力 | 期待結果 |
|---|---|---|
| `A07-36` | retry_wait entered=1000、試料2000/3000/6000、待機5秒 | enteredは1000のまま、6000でだけ期限条件成立。試料ごとの待機延長・連続条件の時刻混用なし |

保存レコードが対応する操作/計画の対象と設定hashを、MonitorMemory.target/config_sha256へ復元時に照合する候補。レコードに未定義のトップレベルtargetを追加せず、対応する操作記録を取得できなければ復元を保留する。要求の照合はこの確認済み値を参照し、request自身のtarget/hashだけで正しさを認定しない。設定変更は新しい監視セッションの明示確認と保存互換の審査へ戻し、記憶内のhashだけを差し替えて旧receiptを再利用しない。

監視対象の同一性比較候補はTargetEvidenceのobserved_atを除く六証拠のobserved値を比較し、時刻だけの更新を別機器としない。通常運転では六証拠を取得できない場合はMONITOR_TARGETとして制御を保留する。boot/実行環境の変化は同じdiskでも新監視世代の照合を要求する。epochはP02の新規OperationId生成と同じ一意な表記を使用し、監視セッションごとに新しく生成する。新epochでも保存済み回数/停止保持/未確認要求を引き継ぐ条件は維持し、epochの新規生成だけをリセット承認にしない。OS乱数の未選定はD02-01の依存として残す。

ControlRequest候補の全項目はtarget（TargetEvidence）・proposal（ControlProposal）・config_sha256・request_id。request_idは本書のengine/epoch/sample_elapsed_ms/action/config_sha256の組を一意に束縛する参照で、衝突時はMONITOR_COLLISION。evaluate_monitorは一試料につき状態表を上から一度だけ評価し、未確定要求中・保持中・unknownの条件を後段の正常試料で上書きしない。observe_controlは同じ要求ID/epoch/actionを一度だけ消費し、別要求のreceiptはMONITOR_RECEIPTとして状態不変で拒否する。制御開始後の失敗では実開始の確証または未確認をpartialへ保持し、再試行枠を未消費へ戻さない。

ControlReceiptのstepsはStartならcontrol.start、Stopならcontrol.stop、Restartならcontrol.stop/control.startの順に全工程を保持する候補。各StepReceiptは同じP07-RUN契約/要求へ結合し、拒否/後続未開始もnot_startedとして省略しない。accepted/startedは要求全体の受付/変更開始であり、実起動確認はcontrol.startのverifiedと必須Checkの成功から照合する。observe_controlはreceipt.stepsからこの証拠を検査し、外側のstarted=trueだけでwarmingへ進めない。stopだけ開始後の失敗は回数を消費した停止/不明として保持する。

observe_controlはrequest.request_idとreceipt.request_id、proposalのengine/epoch/actionと保存memory、request.config_sha256と固定設定を照合する候補。同じIDでも別対象・設定・試料に束縛されている要求は拒否する。accepted=falseならstarted=false、accepted=trueでもstarted=nullは未確認としてpendingを保持し、started=trueの確認なしでwarmingへ遷移しない。初回以外の開始可能性が残る場合は回数を保守的に予約し、実開始なしの確証なしに枠を再利用しない。

制御receiptの候補照合ではelapsed_msを要求試料のelapsed_ms以上かつ同じepochへ限定する。action=noneからControlRequestを生成せず、noneのreceiptもMONITOR_RECEIPT。accepted=false/started=falseは未開始の確証としてpendingを解除するが、処理済みrequest_idを保持して同じ試料の再発行・再消費を防ぐ。stopのstarted=trueは停止要求の開始だけで、process_alive=false等の実停止確認なしに停止完了としない。start/restartの開始とstopの開始は別の遷移へ結び付け、stopのreceiptからwarmingを作らない。

RestartはOSの未定義なrestart argvへ直結せず、同一要求・guard・120秒の親予算内でStop→実停止確認→Start→実開始確認の二工程へ分解する候補。対象は同じengineの固定unitだけ、P04のunit stop/start実行契約を再利用する。stop失敗/停止不明ではstartを呼ばず、start未開始の確証と停止済み事実を別に保持する。制御全体のstartedは最初の変更要求を実際に開始した時点でtrueとし、再試行枠を消費する。warmingへの遷移にはさらにstart工程の実開始証拠が必要で、集約receiptのstarted=trueだけでは足りない。stop/startの各StepReceiptと検証参照をControlLedgerへ結合する。

二工程とも同じ外側期限を使い、各工程へ120秒を新たに与えない。親の停止/記録予約を差し引いて次工程の開始可否を判定する。stop成功後に残量不足なら停止済みの失敗位置を保持し、自動start・元稼働への無確認復帰を行わない。None/Start/StopをRestartへ自動昇格させない。

| ケースID | 固定入力 | 期待結果 |
|---|---|---|
| `A07-37` | restartでstop開始後不明、stop完了後に予算不足、start開始確証なし | start0回/停止済み失敗/warming不可。再試行枠を未消費へ戻さず、二重予算・任意restart argvなし |

## 状態と中断の詳細案

`P07-RUN`の全内部状態を、次の優先規則へ照合する候補。まず世代・対象・時刻の不正、次に未確定要求の照合、次に過熱・温度不明、続いて状態固有行を評価する。未確定要求の照合中も温度の事実を保持するが、新規要求の生成へ進めない。同じ試料から二つの制御要求を出さない。

運転観測の必須性は段階別とする候補。warmingの待機期限未満はprocess/接続/algorithm/hashrateが未準備でも待機を維持し、過熱・必要温度不明の優先規則は省略しない。それ以外ではprocess_aliveを必須とし、falseの確証がある停止中は接続/algorithm/hashrateを開始前の必須条件にしない。process_alive=trueの稼働確認には後三観測も必須。warming期限到達後の未起動は既知障害として継続時間へ、必須観測の取得不能は下表のunknownへ区分する。停止中の未取得hashrateを起動禁止へ、取得不能processを未起動の確証へ変換しない。

| 対象状態・条件 | 次状態と制御 |
|---|---|
| 全状態、対象・epoch・時刻が不正 | unknown、none。連続時間を全て無効化 |
| 全状態、未確定controlあり | 新しい制御はnone。過熱の事実と必要な保護を保持し、既存要求/実状態を照合する。同じengineへ二重のstop/startを発行しない |
| 全状態、有効温度が停止値以上 | thermal_stop、稼働中ならstop、停止確認済みならnone。プロセス状態も不明ならunknown/noneとして無確認の制御をしない |
| 全状態、必要温度がunknown | unknown、none。停止要否の安全判断は担当ポリシー確定まで保留 |
| 全状態、当該段階で必須の運転観測がunknown | unknown、none。連続時間を無効化し、回数/停止保持を維持。確認不能を障害確証としてrestartしない |
| stop_latched=true | 起動/restart提案なし。別の温度状態へ移っても保持を解除しない |
| stopped、明示した新規開始が未承認 | stopped、none。正常試料だけでは再開しない |
| unknown、有効試料を再取得 | unknown、none。再同定・明示復旧の確認待ち |
| preparing、全開始条件成立 | start提案。受理結果の照合前はpreparingを保持 |
| warming、待機期限未満 | warming、none。明確な過熱は先行規則で停止 |
| warming、待機期限到達 | 全必須観測が正常ならrunning、それ以外は障害継続の計測へ |
| running、障害継続が停止判定時間未満 | running、none。healthy_since_msは無効化 |
| running/warming、障害継続が停止判定時間以上 | retry_wait、none。待機開始時刻を固定 |
| retry_wait、全必須観測が正常・保持なし・未確定要求なし・起動不要 | running、none。待機期限到達だけを理由にrestartしない |
| retry_wait、待機未完了 | retry_wait、none。観測ごとに待機開始を更新しない |
| retry_wait、待機完了・許可あり・残り回数あり | restart提案。受理結果まではretry_wait |
| retry_wait、回数上限 | stop_latched=true、stopped、stop提案。停止確認不能はunknownでも保持は維持 |
| thermal_stop、再開値以下の連続安定期間未満 | thermal_stop、none。温度上昇でcool_since_msを無効化 |
| thermal_stop、連続安定期間成立・許可あり・残回数あり・保持なし | start提案。受理結果まではthermal_stop |
| 上記以外で許可・必須値が不足 | 現状態を保持しnone。理由をchecksへ残す |

start/restartの受理とcontrol.startの実起動証拠を同じ要求IDで確認してからwarmingへ遷移する。通常の初回開始を除き、障害restartと温度停止後の自動startはいずれも最初の変更工程開始で1回を消費する候補。拒否・全工程未開始の確証では消費せず、開始後の結果不明は消費済み・unknownとする。受理済みで開始自体が不明ならpending_controlを保持して照合まで次の制御を始めない。同じreceiptは再消費しない。retry_wait中の自然解消も、保持なし・未確定要求なしで起動不要と確認できる場合だけrunningへ戻る。

| ケースID | 固定入力 | 期待結果 |
|---|---|---|
| `A07-39` | restartのstarted=true、control.stopだけverified、control.start=not_started | 一回の消費と停止済み位置を保持、warming不可。observe_controlは同じreceiptの工程証拠を必要とする |

正常継続・過熱/冷却・プロセス再起動・自動的なepoch変更では回数やstop_latchedを解除しない候補。リセットは運用上の明示選択・必要確認を経た新しい監視セッションだけとし、開発の仕様採用承認とは区別する。ControlReceiptの対象・epoch・要求ID・時刻・実状態の対応が不明ならunknownを維持し、receiptの自己申告だけで実開始を認定しない。

温度再開の連続安定期間はmining設定の必須integer秒、1〜600を採用候補とする。取得不能・時刻後退・epoch変更では連続性を破棄する。実機の停止温度・再開温度自体は能力表が正本で、共通値を設定しない。未完了の共通変更があればcontrolは保留し、提案状態を実制御成功として保存しない。

## 依存と互換性の詳細案

`P07-DEVICE`はEX-004/EX-005の個別版・機器・algorithmごとに、取得項目と単位・精度・欠損値・外部APIの契約を固定する候補。メーカー名・GPU名だけの互換認定は行わない。

| 境界 | 内製と差し替えの比較対象 |
|---|---|
| 能力検出adapter | 外部出力をMiningSampleへ変換する内製処理。単位・丸め・unknownの意味を固定 |
| 監視評価 | MonitorMemoryと試料列からEvaluationを返す純粋処理。同じ列で状態・提案・待機・回数を比較 |
| 制御adapter | 許可・共通排他・要求IDを再検査し、該当サービスだけを操作。実施receiptを評価へ戻す |
| エンジン/driver | 外部例外であってRust依存ではない。共有driver/カーネルの変更は運転制御へ含めない |
| 更新互換 | エンジン版・設定版・algorithm・driver・保全物の組を固定。旧エンジンだけでは設定復元可としない |

固定小数への変換候補は、元データの単位・精度が確認できる場合に1/1,000単位へ切り捨てる方式。符号・overflow・非数・単位不明はnullと理由へ対応付ける。温度の負値を一律0へ丸めず、センサーの許容範囲と照合する。丸め方式自体も`D07-01`の採用対象で、機器の安全温度をこれで決めない。

初期は同一パッケージ内のadapter/monitor/controlを分離し、将来のcrate差し替えで能力表・設定・観測形式が変わる場合は互換表と移行・復旧を先に確定する。外部ソフトのコード持込みを内製化と扱わない。

## 受入条件の充足照合案

DEVICEは機器/外部APIの能力、CONFIGは入力全集合と許可、RUNは前状態/試料/receiptの遷移、UPDATEは正常版・設定・driverの互換範囲へ対応する。状態表の優先規則と個別条件が矛盾する試料は正常化せずunknownへ返す。

| ケースID | 区分・固定入力 | 期待結果と確認 |
|---|---|---|
| `A07-14` | 異常。restart提案はあるが制御拒否のreceipt、次に同じ試料 | 回数・warmingへの遷移を確定しない。同じ試料の重複制御も行わない |
| `A07-15` | 境界。retry_waitの待機開始1000ms、待機5秒、入力5999/6000ms | 前者はnone、後者は許可/残回数を満たす場合だけrestart提案。観測ごとの待機リセットなし |
| `A07-16` | 異常。過熱かつprocess_alive=null、epoch変更後の古い正常時間、上限停止後の過熱→冷却 | 不明はunknown/none、連続時間無効。stop_latchedを保持し、冷却やepoch変更による自動再開なし |

再試行の実開始回数はlast_control_receiptを重複消費せず更新する。監視用状態の保存・障害後の再同定・controlの冪等性キーは`D07-03`に残り、評価器の表だけで実制御を完全仕様成立としない。実機の温度/電力・採掘対応は模擬試料の成功から認定しない。

## 入力と保存形式の詳細案

P07-CONFIGのMiningConfig候補は次の全集合とする。従来の項目候補を型へ結合するもので、機器別の安全値やminerの引数を創作しない。

| 型 | 全キー・型 |
|---|---|
| MiningConfig | schema_version: 1、mode: 既存3値、cpu・gpu: EngineConfigまたはnull、monitor: MonitorConfig、maintenance: MaintenanceWindow |
| EngineConfig | capability_ref: ContractRef、material_ref: MaterialRef、algorithm・pool・wallet: string、backup_pools: string[]、device_ids: string[]、threads: u32またはnull、power_milli_watts: u64またはnull、minimum_hashrate_milli: u64 |
| MonitorConfig | interval_seconds・warmup_seconds・failure_seconds・retry_wait_seconds・retry_limit: 先述の整数範囲、temperature_policies: device_id→TemperaturePolicyのobject、temperature_stable_seconds: 1〜600、temperature_unknown_action: hold/stop、allow_service_recovery: bool |
| MaintenanceWindow | timezone: UTCだけ、start_minute・end_minute: 0〜1439のinteger、allow_auto_update・allow_os_reboot: bool、update_targets: string[]。対象名はminer/configuration/os/kernel/driverから重複なし。実際の変更可否はP07-UPDATEと選択済み計画へ照合 |

cpuのみならgpu=null、gpuのみならcpu=null、combinedなら両方必須。各engineのdevice_idsは非空・一意、CPU/GPU間にも重複なし。温度ポリシーのキー集合は選択全機器と完全一致させる。CPUはthreads必須・power=null、GPUはthreads=null・device_ids非空、電力値は確認済みcapabilityの許容値に完全一致させる。最低hashrateは利用者の正整数入力とし、0や未取得を正常閾値にしない。pool/walletの許容文字と送信範囲は採用miner契約が確定するまで検証不能とする。

TemperaturePolicyの全キーはstop_milli_c・resume_milli_c（i64または両方null）。選択した全機器に一行ずつ要求し、各機器の確認済み値域内でresume < stopを検証する。温度二値のnullを監視不要と認定しない。temperature_unknown_actionの候補値は保護判断の審査入力であり、値があるだけで制御を許可しない。実際の未取得時動作はSECURITY_POLICYの判断待ちへ対応付ける。backup_poolsは0〜3件、主pool・相互の重複なし。自動切替の可否・順序はminer契約の採用待ちとする。時間帯は[start,end)とし、start=endは空時間帯、start>endは日跨ぎ。UTC以外と夏時間の推測は拒否する候補。既存運用許可の変更となる部分は採用審査待ちである。

MiningConfigの検証候補順序は形式/版→modeとcpu/gpuのnull条件→各EngineConfig→機器ID集合と能力参照→機器別TemperaturePolicy→monitor→maintenance→相互条件とする。数値文字列・真偽値のinteger化は行わない。power_milli_wattsはP07-DEVICEの範囲/刻みへ照合し、GPU全機器に適用できなければ拒否する。backup_poolsの掲載順は保持するが、切替機能の採用前は自動切替を発生させない。allow_service_recovery=falseなら許可に依存するstart/restartの自動復旧提案を実行せず、温度停止の安全条件を同じboolで解除しない。

temperature_unknown_action=stopは現候補のunknown/none規則を上書きする実装許可ではない。対応する安全ポリシー・機器別停止契約・状態表の採用が揃わない間は、当該設定を形式上識別できても運転計画をblockedにする候補とする。holdも安全性の確証ではなく、必要な保護が未成立なら運転開始を許可しない。A07-06/11/25で形式受理、運転許可、実制御を分けて照合し、実装者が二方式のどちらかを選ばない。

| ケースID | 固定入力 | 期待結果 |
|---|---|---|
| `A07-17` | combinedでgpu=null、minimum_hashrate=0、UTC時間帯23:00〜01:00の00:30/01:00 | 前二件は拒否。時間帯だけの評価では前者内・後者外。時間帯内でも実行許可を代行しない |

## 実行と復旧の確定候補

P07-RUNの制御記録候補はschema_version・operation_id・engine・epoch・sample_elapsed_ms・proposal・receipt・memory・config_sha256を全キーとし、エンジン別のmonitor-state.jsonへ同期保存する。receiptだけnull可。新旧記録を混合せず、保存不能時は新しいstart/restartを開始しない。過熱停止の必要性も担当安全条件へ照合し、保存失敗を正常運転へ変換しない。

| 評価対象 | 遷移の補完候補 |
|---|---|
| process_alive=false | preparingでは開始候補の前提、warmingでは待機期限を先に評価、runningでは障害継続を計測。状態表を飛び越して即restartへ移らない |
| connected=false / algorithm不一致 / 最低hashrate未満 | failure_sinceを最初の不成立試料へ固定し、連続不成立時間が閾値に達してからretry_wait |
| 全必須観測が正常 | failure_sinceを解除。retry_waitで保持なし・未確定要求なし・再起動不要を確認した場合だけrunningへ戻る。stopped/unknown/thermal_stopの解除条件を迂回しない |
| 試料欠損・epoch変更 | 連続時間を無効化しunknown/none。retry回数・stop_latchedは保持 |
| pending_controlあり | 同じ要求を新規発行しない。receiptと実サービスを読取り照合 |
| stopped/stop_latched | 新しい明示的運転セッションの選択・確認までstart/restartなし |

requestキーはengine/epoch/sample_elapsed_ms/action/config_sha256の組へ固定する候補。同じキーの再観測は既存receiptの照合だけで、開始回数を二度消費しない。監視プロセス再起動後は保存memoryと実サービスを照合し、新epochで連続時間を再計測する。保存memory欠損から回数0や許可済み起動を生成しない。

固定小数の切捨て候補は数学的floorとし、負温度も含めて明記する。例として-0.0005℃は-1 milli_c。元単位・範囲・精度を確認できない場合はnull。閾値・温度不明時制御の採用判断は機器安全審査に残し、この丸め候補を安全値の根拠にしない。

監視の秒→ms変換はchecked乗算、継続時間は同一epochのchecked減算とする候補。閾値ちょうどで待機完了/障害成立、停止温度ちょうどで停止、再開温度ちょうどは冷却区間へ算入する。演算失敗/時計後退ではunknown、連続時間を無効化し停止保持と回数を維持する。retry_limit=0でも明示確認済みの初回開始は別に判定するが、障害restart/温度停止後の自動startは生成しない。

| ケースID | 固定入力 | 期待結果 |
|---|---|---|
| `A07-18` | 同じrestart receiptを二回配送、監視再起動、低hashrate後に一度unknown | 消費は一回、保存回数を保持、不成立の連続時間は無効。自動リセット/再発行なし |

## 依存と受入の確定候補

P07-DEVICEの能力入力候補をDeviceCapabilityとし、全キーはdevice_id・kind・vendor・model・driver_material・kernel_release・miner_material・algorithms・sensor_contract・temperature_range・power_rangeとする。kindはcpu/gpu、各materialはMaterialRef、範囲は単位付き最小/最大/stepの整数objectまたは未確認時null。全キー必須。未確認を空の対応表へ変換しない。

能力範囲objectの候補キーはunit・min・max・stepだけ。温度unitはmilli_c、電力はmilli_watts、min<=maxかつstep>0、各値は共通整数範囲とする。設定値vはmin<=v<=maxかつ(v-min)がstepの整数倍のときだけ範囲検査を通す。差分/剰余のoverflowはDEVICE_RANGEで拒否する。sensor_contractは採取元・元単位・精度・変換・欠損の意味を固定し、同じdevice_idでもdriver/kernel/minerのいずれかが変われば旧能力行を再利用しない。algorithmsは確認済みの非空・重複なし集合で、空集合を未確認の代用品にしない。

sample adapterは選択device_idsのみに限定し、各機器hashrateのchecked合計とtemperatures_by_deviceを返す候補。temperature_milli_cは表示用の最大値で、制御判定は各機器の温度と対応するTemperaturePolicyを個別に照合する。一件でも停止条件成立なら当該engineの過熱とし、冷却条件は必要な全機器で成立させる。必須機器一件でも観測欠損なら該当集約値null、別機器の正常値で補完しない。外部APIの意味・単位・精度をsensor_contractへ固定してから変換し、模擬データの成功で実機能力を認定しない。

| 差し替え対象 | 同一fixtureで一致させる結果 |
|---|---|
| 機器adapter | 機器集合・単位・欠損/範囲外・丸め後の全MiningSample |
| monitor | MonitorMemory・試料列・ControlReceipt列からの各state/proposal/待機/消費数 |
| control | 同じ要求キー・許可・共通排他・実サービス状態に対する起動回数とreceipt |
| update | 固定miner/config/driverの互換とStepReceipt。正常版への復元後も元失敗を維持 |

初回は内製adapter/monitor/controlのmodule分離だけで、miner/driverのソースを取り込む内製crateは作らない。将来のcrate抽出は上記境界を一つずつ行い、能力/保存形式変更を伴えば採用仕様と移行計画を先に更新する。実採掘・温度/電力操作は今回未実施。

| ケースID | 固定入力 | 期待結果 |
|---|---|---|
| `A07-19` | GPU二台の片方だけ温度欠損、hashrate和overflow、同じreceipt列を新旧へ入力 | 欠損集約はnull、overflow拒否、純粋評価の全列一致。実機安全性は別の受入 |
| `A07-20` | 機器Aは78℃/停止75℃、機器Bは80℃/停止85℃、別途Aの試料欠損 | 前者はAで過熱成立。Bの最大値だけで正常にしない。欠損は0℃へ補完せず安全条件の判断待ち |
| `A07-21` | engine間で機器ID重複、温度ポリシー一件欠損、時間帯内だが更新未選択 | 前二件は設定拒否。時間帯成立だけでは更新0回、未定義の更新対象名も拒否 |
| `A07-22` | warming待機内のprocess=false、stoppedで正常試料、pending中の過熱 | 即restart/自動再開/二重stopを出さない。各事実と未確認要求を保持して状態表へ照合 |
| `A07-23` | minerのhash一致、再配布条件未確認、正常版と新設定が非互換 | 導入/配布または切戻しを停止。hash一致や模擬運転の成功で不足を補完しない |
| `A07-24` | 模擬power範囲min=100/max=200/step=10にv=150/155、driver版だけ変更 | 150だけ範囲内、155はDEVICE_RANGE。版変更後の旧能力証拠は再照合が必要 |
| `A07-25` | GPU二台でpower値が片方だけ範囲外、backup_poolsありだが方式未採用、復旧許可false | 設定拒否または切替/自動復旧0回。掲載順保持、温度保護を復旧許可で免除しない |
| `A07-26` | 全7状態へ同じ古いepoch、pendingと正常試料の同時成立、別requestのreceipt | 前者unknown/none、pendingは新規制御なし、別receiptはMONITOR_RECEIPT。回数/保持の勝手な解除なし |
| `A07-27` | 旧minerはあるが旧設定なし、CPU更新計画へGPU変更、運転retry残りあり | 復元/対象照合で停止。retry残量を更新再投入へ転用せず、元更新失敗を保持 |
| `A07-28` | 同じ提案だがrequestの設定hash相違、accepted=falseかつstarted=true | MONITOR_RECEIPTで状態不変。提案だけの照合で別要求のreceiptを消費しない |
| `A07-29` | retry_limit=0、初回明示開始/障害再開、ms差分overflow/冷却閾値ちょうど | 初回と自動復旧を分離。自動開始0回、演算失敗はunknown、境界は指定どおり判定 |
| `A07-30` | noneのreceipt、要求試料より前のreceipt時刻、stop開始だけ確認、拒否済みIDの再配送 | 不正receiptを拒否し、停止完了/warmingを捏造しない。拒否でも処理済みIDを保持し二重発行・回数消費なし |
| `A07-31` | observed_atだけ更新、bootだけ変更、新epochへ回数0/保持falseを補完 | 時刻だけは同一性を保ち、boot変化は再同定。新世代でも保存回数/保持を消さず、取得不能で制御しない |
| `A07-32` | retry_waitで自然正常化、別入力でrunning/process=true/hashrate=null、TargetEvidenceの証拠数を型と照合 | 正常化は起動不要を確認してrunning/none。稼働中の不明はunknown/none、六証拠と時刻を区別し再試行・保持解除なし |
| `A07-33` | preparingでprocess=false/hashrate=null、warming期限内は接続未準備、runningでprocess=true/hashrate=null | 開始前に未生成の運転値を要求せず、待機内はwarming/none、稼働中の必須不明はunknown/none。温度保護を待機で免除しない |

## 未決事項と成立判定

### 監視記録の保存と再同定

```rust
pub enum ControlLedgerPhase { Prepared, Started, Finished, Unknown }
pub struct ControlLedger {
    schema_version: u32, request: ControlRequest, operation_id: OperationId,
    phase: ControlLedgerPhase, receipt: Option<ControlReceipt>,
    memory_before: MonitorMemory, memory_after: Option<MonitorMemory>,
    verification_refs: Vec<String>,
}
pub struct MonitorRecord {
    schema_version: u32, operation_id: OperationId, engine: Engine, epoch: String,
    sample_elapsed_ms: u64, proposal: ControlProposal, receipt: Option<ControlReceipt>,
    memory: MonitorMemory, config_sha256: String,
}
```

保存器は上記型を本書の同名保存形式へ一対一に検証する候補。MonitorRecord.memoryとreceipt/requestのengine/epoch/設定/親操作を照合し、同じIDでも別試料のreceiptを受理しない。Ledger.phase=Finishedでもmemory_after欠損なら確定不可。Preparedのreceipt=Noneは未開始の証明でなく、実処理/共通operationとの照合が必要。A07-18/34/41へ型の未知キー/片側保存失敗/再起動後の同ID再消費を含める。

再試行枠の判定候補を、新規要求の予約前と同じ要求の実行前で分ける。新規自動start/restartはattempts_used+既存予約数<retry_limitを要求する。予約した同一requestの実行は、自分の一予約を含めた和<=retry_limit、pending/reservedの同一ID、台帳preparedとの一致を検査する。予約直後に同じstrict比較で自分の最後の枠を拒否しない。別要求へ予約を貸し替えない。

ControlLedger.preparedとmemoryのpending_control/reserved_retryを同期して照合してから制御を開始する。片側同期失敗は開始0件、途中保存不明はunknownとして既存予約を保全する。実開始の確証が得られたら、同じrequestについて予約解除とattempts_used+1を一つの次memoryへ計算し、台帳と照合して確定する。整数overflow/矛盾はMONITOR_RECEIPT。未開始の確証以外で予約を解放せず、同じreceiptを二度加算しない。

| ケースID | 固定入力 | 期待結果 |
|---|---|---|
| `A07-41` | retry_limit=1、使用0から要求rを予約、同じr実行、別s要求、rの二重receipt | rだけ枠内で一回消費、s拒否、二重加算0回。予約後の自己拒否・未確認予約の解放・上限超過なし |

監視状態の候補保存先を/var/lib/serverseed/mining/cpu/monitor-state.jsonとgpu/同名へ固定する。要求別の制御記録は同じengine配下control/OperationId.json、request_idはP02の新規OperationId表記で生成する。生成は純粋evaluateの外で一回だけ行い、engine/epoch/sample時刻/action/config hash/対象の組へ結合する。同じ組の既存要求があれば新IDを発行せず読取り照合する。対象/engine/IDを外部pathへ展開しない。

ControlLedgerの全キー候補はschema_version: 1、request: ControlRequest、operation_id: OperationId、phase: prepared/started/finished/unknown、receipt: ControlReceiptまたはnull、memory_before: MonitorMemory、memory_after: MonitorMemoryまたはnull、verification_refs: string[]。preparedを同期してから制御を開始し、実開始の確証/未確認を保存し、実サービスとreceiptを照合してmemory_afterを確定する。先にMonitorMemoryだけ更新して要求証拠を失う順序は不可。共通operation/historyの確定は親が行う。

reserved_retryは開始不明の自動start/restartについて枠を予約する要求IDまたはnull。新規予約前と同じ要求の実行前の比較は本節冒頭の二条件だけを使用し、予約後にもstrict比較を繰り返して最後の枠を拒否しない。開始確証で予約を解除してattempts_usedへ一回加算する。未開始の確証なら予約だけ解除、未確認なら予約を維持する。初回の明示開始だけは同じ保存セッションの新規開始確認に対応付け、自動復旧へ読み替えない。停止要求は再試行枠を消費しないが、同一要求の重複発行は禁止する。

制御台帳の上限候補は各engineで1,000要求かつ64 MiB、ControlLedger一件64 KiB、MonitorMemory一件64 KiBとする。新規/一時保存分も加算し、どちらかの上限へ達する新規制御を停止する。未完了/unknown台帳は削除せず、完了台帳の退避も別の明示保守計画で参照閉包を保持する。容量不足を理由にstop_latched・attempts_used・reserved_retryを初期化しない。安全上必要な停止を発行できない場合は、その未実行と残存サービスを報告し、停止済みと偽らない。

64 KiBは各完成JSONのescape/キー/区切り/LF込み実長へ適用する候補。ControlLedgerはrequest/receipt/memory_before/memory_afterの合成全体で判定し、各memoryが単独上限内という理由だけで保存可能にしない。監視セッション開始前に、固定機器/Check/参照集合から成功・失敗・unknown各枝の最大合成量とP02の深さ/ノードを照合し、不明/超過なら制御を開始しない。実出力も再検査し、超過時の切詰め・回数/保持の消去は禁止する。A07-18/34/41へ個別memoryは上限内だが合成台帳超過、失敗枝だけ超過を加える。

再起動後は全未完了ControlLedger、保存設定hash、対象/boot、実unitの状態を読み、pending/予約/回数/停止保持を復元する。経過時刻の連続性は引き継がず新epochとし、unknownからの明示照合へ戻す。記録欠損/別設定/別対象/不整合を回数0にしない。消費済みIDは同engineの確定履歴と照合し、last_control_receipt一つだけで過去の全重複を判定しない。保持上限到達では自動制御の新規開始を停止し、ログ整理で制御履歴を自動削除しない。

controlのサービス操作はP04-APPLYのunitテンプレートを使い、cpu/gpuの固定native unitだけへ解決する。監視評価は保存も実行もしない。temperature_unknown_actionの選定・センサーの正確な外部API・実機停止の許可は担当安全審査へ残し、記録設計だけで制御可能としない。

| ケースID | 固定入力 | 期待結果 |
|---|---|---|
| `A07-34` | prepared同期失敗、開始不明の予約一件、過去の古いreceipt再配送、記録欠損後の再起動 | 実行0回/枠保持/再消費0回/unknownを照合。回数0への復元・同要求再実行・保存失敗の成功化なし |

### 機器検出と採用能力行の区分

検出と使用可能性の型候補を次へ閉じる。取得不能を表すEvidenceFieldはP02の型だけを使い、DeviceCapabilityは全必要項目の検証後だけ生成する。

```rust
pub struct DeviceIdentity {
    device_id: EvidenceField, target_machine_id: EvidenceField, boot_id: EvidenceField,
    kind: EvidenceField, topology_ref: EvidenceField, vendor_id: EvidenceField,
    device_code: EvidenceField, subsystem_ref: EvidenceField, driver_ref: EvidenceField,
    observed_at: UtcMillis,
}
pub struct CapabilityRange { unit: String, min: i64, max: i64, step: i64 }
pub struct DeviceCapability {
    device_id: String, kind: Engine, vendor: String, model: String,
    driver_material: MaterialRef, kernel_release: String, miner_material: MaterialRef,
    algorithms: Vec<String>, sensor_contract: ContractRef,
    temperature_range: Option<CapabilityRange>, power_range: Option<CapabilityRange>,
}
pub struct CapabilityEvidence {
    target: TargetEvidence, devices: Vec<DeviceIdentity>, capabilities: Vec<DeviceCapability>,
    verification_ref: String,
}
```

CapabilityEvidenceは収集器が同じ対象/起動世代で検証した能力集合を保持する候補。全キーはtarget/devices/capabilities/verification_refだけ。devicesとcapabilitiesはdevice_idで一対一に解決し、MachineId/boot/topologyと選択機器・材料/algorithm/sensorの照合を終えてから構築する。能力行を別対象へ単独で貸与し、device_idの文字列だけで利用可としない。mining_planはevidence.targetと今回targetの固定証拠を照合し、時刻だけの更新と起動世代の変化を区別する。証拠参照だけ存在し元の観測を取得できない場合はDEVICE_EVIDENCEで保留する。型の存在は実機検証済みの意味ではない。

| ケースID | 固定入力 | 期待結果 |
|---|---|---|
| `A07-42` | 別対象で同じdevice_id、devices一件欠損、旧bootの能力行、同一bootで時刻だけ更新 | 前三件はDEVICE_EVIDENCEで計画不可。最後は固定証拠の照合を維持し、時刻だけの別機器判定・別対象の能力流用なし |

driver_materialはCPUでもOS側の対応実行基盤を台帳参照で特定し、GPU専用driverを架空登録しない。温度は符号付き、電力は0以上で共通外部整数範囲内、step>0を要求する。device_idをObservedから抽出する前に、同じMachineId/boot/topologyの照合が必要。温度/電力のNoneは制御非対応または未確認の区別を担当検査へ残し、確認不能を非対応の確証として処理しない。表示の検出一覧をCapabilityの集合へ無検査変換しない。

| ケースID | 固定入力 | 期待結果 |
|---|---|---|
| `A07-40` | device_id未取得、電力minが負、CPUに架空GPU driver、同位置でboot変更 | 能力構築/値域/資材照合を拒否し再同定。検出だけの対応認定・型宣言のないAPI入力・0埋めなし |

機器の列挙候補では、CPU packageは対象のtopology識別、GPUはPCIのdomain:bus:device.functionとvendor/device/subsystem/driverへ対応付ける。PCIアドレスは当該起動世代内の位置で、恒久的な個体識別子ではない。DeviceIdentity候補はdevice_id・target_machine_id・boot_id・kind・topology_ref・vendor_id・device_code・subsystem_ref・driver_ref・observed_atの全キーとし、未取得は理由付きのEvidenceFieldで表す。候補IDだけで別boot/交換機器へ設定を再適用しない。

温度の標準観測候補は[Linux hwmon](https://www.kernel.org/doc/Documentation/hwmon/sysfs-interface)の入力を、選定済みsensor_contractが示す機器/labelへ対応付けて読む方式。最大4 KiB/一入力、符号付き十進整数と末尾LF一つだけ、許容範囲を照合し、元単位がmilli_cの場合は値をそのまま保持する。hwmon番号/列挙順だけでCPU/GPUへの接続を推測しない。driver固有の温度単位/offset/欠損/更新間隔が不明ならunknown。標準化された名前があっても実際のセンサー配線/用途が一致した証拠を必要とする。

採用能力行の一意キーは機器の能力識別/driver MaterialRef/kernel/miner MaterialRef/algorithm/sensor_contractの組。公式資料の記述は対応候補、選定/検証資料のある組だけを対応行とする。機種名の事前指定を全機器の列挙条件にせず、未登録機器も「検出したが使用未確認」と表示する。実機の安全温度/電力/採掘APIの全契約が未取得なら制御は保留する。CPUのみの運転へGPUを無断代替せず、利用者のmodeを保持する。

[SRBMiner公式資料](https://github.com/doktor83/SRBMiner-Multi)にある対応/手数料/許諾の表示は、採用する正確なreleaseと照合する必要がある。公式配布物が公開されていることを、ISOへの同梱・再配布や内部GPU binaryの抽出の許可としない。個別のライセンス/配布判断は例外台帳へ一意に記録し、本書へ条件を複製しない。現時点の使用可能なminer/driver行・実機安全値は未確定のまま。

| ケースID | 固定入力 | 期待結果 |
|---|---|---|
| `A07-35` | 同じPCI位置に別device、hwmon番号だけ一致、欠損温度、公式READMEだけを根拠に同梱 | 再同定/unknown/配布審査待ち。位置や検出成功の対応認定・共通安全値・未確認同梱なし |

| 判断ID | 残る判断・根拠 | 影響 |
|---|---|---|
| `D07-01` | [機器検出](#機器検出と採用能力行の区分)に案あり。実機ごとのsensor/単位/精度/安全値と外部APIの根拠が未確定 | 機器対応・温度制御 |
| `D07-02` | 機器別設定とUTC時間帯に案あり。運転許可・全値域/能力参照・旧設定移行の採用判断が残る | 採掘設定・サービス生成 |
| `D07-03` | 全状態表と[永続監視記録](#監視記録の保存と再同定)に予約/消費/再同定案あり。固定制御adapterと再起動/保存失敗の成立根拠・採用判断が残る | 自動制御・再試行 |
| `D07-04` | miner/設定とOS/driverの別stepに案あり。実機・採用版・正常版/再配布条件と試験境界は未確定。検出成功を対応認定にしない | 導入・配布・採掘受入 |

現段階ではfixtureによる受入の設計のみで実採掘・実機操作は行っていない。各未達と共通契約・非機能要求を解消するまで完全仕様は成立しない。
