# ServerSeed マスター仕様

- 文書種別: マスター仕様
- 仕様リビジョン: 0.15
- 確定日: 2026-09-03
- ステータス: 確定
- 対象リポジトリ: `fqwink/ServerSeed`

## 1. 目的

ServerSeedは、物理サーバ、VPSおよびクラウド上のサーバを、共通の定義に基づいてセットアップ、アップデート、検証および復旧するGo製CLIのサーバライフサイクル管理ツールである。

主要な利用者インターフェースは`serverseed`コマンドとする。物理サーバでは専用USBから`serverseed`を起動し、VPSおよびクラウドでは既存のDebian 13へ`serverseed`を導入して実行する。特定のクラウドプロバイダーやプロバイダーAPIには依存しない。

## 2. 基本原則

1. セットアップとアップデートは`serverseed` CLIから実行し、ServerSeed Coreの共通定義で管理する。
2. 物理サーバでは1本の専用USBでセットアップ、アップデートおよび復旧を行う。
3. 完全無人インストールは採用しない。
4. ディスクを消去するセットアップでは、対象ディスクと全消去を人が確認する。
5. 状態が不明、検証不能または安全を確認できない場合は、変更を行わず停止する。
6. 更新前の検査、バックアップ確認、更新後の検証および実行記録を必須とする。
7. 認証情報や秘密鍵をUSB、Gitまたは設定ファイルへ平文保存しない。
8. 更新ファイルは電子署名およびチェックサムで検証する。
9. 同じ構成を再実行できるよう、セットアップと更新処理は冪等にする。
10. 特定のVPS・クラウドプロバイダー固有機能はServerSeedの責任範囲外とする。

## 3. 製品構成

| 名称 | 役割 |
|---|---|
| ServerSeed | `serverseed` CLIを中心としたシステム全体 |
| ServerSeed CLI | 利用者および自動処理が実行する主要インターフェース |
| ServerSeed Core | CLIから呼び出される共通のセットアップ、更新、検証および復旧処理 |
| ServerSeed USB | 物理PCおよびベアメタルで`serverseed`を起動する専用USB |
| ServerSeed Cloud | VPSおよびクラウドサーバへ`serverseed`を導入して実行する方式 |
| ServerSeed ISO | ISO起動に対応する仮想環境用。将来対応 |

## 4. 対応範囲

対応OSはDebian 13に一本化する。

開発対象は次の順序とする。

1. ServerSeed CLI
2. ServerSeed Core
3. Debian 13検査および構成適用
4. ServerSeed USB
5. ServerSeed Cloud
6. Debian 13向けcloud-init対応
7. ServerSeed ISO

Ubuntu、RHEL系、FreeBSD、その他OSは対応対象外とする。将来提案や補助文書で他OS対応を扱う場合も、本仕様が更新されない限りServerSeedの実装対象に含めない。

### 4.1 Debian 13判定

Debian 13対応判定は`/etc/os-release`を主情報とする。

対応OSと判定するには、少なくとも次を満たす必要がある。

```text
ID=debian
VERSION_ID=13
```

`VERSION_ID`が`13`以外、`ID`が`debian`以外、または`/etc/os-release`を読み取れない場合は対応OSと判定しない。`VERSION_CODENAME`は補助情報として記録できるが、判定の唯一の根拠にしてはならない。

Debian派生OSはDebian 13として扱わない。`ID_LIKE=debian`のみを根拠に対応OSと判定してはならない。

### 4.2 対応アーキテクチャ

初期対応アーキテクチャは次とする。

| アーキテクチャ | 用途 |
|---|---|
| `amd64` | 物理サーバ、VPS、クラウドの標準対象 |
| `arm64` | ARMサーバ、ARM対応VPS、将来の物理環境対象 |

その他アーキテクチャは対応対象外とする。対応外アーキテクチャでは、読み取り専用の`inspect`のみ実行できる。変更操作は開始しない。

### 4.3 Debianパッケージ管理前提

ServerSeedはDebian 13の標準パッケージ管理を前提とする。

少なくとも次のコマンドまたは機能が利用可能であることを確認する。

```text
dpkg
apt-get
systemctl
journalctl
sha256sum
gpgv
```

変更操作の実行に必要なコマンドが欠けている場合は、変更を開始せず、必要な不足項目を表示する。

## 5. 共通状態判定

ServerSeedは対象ディスクまたは対象OSを検査し、次の状態に分類する。

| 状態 | 意味 | 基本動作 |
|---|---|---|
| `EMPTY` | OSが存在しない | USB版ではセットアップ候補 |
| `ADOPTABLE` | 対応OSだがServerSeed未導入 | Cloud版では初期セットアップ |
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

### 5.1 状態判定の優先順位

状態判定は安全側に倒すため、次の優先順位で行う。

1. ルートファイルシステムまたは対象ディスクを読み取れない場合は`UNKNOWN`とする。
2. 破損、前回処理の未完了、必須管理ファイルの欠損がある管理対象は`RECOVERABLE`とする。
3. ServerSeed管理マーカー、マシン識別情報、構成バージョンが整合する場合は`MANAGED`とする。
4. Debian 13であり、ServerSeed未導入で、Cloud版の前提を満たす場合は`ADOPTABLE`とする。
5. パーティション、OS、ブートローダーが存在しないと判断できる場合のみ`EMPTY`とする。
6. 上記に該当しない場合は`UNKNOWN`とする。

`EMPTY`判定はUSB版の対象ディスクに限って使用する。Cloud版ではルートディスクを`EMPTY`として扱わない。

### 5.2 管理マーカー

`/etc/serverseed/managed`はJSON形式とし、少なくとも次を含める。

```json
{
  "managed": true,
  "machine_id": "string",
  "created_at": "RFC3339",
  "serverseed_version": "0.1.0",
  "config_revision": 1,
  "migration_revision": 0
}
```

