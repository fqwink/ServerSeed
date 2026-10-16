# ServerSeed

Ubuntu Server 24.04 LTS向けのRust製CLI。マスター管理・マイニング・サーバの各構成の導入・更新・検証・復旧を管理します。

[文書一覧](docs/DOCUMENT_INDEX.md)から全ドキュメントを確認できます。

## 詳細ドキュメント

| 確認したい内容 | 参照先 |
|---|---|
| 製品の概要・対応範囲 | [製品仕様](docs/core/product-scope.md) |
| 構成別の仕様 | [マスター管理](docs/master/specification.md)・[マイニング](docs/mining/specification.md)・[サーバ](docs/server/specification.md) |
| 導入・保守の仕様 | [外部媒体・ISO（物理PC）](docs/core/usb.md)・[SSH（VPS・クラウド）](docs/core/cloud.md) |
| 操作・設定の仕様 | [CLI](docs/core/cli.md)・[共通設定](docs/core/configuration.md) |
| 更新・復旧・保全の仕様 | [更新・復旧](docs/core/lifecycle.md)・[バックアップ](docs/core/backups.md)・[内蔵OSの起動確認](docs/core/firmware.md#内蔵osの起動確認) |
| 開発・運用の判断基準 | [方針](docs/PRINCIPLES.md)・[ポリシー](docs/POLICIES.md)・[作業ルール](AGENTS.md) |
| 実装・検証・配布の仕様 | [実装構成](docs/core/implementation.md)・[CI/CD・リリース](docs/core/ci-cd.md) |
| 開発状況 | [ロードマップ](docs/ROADMAP.md) |
| ライセンス・外部依存 | [製品ライセンス](docs/PRINCIPLES.md#製品ライセンス)・[外部依存の例外管理](docs/DEPENDENCY_EXCEPTIONS.md) |
| 報告の受付 | [Issue・脆弱性報告の受付条件と窓口](docs/POLICIES.md#報告の受付) |
