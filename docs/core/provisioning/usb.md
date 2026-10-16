# USBによる導入と保守

[文書一覧](../../DOCUMENT_INDEX.md)

## ServerSeed USB

### 役割

ServerSeed USBは、ISOを書き込んだ起動媒体として、物理サーバおよびベアメタル環境に対して次を提供する。

- Ubuntu 24.04 LTSのセットアップ
- ServerSeed初期構成
- 管理対象OSのアップデート
- 状態検査
- 検証
- 復旧支援

### ISOの作成

ISOはアーキテクチャ別に作成する。`packaging/iso/recipe.yml`に、ベースとなる公式Ubuntu Server 24.04 LTS ISOの正確な版・取得URL・SHA-256、公式署名の検証条件、ServerSeedの固定バージョンと本体ハッシュ、ISO生成ツールとその版を定義する。曖昧な`latest`指定や未検証のベースを使用しない。

生成処理は`release/build-iso`から実行し、固定した`xorriso`とSquashFSの操作ツールをCI用イメージで提供する。ベースISOのブート情報を維持して、検証済み本体、導入テンプレートおよび起動用unitを組み込み、USBへのイメージ書き込みと仮想光学メディア起動の両方を検証する。本体をISO向けに再ビルドせず、[リリース工程](../development/ci-cd.md#リリース工程)の同じ成果物を使う。

起動用unitは同梱バイナリの実行時依存と検証ツールを使用できるLive環境で`serverseed`を開始する。Subiquityの無確認な先行実行を抑止し、[セットアップ](#セットアップ)の確認前にOS導入を開始しない。対応する起動方式とSecure Bootの扱いは[ISOの起動互換性](firmware.md#isoの起動互換性)に従う。

### 使用する本体の固定

収録版、起動時の検証、取得とオフラインの制約は[USB起動と取得](../../POLICIES.md#usb起動と取得)を参照する。

### 使用方法

ServerSeed USBでは、起動時に`serverseed`を自動実行し、状態判定に応じて処理を分岐する。

「USB接続」は、稼働中のOSへ挿した瞬間に処理を開始することではなく、専用USBを接続してサーバを起動または再起動することと定義する。

標準操作は次のとおりとする。

1. サーバを停止する。
2. ServerSeed USBを接続する。
3. USBから起動する。
4. ServerSeedが内蔵ストレージを検査する。
5. 状態に応じて分岐する。初期セットアップでは利用者が構成を手動選択し、更新・復旧では保存済み構成を使用する。
6. 完了後にUSBを抜き、内蔵OSを起動する。

### USB起動時の分岐

状態に応じた動作は[USB起動時の分岐](../../POLICIES.md#usb起動時の分岐)を参照する。

### セットアップ

初期セットアップの構成選択は[初期構成選択と継続管理](../../POLICIES.md#構成選択と継続管理)に従う。対象SSDの表示情報と全消去の確認は[CLIの確認方式](../interfaces/cli.md#確認方式)に従う。

利用者が対象SSDと全データ消去を確認した後、次を自動実行する。

1. Ubuntu 24.04 LTSのインストール
2. 管理ユーザーの作成
3. SSH公開鍵の登録
4. [SSH安全条件](../../POLICIES.md#セキュリティの共通条件)の適用
5. SSH鍵接続の確認と認証設定の検証
6. OS更新および基本パッケージ導入
7. ファイアウォールおよび基本セキュリティ設定
8. Tailscaleの導入
9. 選択済み構成の導入（サーバ構成はDocker Engine・Composeなど、マイニング構成は対応ドライバー・SRBMiner-MULTI・運転サービス）
10. 共通および構成固有のバックアップ、ログおよび監視設定
11. 選択済み構成を含むServerSeed管理情報の作成
12. 最終検証およびレポート作成

Ubuntuの導入設定は確認済みの対象ディスク情報からLive環境の`/autoinstall.yaml`へ生成し、その後に対応版のSubiquityを開始する。このファイル名は[Ubuntuインストーラーの外部契約](https://canonical-subiquity.readthedocs-hosted.com/en/latest/tutorial/providing-autoinstall.html)に従う。ISOのルートに消去対象入りの自動認識ファイルをあらかじめ置かない。

新OSへ収録版の本体を導入し、マウントした新OS内で[共通の導入処理](../operations/lifecycle.md#バイナリ導入)を呼び出す。実行中Live環境の導入情報・ロック・マシン識別情報を新OSのものとしてコピーしない。対象OS内での処理とLive環境側の制御は同じ操作として直列化し、内蔵OSの管理記録を最終検証後に確定する。

対話認証の実施時期は[USBの対話認証](../../POLICIES.md#usbの対話認証)に従う。

### 自動化の範囲

構成選択と自動実行の条件は[構成選択と継続管理](../../POLICIES.md#構成選択と継続管理)、確認の表示・入力は[CLIの確認方式](../interfaces/cli.md#確認方式)を参照する。

ディスク操作の許可条件は[ディスク操作](../../POLICIES.md#ディスク操作)に従う。

### USB構成

論理構成は次を基本とする。

```text
SERVERSEED/
├── boot/
├── installer/
│   ├── Ubuntu installer
│   └── autoinstall.yml
├── repository/
├── serverseed/
│   └── release/
│       ├── serverseed_VERSION_linux_ARCH
│       ├── serverseed_VERSION_build.json
│       ├── serverseed_VERSION_manifest.json
│       └── serverseed_VERSION_manifest.json.asc
└── boot-metadata/
```

`ARCH`は当該媒体のアーキテクチャとする。本体用マニフェストは他CPUのエントリを含められるが、実行するCPUの本体と必要な検証情報だけを収録する。`repository/`には配布条件を確認した導入・復旧資材だけを置く。Ubuntu固有の`casper/`、ブートローダー等の構造も維持する。

USBボリュームラベルは`SERVERSEED`とする。収録を禁止する情報は[USB起動と取得](../../POLICIES.md#usb起動と取得)に従う。

### 媒体マニフェスト

ISOの外側に置く媒体マニフェストは、少なくとも次の形式とする。

```json
{
  "schema_version": 1,
  "serverseed_version": "0.1.0",
  "binary_manifest_sha256": "hex",
  "artifacts": [
    {
      "type": "iso",
      "architecture": "amd64",
      "path": "serverseed_0.1.0_usb_amd64.iso",
      "sha256": "hex",
      "size": 123456789,
      "embedded_binary_sha256": "hex",
      "base_iso_sha256": "hex",
      "recipe_sha256": "hex",
      "boot_modes": ["uefi", "bios"],
      "secure_boot": "not_verified"
    }
  ]
}
```

例は一件を抜粋しており、ハッシュとサイズは形式例とする。実配布ではアーキテクチャごとに一件とし、`type`は`iso`に固定する。`path`、`size`、ハッシュの制約は[本体マニフェスト](../operations/lifecycle.md#更新マニフェスト)と同じ定義を参照する。欠損、未知の版・型、重複エントリ、CPUや収録版の不一致を拒否する。

`binary_manifest_sha256`は収録した本体マニフェスト、`embedded_binary_sha256`はそのCPUの本体を指す。両者が本体マニフェストと一致することを生成後に検証する。ベースISOと生成定義のハッシュも保持し、ISO内には自身の外部マニフェストを収録しない。

`boot_modes`と`secure_boot`の値・検証条件は[ISOの起動互換性](firmware.md#isoの起動互換性)に従う。これらは試験結果であり、初期の仕様段階で実機対応済みと記載しない。

### USBの作成と交換

管理端末で信頼鍵、媒体マニフェストの署名、ISOのハッシュ・サイズを確認し、書き込み対象USBの機種・容量・識別情報を確認する。書き込みはUSB全体を消去する操作として[破壊的操作の確認](../interfaces/cli.md#確認方式)を必要とし、デバイス名だけで対象を選ばない。

検証済みISOをイメージ書き込みツールでUSBへ書き込み、ISOと同じ長さの書き込み領域を読み戻して照合する。対象の除外条件は[ディスク操作](../../POLICIES.md#ディスク操作)に従う。媒体の正式性と交換単位は[USB媒体の更新](../../POLICIES.md#usb媒体の更新)に従う。

### ServerSeed USB自体の更新

媒体の更新場所、交換後の扱いと旧版使用の制約は[USB媒体の更新](../../POLICIES.md#usb媒体の更新)を参照する。
