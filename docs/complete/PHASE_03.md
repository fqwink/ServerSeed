# Phase 3 完全仕様案: 内製IaCの観測・差分・計画

[文書一覧](../DOCUMENT_INDEX.md) | [担当工程](../ROADMAP.md#phase-3-内製iacの観測差分計画)

本案は`B1`、採用未承認。[案の適用範囲](PHASE_01.md#案の適用範囲)と既存の[宣言型管理](../BASIC_SPEC.md#宣言型構成管理内製iac)へ照合する。対象要件は`REQ-004`・`REQ-006`・`REQ-007`・`REQ-008`・`REQ-011`・`REQ-012`。

## 契約と共通参照

| 契約ID | 所有する範囲 |
|---|---|
| `P03-RESOURCE` | 資源定義・管理属性・所有範囲 |
| `P03-FS` | 対象ルートに対するパス・ファイルの観測 |
| `P03-PLAN` | 差分・依存順序・固定した計画の生成 |

共通参照（全フェーズB1）は`P02-MODEL`・[識別型](PHASE_02.md#p02-model)、`P02-PARSE`・[解析](PHASE_02.md#p02-parse)、`P02-STATE`・[対象状態](PHASE_02.md#p02-state)、`P02-CLI`・[出力](PHASE_02.md#p02-cli)。適用・記録・保全は`P04-APPLY`へ渡し、観測・計画だけでは実行しない。

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

観測は、対象内の論理パス、対象ルートと機器識別の組で行う。採用案ではNUL、`..`、空成分、重複区切り、末尾の不要な区切りを拒否し、シンボリックリンクを辿らない。対象ルート自体の識別に失敗したら全体を停止する。

通常ファイル・ディレクトリ以外、読み取り不能、所有情報不明を資源不在として返さない。観測結果は`present`・`absent`・`unknown`と理由を持つ。file内容・属性と親ディレクトリの識別情報を記録し、実行環境の同名パスを検査しない。

検査と利用の間の差し替えを防ぐ具体手段、マウント境界の扱い、対象ルートを安全に固定するOS契約は`D03-02`で審査する。文字列の正規化や事前のsymlink検査だけで安全性を証明した扱いにしない。[導入先の保護](../SECURITY_POLICY.md#導入先と一時ファイル)はその正本へ参照する。

## P03-PLAN

1. 解析済み定義を資源スキーマへ照合し、対象・管理属性・所有者・共通定義との競合を検査する。
2. 資源を観測し、取得不能を実行不能理由へ対応付ける。
3. 属性ごとに`unchanged`・`create`・`modify`・`blocked`へ分類する。削除・管理解除は[初期の適用条件](../POLICIES.md#宣言型構成管理の適用条件)に含めない。
4. 依存グラフを検査し、循環があれば計画全体を停止する。同順位はresource_idのASCII順で並べる採用案とする。
5. 観測した対象・定義の識別情報、属性差分、順序、必要権限、保全・復旧範囲を[既存の更新計画](../BASIC_SPEC.md#更新計画)へ結合する。新しい総合状態は作らない。

同じ定義・実状態では、生成時刻とplan_idを除く変更内容・順序・停止理由が同じになることを要求する案。計画の同一性に使用する正規化バイト列とdigestは`D03-03`で確定し、既存JSON例のハッシュ欄を独自追加した扱いにしない。

適用前に対象・定義・観測前提が変わった場合は計画を破棄して再計画・再確認する。計画の有効期間は生成から10分以内を候補とするが、期間内でも前提照合を省略しない。観測・差分・計画の各上限は30秒、再試行0回を提案する。

## 内製単位とAPI案

配置候補は`src/core/iac/resource.rs`・`observe.rs`・`plan.rs`。同一パッケージ内で解析・対象識別を再利用する。解析・観測・計画間をValueで受け渡す場合も、下記スキーマ検証を通した値だけを受理する案とする。

| 宣言案 | 入出力・エラー・副作用 |
|---|---|
| `resources(input: &Value, target: &TargetEvidence) -> Result<Value, ContractError>` | P03-RESOURCEの検証済み集合を所有して返す。未知属性・競合で失敗 |
| `observe(target: &TargetEvidence, resources: &Value) -> Result<Value, ContractError>` | 資源別実状態を返す。実状態取得以外の副作用なし |
| `plan(desired: &Value, observed: &Value) -> Result<Value, ContractError>` | 差分・順序・blocked理由を返す。OS・管理記録への書き込みなし |

型は`P02-MODEL`。資源・観測・計画のValue全集合のスキーマは未充足であり、汎用Valueを使うことを型・必須性の確定の代わりにしない。標準の集合・パス・I/Oを再利用し、資源評価・差分・計画は新規内製候補。難易度は計画が中、パスの競合対策が高。旧実装は存在せず、抽出用crateは先行作成しない。

## データ契約の具体化案

`P03-RESOURCE`のattributes・adoptionと`P03-PLAN`の差分を具体化する候補。共通のキー・型の検証は[P02-MODEL](PHASE_02.md#共通参照型の候補)へ従い、以下は既存の実設定へ未知キーを追加する採用決定ではない。

| 型・対象 | 全キー・制約の候補 |
|---|---|
| ResourceSet | schema_version: 1、target: TargetEvidence、resources: Resource[]。資源上限はP03-RESOURCE |
| Resource | 同節のresource_id・owner・kind・path・attributes・depends_on・adoption。depends_onは内部値では必ずarrayとして存在 |
| FileAttributes | content_ref: string、content_bytes: 0〜1,048,576のinteger、content_sha256: 既存ハッシュ表記string、uid・gid: 0〜4,294,967,294のinteger、mode: 同節の8進表記 |
| DirectoryAttributes | uid・gid・mode。file内容は指定不可 |
| Adoption | kind: new・recorded・confirmed、evidence_id: stringまたはnull。newだけnull、既存資源は対象へ結合した証拠IDが必須 |
| ResourceObservation | resource_id: string、presence: present・absent・unknown、attributes: 対応属性objectまたはnull、identity: stringまたはnull、reason: stringまたはnull |
| ResourceChange | resource_id: string、action: unchanged・create・modify・blocked、before: ResourceObservation、after: 対応属性object、changed_attributes: string[]、blocked_by: Diagnostic[] |

ownerはcore・server・mining・masterのうち一つ。これは資源の担当機能であり、選択構成値ではない。masterが呼ぶserver機能の資源はserver所有のままとする。共有する同一パスは生成前に単一所有者へ解決し、同値の重複でも二資源として受理しない。resource_idが同じでpathが違う場合も拒否する。

ResourceObservationでpresentはattributes・identityあり、reasonなし。absentは両者nullで不在理由あり、unknownは既知部分を混ぜた属性objectを返さずnullと理由を持つ。identityはOS側の同一性証拠への参照で、パスの文字列一致やmtimeだけを根拠に生成しない。fileの長さ・内容byte列、uid・gid・modeをそれぞれ比較し、変更属性名はASCII順。directoryの配下のファイル一覧は管理属性へ含めない。

計画側の候補データは`target: TargetEvidence`、`definition_revision: integer`、`changes: ResourceChange[]`、`order: string[]`、`contract_refs: ContractRef[]`。既存計画へ結合する位置・形式版は`D03-03`の採用事項とする。orderにはblockedでない実変更資源だけを各1回含め、unchangedの依存先も観測前提として保持する。blockedが1件でもあれば計画全体は実行不能で、一部資源だけの暗黙適用をしない。

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

## 未決事項と成立判定

| 判断ID | 残る採用判断・確認 | 影響先 |
|---|---|---|
| `D03-01` | 管理範囲・取り込み・資源種別ごとの完全スキーマと互換版 | 差分・構成適用 |
| `D03-02` | 競合しない対象ルート固定とOS上のパス操作、特殊FSの扱い | 観測・適用・保全 |
| `D03-03` | 正規化・digest・計画全フィールド・期限の採用と受入資材 | 媒体・SSHへの引継ぎ |

実装移行には、これらと使用するPhase 2契約の承認済み版、非機能要求の不足を解消する必要がある。差分・計画本文案は作成したが、完全仕様の成立・実装・試験成功は未達とする。