`managed`が`true`でない場合、または`machine_id`が`/etc/serverseed/machine-id`と一致しない場合は`MANAGED`と判定しない。

### 5.3 前回処理状態

変更操作は開始時に`/var/lib/serverseed/operation.json`へ実行中状態を記録し、成功または失敗の確定時に終了状態を記録する。

前回処理が`running`、`interrupted`、または終了状態不明のまま残っている場合は`RECOVERABLE`とする。次回の`update`または`setup`は、`recover`または明示的な管理者確認なしに続行してはならない。

### 5.4 状態スナップショット

`/var/lib/serverseed/state.json`は現在の判定結果を保存する。ファイルは再生成可能なキャッシュとして扱い、判定の唯一の根拠にしてはならない。

```json
{
  "schema_version": 1,
  "state": "MANAGED",
  "detected_at": "RFC3339",
  "machine_id": "string",
  "os": {
    "id": "debian",
    "version_id": "13",
    "kernel": "string"
  },
  "boot": {
    "mode": "uefi",
    "secure_boot": "unknown"
  },
  "serverseed": {
    "installed": true,
    "version": "0.1.0",
    "config_revision": 1,
    "migration_revision": 0
  },
  "checks": [
    {
      "id": "os.supported",
      "result": "pass",
      "message": "Debian 13 detected"
    }
  ]
}
```

`state`は`EMPTY`、`ADOPTABLE`、`MANAGED`、`RECOVERABLE`、`UNKNOWN`のいずれかとする。`checks[].result`は`pass`、`warn`、`fail`、`unknown`のいずれかとする。

### 5.5 操作状態

`/var/lib/serverseed/operation.json`は実行中または直近の変更操作を表す。

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

処理中にプロセスが終了し、`running`のまま残った場合、次回起動時にPIDの存在と開始時刻を確認する。実行中プロセスが確認できない場合は`interrupted`として扱う。

## 6. ServerSeed USB

### 6.1 役割

ServerSeed USBは、物理サーバおよびベアメタル環境に対して次を提供する。

- Debian 13のセットアップ
- ServerSeed初期構成
- 管理対象OSのアップデート
- 状態検査
- 検証
- 復旧支援

### 6.2 使用方法

「USB接続」は、稼働中のOSへ挿した瞬間に処理を開始することではなく、専用USBを接続してサーバを起動または再起動することと定義する。

標準操作は次のとおりとする。

1. サーバを停止する。
2. ServerSeed USBを接続する。
3. USBから起動する。
4. ServerSeedが内蔵ストレージを検査する。
5. 状態に応じてセットアップ、アップデート、復旧または安全停止へ分岐する。
6. 完了後にUSBを抜き、内蔵OSを起動する。

### 6.3 USB起動時の分岐

| 検出状態 | 動作 |
|---|---|
| `EMPTY` | セットアップ画面 |
| `MANAGED` | アップデート前検査 |
| `RECOVERABLE` | 復旧メニュー |
| `ADOPTABLE` | 原則停止し、明示的な取り込み操作を要求 |
| `UNKNOWN` | 変更せず停止 |

### 6.4 セットアップ

セットアップ時は対象SSDについて次を表示する。

- デバイスパス
- メーカー
- モデル
- 容量
- シリアル番号
- 消去されること

利用者が対象SSDと全データ消去を確認した後、次を自動実行する。

1. Debian 13のインストール
2. 管理ユーザーの作成
3. SSH公開鍵の登録
4. rootによるSSHログインの禁止
5. SSH鍵接続確認後のパスワード認証無効化
6. OS更新および基本パッケージ導入
7. ファイアウォールおよび基本セキュリティ設定
8. Tailscaleの導入
9. Docker EngineおよびComposeの導入
10. バックアップ、ログおよび監視設定
11. ServerSeed管理情報の作成
12. 最終検証およびレポート作成

Tailscaleなど対話認証が必要な機能は、秘密情報をUSBへ保存せず、セットアップ完了後に利用者が認証する。

### 6.5 完全無人インストール

完全無人インストールは実装対象外とする。

ディスクの自動選択および確認なしの全消去は禁止する。デバイス名だけを根拠に対象ディスクを決定してはならない。

### 6.6 USB構成

論理構成は次を基本とする。

```text
SERVERSEED/
├── boot/
├── installer/
│   ├── Debian installer
│   └── preseed.cfg
├── repository/
├── serverseed/
│   ├── VERSION
│   ├── setup.yml
│   ├── update.yml
│   ├── verify.yml
│   └── migrations/
├── signatures/
└── manifest.json
```

USBボリュームラベルは`SERVERSEED`とする。

## 7. BIOSおよびUEFI

### 7.1 方針

BIOSおよびUEFIは、検査可能な項目を自動検査し、標準化された範囲で安全に変更できる項目だけを自動変更する。

処理方針は次のとおりとする。

| 状態 | 動作 |
|---|---|
| 検査可能かつ変更可能 | 自動変更 |
| 検査可能だが変更不可 | 必要な手動操作を表示 |
| 検査不能 | BIOS画面での確認を要求 |

### 7.2 自動化対象

対応するUEFI環境では次を自動化できる。

- DebianのUEFI起動項目作成
- ブートローダー登録
- 起動順序の検査および変更
- 次回起動先の指定
- 不要な起動項目の検査

### 7.3 手動設定の可能性がある項目

次は機種依存のため、初回に手動設定が必要となる場合がある。

- UEFIモード
- USB Boot
- USBを内蔵SSDより優先する起動順序
- Secure Boot
- 停電復旧後の自動電源ON
- 仮想化機能
- Wake on LAN
- TPM
- ストレージモード

推奨起動順は、ServerSeed USB、内蔵SSD、その他の順とする。USBが存在しなければ内蔵SSDから通常起動する。

未知のBIOS項目を推測して変更してはならない。BIOSパスワードおよびストレージモードは自動変更しない。

