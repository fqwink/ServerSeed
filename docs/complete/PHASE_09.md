# Phase 9 完全仕様案: 既存OSへのSSH導入・保守

[文書一覧](../DOCUMENT_INDEX.md) | [担当工程](../ROADMAP.md#phase-9-マスターから既存osへのssh導入保守)

採用未承認の`B14`であり、[案の適用範囲](PHASE_01.md#案の適用範囲)を適用する。担当は`REQ-001`・`REQ-002`・`REQ-003`・`REQ-004`・`REQ-005`・`REQ-006`・`REQ-007`・`REQ-008`・`REQ-009`・`REQ-011`・`REQ-012`。既存の[マスターの処理](../master/specification.md#自動処理の流れ)と[Cloud導入](../BASIC_SPEC.md#sshによるcloud環境の導入と保守)を具体化する候補で、OS未導入の導入機構はPhase 10へ分離する。

## 契約と共通参照

| 契約ID | 担当 |
|---|---|
| `P09-SSH` | 接続・認証・搬入・実行経路 |
| `P09-REMOTE` | 計画・対象・操作の結合と配送 |
| `P09-RECONNECT` | 切断・再起動・中断後の照合と再接続 |

使用する契約（全フェーズB14）は、`P02-MODEL`・[識別](PHASE_02.md#p02-model)、`P02-CLI`・[出力](PHASE_02.md#p02-cli)、`P02-STORE`・[排他](PHASE_02.md#p02-store)、`P02-EXEC`・[外部実行](PHASE_02.md#p02-exec)、`P04-SELF`・[配置](PHASE_04.md#p04-self)、`P04-UPDATE`・[更新](PHASE_04.md#p04-update)、`P04-BACKUP`・[保全](PHASE_04.md#p04-backup)、`P08-MASTER`・[対象台帳](PHASE_08.md#p08-master)。信頼・認証・秘密情報の許可条件は[Cloudの管理経路](../SECURITY_POLICY.md#cloudの管理経路)へ参照し、本書で独立したセキュリティ仕様を設けない。

## P09-SSH

採用候補はOS側のSSH実行物を引数配列で呼び出す方式。SSHプロトコル・暗号をRustで新規実装する案ではない。具体的な実行物・版・引数・供給元は`EX-001`への個別採用情報とD09-01で確定する必要がある。

| 入力・制限案 | 型・検証・動作 |
|---|---|
| 接続先 | P08-MASTERの対象IDを必須にし、host・port・管理ユーザー・信頼情報の参照へ解決。自由なSSHオプションの連結を受けない |
| 初回・変更時の信頼 | 対象とホスト鍵の独立確認を所定のポリシーへ照合。単なる接続成功・自動受理で置き換えない |
| 認証情報 | 安全な参照と実行環境での受渡しを審査する。台帳・引数・ログへ秘密値を置く方式を採用済みとしない |
| 権限 | 非root接続から対象側変更の権限を確立。昇格不能・対象側追加入力が必要なら開始せず、マスターで必要な選択へ戻す |
| 時間 | 接続1回30秒、搬入300秒、能力検査30秒を候補。変更の再送は0回 |
| 転送 | 固定した本体一式を対象用の作業領域へ搬入し、転送後に照合。転送成功だけで導入成功にしない |

ローカルで引数配列を使っても、SSHの遠隔コマンド文字列が安全になるとは限らない。採用案では遠隔で呼ぶ入口を固定し、構造化リクエストを標準入力から渡す方式を第一候補とする。対象ユーザーのshell解釈、固定入口の初回準備・検証、権限昇格と入力配送の結合はD09-01の成立性確認が必要であり、任意のパス・引数を空白連結する代案にしない。

初回搬入の審査候補BootstrapTransferはtarget_id・binary_material・manifest_material・signature_material・build_metadata_material・keyring_ref・staging_ref・entry_ref・privilege_refを全キーとする。四資材は[P05の独自成果物参照](PHASE_05.md#型とapiの詳細案)のReleaseArtifactRef、残るrefは事前審査した固定入口/保存先/昇格契約のIDであり任意path/オプションではない。SSH・検証実行物のMaterialRefとは区別する。第一候補は検証済み同一本体の搬入と薄い固定wrapper、第二候補は既存の監督入口を利用する方式。どちらも転送→対象側の真正性/実hash検査→root所有領域への確定→同一本体worker開始の全外部契約が必要。独立agent新設は候補に含めない。昇格不能や追加手入力が必要なら配送前にマスターへ不足を返す。

搬入前の候補検証は対象ID→固定リリース版と四資材のmanifest対応→独立した信頼参照→保存先/固定入口/昇格契約→転送予算とする。同じ版名でもmanifest hashが違う組合せをSSH_BOOTSTRAPで拒否する。署名付きmanifestが指すbuild metadataの実hash/CPU/loader/ABIを対象環境へ照合してから候補を起動する。対象側で再検証・root所有領域の確定が終わるまで候補を実行しない。Connect候補の入力はRegistryEntryとConnectionLimitsの型付き値、ConnectionLimitsの項目はconnect_timeout_ms・transfer_timeout_ms・inspection_timeout_msで本書の各上限に従う。ユーザーのSSH設定断片を追加の引数列へ変換しない。

## P09-REMOTE

採用案の要求は次の項目を一組とし、対象側の共通操作へ渡す。

| 項目案 | 意味と照合 |
|---|---|
| `request_id` | 一意な配送識別子。同じ要求の照合に使用し、操作の未実行を証明するIDではない |
| 対象の証拠 | 台帳対象ID・対象OSと実行環境の識別・観測前提。対象側で再取得して照合 |
| 計画の証拠 | 固定plan_id、選択内容、成果物の識別、変更資源・保全・復旧範囲 |
| 対象側操作 | 対象のoperation_id・排他所有者・進行位置とマスターの要求を結合 |
| 応答 | 対象側の実適用位置、検証、確定した結果、未確認範囲。stdout未着を未実行としない |

選択と確認はマスター側だけで行い、対象VPSで追加の手動開始を要求しない。変更前に対象側の記録・前提・排他を照合し、同じ要求が既に記録されていれば実状態を取得して既存の位置へ対応付ける。無条件な二度目の適用はしない。

要求の受理・開始・結果確定の保存形式、受理前後の切断、二重マスターからの要求の防止、初回導入前の記録先はD09-02で決定する。単なるoperation.jsonの存在や送信側の送信記録だけを配送の確定と認めない。

## P09-RECONNECT

| 事象 | 採用案の照合と続行条件 |
|---|---|
| 接続前の失敗 | 対象変更を送っていないことを確認し、接続失敗を表示 |
| 要求送信後の切断 | マスターを未確認状態にし、対象側操作・ロック・実配置を読み取って照合 |
| 対象再起動 | 計画に含まれる再起動かを照合し、同じ対象へ接続して実OS・処理位置を確認 |
| マスター停止 | マスター記録と対象側記録の双方を照合。片方が欠ける間は再投入しない |
| ホスト鍵・接続先変更 | 対象同一性と予定変更の確認へ戻る。自動置換・検証無効化なし |

再接続の候補は待機上限600秒、1回の接続上限30秒、試行間隔5秒、最大12回の小さい方を上限とする。これらは観測の再試行だけであり、変更の再送回数ではない。期限超過・対象不明・記録不一致は未確認のまま停止し、復旧計画の提示へ接続する。

取消要求はマスターの受付、対象への伝達、対象の保全・停止、終了照合を分ける。SSHプロセスを終了させたことを対象の変更中断完了とみなさない。遠隔で続いている変更を再投入しない条件と、親の停止後も排他を維持する契約はD09-02で確定する。

再接続の合否候補はtrust.bound・identity.matches・records.consistentの各検査を独立に照合し、全必須passかつ同じ要求hash/操作結果の確定時だけ成功とする。最初の観測試行も回数へ算入し、各試行を開始できるのは30秒の接続予算と親残期限の小さい方が正の場合。終了後5秒の待機を挟むが、上限到達後の待機・次試行は行わない。読取り成功でもrecord_consistent=nullならresubmit_allowed=false。推奨は固定した切断位置列での再現試験で、実VPSの状態を未確認のまま使用しない。

再接続予算の候補起点は再接続要求を受け付けた単調時計の時点とし、600秒に接続・Query・照合・待機を全て算入する。一試行は一接続とそのQuery/照合を含み、接続確立だけで回数を消費しない方式にはしない。開始済み試行のtimeout/切断も一回、残期限は親期限と本再接続期限の早い方で制限する。待機は前試行の終了から5秒、待機中に期限へ到達した場合は即停止し次を開始しない。必要な停止/結果処理の予約を引いた残量が正でない場合も開始不可。最終試行の接続成功後に記録がunknownなら結果は未確認のまま、回数をリセットしない。

## 内製単位・公開API案

配置候補は`src/profiles/master/ssh.rs`・`remote.rs`・`reconcile.rs`と`src/cloud/`の対象側入口。通信の制御と対応照合を内製し、共通の適用エンジンを複製しない。難易度は遠隔の状態照合・安全な配送が高。

| 宣言案 | 入出力と責務 |
|---|---|
| `connect(target: &RegistryEntry, limits: &ConnectionLimits) -> Result<RemoteSession, ContractError>` | セッションは接続・対象識別を所有。台帳更新・対象変更なし |
| 変更配送 | [型付きsend_change](#型とapiの詳細案)のみ。固定要求と部分配送位置を保持、重複適用なし |
| 読取り照合 | [型付きquery_change](#型とapiの詳細案)のみ。照合の許可を修復・変更へ拡張しない |

型はP02-MODEL、RemoteSessionの宣言は[型とAPI](#型とapiの詳細案)へ集約する。固定入口と実際の接続/監督寿命・排他の成立契約、各payloadの採用はD09-01〜02に残す。新設案で旧実装は存在しない。将来の差し替えには同じ切断位置・重複要求・鍵変更fixtureを適用し、配送記録・対象操作IDと保存形式の互換性・切戻し限界を確認する。

## データ契約の具体化案

`P09-REMOTE`の配送と照合用データを具体化する候補。共通型は[P02-MODEL](PHASE_02.md#データ契約の具体化案)、対象台帳は[P08-MASTER](PHASE_08.md#データ契約の具体化案)へ参照する。表は既存operation.jsonへのフィールド追加ではなく、制御側・対象側が照合する別の候補データである。

| 型 | 全キー・値域の候補 |
|---|---|
| RemoteRequest | schema_version: 1、request_id: OperationId型、master_machine_id: MachineId、target_id: 台帳ID、target: TargetEvidence、plan_id: PlanId、plan_ref: PlanReference、action: install_body・setup・update・backup・recover・install_os、contracts: ContractRef[]。install_osの入力・実行条件はPhase 10のみで定義 |
| DeliveryRecord | request_id: 上記ID、target_id: 台帳ID、plan_id: PlanId、request_digest: string、operation_id: OperationIdまたはnull、status: accepted・started・result_ready・unknown、last_confirmed_status: accepted・started・result_readyまたはnull、receipt: OperationReceiptまたはnull |
| ReconcileResult | request_id: 上記ID、target_id: 台帳ID、delivery: DeliveryRecordまたはnull、identity_match: booleanまたはnull、lock_observed: booleanまたはnull、checks: Check[]、resubmit_allowed: boolean |
| CancelResult | request_id: OperationId、target_id: TargetId、plan_id: PlanId、delivered・acknowledged・stopped: booleanまたはnull、receipt: OperationReceiptまたはnull、checks: Check[]。全キー必須 |

request_idは操作IDと同じ表記・生成器を再利用するが、別に生成する配送の識別子。対象operation_idと同一であるとは要求しない。request_digestの原入力と算定方式は`P03-PLAN`の固定形式と接続して`D09-02`で確定する。同じrequest_idで対象・action・計画・固定資材が異なる要求は衝突として拒否し、前の要求を置換しない。

対象側は、排他・前提を確認してacceptedを永続化し、operationのrunningを確定してからstartedへ進める候補。対象変更は両記録の確認後だけ行う。result_readyには対象の必須記録の確定と実状態の検証が必要で、返信の送信成功とは別に保持する。acceptedだけの記録から「まだ何も適用されていない」と断定せず、対応する操作と実状態を照合する。

DeliveryRecordの候補不変条件は、通常statusならlast_confirmed_statusが同値、unknownなら最後に確認した通常段階、受理の確証がなければnull。started以降はoperation_id必須で、一度確認したIDを後からnull/別IDへ変えない。result_readyは同じ対象/plan/operationのreceiptとRecordReceipt.completeを要求するが、確定した元操作の失敗も結果として回収できる。配送結果が準備済みであることを製品操作の成功へ変換しない。unknownでは既知receipt/到達位置を保持し、不明をacceptedへ戻して二度目の実行を開始しない。

ReconcileResultのdelivery=nullは記録を確認できない状態を含むため、それだけでresubmit_allowed=trueにしない。再投入の可否は、未受理の確証または復旧側で照合した許容範囲、現在の排他・対象・計画の一致を必要とする。無応答・timeout・EOFではfalseを維持する。取消の受付・配送・対象の停止を別々に表し、stoppedがtrueでも実資源の復元成功までは示さない。

リクエストは検証済みの1文書を標準入力で渡し、入力上限・UTF-8・複数文書の拒否は共通解析へ参照する。応答もP02-CLIの構造化結果で一度だけ返す案。バナー・進捗・sudoのプロンプトがstdoutへ混ざる経路は成功として解析しない。配送・記録が別々に確定した場合の全中断位置、固定入口の初回準備、親停止後の処理寿命は`D09-01`・`D09-02`の未達に残る。

## 接続と責務の照合

本体変更を含むresult_readyの追加照合先は[P04の終了処理](PHASE_04.md#本体変更の終了処理)とする候補。coreのRecordReceipt.completeに加え、SelfFinalization.Completeだけ成功候補、同期済みFailedは確定失敗として回収可能、Prepared/Cleaning/Unknownは結果未確定とする。Queryは両者を区別したreceipt/Checkを返すだけで、清掃やcore記録の書換えを行わない。清掃復旧は新しい明示recover計画へ戻す。

旧要求の後に清掃復旧の連鎖が完結した場合も、Queryは旧要求の同期済みreceipt/元失敗と、[清掃停止の解消根拠](PHASE_04.md#清掃復旧の参照連鎖)をCheckで区別する候補。解消Checkのmessageへ新しい自由情報を連結せず、固定分類と検証資料へ対応付ける。旧delivery/receiptを新leafの成功receiptで置換しない。旧Unknownを成功またはresult_readyへ自動書換えせず、確定結果が存在しない事実は維持する。新recoverは新request/plan/operationを必要とし、旧Submitを再送しない。

| ケースID | 固定入力 | 期待結果 |
|---|---|---|
| `A09-42` | Aの同期済み清掃失敗後にB/Cで連鎖完結、別入力はAの終了がUnknownのまま | QueryはAの元結果と解消Checkを分離、旧記録の保存0件。Unknownからの成功/結果確定の捏造、Cのreceipt転用、旧Submit再送なし |

配送の再照合は[対象側記録](PHASE_02.md#p02-store)と[適用結果](PHASE_04.md#データ契約の具体化案)の読み取りで行い、同じRemoteRequestを再送して実行状況を問い合わせる方式にしない。問い合わせ・取消と変更要求の固定入口・型の分離が未確定である間は、任意の遠隔shell文字列を代用にしない。

対象側の排他取得は受理と実行の境界で検証し、accepted・running・startedの部分確定では復旧照合を必要とする。マスター側が未受理と判定できても、変更の自動再送は0回を維持する。再投入を許容する根拠がある場合も、対象・前提・選択・計画を再確認した明示操作として扱う候補であり、resubmit_allowedの値だけで送信しない。

既存OSへの導入と[P10-OS](PHASE_10.md#p10-os)のOS導入は別のactionである。install_bodyでOS未導入を検出しても、ディスク消去・OS展開へ自動変更しない。OS導入後の鍵変更は「予定されていた」ことと実際の信頼確認を別々に満たし、ReconcileResult.identity_match=trueだけで鍵検証を省略しない。

再接続は観測試行の終了から次の開始まで所定間隔を置く候補。親の待機期限と試行上限のいずれかへ先に到達した時点で終了し、次の接続を始めない。予定再起動による切断も、未確認の副作用や記録欠損を成功へ変える理由にしない。

## 受入ケース案

| ケースID | 区分・前提 | 期待結果と検証 |
|---|---|---|
| `A09-01` | 正常。既存の対応OSへ初回導入・構成適用 | マスターでの選択後に自動化。対象での追加手動開始なし |
| `A09-02` | 異常。未知鍵・鍵変更・認証不成立 | 変更配送前に停止。検証を無効化しない |
| `A09-03` | 異常。権限不足、コマンド不足、対象OS不一致 | 実行環境と対象を分けて不足表示。OS再導入へ自動分岐しない |
| `A09-04` | 異常。受理前・開始後・結果確定後の切断 | 各位置の実状態を照合し、変更の重複適用0回 |
| `A09-05` | 異常。同じ要求IDの重複、マスター停止 | 記録・排他の対応を検査。不明を未実行としない |
| `A09-06` | 正常。予定された再起動・再接続 | 同一対象と適用結果を確認してから成功を確定 |
| `A09-07` | 境界。接続・待機上限、最大試行回数 | 上限で停止。再接続を理由に変更を再送しない |
| `A09-08` | 異常。ホスト入力へshell構文、取消直後の応答欠損 | データとして検証。任意実行・中断完了の偽装なし |

## 追加の受入ケース案

`P09-REMOTE`・`P09-RECONNECT`の配送位置を固定した設計。模擬の対象側状態と両側の保存記録を個別に注入し、実VPSへの接続・変更はしない。

| ケースID | 区分・前提と入力 | 期待結果・照合方法 |
|---|---|---|
| `A09-09` | 異常。同じrequest_idに別plan_id・対象・actionを各一つ変更 | 衝突として拒否。既存要求の置換・二度目の適用0回 |
| `A09-10` | 異常。acceptedだけ確定、対象runningとの対応不明で切断 | resubmit_allowed=false。未開始・未受理と推測せず照合待ち |
| `A09-11` | 正常。対象の必須結果は確定、返信だけ欠損 | 読取り照合で同じ結果を回収。RemoteRequest再送・適用0回 |
| `A09-12` | 異常。取消は配送済み、対象ackまたは停止がnull | 中断完了と報告しない。SSHクライアント終了を対象停止へ転用しない |
| `A09-13` | 境界。12回目の観測失敗、待機期限到達、stdoutにプロンプト混入 | 次の接続なし、変更再送なし。不正応答を成功JSONとして受理しない |

## 型とAPIの詳細案

`P09-REMOTE`の変更要求・照合・取消を別の型として配送する候補。install_osは配送層が識別するだけで、Phase 9の既存OS向け操作から自動選択しない。

```rust
pub enum RemoteAction { InstallBody, Setup, Update, Backup, Recover, InstallOs }
pub struct ConnectionLimits {
    connect_timeout_ms: u32, transfer_timeout_ms: u32, inspection_timeout_ms: u32,
}
pub struct BootstrapTransfer {
    target_id: TargetId, binary_material: ReleaseArtifactRef,
    manifest_material: ReleaseArtifactRef, signature_material: ReleaseArtifactRef,
    build_metadata_material: ReleaseArtifactRef, keyring_ref: String,
    staging_ref: String, entry_ref: String, privilege_ref: String,
}
pub struct MasterRemoteRecord {
    schema_version: u32, request_sha256: String, request: RemoteRequest, sent_bytes: u64,
    last_delivery: Option<DeliveryRecord>, observation_refs: Vec<String>,
}
pub struct RemoteSession {
    target: RegistryEntry, observed: TargetEvidence, master_machine_id: MachineId,
    trust_ref: String, authentication_ref: String, connection_ref: String,
    deadline: std::time::Instant,
}
pub struct QueryRequest { request_id: OperationId, target_id: TargetId }
pub struct CancelRequest { request_id: OperationId, target_id: TargetId, plan_id: PlanId }
pub enum RemoteMessage { Submit(RemoteRequest), Query(QueryRequest), Cancel(CancelRequest), QueryIndex(IndexRequest) }
pub enum DeliveryStage { Accepted, Started, ResultReady, Unknown }
pub struct PlanReference {
    plan: Plan, plan_sha256: String, selection: Option<Selection>,
    install_payload: Option<InstallPayload>, backup_set: Option<BackupSet>,
    confirmation: Confirmation,
}
pub struct RemoteRequest {
    schema_version: u32, request_id: OperationId, master_machine_id: MachineId,
    target_id: TargetId, target: TargetEvidence, plan_id: PlanId,
    plan_ref: PlanReference, action: RemoteAction, contracts: Vec<ContractRef>,
}
pub struct DeliveryRecord {
    request_id: OperationId, target_id: TargetId, plan_id: PlanId, request_digest: String,
    operation_id: Option<OperationId>, status: DeliveryStage,
    last_confirmed_status: Option<DeliveryStage>, receipt: Option<OperationReceipt>,
}
pub struct ReconcileResult {
    request_id: OperationId, target_id: TargetId, delivery: Option<DeliveryRecord>,
    identity_match: Option<bool>, lock_observed: Option<bool>, checks: Vec<Check>,
    resubmit_allowed: bool, request_sha256: Option<String>, record_consistent: Option<bool>,
}
pub struct CancelResult {
    request_id: OperationId, target_id: TargetId, plan_id: PlanId,
    delivered: Option<bool>, acknowledged: Option<bool>, stopped: Option<bool>,
    receipt: Option<OperationReceipt>, checks: Vec<Check>,
}
pub struct RemoteRequestRecord {
    schema_version: u32, request_sha256: String, request: RemoteRequest,
    delivery: DeliveryRecord, cancel: Option<CancelResult>,
}
pub struct RemoteFailure {
    error: ContractError, delivery: Option<DeliveryRecord>,
    request_may_have_arrived: bool,
}
pub struct RemoteValidationEvidence {
    plan_digest: PlanDigestEvidence, plan_bytes: Vec<u8>, plan_sha256: String,
    request_bytes: Vec<u8>, request_sha256: String, verification_ref: String,
}
pub fn validate_remote(input: &Value, evidence: Option<&RemoteValidationEvidence>)
    -> Result<RemoteMessage, ContractError>;
pub fn send_change(session: &mut RemoteSession, request: &RemoteRequest)
    -> Result<DeliveryRecord, RemoteFailure>;
pub fn query_change(session: &mut RemoteSession, request: &QueryRequest)
    -> Result<ReconcileResult, RemoteFailure>;
pub fn cancel_change(session: &mut RemoteSession, request: &CancelRequest)
    -> Result<CancelResult, RemoteFailure>;
```

各メッセージのschema_version候補は1、kindはsubmit/query/cancel/indexで、kindとpayloadは一つだけ対応させる。QueryRequestとIndexRequestは変更計画を受け取らず、再送の代用品にしない。CancelRequestのID一致を権限証明にはせず、接続先・マスターの管理権限・現在の操作を再検査する。RemoteFailureは配送開始後の失敗を保持し、request_may_have_arrivedがtrueまたは判定不能なときは再投入不可とする。falseには送信前停止を確認した根拠が必要。

Submitの検証はPlanの全キーを未封印値として検査→P03の固定byte列とplan_digest証拠を照合してseal_plan→Plan全byte列/plan_sha256→RemoteRequest全byte列/request_sha256の順とする候補。RemoteValidationEvidenceは採用hash取得器が同じ検証対象byte列へ束縛した証拠だけで構築し、文字列hexの自己申告を受けない。validate_remoteはI/Oなしで三byte列と証拠の対応を照合する。Submitで証拠なし、他kindで余分な証拠ありはREMOTE_BINDING。受信validatorを封印回避のPlan constructorにせず、比較digestと全入力hashを代用しない。A09-17/35へ同じdigestでも別Plan ID、別byte列のhash証拠、QueryへSubmit証拠の混入を含める。

last_confirmed_statusのSome(Unknown)は型上表現可能でもvalidatorで拒否する。status=Unknownだけが直前の通常段階またはNoneを保持し、statusが通常段階なら同値を要求する。OperationReceiptを含む全子型も担当validatorへ照合し、再帰的なPlan/Request/CommandDataをreceiptへ内包しない。これらは外部全キー表のRust表現であって第二の配送形式ではない。

| ケースID | 固定入力 | 期待結果 |
|---|---|---|
| `A09-35` | last_confirmed_status=unknown、Cancel receiptは別Plan、RemoteRequestRecordの子型に未知キー | REMOTE_BINDING。型名だけの受理・他要求結果の転用・再帰payloadなし |

RemoteFailure.request_may_have_arrivedはboolの候補で、到達可能性が不明な場合もtrueへ保守的に結合し、falseを既定値にしない。Queryは対象のremote/operation/history/監督単位を取得してReconcileResultをメモリー上で生成するだけで、欠損記録の作成・deliveryの修復・取消を行わない。同じSubmit ID/hashを再受信しても既存結果の読取りだけ、同ID別hashはREMOTE_COLLISIONで保持済み記録不変。Cancelのdelivered/acknowledged/stoppedは独立の証拠を要し、acknowledged=trueかつstopped=nullから終了確定へ進めない。部分受理後の保存失敗はRemoteFailure.deliveryに最後の確認済み段階を保持する。

取消と終端の競合候補では、既にresult_readyなら同じ確定receiptを返し、元操作をinterruptedへ書き換えない。実行中なら取消対象のoperation/監督単位を照合して停止工程へ一度だけ渡す。同じCancelの再受信は取消位置の照合だけで追加signalを発行しない。delivered=true/acknowledged=falseは対象へ到達したが受理拒否の確証、stopped=nullは停止未確認。acknowledged=trueなのにdelivered=false/nullという矛盾はREMOTE_CANCELで拒否する。停止の確証がない限り、取消の受付をstopped=true・データ復元・後続変更の許可へ変換しない。

Cancelの候補経路は実行中workerへの制御要求であり、新しい変更operationをbeginして対象の同じexclusive guardを待つ処理にはしない。実行者が保持するguardの下で取消の受理位置/保全/停止結果を確定し、受付側は元operation/historyを直接上書きしない。workerが応答不能ならacknowledged/stoppedを未確認のまま返し、guard取得成功だけを強制終了・取消完了の根拠にしない。安全な制御伝達の実方式はD09-02へ残し、成立しない間は取消可能としない。A09-08/12/30で実行者がguardを保持中の取消と終端競合を照合する。

RemoteSessionは検証済み接続と対象証拠を所有し、Cloneを許可しない候補。drop・SSHプロセス終了は対象操作の取消・ロック解除を意味しない。固定入口・初回搬入の実現手段が未決なため、これらの宣言だけで配送可能とはしない。

RemoteSessionは接続条件と確認資料を束縛する論理セッション候補で、単一のSSHプロセスへ複数JSONを追記する持続接続の保証ではない。一メッセージごとに固定入口/一stdin/EOF/一応答を使用する。Query/Cancelのtarget_idはsessionの台帳IDと一致させ、接続側で検証した主体・権限を照合し、payloadにIDがあるだけで問合せ/取消を許可しない。認証参照は検証済みの接続資格から導き、要求の自己申告で置換しない。ホスト鍵/実対象/接続条件の変化・期限切れはセッションを失効させ再接続へ戻す。対象側固定入口の権限実装は安全ポリシーとD09-01の採用待ちであり、型宣言で成立済みとしない。

## 状態と中断の詳細案

`P09-REMOTE`の配送は未送信・配送開始・accepted・started・result_readyを区別する候補。unknownは観測不能であって、対象側を未送信へ戻す状態ではない。

| 切断位置・観測 | 再照合と禁止する動作 |
|---|---|
| stdin配送開始前の停止が確実 | request_may_have_arrived=false。変更を送っていない根拠を提示 |
| stdinの一部または全部を送信、受理応答なし | true。未受理と推測せずQueryRequestだけを送る |
| acceptedあり、operationなしまたは読取り不能 | unknown。開始前と決めつけず排他・実状態を照合 |
| operation=running、deliveryがacceptedのまま | 開始位置の部分確定。二度目の変更適用なし |
| result_readyと対象必須記録が一致 | 同じ確定結果を返す。返信失敗で適用し直さない |
| result_ready、実状態・履歴・対象IDが不一致 | 成功を受理せず照合待ち。送信側記録で上書きしない |
| CancelRequest受理、停止応答なし | 取消未確認。SSH終了を中断完了にしない |

同じrequest_id・同じ固定要求を再受信した場合は既存記録への照合だけとし、変更入口を再実行しない。同じIDで別要求なら衝突として拒否する。変更の自動再送は常に0回。queryの再試行は読取りだけで、再接続の上限へ算入する。

対象側の処理寿命がSSHセッションへ依存している場合、切断後の停止・続行のいずれも推測しない。セッションから独立した監督単位と記録方式を`D09-01`・`D09-02`で選定し、その確証がない間は取消・排他・再開の仕様充足を認めない。

## 依存と互換性の詳細案

`P09-SSH`はEX-001のOpenSSH等を外部境界とし、SSH/TLS/暗号をRustの外部crateで代替しない候補。信頼・秘密情報・root禁止の条件はセキュリティポリシーだけで定義する。

| 境界 | 採用審査で固定する候補 |
|---|---|
| ローカル接続 | 実行物の絶対パス・固定版・引数全集合。自由な-F/-o/ProxyCommand等の持込みを許可しない |
| 遠隔入口 | 固定した入口と単一構造化stdin。host/user/pathをshell文字列へ連結しない |
| 初回搬入 | 検証済み共通バイナリと検証情報を転送し、対象で再照合。コンパイラー導入を要求しない |
| 昇格 | 非root接続から非対話の明示昇格。対象VPSの追加手入力待ちを残さない |
| 配送互換 | message版・plan形式・対象本体の対応を確認。未知版/未知actionを既存操作へfallbackしない |

固定入口を本体へ内包する案と、独立した監督入口を用意する案を`D09-01`の比較対象とする。前者でも初回の検証済み本体の搬入と実行を自動化する外部契約が必要。後者は共通バイナリ統一方針との影響を明示し、未承認の別エージェントを追加しない。接続成功だけでどちらかが準備済みとしない。

将来のtransport差し替えは切断前後・部分stdin・余分stdout・同一要求再受信・対象再起動を同じfixtureで比較する。配送の成否と対象操作の成否を別に保持し、旧配送記録を読めなくなる変更には移行・照合・切戻し契約を先に確定する。

## 受入条件の充足照合案

SSHは信頼/認証/固定入口、REMOTEは型/配送/排他・記録、RECONNECTは対象同一性/観測再試行へ対応する。配送messageのkindとactionは別に検証し、照合要求へ変更payloadを混在させない。

| ケースID | 区分・固定入力 | 期待結果と確認 |
|---|---|---|
| `A09-14` | 異常。QueryRequestへaction/update計画を混在 | 未知キー/不正messageとして拒否。読取り照合から変更実行0回 |
| `A09-15` | 異常。stdinを1byte配送後に切断、記録を読めない | request_may_have_arrived=true。変更再送0回、照合待ち |
| `A09-16` | 正常・異常。install_os配送と、install_bodyで空ディスク検出 | 前者だけPhase 10へ対応。後者は能力/対象不成立で停止しOS消去へfallbackしない |

固定入口の初回準備・検証済み本体の搬入・昇格・切断後寿命、両側記録の完全形式と排他、全SSH引数と認証参照の契約は未達として残る。接続1回の成功を初回導入・再起動後照合・独立バックアップの成功へ転用しない。

## 入力と保存形式の詳細案

P09-REMOTEの配送形式候補をschema_version・kind・payloadの三キーへ閉じる。JSON UTF-8、末尾LF一つ、入力wrapper全体512 KiB以内・応答全体1 MiB以内、圧縮・複数JSON・余分なstdout文字列を拒否する。payloadの型はkindごとの既存Rust型に一致させる。P02の通常上限を拡張せず、本配送の入力だけをさらに狭める。

RemoteRequestのplan_refは曖昧な文字列を廃し、Planを所有するplanと、その固定byte列のplan_sha256を持つ参照へ具体化する候補。全体のキーはschema_version・request_id・master_machine_id・target_id・target・plan_id・plan_ref・action・contracts。PlanReferenceの全キーはplan・plan_sha256・selection・install_payload・backup_set・confirmation。planはP03のPlan、plan_sha256はhex64。plan_sha256はPlanの全キー（plan_id/created_at/digestを含む）をP02のキー順JSON・末尾LFなしで直列化したbyte列のSHA-256とし、P03の前提比較用Plan.digestとは別の照合値とする。selectionは初回setup/install_osだけ必須で継続管理/self単独はnull、install_payloadはinstall_osだけ必須、backup_setは独立backupと保全を伴う計画で必須。他はnullとする。confirmationは[P05のConfirmation](PHASE_05.md#入力と保存形式の詳細案)の型/二応答を共用し、Submitでは必須・null不可とする。媒体のmedia_idや取外し条件は追加しない。対象・plan_id・構成・採用版の全一致を対象側で再検証する。入力上限を超えるPlanは配送前に拒否し、外部URL参照へ自動変更しない。

confirmationはマスターで実施した選択後の計画確認を当該Plan/対象disk/digestへ結合する候補であり、認証・権限の証明ではない。対象側は接続資格の照合と別に、plan_id/target_disk_id/shown_digest/応答/期限、消去stepの有無と追加確認の対応を検証する。未確認・別計画・消去確認欠損ならaccepted保存/worker変更0件。Query/Cancel/Indexにはconfirmationを受け取らない。A09-14/17/27で不足・別Plan・短いコードだけ一致を照合し、確認が配送で失われる経路を認めない。

Submitの共通参照閉包にはP03-PLANと、Confirmationを定義するP05-MEDIAも含める候補。後者の型の共用から、USB操作/媒体取り外し/物理PC起動をCloudの実行条件へ追加しない。request_digestと保存/応答上界はconfirmationを含む六キーのPlanReference全体から再算定する。旧五キーの要求は欠損として拒否し、対象側で確認済み値を補完しない。A09-17/27/33/36で旧要求・全要求hash差・上界超過を照合する。

| 対象記録 | 全キー候補・配置 |
|---|---|
| RemoteRequestRecord | schema_version: 1、request_sha256: hex64、request: RemoteRequest、delivery: DeliveryRecord、cancel: CancelResultまたはnull。/var/lib/serverseed/remote/内のrequest_id別固定ディレクトリ |
| 取消の照合 | CancelResultの全集合は[データ契約](#データ契約の具体化案)を参照。delivered/acknowledged/stoppedのnullは未確認、falseは否定の確証。stopped=trueには監督単位終了の証拠が必要 |
| ReconcileResult | 既存キーにrequest_sha256: hex64または未取得時null、record_consistent: boolまたは未確認時nullを結合。同じrequest_idで異なる確認済みhashならREMOTE_COLLISION |
| 配送受理 | 対象記録を新規・同期保存してからaccepted。対象側のoperation開始とdelivery.startedを結合し、result_readyは操作の必須記録完了後だけ |

request_digestはRemoteRequest全体をP02-CLIのキー順/escape・末尾LFなしで固定したbyte列のSHA-256とする。Plan.digestと異なりrequest_id/plan_id/時刻も除外しない。RemoteRequestRecord.request_sha256とDeliveryRecord.request_digestは同じ値へ照合する。queryは対象記録の読取りだけ。記録不在は未配送の証明にせず、missingと操作・履歴・監督単位の照合結果を返す。取消も対象操作IDと同じ監督範囲に限定し、他要求やSSHセッション全体を停止対象にしない。記録の保存失敗・応答消失時のpartialはRemoteFailureへ保持する。

配送前の候補検証はmessage版/型→対象台帳とmaster識別→要求/Planの全ID・hash・契約参照→actionとpayload→状態別実行条件→対話選択/確認→対象側の再照合とする。install_bodyはself installの条件、setup/update/recoverは各既存状態条件、backupは構成と対象データの確認、install_osはP10の能力/消去条件を使う。install_body/setupが空diskを見つけてもinstall_osへ変換しない。install_osのSelectionだけで後続setupの別のplan_idを確認済みにはせず、同じ確認済み親計画の工程対応を要求する。

ReconcileResultのresubmit_allowedは照合上の可否だけとする候補。trueでも自動再配送0回を維持し、同じ要求IDを新操作へ再使用しない。明示的な再計画を行う場合は新request/planと元要求の未適用確証または復旧完了根拠を対応付ける。Query/Cancelの応答欠損もそれぞれ未確認のまま保持し、要求の種類をSubmitへ切り替えて照合しない。

| ケースID | 固定入力 | 期待結果 |
|---|---|---|
| `A09-17` | 同じrequest_idでplanの一属性を変更、JSON後にbanner、1 MiB+1 byte | 衝突または形式不正で拒否。変更の再投入・bannerの黙除・切詰め受理なし |

## 実行と復旧の確定候補

P09-SSHの第一候補は固定OpenSSH実行物へargvを渡す方式。ユーザー設定・agent・多重化・forwardingを意図せず継承しない。固定config、IdentityFile、既知ホストファイルはセキュリティポリシーで審査した参照からだけ解決する。

[OpenSSH公式設定](https://man.openbsd.org/ssh_config)へ照合し、-T、BatchMode=yes、IdentitiesOnly=yes、StrictHostKeyChecking=yes、ConnectionAttempts=1、ConnectTimeout=30、ControlMaster=no、ControlPath=none、ForwardAgent=no、ClearAllForwardings=yesを候補とする。ConnectTimeoutだけで要求全体の実行上限を保証しない。外側のP02-EXECと再接続予算も適用する。Ubuntu採用版での各オプション照合は未実施。

remote commandは登録された固定入口文字列だけ。動的target/plan/path/actionをremote shellのコマンド文字列へ連結せず、全変更入力は構造化stdinで渡す。初回本体搬入・昇格・固定入口の設置方式は別の導入契約の採用待ちであり、ここをssh任意コマンドの実装裁量にしない。

対象workerはP02-EXECの監督単位で生存し、SSH切断は取消ではない。取消ackは要求の受理だけ、stopped=trueはunit内全処理と必須記録の確認後だけ。再接続は600秒以内・一接続30秒・間隔5秒・最大12回という既存候補を維持し、最初の接続を一回と数える。最終回後の待機と十三回目はなし。queryの成功でも対象証拠・記録hash不一致なら変更再開不可。

| ケースID | 固定入力 | 期待結果 |
|---|---|---|
| `A09-18` | SSHプロセス終了、対象worker継続、取消ackだけ返却、queryで別disk応答 | 未完了を保持し、変更再送0回・stopped=trueなし・対象不一致で保留 |

## 依存と受入の確定候補

P09のadapterはtransport/remote protocol/reconcileの三境界とし、認証・host key・昇格条件はSECURITY_POLICYだけを参照する候補。SSHそのものを自作暗号や外部Rust crateへ置換しない。

| 担当 | 型付き契約と受入 |
|---|---|
| transport | RemoteSessionと固定入口、stdin byte列を受ける。P02-EXECの送達/終了/制限を保持し、非ゼロ・切断を未変更と推測しない |
| remote protocol | RemoteMessageを検証し同じrequest_id/hashへ既存記録を対応付ける。同一ID別payloadは衝突、同一payloadは再適用0回 |
| reconcile | 対象証拠・remote記録・operation/history・監督unitを読み取り結合。欠損/不一致はunknownまたは不整合。送信側の記録で対象を修復しない |

必須検査ID候補はP09-SSH/trust.bound、P09-REMOTE/request.bound、P09-REMOTE/records.consistent、P09-RECONNECT/identity.matches。再接続のpassでも他の必須検査を代替しない。採用するOpenSSH/昇格/初回搬入/固定入口の実行物・全argv・版・ライセンスを例外台帳へ個別登録するまで使用不可。

差し替え試験はstdin各byte境界の切断、accepted/started/result_ready各境界の欠損、同じIPの別対象、取消ackのみ、十二回目のqueryを固定列として再現する。新旧のRemoteFailure/送信回数/対象変更回数を比較し、出力一致だけで二重実行を見逃さない。

| ケースID | 固定入力 | 期待結果 |
|---|---|---|
| `A09-19` | transportを差替え、accepted直前に切断、次のqueryでは結果確定 | request_may_have_arrived=true、submit再送0回、同じ確定結果へ照合。対象変更は一回以下 |
| `A09-20` | 初回本体は搬入済み、root所有確定または昇格の証拠なし | worker開始0回。転送成功を導入準備完了にせず、別agent/任意shellで補完しない |
| `A09-21` | master_id旧別名、取消stopped=null、requestのplan_idだけ変更 | 別名拒否・取消未確認・要求hash変化。Plan.digest一致だけで同一要求と扱わない |
| `A09-22` | 十二回目の接続成功、identity一致だがrecord_consistent=null、残期限0 | 成功/再投入を許可せず照合待ち。追加待機・十三回目・submit再送なし |
| `A09-23` | 四搬入物は同じ版名だがmanifest hash相違、または昇格契約未確定 | SSH_BOOTSTRAPでworker起動0回。独自成果物を外部EX登録で代用せず対象側再検証を要求 |
| `A09-24` | Query先のdelivery欠損、到達可能性不明、同ID別hashのSubmit | Query保存0件/可能性true/REMOTE_COLLISION。記録修復・再配送・既存要求の置換なし |
| `A09-25` | 十一試行終了後の待機中に600秒到達、十二試行目は接続成功だがQuery不明 | 前者は十二回目開始なし、後者は未確認で終了。接続成功で回数/期限を更新せずsubmit再送0回 |
| `A09-26` | 本体/manifest/署名だけでbuild metadata欠損、別CPUのloader条件 | SSH_BOOTSTRAP、候補起動0回。転送成功・署名成功だけで実行依存を省略しない |
| `A09-27` | setupで空disk、resubmit_allowed=trueだが新計画/確認なし | install_osへの変更・自動再送0回。元要求の照合結果を新操作の承認にしない |
| `A09-28` | sessionは対象a、Query/Cancelは対象b、自己申告master ID、二JSONを一stdinへ追記 | 束縛/権限/形式不一致で拒否。一致するIDだけで照合・取消を許可せず、持続接続を捏造しない |
| `A09-29` | started後の応答欠損、終端失敗のreceiptは確定、startedでoperation_id=null | 最後のstartedを保持し再投入なし。確定失敗は回収可だが成功化せず、ID欠損のstartedは拒否 |
| `A09-30` | 完了直後のCancel、同じCancel再受信、ack=true/delivered=null | 元確定結果を保持し追加停止0回。矛盾はREMOTE_CANCEL、受付だけの中断完了・結果書換えなし |

### SSH起動と初回搬入の分離

ConnectionLimits/BootstrapTransfer/MasterRemoteRecordは担当全キー表の型候補。接続/検査/転送時間はmsへchecked変換して本書の各上限へ照合し、0・親残期限不足を接続前拒否する。MasterRemoteRecord.sent_bytesは固定wrapperのLF込み実長以下で、要求hashはRemoteRequest全体へ一致させる。初回転送のstagingが新規でも、SFTP側の初期mode/所有者/親の置換排除が確認できなければputを開始しない。mkdir後のchmodだけを作成からの保護根拠にせず、採用server/subsystemの生成条件を別途必要とする。A09-23/34/37へ単位違い・送信量超過・初期mode不明を含める。

初回転送の具体候補はOpenSSH SFTPのbatchモードとする。SSHと同じ固定config/鍵/host検証、-b -、-B 32768、-R 1、-Pの固定portを使い、転送指示だけをstdinで渡す。destinationは検証済みuser/hostから生成し、pathや任意SSH設定を付加しない。圧縮・再開・再帰転送・失敗無視・ローカルshell escapeを使わない。[公式SFTP](https://man.openbsd.org/sftp)のbatch失敗停止と、転送完了/同期/真正性の別判定へ照合する。

staging候補は対象の/var/tmp/serverseed-bootstrap一件。新規mkdirが失敗したら既存領域を再利用/削除せず、所有者・種別・未完了初回要求を照合する。新規成功後にmode=0700を確認し、四資材をserverseed/manifest.json/manifest.asc/build.jsonの固定名だけへputする。実行候補serverseedだけに固定chmod 0700を行い、データ三件は0600として種別/所有者/実modeを確認する。ローカル元名も保護領域内の同じ固定名としてglob/空白/引用の解釈を不要にする。put完了後の再取得/hash一致は転送の検証であってroot実行の許可ではない。転送上限300秒と各資材量を維持し、途中失敗の再開/上書き0回。

root確定の比較候補は、既存OS側の信頼済みプロビジョニング入口が四資材をroot専用領域へ固定し再検証する方式と、独立承認済みdigest付きsudo実行契約で同一バイナリの初回内部入口を呼ぶ方式の二つ。後者でも[sudoersのdigest/fdexec](https://www.sudo.ws/docs/man/1.9.14/sudoers.man.pdf)だけを、開いたinodeへの同時書込み・動的loader・親の保護の全証明にしない。対象実体の固定/書込み排除・環境/昇格・信頼を別に成立させる必要がある。

どちらも通常のserverseed-remote-entryの存在を前提にせず、事前に存在するOS能力と所有者が認めた権限からだけ開始する。入口・実版・権限が未指定ならSSH_BOOTSTRAPで停止する。SSH認証成功からroot権限を生成する方式はなく、SSHだけを外部入口とすることと、内部の必要能力が何も不要であることは同義にしない。新しいsudoers設定/初回権限の付与は採用前に実施しない。

| ケースID | 固定入力 | 期待結果 |
|---|---|---|
| `A09-37` | staging既存/途中put、四資材一致だが昇格入口なし、fdexecだけ確認し同時writerあり | 再利用/自動再開/worker開始を行わない。未設置wrapperへの依存循環・転送成功だけのroot実行・digestだけの対象固定認定なし |

通常配送のargv候補は`ssh -F /dev/null -T -o BatchMode=yes -o IdentitiesOnly=yes -o IdentityAgent=<固定agent_socket> -o StrictHostKeyChecking=yes -o UpdateHostKeys=no -o VerifyHostKeyDNS=no -o UserKnownHostsFile=<固定known_hosts> -o GlobalKnownHostsFile=/dev/null -o ConnectionAttempts=1 -o ConnectTimeout=30 -o ControlMaster=no -o ControlPath=none -o ForwardAgent=no -o ForwardX11=no -o ClearAllForwardings=yes -o PermitLocalCommand=no -o ProxyCommand=none -o ProxyJump=none -i <固定公開identity> -p <port> -l <user> -- <host> <固定入口>`とする。各空白区切りは独立した引数で、山括弧部分は検証済み参照の解決値だけ。host/user/portの文法は台帳の正本へ照合し、hostをoptionとして扱わない。cwdはマスターの固定作業領域、環境はLANG/LC_ALLと必要な固定PATHだけを新規構成し、SSH_AUTH_SOCK・SSH_ASKPASS・DISPLAYを継承しない。秘密鍵の保護条件はセキュリティポリシーで審査する。

本候補の認証は、所有者が許可したローカルagentの固定socketと公開鍵の組に限定する。[OpenSSH公式設定](https://man.openbsd.org/ssh_config#IdentityAgent)のIdentityAgentは環境変数の継承と別に指定でき、IdentityFileはagent内の鍵を選ぶ公開鍵にもできる。暗号化秘密鍵を非対話SSHが自力で解錠できるとは仮定しない。解錠/署名操作を含む資格準備はマスター側の事前工程で完了し、対象VPSでの追加入力、平文秘密鍵file、汎用agentの自動探索へ切り替えない。socket所有者/実体/利用資格/寿命と対象公開鍵の対応が未確認ならSSH_AUTHENTICATIONで停止する。SFTPも同じ固定認証契約を使う。方式の採用と実認証の根拠はD09-01に残す。

| ケースID | 固定入力 | 期待結果 |
|---|---|---|
| `A09-40` | 暗号化鍵だけでagent未準備、別socketへの交換、agentの鍵と公開identityが不一致 | 変更配送前停止。平文鍵へのfallback・環境agentの継承・対象での解錠待ちなし |

認証方式の閉包候補として通常argvの-o列へPreferredAuthentications=publickey、PubkeyAuthentication=yes、PasswordAuthentication=no、KbdInteractiveAuthentication=no、GSSAPIAuthentication=no、HostbasedAuthentication=no、PKCS11Provider=none、AddKeysToAgent=no、CanonicalizeHostname=noを明示する。方式別の意味は[公式設定資料](https://man.openbsd.org/ssh_config)へ照合し、BatchModeだけを方式制限にしない。未対応optionを削って動かさず、固定版の全有効設定と実認証をD09-01で確認する。

agent socket/公開identity/known_hostsの解決値は、固定の絶対pathでASCII英数字・/・dot・underscore・hyphenだけ、空成分・.・..を拒否する候補。OpenSSH側のtoken/環境/tilde展開を起こす値を受けず、引数配列だから再解釈がないとは扱わない。公開identityだけを置く専用保護領域とし、隣接する-cert.pubや秘密鍵の暗黙読込み、agent内の別資格/証明書による認証がないことを固定版の候補集合へ照合する。IdentitiesOnlyだけで証明書の自動探索まで無効になるとはしない。未確認なら認証前停止し、未確認のCertificateFile値を創作しない。SFTPは同じ候補設定集合をその対応する-F/-o/-i引数へ束縛し、ssh専用フラグをSFTPへコピーしない。

| ケースID | 固定入力 | 期待結果 |
|---|---|---|
| `A09-41` | agent参照に%h/$/tilde、隣接-cert.pub、passwordのみ成功する接続先、未対応固定option | 入力/資格候補/方式/adapterの各不成立で配送0回。token展開・別認証・option削除での続行なし |

固定入口の候補は`/usr/bin/sudo -n -- /usr/local/libexec/serverseed-remote-entry`という変更不能な文字列一つ。構造化stdin以外からactionやpathを取らず、wrapperは所有者・mode・hashと許可された呼出主体を確認し、同一本体の内部入口だけへ引き渡す。root-ownedであることだけでは許可主体を証明しない。初回にwrapperがない対象へ、この通常配送を呼び出してはならない。

初回搬入はBootstrapTransferの四資材を、選定するSFTP等の固定転送契約で専用stagingへ送り、対象上で署名・hash・CPU/loaderを検証してからroot所有領域へ公開する別工程とする。転送方式・stagingの確保・初回昇格の固定入口が未確定なら`SSH_BOOTSTRAP`で停止する。上記wrapperを設置するために未設置wrapperを要求する循環、任意shellスクリプトの送信、SCP/SFTPの暗黙採用は禁止する候補。現時点では初回搬入の成立性は未確認で、SSHのみを外部入口とする要求は維持する。

[OpenSSHのコマンド配送](https://man.openbsd.org/ssh)と[設定項目](https://man.openbsd.org/ssh_config)を比較資料とする。これはUbuntu採用版の全オプション・実権限・初回搬入の証明ではない。固定版で未対応のoptionが一つでもあれば削除して続行せず、adapter不成立として採用審査へ戻す。

| ケースID | 固定入力 | 期待結果 |
|---|---|---|
| `A09-31` | ユーザーconfigへProxyCommand、agent設定、初回対象にwrapperなし、未対応optionを各注入 | 継承0件または起動前停止。任意shellへのfallback・初回準備済み扱い・秘密値のargv混入なし |

### 両側の配送保存と容量上限

旧マスターの保全後に発行された要求も照合するため、非変更のindexメッセージ候補を追加する。通常のQueryは既知一要求の照合、indexは認証済みの回復主体に許された旧master_idの全要求識別を取得する。master_idの自己申告だけで閲覧権限を与えず、回復資格の安全条件が未採用なら実行しない。公開CLIの追加構文ではなく、P08の復元照合から使う内部契約である。

```rust
pub struct IndexRequest { master_machine_id: MachineId, offset: u32, generation: Option<String> }
pub struct IndexEntry {
    request_id: OperationId, plan_id: PlanId, request_sha256: String,
    operation_id: Option<OperationId>, status: DeliveryStage,
}
pub struct IndexResult {
    master_machine_id: MachineId, generation: String, total: u32,
    offset: u32, entries: Vec<IndexEntry>, next_offset: Option<u32>,
}
pub fn query_index(session: &mut RemoteSession, request: &IndexRequest)
    -> Result<IndexResult, RemoteFailure>;
```

一pageはrequest_idのASCII順で最大128件。totalは既存上限内の全件数、generationは同じ主体の全IndexEntryを固定順/JSONで結合した元byte列のhash。初回offset=0/generation=None、続きは返却generationとnext_offsetだけを使う。既存共通lockを新規作成せず非待機shared取得して読み、競合ならREMOTE_BUSYで変更/記録修復0件。active workerがexclusive所有中は索引を完結できたとしない。ページ間で世代が変われば最初から再照合し、部分集合を全件成功にしない。

各pageは通常のbyte/深さ/ノード制限内、全列挙/検証はP09再接続の同じ残予算内で行う。最終pageの後に同じgeneration/totalを照合し、件数/ID重複/不足を検出する。ここで得た各IDへ通常Queryを対応付け、一覧のstatusだけで実状態/記録の整合を認定しない。索引や回復のための管理file作成・Submit・Cancelは0件。

| ケースID | 固定入力 | 期待結果 |
|---|---|---|
| `A09-38` | 129要求の二page、間で新要求、旧masterの未保全要求、回復権限不明 | 全page/世代不一致/追加IDのQuery/権限停止を照合。128件での打切り・旧保全物だけの全件認定・索引からの変更なし |

マスター側の送信記録候補は`/var/lib/serverseed/master/remote/<request_id>/request.json`一つへ集約し、delivery.json/observations.jsonという第二の正本を作らない。対象側のRemoteRequestRecordは`/var/lib/serverseed/remote/<request_id>/request.json`へ集約する。前者の全キーはschema_version=1・request_sha256・request・sent_bytes・last_delivery・observation_refs、後者の全キーは既存のRemoteRequestRecordを参照し、別の受理形式を作らない。last_deliveryはDeliveryRecordまたは未取得時null、observation_refsは所有者管理下の検証済み証拠IDの一意配列で、外部URL/任意path/未保存の資料を受けない。sent_bytesは確認できた送信byte数で、0であっても送信開始後の未確認を未配送の確証にしない。

送信記録の置換はP02の原子的保存を使い、sent_bytesは0から送信したwrapperの全byte数までの単調非減少値とする。last_deliveryの後退・既知operation IDの変更はREMOTE_COLLISION。Queryはこのファイルを変更せず、所有者が選択した照合結果保存工程だけがobservation_refs/last_deliveryを更新する。全量未送信でも一部受理の可能性を保持し、送信量を到達/適用量に読み替えない。

| ケースID | 固定入力 | 期待結果 |
|---|---|---|
| `A09-34` | last_deliveryはstarted、旧accepted応答を後受信、sent_bytesが減少 | 正本の後退を拒否して既知startedを保持。二ファイルの後勝ち・Query修復・部分送信の未配送化なし |

マスターは要求の同期保存→送信→応答検証→last_delivery同期の順。対象は固定入口の非変更検証→同要求に束縛した監督workerの起動→worker自身の共通guard取得→ActiveController照合→同ID/hash検索→新規要求同期→accepted→同じworkerのbeginと監督unitの対応同期→started→必須結果同期→result_readyの順とする候補。受付プロセスがguardを保持したまま別workerの取得を待たず、acceptedから記録確定まで一主体が所有する。同名worker/未完了要求を発見した入口は第二workerを起動せずQueryへ戻す。起動予約・入口間競合のOS方式はD09-02の成立条件であり、名前の事前確認だけでは解消しない。受理後の一段だけの保存成功は段階の不一致として保持する。Queryは両側の欠損・hash・operation・unit・実対象の照合結果だけを返し、保存の補修を行わない。照合結果を保管する行為はマスター側の変更工程として区別する。

Queryは実行中workerのexclusive guard取得を待つ変更処理へ変換しない候補。原子的に保存された確認可能な同一要求の位置だけを読み、不整合/採取中変化はrecord_consistent=null/falseとして返す。workerが書込み中という事実だけを元操作の失敗とせず、全必須記録が揃うまでresult_readyも生成しない。Indexの非待機shared取得/REMOTE_BUSYは別の全件索引契約を維持する。A09-04/10/24/38で実行中のQuery、同時Submit、Index競合を分けて照合する。

配送記録の候補上限は両側それぞれ1,000要求かつ専用領域64 MiB、新規要求・原子的保存の一時byteを含む。要求payloadは入力wrapper込み512 KiB内、各delivery/cancel/観測一覧はそれぞれ64 KiB以内、合成した各request.json全体は1 MiB以内とする。JSONの実byte列（escape/区切り/末尾LFを含む）で評価し、512 KiBの要求が1 MiBの記録へ無条件に収まるとは仮定しない。受理前に合成上限と当該Planの結果上界を照合し、上界を確定できない場合もREMOTE_CAPACITYで受理を拒否する。開始後の予期しない超過では元の対象側証拠と最後の配送段階を保持し、receiptのstepを切り詰めてresult_readyにしない。

上限のいずれかを超える計画は受理前停止し、古い未完了要求の自動削除・ID再利用をしない。上限ちょうどは許容するが、新規/一時byteを含む合計を別途満たす必要がある。完了記録の退避・除去は別の明示保守計画で、対象のoperation/historyと対応を失わない条件を要求する。これらはP02の履歴保持条件を緩和しない。

byte数に加え、wrapper・両側保存記録・応答の完成した各文書をP02の通常profileで検査する。内包Plan単体の深さ/ノードが上限内でも、外側objectの階層/キーと結果集合を合算し直す。要求の一部だけにprofileを適用せず、受入専用16 MiB profileを遠隔保存へ使用しない。事前算定する結果上界にもこの深さ/ノード/文字列/論理予算を含め、未確認なら受理前停止とする。

| ケースID | 固定入力 | 期待結果 |
|---|---|---|
| `A09-36` | Plan単体depth=32、wrapper全体depth=35、保存記録はbyte内だがnode超過 | REMOTE_CAPACITY/共通解析上限で受理前拒否。内包物だけの検査・専用profileの転用なし |

| ケースID | 固定入力 | 期待結果 |
|---|---|---|
| `A09-33` | wrapper 512 KiB/+1、合成記録1 MiB/+1、結果上界が不明 | 境界内だけ次段へ。超過/不明は受理前拒否し、共通パーサー上限解除・部分receipt成功化なし |

| ケースID | 固定入力 | 期待結果 |
|---|---|---|
| `A09-32` | マスター送信記録だけ確定、対象accepted直後保存失敗、1,001件目または64 MiB超過 | 到達可能性を保持し変更再送0回。容量超過は受理前停止、自動削除・Queryによる記録修復なし |

## 未決事項と成立判定

### 配送結果の事前上界

出力専用の狭い上界候補として、Diagnostic/ContractErrorのcodeは既存の固定コード集合・ASCII128bytes以内、messageは固定分類文・UTF-8で256bytes以内、fieldは既知Pointer/引数名またはnull・1,024bytes以内とする。Check.message/code/fieldも同じ条件、Check.idは登録済み固定IDの実長で計数する。入力stringの64 KiB制限や一般設定の受理範囲を変更しない。原入力/生診断を切り詰めてこの条件へ合わせず、固定分類と検証済み位置だけを生成する。上界違反の出力を成功receiptへ変換しない。

固定済みtarget/ID/参照/記録pathは親Planと保存集合の実byte長、未固定の診断位置だけ上記最大値へ対応付ける。各stepのCheck集合はrequired_checksと担当登録から求め、診断は同じcode/fieldの集約後の最大件数を全失敗枝から算定する。無関係な全stringへ64 KiBを機械適用して全計画を常に拒否する算定と、根拠なく短く見積もる算定の両方を避ける。未登録コード/可変参照集合の上界不明は既定どおり受理前拒否。

```rust
pub struct ResultBound { bytes: u64, nodes: u64, depth: u32, logical_bytes: u64 }
```

A09-33/39には同じ短い正常計画の全失敗枝、最大長diagnostic、escape長、固定参照と未固定文字列の誤分類を含める。容量内となる非空計画のfixtureも必要とし、拒否するケースだけで配送の成立を認定しない。実行成功の証拠とは別に、候補の有限性/容量内経路を検証する。

受理前の計算候補ResultBoundはbytes/nodes/depth/logical_bytesの四つの非負整数。型の全variantと必須/失敗枝を展開し、固定済みID/参照/時刻/文字列は実直列化長、未固定stringは担当契約のUTF-8長上限mに対する6*m+2、integerは20bytes、boolは5、nullは4を上界に使う。object/arrayの区切り・キー・末尾LFと、P02のキー込みノード数/深さ/Mを加算する。配列数は親Planの全step/固定Check集合/記録参照集合の最大件数で、実行成功時だけの短い枝へ限定しない。

各型の有限集合を再帰的に辿り、同じ型の再帰内包、未知の出力variant、string/配列の上限不明、checked演算失敗はREMOTE_CAPACITY。DeliveryRecord.receiptの全step、失敗Diagnostic、CancelResultと両側の保存wrapperを別々に合成する。delivery/cancel各64 KiB、request保存1 MiB、応答1 MiB、および通常解析profileの全制限を独立に照合する。小さい上界を選ぶため検査を削らず、上限内のPlanだけを配送可能にする。

出力の動的集合/文字列が担当契約で固定されていなければ、この計算は不成立のまま。実出力も同じ型/集合/量へ再検証し、想定外超過はunknownと既存証拠を保持する。上界の成功は容量条件だけで、実行可能性/製品機能の検証成功ではない。

| ケースID | 固定入力 | 期待結果 |
|---|---|---|
| `A09-39` | 正常結果は64 KiB内だが失敗枝は超過、escapeで6倍、上限なし診断、array件数overflow | 受理前REMOTE_CAPACITY。成功枝だけの見積・byteだけの検査・receipt切詰めによる完了なし |

| 判断ID | 必要な判断・確認 | 影響 |
|---|---|---|
| `D09-01` | [SSHと初回搬入](#ssh起動と初回搬入の分離)に通常argv/固定agent案あり。資格準備の安全条件・実認証と、初回転送/署名検証/昇格/入口設置の循環しない実契約・固定版は未完了 | 初回搬入・遠隔実行 |
| `D09-02` | [両側配送保存](#両側の配送保存と容量上限)に順序/上限案あり。対象worker寿命・部分同期の実方式と採用判断が残る。Queryは修復しない | 重複防止・切断復旧 |
| `D09-03` | 三独立照合・有限再接続に案あり。固定fixture/隔離VM・各対象証拠条件と採用判断が残る。成功接続を再送許可にしない | 遠隔の結果判定 |

未達の解消前はSSH接続だけを完全自動導入の成立根拠にしない。固定版・個別例外・非機能要求と共通契約の採用承認も未完了。遠隔接続・対象変更・製品受入は実施していない。
