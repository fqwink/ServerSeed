# ServerSeed 文書索引

- 文書種別: 文書索引
- 対象リポジトリ: `fqwink/ServerSeed`

## 仕様版情報

- 仕様リビジョン: 0.19
- 確定日: 2026-10-16
- ステータス: 確定

開発フェーズと実装の進捗は[ロードマップ](ROADMAP.md)を参照する。

## 案内・作業ルール

| 文書 | 内容 |
|---|---|
| [README.md](../README.md) | プロジェクト概要と文書への入口 |
| [AGENTS.md](../AGENTS.md) | 作業・文書管理ルール |

## 開発計画

| 文書 | 内容 |
|---|---|
| [ROADMAP.md](ROADMAP.md) | 開発フェーズ、依存関係、完了条件、現在の進捗 |

## 製品仕様

| 分類 | 文書 | 定義する内容 |
|---|---|---|
| 製品の目的と対応範囲 | [product-scope.md](product-scope.md) | 目的、基本原則、製品構成、OS・アーキテクチャ・プロバイダーの対応範囲 |
| 共通基盤 | [common-platform.md](common-platform.md) | 本体導入と構成管理の分離、状態判定、操作状態、排他制御、実行記録 |
| サーバ構成 | [profiles/server.md](profiles/server.md) | Dockerアプリ、サーバ固有の更新・復旧方針、apps.yml |
| マイニング構成 | [profiles/mining.md](profiles/mining.md) | 採掘対応、CPU・GPU運転、監視、自動復旧・更新 |
| USB導入 | [provisioning/usb.md](provisioning/usb.md) | ISO生成・収録版固定、媒体マニフェスト、USB作成・交換、OS導入 |
| Cloud導入 | [provisioning/cloud.md](provisioning/cloud.md) | バイナリの直接導入、cloud-init、SSH保護、起動不能時の復旧経路 |
| BIOS・UEFI | [provisioning/firmware.md](provisioning/firmware.md) | ISO起動互換性、機種別設定、起動項目、復電確認 |
| 更新と署名検証 | [operations/updates.md](operations/updates.md) | 更新手順、計画、切り戻し、マニフェスト、署名、バージョン |
| セキュリティ | [operations/security.md](operations/security.md) | 権限、秘密情報、通信、ファイル権限 |
| バックアップ | [operations/backups.md](operations/backups.md) | バックアップ状態、復元確認、データ保護 |
| CLI | [interfaces/cli.md](interfaces/cli.md) | コマンド、オプション、確認方式、JSON出力、終了コード |
| 共通設定 | [interfaces/configuration.md](interfaces/configuration.md) | ファイル形式、管理ファイル一覧、config.yml、policy.yml、構成固有設定の振り分け |
| 実装・配布・検証 | [development/implementation.md](development/implementation.md) | 責任境界、Rust・Cargo、配布ターゲットとリンク、バイナリ導入・削除、systemd、受入条件、リポジトリ構成 |
| CI/CD | [development/ci-cd.md](development/ci-cd.md) | workflow、ビルド・検査、署名・リリース、デプロイ境界 |