## 8. ServerSeed Cloud

### 8.1 役割

ServerSeed Cloudは、VPSまたはクラウド事業者が準備した対応OSへServerSeedを導入し、共通定義に基づいて構成、更新および検証する。

### 8.2 前提

ServerSeed Cloudは原則としてOSインストールを担当しない。利用者または事業者がDebian 13を作成し、SSH接続できる状態にする。

セットアップ方法は次をサポートする。

- cloud-init
- SSHからのセットアップ
- ローカルへ搬入したServerSeedパッケージ
- 将来のServerSeed ISO

### 8.3 初期セットアップ

`ADOPTABLE`状態の対応OSに対して、次を行う。

1. OS、systemd、ネットワークおよび空き容量の検査
2. 現在のSSH接続および管理ユーザーの保護
3. 基本セキュリティ設定
4. 必要パッケージおよびアプリ実行基盤の導入
5. バックアップ、ログおよび監視設定
6. ServerSeed管理情報の作成
7. 最終検証

既存のSSH鍵、ネットワーク設定、cloud-initおよびゲストエージェントを、確認なしに削除または上書きしてはならない。汎用Cloud版ではルートディスクを再パーティションしない。

## 9. プロバイダー非依存

ServerSeedは、AWS、Google Cloud、Azure、国内VPSなど特定事業者のAPIに依存しない。

環境はプロバイダー名ではなく、次の能力に基づいて判定する。

- Debian 13であること
- systemdが利用可能であること
- SSH管理経路を維持できること
- cloud-initの有無
- 仮想化環境の有無
- 必要なディスク容量
- 必要な外部通信またはローカル更新元の有無

次はServerSeedの責任範囲外とする。

- VPSまたはインスタンスの作成・削除
- クラウドディスクの作成・追加
- セキュリティグループ操作
- クラウドスナップショット
- 固定IP割り当て
- DNS設定
- ロードバランサー設定
- プロバイダー固有バックアップ
- 課金および契約管理
- プロバイダーAPI認証

プロバイダー別アダプターは実装しない。

## 10. アップデート

### 10.1 更新対象

更新対象を次の層に分離する。

| 対象 | 方針 |
|---|---|
| Debianセキュリティ更新 | 自動適用可能 |
| 通常のOS更新 | ServerSeedによる検査後に適用 |
| ServerSeed Coreおよび構成定義 | 署名付き更新として適用 |
| Dockerアプリ | アプリ単位でバックアップおよび検証して適用 |
| Debianメジャー更新 | 専用移行手順 |
| BIOS更新 | 原則として自動化しない |

勝手な自動再起動は行わない。再起動が必要な場合は明示する。

### 10.2 更新元

更新元は抽象化し、次を利用できるようにする。

- ServerSeed USB
- 署名付きHTTPSリポジトリ
- ローカルディレクトリ
- 手動搬入した更新パッケージ

Cloud版はネットワーク更新を標準とし、閉域環境ではローカルパッケージを使用できるものとする。特定の外部サービス停止によって、ローカル更新や検証まで利用不能にしてはならない。

### 10.3 更新前検査

更新開始前に少なくとも次を確認する。

- 更新情報の署名
- ファイルのチェックサム
- 対象マシンおよび管理状態
- 現在と更新後のバージョン
- OS互換性
- ディスク空き容量
- パッケージ管理の正常性
- SSDまたはストレージ状態
- バックアップの最終成功状態
- 前回処理が完了していること
- SSHまたはコンソールによる復旧経路

安全を確認できない場合は更新を開始しない。

### 10.4 更新処理

1. 更新情報を取得して検証する。
2. 変更予定を事前検査する。
3. 現在の設定とバージョン情報を保存する。
4. 必要なデータバックアップを確認または実行する。
5. OS、ServerSeedおよび対象アプリを更新する。
6. 必要な構成移行を実行する。
7. サービスを起動または再読み込みする。
8. ヘルスチェックおよび総合検証を行う。
9. 成功状態または失敗状態を記録する。
10. 再起動の要否を表示する。

処理途中で失敗した場合は成功として記録せず、後続処理を停止する。

### 10.5 ロールバック

設定ファイル、構成バージョンおよび更新前情報を保存し、設定変更を直前の状態へ戻せるようにする。

OSパッケージの無条件な自動ダウングレードは行わない。データ形式が変更されるアプリは、アプリ固有のバックアップおよび復元手順を必要とする。

### 10.6 更新計画

`update`、`setup`、`recover`は、変更開始前に計画を作成する。計画は人間向けに表示し、`--json`では構造化して出力する。

```json
{
  "schema_version": 1,
  "plan_id": "string",
  "command": "update",
  "created_at": "RFC3339",
  "target_state": "MANAGED",
  "requires_confirmation": true,
  "requires_reboot": false,
  "steps": [
    {
      "id": "backup.check",
      "title": "Check last backup",
      "destructive": false,
      "requires_network": false
    }
  ],
  "blocked_by": []
}
```

`blocked_by`が空でない計画は実行してはならない。`--dry-run`では計画作成まで行い、変更操作、パッケージ更新、サービス再起動、ファイル書き換えを行わない。

### 10.7 原子性と書き込み

設定、状態、履歴、レポートを書き換える場合は、一時ファイルへ書き込み、同期し、同一ファイルシステム内のrenameで置き換える。

重要ファイルの部分書き込み、空ファイル化、不完全なJSONまたはYAMLを残してはならない。書き込み失敗時は失敗状態を記録し、次回起動時に`RECOVERABLE`として扱う。

### 10.8 外部コマンド実行

ServerSeedが外部コマンドを実行する場合は、引数配列として実行し、シェル展開に依存しない。

外部コマンドの標準出力、標準エラー、終了コード、実行時間を記録する。ただし秘密情報を含む可能性がある値はログ出力前にマスクする。

