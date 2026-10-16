# ServerSeed 文書索引

- 文書種別: 文書索引
- 対象リポジトリ: `fqwink/ServerSeed`

## 仕様版情報

- 仕様リビジョン: 0.29
- 確定日: 2026-10-16
- ステータス: 確定

本欄の確定は承認済みの文書版を表し、製品全体の完全仕様化や実装完了を意味しない。仕様充足の判定は[開発ポリシー](POLICIES.md#実装着手と仕様充足の判定)、現在の未達事項、開発フェーズと実装の進捗は[ロードマップ](ROADMAP.md)を参照する。

## 憲章

| 文書 | 内容 |
|---|---|
| [CHARTER.md](CHARTER.md) | 開発・運用全般の根本義務と絶対準拠、権限と統制、適合判定、停止・是正・再開。発効条件は[第12条](CHARTER.md#第12条-制定改定と発効)を参照 |

## 案内・作業ルール

| 文書 | 内容 |
|---|---|
| [README.md](../README.md) | プロジェクト概要、文書と報告窓口への入口 |
| [AGENTS.md](../AGENTS.md) | 憲章に基づく作業・開発・文書管理の実施手順 |

## 開発計画

| 文書 | 内容 |
|---|---|
| [ROADMAP.md](ROADMAP.md) | 開発フェーズ、内製化の段階、依存関係、完了条件、報告受付の準備状況、将来の基盤移行の時期と進捗 |

## 共通の方針・ポリシー・例外管理

| 文書 | 定義する内容 |
|---|---|
| [PRINCIPLES.md](PRINCIPLES.md) | 製品・設計・開発・配布・運用に関する判断の方針 |
| [POLICIES.md](POLICIES.md) | 方針を適用する条件、許可範囲、安全条件と既定値 |
| [DEPENDENCY_EXCEPTIONS.md](DEPENDENCY_EXCEPTIONS.md) | 外部依存の個別採用情報、用途・適用範囲、固定参照、検証・承認根拠、現在状態 |

## 製品仕様

### 共通基盤

| 文書 | 定義する内容 |
|---|---|
| [core/product-scope.md](core/product-scope.md) | 目的、製品構成、OS・アーキテクチャ・プロバイダーの対応範囲 |
| [core/common-platform.md](core/common-platform.md) | 本体導入と構成管理の分離、状態判定、操作状態、排他制御、実行記録 |
| [core/cli.md](core/cli.md) | コマンド、オプション、確認表示・文字列、JSON出力、終了コード |
| [core/configuration.md](core/configuration.md) | ファイル形式、管理ディレクトリ・ファイル配置、設定の型・必須項目、構成固有設定の参照先 |
| [core/lifecycle.md](core/lifecycle.md) | 本体導入・再導入・削除、更新・復旧手順、計画、マニフェスト、署名、バージョン |
| [core/backups.md](core/backups.md) | 実行側・対象側・保存先の分離、バックアップ状態、復元契約の未確定事項 |
| [core/usb.md](core/usb.md) | ISO生成・収録版固定、媒体マニフェスト、外部媒体による導入・保守、作成・交換 |
| [core/cloud.md](core/cloud.md) | SSH実行環境と対象OSの分離、空ディスクへのOS導入、既存OSの構成・保守、再接続と復旧 |
| [core/firmware.md](core/firmware.md) | ISO起動互換性、媒体なしの内蔵OS起動確認、機種別設定、起動項目、復電確認 |
| [core/implementation.md](core/implementation.md) | CLI・Core・構成の責任境界、内製crateの差し替え契約、systemd、リポジトリ構成 |
| [core/ci-cd.md](core/ci-cd.md) | workflow、ビルドターゲット、仕様充足・依存検査、テスト体系・追跡・受入条件、署名・リリース、デプロイ境界 |

### サーバ構成

| 文書 | 定義する内容 |
|---|---|
| [server/specification.md](server/specification.md) | Dockerアプリ、apps.yml、更新・復旧の参照先 |

### マイニング構成

| 文書 | 定義する内容 |
|---|---|
| [mining/specification.md](mining/specification.md) | 採掘対応、機器・サービス構成、運転条件の参照先 |

### マスター管理構成

| 文書 | 定義する内容 |
|---|---|
| [master/specification.md](master/specification.md) | 内包機能の統合、初期構築、固有の設定・CLI、対象管理、SSH自動処理、中断照合、構成検証と同居制御の未確定契約 |
