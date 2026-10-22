# Phase 4 完全仕様案: 導入・更新・保全・復旧とIaC適用

[文書一覧](../DOCUMENT_INDEX.md) | [担当工程](../ROADMAP.md#phase-4-本体導入更新バックアップ復旧とiac適用)

採用審査用の`B11`。契約採用・仕様充足の承認はなく、[案の適用範囲](PHASE_01.md#案の適用範囲)を適用する。対象要件は`REQ-004`・`REQ-006`・`REQ-007`・`REQ-008`・`REQ-009`・`REQ-010`・`REQ-011`・`REQ-012`。

## 契約と依存

| 契約ID | 担当する具体化 |
|---|---|
| `P04-SELF` | 本体取得・互換性・配置・削除・本体専用の復旧 |
| `P04-UPDATE` | OS・構成・アプリと本体の更新・移行の統合 |
| `P04-BACKUP` | 独立バックアップ・復元と更新前保全の共通手段 |
| `P04-APPLY` | IaC資源の適用・検証・中断と復旧 |

共通参照（全フェーズB11）は`P02-MODEL`・[型](PHASE_02.md#p02-model)、`P02-PARSE`・[形式](PHASE_02.md#p02-parse)、`P02-STORE`・[記録と排他](PHASE_02.md#p02-store)、`P02-EXEC`・[実行制御](PHASE_02.md#p02-exec)、`P03-RESOURCE`・[資源](PHASE_03.md#p03-resource)、`P03-FS`・[観測](PHASE_03.md#p03-fs)、`P03-PLAN`・[計画](PHASE_03.md#p03-plan)。共通CLIは`P02-CLI`、秘密情報・真正性・権限は[セキュリティポリシー](../SECURITY_POLICY.md)を参照する。

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

FetchResultは取得成功時だけ返す候補とし、sha256は実保存byte列の照合値、expected_bytesは信頼を確認した入力の宣言量、observed_bytesは実受信量で両者の完全一致を要求する。source_refは固定した取得要求ID、destination_refは排他的に作成した保存物IDで、任意URL/pathを事後評価しない。API候補`fetch_artifact(request: &FetchRequest, ctx: &ExecutionContext) -> ChangeResult<FetchProgress>`のFetchProgressはstage（prepared/receiving/verified/failed/unknown）、observed_bytes、temporary_ref、resultを全項目とし、resultはverifiedだけSome(FetchResult)、temporary_refは未作成時だけNone。部分受信・同期失敗・hash不一致はSome(partial)に保存位置を保持し、成功hashを推測して埋めない。メタデータの読取り取得は別のReadOnly文脈でメモリーだけへ受ける。

## P04-UPDATE

既存の[更新順序](../BASIC_SPEC.md#更新処理)に、変更する層・対象構成・成果物・保全・復元確認・再起動を対応付ける。採用案では各変更ステップについて、開始条件、変更資源、期待結果、検証、復旧可否を計画へ保持する。

OSパッケージ操作は1回3,600秒以内、サービス操作は1回120秒以内、総計画は変更開始から3,600秒以内を候補とする。総期限を超える工程を含む計画は開始前に実行不能として返す。パッケージ操作の失敗を同じ入力で自動再投入せず、実状態とパッケージ管理の状態を照合して復旧計画へ移る。

本体変更は既存仕様どおり最後の変更ステップとし、構成の適用済みrevisionと本体収録revisionを区別する。旧二択形式と三構成の移行・不可逆なアプリ更新は、互換範囲と復元検証が確定するまで開始不可。具体的な移行表は`D04-03`に残す。

移行の審査入力候補MigrationRuleはsource_profile・target_profile・source_config_revision・target_config_revision・source_migration_revision・target_migration_revision・method・reversible・restore_methodを全キーとする。構成は既存enum、revisionはP02共通範囲、methodはContractRef、reversibleはbool、restore_methodだけnull可。reversible=trueなら復元方法と新旧双方の読取り互換が必須。適用済み版と一致する行が0件または複数ならblocked。同じ採用版への無変更行とデータ移行行を区別し、版の大小から変換を推測しない。具体的revision値は所有者の別採用を必要とする。

MigrationRuleの候補照合はsourceの三値（構成/config/migration）と計画で固定したtargetの三値の完全一致とする。通常updateではsource_profile=target_profileが必須で、構成変更の行を選択しない。self単独では行を実行せず現在revisionの互換範囲だけを判定する。無変更行もmethodが指定する非変更検証を通す。不可逆行は保全・復元可能な範囲と不可逆な損失を計画へ明示し、reversible=falseを保護免除にしない。開始後の失敗は移行後の実形式・revision・step位置を記録し、設定の数値だけを旧値へ戻さない。

```rust
pub enum UpdateLayer { Os, Configuration, Application, Miner, Body }
pub enum UpdateMaterial { External(MaterialRef), Owned(ReleaseArtifactRef) }
pub struct UpdatePayload {
    layers: Vec<UpdateLayer>, materials: Vec<UpdateMaterial>,
    migration: Option<MigrationRule>, backup_id: Option<OperationId>,
}
pub struct MigrationRule {
    source_profile: Profile, target_profile: Profile,
    source_config_revision: u32, target_config_revision: u32,
    source_migration_revision: u32, target_migration_revision: u32,
    method: ContractRef, reversible: bool, restore_method: Option<ContractRef>,
}
```

UpdatePayloadは子stepの索引であり、それ自身から同じupdateを再帰実行しない。layersは上記宣言順の一意集合、materialsは参照種別と固定識別子で一意とし、消去/OS再導入を含めない。bodyだけのself操作は本payloadでなくPreparedSelfを使う。通常update内のbody子stepは最後の変更stepだけに対応させる。migration=Noneはrevision不変のときだけ、backup_id=Noneは該当層の保全が明示非適用である根拠があるときだけ許可し、未取得の保全をNoneにしない。

| ケースID | 固定入力 | 期待結果 |
|---|---|---|
| `A04-44` | revision変更でmigration=None、body後にapp変更、updateの自己呼出し | UPDATE_BINDINGで変更前拒否。独立selfとの混同・保全未取得の非適用化・再帰実行なし |

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

属性適用の候補順は内容の準備→uid/gid→mode→内容/全属性の再観測→file同期→確定/親同期。[Rustのchown](https://doc.rust-lang.org/std/os/unix/fs/fn.chown.html)では所有者変更がsetuid/setgid等へ影響し得るため、modeを先に確定したまま成功としない。[fchown](https://doc.rust-lang.org/std/os/unix/fs/fn.fchown.html)とFileの権限設定で、採用済みRootBindingが固定した同じ実体へ適用する。属性だけを変える通常fileでも複数hard linkが判明した場合は共有inodeを直接変更せず、別entryへの影響を解消する計画がなければAPPLY_SHAREDで停止する。ACL/capability等の未管理属性を保持/復元できる根拠がなければ置換を開始しない。標準APIの存在でroot固定D03-02を解消したとはしない。

資源別記録の採用案は、親のplan_id・operation_id、resource_id、処理位置、変更前後の識別情報、保全元、再観測結果を持つ別レコード。既存の本体専用トランザクションへ混在させない。具体的な配置・版・同期順序は`D04-04`の未決事項とする。

### パッケージ・systemd拡張案

限定資源の受入後に、パッケージ名・固定版・所有範囲・期待導入状態と、unit名・有効化・稼働状態・再読込条件の資源契約を追加する。実コマンドの引数・例外ID、他のパッケージ管理との競合、サービス操作中の失敗、無条件ダウングレードをしない復旧範囲は`D04-04`で確定する。ファイル資源の成功を拡張資源の試験結果へ流用しない。

拡張資源の候補型PackageResourceはresource_id・owner・package_name・exact_version・adoption・depends_on、UnitResourceはresource_id・owner・unit_name・definition_ref・enabled・active・reload_required・adoption・depends_onを全キーとする。共通資源ID/所有/取り込み/依存はP03を再利用し、enabled/active/reload_requiredはbool。名前・版・定義は採用OS側契約で検証し、任意コマンドに展開しない。拡張を採用するまでP03のfile/directory集合へ混入させず、拡張の版・PlanStep payload・query/apply/verify/restoreの全argvと終了分類を揃えてから追加する。推奨は資源別adapterで、ファイル適用器の別名として実装しない。

## 内製単位・公開API・差し替え

配置候補は`src/core/self_manager/`・`lifecycle/`・`backup/`・`iac/apply.rs`。新設する判断・実行調整は内製、OS機能の呼出しは個別例外へ接続する。暗号・TLS・圧縮を自動的に新規内製対象へ取り込まず、難易度・独立検証の成立性を審査する。

| 宣言案 | 所有権・結果・副作用 |
|---|---|
| `prepare_self(input: &Invocation, target: &TargetEvidence, plan_id: &PlanId) -> Result<PreparedSelf, ContractError>` | 先に生成した親計画IDへ束縛し、固定候補と検査結果を所有して返す。plan時の取得制約を維持 |
| 適用・保全・復元・資源適用 | 宣言は[型付きの返却契約](#型とapiの詳細案)へ集約。本体だけの変更、指定した保存先、OS再導入の分離、親による記録確定を維持 |

Invocation/PreparedSelfと各型付きAPIだけを内部契約とし、未検証Valueを適用器へ渡さない。各契約の未決事項は型・値・外部方式の採用へ対応付ける。初期に独立crateは作らない。将来は同じA04ケースで新旧の出力・副作用・復旧可能範囲を照合し、呼出元切替・互換移行・旧実装除去を別途承認する。今回は旧コード・除去対象なし。

## データ契約の具体化案

PreparedSelfを作る前に親PlanIdを一度だけ生成する候補。prepare_selfはそのIDを借用して返却値へ保持し、IDの新規生成・空値補完・別計画の流用をしない。対象証拠と親IDを確定して準備→P03のPlanDraft構築/封印→利用者確認→適用、の順で、封印済みPlanがなければ準備できない循環を作らない。apply_selfはctxの親ID/対象と再照合し、不一致ならSELF_BINDINGで配置0件。digestの比較用射影はP03、全入力の配送hashはP09の担当である。

| ケースID | 固定入力 | 期待結果 |
|---|---|---|
| `A04-46` | ID Aで準備し親Plan/ctxをBへ変更、別入力は同じAの未封印Draft | 前者は封印/適用前拒否、後者は準備値を含めて封印可能。準備中のID生成・封印済みPlanを要求する循環なし |

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

同じ親操作から呼ぶself/apply/backupと構成別の子処理は、親のoperation_idとguardを使用する候補。子の完了は担当StepReceiptまでで、finishによる履歴/終了記録の確定は全step照合後に親だけが一度行う。子APIが返すOperationReceipt.recordはその時点の親の記録段階であり、completeでないことを子資源の検証失敗と混同しない。逆に子資源verifiedを親全体のsuccessへ変換しない。

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
pub enum SelfAction { Install, Update, Uninstall, Recover }
pub struct PackageResource {
    resource_id: String, owner: String, package_name: String, exact_version: String,
    adoption: Adoption, depends_on: Vec<String>,
}
pub struct UnitResource {
    resource_id: String, owner: String, unit_name: String, definition_ref: ContentRef,
    enabled: bool, active: bool, reload_required: bool,
    adoption: Adoption, depends_on: Vec<String>,
}
pub struct PreparedSelf {
    target: TargetEvidence, plan_id: PlanId, action: SelfAction, release_version: String,
    binary_sha256: Option<String>, metadata_sha256: String, source: Option<String>, checks: Vec<Check>,
}
pub struct StepReceipt {
    step_id: String, contract: ContractRef, phase: StepPhase,
    before_ref: Option<String>, after_ref: Option<String>, backup_ref: Option<String>,
    checks: Vec<Check>, error: Option<Diagnostic>,
}
pub struct OperationReceipt {
    operation_id: OperationId, plan_id: PlanId, target: TargetEvidence,
    steps: Vec<StepReceipt>, recovery: RecoveryAvailability, record: RecordReceipt, checks: Vec<Check>,
}
pub struct RestoreEvidence {
    method_ref: ContractRef, artifact_sha256: String, target_ref: String,
    verified_at: UtcMillis, valid_until: UtcMillis, checks: Vec<Check>, verified: bool,
}
pub struct BackupReceipt {
    backup_id: OperationId, operation_id: OperationId, target: TargetEvidence,
    profile: Profile, data_kind: String, location: String,
    bytes: u64, sha256: String, restore: RestoreEvidence,
}
pub struct BackupRequest {
    target: TargetEvidence, profile: Profile, data_kind: String,
    destination: BackupDestination, method: ContractRef,
    backup_id: OperationId, entries: Vec<BackupEntry>, restore_ttl_seconds: u32,
}
pub struct BackupDestination {
    device_id: String, filesystem_id: String, path: std::path::PathBuf,
    expected_available_bytes: u64, artifact_name: String,
}
pub enum BackupEntryKind { File, Directory }
pub struct BackupEntry {
    entry_id: String, relative_path: String, kind: BackupEntryKind,
    bytes: u64, sha256: Option<String>, mode: u16, uid: u32, gid: u32,
}
pub struct BackupSet {
    schema_version: u32, backup_id: OperationId, source: TargetEvidence,
    profile: Profile, data_kind: String, entries: Vec<BackupEntry>,
    method: ContractRef, destination: BackupDestination, restore_ttl_seconds: u32,
}
pub struct ResourceJournalEntry {
    resource_id: String, step_id: String, sequence: u32, phase: StepPhase,
    before: ResourceObservation, expected: Resource, backup_ref: Option<String>,
    after: Option<ResourceObservation>, error: Option<Diagnostic>,
}
pub struct ResourceJournal {
    schema_version: u32, operation_id: OperationId, plan_id: PlanId,
    target: TargetEvidence, entries: Vec<ResourceJournalEntry>,
}
pub struct RestoreRequest {
    source: BackupReceipt, destination: TargetEvidence, method: ContractRef,
}
pub struct BackupProgress {
    operation_id: OperationId, stage: String, last_confirmed_stage: String,
    temporary_evidence: Vec<String>,
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

BackupReceipt.backup_idはBackupRequest.backup_idと同じOperationId形式で、親operation_idと同一である必要はない。sha256は保存した暗号物、restore.artifact_sha256も同じ暗号物へ一致させる。検証前の平文hashをその欄へ代入しない。子OperationReceipt.recordは[P02のRecordReceipt](PHASE_02.md#記録確定と実行結果)を参照し、親finish前のprepared/history_committedをcompleteへ補完しない。step失敗時はerror必須、verified時はerrorなし・担当必須checks成功が必要。

| ケースID | 固定入力 | 期待結果 |
|---|---|---|
| `A04-43` | 子だけverified、親recordはprepared、保存暗号hashとrestore参照hashが異なる | 親完了不可、BackupReceipt拒否。子の成功と親記録確定・復元対象を混同しない |

## 状態と中断の詳細案

BackupProgress.stageの候補全集合はpreparing・capturing・saved・restore_checked・complete・failed・unknown。receiptはcompleteだけでSomeとし、途中失敗では一時物の参照・確認済み位置とerrorを保持する。savedはハッシュ一致までで復元可能性を意味しない。取得対象の整合性確保に失敗したらcapturingへ進まない。

BackupProgress.last_confirmed_stageはpreparing/capturing/saved/restore_checked/completeのいずれかで、stageが通常段階なら同値、failed/unknownなら最後に証拠を確認した段階を保持する候補。capturingは開始の確証であり全entry取得済みの意味ではない。receipt=Noneのfailedでもsavedまで到達した事実を保持し、失敗した保存物を未作成として再取得しない。stageだけをcompleteへ変更せず、対象固有の復元確認と保存物の検証記録の確定後に正式receiptを結合する。親update内ではこの子のreceiptを親へ返した後、全stepを照合して親がfinishする。子receiptの生成条件に親のfinish完了を要求して循環させず、子のcompleteを親操作の成功とも扱わない。

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
| ResourceJournalEntry | resource_id・step_id: string、sequence: u32、phase: StepReceiptのphase、before: ResourceObservation、expected: Resource、backup_ref: stringまたはnull、after: ResourceObservationまたはnull、error: Diagnosticまたはnull |

BackupRequestは検証済みBackupSetのsourceをtargetへ対応付け、残る同名の値をそのまま所有する内部型とする。BackupEntryは本表の検証済み各値を非公開フィールドに持つstruct。pathの外部表現はUTF-8絶対パス、内部はPathBufとし、保存先の同一性を文字列だけで認定しない。

restore_ttl_secondsはデータ方法側の必須入力とし、共通の推測既定値を作らない。valid_untilはverified_atをUTCミリ秒へ変換し、TTLを1,000倍してchecked加算し、UtcMillisの暦日範囲へ戻して求める候補。秒をmsへ無変換で足さず、演算overflow・表現範囲外・時計後退・方法の変更は無効。更新前判定では同じ保全物・方法・対象版を照合したうえでverified_at <= now < valid_untilを一度の確認時刻へ評価する。nowが確認時刻より前、時刻取得不能、判定から適用までの前提変化は再確認待ち。候補保存先は利用者が選んだローカルのmount済みFSだけ。ネットワーク転送・暗号方式の未選定を、平文保存や任意tar実行の許可にしない。

BackupEntry集合の候補順序はrelative_pathのUTF-8バイト順。各pathは空でない相対成分列で、directory自身のbytes=0・sha256=null、fileは実byte数/hashを必須とする。親directoryを含むことを未列挙の子の取得許可にしない。復元時は全entryと保全物の版/暗号/方法を先に検証し、全pathの出力先を固定してから変更する。復元確認が失敗した保存物は参照と検証失敗をBackupProgressへ残すが、正式BackupReceiptは生成しない。verified=falseの途中結果は成功receiptではなく検査証拠として扱い、更新前提にも保持期限による削除根拠にも使用しない。

BackupEntryの親子検証候補は、pathの成分列を比較して重複を拒否し、列挙したfileの配下へ別entryが存在する場合も拒否する。列挙directory配下は明示したentryだけを取得し、再帰探索で未管理データを追加しない。検査から取得までに種別・同一性・内容が変わったらBACKUP_CHANGED。総量はfileのbytesのchecked合計に形式/暗号/作業領域を加えてFS別に計算し、暗号形式の上限が未確定なら容量検査を成功にしない。

BackupSet→BackupRequestはsourceをtargetへ改名して移す唯一の内部変換とし、schema_versionを検証してから残る値を無変更で移す。target識別やentriesを再収集して別内容へ差し替えない。ResourceJournalEntry.expectedはResource全体、afterは実観測であり、期待値をafterへ複製して適用成功を捏造しない。journalのentriesは資源stepのsequence順、欠損を未開始と推測して新しい空行を自動作成しない。

| ケースID | 固定入力 | 期待結果 |
|---|---|---|
| `A04-45` | BackupSet.sourceと変換後target不一致、directoryにhash、未観測afterへexpectedを複製 | 変換/型/証拠不一致で拒否。別対象保全・暗黙取得・期待値だけの適用成功なし |

保存先名の候補はartifact_nameをbackup_idの表記そのものへ固定し、拡張子・任意の利用者名・path成分を追加しない。形式の識別は保存内容の採用版/方法で行い、名前から暗号方式を推測しない。保存先directoryと取得する全entryについて実device/FSを照合し、sourceの取得範囲内へ解決する保存先、同じ消去予定disk上の保存先、同一性不明をBACKUP_DESTINATIONで拒否する。expected_available_bytesは観測証拠であって実容量の保証ではなく、書込み前の再測定と実書込み時の失敗を検出する。既存同名を新しい保全物で上書きしない。

互換表は旧二択形式→三構成形式を自動変換不可、三構成形式→同じ採用版を検証後のみ可、未知revisionを不可とする候補。server/miningからmasterへの変更も単なるprofile書換えでは不可。対象revisionの実値・移行方法・切戻し可能範囲を所有者が決定するまで更新計画をblockedにする。

| ケースID | 固定入力 | 期待結果 |
|---|---|---|
| `A04-19` | BackupEntryに../経路、TTL加算overflow、旧二択からmasterへの無定義移行 | 保存・更新開始前に拒否。既存データと構成記録を変更しない |

## 実行と復旧の確定候補

P04-APPLYのjournal確定単位を一資源一工程とする候補。各変更前にbefore/backup_refとstartedを同期し、変更後に実観測を保存しapplied、全期待属性一致後にverifiedへ進める。startedの保存失敗なら変更しない。適用後の保存失敗はunknownとして最後の保存済み根拠と実適用位置をChangeFailureへ返す。

journalのsequenceは親Plan.stepsの0始まりindexへ固定する候補。resource_idとstep IDを両方照合し、同一資源のstarted/applied/verifiedは同じ行の段階更新とする。重複行・逆行・別Planのbackup_refを拒否する。復旧は元計画で許可された逆依存順だけを候補とし、実状態が記録したafterに一致し、beforeと保全物が検証可能な場合だけ復元へ進む。after不明・第三者変更・復元元不明はAPPLY_RECONCILEとしてその依存範囲を保留し、再実行や全資源一括巻戻しを行わない。資源復旧の成否を元stepのfailedと別に保持する。

sequenceで引いたPlanStep.idがstep_idと一致し、そのresource_change.resource_id/beforeがjournalのresource_id/beforeとそれぞれ完全一致することを変更前に検証する候補。resource_change.afterはjournal.expected.attributesへ一致させ、expected自体は同じresource_idのPlan.resources一件へ一致させる。属性objectとResource全体を同じ型として比較しない。同一資源の段階更新では三つの識別値を変更しない。非資源stepにはこのjournal行を作らず、担当するStepReceiptへ記録する。空のjournalを全計画未実行の証明にはしない。

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

2026-10-22に[GitHub asset API](https://docs.github.com/en/rest/releases/assets#get-a-release-asset)が200の直接応答と302を返し得ることを確認した。採用審査へ追加する比較案は、curl自体は非追従のまま、一段ずつLocationを検証して再要求する方式。最大3段、HTTPSだけ、明示許可した配布host/portだけ、循環・userinfo・fragment・複数Location・不正URLを拒否し、別originへ認証情報を転送しない。全段を同じ取得期限へ算入し、最終200のbodyだけを固定size/hashへ照合する。これは候補であり、許可先・全argv・header/URL秘匿化・対象版の試験が揃うまで現行の非追従案を切り替えない。無制限の--locationで解消済みとはしない。

lifecycle/backup/applyの新旧差し替えは同じPlan・固定資材・故障注入点でStepReceipt/partial/実書込み集合を比較し、旧コード除去前に復元可能範囲を再確認する。暗号等を高難易度のまま未採用とすることと、平文へ代替することを区別する。

| ケースID | 固定入力 | 期待結果 |
|---|---|---|
| `A04-21` | 取得先302、宣言size一致だが実byte+1、署名成功だが別metadata | 固定取得条件/実量/内容照合で停止。候補起動・配置・自動追従なし |
| `A04-22` | 正しいmanifest、本体の実量は宣言内、file-sink同期失敗またはHTTPエラー本文 | synced/passを返さず起動0回。成功hashだけで取得完了にしない |
| `A04-23` | 同一保全物の暗号方式/方法版だけ変更、復元TTL到達 | 旧verifiedを使用せず重要更新停止。期限切れだけを理由に保全物を削除しない |
| `A04-24` | 同一sourceに一致する移行行が二件、reversible=trueだが復元方法null | 変更前blocked。先勝ちの行選択・旧版への無条件切戻しなし |
| `A04-25` | file/directory版へPackageResource、unit起動要求0だがactive=false、package途中設定 | 版不一致または検証失敗。資源拡張の黙認・要求成功だけのverified化なし |
| `A04-26` | 宣言128byteに対し受信127/129byte、128byteだがsync失敗 | FetchResultなし。FetchProgressに実量と一時物参照を保持し、候補起動・正式昇格0回 |
| `A04-27` | 通常updateにserver→master行のみ存在、移行後データ変更済みだが版記録は旧値 | 前者blocked、後者は実形式と記録の照合待ち。構成変更・版数だけの巻戻しなし |
| `A04-28` | directoryに非zero bytes、未列挙の子、保存済みだが隔離復元失敗 | 形式拒否または対象外不取得。失敗物の証拠は保持するがBackupReceiptなし、重要更新停止 |
| `A04-29` | journalのsequence重複、別Planのbackup_ref、適用後に第三者が内容変更 | journal不正またはAPPLY_RECONCILEで停止。逆依存復旧でも第三者変更を上書きしない |
| `A04-30` | sequenceの指すstepとstep_idだけ相違、正しいresource_idは一致 | journal照合失敗。資源IDだけで別stepの適用/復旧を実行しない |
| `A04-31` | file entry aとa/b、directory aと未列挙のa/c、暗号overhead未定義 | 前者拒否、未列挙の取得0件、容量判定は未成立。再帰取得・平文代替なし |
| `A04-32` | redirect比較案でHTTPS三段/四段、別originの認証要求、最終200が別hash | 四段・認証転送・内容不一致を拒否。採用前は比較案を実取得へ使用しない |
| `A04-33` | 親update内でアプリ一件とselfが各完了、履歴/終了記録は未確定 | 子stepのverifiedだけ保持。親finishは一回、子が履歴を個別確定・ロック解放しない |
| `A04-34` | mode設定後に所有者変更、属性変更対象fileに別hard link、未管理ACL保持不能 | 属性再照合またはAPPLY_SHARED/保全不成立で停止。別entryの権限変更・modeの成功偽装なし |
| `A04-35` | TTL=1秒、verified_at末尾.500、期限直前/同時、年範囲を超える加算 | 期限は正確に1,000ms後。直前だけ有効、同時/表現範囲外は無効で更新停止、保持物の削除なし |
| `A04-36` | 保存先が取得元の子directory、消去予定disk上、artifact_nameが別operation | BACKUP_DESTINATION/名前不一致で保存開始0回。空き容量の入力値だけで安全性・保存成功を認定しない |
| `A04-37` | saved後に隔離復元失敗、stage=failed/receipt=None | last_confirmed_stage=savedと保存物参照を保持。未作成扱い・再取得・正式receipt生成なし |
| `A04-38` | 親updateが子backupのreceipt待ち、子の保存/復元確認は成功、親finish未実施 | 子receiptを返して親の残step/finishへ接続。互いの完了待ち・子だけで親成功・履歴の二重確定なし |

## 未決事項と成立判定

### 配布取得と専用file-sink

```rust
pub enum FetchKind { Metadata, Binary, Iso }
pub enum FetchStage { Prepared, Receiving, Verified, Failed, Unknown }
pub struct FetchRequest {
    request_id: OperationId, source: String, destination_ref: String,
    expected_bytes: u64, expected_sha256: String, kind: FetchKind,
    authentication_ref: Option<String>, allowed_redirects: Vec<String>,
}
pub struct FetchResult {
    source_ref: String, destination_ref: String, expected_bytes: u64,
    observed_bytes: u64, sha256: String, synced: bool, checks: Vec<Check>,
}
pub struct FetchProgress {
    stage: FetchStage, observed_bytes: u64, temporary_ref: Option<String>,
    result: Option<FetchResult>,
}
```

これらは既定の全キーを所有する内部型候補。FetchResult.source_refはFetchRequest.request_idの表記と一致、destination_refは同じ新規保存物ID、syncedは成功時trueだけ。FetchProgress.resultはVerifiedだけSomeで、未確認のhashを生成しない。ReadOnlyのmetadata取得はこの保存要求を呼ばず、同じURL/認証/redirect検証を借用するメモリー専用adapterへ渡す。A04-26/39に要求ID差替え、verified/resultなし、ReadOnlyからfile-sink呼出しの拒否を含める。

FetchRequestの候補全キーはrequest_id: OperationId、source: 検証済みHTTPS URL、destination_ref: 新規保存先ID、expected_bytes: 正整数、expected_sha256: hex64、kind: metadata/binary/iso、authentication_ref: stringまたはnull、allowed_redirects: 固定originの一意配列。URLはASCIIのhttps、DNS/IPv4/角括弧IPv6と明示port（省略443）、空でない/始まりpath、queryだけを受理し、userinfo/fragment/空白/制御文字/不正percent escapeを拒否する。保存path・自由curl optionは要求キーにしない。

file-sinkの候補は親が保護directoryにcreate_newで開いた通常fileだけを渡し、開始時長0・同FS・nlink=1・対象実体固定を照合する。bodyを書ける実行物のworkerにLimitFSIZE=許可bytesを設定し、設定読戻しと上限+1の拒否試験を必要とする。LimitFSIZEは各fileのサイズ制限であり、総disk使用量制限ではない。stderr/capture/別file/子孫は別の量制約を維持し、disk予約と書込み失敗も検出する。子の終了・file同期・実長・SHA-256照合後だけFetchResultを生成する。renameで既存物を上書きしない。

curlの全argv候補は、固定実行path、--disable、--globoff、--proto、=https、--proto-redir、=https、--connect-timeout、30、--max-time、残通常秒の十進値、--retry、0、--silent、--show-error、--fail、--max-filesize、許可bytes、--output、固定一時path、--write-out、固定書式、--url、検証済みURLの順。書式はhttp_code/size_download/redirect_urlを各一行とする。URL/pathは固定構築した一引数、--locationなし、cwdは当該作業directory、環境はLANG=C/LC_ALL=Cだけ、stdinは空。API認証が必要なら登録済み認証参照から生成したheaderをstdinへ渡す--header/@-の二引数だけを追加する候補とし、秘密値をargv/環境/記録へ置かない。

redirect比較方式では3xxの応答を完成bodyへ昇格せず、Location由来のredirect_urlを元URLとの相対解決後に再検証する。origin許可・最大三段・循環・認証の非転送を全段に適用する。各段は新しい空の一時fileを使い、3xx bodyを最終payloadへ連結しない。固定write-outの量/行数/整数/URLを検証し、外部出力をコマンドとして評価しない。curlの非ゼロ、取得量の相違、HTTP 200以外の最終応答はFETCH_RESPONSE。採用版で全option・TLS・容量強制を実証するまで実行不可。

metadataのplan/dry-run取得はP02のReadOnly文脈とメモリーcaptureだけで、file-sinkを作らない。二入力の第一比較候補は[匿名pipeによるdetached検証](#匿名pipeによるdetached検証)とする。固定版での成立確認・採用はD04-01に残し、計画作成を理由に一時保存・署名検証省略を追加しない。

### 匿名pipeによるdetached検証

```rust
pub struct DetachedInput {
    data: Vec<u8>, signature: Vec<u8>, keyring_ref: String, material: MaterialRef,
}
pub struct SignatureProof {
    data_sha256: String, signature_sha256: String, signer_fingerprint: String, checks: Vec<Check>,
}
```

純粋な構文解析とOS検証器を分ける候補。DetachedInputの全項目はdata/signature（各1 MiB以内の所有byte列）、keyring_ref（独立信頼済み固定参照）、material（gpgvのMaterialRef）。SignatureProofはdata_sha256/signature_sha256/signer_fingerprint/checksを保持する。hash取得も固定元byte列へ束縛し、再生成JSONを署名対象にしない。

API候補`verify_detached_memory(input: &DetachedInput, context: CommandContext<'_>) -> Result<SignatureProof, ContractError>`はReadOnlyだけを受ける専用adapterとする。[std::io::pipe](https://doc.rust-lang.org/std/io/fn.pipe.html)で二組の匿名pipeを作り、子のfd0へ署名用read端、fd1へdata用read端をStdioとして渡す。fd1はこの専用契約では出力でなく検証器が読む入力である。stderrのfd2だけを有界captureにし、通常run_commandのstdout-capture形へ無理に当てはめない。

gpgvの全argv候補は固定path、--quiet、--homedir、既存の固定公開鍵home、--keyring、固定keyring、--status-fd、2、--logger-fd、2、--enable-special-filenames、--、-&0、-&1。cwd=/、環境LANG=C/LC_ALL=Cだけ。二つのwriterとstderr回収を並行し、全input送信後に各write端と親側の不要複製を閉じてEOFを渡す。pipe容量を仮定した全量先書込み・同じwrite端の残存・fdの任意追加は行わない。file/FIFO/socket path・管理directory・ログの作成0件。

[gpgvのfd入力](https://gnupg.org/documentation/manuals/gnupg-devel/gpgv.html)に基づく比較案で、検証器の終了0、全入力消費、完全fingerprint/署名対象/許可algorithm/時刻・失効の独立資料を要求する。gpgv自体はkeyring中の鍵を信頼し失効/期限を検査しないため、その終了0だけで信頼条件をpassにしない。statusの版別文法/許容集合・監督停止/pipe例外の成立根拠は固定環境で確認するまで未達。予算は既存metadata検証の残期限内、再送0回、停止不明はFETCH_VERIFYで成功証拠なし。

| ケースID | 固定入力 | 期待結果 |
|---|---|---|
| `A04-47` | 二入力がpipe容量超、片writer EOF欠損、署名成功だが失効資料不明 | 並行配送/期限内停止/信頼未達を照合。一時file作成・署名省略・gpgv終了0だけの信頼成立なし |

| ケースID | 固定入力 | 期待結果 |
|---|---|---|
| `A04-39` | 128 MiB/+1のbody、第三/第四redirect、秘密header、metadata planでtemp必須 | 厳密量/三段制限/非公開stdin/方式未成立を照合。通常capture上限解除・認証転送・plan中の保存なし |

### 公開鍵暗号化バックアップの形式候補

```rust
pub enum EncryptionMode { CfbMdc, Ocb }
pub struct EncryptionBinding {
    material: MaterialRef, recipient_primary_fingerprint: String,
    recipient_subkey_fingerprint: String, public_keyring_ref: String,
    decryption_ref: String, mode: EncryptionMode, cipher: String,
    format_version: u32, verification_ref: String,
}
```

EncryptionBindingは後述の九キーだけへ対応する型候補。modeの外部名はcfb_mdc/ocb、cipher/format_versionは同節の固定値だけを受理する。enumに両方式があることは両方の採用・安全性・生成可能性の証明でなく、profileに未対応の組は暗号化前拒否。暗号物の外側packetと復号statusだけでは、暗号内部のliteral metadata/partial chunk境界の全条件を証明できない。上界Cの利用には、採用固定版の生成契約が内部条件を保証する独立資料と反証fixtureを必要とし、根拠不明なら容量検査を未成立にする。A04-40/48に外側は同じだが内部圧縮/metadataが異なる試料を含める。

第一比較候補をGnuPGの公開鍵暗号化とする。gpgvは署名検証用であり、この用途の代わりにしない。EncryptionBindingの全キー候補はmaterial: MaterialRef、recipient_primary_fingerprint: string、recipient_subkey_fingerprint: string、public_keyring_ref: string、decryption_ref: string、mode: cfb_mdc/ocb、cipher: AES256、format_version: 1、verification_ref: string。fingerprintはGnuPGの完全な値を検査してから固定し、短縮ID/メールアドレスによる鍵選択なし。暗号profileと固定版/鍵方式は安全ポリシーの採用対象とする。

暗号化する平文container候補はmagicのASCII8bytes SSBK0001、続いてmanifest長のunsigned little-endian64bit、UTF-8のmanifest、昇順entryのfile内容を順に連結する。manifestの全キーはformat_version: 1、schema_version: 1、backup_id、source、profile、data_kind、entries、method、restore_ttl_seconds、recipient_primary_fingerprint、recipient_subkey_fingerprint。共通項目の型はBackupSetを参照し、destinationや秘密鍵参照は入れない。JSONはP02のキー順/escape、末尾LFなしで固定する。format_versionはcontainer形式、schema_versionはBackupSetの版として区別する。

manifest長は1 MiB以下、entriesは最大4,096件、各fileは最大16 GiB、平文全量（16byte header+manifest+全file）は最大64 GiBを候補とする。directoryは内容0bytes。全量はchecked加算で暗号化前に算定し、実streamが宣言量と一致することを復号後にも検査する。各entryの実hash・順序を照合し、末尾の一byteでも余剰があれば拒否する。圧縮・tar・シンボリックリンク・外部URLを内包しない。別アプリの整合性はcontainerでは保証せずP06-DATAへ従う。暗号後容量の上界は選定するOpenPGP packet形式とchunk設定から別途証明する必要があり、平文64 GiBだけでfile-sink容量を確定しない。

平文は対象側の固定entryから64 KiB以下のbufferで匿名pipeへ直列に送り、ディスク/USBに平文一時archiveを作らない。これは通常run_commandの1 MiB stdinとは別のBackupStream契約で、入力実量/最大量・同一性を全区間検査する専用adapter。buffer再利用と取得対象の固定を要求し、無制限Vecへ集めない。暗号化stdoutは保護されたfile-sink、status/診断はstderr各1 MiB上限へ分け、秘密断片を診断に転記しない。書込み中断・source変化は全体失敗とし、平文へのfallbackなし。

暗号化argv候補は固定gpg path、--no-options、--homedir、公開鍵専用の固定home、--batch、--no-tty、--no-auto-key-retrieve、--auto-key-locate、clear、--no-default-recipient、--trust-model、always、--recipient、完全subkey fingerprintに!を付けた値、--personal-cipher-preferences、AES256、--compress-algo、none、--status-fd、2、--set-filename、空文字、--output、-、--encrypt。cwdは保護作業領域、環境はLANG=C/LC_ALL=Cだけ。alwaysは独立に検証した当該公開鍵だけの選択に限定する審査候補で、信頼根拠を省略する包括許可ではない。候補modeは鍵の能力/固定gpg版と照合し、異なるmode/cipherで生成された物を無条件受理しない。

復号argv候補は固定gpg path、--no-options、--homedir、登録済み復号環境、--batch、--no-tty、--no-auto-key-retrieve、--auto-key-locate、clear、--status-fd、2、--max-output、平文許可bytes、--output、-、--decrypt、--、固定暗号物path。秘密鍵の解錠は安全なagent/token参照を必要とし、passphraseを引数/環境/設定へ渡さない。復号の途中出力は未検証であり、完全性成功前に本番復元へ使わない。隔離された非永続検証領域へ流し、全体検査後だけ復元計画へ渡す。

[GnuPG操作](https://gnupg.org/documentation/manuals/gnupg/Operational-GPG-Commands.html)、[入力/出力](https://gnupg.org/documentation/manuals/gnupg/GPG-Input-and-Output.html)、[OpenPGP option](https://gnupg.org/documentation/manuals/gnupg/OpenPGP-Options.html)へ照合した候補。force-mdcは現在の資料では無効な旧optionであり、完全性の証明に使わない。終了0だけでなく、暗号処理完了・復号完全性・宣言cipher/mode・信頼済み保全物hash・全entryの対応を別に要求する。採用版のstatus全集合/packet互換・容量overheadの根拠は個別審査し、候補argvだけで使用可能にしない。

| ケースID | 固定入力 | 期待結果 |
|---|---|---|
| `A04-40` | 正常containerの往復、暗号文1byte破損、復号途中EOF、別recipient、余剰entry/末尾 | 完全性/鍵/内容対応を個別検査。不完全な復号の本番適用・平文保存・任意path復元なし |

### 暗号profileと容量の閉じた比較案

cfb_mdc側の比較profile名をopenpgp-cfb-v1とし、AES256、一recipient、一つのPKESK、一つのversion=1 SEIPD、圧縮なし・armorなし、平文はbinary literal一件だけとする候補。ocbを同じprofileで自動受理しない。これは互換性比較用の未採用案で、現行の暗号方式として推奨/使用可能としたものではない。[RFC 9580](https://www.rfc-editor.org/rfc/rfc9580.html)と旧[RFC 4880](https://www.rfc-editor.org/rfc/rfc4880.html)の位置付け、安全ポリシー、固定GnuPG版を所有者が審査して採否を決める。

候補の容量上界はP=平文container全byte数、H(n)=6+ceil(n/512)、L=P+6+H(P+6)、E=L+41、C=65,536+E+H(E)とする。全演算はchecked、ceilは整数算定。PKESK全packetを65,536bytes以内、partial bodyの一chunkは512bytes以上、最後はdefinite length、literalのfilename長0と固定6byte metadataに限定してこの上界を使う。余剰packet/圧縮/許容外chunk/不明版は拒否する。出力がC以内という必要条件だけで暗号の正しさを認定しない。暗号file-sinkはC、平文解析はPへ独立制限し、FS別の一時物/旧物/記録予約も加算する。実装環境でこのprofileを生成/復号できることは未検証。

暗号argv候補は既定列の--no-options直後に--rfc4880を置き、cipher指定を--cipher-algo/AES256へ固定する。これだけを出力profileの証明にせず、外側packet検査と復号時のstatusを照合する。[GnuPG DETAILS](https://raw.githubusercontent.com/gpg/gnupg/master/doc/DETAILS)の版を実行物へ固定し、BEGIN_ENCRYPTION/END_ENCRYPTION、DECRYPTION_INFO/DECRYPTION_KEY/DECRYPTION_OKAY/END_DECRYPTIONと完全性結果を要求する。選定版で必要なMDC結果・cipher=9/AEADなしを一意に検査できなければ不成立。鍵のfingerprint/用途/期限/失効は安全ポリシーから独立に照合する。エラー/失敗status/未知の必須文法/途中EOFがあれば正式receiptなし。

| ケースID | 固定入力 | 期待結果 |
|---|---|---|
| `A04-48` | Pの最小/64 GiB境界、C+1暗号、別mode/圧縮packet、cipher宣言不一致 | 上界を独立算定し超過/profile不一致で停止。平文量だけの容量合格・gpg終了0だけの完全性/信頼認定なし |

### 共通設定の復元確認方法

configuration用の方法候補configuration-v1は、[管理ファイル一覧](../BASIC_SPEC.md#管理ファイル一覧)のconfig/policy/選択機能の設定、対応するversion/managed/machine-id、公開検証keyringと署名済みrelease検証情報を対象とする。選択機能の必須設定はP06/P07/P08の各validatorで決め、未適用機能のファイルを黙って採取しない。秘密鍵/認証token/アプリデータ/DB/実行中lock・PID・媒体の状態をこの方法へ追加しない。master用の固有情報はP08-BACKUPへ渡す。

復元確認は新規の隔離rootへ全entryを復号し、実量/hash/属性を一対一照合→共通と構成別のvalidator→構成/revision/対象の横断整合→署名済み本体が各設定を読める診断、の順。ネットワーク接続や本番サービス起動を行わない。machine-id/managedの保存値は整合確認だけに使い、別機器の現在IDを上書きしない。実復元先が元root/アプリdataへ解決する場合はBACKUP_ISOLATIONで拒否する。

この方法のTTL候補は86,400秒、通常は一日以内に復元確認を更新する。有効性にはTTLに加え、対象ID・構成・設定revision・方法/暗号/鍵参照・全設定content hashの一致を要求する。一項目の変化で旧verifiedを無効にし、TTL内でも新しい保全/確認が必要。アプリ/DB/masterデータへこの期限を一般化しない。別方法の有効期限は、その一意なデータ契約で確定した値を使う。

復元確認が終わった隔離領域は、全子処理停止・証拠確定・参照が残らないことを確認した後、当該計画で作成した領域だけを閉じる。失敗/不明時は非公開で保全し、任意の再帰削除へ進まない。保持は自動削除なしを初期候補とし、容量不足では新しい保全/更新を停止する。保持期間と復元確認TTLを同じ意味にしない。

| ケースID | 固定入力 | 期待結果 |
|---|---|---|
| `A04-41` | 設定の隔離往復、TTL未満で設定hash変更、master情報をconfigurationへ混入、本番rootへの復元 | 同一物だけ有効、変更後無効、方法/隔離違反を拒否。共通期限のアプリ流用・ID上書き・失敗領域の無条件削除なし |

### 復元の検証先と平文容量

```rust
pub struct RestoreFixture {
    environment_ref: String, root_binding: RootBinding, volatile_filesystem_ref: String,
    available_bytes: u64, required_bytes: u64, expected_checks: Vec<String>,
}
```

復元確認の候補入力RestoreFixtureはenvironment_ref/root_binding/volatile_filesystem_ref/available_bytes/required_bytes/expected_checksの六項目。参照は固定した隔離環境/RootBinding、bytesは正の共通整数、expected_checksは方法の非空・一意集合。required_bytesは全復号fileの実量+同時保持する作業物/属性/記録予約をFS別にchecked算定し、実RAM/FS/有効制限の最小へ照合する。64 GiBの保存許容を64 GiBの復元先が常にあるという保証にしない。

検証先の候補は非永続の専用FSだけで、swap/core dump/ログ等の保護条件は安全ポリシーの採用が前提。平文全量を無制限Vecへ集めず、64 KiB以下のbufferと個別fileへ展開する。復号完全性・信頼済み暗号hash・container/全entry検査・方法の実復元を終える前に、本番対象へ平文を適用しない。隔離先不足ならBACKUP_ISOLATIONで開始0件、通常diskへの平文一時archiveへ切り替えない。

本番への実復元は、同じ固定暗号実体を再照合し、新しい対象/復元計画とRootBindingの下で行う。隔離復元成功だけで本番書込み許可を生成せず、暗号物/方法/対象版が変われば旧確認は無効。平文の後始末は対象子処理停止・参照解消と許可済み範囲だけで、失敗後の任意再帰削除はしない。

| ケースID | 固定入力 | 期待結果 |
|---|---|---|
| `A04-49` | 暗号物は上限内だが隔離FS容量不足、swap保護不明、隔離検査後に暗号実体変更 | 復元開始0件/安全審査待ち/確認無効。通常disk平文保存・RAM無制限割当・旧検証の流用なし |

### packageとunitの実行テンプレート

外部adapter候補は次表の固定テンプレートだけとする。package名はDebianの名前規則と対象CPUを検証し、option開始・空白・wildcard・shell記号を受理しない。版は実際の署名付き採用package/indexへ完全一致、依存を含む変更予定package集合と版/取得元/容量をPlanへ固定する。自由なapt設定を利用者入力から受けない。

| 操作 | argv候補と照合 |
|---|---|
| package観測 | 固定dpkg-query path、--show、--showformat=${db:Status-Abbrev}\\t${binary:Package}\\t${Version}\\n、--、package名。書式全体は一引数でshell評価しない。終了1の不在とDB読取り失敗を別扱い |
| package計画 | 固定apt-get path、--simulate、--no-remove、--no-install-recommends、install、--、固定name=version列。locale固定、対象版の出力文法を検査し、未知行/追加変更/除去予定はblocked |
| package適用 | 固定apt-get path、--assume-yes、--no-remove、--no-install-recommends、-o、Dpkg::Use-Pty=0、-o、DPkg::Lock::Timeout=0、-o、Dpkg::Options::=--force-confold、install、--、同じ固定name=version列。DEBIAN_FRONTEND=noninteractiveを明示。対話選択・消去確認の代行には使わない |
| unit観測 | 固定systemctl path、--no-pager、--no-ask-password、show、--property=Id,LoadState,UnitFileState,ActiveState,SubState,FragmentPath、--、unit名。各key一件/未知/欠損を検査 |
| unit変更 | 固定systemctl path、--no-pager、--no-ask-password、enable/disable/start/stop/reloadのうち計画した一語、--、unit名。任意のrestartへ置換しない |
| 定義再読込み | 固定systemctl path、--no-pager、--no-ask-password、daemon-reload。変更したunit定義と全同居機能への影響が計画にある場合だけ一回 |

unit名候補はASCII英数字で始まる英数字/underscore/dot/hyphen、128bytes以内、末尾.service/.timer/.sliceの一件とし、glob/空白/slash/任意instance展開は拒否する。file資源で固定した定義hashとFragmentPath、drop-inの検証を必要とする。enableとactiveは独立の状態で、起動要求0をactiveの証明にしない。

cwd=/、基本環境LANG=C/LC_ALL=C/PATH=/usr/sbin:/usr/bin:/sbin:/bin、stdin空を候補とし、aptだけ表の非対話変数を追加する。PATHは選定済みOSの子コマンド集合に限定し、実行物の探索/動的追加の許可ではない。package全変更集合とDB/index/構成hashを適用直前に再照合し、別apt稼働・版変化・未知conffile選択があれば停止する。package操作は自動再投入/自動ダウングレード/除去0回。部分導入は実DBとjournalへ残し、元状態への復元可能性を個別に判定する。採用版の出力/依存/maintainer script副作用の契約が未評価なら、テンプレートがあっても適用可能とはしない。

| ケースID | 固定入力 | 期待結果 |
|---|---|---|
| `A04-42` | simulateに未計画package/除去、適用前DB変更、enable成功/active失敗、未知drop-in | 計画破棄または実結果不成立。全体verified・apt自動再投入・未計画サービス再起動なし |

| 判断ID | 残る具体化・採用判断 | 未達の影響 |
|---|---|---|
| `D04-01` | [配布取得](#配布取得と専用file-sink)と[二入力署名検証](#匿名pipeによるdetached検証)に案あり。固定curl/gpgv版のfd/文法/監督、強制容量/停止と独立した鍵信頼の成立根拠・採用判断が残る | 資材準備・本体導入 |
| `D04-02` | [暗号profile/容量](#暗号profileと容量の閉じた比較案)、[設定復元](#共通設定の復元確認方法)、[隔離容量](#復元の検証先と平文容量)、P08 master-v1に案あり。固定GnuPGでのprofile生成/上界/完全性の実証、暗号/鍵/方法の採用、個別アプリ方法行が残る | 保全・復元・重要更新 |
| `D04-03` | MigrationRuleの照合案あり。実revisionと旧二択/三構成移行の全対応、不可逆性/復元条件と採用判断が残る | 更新互換性 |
| `D04-04` | [package/unit](#packageとunitの実行テンプレート)に固定argv案あり。推移実行物/maintainer scriptの範囲、外部版と同期/復旧不能範囲の成立根拠が残る。root固定はD03-02 | IaC・構成適用 |

共通契約の未承認、セキュリティ設定の未確定条件、外部依存の採用情報、非機能の定量条件も残る。これらに依存する契約は完全仕様未成立とし、文書案の検査を製品受入へ転用しない。
