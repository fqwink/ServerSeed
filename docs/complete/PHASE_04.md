# Phase 4 完全仕様案: 導入・更新・保全・復旧とIaC適用

[文書一覧](../DOCUMENT_INDEX.md) | [担当工程](../ROADMAP.md#phase-4-本体導入更新バックアップ復旧とiac適用)

採用審査用の`B4`。契約採用・仕様充足の承認はなく、[案の適用範囲](PHASE_01.md#案の適用範囲)を適用する。対象要件は`REQ-004`・`REQ-006`・`REQ-007`・`REQ-008`・`REQ-009`・`REQ-010`・`REQ-011`・`REQ-012`。

## 契約と依存

| 契約ID | 担当する具体化 |
|---|---|
| `P04-SELF` | 本体取得・互換性・配置・削除・本体専用の復旧 |
| `P04-UPDATE` | OS・構成・アプリと本体の更新・移行の統合 |
| `P04-BACKUP` | 独立バックアップ・復元と更新前保全の共通手段 |
| `P04-APPLY` | IaC資源の適用・検証・中断と復旧 |

共通参照（全フェーズB4）は`P02-MODEL`・[型](PHASE_02.md#p02-model)、`P02-PARSE`・[形式](PHASE_02.md#p02-parse)、`P02-STORE`・[記録と排他](PHASE_02.md#p02-store)、`P02-EXEC`・[実行制御](PHASE_02.md#p02-exec)、`P03-RESOURCE`・[資源](PHASE_03.md#p03-resource)、`P03-FS`・[観測](PHASE_03.md#p03-fs)、`P03-PLAN`・[計画](PHASE_03.md#p03-plan)。共通CLIは`P02-CLI`、秘密情報・真正性・権限は[セキュリティポリシー](../SECURITY_POLICY.md)を参照する。

## P04-SELF

既存の[self入力](../BASIC_SPEC.md#self)、[本体導入情報](../BASIC_SPEC.md#本体導入と構成管理)、[置換と復旧](../BASIC_SPEC.md#本体の置換と復旧)を維持し、以下はそれらの未定義部分に対する候補とする。

| 対象 | 採用案 |
|---|---|
| ローカル入力 | 通常ファイルだけを対象とし、固定版の必須成果物を一式検証。外部入力の相対パスを配置先へ連結しない |
| HTTPS取得 | 候補固定後のメタデータ取得30秒、本体取得300秒、各自動再試行0回。TLS実装・取得用外部実行物はD04-01の未決事項 |
| 容量検査 | 必要量は候補本体・検証情報・既存退避対象・作業一時領域の合計。重複計上と別FSを区別し、計算不能なら停止 |
| 本体診断 | 置換前後のversionとtransaction診断は各30秒、再試行なし。診断失敗とSSH応答欠損を分ける |
| 移行 | self単独では構成・migration revisionを変更しない。互換性不足はP04-UPDATEへ戻す |
| 復旧 | 実ハッシュと署名付き候補・退避物・トランザクションを照合。情報不足なら変更せず停止 |

候補の署名確認・本体ハッシュ照合・自己申告の版照合は、それぞれ別の検査結果を持つ案。署名の成功だけで安全な実行としない。空き容量検査後にも取得失敗・同期失敗を検出し、成功記録を先に確定しない。

P04-SELFの大容量取得結果候補FetchResultはsource_ref・destination_ref・expected_bytes・observed_bytes・sha256・synced・checksの全キー。参照は固定済み資材/新規保存先、bytesは共通整数制約、syncedはbool、checksはCheck[]とする。manifestの宣言量と本書の上限の小さい方で受信を制限し、超過検出の一byteを成功物へ含めない。metadataだけはメモリー、実本体/ISOは採用済み専用file-sinkを推奨する。HTTPエラー本文・TLS不成立・redirect・非ゼロ終了は正式物へ昇格せず、同期と実hash確認が完了するまで候補を起動しない。OS強制容量/停止方式が未成立なら取得開始も保留する。

## P04-UPDATE

既存の[更新順序](../BASIC_SPEC.md#更新処理)に、変更する層・対象構成・成果物・保全・復元確認・再起動を対応付ける。採用案では各変更ステップについて、開始条件、変更資源、期待結果、検証、復旧可否を計画へ保持する。

OSパッケージ操作は1回3,600秒以内、サービス操作は1回120秒以内、総計画は変更開始から3,600秒以内を候補とする。総期限を超える工程を含む計画は開始前に実行不能として返す。パッケージ操作の失敗を同じ入力で自動再投入せず、実状態とパッケージ管理の状態を照合して復旧計画へ移る。

本体変更は既存仕様どおり最後の変更ステップとし、構成の適用済みrevisionと本体収録revisionを区別する。旧二択形式と三構成の移行・不可逆なアプリ更新は、互換範囲と復元検証が確定するまで開始不可。具体的な移行表は`D04-03`に残す。

移行の審査入力候補MigrationRuleはsource_profile・target_profile・source_config_revision・target_config_revision・source_migration_revision・target_migration_revision・method・reversible・restore_methodを全キーとする。構成は既存enum、revisionはP02共通範囲、methodはContractRef、reversibleはbool、restore_methodだけnull可。reversible=trueなら復元方法と新旧双方の読取り互換が必須。適用済み版と一致する行が0件または複数ならblocked。同じ採用版への無変更行とデータ移行行を区別し、版の大小から変換を推測しない。具体的revision値は所有者の別採用を必要とする。

## P04-BACKUP

独立操作と更新内保全を[既存区分](../BASIC_SPEC.md#バックアップの実行と保存先)に従って扱う。本案では利用者が入口側で対象・データ種別・保存先・復元元を選択し、共通処理は確定した計画だけを受け取る。新しい`backup`コマンドを採用済みとはしない。

| 入出力案 | 型・意味・検証 |
|---|---|
| `backup_id` | 操作IDへ結合した一意な識別子。形式はP02-MODELとの対応をD04-02で確定 |
| `source` | 対象ID、構成、データ種別、整合性取得方式、取得対象の集合。稼働環境自身を対象と誤認しない |
| `destination` | 利用者が選択した機器・FS識別と保存位置、容量、既存成果物の検査結果。書込み開始前に再照合 |
| `restore_evidence` | 方法ID、検証した対象・版、実施時刻、期限、期待結果と確認結果。未確認をverifiedへ変換しない |
| `result` | 実保存物の識別・形式版・サイズ・照合結果と、既存の[バックアップ状態](../BASIC_SPEC.md#バックアップ状態)への対応 |

採用案の流れは、対象・保存先確認、アプリ固有の整合性確保、取得、保存物の照合、隔離先への復元確認、有効性判定、結果確定とする。取得の成功、ハッシュ一致、復元可能性を別の結果にする。保存と復元各操作の上限は3,600秒、再試行0回を候補とする。

暗号形式、鍵の保全、保存先の対応方式、保持・削除、復元先の隔離、定期確認の期限、アプリ別の検証方式は`D04-02`の未達。仮の汎用アーカイブだけでデータベースやマスター管理情報の復元可能性を保証しない。`require_backup_before_update=false`の適用条件も[担当ポリシー](../POLICIES.md#バックアップ設定の未確定条件)の決定待ちとし、本書で免除案を追加しない。

バックアップの暗号境界は、第一候補を個別採用するOS暗号実行物、第二候補を検証可能な内製形式とする比較案。鍵生成・暗号文形式・暗号実行物の版・復号/破損検出・喪失時の復旧可否を同時に決定し、署名検証器の採用を暗号化許可へ流用しない。復元確認では同じ保全物/方法/データ版/隔離先とTTLの対応を固定し、一つでも変われば過去のverifiedを再利用しない。保持期限は復元確認の有効期限と別の要求であり、TTL到達による保全物削除は行わない。未選定の暗号を平文へ代替する案はない。

## P04-APPLY

### 限定資源の適用案

1. 排他と操作開始記録を確定し、対象・計画の前提を再検査する。
2. 変更する資源だけについて変更前内容・属性・所有範囲と復元元を保全する。
3. P03-PLANの依存順序で適用する。ファイル置換と属性変更の成功を分け、変化が不要な属性は書き換えない。
4. 対象資源を再観測して期待結果と照合し、資源別の位置・結果を記録する。
5. 途中失敗では依存する後続を停止し、適用済み・未適用・確認不能を照合して復旧計画を返す。

1資源の上限は30秒、限定資源全体の上限は300秒、再試行なしを候補とする。復元は資源の逆依存順を候補とするが、他機能の変更と競合する場合は自動復元しない。既存ファイルの内容・mode・所有権すべてを復元できることを先に確認する。

資源別記録の採用案は、親のplan_id・operation_id、resource_id、処理位置、変更前後の識別情報、保全元、再観測結果を持つ別レコード。既存の本体専用トランザクションへ混在させない。具体的な配置・版・同期順序は`D04-04`の未決事項とする。

### パッケージ・systemd拡張案

限定資源の受入後に、パッケージ名・固定版・所有範囲・期待導入状態と、unit名・有効化・稼働状態・再読込条件の資源契約を追加する。実コマンドの引数・例外ID、他のパッケージ管理との競合、サービス操作中の失敗、無条件ダウングレードをしない復旧範囲は`D04-04`で確定する。ファイル資源の成功を拡張資源の試験結果へ流用しない。

拡張資源の候補型PackageResourceはresource_id・owner・package_name・exact_version・adoption・depends_on、UnitResourceはresource_id・owner・unit_name・definition_ref・enabled・active・reload_required・adoption・depends_onを全キーとする。共通資源ID/所有/取り込み/依存はP03を再利用し、enabled/active/reload_requiredはbool。名前・版・定義は採用OS側契約で検証し、任意コマンドに展開しない。拡張を採用するまでP03のfile/directory集合へ混入させず、拡張の版・PlanStep payload・query/apply/verify/restoreの全argvと終了分類を揃えてから追加する。推奨は資源別adapterで、ファイル適用器の別名として実装しない。

## 内製単位・公開API・差し替え

配置候補は`src/core/self_manager/`・`lifecycle/`・`backup/`・`iac/apply.rs`。新設する判断・実行調整は内製、OS機能の呼出しは個別例外へ接続する。暗号・TLS・圧縮を自動的に新規内製対象へ取り込まず、難易度・独立検証の成立性を審査する。

| 宣言案 | 所有権・結果・副作用 |
|---|---|
| `prepare_self(input: &Value, target: &TargetEvidence) -> Result<Value, ContractError>` | 固定候補と検査結果を所有して返す。planだけの取得制約を維持 |
| 適用・保全・復元・資源適用 | 宣言は[型付きの返却契約](#型とapiの詳細案)へ集約。本体だけの変更、指定した保存先、OS再導入の分離、親による記録確定を維持 |

Valueの各リクエスト・結果の全フィールドは各契約の未決事項に含める。初期に独立crateは作らない。将来の抽出・差し替えは、同じA04ケースで新旧の出力・副作用・復旧可能範囲を照合し、呼出元切替・互換移行・旧実装除去を別途承認対象にする。今回は旧コードも除去対象も存在しない。

## データ契約の具体化案

`P04-SELF`・`P04-UPDATE`の準備結果と、`P04-BACKUP`・`P04-APPLY`の操作結果を次の候補へ具体化する。共通形式は[P02-MODEL](PHASE_02.md#共通参照型の候補)、資源は[P03-RESOURCE](PHASE_03.md#データ契約の具体化案)を再利用する。

| 型 | 全キー・値域の候補 |
|---|---|
| PreparedSelf | target: TargetEvidence、plan_id: PlanId、action: install・update・uninstall・recover、release_version: string、binary_sha256: stringまたはnull、metadata_sha256: string、source: stringまたはnull、checks: Check[] |
| StepReceipt | step_id: string、contract: ContractRef、phase: not_started・started・applied・verified・failed・unknown、before_ref・after_ref・backup_ref: stringまたはnull、checks: Check[]、error: Diagnosticまたはnull |
| OperationReceipt | operation_id: OperationId、plan_id: PlanId、target: TargetEvidence、steps: StepReceipt[]、recovery: not_needed・available・unavailable・unknown、record: RecordReceipt、checks: Check[] |
| BackupReceipt | backup_id: string、operation_id: OperationId、target: TargetEvidence、profile: 既存構成、data_kind: configuration・application・database・master、location: string、bytes: 0以上integer、sha256: string、restore: RestoreEvidence |
| RestoreEvidence | method_ref: ContractRef、artifact_sha256: string、target_ref: string、verified_at・valid_until: 時刻string、checks: Check[]、verified: boolean |

ハッシュは[既存マニフェスト](../BASIC_SPEC.md#更新マニフェスト)の実値制約へ参照する。PreparedSelfのuninstallだけbinary_sha256・sourceをnullとし、それ以外は対象候補の識別を必須とする。recoverの正常版の選定条件は本体復旧の正本を維持する。準備段階のchecksは本体未取得の検査をpassへしない。互換性に使うリビジョンは適用済み記録から取得し、本体の収録値で置換しない。

StepReceiptは親計画の各stepと1対1に対応し、未開始も省略しない。started後に結果を確認できなければunknownとし、not_startedへ戻さない。検証済みの前提が変わったら、そのstepだけでなく依存する後続の実行可否を再判定する。OperationReceiptは全必須stepがverifiedかつRecordReceiptがcompleteの場合にだけ親が成功を返せる根拠となる。復旧操作が成功しても元stepのfailedは保持する。

BackupReceiptは取得・保存・復元確認の参照を結合する候補で、既存のバックアップ成功状態の代替ではない。verified=trueは同じartifact_sha256の保全物を所定の隔離先へ復元し、方法が要求する全必須検査に成功した場合だけ。`verified_at <= now < valid_until`で判定し、時刻不明・逆転・方法/版/対象不一致は無効とする候補。具体的な有効期間と対象別の検査方法は`D04-02`で決め、仮の共通期限を作らない。

適用関数がエラーを返す場合も、最後に確認したStepReceiptと保存済み記録への参照を親へ保持させる契約が必要である。汎用ContractErrorだけで実適用位置を失う経路は未充足とし、エラーと部分結果を結合するRust型・永続化の全形式は`D04-01`〜`D04-04`の採用審査に残す。新しいファイルパスを復旧時の任意書込み先として受理する案にはしない。

## 接続と責務の照合

PreparedSelf・StepReceiptは[P03-PLAN](PHASE_03.md#データ契約の具体化案)の対象・plan_id・契約参照へ結合する。self単独は構成の適用・migration変更を含めず、通常update内のselfは親operation_idとロックを使用する。署名・メタデータ・実配置の各照合が異なる段階で失敗した場合、到達していないstepはnot_startedとして保持する。

工程上限の最大値の合計と、実際の見積時間・親期限を区別する。全stepに個別最大値をそのまま消費させず、実行時は`min(個別上限, 親の残期限)`で制御する候補。開始前に必要な処理時間・保全・停止予算を確保できる根拠がなければblockedとし、既定の上限値だけで実行可能性を認定しない。見積方法と停止の実現性は各未決事項に残す。

BackupReceipt.restoreは[P06-DATA](PHASE_06.md#p06-data)・[P08-BACKUP](PHASE_08.md#p08-backup)の対象固有検証へ結び付ける。同じパスでも対象・形式・アプリ版が違えば再利用しない。CLIの成功表示・SSHのresult_ready・媒体のtarget_verifiedはいずれも、対象側OperationReceiptと必須記録の一致を必要とする。出力送達失敗は適用済み状態を巻き戻す根拠にしない。

## 受入ケース案

| ケースID | 区分・入力 | 期待する結果・検証 |
|---|---|---|
| `A04-01` | 正常。同一版・同一ハッシュの再導入 | 既存の冪等条件を満たし、設定再生成なし |
| `A04-02` | 異常。署名・ハッシュ不正、CPU・互換範囲不一致 | 本体起動・配置前に停止。各失敗を区別 |
| `A04-03` | 異常。置換直前・rename直後・記録直前の中断 | 実ハッシュと記録から位置照合。無条件再実行なし |
| `A04-04` | 異常。配置後診断失敗、旧版が設定非互換 | 許可範囲の復旧だけ。互換でない退避本体へ戻さない |
| `A04-05` | 正常。独立バックアップと更新内保全 | 目的・親操作を区別し、独立要求で更新を開始しない |
| `A04-06` | 異常。保存先未選択・容量不足・復元期限切れ | 必須保護条件に照合して停止。verifiedを偽装しない |
| `A04-07` | 正常・境界。資源一致・変更・時間上限 | 一致資源へ不要な書込みなし。上限超過は失敗 |
| `A04-08` | 異常。適用途中失敗、記録失敗、復元不能 | 資源ごとの位置・保全範囲を返し、全体成功にしない |
| `A04-09` | 異常。削除対象が別ファイルへ交換、稼働unitあり | 本体削除を停止し、保護対象は保持 |
| `A04-10` | 異常。OSパッケージ更新失敗、復元元破損 | 自動ダウングレード・無検証復元なし。後続停止 |

## 追加の受入ケース案

`P04-SELF`・`P04-BACKUP`・`P04-APPLY`の中断位置と証拠の結合を検証する設計。全ケースで対象・計画・操作IDを固定し、対象外の資源が不変であることも照合する。

| ケースID | 区分・前提と入力 | 期待結果・確認方法 |
|---|---|---|
| `A04-11` | 異常。署名は成功、metadataハッシュだけ不一致 | 候補起動前停止。署名成功で後段検査を省略しない |
| `A04-12` | 異常。step=appliedの直後に記録I/O失敗 | 適用位置を保持し、親成功なし。ContractErrorだけで部分結果を失わない |
| `A04-13` | 境界。restoreのnowがvalid_until直前・同時、artifactハッシュ変更 | 直前かつ同一物だけ有効候補。同時・別物は更新前提にせず停止 |
| `A04-14` | 異常。復旧処理は成功、元の更新stepはfailed | 元結果を成功へ変更しない。復旧結果を別に記録 |
| `A04-15` | 境界。個別上限より親残期限が短い、停止予算を確保不能 | 親期限を超えて実行しない。開始不可の理由と未実施stepを保持 |

## 型とAPIの詳細案

`P04-SELF`・`P04-UPDATE`・`P04-APPLY`の開始後失敗は、`P02-MODEL`のChangeResultで部分適用位置を必ず返す候補。StepReceipt・OperationReceipt・PreparedSelf・BackupReceiptの外部項目は[データ契約](#データ契約の具体化案)を参照し、同名の内部structはその検証済み値だけを保持する。

```rust
pub enum StepPhase { NotStarted, Started, Applied, Verified, Failed, Unknown }
pub enum RecoveryAvailability { NotNeeded, Available, Unavailable, Unknown }
pub struct BackupRequest {
    target: TargetEvidence, profile: Profile, data_kind: String,
    destination: BackupDestination, method: ContractRef,
    backup_id: OperationId, entries: Vec<BackupEntry>, restore_ttl_seconds: u32,
}
pub struct BackupDestination {
    device_id: String, filesystem_id: String, path: std::path::PathBuf,
    expected_available_bytes: u64, artifact_name: String,
}
pub struct RestoreRequest {
    source: BackupReceipt, destination: TargetEvidence, method: ContractRef,
}
pub struct BackupProgress {
    operation_id: OperationId, stage: String, temporary_evidence: Vec<String>,
    receipt: Option<BackupReceipt>,
}
pub fn apply_self(ctx: &mut ExecutionContext, prepared: &PreparedSelf)
    -> ChangeResult<OperationReceipt>;
pub fn backup(ctx: &mut ExecutionContext, request: &BackupRequest)
    -> ChangeResult<BackupProgress>;
pub fn restore(ctx: &mut ExecutionContext, request: &RestoreRequest)
    -> ChangeResult<OperationReceipt>;
pub fn apply_resources(ctx: &mut ExecutionContext, plan: &Plan)
    -> ChangeResult<OperationReceipt>;
```

BackupRequest.profileは構成選択済みのデータ保全だけを対象とする候補。本体専用の退避は既存selfトランザクションの責務とし、架空のprofileでBackupReceiptを生成しない。BackupDestinationは保存先の識別済み値であり、そのパスを復元時の書込み先へ無条件転用しない。BackupProgressは途中位置を持ち、正式なBackupReceiptは全保存・復元確認の成功後だけSomeとする。stageの遷移と一時物の取扱いは中断契約の採用対象として具体化し、未完了の保存物を正常なreceiptへ変換しない。

## 状態と中断の詳細案

BackupProgress.stageの候補全集合はpreparing・capturing・saved・restore_checked・complete・failed・unknown。receiptはcompleteだけでSomeとし、途中失敗では一時物の参照・確認済み位置とerrorを保持する。savedはハッシュ一致までで復元可能性を意味しない。取得対象の整合性確保に失敗したらcapturingへ進まない。

| 中断・失敗位置 | 停止と復旧候補 |
|---|---|
| 容量・対象・保存先の再照合前 | 対象変更なし。入力・計画を再確認 |
| capturing中、転送切断・容量不足 | 不完全物を正式名・正常receiptにしない。再取得の自動開始なし |
| saved後、復元確認の失敗 | 保存物の事実は保持、verified=false。更新前提は不成立 |
| アプリ停止後に保全失敗 | 許可済みの元稼働状態への復帰を試み、元の保全失敗と復帰結果を別に残す |
| self preparedで配置だけが候補へ変化 | 実ハッシュで位置を照合し、正本の本体復旧へ。元本体への無条件上書きなし |
| 資源applied後、再観測・記録失敗 | 当該資源と依存後続を停止。部分結果をChangeFailureへ保持 |
| 復旧も失敗・不明 | unavailable/unknownを保持。新しい通常更新へ進まない |

資源復元の候補は、元計画で許可された保全元と同一対象だけを逆依存順に処理する。作成したdirectoryへ他者のファイルが増えた場合は再帰削除せず復旧不能範囲として止める。復旧成功でも元操作のfailedをsuccessへ変更しない。バックアップの暗号形式と不完全物の保存・除去条件が未確定な間は、cleanupを理由にデータを自動削除しない。

## 依存と互換性の詳細案

取得・検証の第一候補を次へ具体化する。個別実行物の固定版・全引数・検証・採用承認は[台帳の審査候補](../DEPENDENCY_EXCEPTIONS.md#追加の個別審査候補)で扱い、現在の使用可能判定を変更しない。

| 責務 | 採用審査候補と境界 |
|---|---|
| HTTPS取得 | OS提供curl等の取得実行物へ委譲。TLS・redirect・取得先・上限・全引数を固定し、任意オプションを受けない |
| SHA-256 | 既存OS検証コマンドの採用情報へ接続。期待値は署名済みmanifestから取得し、同じ未検証元の自己申告だけで判定しない |
| OpenPGP検証 | 既存gpgvの契約を使用。秘密鍵・署名生成を取得処理へ持たせない |
| アーカイブ・暗号化 | 既存OSツール利用案と専用内製形式案を別に比較。暗号を未評価で新規内製化しない |
| package/systemd | EX-001の個別コマンド契約。パッケージ管理競合・unit状態と副作用を実行前後に照合 |

更新の互換表候補は、現在のconfig/migration各revisionと選択構成、候補の対応範囲、必要な移行契約ID、移行後revision、元本体/データへ戻せる範囲を1行として扱う。行がない組合せは停止し、番号差だけで移行可能としない。self単独は移行行を実行せず、現在revisionが互換範囲内の場合だけ本体を変更する。

初回は旧実装がないため新旧比較なし。将来の分離・差し替えは取得→準備→実適用→検証→記録の境界を維持し、各障害位置のChangeResultと永続証拠を同じテストで比較する。旧コード除去は全呼出元の切替・依存検査・復旧経路確認の後だけとし、暗黙のFFI・外部crate追加を行わない。

## 受入条件の充足照合案

SELFは既存置換/復旧とPreparedSelf、UPDATEは層別工程と互換表、BACKUPはBackupProgress/Receipt、APPLYは資源別StepReceiptへ対応する。共通のChangeFailureは開始後の適用済み位置を保持する。

| ケースID | 区分・固定入力 | 期待結果と確認 |
|---|---|---|
| `A04-16` | 異常。capturing中に保存容量不足、正式receiptなし | partial=Some(BackupProgress)、receipt=None。不完全物の正式昇格・無確認除去なし |
| `A04-17` | 異常。復旧対象directoryに新しい未管理ファイルが存在 | 再帰削除0回。復旧不能範囲を保持し、元操作も失敗のまま |
| `A04-18` | 異常。selfは互換範囲外、構成移行行だけが存在 | self単独では移行せず停止。確認済み通常updateへの判断が必要 |

設定値の未確定動作は[担当ポリシーの審査候補](../POLICIES.md#完全仕様化に伴うバックアップ設定の審査候補)へ集約する。暗号・鍵・保存/削除・復元期限・アプリ別形式を決めずに取得成功だけを完全仕様成立としない。対象外である本体専用退避へ架空のprofileを追加せず、既存self保全と構成データの独立バックアップを区別する。

## 入力と保存形式の詳細案

P04-BACKUPの保存単位をBackupSetとする候補。既存BackupReceiptは検証結果として維持し、保存対象の定義と混同しない。形式・鍵の審査は[セキュリティ審査候補](../SECURITY_POLICY.md#完全仕様化に伴う審査候補)へ接続する。

| 型 | 全キーと制約 |
|---|---|
| BackupSet | schema_version: 1、backup_id: OperationId、source: TargetEvidence、profile: 構成enum、data_kind: 既存enum、entries: BackupEntry[]、method: ContractRef、destination: BackupDestination、restore_ttl_seconds: 1以上u32 |
| BackupEntry | entry_id: ASCII資材ID、relative_path: string、kind: file/directory、bytes: u64、sha256: hex64またはdirectory時null、mode: P03属性、uid/gid: 同属性。symlink・特殊file・絶対path・親逸脱・重複を拒否 |
| BackupDestination | device_id・filesystem_id・path: string、expected_available_bytes: u64、artifact_name: string。pathは選択した保存directory、名前はbackup_idに結合、既存同名への上書きなし |
| ResourceJournal | schema_version: 1、operation_id: OperationId、plan_id: PlanId、target: TargetEvidence、entries: ResourceJournalEntry[]。配置候補は操作ID別のiac-journal.json |
| ResourceJournalEntry | resource_id: string、sequence: u32、phase: StepReceiptのphase、before: ResourceObservation、expected: Resource、backup_ref: stringまたはnull、after: ResourceObservationまたはnull、error: Diagnosticまたはnull |

BackupRequestは検証済みBackupSetのsourceをtargetへ対応付け、残る同名の値をそのまま所有する内部型とする。BackupEntryは本表の検証済み各値を非公開フィールドに持つstruct。pathの外部表現はUTF-8絶対パス、内部はPathBufとし、保存先の同一性を文字列だけで認定しない。

restore_ttl_secondsはデータ方法側の必須入力とし、共通の推測既定値を作らない。valid_untilは確認時刻にTTLをchecked加算して求め、overflow・時計後退・方法の変更は無効。候補保存先は利用者が選んだローカルのmount済みFSだけ。ネットワーク転送・暗号方式の未選定を、平文保存や任意tar実行の許可にしない。

互換表は旧二択形式→三構成形式を自動変換不可、三構成形式→同じ採用版を検証後のみ可、未知revisionを不可とする候補。server/miningからmasterへの変更も単なるprofile書換えでは不可。対象revisionの実値・移行方法・切戻し可能範囲を所有者が決定するまで更新計画をblockedにする。

| ケースID | 固定入力 | 期待結果 |
|---|---|---|
| `A04-19` | BackupEntryに../経路、TTL加算overflow、旧二択からmasterへの無定義移行 | 保存・更新開始前に拒否。既存データと構成記録を変更しない |

## 実行と復旧の確定候補

P04-APPLYのjournal確定単位を一資源一工程とする候補。各変更前にbefore/backup_refとstartedを同期し、変更後に実観測を保存しapplied、全期待属性一致後にverifiedへ進める。startedの保存失敗なら変更しない。適用後の保存失敗はunknownとして最後の保存済み根拠と実適用位置をChangeFailureへ返す。

| 資源 | 適用直後に照合する事実 | 復旧の限界 |
|---|---|---|
| file | 内容hash/長さ、mode/uid/gid、対象rootとentryの同一性 | 元内容・属性と復元先が一致する場合だけ計画。新しい第三者変更は上書きしない |
| directory | 種別、所有属性、所有範囲 | 新規作成でも未管理の子があれば自動削除しない |
| package | 固定版と管理DBの実状態、途中設定の有無 | package操作の非ゼロを未変更にせず、自動再投入・ダウングレードなし |
| systemd | 定義hash、load/enabled/activeの各実状態、対象unit | 起動要求成功と稼働確認は別。未確認状態で次のサービス変更なし |

バックアップはcapturing中の成果物を正式名にしない。保存・hash確認・対象固有の隔離復元・必須記録の確定を終えた同一物だけcompleteへ進める。取得のため停止したサービスを復帰させる場合は、その事前計画・元稼働状態・復帰結果を記録し、バックアップ失敗は維持する。保存失敗の後処理として未確認物を無断で消去しない。

通常更新と復旧は同じ親guard内の別stepとし、元失敗を上書きしない。再起動が必要でも必須記録未確定なら成功の終了10へ変換せず、未完了位置を報告する。消去を伴うOS再導入はこの復旧の自動分岐へ含めない。

| ケースID | 固定入力 | 期待結果 |
|---|---|---|
| `A04-20` | file置換成功後journal同期失敗、packageは途中設定、サービス復帰だけ成功 | 元操作は失敗/照合待ち。各実位置を保持し、再適用・全体成功なし |

## 依存と受入の確定候補

P04-SELFの取得adapterは取得・hash・署名の三責務に分ける候補。判断器はPreparedSelfと検証済みmetadataだけを受け、外部実行物の生出力を適用指示として評価しない。各実行物は例外台帳の個別採用前に使用不可。

| 外部境界候補 | 固定する入力/上限と成功条件 |
|---|---|
| HTTPS取得 | curl比較候補。最初の引数--disable、--globoff、--proto =https、--connect-timeout 30、--max-timeは担当期限、--retry 0、--silent、--show-errorをargvで渡す。proxy等の継承を除きTLS検証を維持。初期案はredirect非追従、別URLは固定候補の再審査が必要 |
| 配布物量 | metadata一件1 MiB、本体128 MiB、ISO8 GiBを上限候補。manifest記載sizeとも一致必須。大容量はP02専用file-sinkとOS強制容量制限の採用後だけ使用 |
| hash | OSの採用済みSHA-256実行物へ固定pathを一引数で渡す。出力はhex64一値と対象対応を検証。自己申告版・ファイル名をhashの代わりにしない |
| 署名 | gpgv等の採用実行物、固定keyring、detached署名と署名対象byte列の三参照を対応付ける。終了0・鍵適用範囲・hash群を別検査。鍵条件はSECURITY_POLICYに集約 |
| アーカイブ/暗号 | 暗号化バックアップの形式・鍵参照・復元検査は未採用。tar.gzを暗号形式や任意抽出許可に流用しない |

[curl公式資料](https://curl.se/docs/manpage.html)は設定読込みの無効化とprotocol制限の意味の確認に使用した。上限値と非追従は製品の候補であり、Ubuntu採用版でのオプション・file-sink・GitHub配布先への適用可能性は未検証。取得先がredirect必須なら無確認追従せずD04-01へ戻す。

lifecycle/backup/applyの新旧差し替えは同じPlan・固定資材・故障注入点でStepReceipt/partial/実書込み集合を比較し、旧コード除去前に復元可能範囲を再確認する。暗号等を高難易度のまま未採用とすることと、平文へ代替することを区別する。

| ケースID | 固定入力 | 期待結果 |
|---|---|---|
| `A04-21` | 取得先302、宣言size一致だが実byte+1、署名成功だが別metadata | 固定取得条件/実量/内容照合で停止。候補起動・配置・自動追従なし |
| `A04-22` | 正しいmanifest、本体の実量は宣言内、file-sink同期失敗またはHTTPエラー本文 | synced/passを返さず起動0回。成功hashだけで取得完了にしない |
| `A04-23` | 同一保全物の暗号方式/方法版だけ変更、復元TTL到達 | 旧verifiedを使用せず重要更新停止。期限切れだけを理由に保全物を削除しない |
| `A04-24` | 同一sourceに一致する移行行が二件、reversible=trueだが復元方法null | 変更前blocked。先勝ちの行選択・旧版への無条件切戻しなし |
| `A04-25` | file/directory版へPackageResource、unit起動要求0だがactive=false、package途中設定 | 版不一致または検証失敗。資源拡張の黙認・要求成功だけのverified化なし |

## 未決事項と成立判定

| 判断ID | 残る具体化・採用判断 | 未達の影響 |
|---|---|---|
| `D04-01` | metadataメモリー/本体専用file-sinkとFetchResultを推奨。所有者は取得実行物・全argv・容量/停止の成立性・redirectが必要な公式取得先の扱いを決定。現行の非追従候補でGitHub取得成功を保証しない | 本体の準備・適用 |
| `D04-02` | OS暗号実行物を第一、内製形式を第二候補として比較。所有者は暗号/鍵の担当ポリシー、保存形式、対象別隔離復元/TTL、保持・除去の条件を一括で決定する。候補作成は鍵運用/平文保存の承認ではない | 全バックアップ・復元・重要更新 |
| `D04-03` | MigrationRuleの一意照合方式を推奨。所有者は実revision・対応する対象形式・不可逆性/復元方法を決定する。旧二択/構成変更を通常の同版行に紛れ込ませない | 更新・本体互換性 |
| `D04-04` | 種別adapter・資源journalと拡張型を推奨。所有者は拡張形式版、全外部argv/例外、同期と復旧不能範囲を決定する。root固定D03-02の成立前にfile適用成功を認定しない | IaC適用と構成機能 |

共通契約の未承認、セキュリティ設定の未確定条件、外部依存の採用情報、非機能の定量条件も残る。これらに依存する契約は完全仕様未成立とし、文書案の検査を製品受入へ転用しない。