タイムアウトを持たない外部コマンドを実行してはならない。タイムアウト時は処理を失敗として記録し、後続の変更操作を停止する。

## 11. セキュリティ

- rootによるSSHログインを禁止する。
- SSH鍵接続を確認してからパスワード認証を無効化する。
- 管理経路を確認できない状態でSSH設定を厳格化しない。
- USBおよび更新パッケージの真正性を検証する。
- SSH秘密鍵、APIトークン、バックアップ暗号鍵を平文保存しない。
- Dockerソケットを外部公開しない。
- コンテナポートを必要なく外部公開しない。
- 更新処理は必要な権限に限定する。
- すべての重要操作について開始、終了、成功、失敗および検証結果を記録する。

### 11.1 権限モデル

読み取り専用コマンドは、可能な限り一般ユーザーで実行できるようにする。ただし、OS、ストレージ、systemd、ログの読み取りにroot権限が必要な場合は、不足権限を明示して`unknown`または`warn`として扱う。

変更操作はroot権限を必要とする。root権限がない状態で`setup`、`update`、`recover`を実行した場合は変更を行わず終了コード`2`で停止する。

ServerSeedはsetuidバイナリとして配布しない。権限昇格は利用者が`sudo`、rootログイン、または管理用実行環境で明示的に行う。

### 11.2 秘密情報の扱い

秘密情報として扱う値は次を含む。

- SSH秘密鍵
- APIトークン
- パスワード
- バックアップ暗号鍵
- Tailscaleなどの認証トークン
- Cookie、セッション、Bearerトークン

秘密情報は設定ファイル、Git、USB内の平文ファイル、ログ、レポート、履歴JSONLへ出力しない。必要な場合は`******`でマスクする。

### 11.3 ネットワーク安全性

更新元がHTTPSの場合、TLS検証を無効化してはならない。証明書検証に失敗した場合は更新を停止する。

閉域環境ではローカルディレクトリまたはUSB上の更新元を使用できる。この場合でも署名検証とチェックサム検証を省略してはならない。

## 12. バックアップおよびデータ保護

- 重要更新前にバックアップの成功を確認する。
- バックアップ先が不明、容量不足または検証不能の場合、重要更新を停止する。
- RAIDやディスクミラーをバックアップとして扱わない。
- 設定、アプリデータおよびデータベースを区別する。
- データベースはサービス固有の整合したダンプまたはスナップショット方式を使用する。
- バックアップの存在だけでなく、定期的な復元確認を前提とする。
- ディスク初期化およびバックアップ先のフォーマットは自動実行しない。

### 12.1 バックアップ状態

バックアップ状態はServerSeedが直接作成したもの、またはアプリ定義で指定された確認コマンドの結果から判定する。

バックアップ成功状態は少なくとも次を含める。

```json
{
  "schema_version": 1,
  "app": "example",
  "last_success_at": "RFC3339",
  "method": "command",
  "location": "string",
  "verified": true
}
```

`verified`が`true`でないバックアップは、重要更新の前提条件として扱わない。

### 12.2 復元確認

ServerSeedはバックアップの存在だけで成功と判定しない。アプリまたはデータ種別ごとに復元確認の方法を定義できるようにする。

復元確認が未実施、期限切れ、または失敗している場合、重要更新では警告または停止条件とする。停止にするか警告にするかは`policy.yml`で指定する。

## 13. コマンド

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

ServerSeed USBでは、起動時に`serverseed`を自動実行し、状態判定に応じて処理を分岐する。Cloud版でも導入後の操作入口は`serverseed`とする。破壊的処理には明示的確認を必要とする。

### 13.1 コマンド別の変更可否

| コマンド | 既定動作 | 変更操作 |
|---|---|---|
| `inspect` | 読み取り専用 | なし |
| `status` | 読み取り専用 | なし |
| `version` | 読み取り専用 | なし |
| `report` | 読み取り専用 | なし |
| `verify` | 読み取り専用 | なし |
| `setup` | 計画表示後に確認 | あり |
| `update` | 計画表示後に確認 | あり |
| `recover` | 復旧計画表示後に確認 | あり |

`verify`は状態確認と検証のみを行い、設定修正、パッケージ更新、サービス再起動を行わない。修正が必要な場合は、必要な操作をレポートする。

### 13.2 `inspect`

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

### 13.3 `setup`

`setup`は`EMPTY`または`ADOPTABLE`に対して実行する。

USB版の`EMPTY`セットアップでは、対象ディスクのメーカー、モデル、容量、シリアル番号、現在のパーティション情報を表示し、利用者に確認を求める。

Cloud版の`ADOPTABLE`セットアップでは、既存SSH接続を保護し、確認なしにネットワーク設定、SSH鍵、cloud-init、ゲストエージェントを削除または上書きしない。

### 13.4 `update`

`update`は`MANAGED`に対して実行する。`UNKNOWN`、`EMPTY`、`ADOPTABLE`に対しては実行しない。`RECOVERABLE`に対しては、復旧が必要であることを表示して停止する。

更新前検査に失敗した場合は変更を開始しない。更新開始後に失敗した場合は失敗状態を記録し、後続処理を停止する。

### 13.5 `recover`

`recover`は`RECOVERABLE`に対して実行する。復旧内容は、起動修復、設定復元、前回操作の後始末、レポート生成に限定する。

データを削除する復旧、OSパッケージの無条件なダウングレード、バックアップ未確認の復元は行わない。

### 13.6 `report`

`report`は直近の操作、検査結果、検証結果、失敗理由、再起動要否、次に必要な手動操作を表示する。

人間向け出力では要約を優先し、`--json`では監視や外部ツールが処理しやすい構造化データを出力する。

## 14. 管理対象サーバのディレクトリ

