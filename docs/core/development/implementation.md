# 実装構成と責任境界

[文書一覧](../../DOCUMENT_INDEX.md)

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

採用言語とスクリプトの利用範囲は[実装言語](../../PRINCIPLES.md#実装言語)を参照する。

#### Rustツールチェーン設定

採用版、edition、最低対応版と設定方法は[Rustツールチェーン](../../PRINCIPLES.md#rustツールチェーン)を参照する。

## 内製crateの境界と差し替え

依存の制約と変更手順は[AGENTS.md](../../../AGENTS.md#rustと内製crateの依存制約)、抽出の判断は[段階的な内製化](../../PRINCIPLES.md#段階的な内製化)を参照する。

内製crateを抽出する場合、対象責務の正本に次の契約を対応付ける。共通基盤・サーバ構成・マイニング構成の所有境界を、crate化だけを理由に変更しない。

| 契約項目 | 対応付ける内容 |
|---|---|
| 所有責務 | 正本のファイル・節と呼び出し元。共通契約か構成固有契約か |
| 入出力 | 公開型・関数の入力と出力、許容値、既定値の参照先 |
| 失敗と副作用 | エラー分類、中断・タイムアウト、変更する資源、再試行・復旧の契約 |
| 互換性 | CLI・設定・保存データ・処理結果との対応、維持する版・形式の範囲 |
| 検証 | 正常系・異常系・境界値の受入条件と、差し替え前後に共通して実行するテスト |
| 差し替え単位 | 旧モジュールまたは内製crateと新しい内製crateの対応、移行順序、依存方向 |

呼び出し元と実装の間は責務に必要な公開契約だけで接続し、具体的な実装型、グローバル状態または下位実装の詳細へ直接依存させない。単一パッケージ内のモジュールで契約を検証した後、必要な責務だけをリポジトリ内のcrateとして抽出する。Cargoのpath依存で接続し、パッケージ間の循環参照を作らない。

差し替え前後の同一入力に対する出力・失敗・副作用を照合し、移行が必要な場合は対象仕様の互換範囲と移行手順へ対応付ける。テストの配置と追跡形式は[テスト体系](ci-cd.md#テスト体系)に従う。

## 構成別の責任境界

サーバ構成とマイニング構成は`src/profiles/server/`および`src/profiles/mining/`へ分離する。共通の状態判定、機器情報、更新計画、署名検証、ロックおよび履歴はCoreを再利用し、各構成へ重複実装しない。構成固有の導入、検証、復旧は当該構成が担当する。

構成間の適用制約は[構成の分離](../../POLICIES.md#構成の分離)を参照する。

## systemd連携

常駐方式は[systemd連携](../../PRINCIPLES.md#systemd連携)、運転サービスとの接続は[監視サービスと共通基盤の境界](../../mining/specification.md#監視サービスと共通基盤の境界)を参照する。

必要に応じて、将来次のunitを追加できる。

```text
serverseed-verify.service
serverseed-verify.timer
```

timerの実行範囲は[共通timer](../../POLICIES.md#共通timer)を参照する。

## 初期リポジトリ構成

初期実装のリポジトリ構成は次を基本とする。文書以外は将来の実装配置であり、現在作成済みであることを示さない。内製crateを抽出する際の配置は[差し替え契約](#内製crateの境界と差し替え)で責務とともに確定する。

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
    ├── DEPENDENCY_EXCEPTIONS.md
    ├── core/
    │   ├── product-scope.md
    │   ├── common-platform.md
    │   ├── provisioning/
    │   │   ├── usb.md
    │   │   ├── cloud.md
    │   │   └── firmware.md
    │   ├── operations/
    │   │   ├── lifecycle.md
    │   │   └── backups.md
    │   ├── interfaces/
    │   │   ├── cli.md
    │   │   └── configuration.md
    │   └── development/
    │       ├── implementation.md
    │       └── ci-cd.md
    ├── server/
    │   └── specification.md
    └── mining/
        └── specification.md
```

実装はこの構成を起点とし、仕様と矛盾するディレクトリや責務分離を追加しない。
