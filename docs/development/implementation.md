# 実装・配布・検証方針

[文書一覧](../DOCUMENT_INDEX.md)

## 実装方針

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

ServerSeed CLIおよびCoreの主実装言語はRustとする。

Rustは型と所有権によるリソース管理、バイナリ配布、およびCargoによるビルド・依存管理・テストを理由に採用する。

初期実装は単一のCargoパッケージを基本とし、CLIの入口を`src/main.rs`、共通ロジックの入口を`src/lib.rs`に配置する。CLIから各責務のモジュールを呼び出す。

シェルスクリプトは、初期ブート、導入準備、systemd連携、USB起動時の薄いラッパーに限定する。状態判定、設定解釈、検証、更新判断、レポート生成などの中核処理はRustで実装する。

#### Rustツールチェーン方針

固定採用バージョンは、初回実装開始時に公式Rustの安定版、依存crate、Ubuntu 24.04 LTSおよびCI実行環境の対応状況を確認して決定する。editionと最低対応Rustバージョンも併せて決定する。

実装開始時に`rust-toolchain.toml`へ具体的な採用バージョンを指定し、rustfmtとClippyも同じツールチェーンから使用する。`Cargo.toml`にはeditionと`rust-version`を明記する。採用バージョンを決めるまでは、これらに仮の値を記載しない。

採用バージョン、edition、最低対応Rustバージョン、確認日および採用理由は、本節に現在有効な内容として記録する。

## 外部依存

Rustの依存crateは最小限とし、追加する場合は次を記録する。

- crate名
- バージョン
- ライセンス
- 採用理由
- 供給元
- 代替案
- 更新方針（セキュリティ更新を含む）

Rust標準ライブラリで実装できる処理は標準ライブラリを優先する。

`Cargo.toml`で依存を宣言し、`Cargo.lock`をGit管理する。依存解決を伴うCI・リリースのCargoコマンドには`--locked`を指定し、lockfileの変更が必要な場合は停止する。依存更新はリリースビルドと分離した変更として扱う。

## 配布バイナリのターゲットとリンク