```text
/usr/bin/serverseed        CLIエントリーポイント
/usr/lib/serverseed/       補助ファイル
/etc/serverseed/           設定
/var/lib/serverseed/       状態、バージョンおよび履歴
/var/log/serverseed/       セットアップ、更新および検証ログ
/srv/                      アプリおよび保存データ
```

主なレポートおよび状態情報は次に保存する。

```text
/var/log/serverseed/setup.log
/var/log/serverseed/update.log
/var/log/serverseed/report.txt
/var/lib/serverseed/last-success
/var/lib/serverseed/last-failure
```

`/opt/serverseed/`は、手動搬入版、開発版、またはUSB内の実行コード配置に使用できる。管理対象サーバへDebianパッケージとして導入する通常版では、実行コードを`/opt/serverseed/`へ配置しない。

## 15. ServerSeed USB自体の更新

ServerSeed USBは別の管理端末で更新する。

1. 公式の更新情報を取得する。
2. 電子署名およびチェックサムを検証する。
3. USBへ反映する。
4. USB全体を検証する。
5. 更新結果を記録する。

USB内のバージョンが管理対象サーバより古い場合は、通常モードでダウングレードしない。旧版の使用は、明示的な復旧操作に限定する。

## 16. 実装上の境界

ServerSeed CLIは、入力解析、出力形式、確認プロンプト、終了コード、ロック取得を担当する。

ServerSeed Coreには、CLIから呼び出される環境共通の次の処理を配置する。

- 状態判定
- 構成適用
- 更新
- マイグレーション
- 検証
- ログ
- 署名およびチェックサム検証
- バックアップ前提条件の確認

USB、Cloudおよび将来のISOに固有の起動・取得処理はCoreから分離し、CLIのサブコマンドまたは内部パッケージから呼び出す。ただし、プロバイダー別アダプターは作成しない。

## 17. 実装方針

### 17.1 実装言語

ServerSeedはGo製CLIとして実装する。ServerSeed CLIおよびCoreの主実装言語はGoとする。

Goは静的型付け、単一バイナリ配布、クロスコンパイル、標準ライブラリによるOS操作およびテストの容易さを理由に採用する。

初期実装ではDebian 13で提供されるGoツールチェーンまたは公式Goツールチェーンでビルドする。外部依存は最小限とし、依存を追加する場合は理由、ライセンス、供給元、更新方針を記録する。

シェルスクリプトは、初期ブート、パッケージ導入、systemd連携、USB起動時の薄いラッパーに限定する。状態判定、設定解釈、検証、更新判断、レポート生成などの中核処理はGoで実装する。

### 17.1.1 Goバージョン方針

Goの固定採用バージョンは、初回実装開始時にDebian 13で利用可能なGo、公式Goの安定版、CI実行環境の対応状況を確認して決定する。

固定採用バージョンを決めるまでは、`go.mod`の`go` directiveを仮固定してはならない。初回実装時に採用バージョン、確認日、採用理由を`MASTER_SPEC.md`または補助文書に記録する。

Go moduleの外部依存は最小限とし、追加する場合は次を記録する。

- モジュール名
- バージョン
- ライセンス
- 採用理由
- 代替案
- セキュリティ更新方針

Go標準ライブラリで実装できる処理は標準ライブラリを優先する。

### 17.2 パッケージ形式

ServerSeed CLIおよびCoreはDebianパッケージとして配布する。パッケージ名は`serverseed`とする。

初期のインストール先は次のとおりとする。

```text
/usr/bin/serverseed              CLIエントリーポイント
/usr/lib/serverseed/             補助ファイル
/etc/serverseed/                 管理設定
/var/lib/serverseed/             状態、履歴、ロック
/var/log/serverseed/             ログ、レポート
```

`/opt/serverseed/`は手動搬入または開発版の配置先として使用できるが、通常配布版の標準配置はDebianパッケージ規約に従う。

### 17.2.1 Debianパッケージ要件

Debianパッケージは次を満たす。

- `/usr/bin/serverseed`を配置する。
- 初期設定テンプレートを`/usr/lib/serverseed/defaults/`に配置する。
- `/etc/serverseed/`、`/var/lib/serverseed/`、`/var/log/serverseed/`を作成する。
- 設定ファイルをユーザー変更可能なconffileとして扱う。
- パッケージ削除時に`/etc/serverseed/`、`/var/lib/serverseed/`、`/var/log/serverseed/`を自動削除しない。
- パッケージ更新時に既存設定を確認なしに上書きしない。

パッケージのpostinstは必要最小限のディレクトリ作成と権限設定のみを行う。ネットワーク通信、サーバ設定変更、サービス再起動、ディスク操作はpostinstで実行しない。

### 17.2.2 systemd連携

初期実装では常駐デーモンを持たない。

必要に応じて、将来次のunitを追加できる。

```text
serverseed-verify.service
serverseed-verify.timer
```

timerは検証とレポート作成のみを行い、設定変更、パッケージ更新、サービス再起動を行わない。自動更新を行うtimerは初期実装では作成しない。

### 17.3 設定形式

人が編集する構成定義はYAML形式とする。ファイル拡張子は`.yml`に統一する。

機械処理を主目的とするマニフェスト、状態スナップショット、実行履歴の構造化データはJSON形式とする。

初期に扱う主なファイルは次のとおりとする。

```text
/etc/serverseed/config.yml
/etc/serverseed/policy.yml
/etc/serverseed/apps.yml
/var/lib/serverseed/state.json
/var/lib/serverseed/history.jsonl
```

YAMLは安全なパーサーで読み込み、任意コード実行や独自タグの評価を行わない。

設定ファイルには認証情報、秘密鍵、APIトークン、バックアップ暗号鍵を保存しない。秘密情報が必要な処理は、OSの権限管理、対話入力、または外部の安全な秘密情報管理に委ねる。

### 17.3.1 `config.yml`

`config.yml`はServerSeed全体の基本設定を定義する。

