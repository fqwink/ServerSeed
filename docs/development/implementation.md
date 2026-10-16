# 実装・配布・検証仕様

[文書一覧](../DOCUMENT_INDEX.md)

## 実装構成

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

USB起動用ISOとCloudに固有の起動・取得処理はCoreから分離し、CLIのサブコマンドまたは内部モジュールから呼び出す。事業者固有機能の扱いは[プロバイダー非依存](../product-scope.md#プロバイダー非依存)に従う。

### 実装言語

採用言語とスクリプトの利用範囲は[実装言語](../PRINCIPLES.md#実装言語)を参照する。

#### Rustツールチェーン設定

採用版、edition、最低対応版と設定方法は[Rustツールチェーン](../PRINCIPLES.md#rustツールチェーン)を参照する。

## 外部依存

依存の選定、記録およびCargoの利用条件は[外部依存](../PRINCIPLES.md#外部依存)を参照する。

## 配布バイナリのターゲットとリンク

初期のRustビルドターゲットは、[対応アーキテクチャ](../product-scope.md#対応アーキテクチャ)に対応する次のGNUターゲットとする。

| 配布アーキテクチャ | Rustターゲット |
|---|---|
| `amd64` | `x86_64-unknown-linux-gnu` |
| `arm64` | `aarch64-unknown-linux-gnu` |

リンク方式と実行検証の判断基準は[配布バイナリの検証](../PRINCIPLES.md#配布バイナリの検証)を参照する。

## 構成別の責任境界

サーバ構成とマイニング構成は`src/profiles/server/`および`src/profiles/mining/`へ分離する。共通の状態判定、機器情報、更新計画、署名検証、ロックおよび履歴はCoreを再利用し、各構成へ重複実装しない。構成固有の導入、検証、復旧は当該構成が担当する。

構成間の適用制約は[構成の分離](../POLICIES.md#構成の分離)を参照する。

## 管理対象サーバのディレクトリ

```text
/usr/local/bin/serverseed  CLIエントリーポイント
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

配置先の固定と併設の禁止は[本体導入](../POLICIES.md#本体導入)を参照する。

## バイナリ導入

配布形式とOSパッケージ管理との境界は[バイナリ配布](../PRINCIPLES.md#バイナリ配布)を参照する。

本体へ収録する資材とリビジョンの対応は[組込み資材](../PRINCIPLES.md#組込み資材)を参照する。

### 初回の取得と配置

1. 対象OS、CPU、管理権限と実行時依存を確認する。利用者はバージョンを指定し、[配布先](ci-cd.md#成果物保管)からマニフェスト、署名および該当バイナリを一般ユーザーの作業領域へ取得する。手動搬入した同じファイル一式も使用できる。
2. [信頼鍵の初回導入と更新](../POLICIES.md#信頼鍵の初回導入と更新)と[署名およびチェックサム](../operations/updates.md#署名およびチェックサム)に従い、既存の`gpgv`と`sha256sum`で確認する。実行時依存が欠ける場合は不足項目を表示し、実行や本体配置を開始しない。
3. 検証済みバイナリを絶対パスで指定し、[selfコマンド](../interfaces/cli.md#self)の`install`を実行する。本体の`version --json`とマニフェストのバージョンが一致することを確認する。
4. root権限、配置先の所有権、競合する実行本体、空き容量、既存の導入情報と未完了操作を確認し、計画・確認を経て[本体の置換と復旧](../operations/updates.md#本体の置換と復旧)を実行する。
5. [管理対象サーバのディレクトリ](#管理対象サーバのディレクトリ)と[権限](../POLICIES.md#ファイル権限)に従って配置し、必要な管理ディレクトリと共通ポリシーの初期値を用意する。初回の信頼済みキーリングは未配置の場合のみ登録する。
6. 配置した本体のバージョン・ハッシュと読み取り専用診断を確認し、[本体導入情報](../common-platform.md#本体導入と構成管理)を確定する。設定未作成は構成セットアップ待ちとして通知する。

初回導入時の変更範囲と設定保持は[初回導入の保護条件](../POLICIES.md#初回導入の保護条件)を参照する。

### 既存配置と再導入

導入情報に一致する同一バージョン・ハッシュが配置済みの場合は、検査結果を返して正常終了し、設定を再生成しない。新しいバージョンへの変更は`self update`、記録と実ファイルの不一致は`recover --self`へ誘導する。

既存配置との競合時は[競合配置の保護](../POLICIES.md#競合配置の保護)に従う。

### 本体だけの削除

削除の許可条件と保持対象は[本体だけの削除](../POLICIES.md#本体だけの削除)、トランザクションは[本体の置換と復旧](../operations/updates.md#本体の置換と復旧)を参照する。

## systemd連携

常駐方式は[systemd連携](../PRINCIPLES.md#systemd連携)、運転サービスとの接続は[監視サービスと共通基盤の境界](../profiles/mining.md#監視サービスと共通基盤の境界)を参照する。

必要に応じて、将来次のunitを追加できる。

```text
serverseed-verify.service
serverseed-verify.timer
```

timerの実行範囲は[共通timer](../POLICIES.md#共通timer)を参照する。

## テスト体系

テストは次の階層で実施する。

| 種別 | 対象 |
|---|---|
| unit | 状態判定、設定読み込み、マニフェスト検証、更新計画 |
| integration | Ubuntu 24.04 LTSコンテナ上でのCLI、隔離した使い捨て環境での本体導入・更新・削除、読み取り専用検査 |
| vm | ディスク、ブート、復旧、破壊的操作を含む検証 |
| fixture | `/etc/os-release`、lsblk、systemctlなどの実行結果サンプル |

unit testは対象モジュール内の`#[cfg(test)]`で定義し、integration testの入口は`tests/*.rs`に配置する。fixtureは`tests/fixtures/`に保存する。実行コマンドは[CIジョブ](ci-cd.md#cicd)に従う。

構成選択に既定値がないこと、`--yes`・cloud-initが初回選択を代行しないこと、保存構成の継承、不整合時の停止、未選択構成を適用しないことをunit・integrationで検証する。マイニングの採掘停止、通信断、温度異常および更新失敗はfixture・模擬サービスで検証し、実採掘の制約は[検証環境の制約](../POLICIES.md#検証環境の制約)に従う。GPU・BIOS・復電起動は別の実機検証として結果を記録する。

テスト階層の適用条件と危険操作の隔離は[検証環境の制約](../POLICIES.md#検証環境の制約)に従う。

### 配布・導入の受入条件

| 対象 | 必須の検証 |
|---|---|
| 初回導入 | 対応OS・CPUへの配置、構成未選択の維持、依存不足・権限不足・容量不足時の停止 |
| 再導入・削除 | 同一版の冪等性、所有元不明・deb所有ファイルとの競合停止、設定保持、利用中本体の削除拒否 |
| 取得・検証 | 通信断、TLS不正、署名不正、失効鍵、ハッシュ不一致、異なるCPU、旧版への巻き戻しの拒否 |
| 本体更新・復旧 | ロック競合、配置直前・直後の中断、診断失敗、旧版復元、復旧不能時の停止と記録 |
| USB用ISO | [ISOの作成](../provisioning/usb.md#isoの作成)に従うバイナリ一致、USB・仮想メディア起動、外部通信なしでの本体起動 |
| VPS | [Cloud導入](../provisioning/cloud.md)に従うSSH保護、ISO不要の導入、cloud-init後の手動構成選択 |

正常系・異常系とも、成功・停止・失敗の終了コードとJSON出力を[CLI契約](../interfaces/cli.md#共通json出力)に照合する。シンボリックリンクと一般ユーザーによる入力差し替えも検査する。試験対象の保護条件は[検証環境の制約](../POLICIES.md#検証環境の制約)を参照する。

## 初期リポジトリ構成

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
│   ├── run-cli-contract
│   ├── run-format
│   ├── run-integration
│   ├── run-lint
│   ├── run-package
│   ├── run-spec-check
│   └── run-unit
├── release/
│   ├── build
│   ├── build-iso
│   ├── publish
│   ├── sign
│   └── verify
├── Cargo.toml
├── Cargo.lock
├── rust-toolchain.toml
├── AGENTS.md
├── README.md
├── src/
│   ├── main.rs
│   ├── lib.rs
│   ├── cli/
│   ├── core/
│   ├── platform/
│   ├── profiles/
│   │   ├── server/
│   │   └── mining/
│   ├── usb/
│   └── cloud/
├── tests/
│   ├── cli_contract.rs
│   ├── binary_install.rs
│   ├── integration.rs
│   └── fixtures/
├── packaging/
│   └── iso/
└── docs/
    ├── DOCUMENT_INDEX.md
    ├── ROADMAP.md
    ├── PRINCIPLES.md
    ├── POLICIES.md
    ├── product-scope.md
    ├── common-platform.md
    ├── profiles/
    │   ├── server.md
    │   └── mining.md
    ├── provisioning/
    │   ├── usb.md
    │   ├── cloud.md
    │   └── firmware.md
    ├── operations/
    │   ├── updates.md
    │   ├── security.md
    │   └── backups.md
    ├── interfaces/
    │   ├── cli.md
    │   └── configuration.md
    └── development/
        ├── implementation.md
        └── ci-cd.md
```

実装はこの構成を起点とし、仕様と矛盾するディレクトリや責務分離を追加しない。
