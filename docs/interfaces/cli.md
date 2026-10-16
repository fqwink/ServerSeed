# CLI仕様

[文書一覧](../DOCUMENT_INDEX.md)

## コマンド

共通コマンド名は`serverseed`とする。ServerSeedの初期提供形態はCLIであり、すべての主要操作は`serverseed`サブコマンドとして提供する。

```bash
serverseed inspect
serverseed setup
serverseed update
serverseed recover
serverseed verify
serverseed report
serverseed status
serverseed version
serverseed doctor
serverseed plan
serverseed support-bundle
serverseed self install
serverseed self update
serverseed self uninstall
```

| コマンド | 役割 |
|---|---|
| `inspect` | OS、ストレージ、BIOS、USBおよび環境を検査 |
| `setup` | 対応環境を初期構築 |
| `update` | 管理対象サーバを更新 |
| `recover` | 起動修復または設定復元 |
| `verify` | セキュリティ、サービスおよびデータ保護状態を検証 |
| `report` | 実行結果と履歴を表示 |
| `status` | 現在の総合状態を表示 |
| `version` | ServerSeed、OSおよび構成バージョンを表示 |
| `doctor` | ServerSeed自身の導入状態、設定、権限、依存コマンドを診断 |
| `plan` | 変更操作の実行計画だけを作成 |
| `support-bundle` | 調査用情報を秘密情報を除外して収集 |
| `self` | 本体だけの導入・更新・削除 |