```yaml
serverseed:
  config_revision: 1
  environment: cloud
  log_level: info
  allow_reboot: false
  update_channel: stable
```

`environment`は`usb`、`cloud`、`iso`のいずれかとする。初期実装では`usb`と`cloud`を扱い、`iso`は予約値とする。

`config.yml`の必須項目は次とする。

| 項目 | 型 | 必須 | 既定値 |
|---|---|---|---|
| `serverseed.config_revision` | integer | はい | なし |
| `serverseed.environment` | string | はい | なし |
| `serverseed.log_level` | string | いいえ | `info` |
| `serverseed.allow_reboot` | boolean | いいえ | `false` |
| `serverseed.update_channel` | string | いいえ | `stable` |

`log_level`は`debug`、`info`、`warn`、`error`のいずれかとする。`update_channel`は初期実装では`stable`のみ有効とする。

未知のトップレベルキーは設定エラーとする。未知の下位キーは警告ではなく設定エラーとし、変更操作を開始しない。

### 17.3.2 `policy.yml`

`policy.yml`は安全確認と変更許可の方針を定義する。

```yaml
policy:
  require_backup_before_update: true
  require_ssh_recovery_path: true
  allow_password_auth_disable: true
  allow_unattended_security_updates: true
  allow_destructive_disk_setup: false
```

`allow_destructive_disk_setup`が`false`の場合、USB版でもディスク消去を開始しない。`true`の場合でも対象ディスク識別情報を含む明示確認を必須とする。

`policy.yml`の必須項目は次とする。

| 項目 | 型 | 必須 | 既定値 |
|---|---|---|---|
| `policy.require_backup_before_update` | boolean | いいえ | `true` |
| `policy.require_ssh_recovery_path` | boolean | いいえ | `true` |
| `policy.allow_password_auth_disable` | boolean | いいえ | `true` |
| `policy.allow_unattended_security_updates` | boolean | いいえ | `true` |
| `policy.allow_destructive_disk_setup` | boolean | いいえ | `false` |

ポリシー未指定時は安全側の既定値を使用する。型が一致しない場合は設定エラーとする。

### 17.3.3 `apps.yml`

`apps.yml`はServerSeedが管理するDockerアプリを定義する。

```yaml
apps:
  - name: example
    compose_file: /srv/example/docker-compose.yml
    healthcheck:
      type: http
      url: http://127.0.0.1:8080/health
    backup:
      required: true
      command: /usr/local/bin/example-backup
```

アプリ定義にバックアップが必要と指定されている場合、更新前にバックアップ成功状態を確認できなければ更新しない。

`apps.yml`が存在しない場合、管理対象Dockerアプリなしとして扱う。空の`apps`配列は有効とする。

アプリ名は英数字、ハイフン、アンダースコアのみを許可する。`compose_file`は絶対パスとし、`/srv/`配下を推奨する。相対パス、空文字、親ディレクトリ参照を含むパスは設定エラーとする。

`healthcheck.type`は初期実装では`http`または`command`とする。`backup.command`は絶対パスとし、シェル文字列ではなく実行ファイルと引数配列へ分解できる形式にする。

### 17.3.4 更新マニフェスト

更新マニフェストはJSON形式とし、署名対象に含める。

```json
{
  "schema_version": 1,
  "serverseed_version": "0.1.0",
  "config_revision": 1,
  "migration_revision": 0,
  "target_debian_version": "13",
  "artifacts": [
    {
      "name": "serverseed",
      "type": "deb",
      "path": "packages/serverseed_0.1.0_amd64.deb",
      "sha256": "hex"
    }
  ]
}
```

マニフェストに未対応の`schema_version`、Debian 13以外の対象バージョン、現在状態から移行不能なリビジョンが含まれる場合は更新しない。

### 17.3.5 実行履歴

`/var/lib/serverseed/history.jsonl`は1行1JSONとし、操作ごとに追記する。

各行は少なくとも次を含める。

```json
{
  "operation_id": "string",
  "command": "update",
  "started_at": "RFC3339",
  "finished_at": "RFC3339",
  "result": "success",
  "serverseed_version_before": "0.1.0",
  "serverseed_version_after": "0.1.1"
}
```

履歴ファイルの書き込みに失敗した場合、変更操作は成功として扱わない。

### 17.4 署名およびチェックサム

更新マニフェストはSHA-256チェックサムとOpenPGP detached signatureで検証する。

署名検証には`gpgv`を使用し、信頼する公開鍵はServerSeed管理用キーリングに保存する。

```text
/etc/serverseed/trustedkeys.gpg
```

更新処理は次の順序で検証する。

1. 更新マニフェストの署名を検証する。
2. マニフェスト内の対象バージョン、互換性および必須条件を検査する。
3. 取得済みファイルのSHA-256チェックサムを検証する。
4. すべて成功した場合のみ更新計画を作成する。

署名検証またはチェックサム検証に失敗した更新は適用しない。

### 17.5 バージョン管理

ServerSeed本体、構成定義、マイグレーションにはそれぞれバージョンを持たせる。

バージョン形式はSemantic Versioningを基本とする。ただし、構成定義とマイグレーションは互換性判定のために独立した整数リビジョンを併用できる。

管理対象サーバには、少なくとも次を記録する。

```text
serverseed_version
config_revision
migration_revision
last_successful_operation
last_successful_operation_at
```

### 17.6 CLI動作

すべてのコマンドは、標準出力に人が読む要約を出し、`--json`指定時は機械処理用JSONを出力する。

破壊的操作または復旧不能な変更を含む操作は、原則として事前計画を表示し、明示確認を要求する。

初期実装で共通対応するオプションは次のとおりとする。

```bash
--json
--dry-run
--yes
--config PATH
--log-level LEVEL
```

`--yes`は通常の確認を省略できるが、ディスク消去などの破壊的操作では、対象ディスク識別情報を含む追加確認を必要とする。

