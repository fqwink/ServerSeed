# Phase 2 完全仕様案: 共通契約と構成選択

[文書一覧](../DOCUMENT_INDEX.md) | [担当工程](../ROADMAP.md#phase-2-共通契約と構成選択)

案版`D2`・採用未承認。[案の適用範囲](PHASE_01.md#案の適用範囲)に従う。既存の設定値・状態・終了コード・記録例は基本仕様が正本であり、本書では未定義部分の候補を扱う。今回の詳細化は`P02-PARSE`と、その値型・参照・受入設計に限り、他の契約内容の採用や既存仕様の切替を含めない。

## 要件と契約の対応

対象要件は`REQ-001`・`REQ-002`・`REQ-003`・`REQ-004`・`REQ-006`・`REQ-007`・`REQ-008`・`REQ-009`・`REQ-011`・`REQ-012`。

| 契約ID | 担当と参照元 |
|---|---|
| `P02-MODEL` | 識別・時刻・エラーの共通型。[共通管理記録](../BASIC_SPEC.md#共通管理記録の未確定契約) |
| `P02-CLI` | 共通入力・診断・出力。[共通CLI](../BASIC_SPEC.md#共通cliの未確定契約) |
| `P02-PARSE` | 形式解析・スキーマ検証。[構成定義の解析](../BASIC_SPEC.md#構成定義の解析) |
| `P02-STATE` | 対象・三構成・状態判定。[状態判定の優先順位](../BASIC_SPEC.md#状態判定の優先順位) |
| `P02-STORE` | 排他・記録・原子性。[共通基盤](../BASIC_SPEC.md#共通基盤と状態管理) |
| `P02-EXEC` | 外部実行の制御。[外部コマンド](../BASIC_SPEC.md#外部コマンドの未確定契約) |

`P01-CLI`・`P01-CHECK`はD1の[Phase 1](PHASE_01.md#要件と境界)を参照する。Phase 2を使用する後続フェーズは必要な本書の契約IDと節を個別に参照し、共通値を再定義しない。

## P02-MODEL

採用案の共通型を以下に置く。配置候補は`src/core/model.rs`。検証済みの対象・識別・実行型は生成関数で検証し、フィールドを外部から直接変更させない。汎用の`Value`は構文上の値を表し、対象・設定・計画の検証済み型の代わりにしない。

| 型 | 内容・生成条件案 |
|---|---|
| `TargetEvidence` | 実行環境ID、対象ディスクの安定識別子、対象ルート、対象OS識別、観測時点。未取得値は`None`と理由を持ち、実行環境の値で補完しない |
| `PlanId`・`OperationId` | 接頭辞`plan-`・`op-`と32桁の小文字16進数。16バイトのOS乱数を取得し、一意性を確認。乱数取得・照合不能なら失敗 |
| `MachineId` | ServerSeed固有の32桁小文字16進数。構成初回確定時に生成し、OSのmachine-idとは別に保持。復元時の継承は`P04-BACKUP`で審査 |
| `ContractError` | `code: String`、`message: String`、`field: Option<String>`。コードは契約別の固定識別子、表示文は秘匿化済み。内部エラーをそのまま転記しない |
| `Value` | 宣言・数値の表現・所有権は[P02-PARSEの型とAPI](#解析の型とapi案)に集約。担当スキーマの検証前に適用可能とは扱わない |
| `ExecutionContext` | 借用する対象証拠・計画・操作、排他所有者、期限、取消状態。単なるJSONの読み込みだけでは生成しない |

時刻保存はUTCのRFC3339、秒未満3桁、末尾`Z`へ正規化する採用案とする。期限判定は経過時間と絶対期限を分離し、時計後退・再起動後の期限不明は有効と推測しない。乱数の取得手段・OS識別情報との結合は`D02-01`の調査対象であり、識別子生成を実装可能と判定しない。

## P02-CLI

### 入力案

オプション名・既存の操作別必須引数は[CLI](../BASIC_SPEC.md#cli)を参照する。採用案ではオプションをサブコマンドの前後で受理し、`--name=value`と`--name value`を同一視する。boolフラグへの値、未知の短縮形、同名重複、余分な位置引数は入力エラーとする。

`--config`の未指定時は既存の基本設定配置を選び、指定時は絶対パスだけを受理する。欠損・非通常ファイル・読み取り不能は入力エラー。構成未セットアップの既定ファイル欠損だけは、既存の`setup_required`を使用する。設定スキーマは拡張せず、共通設定の候補を独自キーとして追加しない。

`--log-level`指定時はその値、未指定時は基本設定、基本設定も未作成なら[既定値](../POLICIES.md#基本設定の既定値)を使う案。無効な設定をCLI指定で隠さず、必要な設定検証は継続する。非UTF-8パスの入出力表現は`D02-02`に残す。

### 出力案

共通JSONのトップレベルは[既存の枠](../BASIC_SPEC.md#共通json出力)を変更しない。`warnings`・`errors`の要素を`{code, message, field}`とし、`field`は該当しない場合`null`とする案。表示順は検証順、同じコードとフィールドの重複は一件にまとめる。JSONは一オブジェクトと末尾LFのみをstdoutへ出す。

| コマンド | `data`の追加具体化案 | 結果と終了区分の案 |
|---|---|---|
| `status` | `checks`と`pending_operation`。checksはid・結果・理由を持つ | 判定に必要な情報不足は`blocked`、既存区分の安全確認失敗 |
| `verify` | `checks`と対象範囲。各検査をpass・fail・unknownで区別 | failは検証失敗、unknownが必須項目にあれば安全確認失敗 |
| `inspect` | 実行環境と対象の観測情報、取得できない項目の理由 | 項目不足を成功した観測と混同しない。総合結果の優先順はD02-02 |
| `doctor` | 既存の導入状態とsetup_required、個別診断 | 不整合は検証失敗。構成未作成だけは失敗に置換しない |
| `report` | 対象操作IDと読めた結果、欠損した記録 | 未確認を未実行として返さない |
| `version` | [既存binary・applied](../BASIC_SPEC.md#version) | 読めない適用情報は理由を付ける。本体版と取り違えない |

`CommandResponse`は上記共通JSON相当の値と終了区分を所有する型とする。`render(response: &CommandResponse, json: bool) -> Result<Vec<u8>, ContractError>`は出力バイト列を返し、保存は行わない。必須情報が欠けるときのコマンド別の厳密な集約順序・フィールド全集合は`D02-02`の未達とする。

### 調査成果物案

`support-bundle`は`--output`必須、指定先は新規ディレクトリを作らず既存の保存先ディレクトリとする候補。採取上限は100ファイル、合計16 MiB、1ファイル1 MiB、全工程30秒。直近履歴は最大100行とし、不完全な末尾行を黙って成功扱いにしない。ファイル名の識別子はP02-MODEL、時刻はUTCの`YYYYMMDDTHHMMSSmmmZ`とする案。

保存前に全採取項目を検査し、秘匿化の成立性を確認できない項目は[秘密情報](../SECURITY_POLICY.md#秘密情報)へ照合して除外する。保存先が既存なら停止し、dry-run時は採取計画の表示だけとする。tar.gzの内製形式・検証と部分失敗時の出力仕様は`D02-06`へ残し、外部のtar実行物を採用済みとしない。

## P02-PARSE

### 解析の範囲と一次資料

担当要件は`REQ-004`・`REQ-007`・`REQ-008`・`REQ-011`・`REQ-012`。配置案は`src/core/format/`とし、字句・構文・値への変換を内製する。標準機能と再利用対象の確認は[調査表](PHASE_01.md#標準機能の充足調査)を参照する。既存内製パーサー・旧実装はなく、crateの新設・抽出・差し替えは行わない。

照合日は2026-10-20。JSONは[RFC 8259の第2〜9節](https://www.rfc-editor.org/rfc/rfc8259)、YAMLは[YAML 1.2.2の第5〜8章・第10章](https://yaml.org/spec/1.2.2/)へ照合する。本節の数値上限、限定構文、型解決、エラーはServerSeedの採用候補であり、外部標準がその値や制限を要求するとの主張ではない。YAML全構文・標準スキーマとの完全互換は提供範囲に含めない。

解析は借用した1文書のバイト列から所有する値を返す。形式の自動推定、別形式へのフォールバック、ファイル読書き、通信、外部コマンド、環境変数の展開、設定の適用、ログ保存は行わない。呼出元がJSONかYAMLかを選ぶ。同時呼出しは入力・予算・途中状態を共有せず、失敗後の部分値を返さない。署名検証に用いる原バイト列を解析・再生成した値で置き換えない。

### 解析の型とAPI案

次は実装予定の宣言であり、実装コードではない。値型は`src/core/model.rs`、解析型・関数は`src/core/format/mod.rs`、形式固有の処理は同ディレクトリの`json.rs`・`yaml.rs`へ置く候補とする。依存方向は各パーサーから共通値型へ向け、構成別モジュールを参照しない。

```rust
pub(crate) enum Value {
    Null,
    Bool(bool),
    Integer(i64),
    Number(String),
    String(String),
    Array(Vec<Value>),
    Object(std::collections::BTreeMap<String, Value>),
}

pub(crate) enum ParseErrorCode {
    InputLimit, Utf8, Bom, Empty, Syntax, Trailing,
    Escape, Unicode, DuplicateKey, IntegerRange,
    Unsupported, Indent, MissingValue, YamlKeyLimit,
    DepthLimit, ElementLimit, StringLimit, MemoryLimit,
    Timeout, Clock,
}

pub(crate) struct ParseError {
    pub(crate) code: ParseErrorCode,
    pub(crate) byte_offset: usize,
    pub(crate) line: usize,
    pub(crate) column: usize,
}

pub(crate) fn parse_json(input: &[u8]) -> Result<Value, ParseError>;
pub(crate) fn parse_yaml(input: &[u8]) -> Result<Value, ParseError>;
```

入力の寿命に返却値を結び付けず、文字列・数値の字面・配列・objectを返却値が所有する。parse成功の`Value`には本節の構文・上限を適用するが、手作成・変更された値を自動的に検証済みとはしない。objectのキーは復号後のUTF-8バイト列で比較し、Unicode正規化・大小文字変換をしない。ソース上の順序をobjectの意味へ含めず、配列の順序は保存する。

### 解析の入力と上限

既定値として緩和できない固定候補を次に定める。MiBは1,048,576 bytes、KiBは1,024 bytesとする。設定・CLIから解析上限を変更する入口は追加しない。

| 対象 | 算定と境界案 |
|---|---|
| 入力 | 借用slice全体が1 MiB以下。空白・改行・コメント・escapeの元表現も含む。超過は構文解析前に拒否 |
| 文字コード | UTF-8だけ。先頭BOMを拒否し、不正UTF-8を置換文字へ修復しない。空白だけ・YAMLのコメントだけは文書なしとして拒否 |
| 深さ | root値の深さ1、配列要素・objectの値は親+1。最大32。キーは深さを増やさない。空collectionも1段として数える |
| 要素数 | rootを含む全Valueと、objectの各キーをそれぞれ1件として合計10,000以下。空collectionは1件。コメント・空白・区切りは数えない |
| 文字列 | 復号したUTF-8の長さが1個64 KiB以下。キーも対象。元のescapeの長さは入力上限へ、復号後の長さは本上限へ算入 |
| 数値字面 | Numberへ保持する字面は入力の該当範囲そのもの。単独の追加上限は設けず、入力・下記予算の双方で制限 |
| YAMLキー | 区切りcolonを除くキーのソース表現が1,024 Unicode scalar values以下。引用符・escapeの各文字も数え、ASCIIの単純キーにも適用 |
| 時間 | 関数入口から最終結果確定前までの経過が1秒未満。1秒以上を検出した呼出しは成功を返さない。I/Oは含まない |
| 計算上のメモリー予算 | 下記Mが32 MiB以下。実ヒープ・RSSの上限保証とは区別する |

計算上の予算は`M = I + S + R + 128*N + 1024*D + 8192` bytesとする。Iは入力slice長、Sは生成した全キー・文字列の復号後長の累計、Rは保持する数値字面長の累計、Nは上表の要素数、Dは現在までの最大深さ。いずれも削除・巻戻しで予算を戻さず、checked arithmeticの失敗もMemoryLimitとする。ノード・文字列の生成前に予定値を検査し、超過分を構築してから捨てる方式にしない。

Mは型やallocatorに左右されない論理計数であり、BTreeMapの内部割当、Vec/Stringの余剰capacity、allocator管理領域、実行stack、Rust実行基盤を実測した値ではない。実使用量の32 MiB保証に読み替えず、実ヒープ・stackの計測方法、実割当失敗時の保全と対象環境での検証は`D02-03`の未達へ残す。通常の入力上限が先に働く場合に、メモリー超過入力が存在したと捏造しない。予算計算の境界は計数器単体の受入で照合する。

時間計測には[Instant](https://doc.rust-lang.org/std/time/struct.Instant.html)を用いる案とする。入口、標準機能による全文UTF-8確認の前後、各トークンの開始前、内製処理で最大4,096入力bytesを走査するごと、値の返却前に期限を検査する。内製の文字列比較等も4,096 bytes以下の区間で検査する。経過時間の算定不能・逆転検出はClock。成功直前の検査も省略せず、タイムアウト後の自動再試行は0回とする。標準機能の一括処理内やOSによる休止・割当待ちを強制中断する方式ではなく、1秒で必ず制御が戻るリアルタイム保証はしない。

### JSONの受理契約案

JSONではobject・array・string・number・小文字のtrue/false/nullを1値だけ受け付ける。値の前後と構造記号間の空白はASCII space・HT・LF・CRのみ。末尾の空白後に内容が残ればTrailingとする。コメント、末尾comma、単引用文字列、引用のないキー、欠損値を受理しない。

文字列は二重引用とし、rawのU+0000〜U+001Fを拒否する。escapeは引用符、backslash、slash、b・f・n・r・t、uと4桁hexの組だけを受け付ける。hexの大小は問わない。high surrogateは直後のu escapeのlow surrogateと対になる場合だけ1 scalarへ復号し、単独・順序逆転・範囲不正はUnicodeとする。escapeの不明種別・桁不足・非hexはEscape。raw UTF-8とescapeの同じ復号結果は同一のキーとして照合する。

numberは全体が`-?(0|[1-9][0-9]*)(\.[0-9]+)?([eE][+-]?[0-9]+)?`に一致する字面だけを受け付ける。この記法は文法の説明で、外部の正規表現エンジンを導入する意味ではない。先頭plus・不要な先頭zero・小数点や指数の桁欠損・NaN・InfinityはSyntax。小数部も指数部もなければi64の範囲へ検査してIntegerとし、範囲外はIntegerRange。`-0`はInteger(0)とする。小数部または指数部があればNumberに元の字面を保持し、f64への変換・丸め・指数展開をしない。例えば`1e400`の構文受理は、下流で有限の計算値として使用できるとの認定ではない。

数値の終了は空白・comma・閉じ括弧・EOFに限り、妥当な数値の直後にそれ以外のbyteが続く場合も数値のSyntaxとする。`01`をInteger(0)と余分なrootへ分離せず、offset=1で拒否する。数値以外のrootが完成した後に別内容がある場合はTrailingを使用する。

### YAMLの構造契約案

YAMLの限定形式は次の全条件を満たす1文書とする。ここで行とは元入力のLFまたはCRLFで区切る行であり、最終行の改行なしも認める。単独CR、rawのHT、rawの制御文字を拒否する。通常のraw文字はU+0020〜U+007E、U+00A0〜U+D7FF、U+E000〜U+FFFD、U+10000〜U+10FFFFとし、改行は上記の区切りだけを使う。

1. rootの有効行は字下げ0。block mapping・block sequence、1行scalar、空collectionを扱う。空白行・コメントだけの行は構造を変えない。
2. blockの子は親の基準位置からspace2個だけ深くし、兄弟は同じ位置へ揃える。深さ飛越し、奇数の字下げ、既存の親位置へ一致しないdedent、scalar後の子行はIndentとする。
3. mappingは1行のstringキーとcolonで始める。colonの直前にspaceを置かず、直後は1個以上のspaceまたは行末。キーは`[A-Za-z_][A-Za-z0-9_-]*`のplain、または下記の引用文字列とし、空の引用キーは許容する。複合キー・explicit keyは対象外。
4. sequenceは`-`とspace1個の後に値を書くか、`-`だけの行の次へ子を置く。inlineのmapping開始`- key: value`も認め、そのmapping基準位置はsequenceの基準+2とする。後続のキーはその位置、キーの子はさらに+2。inlineでさらにsequenceを始める省略形は認めない。
5. colon・単独の`-`の後に値がなければ、次の有効行に上記位置の子を必須とする。子がない場合はMissingValueとし、暗黙のnullや空文字へ変換しない。nullは明示する。
6. flowの例外は空配列`[]`と空object`{}`の完全一致だけとする。非空flow、括弧内のspace・コメントはUnsupported。配列・objectを表す必要がある場合はblockを使う。
7. 引用外の`#`は行頭または直前がspaceの場合だけ行末コメントを開始する。値とコメントの間の末尾spaceを除き、scalar内容の内部spaceを保存する。`abc#def`内の`#`と引用内の`#`は内容。
8. directive、文書区切り`---`・`...`、タグ、anchor・alias、block scalar、引用scalarの改行継続、merge keyを受理しない。構文として現れた場合はUnsupported。引用文字列中の同じ記号は内容であり、objectキーの復号結果`<<`だけは引用の有無を問わず拒否する。plain scalarの複数行折畳はせず、その後の子行は手順2のIndentとする。

同じblock位置にmappingとsequenceの項目を混在させた場合はSyntax、完成したroot scalarの後の別の有効行はTrailingとする。quoteが閉じないまま改行へ達した場合はその改行位置でUnsupported、EOFへ達した場合はEOFでSyntaxとする。

### YAMLのscalarと型案

| 表記 | 復号・型・拒否条件 |
|---|---|
| single quote | 同じ行の対応するquoteで閉じ、内部のquote2個を1個へ復号。backslashはそのまま保持 |
| double quote | 同じ行で閉じる。JSONの単純escapeとu4桁、およびU8桁hexを認める。u/Uの値は単独でUnicode scalarの範囲にあることを要求し、surrogate値を拒否。YAMLではJSON型のsurrogate対を結合しない。その他のYAML 1.2.2で有効なescapeはUnsupported、不明なescape種別・hex不正・桁不足はEscape |
| plainの範囲 | 空でなく1行。先頭は`! & * [ ] { } # , : ?`、quote、backtick、percent、at-sign、block scalar記号を禁止。先頭hyphenは次がspaceまたは行末ならsequence扱い。内部の`[ ] { } ,`とquote・backtickを禁止。colonは直後が非spaceの内容文字のときだけ許容し、URLの`http://`を保持。引用が必要な値を自動補正しない |
| 明示型 | 引用しない完全一致の`true`・`false`はBool、`null`はNull。JSONと同じnumber文法は上記のInteger/Number規則へ接続 |
| その他のplain | String。`yes`・`no`・`True`・`~`・日付・`01`・`+1`・`0x10`・`.nan`は暗黙変換しない。引用した数値・true/nullもString |

scalarに関係する構文の優先順は、引用、空collection、禁止された構文、明示型、その他plainとする。quoteの閉じ後はspaceとコメントまたは行末だけを認め、別の値の連結を拒否する。キーは常にStringとし、値用の暗黙型解決を適用しない。復号したキーの重複は、1個目と同じ値でもDuplicateKeyとする。

### 解析のエラーと位置案

ParseErrorは1個だけ返す。入力上限、全文UTF-8、先頭BOM、YAMLの全文文字・改行条件、左からの構文・値構築、末尾の順で確認する。前段の失敗が後段より優先し、後段のエラーを得るために解析を続けない。各確認点で期限を先に照合し、同時に検出する制限はMemoryLimit、DepthLimit、ElementLimit、StringLimit、YamlKeyLimitの順とする。文法から分類が一意なものは下表の専用分類を使い、それ以外だけSyntaxとする。

| codeの値 | ParseErrorCode・検出対象・位置 |
|---|---|
| `parse.input_limit` | InputLimit。入力上限を初めて超えるbyteの位置 |
| `parse.utf8` | Utf8。[from_utf8](https://doc.rust-lang.org/std/str/fn.from_utf8.html)のvalid_up_toに対応する位置 |
| `parse.bom` | Bom。先頭の位置0 |
| `parse.empty` | Empty。有効値なし。EOF |
| `parse.syntax` | Syntax。要求された構文に合わない最初のbyte。必要なbyteなしならEOF |
| `parse.trailing` | Trailing。root後に残る最初の非空白内容。YAMLの許可されたコメントは除く |
| `parse.escape`・`parse.unicode` | Escape/Unicode。問題のescapeのbackslash。rawの不正UTF-8は前段のUtf8 |
| `parse.duplicate_key` | DuplicateKey。2個目のキーの開始位置。quote込みのソース開始 |
| `parse.integer_range` | IntegerRange。符号を含む数値の開始位置 |
| `parse.unsupported` | Unsupported。対象外構文の開始位置。merge keyならキーの開始位置 |
| `parse.indent` | Indent。異常な有効行の最初の非spaceの位置 |
| `parse.missing_value` | MissingValue。子を持たないcolonまたは単独hyphenの位置 |
| `parse.yaml_key_limit` | YamlKeyLimit。長過ぎるキーの開始位置 |
| `parse.depth_limit`・`parse.element_limit` | DepthLimit/ElementLimit。超過する値またはキーの開始位置 |
| `parse.string_limit` | StringLimit。超過するstringの開始位置。quoteを含む |
| `parse.memory_limit` | MemoryLimit。予定計数が超過する値・キー・文字列の開始位置。入口の算定なら位置0 |
| `parse.timeout`・`parse.clock` | Timeout/Clock。検査時点の走査位置。走査前は0、返却前はEOF |

byte_offsetは原入力の先頭を0とするusizeで、EOFは入力長。line・columnは1始まりとし、columnはその行の先頭からのbyte数+1である。LFで次行へ進み、CRLFを1改行として扱う。JSONの単独CRも1改行とし、YAMLの不正な単独CRではその位置を指す。不正UTF-8の位置も有効UTF-8への変換後の文字数で数えない。位置を求めるために別の予算なし全文コピーを作らない。

呼出元のContractErrorへの変換ではcodeをそのまま固定文字列に対応付け、messageは固定の分類名と数値の位置だけ、fieldはNoneとする。キー名・値・入力断片・ファイル内容・パスをエラーへ含めない。CLIのトップレベル枠・終了区分の選択は[P02-CLI](#p02-cli)と操作別契約が担当し、パーサーが終了コードを返したりprocessを終了させたりしない。

### スキーマ検証との境界案

解析は未知キーを消去せず、null・空文字・型・配列順序をそのまま返す。必須項目、未知キー、設定版、許容値・既定値・互換性は担当スキーマが後段で検証し、型の不一致をstringからinteger/boolへ再変換しない。構文として妥当な値を、既存の設定・マニフェストがすべて受理するとは扱わない。

既存候補`validate(value: &Value, schema_id: &str) -> Result<(), ContractError>`のschema_id全集合・版・定義構造・検証順序と、各形式の移行は`D02-03`に残る。今回そのAPIや三構成の設定形式を確定せず、構文解析の詳細化をスキーマ検証まで完成した根拠にしない。JSON生成の既存候補はobjectキーをUTF-8バイト順とする。その他の出力byte・丸め・escapeは[P02-CLI](#p02-cli)の未達へ対応付け、解析成功から補完しない。

### 解析の内製化と差し替え案

標準機能で再利用する範囲はUTF-8確認、文字・整数の基本操作、集合、所有する文字列と経過時間。字句・構文・上限計数・位置・エラーの評価は新規内製で、外部crateや外部パーサー実行物を必要とする案にはしない。難易度は限定解析でも中、Unicode・時間・実割当を含む敵対入力の保全検証は高として評価し、未実証を低難易度へ移さない。

将来の抽出・差し替えは本節の宣言・Valueの意味・エラーと位置・受理集合を共通契約とし、[解析の受入ケース](#解析の受入ケース案)を新旧へ同じ入力で適用する。受理集合や型の変更、保存済み設定の再解析、共通CLIへの変換が変わる場合は互換変更と決めつけず、移行と復旧可能範囲を別途審査する。初回のため旧実装・旧crate・除去対象はなく、新旧比較結果も存在しない。

### 解析の受入ケース案

表内の`\n`・`\r`・`\t`は、特に「字面」と記した箇所以外ではテスト入力のLF・CR・HTを表す。JSON/YAMLのescapeを示すbackslashは原入力の文字とする。反復表記は指定した文字列を正確な回数だけ連結する入力生成規則であり、外部fixtureのコピーではない。全ケースは文書上の設計で、内製パーサーの実行結果ではない。

| ケースID | 前提・入力案 | 期待結果・合否の照合 |
|---|---|---|
| `A02-11` | JSON `{"b":[true,null],"a":-0}` | objectにa=Integer(0)、b=Array(Bool(true),Null)。入力と外部状態は不変 |
| `A02-12` | JSON `null`、`[]`、`{}`を別呼出し | Null、空Array、空Objectを区別。rootの型をobjectへ補完しない |
| `A02-13` | JSON `{}`の後にspaceを入力上限ちょうどまで追加。同じ入力へspace1 byte追加 | 前者成功、後者InputLimit・offset=入力上限。コメントや空白も数える |
| `A02-14` | JSONで`[`を31個、`0`、`]`を31個。括弧を各32個に変更 | 前者の最大深さは32、後者はDepthLimit。閉じ括弧を追加して回避不可 |
| `A02-15` | JSON配列にnullを9,999個、次に10,000個。別入力で一意なstringキーとnull値を4,999組・5,000組持つobject | 配列のroot込み10,000件とobjectの9,999件は成功。それぞれ超過側はElementLimit |
| `A02-16` | JSON引用内にASCII aを文字列上限ちょうど・+1 bytes | 前者成功、後者StringLimit。キーにも同じ上限を適用 |
| `A02-17` | JSON引用内にu0061 escapeを文字列上限の回数だけ反復 | 復号後の長さで受理し、入力長との二重の計数を照合 |
| `A02-18` | JSONのi64最小・最大値と、その範囲を1超える正負の値 | 範囲内はInteger、範囲外はIntegerRange・数値開始位置。丸めなし |
| `A02-19` | JSON `1.00`、`1e400`、`-0.0` | Numberへ各字面を保持。f64化やIntegerへの縮約なし |
| `A02-20` | JSON `01`、`+1`、`1.`、`1e`、`NaN`、`Infinity` | いずれもSyntax。順にoffset=1・0・2・2・0・0を照合 |
| `A02-21` | JSONのquote内にuD834・uDD1Eの連続escape、単独uD834、単独uDD1E | 対だけ1 scalarへ復号。単独はUnicode・backslash位置 |
| `A02-22` | JSON `{"a":0,"\u0061":1}` | DuplicateKey・offset=7、line=1、column=8。値一致でも同じ失敗 |
| `A02-23` | JSON `"é"`の後にspaceとx。別入力でNFC/NFDの異なるキー | 前者Trailing・offset=5、column=6。後者の異なるUTF-8列を自動統合しない |
| `A02-24` | BOM付きJSON、byte FF、空入力、spaceだけ | Bom・Utf8・Emptyを区別。いずれも部分値・外部出力なし |
| `A02-25` | 不正構文の後に不正UTF-8。別入力で入力上限超過とBOMを併存 | 前者Utf8、後者InputLimit。前段検証の優先順を照合 |
| `A02-26` | JSON `[0,]`、`/*x*/null`、`null null`、raw LFを含む引用文字列、qの未知escapeを含む引用文字列 | 順にSyntax(offset=3)、Syntax(0)、Trailing(5)、Syntax(raw LF位置)、Escape(backslash位置) |
| `A02-27` | YAML `apps:\n  - name: example\n    enabled: true\n`。同じ内容をCRLF化 | 同じObject/Array/String/Bool。inline mappingの継続位置が一致 |
| `A02-28` | YAML `apps: []`、`mapping: {}`。別入力で`[ ]`・`[1]`・`{a: 1}` | 空collectionだけ受理。他はUnsupportedでblockへ黙って変換しない |
| `A02-29` | YAML `a: yes\nb: "true"\nc: null\nd: 01\ne: 1.5\nf: 2026-10-20` | a/b/d/fはString、cはNull、eはNumber。日付・8進・boolの推測なし |
| `A02-30` | YAML `a: http://127.0.0.1/x#part # note\nb: 'it''s'` | URL内のcolon・hashを保持し末尾コメントを除外。bはString("it's") |
| `A02-31` | YAML double quoteのu0041、U0001D11E、uD834、qの各escape。single quote内のbackslash | 順にA・U+1D11Eへ復号、Unicode、Escape。single quoteのbackslashは不変 |
| `A02-32` | YAMLのodd indent・子の深さ飛越し・raw HT・単独CR・scalar後の子 | 順にIndent・Indent・Syntax・Syntax・Indent。raw HT/CRは全文文字検査で先に拒否 |
| `A02-33` | YAML `a:\nb: 1`、単独hyphen、空文書、コメントだけ | 前2件はMissingValue、後2件はEmpty。省略値をnullへ変換しない |
| `A02-34` | YAML tag・anchor・alias・merge key・directive・文書区切り・block scalar | Unsupported。引用内の同記号は内容として保持し、キー`<<`だけは拒否 |
| `A02-35` | YAML `a: 0\n"a": 1`。quoted keyのソース長1,024・1,025 scalar values | 重複はDuplicateKey・offset=5、line=2、column=1。長さ超過はYamlKeyLimit |
| `A02-36` | 空string・null・未知キー・型の違う値を構文解析 | 値を保持して成功。担当スキーマの拒否・既定値検証は別で、適用開始なし |
| `A02-37` | 計数器のMが予算ちょうど・+1、算定の整数overflowを単体で与える | ちょうどを許容、超過/overflowはMemoryLimit。実RSS保証の試験とはしない |
| `A02-38` | 差し替え可能な試験時計で、返却前の経過を1秒未満・ちょうど・超過にする | 未満だけ成功、残りはTimeout。実時間のsleepで境界を曖昧にしない |
| `A02-39` | 時計の逆転・計測不能、同時2呼出し、エラーに秘密値を含む入力 | Clock・予算独立を照合。エラーは分類と位置だけで入力断片を含めない |
| `A02-40` | 現存する基本仕様とサーバ構成のYAML形式例、基本仕様のJSON形式例 | 解析値の型・配列・文字列を期待木と照合。通常パーサーの成功を内製試験結果にしない |

境界ケースの計数器・試験時計は、将来のunit testで内部へ注入する試験条件であり、製品のCLI・設定で上限や時刻を改変する入口ではない。位置と返却値の一致、部分値の不在、外部状態の不変を全ケースの合否へ含める。実割当・時間制約の対象OS・両CPUでの実証と、担当スキーマによる統合試験は未実施とする。

## P02-STATE

`detect(target: &TargetEvidence, records: &Value) -> Result<Value, ContractError>`は[既存の判定順](../BASIC_SPEC.md#状態判定の優先順位)を変えず、採用案では各判定について根拠の出所・対象・未取得理由を返す。状態スナップショットを入力の事実より優先しない。

初回の構成選択結果は、利用者の対話結果、対象証拠、計画IDに結合する値としてメモリー上で保持する案。設定ファイルだけで選択成立とはしない。変更開始直前に再照合し、保存は最終構成検証後にP02-STOREへ渡す。保存された構成の値と三構成の適用範囲は[各正本](../POLICIES.md#構成選択と継続管理)へ参照する。

構成リビジョンの旧二択形式移行、初回選択結果を媒体・SSHへ渡す形式は`D02-03`と各入口の契約へ残す。既存の版番号を変更・互換認定しない。

## P02-STORE

### 排他と確定案

変更の親プロセスが既存の共通ロックを一度取得し、子処理は借用する`ExecutionContext`を使う案。`File::try_lock`を候補とし、競合時は待ち時間0で停止する。ロックファイルをunlink・renameして所有者をすり替えない。読み取り専用ではファイルもロックも作成しない。

プロセス識別にはPIDだけでなくOS起動識別子とプロセス開始情報を対応付ける案。子への引き継ぎ方式・子だけ残った場合のロック寿命は`D02-04`で確定するまで実装しない。

管理ファイルの置換は[原子性](../BASIC_SPEC.md#原子性と書き込み)に従い、親ディレクトリの同期と書き込み先の同一性確認も候補に含める。履歴は操作IDごとの重複を照合し、部分的な追記を成功記録としない。追記型履歴と複数ファイル確定の具体方式は`D02-05`で審査し、通常のrenameだけでトランザクション成立を保証しない。

### 記録とAPI案

`version`は既存の製品バージョン情報のJSONオブジェクト、`last-success`・`last-failure`は操作ID・終了時刻・結果・対象IDを持つJSONオブジェクトとする候補。現在状態の置換と製品の追記型実行履歴を分ける。管理権限・秘密情報は[セキュリティポリシー](../SECURITY_POLICY.md#ファイル権限)へ参照し、新たな権限規定を本書で作らない。

| 宣言案 | 制約 |
|---|---|
| `acquire(target: &TargetEvidence) -> Result<OperationGuard, ContractError>` | guardはClone不可、対象・所有プロセスと結合。取得失敗後の変更なし |
| `begin(guard: &OperationGuard, plan: &Value) -> Result<ExecutionContext, ContractError>` | 既存の操作状態を記録できた場合のみ返す |
| `finish(ctx: &mut ExecutionContext, result: &Value) -> Result<(), ContractError>` | 必須記録の確定に失敗したら成功を返さない。失敗記録も書けない場合はstderrと残存証拠を使い復旧待ち |

ログの採用案は、操作ID・対象ID・時刻・level・イベントID・秘匿化済みメッセージをJSONLへ記録する方式。保持・ローテーションは10 MiB×5世代を候補とするが、必須操作記録をログ整理で消去しない。履歴容量の保全・退避と操作上限の関係は`D02-05`の未決事項。

## P02-EXEC

配置案は`src/platform/command.rs`。`Command`の引数配列と`Instant`を再利用し、呼び出しごとに操作契約の制限値を必須入力とする。外部実行物の版・パス・引数・例外IDの照合なしに起動しない。

| 入力 | 採用案の型・範囲 |
|---|---|
| 実行物・引数 | 絶対パス、`Vec<OsString>`。シェル文字列を受けない。環境変数は必要な固定集合だけ |
| 時間 | timeout_msは1〜3,600,000、終了待ちgrace_msは1〜30,000。未指定・0・範囲外を拒否 |
| 出力 | stdout・stderr各1 MiBまで保持。並行して読み、超過時は失敗へ移る。末尾だけ捨てて成功にしない |
| 再試行 | 共通実行器は自動再試行0回。呼出元だけが回数・待機・再照合を定義できる |
| 取消 | 起動前は起動しない。起動後は対象契約の保全・停止へ移り、単に接続を閉じたことを終了とみなさない |

`run(spec: &Value, ctx: &ExecutionContext) -> Result<Value, ContractError>`は終了状態・出力バイト数・制限超過・経過時間・停止確認を返す案。標準入力の配送量上限、子孫停止の実現方式、ブロッキングI/Oの中断と親停止時の保全は未確定。`Child::kill`だけでは全条件を満たせず、新しい外部ツール・FFIを暗黙に追加しない。

## 受入ケース案

| ケースID | 区分・入力 | 期待結果と確認方法 |
|---|---|---|
| `A02-01` | 正常・異常。整合対象、対応外OS、読み取り不能 | 基本仕様の優先順と根拠を照合。キャッシュのみでMANAGEDとしない |
| `A02-02` | 異常。未選択、対話不能、--yes、構成不一致 | 選択・保存を代行せず停止。ファイル差分なし |
| `A02-03` | 境界。入力1 MiB・1 MiB+1、深さ32・33 | 境界内を構文検査、超過を拒否。設定適用0回 |
| `A02-04` | 異常。重複キー、型不正、未知キー、YAML alias | 誤った値の自動補完・コード評価なし。位置と分類を返す |
| `A02-05` | 異常。重複オプション、無効config、CLIログ値 | 入力エラー。CLI値による不正設定の隠蔽なし |
| `A02-06` | 正常。読み取り専用で中断記録を検出 | メモリー上の診断だけ。管理ディレクトリ・ログの不変 |
| `A02-07` | 異常。別プロセスのロック、PID再利用 | 新変更なし。PID一致だけで所有者・完了を認定しない |
| `A02-08` | 異常。記録途中の容量不足・同期失敗 | 成功を返さず、実適用位置の照合待ち。無承認巻戻しなし |
| `A02-09` | 境界。時間・出力量の上限と直後の超過 | 所定の失敗分類。子孫終了と記録確定を別に照合 |
| `A02-10` | 異常。秘密項目混在、既存保存先、dry-run | 秘匿化不能項目を採取せず、上書き・保存の迂回なし |

## 未決事項と成立判定

| 判断ID | 採用前に必要な判断・調査 | 影響 |
|---|---|---|
| `D02-01` | 識別子・時刻の正確な型、OS乱数・起動情報の取得と復元時の同一性 | 全変更の対象照合 |
| `D02-02` | CLIの全受理文法、コマンド別data全集合・集約順、非UTF-8の表現 | 全CLIと骨格 |
| `D02-03` | [解析の詳細案](#p02-parse)の採用審査、実ヒープ・stackの計測・割当失敗の保全と対象環境での上限実証、担当スキーマ全集合・版・検証順序と三構成の移行契約 | 全設定・状態・マニフェスト |
| `D02-04` | 子孫を含む停止・排他引継ぎ・親停止の実現可能性とOS外部契約 | 全実行・中断復旧 |
| `D02-05` | 複数記録と追記履歴の確定方式、保存形式全集合、保持容量・失敗時照合 | 全変更の成功判定 |
| `D02-06` | tar.gz・HTTPSの内製境界・正式仕様・互換試験。形式解析の候補はP02-PARSEを参照し、その実証不足はD02-03で管理 | 調査成果物・取得処理 |

本書の内製候補は形式解析・対象判定・識別・記録調整・実行制御。モジュール新設案とし、初期からcrateへ抽出しない。難易度は[調査表](PHASE_01.md#標準機能の充足調査)へ対応付ける。旧実装は存在せず、差し替え時の共通ケースは上表を拡張して契約ごとに固定する。移行・抽出・旧コード除去の具体的変更は今回の対象外とする。

候補値の採用、公開型の完全な宣言、外部実行物の個別情報、非機能要求・試験条件の確定が残るため、Phase 2と依存範囲は完全仕様未成立。本文案の存在を実装着手可能の根拠にしない。
