# ServerSeedの目的と対応範囲

[文書一覧](../DOCUMENT_INDEX.md)

## 目的

ServerSeedは、共通基盤を主軸とし、サーバ構成とマイニング構成を同一リポジトリで管理するRust製CLIのライフサイクル管理ツールである。物理サーバ、ミニPC、CPU・GPU搭載PC、VPSおよびクラウド上の対応環境を、共通の定義に基づいてセットアップ、アップデート、検証および復旧する。

主要な利用者インターフェースは`serverseed`コマンドとする。本体は[バイナリ導入](lifecycle.md#バイナリ導入)を共通方式とし、物理サーバでは[USB起動用ISO](usb.md#isoの作成)、VPSおよびクラウドでは[既存OSへの導入](cloud.md#本体の導入)を使用する。事業者との責任境界は[プロバイダー非依存](#プロバイダー非依存)を参照する。

## 基本原則

製品全体の判断基準は[基本方針](../PRINCIPLES.md#製品の基本方針)を参照する。

## 製品構成

| 名称 | 役割 |
|---|---|
| ServerSeed | `serverseed` CLIを中心としたシステム全体 |
| ServerSeed CLI | 利用者および自動処理が実行する主要インターフェース |
| ServerSeed Core | 主軸となる共通基盤。CLIから呼び出される共通のセットアップ、更新、検証および復旧処理 |
| サーバ構成（server） | [サーバ構成仕様](../server/specification.md) |
| マイニング構成（mining） | [マイニング構成仕様](../mining/specification.md) |
| ServerSeed USB | ServerSeed ISOを書き込んで物理PCおよびベアメタルを起動する専用USB |
| ServerSeed Cloud | VPSおよびクラウドサーバへ`serverseed`を導入して実行する方式 |
| ServerSeed ISO | 共通バイナリを収録したUSB用の起動・導入・復旧媒体。検証済みの仮想メディア起動にも使用する |

ISOを独立した共通基盤や構成として実装しない。配布成果物とバイナリの同一性は[リリース工程](ci-cd.md#リリース工程)で保証する。

## 対応範囲

対応OSは両構成ともUbuntu Serverに統一し、初期対応バージョンはUbuntu Server 24.04 LTSに一本化する。

開発フェーズと実施順序は[ロードマップ](../ROADMAP.md)を参照する。

Debian、RHEL系、FreeBSD、その他OSは対応対象外とする。将来提案や補助文書で他OS対応を扱う場合も、本仕様が更新されない限りServerSeedの実装対象に含めない。

### Ubuntu 24.04 LTS判定

Ubuntu 24.04 LTS対応判定は`/etc/os-release`を主情報とする。

対応OSと判定するには、少なくとも次を満たす必要がある。

```text
ID=ubuntu
VERSION_ID=24.04
```

`VERSION_ID`が`24.04`以外、`ID`が`ubuntu`以外、または`/etc/os-release`を読み取れない場合は対応OSと判定しない。`VERSION_CODENAME`は補助情報として記録できるが、判定の唯一の根拠にしてはならない。

Ubuntu派生OSはUbuntu 24.04 LTSとして扱わない。`ID_LIKE=debian`のみを根拠に対応OSと判定してはならない。

### 対応アーキテクチャ

初期対応アーキテクチャは次とする。

| アーキテクチャ | 用途 |
|---|---|
| `amd64` | 物理サーバ、VPS、クラウドの標準対象 |
| `arm64` | ARMサーバ、ARM対応VPS、将来の物理環境対象 |

マイニング構成の利用条件は[マイニング構成](../mining/specification.md#対応範囲)を参照する。

対応外アーキテクチャでの実行可否は[対応外アーキテクチャ](../POLICIES.md#対応外アーキテクチャ)を参照する。

### Ubuntuパッケージ管理前提

ServerSeedはUbuntu 24.04 LTSの標準パッケージ管理を前提とする。

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

### プロバイダー非依存

事業者APIとの関係は[プロバイダー非依存](../PRINCIPLES.md#プロバイダー非依存)を参照する。

環境はプロバイダー名ではなく、次の能力に基づいて判定する。

- Ubuntu 24.04 LTSであること
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