対話不能な環境で確認が必要になった場合は、変更を行わず終了コード`3`で停止する。

### 17.6.1 共通JSON出力

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

### 17.6.2 確認方式

破壊的操作では、単純な`yes`入力だけでは確認成立としない。

ディスク消去を伴う場合、ServerSeedは次の情報を表示する。

- デバイスパス
- メーカー
- モデル
- 容量
- シリアル番号
- 消去されるパーティション一覧

利用者は表示された確認文字列を正確に入力する必要がある。確認文字列には対象ディスクのシリアル番号またはServerSeedが生成した短い確認コードを含める。

### 17.6.3 ログ形式

ログは人が読むテキストログと、機械処理用JSONLを分けて保存する。

```text
/var/log/serverseed/serverseed.log
/var/log/serverseed/events.jsonl
```

ログには秘密情報を出力しない。コマンドライン引数、環境変数、設定値に秘密情報が含まれる可能性がある場合はマスクする。

### 17.7 終了コード

CLIの終了コードは次を基本とする。

| 終了コード | 意味 |
|---|---|
| 0 | 成功 |
| 1 | 一般エラー |
| 2 | 引数または設定エラー |
| 3 | 安全確認失敗 |
| 4 | 検証失敗 |
| 5 | 更新またはセットアップ失敗 |
| 6 | 復旧経路未確認 |
| 10 | 再起動が必要 |

再起動が必要な場合でも処理自体が成功しているときは、レポートに成功状態を記録した上で終了コード`10`を返す。

### 17.8 ロックおよび同時実行

ServerSeedは同時に複数の変更操作を実行してはならない。

変更操作は`/var/lib/serverseed/lock`を使用して排他制御する。読み取り専用の`inspect`、`status`、`version`は原則としてロック取得なしで実行できる。

ロック取得に失敗した場合は、実行中のoperation情報を表示して停止する。強制解除は通常コマンドでは行わず、復旧操作の一部として明示的に扱う。

### 17.8.1 ファイル権限

管理ディレクトリとファイルの権限は次を基本とする。

| パス | 所有者 | 権限 |
|---|---|---|
| `/etc/serverseed/` | `root:root` | `0755` |
| `/etc/serverseed/*.yml` | `root:root` | `0644` |
| `/etc/serverseed/trustedkeys.gpg` | `root:root` | `0644` |
| `/var/lib/serverseed/` | `root:root` | `0750` |
| `/var/log/serverseed/` | `root:adm` | `0750` |
| `/usr/bin/serverseed` | `root:root` | `0755` |

秘密情報を含む可能性があるファイルは`0600`とし、root以外に読ませない。

### 17.9 テスト方針

テストは次の階層で実施する。

| 種別 | 対象 |
|---|---|
| unit | 状態判定、設定読み込み、マニフェスト検証、更新計画 |
| integration | Debianコンテナ上でのCLI、パッケージ導入、読み取り専用検査 |
| vm | ディスク、ブート、復旧、破壊的操作を含む検証 |
| fixture | `/etc/os-release`、lsblk、systemctlなどの実行結果サンプル |

初期CIではunitとintegrationを必須とし、USB起動、ディスク消去、UEFI変更を伴うテストはVM専用とする。開発者の通常環境やCIコンテナ上でディスク消去を実行してはならない。

### 17.10 CI/CD

ServerSeedリポジトリにおけるCI/CD実行ツールはGitHubツールとする。

GitHubのリポジトリ管理機能、ブランチ保護、CI/CD workflow、リリース管理を使用する。

CIは、変更ブランチ、`main`ブランチ、リリース候補で実行できるものとする。CDは、管理者が明示的にリリース操作を実行した場合にのみ実行する。

GitHub workflowはServerSeedリポジトリのCI/CD実行定義として扱う。ServerSeed固有の検査、ビルド、Debianパッケージ作成、署名、GitHub Releases公開はGitHubツール上で実行する。

初期workflowは次とする。

| workflow | トリガー | 目的 |
|---|---|---|
| `ci.yml` | pull request、`main`へのpush | 通常CI |
| `release.yml` | `v*`タグ、手動実行 | リリース成果物作成 |
| `vm-test.yml` | 手動実行のみ | VMを使う破壊的操作の検証 |

`vm-test.yml`は通常CIの必須条件に含めない。実ディスク、実ホスト、利用者データを対象にしてはならない。

初期CIで必須とするジョブは次のとおりとする。

| ジョブ | 内容 |
|---|---|
| `format` | `gofmt`およびMarkdown整形確認 |
| `lint` | Go静的解析、未使用コード、危険なシェル実行の検査 |
| `test-unit` | Go unit test |
| `test-integration` | Debian 13コンテナ上での読み取り専用CLI検査 |
| `build` | Linux amd64およびarm64向け`serverseed`バイナリ作成 |
| `package-deb` | Debianパッケージの作成と内容検査 |
| `spec-check` | `MASTER_SPEC.md`と実装上の固定値の矛盾検査 |

CIで実行してはならない処理は次のとおりとする。

- 実ディスクの消去
- 実ホストのパーティション変更
- UEFIまたはBIOS設定変更
- 実ホストのSSH設定変更
- 実ホストのDockerサービス再起動
- `apt upgrade`などCI実行環境自体を変更する更新
- 許可されていないリモートリポジトリへの自動push
- 対象サーバへの自動デプロイ

VMを使用する破壊的操作の検証は、管理者が明示的に起動する専用CIジョブでのみ実行する。専用CIジョブは、テスト用ディスクイメージ以外を対象にしてはならない。

ServerSeedのCIは、GitHub workflowから少なくとも次のローカルスクリプトを呼び出せるようにする。

```bash
ci/run-format
ci/run-lint
ci/run-unit
ci/run-integration
ci/run-build
ci/run-package
ci/run-spec-check
ci/run-all
```