実行環境別の起動・導入手順は[USB導入](../provisioning/usb.md)と[Cloud導入](../provisioning/cloud.md)を参照する。破壊的操作の確認は[確認方式](#確認方式)に従う。

### コマンド別の変更可否

| コマンド | 既定動作 | 変更操作 |
|---|---|---|
| `inspect` | 読み取り専用 | なし |
| `status` | 読み取り専用 | なし |
| `version` | 読み取り専用 | なし |
| `report` | 読み取り専用 | なし |
| `verify` | 読み取り専用 | なし |
| `doctor` | 読み取り専用 | なし |
| `plan` | 読み取り専用 | なし |
| `support-bundle` | 読み取り専用 | なし |
| `setup` | 計画表示後に確認 | あり |
| `update` | 計画表示後に確認 | あり |
| `recover` | 復旧計画表示後に確認 | あり |
| `self install`、`self update`、`self uninstall` | 本体変更計画を表示後に確認 | あり |

`verify`は状態確認と検証のみを行い、設定修正、パッケージ更新、サービス再起動を行わない。修正が必要な場合は、必要な操作をレポートする。

### `self`

`self`は実行中OSの本体を管理する。対象ディスクへのOS導入は行わず、USBセットアップでは同じ導入処理を新OS内で呼び出す。各操作の状態は[本体導入と構成管理](../common-platform.md#本体導入と構成管理)、配置手順は[バイナリ導入](../development/implementation.md#バイナリ導入)に従う。

| 操作 | 入力と条件 |
|---|---|
| `self install` | ローカルの検証済み本体から実行する。`--source DIR`、`--release-version VERSION`、`--keyring PATH`を必須とする。本体未導入、または導入情報が一致する同一版だけを受け入れる |
| `self update` | `--source DIR_OR_HTTPS_URL`と`--release-version VERSION`を必須とする。整合する導入情報が必要で、管理構成の有無にかかわらず本体だけを更新する |
| `self uninstall` | 導入済み本体の配置情報から削除対象を確定する。削除条件は[本体だけの削除](../development/implementation.md#本体だけの削除)に従う |

`--source`は[成果物一式](../development/ci-cd.md#リリース)を置くローカルディレクトリまたはHTTPSの基点URLを指定する。`--release-version`はタグの先頭`v`を除いたバージョンを指定し、マニフェスト・本体との不一致を拒否する。`--keyring`は初回に外部確認した公開鍵リングであり、秘密鍵を受け取らない。導入済み本体の更新・復旧は保存済みの信頼鍵を使用する。

通常の`self`変更操作は`UNKNOWN`、`EMPTY`、`RECOVERABLE`では実行しない。管理対象かどうかによる構成の扱いは[共通基盤](../common-platform.md#本体導入と構成管理)を参照する。

`--dry-run`では、マニフェストの読み取り・検証とローカルの前提検査までを行う。HTTPSのメタデータはメモリ上で処理し、バイナリの取得・配置、一時ファイルや管理ディレクトリの作成を行わない。`plan self install`、`plan self update`、`plan self uninstall`も同じ入力を受け付ける。

構成セットアップ前の診断では、設定・構成未作成を`setup_required`として示し、構成値を補完しない。本体・導入情報・権限の不整合は失敗として区別する。初回取得が未完了の段階はシェルからの検証手順を使用し、未導入の`serverseed`を呼び出す前提にしない。

### `inspect`

`inspect`は対象環境の事実を収集し、推奨状態を判定する。外部通信、パッケージ更新、設定変更は行わない。

収集対象は次を基本とする。

- OS情報
- カーネル
- systemd状態
- ストレージ情報
- ファイルシステム
- ブート方式
- ネットワーク到達性
- SSH設定
- Docker導入状態
- ServerSeed管理情報

### `setup`

`setup`は`EMPTY`または`ADOPTABLE`に対して実行する。

環境別の処理は[USBのセットアップ](../provisioning/usb.md#セットアップ)と[Cloudの初期セットアップ](../provisioning/cloud.md#初期セットアップ)に従う。

### `update`

`update`は`MANAGED`に対して実行する。`UNKNOWN`、`EMPTY`、`ADOPTABLE`に対しては実行しない。`RECOVERABLE`に対しては、復旧が必要であることを表示して停止する。

更新前検査に失敗した場合は変更を開始しない。更新開始後に失敗した場合は失敗状態を記録し、後続処理を停止する。

`--source`と`--release-version`を指定する場合は[self](#self)と同じ定義を使用し、両方を指定する。両方を省略した場合は[候補の固定](../operations/updates.md#候補の固定)に従う。`plan update`も同じ入力を受け付ける。

### `recover`

`recover`は`RECOVERABLE`に対して実行する。復旧内容は、起動修復、設定復元、前回操作の後始末、レポート生成に限定する。

データを削除する復旧、OSパッケージの無条件なダウングレード、バックアップ未確認の復元は行わない。

`recover --self`は[本体の置換と復旧](../operations/updates.md#本体の置換と復旧)の退避・中断記録を対象とし、構成やOSの復旧と区別する。配置本体を実行できない場合は、別途検証した同じバージョンまたは互換性のある復旧用バイナリを絶対パスで実行する。取得元を変更する場合も署名・互換性検証を省略しない。

初回導入の中断で保存済みキーリングがまだ存在しない場合だけ、初回と同じ独立確認を済ませた`--keyring PATH`を要求する。保存済みキーリングがある場合の差し替えや、検証失敗を回避する指定には使用しない。

### `report`

`report`は直近の操作、検査結果、検証結果、失敗理由、再起動要否、次に必要な手動操作を表示する。

人間向け出力では要約を優先する。JSON出力は[共通JSON出力](#共通json出力)に従う。

### `version`

`version --json`の`data.binary`には、実行本体の`serverseed_version`、`architecture`、収録した`config_revision`・`migration_revision`、および`compatible_config_revision`・`compatible_migration_revision`を出力する。互換範囲は[本体マニフェスト](../operations/updates.md#更新マニフェスト)と照合する。

対象OSへ適用済みの情報は`data.applied`として分離し、構成未セットアップなら`null`とする。実行本体の自己申告を署名検証の代わりにしない。

### `doctor`

`doctor`はServerSeed自身の導入状態を診断する。

診断対象は次を基本とする。

- `serverseed`バイナリの存在と実行権限
- 本体導入情報と配置本体のハッシュ・バージョンの一致
- Ubuntu 24.04 LTS判定
- 対応アーキテクチャ判定
- 必須外部コマンドの存在
- `/etc/serverseed/`、`/var/lib/serverseed/`、`/var/log/serverseed/`の存在と権限
- 設定ファイルの構文と型
- 管理マーカーの整合性
- ロックファイルと実行中操作の状態
- 信頼鍵リングの存在
- ログ書き込み可否

`doctor`は修正を実行しない。修正候補がある場合は、必要な操作と影響範囲を表示する。

構成未選択の扱いは[self](#self)に従う。診断結果の`data`に`installation_status`と`setup_required`を含める。前者は`absent`、`installed`、`removed`、`inconsistent`、`pending`、後者はbooleanとする。

`doctor --transaction OPERATION_ID --json`は、本体変更処理が呼び出す読み取り専用の診断モードとする。root権限、保持中の共通ロック、実行中操作・トランザクションのID一致、署名付き候補の検証と実配置ハッシュの一致を確認し、置換直後の導入情報が未確定でも`pending`として候補を検査する。ロック保持プロセスとその診断子プロセスの対応も確認し、条件不成立を通常診断への無条件な成功扱いにしない。このモードは状態や導入情報を確定・修正しない。

### `plan`

`plan`は`setup`、`update`、`recover`、`recover --self`および[self](#self)の実行計画だけを作成する。

```bash
serverseed plan setup
serverseed plan update
serverseed plan recover
serverseed plan recover --self
serverseed plan self update --source DIR_OR_HTTPS_URL --release-version VERSION
```

`plan`は`--dry-run`と同様に変更操作を行わない。計画作成に必要な検査だけを行い、`blocked_by`がある場合は実行不能理由を表示する。

### `support-bundle`

`support-bundle`は障害調査に必要な情報を収集する。

収集対象は次を基本とする。

- `serverseed version --json`
- `serverseed status --json`
- `serverseed inspect --json`
- `serverseed doctor --json`
- 直近の`operation.json`
- 直近の`history.jsonl`
- ServerSeedログ
- 設定ファイルの秘密情報マスク済み内容

`support-bundle`は秘密情報を含めてはならない。秘密情報を除外またはマスクできない項目は収集しない。

出力形式はtar.gzとし、ファイル名は次を基本とする。

```text
serverseed-support_MACHINEID_TIMESTAMP.tar.gz
```

### ツール強化方針

ServerSeed CLIは、危険操作を実行するだけの道具ではなく、管理者が判断できる情報を提供する運用ツールとして強化する。

強化対象は次とする。

- 事実収集
- 状態判定
- 実行計画作成
- ブロック理由の説明
- 修正候補の提示
- 検証結果の機械処理出力
- 調査用情報の安全な収集
- リリース成果物の検証
- 自己診断

サーバ構成の自動修復の扱いは[サーバ構成](../profiles/server.md#更新と復旧)を参照する。マイニング構成の採掘運転に限定した自動復旧は[監視と自動復旧](../profiles/mining.md#監視と自動復旧)に従う。共通の`doctor`、`verify`は引き続き読み取り専用とする。

### 共通オプションと出力

すべてのコマンドは、標準出力に人が読む要約を出し、`--json`指定時は機械処理用JSONを出力する。

破壊的操作または復旧不能な変更を含む操作は、原則として事前計画を表示し、明示確認を要求する。

初期実装で共通対応するオプションは次のとおりとする。

```bash
--json
--dry-run
--yes
--config PATH
--log-level LEVEL
--no-color
--output PATH
```

`--yes`は通常の確認を省略できる。初回構成選択の扱いは[初期構成選択と継続管理](../common-platform.md#初期構成選択と継続管理)、破壊的操作の追加確認は[確認方式](#確認方式)に従う。マイニング構成の自動更新における対話確認の省略は[自動更新と再起動](../profiles/mining.md#自動更新と再起動)の事前ポリシーに従う。

対話不能な環境で確認が必要になった場合は、変更を行わず終了コード`3`で停止する。

`--no-color`は人間向け出力の色付けを無効化する。`--json`指定時は常に色付けしない。

`--output PATH`は、`report`、`plan`、`support-bundle`などの成果物出力先を指定するために使用する。指定先に既存ファイルがある場合は、明示オプションなしに上書きしてはならない。

#### 共通JSON出力

`--json`指定時の出力は、すべてのコマンドで次の基本形に従う。

```json
{
  "schema_version": 1,
  "command": "inspect",
  "result": "success",
  "started_at": "RFC3339",
  "finished_at": "RFC3339",
  "state": "MANAGED",
  "summary": "string",
  "data": {},
  "warnings": [],
  "errors": []
}
```

`result`は`success`、`failed`、`blocked`のいずれかとする。`errors`が空でない場合、終了コードは`0`にしてはならない。

人間向け出力とJSON出力の内容は意味的に一致させる。JSON出力時に進捗ログを標準出力へ混在させてはならない。進捗やログは標準エラーまたはログファイルへ出力する。

#### 確認方式

破壊的操作では、単純な`yes`入力だけでは確認成立としない。

ディスク消去を伴う場合、ServerSeedは次の情報を表示する。

- デバイスパス
- メーカー
- モデル
- 容量
- シリアル番号
- 消去されるパーティション一覧

利用者は表示された確認文字列を正確に入力する必要がある。確認文字列には対象ディスクのシリアル番号またはServerSeedが生成した短い確認コードを含める。

### ログ

ログの形式と保存先は[共通基盤のログ形式](../common-platform.md#ログ形式)を参照する。

### 終了コード

CLIの終了コードは次を基本とする。

| 終了コード | 意味 |
|---|---|
| 0 | 成功 |
| 1 | 一般エラー |
| 2 | 引数または設定エラー |
| 3 | 安全確認失敗 |
| 4 | 検証失敗 |
| 5 | 本体導入・更新・削除またはセットアップ失敗 |
| 6 | 復旧経路未確認 |
| 10 | 再起動が必要 |

再起動が必要な場合でも処理自体が成功しているときは、レポートに成功状態を記録した上で終了コード`10`を返す。