初期のRustビルドターゲットは、[対応アーキテクチャ](../product-scope.md#対応アーキテクチャ)に対応する次のGNUターゲットとする。

| 配布アーキテクチャ | Rustターゲット |
|---|---|
| `amd64` | `x86_64-unknown-linux-gnu` |
| `arm64` | `aarch64-unknown-linux-gnu` |

単一の実行ファイルによる配布と、完全な静的リンクは区別する。リンク方式、必要な共有ライブラリおよびローダーを確認し、[成果物のビルドメタデータ](ci-cd.md#成果物保管)に記録する。完全な静的リンクや実行時依存がないことを、未検証のまま保証してはならない。

各ターゲットのバイナリは、対応アーキテクチャのUbuntu 24.04 LTS上で実行を検証する。クロスコンパイルの成功だけで実行可能と判断しない。

## 構成別の責任境界

サーバ構成とマイニング構成は`src/profiles/server/`および`src/profiles/mining/`へ分離する。共通の状態判定、機器情報、更新計画、署名検証、ロックおよび履歴はCoreを再利用し、各構成へ重複実装しない。構成固有の導入、検証、復旧は当該構成が担当する。

サーバ構成のDockerアプリ定義をマイニングのネイティブサービスへ流用しない。設定、テスト、仕様および更新対象を構成別に分離し、未選択構成の処理を実行しない。

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

本体の配置先は固定し、管理対象OSで`/opt/serverseed/`やパッケージ管理下の配置先へ別の実行本体を併設しない。ISO内の搬入元は[USB構成](../provisioning/usb.md#usb構成)で定義する。

## バイナリ導入

本体は[リリース成果物](ci-cd.md#リリース)の単一実行ファイルを使用する。ServerSeed専用のdebパッケージ、APTリポジトリおよびパッケージスクリプトを標準導入経路にしない。Ubuntu、ドライバーおよびアプリのパッケージ管理は引き続きOSの仕組みを使用する。

初期設定・systemd unitのテンプレートと本体に対応する構成定義・マイグレーション処理はバイナリへ収録し、ビルド時に構文を検証する。利用者の設定、秘密情報、第三者の採掘ソフトやGPUドライバーは本体へ埋め込まない。収録する定義のリビジョンは[更新マニフェスト](../operations/updates.md#更新マニフェスト)と一致させる。

### 初回の取得と配置

1. 対象OS、CPU、管理権限と実行時依存を確認する。利用者はバージョンを指定し、[配布先](ci-cd.md#成果物保管)からマニフェスト、署名および該当バイナリを一般ユーザーの作業領域へ取得する。手動搬入した同じファイル一式も使用できる。
2. [信頼鍵の初回導入と更新](../operations/security.md#信頼鍵の初回導入と更新)と[署名およびチェックサム](../operations/updates.md#署名およびチェックサム)に従い、既存の`gpgv`と`sha256sum`で確認する。実行時依存が欠ける場合は不足項目を表示し、実行や本体配置を開始しない。
3. 検証済みバイナリを絶対パスで指定し、[selfコマンド](../interfaces/cli.md#self)の`install`を実行する。本体の`version --json`とマニフェストのバージョンが一致することを確認する。
4. root権限、配置先の所有権、競合する実行本体、空き容量、既存の導入情報と未完了操作を確認し、計画・確認を経て[本体の置換と復旧](../operations/updates.md#本体の置換と復旧)を実行する。
5. [管理対象サーバのディレクトリ](#管理対象サーバのディレクトリ)と[権限](../operations/security.md#ファイル権限)に従って配置し、必要な管理ディレクトリと共通ポリシーの初期値を用意する。初回の信頼済みキーリングは未配置の場合のみ登録する。
6. 配置した本体のバージョン・ハッシュと読み取り専用診断を確認し、[本体導入情報](../common-platform.md#本体導入と構成管理)を確定する。設定未作成は構成セットアップ待ちとして通知する。

初回導入は構成セットアップやOS設定変更を開始せず、管理マーカー、`config.yml`および構成固有設定を作成しない。未配置ファイルの生成だけを行い、既存設定・信頼鍵は上書きしない。初期構成の選択と設定生成は[共通基盤](../common-platform.md#初期構成選択と継続管理)に従う。

### 既存配置と再導入

導入情報に一致する同一バージョン・ハッシュが配置済みの場合は、検査結果を返して正常終了し、設定を再生成しない。新しいバージョンへの変更は`self update`、記録と実ファイルの不一致は`recover --self`へ誘導する。

`dpkg-query`による所有元確認、`/usr/bin/serverseed`などの競合配置、既存の導入情報がない実行本体を検出した場合は停止する。パッケージの自動削除、他の配置先へのリンク作成、所有元不明ファイルの上書きによる移行は行わない。管理者が別途保全・整理した後に再検査する。

### 本体だけの削除

`self uninstall`は導入情報のハッシュと一致する配置本体だけを削除し、導入情報を`removed`へ更新する。設定、状態、鍵、ログ、退避ファイルおよびアプリデータは保持する。管理マーカーがある場合、またはServerSeedを参照する稼働・有効化済みunitがある場合は停止する。管理解除やアプリの撤去、データ消去を本体削除に含めない。

## systemd連携

ServerSeed共通基盤の初期実装では常駐デーモンを持たない。マイニング構成に固有の運転サービスは[監視サービスと共通基盤の境界](../profiles/mining.md#監視サービスと共通基盤の境界)に従って導入する。

必要に応じて、将来次のunitを追加できる。

```text
serverseed-verify.service
serverseed-verify.timer
```

timerは検証とレポート作成のみを行い、設定変更、パッケージ更新、サービス再起動を行わない。サーバ構成および共通の検証timerでは自動更新を行わない。マイニング構成の更新処理は構成固有のtimerから共通CLIを呼び出し、[自動更新と再起動](../profiles/mining.md#自動更新と再起動)のポリシーと共通のロック・検証を適用する。

## テスト方針

テストは次の階層で実施する。

| 種別 | 対象 |
|---|---|
| unit | 状態判定、設定読み込み、マニフェスト検証、更新計画 |
| integration | Ubuntu 24.04 LTSコンテナ上でのCLI、隔離した使い捨て環境での本体導入・更新・削除、読み取り専用検査 |
| vm | ディスク、ブート、復旧、破壊的操作を含む検証 |
| fixture | `/etc/os-release`、lsblk、systemctlなどの実行結果サンプル |

unit testは対象モジュール内の`#[cfg(test)]`で定義し、integration testの入口は`tests/*.rs`に配置する。fixtureは`tests/fixtures/`に保存する。実行コマンドは[CIジョブ](ci-cd.md#cicd)に従う。

構成選択に既定値がないこと、`--yes`・cloud-initが初回選択を代行しないこと、保存構成の継承、不整合時の停止、未選択構成を適用しないことをunit・integrationで検証する。マイニングの採掘停止、通信断、温度異常および更新失敗はfixture・模擬サービスで検証し、通常CIで実採掘は行わない。GPU・BIOS・復電起動は別の実機検証として結果を記録する。

初期CIではunitとintegrationを必須とし、USB起動、ディスク消去、UEFI変更を伴うテストはVM専用とする。開発者の通常環境やCIコンテナ上でディスク消去を実行してはならない。

### 配布・導入の受入条件

| 対象 | 必須の検証 |
|---|---|
| 初回導入 | 対応OS・CPUへの配置、構成未選択の維持、依存不足・権限不足・容量不足時の停止 |
| 再導入・削除 | 同一版の冪等性、所有元不明・deb所有ファイルとの競合停止、設定保持、利用中本体の削除拒否 |
| 取得・検証 | 通信断、TLS不正、署名不正、失効鍵、ハッシュ不一致、異なるCPU、旧版への巻き戻しの拒否 |
| 本体更新・復旧 | ロック競合、配置直前・直後の中断、診断失敗、旧版復元、復旧不能時の停止と記録 |
| USB用ISO | [ISOの作成](../provisioning/usb.md#isoの作成)に従うバイナリ一致、USB・仮想メディア起動、外部通信なしでの本体起動 |
| VPS | [Cloud導入](../provisioning/cloud.md)に従うSSH保護、ISO不要の導入、cloud-init後の手動構成選択 |

正常系・異常系とも、成功・停止・失敗の終了コードとJSON出力を[CLI契約](../interfaces/cli.md#共通json出力)に照合する。シンボリックリンクと一般ユーザーによる入力差し替えも検査する。実環境の機密情報・ディスクや実採掘を試験に使用しない。

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
