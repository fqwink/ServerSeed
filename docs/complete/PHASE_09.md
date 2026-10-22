# Phase 9 完全仕様案: 既存OSへのSSH導入・保守

[文書一覧](../DOCUMENT_INDEX.md) | [担当工程](../ROADMAP.md#phase-9-マスターから既存osへのssh導入保守)

採用未承認の`B5`であり、[案の適用範囲](PHASE_01.md#案の適用範囲)を適用する。担当は`REQ-001`・`REQ-002`・`REQ-003`・`REQ-004`・`REQ-005`・`REQ-006`・`REQ-007`・`REQ-008`・`REQ-009`・`REQ-011`・`REQ-012`。既存の[マスターの処理](../master/specification.md#自動処理の流れ)と[Cloud導入](../BASIC_SPEC.md#sshによるcloud環境の導入と保守)を具体化する候補で、OS未導入の導入機構はPhase 10へ分離する。

## 契約と共通参照

| 契約ID | 担当 |
|---|---|
| `P09-SSH` | 接続・認証・搬入・実行経路 |
| `P09-REMOTE` | 計画・対象・操作の結合と配送 |
| `P09-RECONNECT` | 切断・再起動・中断後の照合と再接続 |

使用する契約（全フェーズB5）は、`P02-MODEL`・[識別](PHASE_02.md#p02-model)、`P02-CLI`・[出力](PHASE_02.md#p02-cli)、`P02-STORE`・[排他](PHASE_02.md#p02-store)、`P02-EXEC`・[外部実行](PHASE_02.md#p02-exec)、`P04-SELF`・[配置](PHASE_04.md#p04-self)、`P04-UPDATE`・[更新](PHASE_04.md#p04-update)、`P04-BACKUP`・[保全](PHASE_04.md#p04-backup)、`P08-MASTER`・[対象台帳](PHASE_08.md#p08-master)。信頼・認証・秘密情報の許可条件は[Cloudの管理経路](../SECURITY_POLICY.md#cloudの管理経路)へ参照し、本書で独立したセキュリティ仕様を設けない。

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

初回搬入の審査候補BootstrapTransferはtarget_id・binary_material・manifest_material・signature_material・keyring_ref・staging_ref・entry_ref・privilege_refを全キーとする。先頭の三資材は[P05の独自成果物参照](PHASE_05.md#型とapiの詳細案)のReleaseArtifactRef、残るrefは事前審査した固定入口/保存先/昇格契約のIDであり任意path/オプションではない。SSH・検証実行物のMaterialRefとは区別する。第一候補は検証済み同一本体の搬入と薄い固定wrapper、第二候補は既存の監督入口を利用する方式。どちらも転送→対象側の真正性/実hash検査→root所有領域への確定→同一本体worker開始の全外部契約が必要。独立agent新設は候補に含めない。昇格不能や追加手入力が必要なら配送前にマスターへ不足を返す。

搬入前の候補検証は対象ID→固定リリース版と三資材のmanifest対応→独立した信頼参照→保存先/固定入口/昇格契約→転送予算とする。同じ版名でもmanifest hashが違う組合せをSSH_BOOTSTRAPで拒否する。対象側で再検証・root所有領域の確定が終わるまで候補を実行しない。Connect候補の入力はRegistryEntryとConnectionLimitsの型付き値、ConnectionLimitsの項目はconnect_timeout_ms・transfer_timeout_ms・inspection_timeout_msで本書の各上限に従う。ユーザーのSSH設定断片を追加の引数列へ変換しない。

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

型はP02-MODEL、RemoteSessionの完全な宣言・寿命・排他とpayloadはD09-01〜02の未達。新設案で旧実装は存在しない。将来の差し替えには同じ切断位置・重複要求・鍵変更fixtureを適用し、配送記録・対象操作IDと保存形式の互換性・切戻し限界を確認する。

## データ契約の具体化案

`P09-REMOTE`の配送と照合用データを具体化する候補。共通型は[P02-MODEL](PHASE_02.md#データ契約の具体化案)、対象台帳は[P08-MASTER](PHASE_08.md#データ契約の具体化案)へ参照する。表は既存operation.jsonへのフィールド追加ではなく、制御側・対象側が照合する別の候補データである。

| 型 | 全キー・値域の候補 |
|---|---|
| RemoteRequest | schema_version: 1、request_id: OperationId型、master_machine_id: MachineId、target_id: 台帳ID、target: TargetEvidence、plan_id: PlanId、plan_ref: PlanReference、action: install_body・setup・update・backup・recover・install_os、contracts: ContractRef[]。install_osの入力・実行条件はPhase 10のみで定義 |
| DeliveryRecord | request_id: 上記ID、target_id: 台帳ID、plan_id: PlanId、request_digest: string、operation_id: OperationIdまたはnull、status: accepted・started・result_ready・unknown、receipt: OperationReceiptまたはnull |
| ReconcileResult | request_id: 上記ID、target_id: 台帳ID、delivery: DeliveryRecordまたはnull、identity_match: booleanまたはnull、lock_observed: booleanまたはnull、checks: Check[]、resubmit_allowed: boolean |
| CancelResult | request_id: OperationId、target_id: TargetId、plan_id: PlanId、delivered・acknowledged・stopped: booleanまたはnull、receipt: OperationReceiptまたはnull、checks: Check[]。全キー必須 |

request_idは操作IDと同じ表記・生成器を再利用するが、別に生成する配送の識別子。対象operation_idと同一であるとは要求しない。request_digestの原入力と算定方式は`P03-PLAN`の固定形式と接続して`D09-02`で確定する。同じrequest_idで対象・action・計画・固定資材が異なる要求は衝突として拒否し、前の要求を置換しない。

対象側は、排他・前提を確認してacceptedを永続化し、operationのrunningを確定してからstartedへ進める候補。対象変更は両記録の確認後だけ行う。result_readyには対象の必須記録の確定と実状態の検証が必要で、返信の送信成功とは別に保持する。acceptedだけの記録から「まだ何も適用されていない」と断定せず、対応する操作と実状態を照合する。

ReconcileResultのdelivery=nullは記録を確認できない状態を含むため、それだけでresubmit_allowed=trueにしない。再投入の可否は、未受理の確証または復旧側で照合した許容範囲、現在の排他・対象・計画の一致を必要とする。無応答・timeout・EOFではfalseを維持する。取消の受付・配送・対象の停止を別々に表し、stoppedがtrueでも実資源の復元成功までは示さない。

リクエストは検証済みの1文書を標準入力で渡し、入力上限・UTF-8・複数文書の拒否は共通解析へ参照する。応答もP02-CLIの構造化結果で一度だけ返す案。バナー・進捗・sudoのプロンプトがstdoutへ混ざる経路は成功として解析しない。配送・記録が別々に確定した場合の全中断位置、固定入口の初回準備、親停止後の処理寿命は`D09-01`・`D09-02`の未達に残る。

## 接続と責務の照合

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
pub struct QueryRequest { request_id: OperationId, target_id: TargetId }
pub struct CancelRequest { request_id: OperationId, target_id: TargetId, plan_id: PlanId }
pub enum RemoteMessage { Submit(RemoteRequest), Query(QueryRequest), Cancel(CancelRequest) }
pub enum DeliveryStage { Accepted, Started, ResultReady, Unknown }
pub struct RemoteFailure {
    error: ContractError, delivery: Option<DeliveryRecord>,
    request_may_have_arrived: bool,
}
pub fn validate_remote(input: &Value) -> Result<RemoteMessage, ContractError>;
pub fn send_change(session: &mut RemoteSession, request: &RemoteRequest)
    -> Result<DeliveryRecord, RemoteFailure>;
pub fn query_change(session: &mut RemoteSession, request: &QueryRequest)
    -> Result<ReconcileResult, RemoteFailure>;
pub fn cancel_change(session: &mut RemoteSession, request: &CancelRequest)
    -> Result<CancelResult, RemoteFailure>;
```

各メッセージのschema_version候補は1、kindはsubmit/query/cancelで、kindとpayloadは一つだけ対応させる。QueryRequestは変更計画を受け取らず、再送の代用品にしない。CancelRequestのID一致を権限証明にはせず、接続先・マスターの管理権限・現在の操作を再検査する。RemoteFailureは配送開始後の失敗を保持し、request_may_have_arrivedがtrueまたは判定不能なときは再投入不可とする。falseには送信前停止を確認した根拠が必要。

RemoteFailure.request_may_have_arrivedはboolの候補で、到達可能性が不明な場合もtrueへ保守的に結合し、falseを既定値にしない。Queryは対象のremote/operation/history/監督単位を取得してReconcileResultをメモリー上で生成するだけで、欠損記録の作成・deliveryの修復・取消を行わない。同じSubmit ID/hashを再受信しても既存結果の読取りだけ、同ID別hashはREMOTE_COLLISIONで保持済み記録不変。Cancelのdelivered/acknowledged/stoppedは独立の証拠を要し、acknowledged=trueかつstopped=nullから終了確定へ進めない。部分受理後の保存失敗はRemoteFailure.deliveryに最後の確認済み段階を保持する。

RemoteSessionは検証済み接続と対象証拠を所有し、Cloneを許可しない候補。drop・SSHプロセス終了は対象操作の取消・ロック解除を意味しない。固定入口・初回搬入の実現手段が未決なため、これらの宣言だけで配送可能とはしない。

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

P09-REMOTEの配送形式候補をschema_version・kind・payloadの三キーへ閉じる。JSON UTF-8、末尾LF一つ、入力と応答各1 MiB以内、圧縮・複数JSON・余分なstdout文字列を拒否する。payloadの型はkindごとの既存Rust型に一致させる。

RemoteRequestのplan_refは曖昧な文字列を廃し、Planを所有するplanと、その固定byte列のplan_sha256を持つ参照へ具体化する候補。全体のキーはschema_version・request_id・master_machine_id・target_id・target・plan_id・plan_ref・action・contracts。PlanReferenceの全キーはplan・plan_sha256・selection・install_payload・backup_set。planはP03のPlan、plan_sha256はhex64。plan_sha256はPlanの全キー（plan_id/created_at/digestを含む）をP02のキー順JSON・末尾LFなしで直列化したbyte列のSHA-256とし、P03の前提比較用Plan.digestとは別の照合値とする。selectionは初回setup/install_osだけ必須で継続管理/self単独はnull、install_payloadはinstall_osだけ必須、backup_setは独立backupと保全を伴う計画で必須。他はnullとする。対象・plan_id・構成・採用版の全一致を対象側で再検証する。入力上限を超えるPlanは配送前に拒否し、外部URL参照へ自動変更しない。

| 対象記録 | 全キー候補・配置 |
|---|---|
| RemoteRequestRecord | schema_version: 1、request_sha256: hex64、request: RemoteRequest、delivery: DeliveryRecord、cancel: CancelResultまたはnull。/var/lib/serverseed/remote/内のrequest_id別固定ディレクトリ |
| 取消の照合 | CancelResultの全集合は[データ契約](#データ契約の具体化案)を参照。delivered/acknowledged/stoppedのnullは未確認、falseは否定の確証。stopped=trueには監督単位終了の証拠が必要 |
| ReconcileResult | 既存キーにrequest_sha256: hex64または未取得時null、record_consistent: boolまたは未確認時nullを結合。同じrequest_idで異なる確認済みhashならREMOTE_COLLISION |
| 配送受理 | 対象記録を新規・同期保存してからaccepted。対象側のoperation開始とdelivery.startedを結合し、result_readyは操作の必須記録完了後だけ |

request_digestはRemoteRequest全体をP02-CLIのキー順/escape・末尾LFなしで固定したbyte列のSHA-256とする。Plan.digestと異なりrequest_id/plan_id/時刻も除外しない。RemoteRequestRecord.request_sha256とDeliveryRecord.request_digestは同じ値へ照合する。queryは対象記録の読取りだけ。記録不在は未配送の証明にせず、missingと操作・履歴・監督単位の照合結果を返す。取消も対象操作IDと同じ監督範囲に限定し、他要求やSSHセッション全体を停止対象にしない。記録の保存失敗・応答消失時のpartialはRemoteFailureへ保持する。

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
| `A09-23` | 三搬入物は同じ版名だがmanifest hash相違、または昇格契約未確定 | SSH_BOOTSTRAPでworker起動0回。独自成果物を外部EX登録で代用せず対象側再検証を要求 |
| `A09-24` | Query先のdelivery欠損、到達可能性不明、同ID別hashのSubmit | Query保存0件/可能性true/REMOTE_COLLISION。記録修復・再配送・既存要求の置換なし |
| `A09-25` | 十一試行終了後の待機中に600秒到達、十二試行目は接続成功だがQuery不明 | 前者は十二回目開始なし、後者は未確認で終了。接続成功で回数/期限を更新せずsubmit再送0回 |

## 未決事項と成立判定

| 判断ID | 必要な判断・確認 | 影響 |
|---|---|---|
| `D09-01` | 同一本体搬入+薄い固定wrapperを第一候補としBootstrapTransferへ対応。所有者は転送/署名検証/昇格/固定入口の全argv・個別版・権限を決定。未準備入口への任意shell連結は不可 | 接続・初回搬入・対象実行 |
| `D09-02` | master_machine_id・PlanReference・単一CancelResultと全要求hashを推奨。所有者は部分受理/確定と監督寿命を採用判断。nullの未確認をfalseへ縮約せず対象記録を再照合する | 重複防止・切断復旧 |
| `D09-03` | 三つの独立照合と有限観測再試行を推奨。所有者は固定fixture/隔離VMと各対象契約の適用を決定。再接続の成功を要求再送や対象変更の許可にしない | 成功判定・受入 |

未達の解消前はSSH接続だけを完全自動導入の成立根拠にしない。固定版・個別例外・非機能要求と共通契約の採用承認も未完了。遠隔接続・対象変更・製品受入は実施していない。
