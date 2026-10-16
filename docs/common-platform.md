# 共通基盤と状態管理

[文書一覧](DOCUMENT_INDEX.md)

## 共通基盤と構成の分離

ServerSeed Coreを主軸とし、その上にサーバ構成とマイニング構成を設ける。両構成は同じServerSeedリポジトリで管理する。構成はプラグインではなく、公式機能として責任別の内部モジュールで実装する。

共通基盤と各構成の責任、設定、検証および運用ポリシーを分離する。

| 区分 | 責任 |
|---|---|
| 共通基盤 | Ubuntu Server導入、USB・Cloud実行、機器検出、構成適用の実行制御、署名検証、更新・検証・復旧の共通手順、排他制御、履歴 |
| サーバ構成 | [サーバ構成の責任範囲](profiles/server.md#責任範囲) |
| マイニング構成 | [マイニング構成仕様](profiles/mining.md) |

機器検出は選択済み構成に必要なドライバーや設定の決定に使用し、用途の自動選択には使用しない。

## 本体導入と構成管理

本体の配置と、対象OSをServerSeed管理下へ取り込むセットアップは別操作とする。本体だけを導入しても管理マーカーを作成せず、構成を選択しない。導入操作は[バイナリ導入](development/implementation.md#バイナリ導入)に従う。

本体の現在の導入情報は`/var/lib/serverseed/install.json`に保存する。

```json
{
  "schema_version": 1,
  "status": "installed",
  "serverseed_version": "0.1.0",
  "architecture": "amd64",
  "binary_sha256": "hex",
  "installed_at": "RFC3339"
}
```

`status`は`installed`または`removed`とする。`removed`は本体削除が正常終了したことを表し、バージョンとハッシュは削除した本体の情報を保持する。導入情報は現在状態の記録であり、追記型の履歴にはしない。

管理マーカーのない対応OSは、本体導入済みでも`ADOPTABLE`とする。`installed`の記録と実ファイルが不一致、または本体変更操作が未完了なら`RECOVERABLE`とする。正常に本体を削除した未管理OSは`ADOPTABLE`であり、設定やログが残るだけで破損と判定しない。管理マーカーが残るOSは、導入情報にかかわらず管理対象として整合性を検査する。

## 初期構成選択と継続管理

初期の構成セットアップでは、USB・Cloudのいずれでも利用者が対話操作により `server` または `mining` の一方を明示的に選択する。既定構成、自動判定、設定ファイル、cloud-init、`--yes`による初回選択の代行は認めない。対話不能または選択未完了の場合は構成・OSへの変更を開始せず、終了コード`3`で停止する。本体だけの導入は[本体導入と構成管理](#本体導入と構成管理)に従う。

選択後は必要な入力、対象ディスク消去確認、実行計画および前提検査を経て導入を自動実行する。構成の同時選択は初期対応に含めない。

選択結果は`config.yml`、管理マーカーおよび実行履歴に記録する。更新・復旧は保存済み構成を引き継ぎ、毎回の再選択を要求しない。構成不明または記録不整合の管理対象は`RECOVERABLE`とし、通常の変更を停止する。旧形式の管理対象もサーバ構成と推測せず、明示的な管理者操作で選択・移行する。

構成変更は通常の更新・再実行と分離した明示操作とし、変更計画、影響確認、バックアップ、旧構成の停止・除去および変更後検証を定義してから実装する。

## 共通状態判定

ServerSeedは対象ディスクまたは対象OSを検査し、次の状態に分類する。

| 状態 | 意味 | 基本動作 |
|---|---|---|
| `EMPTY` | OSが存在しない | USB版ではセットアップ候補 |
| `ADOPTABLE` | 対応OSだが構成セットアップ未完了 | 本体導入または初期セットアップ |
| `MANAGED` | ServerSeed管理対象OS | アップデート |
| `RECOVERABLE` | 管理対象だが正常起動または構成に問題がある | 復旧メニュー |
| `UNKNOWN` | 未対応OS、判定不能または安全未確認 | 変更せず停止 |

管理対象であることは、少なくとも次の情報を使用して判定する。

- OSおよびバージョン
- ルートファイルシステム
- `/etc/os-release`
- ServerSeed管理マーカー
- マシン識別情報
- ServerSeed構成バージョン
- 前回処理の完了状態

主要な管理情報は次に保存する。

```text
/etc/serverseed/managed
/etc/serverseed/machine-id
/etc/serverseed/version
```

単にOSらしいファイルが存在するだけでは、管理対象と判定しない。

### 状態判定の優先順位

状態判定は安全側に倒すため、次の優先順位で行う。

1. ルートファイルシステムまたは対象ディスクを読み取れない場合は`UNKNOWN`とする。
2. 前回の変更操作が未完了、管理対象が破損、必須管理ファイルが欠損、保存済み構成が不明・不正・不一致、または[本体導入が不整合・未完了](#本体導入と構成管理)の場合は`RECOVERABLE`とする。
3. ServerSeed管理マーカー、マシン識別情報、構成バージョンおよび保存済み構成が整合する場合は`MANAGED`とする。
4. 管理マーカーがなく、Ubuntu 24.04 LTSであり、[本体導入と構成管理](#本体導入と構成管理)およびCloud版の前提を満たす場合は`ADOPTABLE`とする。
5. パーティション、OS、ブートローダーが存在しないと判断できる場合のみ`EMPTY`とする。
6. 上記に該当しない場合は`UNKNOWN`とする。

`EMPTY`判定はUSB版の対象ディスクに限って使用する。Cloud版ではルートディスクを`EMPTY`として扱わない。

### 管理マーカー

`/etc/serverseed/managed`はJSON形式とし、少なくとも次を含める。

```json
{
  "managed": true,
  "machine_id": "string",
  "created_at": "RFC3339",
  "serverseed_version": "0.1.0",
  "config_revision": 1,
  "migration_revision": 0,
  "profile": "server"
}
```

`profile`は[`config.yml`](interfaces/configuration.md#configyml)の`serverseed.profile`と一致しなければならない。欠損、不正値または不一致の場合は`RECOVERABLE`とする。

`managed`が`true`でない場合、または`machine_id`が`/etc/serverseed/machine-id`と一致しない場合は`MANAGED`と判定しない。

### 前回処理状態

前回処理の記録形式とプロセスの確認方法は[操作状態](#操作状態)を参照する。

前回処理が`running`、`interrupted`、または終了状態不明のまま残っている場合は`RECOVERABLE`とする。次回の変更操作は、`recover`または明示的な管理者確認なしに続行してはならない。実行中の操作に属する診断は[トランザクション診断](interfaces/cli.md#doctor)として区別し、新しい変更操作を並行開始しない。

### 状態スナップショット

`/var/lib/serverseed/state.json`は現在の判定結果を保存する。ファイルは再生成可能なキャッシュとして扱い、判定の唯一の根拠にしてはならない。

```json
{
  "schema_version": 1,
  "state": "MANAGED",
  "detected_at": "RFC3339",
  "machine_id": "string",
  "os": {
    "id": "ubuntu",
    "version_id": "24.04",
    "kernel": "string"
  },
  "boot": {
    "mode": "uefi",
    "secure_boot": "unknown"
  },
  "serverseed": {
    "installed": true,
    "profile": "server",
    "version": "0.1.0",
    "config_revision": 1,
    "migration_revision": 0
  },
  "checks": [
    {
      "id": "os.supported",
      "result": "pass",
      "message": "Ubuntu 24.04 LTS detected"
    }
  ]
}
```

`state`は[共通状態判定](#共通状態判定)の状態名を使用する。`checks[].result`は`pass`、`warn`、`fail`、`unknown`のいずれかとする。

構成未選択の場合、`serverseed.profile`、`config_revision`および`migration_revision`は`null`とし、未確定の値を生成しない。`serverseed.installed`は管理対象判定ではなく、検査対象OSの本体導入状態を表す。USB上の実行本体を、内蔵OSへ導入済みと取り違えない。

### 操作状態

`/var/lib/serverseed/operation.json`は実行中または直近の変更操作を表す。

変更操作の開始時に実行中状態を記録し、成功または失敗の確定時に終了状態を記録する。

```json
{
  "schema_version": 1,
  "operation_id": "string",
  "command": "update",
  "status": "running",
  "started_at": "RFC3339",
  "finished_at": null,
  "pid": 1234,
  "plan_id": "string",
  "lock_path": "/var/lib/serverseed/lock"
}
```

`status`は`running`、`success`、`failed`、`interrupted`のいずれかとする。変更操作の開始時に`running`を記録できない場合、その操作を開始してはならない。

独立した本体変更操作では`command`を`self install`、`self update`、`self uninstall`のいずれかとし、構成未選択の実行履歴では`profile`を`null`にする。通常の`update`内では親の操作ID・ロックを引き継ぎ、親の`command`を上書きしない。初回は権限を確認して管理ディレクトリとロックを用意した後に操作状態を記録し、配置を開始する。

処理中にプロセスが終了し、`running`のまま残った場合、次回起動時にPIDの存在と開始時刻を確認する。実行中プロセスが確認できない場合は`interrupted`として扱う。

## ロックおよび同時実行

ServerSeedは同時に複数の変更操作を実行してはならない。

変更操作は`/var/lib/serverseed/lock`を使用して排他制御する。本体変更と構成・OSの変更も同じロックを使用し、子処理へ分ける場合も検証と操作状態の確定までロックを維持する。読み取り専用の`inspect`、`status`、`version`は原則としてロック取得なしで実行できる。

ロック取得に失敗した場合は、実行中のoperation情報を表示して停止する。強制解除は通常コマンドでは行わず、復旧操作の一部として明示的に扱う。

## 原子性と書き込み

設定、状態、履歴、レポートを書き換える場合は、一時ファイルへ書き込み、同期し、同一ファイルシステム内のrenameで置き換える。本体置換の手順と中断時の扱いは[本体の置換と復旧](operations/updates.md#本体の置換と復旧)に従う。

重要ファイルの部分書き込み、空ファイル化、不完全なJSONまたはYAMLを残してはならない。書き込み失敗時は失敗状態を記録し、次回起動時に`RECOVERABLE`として扱う。

## 外部コマンド実行

ServerSeedが外部コマンドを実行する場合は、引数配列として実行し、シェル展開に依存しない。

外部コマンドの標準出力、標準エラー、終了コード、実行時間を記録する。ログ出力時の秘匿化は[秘密情報の扱い](operations/security.md#秘密情報の扱い)に従う。

タイムアウトを持たない外部コマンドを実行してはならない。タイムアウト時は処理を失敗として記録し、後続の変更操作を停止する。

## 実行履歴

`/var/lib/serverseed/history.jsonl`は1行1JSONとし、操作ごとに追記する。

各行は少なくとも次を含める。

```json
{
  "operation_id": "string",
  "command": "update",
  "started_at": "RFC3339",
  "finished_at": "RFC3339",
  "result": "success",
  "profile": "server",
  "serverseed_version_before": "0.1.0",
  "serverseed_version_after": "0.1.1"
}
```

履歴ファイルの書き込みに失敗した場合、変更操作は成功として扱わない。

## ログ形式

ログは人が読むテキストログと、機械処理用JSONLを分けて保存する。

```text
/var/log/serverseed/serverseed.log
/var/log/serverseed/events.jsonl
```

ログの秘匿化は[秘密情報の扱い](operations/security.md#秘密情報の扱い)に従う。