各CIコマンドはローカル開発環境でも実行できるようにする。これはGitHub workflowの重複を避けるための補助であり、ServerSeedのCI/CD実行ツールをGitHubツールから置き換えるものではない。

### 17.10.1 GitHubツールの役割

GitHubは次の役割を担当する。

- Gitリポジトリ管理
- ブランチ保護
- レビューおよび変更取り込み
- CI/CD workflowの起動
- CI結果の表示
- リリースタグ管理
- リリース成果物の公開または公開入口

GitHub Actionsを使用する場合、workflowは`.github/workflows/`に配置する。重複を避けるため、必要に応じて`ci/`および`release/`配下の補助スクリプトを呼び出す。

### 17.10.2 CI実行環境

CIの標準実行環境はDebian 13コンテナまたはDebian 13仮想マシンとする。

CI実行環境には、必要なGoツールチェーン、Debianパッケージ作成ツール、静的解析ツール、署名検証ツールを事前に導入する。

CI実行中にホストOSを変更してはならない。依存関係の導入が必要な場合は、CI用イメージを更新し、その変更を履歴として記録する。

CI用イメージは、再現可能な定義ファイルから作成する。イメージ定義はリポジトリ内で管理する。

### 17.10.3 ブランチと保護

標準ブランチは`main`とする。

`main`への直接pushは原則禁止し、変更ブランチからレビューを経て取り込む。`main`へ取り込むには、必須CIジョブの成功とレビューを必要とする。

仕様変更を含む変更では、`MASTER_SPEC.md`の仕様リビジョンを更新しなければならない。

### 17.10.4 リリース

リリースはGit tagで管理する。タグ形式は`vMAJOR.MINOR.PATCH`とする。

管理者がリリース操作を実行した場合、CDは次を生成する。

- Linux amd64用`serverseed`バイナリ
- Linux arm64用`serverseed`バイナリ
- Debian amd64パッケージ
- Debian arm64パッケージ
- 更新マニフェスト
- SHA-256チェックサム
- OpenPGP detached signature

リリース成果物は、署名検証とチェックサム検証に成功したものだけを公開対象とする。

リリース成果物名は次を基本とする。

```text
serverseed_VERSION_linux_amd64
serverseed_VERSION_linux_arm64
serverseed_VERSION_amd64.deb
serverseed_VERSION_arm64.deb
serverseed_VERSION_manifest.json
serverseed_VERSION_checksums.txt
serverseed_VERSION_manifest.json.asc
```

`VERSION`はGit tagの先頭`v`を除いたSemantic Versioning値とする。例として、タグ`v0.1.0`の場合は`serverseed_0.1.0_linux_amd64`とする。

リリース操作は、少なくとも次のコマンドで実行できるようにする。

```bash
release/build
release/sign
release/verify
release/publish
```

`release/publish`は、署名済みかつ検証済みの成果物だけを公開する。

### 17.10.5 署名鍵の扱い

リリース署名鍵はCIログに出力してはならない。

署名処理はリリース操作に限定する。通常CIでは署名鍵へアクセスできないようにする。

初期実装では署名用の実行環境を通常CIと分離する。署名鍵の保管、利用権限、ローテーション手順をリリース手順書に明記する。

### 17.10.6 アーティファクト検査

生成したDebianパッケージは、少なくとも次を検査する。

- `/usr/bin/serverseed`が存在すること
- maintainer scriptが禁止処理を含まないこと
- conffileが正しく扱われること
- パッケージ削除時に状態、ログ、設定を削除しないこと
- インストール後に`serverseed version --json`が成功すること

生成したバイナリは、少なくとも次を検査する。

- `serverseed version`が成功すること
- `serverseed inspect --json`がJSONのみを標準出力へ出すこと
- `serverseed --help`が成功すること

### 17.10.7 成果物保管

CI成果物は一時保管とし、リリース成果物と区別する。

リリース成果物は、GitHub Releasesを標準の公開先とする。必要に応じて静的ファイル配布場所にも保存できる。どの公開先を使用する場合でも、成果物のチェックサム、署名、ビルドメタデータを併せて公開する。

成果物には、ビルド元commit、タグ、ビルド日時、ビルド環境識別子、チェックサム、署名情報を紐づける。

### 17.10.8 デプロイ方針

ServerSeedは対象サーバへ自動デプロイしない。

CDが行うのは、成果物のビルド、検査、署名、公開までとする。既存サーバへの適用は、管理者が`serverseed update`を明示的に実行する。

自動更新配布を将来追加する場合でも、ServerSeed本体の仕様、署名検証、更新計画、明示確認の制約を緩めてはならない。

### 17.11 初期リポジトリ構成

初期実装のリポジトリ構成は次を基本とする。

```text
serverseed/
├── .github/
│   └── workflows/
│       ├── ci.yml
│       ├── release.yml
│       └── vm-test.yml
├── ci/
│   ├── images/
│   ├── run-all
│   ├── run-build
│   ├── run-format
│   ├── run-integration
│   ├── run-lint
│   ├── run-package
│   ├── run-spec-check
│   └── run-unit
├── release/
│   ├── build
│   ├── publish
│   ├── sign
│   └── verify
├── go.mod
├── go.sum
├── README.md
├── MASTER_SPEC.md
├── cmd/
│   └── serverseed/
│       └── main.go
├── internal/
│   ├── core/
│   ├── platform/
│   ├── usb/
│   └── cloud/
├── pkg/
├── tests/
│   ├── unit/
│   ├── integration/
│   └── fixtures/
├── packaging/
│   └── debian/
└── docs/
```

実装はこの構成を起点とし、仕様と矛盾するディレクトリや責務分離を追加しない。

## 18. 仕様変更

本書をServerSeedのマスター仕様とする。

実装、補助文書および将来の提案が本書と矛盾する場合、本書を優先する。仕様変更は本書を更新し、変更理由と履歴をGitコミットに記録する。
