# Phase 3 完全仕様案: 内製IaCの観測・差分・計画

[文書一覧](../DOCUMENT_INDEX.md) | [担当工程](../ROADMAP.md#phase-3-内製iacの観測差分計画)

本案は`B10`、採用未承認。[案の適用範囲](PHASE_01.md#案の適用範囲)と既存の[宣言型管理](../BASIC_SPEC.md#宣言型構成管理内製iac)へ照合する。対象要件は`REQ-004`・`REQ-006`・`REQ-007`・`REQ-008`・`REQ-011`・`REQ-012`。

## 契約と共通参照

| 契約ID | 所有する範囲 |
|---|---|
| `P03-RESOURCE` | 資源定義・管理属性・所有範囲 |
| `P03-FS` | 対象ルートに対するパス・ファイルの観測 |
| `P03-PLAN` | 差分・依存順序・固定した計画の生成 |

共通参照（全フェーズB10）は`P02-MODEL`・[識別型](PHASE_02.md#p02-model)、`P02-PARSE`・[解析](PHASE_02.md#p02-parse)、`P02-STATE`・[対象状態](PHASE_02.md#p02-state)、`P02-CLI`・[出力](PHASE_02.md#p02-cli)。適用・記録・保全は`P04-APPLY`へ渡し、観測・計画だけでは実行しない。

## P03-RESOURCE

採用案では、組込み構成定義から次の資源集合を生成する。利用者向けの独立したIaCファイルやコマンドを追加しない。

| フィールド案 | 型・必須性・制約 |
|---|---|
| `resource_id` | 必須string、`[a-z][a-z0-9._-]{0,63}`。同一対象内で一意 |
| `owner` | 必須string。共通基盤または選択構成で適用する担当機能。共有資源の担当を一つへ解決 |
| `kind` | 必須string。Phase 3の候補はfile・directoryのみ |
| `path` | 必須string。P03-FSの対象内絶対パス |
| `attributes` | 必須object。管理する属性だけを列挙。fileは内容・所有者・group・mode、directoryは所有者・group・mode |
| `depends_on` | 任意array、未指定は空。resource_idの一意な集合。自己参照・欠損参照を拒否 |
| `adoption` | 必須の取り込み判定。既存資源の所有根拠と計画確認への対応を持つ。存在だけでは管理下にしない |

内容はバイト単位、modeは4桁8進文字列、所有者・groupは対象側の数値IDへ解決した値で比較する候補。属性未指定を空値・削除と解釈しない。資源数は1対象1,000件以下、fileの内容は1件1 MiB以下を提案する。具体的な所有根拠・ID型・既存資源の取り込み表現は`D03-01`に残す。

## P03-FS

観測は、対象内の論理パス、対象ルートと機器識別の組で行う。採用案の資源pathは/から始まるUTF-8文字列で、NUL・制御文字・.・..の成分、空成分、重複区切り、末尾の区切りを拒否する。rootの/自体はRootBindingの対象であり管理資源にしない。trim・相対path補完・Unicode正規化・.の除去で入力を修復せず、シンボリックリンクを辿らない。対象ルート自体の識別に失敗したら全体を停止する。

通常ファイル・ディレクトリ以外、読み取り不能、所有情報不明を資源不在として返さない。観測結果は`present`・`absent`・`unknown`と理由を持つ。file内容・属性と親ディレクトリの識別情報を記録し、実行環境の同名パスを検査しない。

検査と利用の間の差し替えを防ぐ具体手段、マウント境界の扱い、対象ルートを安全に固定するOS契約は`D03-02`で審査する。文字列の正規化や事前のsymlink検査だけで安全性を証明した扱いにしない。[導入先の保護](../SECURITY_POLICY.md#導入先と一時ファイル)はその正本へ参照する。

P03-FSの成立根拠候補は、対象root/FS・親entry識別列・使用する操作集合・競合writerの排除根拠・有効な採取世代を一組で保持するRootBinding。操作前後の一致だけを排除根拠にはできない。比較案Aは排他を証明できるオフライン対象を標準File APIで扱う方式、案Bは固定root相対操作を提供する個別OS実行物へ委譲する方式。推奨は成立根拠を揃えた方式だけを対象環境単位で採用し、どちらも証明不能なら変更機能を保留すること。任意helperの新設・FFI・未採用実行物への委譲は許可しない。特殊FS/mountの許可集合は選定資料で一意に決める必要がある。

## P03-PLAN

1. 解析済み定義を資源スキーマへ照合し、対象・管理属性・所有者・共通定義との競合を検査する。
2. 資源を観測し、取得不能を実行不能理由へ対応付ける。
3. 属性ごとに`unchanged`・`create`・`modify`・`blocked`へ分類する。削除・管理解除は[初期の適用条件](../POLICIES.md#宣言型構成管理の適用条件)に含めない。
4. 資源をstepへ対応付け、[固定形式の単一の順序](#入力と保存形式の詳細案)で依存グラフを検査する。循環があれば計画全体を停止し、資源だけの別の実行順序を作らない。
5. 観測した対象・定義の識別情報、属性差分、順序、必要権限、保全・復旧範囲を[既存の更新計画](../BASIC_SPEC.md#更新計画)へ結合する。新しい総合状態は作らない。

同じ定義・実状態では、生成時刻とplan_idを除く変更内容・順序・停止理由が同じになることを要求する案。計画の同一性に使用する正規化バイト列とdigestは`D03-03`で確定し、既存JSON例のハッシュ欄を独自追加した扱いにしない。

適用前に対象・定義・観測前提が変わった場合は計画を破棄して再計画・再確認する。計画の期限境界は[状態と中断](#状態と中断の詳細案)に集約し、期間内でも前提照合を省略しない。観測・差分・計画の各上限は30秒、再試行0回を提案する。

## 内製単位とAPI案

配置候補は`src/core/iac/resource.rs`・`observe.rs`・`plan.rs`。同一パッケージ内で解析・対象識別を再利用する。Valueは解析境界だけで受け、内部の受渡し宣言は[型付きAPI](#依存と受入の確定候補)へ集約する。

| 宣言案 | 入出力・エラー・副作用 |
|---|---|
| 資源検証 | validate_resourcesだけ。検証済みResourceSetを所有して返す |
| 観測 | observe_resourcesだけ。実状態取得以外の副作用なし |
| 計画 | build_plan/seal_planだけ。OS・管理記録への書き込みなし |

型はP02-MODELと本書のResourceSet/ResourceObservation/Planへ対応付け、旧Value返却APIは実装しない。資源・所有根拠・対象固定・計画形式の採用判断はD03-01〜03に残す。標準の集合・パス・I/Oを再利用し、資源評価・差分・計画を内製する。難易度は計画が中、パス競合対策が高。旧実装・抽出用crateは存在しない。

## データ契約の具体化案

`P03-RESOURCE`のattributes・adoptionと`P03-PLAN`の差分を具体化する候補。共通のキー・型の検証は[P02-MODEL](PHASE_02.md#共通参照型の候補)へ従い、以下は既存の実設定へ未知キーを追加する採用決定ではない。

| 型・対象 | 全キー・制約の候補 |
|---|---|
| ResourceSet | schema_version: 1、target: TargetEvidence、resources: Resource[]。資源上限はP03-RESOURCE |
| Resource | 同節のresource_id・owner・kind・path・attributes・depends_on・adoption。depends_onは内部値では必ずarrayとして存在 |
| FileAttributes | content_ref: string、content_bytes: 0〜1,048,576のinteger、content_sha256: 既存ハッシュ表記string、uid・gid: 0〜4,294,967,294のinteger、mode: 同節の8進表記 |
| DirectoryAttributes | uid・gid・mode。file内容は指定不可 |
| Adoption | kind: new・recorded・confirmed、evidence_ref: stringまたはnull。newだけnull、既存資源は対象へ結合した証拠参照が必須 |
| ResourceObservation | resource_id: string、presence: present・absent・unknown、attributes: 対応属性objectまたはnull、identity: stringまたはnull、reason: stringまたはnull |
| ResourceChange | resource_id: string、action: unchanged・create・modify・blocked、before: ResourceObservation、after: 対応属性object、changed_attributes: string[]、blocked_by: Diagnostic[] |

ownerはcore・server・mining・masterのうち一つ。これは資源の担当機能であり、選択構成値ではない。masterが呼ぶserver機能の資源はserver所有のままとする。共有する同一パスは生成前に単一所有者へ解決し、同値の重複でも二資源として受理しない。resource_idが同じでpathが違う場合も拒否する。

Adoptionの照合候補はnewなら実観測absent、recordedなら同じtarget/resource_id/path/owner/管理属性集合を持つ確定記録、confirmedなら当該計画での取り込み確認と同じ対象・所有範囲への証拠を要求する。presentへnewを指定、unknownへ任意kindを指定、別対象のevidence_refを指定した場合はRESOURCE_ADOPTIONでblockedとする。content_refの解決表は資材IDから不変byte列への一対一対応とし、同じIDに二つの内容を登録できない。ContentBlobのhash/長さ/参照が揃わないfileは計画へ含めず、directoryへfile内容を補完しない。

ResourceObservationでpresentはattributes・identityあり、reasonなし。absentは両者nullで不在理由あり、unknownは既知部分を混ぜた属性objectを返さずnullと理由を持つ。identityはOS側の同一性証拠への参照で、パスの文字列一致やmtimeだけを根拠に生成しない。fileの長さ・内容byte列、uid・gid・modeをそれぞれ比較し、変更属性名はASCII順。directoryの配下のファイル一覧は管理属性へ含めない。

計画の外部全集合は[Planの固定形式](#入力と保存形式の詳細案)だけで定義する。ResourceChangeは資源差分の内部値としてPlanStep.resource_changeへ結合し、独立したchanges/order/contract_refsの第二形式を保存しない。unchangedの依存資源も観測前提として保持し、blockedが1件でもあれば計画全体は実行不能とする。

差分の固定対象は、対象証拠・定義版・属性・所有範囲・依存・取り込み根拠。時刻・生成IDを除いた構造比較と、配布物の原バイトに対するSHA-256を混同しない。計画digestの実現方式が未確定なら構造比較だけで配送の真正性が成立したとは扱わない。`D03-02`の安全なOS操作・`D03-03`の固定形式とdigest・属性の完全なRust型は残る採用事項とする。

## 接続と責務の照合

file内容は、組込み資材または検証済み保全領域にある不変byte列へのcontent_refで識別する候補。任意ファイルパスやURLをその参照として直接評価しない。取得後の長さ・ハッシュと固定資材を照合し、内容そのものは専用のbyte列として比較する。各byteをJSONのinteger配列にして1 MiBの内容を共通解析のノード制限へ押し込まない。参照の解決・保全・所有権とAPIの型は`D03-01`・`D03-02`で確定する必要がある。

資源数上限と[P02-PARSE](PHASE_02.md#解析の入力と上限)の文書上限は別の制約で、1,000資源なら必ず単一文書として解析・表示できるとの保証をしない。実入力が解析上限を超えれば先に拒否し、内部集合の件数境界は構築器単体で検証する。CLIの表示上限超過もP02-CLIの整形失敗へ接続し、資源を黙って省いて全件成功と表示しない。

計画の作成・確認と実行は時点を分ける。[P04-APPLY](PHASE_04.md#p04-apply)はロック取得後に同じ定義・所有範囲・対象と全観測前提を再取得し、[P09-REMOTE](PHASE_09.md#p09-remote)は固定した計画を配送する。配送IDの一致で計画前提の再照合を省略しない。実行前の期限超過は再計画、実行開始後の期限超過は中断位置の照合であり、同じ扱いにしない。

2026-10-22に[OpenOptionsExt](https://doc.rust-lang.org/std/os/unix/fs/trait.OpenOptionsExt.html)と[File](https://doc.rust-lang.org/std/fs/struct.File.html)を照合した。カスタムフラグ指定やファイルハンドルの保持だけで、対象ルート内の全経路・親ディレクトリ差し替えの防止が成立したとは判断しない。専用OS契約・隔離・反証試験が未確定のため`D03-02`は未解消とする。

## 受入ケース案

| ケースID | 区分・前提 | 期待する出力・副作用と検証 |
|---|---|---|
| `A03-01` | 正常。管理属性がすべて一致 | unchangedのみ。対象と管理状態の内容・時刻が不変 |
| `A03-02` | 正常。同一定義を2回観測・計画 | ID・時刻を除く計画内容と順序が一致 |
| `A03-03` | 異常。同一パスの競合定義、所有範囲不明 | blocked。片方の定義を優先して採用しない |
| `A03-04` | 異常。観測権限不足、特殊ファイル、symlink | unknownまたはblocked。不在としてcreateを計画しない |
| `A03-05` | 境界。1,000件・1,001件、循環・欠損参照 | 件数内の妥当集合だけ処理。超過・不正グラフを拒否 |
| `A03-06` | 異常。親配下の未管理ファイル、定義からの省略 | 無確認の削除・管理解除・巻込みなし |
| `A03-07` | 異常。計画後に対象・定義・内容を交換 | 適用前の再照合で不一致。P04-APPLYを開始しない |
| `A03-08` | 境界。計画期限10分と期限超過、時計不明 | 有効性を個別照合。超過・不明は再計画。実適用なし |

## 追加の受入ケース案

`P03-RESOURCE`・`P03-FS`・`P03-PLAN`の接続に対する設計。実パスの差し替え試験は許可された隔離FSで行い、通常ホストを変更しない。

| ケースID | 区分・前提と入力 | 期待結果・検証方法 |
|---|---|---|
| `A03-09` | 正常。同じcontent_ref・長さ・ハッシュのbyte列、属性差分がmodeだけ | changed_attributes=[mode]、内容変更なし。実byte比較と参照照合を別々に検証 |
| `A03-10` | 異常。同じpathをcore/serverが各資源として宣言、値は同じ | 所有者未解決の競合として全計画blocked。片方の黙認なし |
| `A03-11` | 異常。観測後に親directoryをsymlinkへ交換 | 適用前の対象固定・再照合で停止。方式未確定の間は試験保留として残す |
| `A03-12` | 境界。構築器へ1,000/1,001資源、別途単一文書の解析上限超過 | 件数検査と文書上限検査を分離。前者の件数内を解析成功・全件表示可能と誤認しない |
| `A03-13` | 異常。unchanged依存先だけを計画後に変更、または計画期限超過 | 実変更資源が不変でも適用停止・再計画。残り資源だけ自動適用しない |

## 型とAPIの詳細案

`P03-RESOURCE`は外部Valueを検証後、下記の所有する型へ変換する候補。FileAttributes・DirectoryAttributes・Adoptionの項目と制約は[データ契約](#データ契約の具体化案)が定義先で、Rust表現はそれぞれ同名の非公開フィールドstructとする。内容そのものはbyte列で、ContentRefの解決器だけが取得する。

```rust
pub struct ContentRef(String);
pub struct FileAttributes {
    content_ref: ContentRef, content_bytes: u64, content_sha256: String,
    uid: u32, gid: u32, mode: u16,
}
pub struct DirectoryAttributes { uid: u32, gid: u32, mode: u16 }
pub enum AdoptionKind { New, Recorded, Confirmed }
pub struct Adoption { kind: AdoptionKind, evidence_ref: Option<String> }
pub enum ResourceAttributes { File(FileAttributes), Directory(DirectoryAttributes) }
pub struct Resource {
    resource_id: String, owner: String, path: std::path::PathBuf,
    attributes: ResourceAttributes, depends_on: Vec<String>, adoption: Adoption,
}
pub struct ResourceSet { schema_version: u32, target: TargetEvidence, resources: Vec<Resource> }
pub enum Presence { Present, Absent, Unknown }
pub struct ResourceObservation {
    resource_id: String, presence: Presence,
    attributes: Option<ResourceAttributes>, identity: Option<String>, reason: Option<String>,
}
pub enum ResourceAction { Unchanged, Create, Modify, Blocked }
pub struct ResourceChange {
    resource_id: String, action: ResourceAction, before: ResourceObservation,
    after: ResourceAttributes, changed_attributes: Vec<String>, blocked_by: Vec<Diagnostic>,
}
pub struct ContentBlob { reference: ContentRef, bytes: Vec<u8>, sha256: String }
pub fn resolve_content<'a>(reference: &ContentRef, available: &'a [ContentBlob])
    -> Result<&'a ContentBlob, ContractError>;
```

resolve_contentは収集器が所有する検証済みContentBlob集合を借用する純粋な参照解決器候補。集合は組込み資材・検証済み保全物へ限定し、URL・任意path・未取得資材を登録しない。返却参照は集合より長生きできず、解決器は取得・hashコマンド・暗黙コピーを行わない。対応が0件または複数ならCONTENT_REFERENCE、長さ/hash不一致ならCONTENT_MISMATCH。資源のkindはenumから一意に求め、kindとattributesの不一致を構築時に拒否する。uid/gidはu32、modeは検証済みu16、外部の8進stringとの変換でも正確な桁・許可ビットを検証する。実ファイルの固定をこの型で代替しない。

解析入力はvalidate_resourcesだけでValueから変換し、内部ではResourceSet・ResourceObservationを使う。型変換失敗時に観測・計画・適用を呼ばない。Planの形式・digest・全step payloadの採用判断はD03-03へ対応付ける。

changed_attributesはfileならcontent/uid/gid/mode、directoryならuid/gid/modeの一意ASCII順集合とする候補。content_ref/bytes/hashの三項目を別々の部分変更へ分けずcontentとして一体比較する。before.presence=unknownはBlocked、AbsentはCreate、Presentで差なしはUnchanged、差ありはModifyだけを許可する。kind変更・未定義属性・不一致のactionをPLAN_RESOURCEで拒否する。Blockedには非空blocked_by、他actionには空blocked_byを要求する。

| ケースID | 固定入力 | 期待結果 |
|---|---|---|
| `A03-34` | directoryにcontent変更、unknownをCreate、同一属性をModify | PLAN_RESOURCE。部分contentの書換え、取得不能を不在へ変換、不要な再適用なし |

## 状態と中断の詳細案

`P03-PLAN`の計画はgenerated・blocked・confirmed・invalidatedの内部位置を持つ候補。confirmedへ進む条件は対象・定義・全観測前提・契約版が一致し、blocked_byが空で、必須選択と確認を終えていること。確認前に適用を開始しない。

| 事象 | 判定・後続 |
|---|---|
| 1資源でも観測unknown・競合・循環 | blocked。変更可能な他資源だけを適用しない |
| 計画と同じ資源・属性を再観測 | 差分と依存順を構造比較。時刻・生成ID以外の差があればinvalidated |
| 変更しない依存資源だけが変化 | invalidated。変更対象だけの一致では続行しない |
| 確認後に対象・所有根拠・定義・契約版が変化 | invalidated。再計画と再確認が必要 |
| 作成から600秒未満で前提一致 | 期限条件だけは成立。他の実行条件を省略しない |
| 600秒ちょうど・超過・時刻不明 | invalidated。適用開始0件 |
| 適用開始後に期限到達または応答欠損 | P04-APPLYの実位置照合へ移る。未開始計画へ戻さない |

上記の厳密な期限境界は採用候補であり、既存の「10分以内」という候補をこの境界へ統一する審査対象とする。読み取り観測が途中で失敗した場合は完結した資源の事実を報告できるが、欠損集合を空として実行可能計画にしない。差分・計画の処理に管理状態の修復・自動保存を追加しない。

## 依存と互換性の詳細案

`P03-RESOURCE`はP02-PARSE→資源validator、`P03-FS`は対象識別→OS観測、`P03-PLAN`は検証済み資源・観測→差分/依存順序の方向とする候補。計画器からP04適用器を呼ばず、適用器が計画を入力として受ける。

| 境界 | 完全仕様化で固定する候補 |
|---|---|
| 所有範囲 | root内のパス・属性と機能所有者の組。親directoryの管理を子の所有権に拡大しない |
| 観測 | 対象の機器・root・親の同一性と内容byte列。mtimeだけで一致判定しない |
| ファイル安全性 | 標準File APIだけでTOCTOU防止を保証しない。OS方式と反証試験が未確定なら適用不可 |
| 資源版 | 初期はfile/directoryのみ。package/systemdの型はP04-APPLYの別採用後だけ追加 |
| 計画互換 | 対象・所有範囲・全観測前提・変更属性・順序・停止理由が比較対象。除外は[型付き射影](#入力と保存形式の詳細案)の位置だけ |

計画の比較用digestは[型付き射影](#入力と保存形式の詳細案)の候補だけを使い、配送された全Planの同一性hashはP09に従う。署名対象の配布物を再生成する正規化とは別用途で、hash実行契約はP04-SELFと例外台帳へ接続する。射影の具体案があることと、方式/外部hash実行物の採用・真正性確認の成立は別とする。

抽出候補は観測器と純粋な差分/グラフ評価を分け、観測器を模擬へ差し替えても本番FS安全性の成功証拠にしない。同じResourceSetと観測列で変更集合・順序・blocked理由を新旧比較し、対象固定方式の差は隔離環境で別に再検証する。

## 受入条件の充足照合案

RESOURCEは型・属性・取り込み、FSは対象固定、PLANは依存グラフ・差分・期限へ対応する。外部modeは4桁の`[0-7]{4}`、内部u16の許容範囲は0〜0o7777とする候補。uid/gidの範囲はデータ契約へ照合し、Rust整数へ変換できただけでは受理しない。

| ケースID | 区分・固定入力 | 期待結果と確認 |
|---|---|---|
| `A03-14` | 異常。content_refは同じ、解決したbyte列または長さだけ相違 | 内容照合失敗。異なる内容を同じ資材として計画・適用しない |
| `A03-15` | 境界。mode=0000/7777/8888、uid=4,294,967,294/4,294,967,295 | 前2modeと範囲内uidだけ次段へ。範囲外はvalidatorで拒否 |
| `A03-16` | 境界。確認済み計画の経過599,999/600,000ms、依存資源だけ変化 | 期限未満かつ全前提一致だけ期限検査を通過。同時到達・依存変更は再計画 |

安全なroot固定方式と計画全形式が未確定なら、差分の純粋評価の設計と実FS適用の成立を分ける。上記のuid/mode受理は権限変更の許可ではなく、操作の安全条件を別に満たす必要がある。未達は`D03-01`〜`D03-03`とPhase 2の依存へ追跡する。

## 入力と保存形式の詳細案

P03-PLANの固定形式を以下の候補へ具体化する。Resourceの既存属性は[資源のデータ契約](#データ契約の具体化案)を使用し、同じ属性を計画用の別形式へ再定義しない。

| object | 全キー・型・条件 |
|---|---|
| Plan | schema_version: 1、plan_id: PlanId、created_at: UtcMillis、command: 既存操作名string、target_state: 既存総合状態、requires_confirmation・requires_reboot: bool、definition_revision: u32、target: TargetEvidence、profile: 構成enumまたはself単独時null、contracts: ContractRef[]、resources: Resource[]、observations: ResourceObservation[]、steps: PlanStep[]、blocked_by: Diagnostic[]、digest: hex64string |
| PlanStep | id・title: string、destructive・requires_network・required: bool、contract: ContractRef、depends_on: step IDのstring[]、resource_change: ResourceChangeまたはnull、payload: 担当契約で型を固定したobjectまたはnull。資源stepはpayload=null、非資源stepはresource_change=nullとし、非資源payloadの型はcontractで一意に識別する。未定義のstepは実行不可 |
| ContentRef | 組込み資材はembedded:資材ID、保全物はbackup:OperationId:entry_id。資材ID/entry_idはresource_idの文字・長さ制約。参照表にないID、追加のcolon、URL、絶対パス、親への逸脱を拒否 |

resourcesとobservationsはresource_idで一対一、各資源差分はresource_changeへ一回だけ結合する。非資源stepも固有idで追跡し、P04のStepReceipt.step_idはそのidと一致する。contracts内のcontract_idの重複と同じpathの異なる資源を拒否する。各step.contractはcontractsの一件へ完全一致させ、複数stepが同じ契約を使うことと契約一覧の二重登録を区別する。差分属性・depends_onはASCII順、stepsは依存を満たす安定トポロジカル順で、同順位はstep idのASCII順。blockedが一つでもあれば全体を適用不可にする。

Planを封印する前の候補正規順はresources/observationsをresource_id、contractsをcontract_id、各blocked_byをP02-CLIの診断順とする。資源のdepends_onと変更属性も既定順へ揃え、重複は並替え前に拒否する。意味に順序がある担当payload内の配列は担当正本の順を保存する。digest計算器で配列を追加整列するのではなく、PlanDraft生成時にこの形へ固定する。同じ入力集合の列挙順だけが違う場合に、別digest・別計画を生成しない。

計画順序の候補は、全ID一意/全依存先存在→自己辺拒否→入次数0のstep IDをASCII昇順で一件選択→後続辺を除去して反復、の順。未出力stepが残り入次数0が空ならPLAN_CYCLE、実行可能な先頭部分だけを返さない。資源の順序は対応するstepを生成してこの順序へ結合し、resource_id順とstep ID順の二つの実行列を保存しない。計画同一性はこの確定順の全step・停止理由・全前提で比較し、入力objectの掲載順で変わらない。

digest入力は次の型付き射影だけから生成する候補。最初に親子の束縛と全payloadを検証し、その後に表の項目だけを除く。キー名が同じという理由で再帰削除しない。

| 射影する位置 | 除外する値・事前条件 |
|---|---|
| Plan直下 | plan_id・created_at・digest。PlanDraftにはdigestがない |
| 今回の対象を表すTargetEvidence | Plan.target、SelfChange.target、Backup.source、Restore.destination、App.resources.target、MiningControl.target、Capacity.evidenceのobserved_atだけ。各対象は親Planの対象同一性検査を通す |
| SelfChange | PreparedSelf.plan_idだけ。親Plan.plan_idとの完全一致が必要 |
| MediaInstall / OsInstall.install | StorageInput.erase_confirmed_for_planだけ。親Plan.plan_idとの完全一致が必要。実際の消去許可は別の確認工程で検証する |

Restore.sourceに含む過去のBackupReceipt/RestoreEvidence、backup_id・request_id・epoch、検証済み保全物の参照、ContractRef.commit、payloadの意味を持つ時刻は除外しない。過去計画の参照を現在の親IDとして削除しない。表にない新しい親ID/対象証拠を持つvariantの追加は、この射影契約と受入の改定前にはPLAN_PAYLOADで停止する。

純粋API候補`fixed_plan_bytes(draft: &PlanDraft) -> Result<Vec<u8>, ContractError>`がこの射影と直列化を担当する。objectキーはP02-CLIの整形順、空白なし、末尾LFなし、整数は十進、文字列は同契約のescape、配列は確定順のまま。型の検証前に射影して親ID不一致を隠さない。seal_planも同じ関数を使い、ID・時刻の除外を期限・選択の照合省略へ転用しない。hash取得はP04-SELFの外部境界、Plan全体の配送hashはP09の別契約とする。

| ケースID | 固定入力 | 期待結果 |
|---|---|---|
| `A03-36` | SelfChangeとOsInstallを含む同じ計画で親と対応する子ID/今回の観測時刻だけ変更。次に子IDだけ不一致、過去backup_idだけ変更 | 前者の固定byte列/digestは同一。子だけの不一致は射影前拒否、過去保全参照の変更は別digest。P09の全Plan hashは前者でも変わる |

| ケースID | 固定入力 | 期待結果 |
|---|---|---|
| `A03-17` | 同じ計画のplan_id/created_atだけ変更、次にunchanged資源のbeforeだけ変更 | 前者の固定byte列は同一、後者は異なる。後者を同じ計画として確認・配送しない |

## 実行と復旧の確定候補

P03-FSのrootはパス文字列ではなく観測したdisk/root/FS/bootの証拠へ結合する候補。各資源について対象root、全親directory、最終entryの種別・所有者・同一性を読み、P04-APPLY直前にも同じ集合を再照合する。symlink、mount境界、特殊FS、取得不能を通常fileへ変換しない。

前後の再照合だけでは、二つの照合の間のパス交換を防げない。標準ライブラリだけで安全な相対操作を証明できない環境は適用不可とする。採用方式には、競合する書き手の排除根拠、固定したrootに対する全操作、交換後に別対象へ書かないことの反証試験が必要。保護された親directoryという前提も、同権限の他処理を除外できないなら充足としない。

RootBindingの審査候補の全キーはtarget・root_identity・filesystem_identity・boot_id・generation・parent_entries・operations・writer_exclusion_ref・method_ref。targetはTargetEvidence、親entry列はrootから末端の親までのpath/identity/kind、operationsはread/create/replace/metadata/sync/removeの必要集合、method_refは採用する方式のContractRef、残る識別/根拠は空でない固定参照とする。変更予定の全操作が同じ方式の保証範囲内であることを要求し、一操作でも対象固定・排他の根拠が欠ければFS_BINDINGで適用を停止する。観測だけ可能な方式から変更権限を導かず、ブート世代やmount集合の変化はBindingを失効させる。方式A/Bの成立証拠そのものはD03-02の未達に残す。

計画中の取消は以降の観測を中止し、完結した診断だけ返す。部分観測集合で適用計画を作らない。適用直前の期限切れは再計画・再選択/確認へ戻し、既に開始した操作はP04の中断照合へ渡す。計画器から自動復旧・ファイル修復を呼ばない。

| ケースID | 固定入力 | 期待結果 |
|---|---|---|
| `A03-18` | 前後の観測時だけ元に戻る親交換、root配下への別mount挿入 | 前後一致だけで安全認定しない。固定操作方式の反証試験未達なら適用不可 |

## 依存と受入の確定候補

P03-RESOURCEのvalidator→P03-FSの観測→純粋なdiff/plan→P04-APPLYの適用という一方向の境界を固定する候補。JSON/YAML解析・共通型はP02から再利用し、hash/TLS/圧縮の新規crateを追加しない。

| APIの具体化候補 | 入力の所有・戻り値 |
|---|---|
| validate_resources(input: &Value, target: &TargetEvidence) | Result<ResourceSet, ContractError>。妥当な資源を所有して返す。OSを読まない |
| observe_resources(resources: &ResourceSet) | Result<Vec<ResourceObservation>, ContractError>。OS読取りだけ。部分集合は完全計画へ渡さない |
| build_plan(resources: ResourceSet, observed: Vec<ResourceObservation>, identity: PlanIdentity) | Result<PlanDraft, ContractError>。PlanIdentityはplan_id/created_at/契約参照/対象構成を持つ検証済み値。時計・ID・digestを内部で捏造しない |
| seal_plan(draft: PlanDraft, evidence: &PlanDigestEvidence) | Result<Plan, ContractError>。固定byte列と外部hash検証の証拠を照合した後だけ配送可能なPlanを返す |

PlanDraftはdigestを除くPlanの全検証済み値を所有する内部型で、配送・確認・保存のPlanとして使用できない候補。PlanDigestEvidenceの全項目はfixed_bytes: Vec<u8>、sha256: string、verification_ref: string。seal_planはdraftから同じ固定byte列を再生成して完全一致を確認し、採用済みhash検証契約の証拠と結合する。任意のhex64を渡すだけのsealや、空digestのPlanを適用器へ渡す経路を設けない。

内部所有型の候補を次へ固定する。PlanBodyの各項目は[外部Plan形式](#入力と保存形式の詳細案)の同名キーへ平坦に出力し、body/identityという外部キーを新設しない。

```rust
pub struct PlanIdentity {
    plan_id: PlanId, created_at: UtcMillis, command: String, target_state: String,
    definition_revision: u32, profile: Option<Profile>, contracts: Vec<ContractRef>,
    requires_confirmation: bool, requires_reboot: bool,
}
pub struct PlanBody {
    schema_version: u32, identity: PlanIdentity, target: TargetEvidence,
    resources: Vec<Resource>, observations: Vec<ResourceObservation>,
    steps: Vec<PlanStep>, blocked_by: Vec<Diagnostic>,
}
pub struct PlanDraft { body: PlanBody }
pub struct Plan { body: PlanBody, digest: String }
pub struct PlanStep {
    id: String, title: String, destructive: bool, requires_network: bool, required: bool,
    contract: ContractRef, depends_on: Vec<String>,
    resource_change: Option<ResourceChange>, payload: Option<StepPayload>,
}
pub struct PlanDigestEvidence { fixed_bytes: Vec<u8>, sha256: String, verification_ref: String }
```

schema_versionは1だけ。PlanBodyはvalidator/計画器が所有し、外部から無検証で作らせない。ResourceSet.targetを計画targetへ一度だけ移し、identityから別targetを受けない。command・profile・requires_confirmationは担当操作/必須選択と照合し、呼出元がfalseを指定しただけで確認不要にしない。seal後のbody変更は不可、変更には新しいPlanDraftと再封印/再確認を要求する。

| ケースID | 固定入力 | 期待結果 |
|---|---|---|
| `A03-33` | 封印後にprofile変更、外部JSONにbodyキー、対象の異なる資源集合を結合 | PLAN_PAYLOADまたは対象不一致。封印済み型の再利用・別形式の受理・確認省略なし |

PlanStep.idはresource_idと同じ文字/長さ制約を使う候補だが、資源IDとは別の名前空間とする。全step ID・依存先は一意に解決し、資源一件は一stepにだけ結合する。非資源stepは担当契約のpayload validatorを必須とし、resource_change/payloadの両方null・両方非nullを拒否する。任意stepをrequired=falseにして必須の保全・検証・記録を省略することはできない。

上記の型付きAPIを本案の唯一の内部境界とする。初期の観測器と純粋計画器は別module、将来の抽出でも模擬観測と実FS試験を分離する。新旧でResourceObservation全フィールド・差分・順序・blocked理由・固定byte列を比較し、root固定方式の変更時はP03-FSの反証試験を再実施する。

| ケースID | 固定入力 | 期待結果 |
|---|---|---|
| `A03-19` | 観測adapterだけ差替え、同一資源のidentity一件欠損、同じdiffだが順序が違う | 前者も同一契約を要求。欠損はunknown、順序差は互換不成立。実FS安全性は別検証 |
| `A03-20` | Adoptionへ旧名evidence_id、ResourceSetのschema_version欠損、既存資源にnewを指定 | 型/所有根拠不一致で拒否。別名の黙認・版補完・未確認取り込みなし |
| `A03-21` | root/親entryの前後値は同じだが競合writerを排除不能、未登録FS | RootBinding不成立。観測事実は保持しても変更開始不可、再照合だけの安全認定なし |
| `A03-22` | 同じ資源差分でcommand/再起動要否だけ変更、StepReceiptのIDだけ別値 | 固定byte列は変化し、別IDのreceiptは拒否。第二Plan形式や順序配列を生成しない |
| `A03-23` | present資源にnew、別targetの取り込み証拠、同じcontent IDに別byte列 | RESOURCE_ADOPTIONまたは内容参照衝突でblocked。存在だけによる管理下取り込み・後勝ち登録なし |
| `A03-24` | Bindingはreadだけ保証、適用はreplaceを要求、またはboot/mount集合が変化 | FS_BINDINGで適用0件。観測成功・前後一致を変更の保証へ拡張しない |
| `A03-25` | step a/bが同順位、cはa/bへ依存。入力順逆転、次にb/cの循環を追加 | 正常順はa,b,cで一定。循環はPLAN_CYCLEで全体blocked、先頭aだけの適用なし |
| `A03-26` | resource ID順と対応step ID順が逆、非資源stepも同順位 | 全stepの単一トポロジカル順だけを使用。別の資源実行列を生成しない |
| `A03-27` | 未封印PlanDraft、別byte列のdigest証拠、payload両方null | 配送/適用不可またはseal/形式拒否。空digestの仮Planを成功扱いにしない |
| `A03-28` | resources/observations/contractsの列挙順だけ逆転、担当payloadは順序あり | 前三集合は同じ固定順・digest入力。意味のあるpayload順は保持し、重複を整列で消さない |
| `A03-29` | 資源pathが/、/a/./b、/a//b、末尾slash、制御文字 | 資源validatorで拒否し観測/変更0件。正規化して別pathへ管理範囲を変更しない |
| `A03-30` | 同じContentRefが0件/2件、byte数相違、未登録のbackup参照 | CONTENT_REFERENCE/CONTENT_MISMATCH。純粋解決器は取得せず、借用集合の外へ参照を持ち出さない |

## 未決事項と成立判定

### 非資源stepの閉じたpayload

payload復号の識別候補はcontract_idとstep.idの区分の組とする。check.で始まるIDを検証step専用に予約し、destructive=false・resource_change=null・CheckStepPayloadを要求する。残りでは担当表の型を使うが、P04-APPLYはpackage./unit.、P04-BACKUPはbackup.capture/backup.restoreの区分を必須とする。backupの二IDは完全一致、またはdot一つと資源IDを付けた形だけを許す。他の担当で同じcontract_idに複数の変更型を割り当てる案を追加しない。

型のキーを試し読みして都合のよいvariantを選ばず、復号前に上記の組から一型へ決める。check.以外でCheckの形が来た場合、check.なのに変更型が来た場合、package/unitの区分欠損はPLAN_PAYLOAD。検証stepのexecution_refsは非変更性を別に証明し、接続検査ならrequires_networkとの一致も要求する。資源stepはresource_changeのみという既定の境界を維持する。

| ケースID | 固定入力 | 期待結果 |
|---|---|---|
| `A03-37` | 同じP04-BACKUPでcapture/restore/checkの三step、区分不明ID、checkにPackageResource | 最初の三つはそれぞれ一型へ解決。他は封印前拒否。キー推測・variant tag追加・非変更検査からの更新なし |

PlanStep.payloadの候補は下表の担当型だけとする。型の選択は上記のcontract_id/step.idの組で行い、入力の自己申告で別契約へ切り替えない。表にない契約を変更stepとして受理しない。観測/検証だけのstepはCheckStepPayloadを使い、対象検査ID・期待参照・非変更の実行契約を固定する。外側のMediaRequest/RemoteRequest/Planをpayloadへ入れる循環形式を作らない。

| 担当契約 | payloadの型・全集合 |
|---|---|
| P04-SELF | PreparedSelf。action/target/plan_idを親へ照合 |
| P04-UPDATE | UpdatePayload: layers（os/configuration/application/miner/bodyの一意配列）、materials（MaterialRef/ReleaseArtifactRefの区分付き配列）、migration（MigrationRuleまたはnull）、backup_id（OperationIdまたはnull）。各層の具体stepが別に存在することを検査 |
| P04-BACKUP | BackupSet。復元stepだけRestoreRequestとしてsource/destination/methodを全キーとし、step.idのbackup.capture/backup.restore区分と照合 |
| P04-APPLY | file/directoryはresource_changeだけ。package/unit拡張は対応PackageResource/UnitResourceだけ。任意CommandSpecを外部入力として直結しない |
| P05-MEDIA | InstallPayload。action=installのsetup計画だけで、adoptへpartition工程を加えない |
| P06-APP / P06-DATA | AppPlan / ApplicationDataMethod。アプリ名/全サービス/方法と同じ保全stepへ結合 |
| P07-RUN / P07-UPDATE | ControlRequest / MiningUpdateBinding。ホスト更新を運転制御へ混入しない |
| P08-MASTER / P08-COHOST | RegistryChange / CapacityInput。台帳変更と同居制約設定は別step |
| P10-OS | InstallPayloadとInstallerBindingを持つInstallStepPayload。マスターの配送wrapperを内包しない |
| 各担当の検証step | CheckStepPayload: check_ids（重複なしstring[]）、expected_refs（ContractRef[]）、execution_refs（ContractRef[]）。空集合・未採用の変更実行を検査と偽装した参照を拒否 |

payloadの型は仕様上の担当フェーズが定義し、共通で運ぶdata型はsrc/core/model/payload.rsに置く候補。Coreからprofilesの実行器へ依存するenumにはしない。生成器は型付き値を検証してから固定JSONへ変換し、受信側は同じ契約のvalidatorで検証済み型へ戻す。入力と同じPlanを含む再帰payload、未知の自由object、二契約のキーの混在はPLAN_PAYLOADで拒否する。上記各型の採用・全外部実行契約が不足するstepはblockedを維持する。

```rust
pub enum StepPayload {
    SelfChange(PreparedSelf), Update(UpdatePayload), Backup(BackupSet), Restore(RestoreRequest),
    Package(PackageResource), Unit(UnitResource), MediaInstall(InstallPayload),
    App(AppPlan), AppData(ApplicationDataMethod), MiningControl(ControlRequest),
    MiningUpdate(MiningUpdateBinding), Registry(RegistryChange), Capacity(CapacityInput),
    OsInstall(InstallStepPayload), Check(CheckStepPayload),
}
pub struct CheckStepPayload {
    check_ids: Vec<String>, expected_refs: Vec<ContractRef>, execution_refs: Vec<ContractRef>,
}
```

外部payloadにはvariantのtagを増やさず担当契約とstep区分から一意に復号する。CheckStepPayloadの全check_idsを担当の固定検査集合へ照合し、expected_refsは期待結果、execution_refsは非変更の採取/照合だけを許可する。StepPayload::Checkへ変更実行の契約を入れた場合はPLAN_PAYLOAD。型の参照閉包に未宣言の子型が一件でもあれば当該stepは未成立であり、enumの存在だけを閉包完成に数えない。

| ケースID | 固定入力 | 期待結果 |
|---|---|---|
| `A03-35` | Check payloadに更新実行参照、未登録check ID、外部variant tag | PLAN_PAYLOAD。検査名を根拠に変更を実行せず、未知キーを黙認しない |

| ケースID | 固定入力 | 期待結果 |
|---|---|---|
| `A03-31` | P04-SELFへInstallPayload、payload内にRemoteRequest/Plan、未登録の任意command | PLAN_PAYLOADで封印/配送/適用不可。wrapperの再帰・別契約へのfallback・Core依存方向の逆転なし |

### 内容カタログとRootBindingの失効

RootBindingの型候補を次へ固定し、P04のRestoreFixtureも同じ型を参照する。各値の保証範囲は本節の方式A/B審査が定義し、型の存在は方式の成立根拠ではない。

```rust
pub enum BoundFsOperation { Read, Create, Replace, Metadata, Sync, Remove }
pub struct BoundParentEntry { path: std::path::PathBuf, identity: String, kind: String }
pub struct RootBinding {
    target: TargetEvidence, root_identity: String, filesystem_identity: String,
    boot_id: String, generation: String, parent_entries: Vec<BoundParentEntry>,
    operations: Vec<BoundFsOperation>, writer_exclusion_ref: String, method_ref: ContractRef,
}
```

parent_entriesはrootから末端親までの順序・重複なし、kind=directory、pathは対象内の検証済み絶対パス。root自身のpath=/だけを本型の親列に許し、管理資源としての/の禁止は維持する。operationsは一意な必要操作集合で、読取りだけのBindingをreplaceへ拡張しない。generationは固定した採取世代の証拠ID、writer_exclusion_refは同じ主体/期間/対象の排除資料。未知/空の参照、別boot/世代の混在、対象同一性不成立はFS_BINDING。外部の任意path/自由コマンドへ解決しない。A03-24/32の失効・必要操作の照合にもこの型を適用する。

RemoveはP02の既知一時名清掃、P04の許可済み本体削除/復旧で特定した単一entryにだけ使う候補。呼出元の正本/Plan/所有根拠/実inodeを照合し、directoryの再帰削除・一般資源の削除・保全物の自動除去へ拡張しない。remove保証のないBindingでは、作成/検査が成功しても清掃や本体除去を開始しない。削除済みの確証と、要求後の結果不明を別に保持する。A03-24には清掃だけが保証範囲外の入力も含める。

ContentBlob集合の生成候補は、組込み資材の署名済みrelease対応、または復元検証済みbackupのentry対応を採取器が照合し、全byte列を固定してから純粋解決器へ貸与する方式。資材ID/参照は一意、長さ/hashは採取した同じ実体へ結合する。初期の集合同時保持上限は64 MiB、個別上限はP03-RESOURCE、加算overflow/総量超過はCONTENT_LIMITで計画前停止する。上限を超える場合に任意pathへの遅延参照へ切り替えない。貸与期間中の変更・再取得・同名資材の差替えを許可しない。

旧カタログを除去できるのは、それを参照する未完了Plan/operation/journal/復旧計画が存在しないと照合できる場合だけ。保持期限だけで保全物を自動削除しない。借用側はbyte列を暗黙複製せず、APIを抽出してもカタログ所有者の寿命へ従う。署名済み資材参照と対象パス固定は別の根拠であり、内容が正しいだけで別rootへの書込みを許可しない。

RootBinding方式Aの審査資料は、対象OS停止の証拠、対象block deviceの全mount/holder集合、writer/自動mount/対象変更timerの排除方法、許可FSとmount option、操作を行う全プロセス、排他を維持する期間、namespaceとhotplugの変化検出を全項目とする。方式Bの資料には各操作におけるroot相対ハンドルの使用とsymlink/mount越えを拒否する外部インターフェースの契約を要求する。root pathの前後検査や保護directoryだけの方式はどちらにも分類しない。

Bindingの有効期間は採取世代から一つの親操作終了までで、boot/対象root/mount/親entry/実行主体/排除条件の変更を検出した時点で失効する。新たな対象観測を旧Bindingへ継ぎ足さず、既適用部分を保持してFS_BINDINGで後続停止する。隔離設定の名称は保証ではない。[systemdのRootDirectory](https://www.freedesktop.org/software/systemd/man/255/systemd.exec.html)だけを競合writer排除や同権限プロセスに対する完全な防壁として採用しない。実現方式が未選定の通常稼働OSへ、オフライン方式の成立を一般化しない。

| ケースID | 固定入力 | 期待結果 |
|---|---|---|
| `A03-32` | カタログ合計64 MiB/+1、未完了Planが参照中の除去、操作途中でmount集合変更 | 上限内だけ保持/除去拒否/Binding失効。内容の遅延再取得・再照合だけの安全認定・後続適用なし |

| 判断ID | 残る採用判断・確認 | 影響先 |
|---|---|---|
| `D03-01` | [内容カタログ](#内容カタログとrootbindingの失効)と単一Adoptionに案あり。既存資源の取り込み証拠、生成/保持の実方式と採用判断が残る | 資源の観測・取り込み |
| `D03-02` | [RootBindingの失効](#内容カタログとrootbindingの失効)に案あり。オフライン排他A/OS側root固定Bの成立根拠・許可FS・反証試験は未取得 | 全path操作 |
| `D03-03` | 単一Planと[非資源payload](#非資源stepの閉じたpayload)に案あり。担当payload全子型の閉包、digest取得器の個別版と採用判断が残る | 媒体・SSHの計画引渡し |

実装移行には、これらと使用するPhase 2契約の承認済み版、非機能要求の不足を解消する必要がある。差分・計画本文案は作成したが、完全仕様の成立・実装・試験成功は未達とする。
