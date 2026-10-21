# ServerSeed 文書索引

- 文書種別: 文書索引
- 対象リポジトリ: `fqwink/ServerSeed`

## 仕様版情報

- 仕様リビジョン: 0.30
- 確定日: 2026-10-16
- ステータス: 確定

本欄の確定は承認済みの文書版を表し、製品全体の完全仕様化や実装完了を意味しない。仕様充足の判定は[開発ポリシー](POLICIES.md#実装着手と仕様充足の判定)、現在の未達事項、開発フェーズと実装の進捗は[ロードマップ](ROADMAP.md)を参照する。

フェーズ別完全仕様の本文案は下記の案内から確認する。各案の作成承認と契約内容の採用承認は別であり、採用未承認案を本欄の確定済み仕様版へ含めない。

## 文書体系

確認目的ごとの参照先は次のとおり。定義・条件の本文は各正本を参照し、本索引に複製しない。

| 確認目的 | 参照先 |
|---|---|
| 文書の定義・集約単位と基本仕様の記載内容 | [憲章第3条](CHARTER.md#第3条-文書の責務と正本)、[基本仕様の必須内容](CHARTER.md#基本仕様の必須内容) |
| 策定と実装の関係・策定権限 | [憲章第4条](CHARTER.md#第4条-仕様による実装の拘束)、[第5条](CHARTER.md#第5条-承認と権限) |
| 完全仕様の成立・内製機能とcrateの必須契約 | [成立条件](CHARTER.md#完全仕様の成立条件)、[内製の必須契約](CHARTER.md#内製機能とcrateの完全仕様)、[完全仕様の必須条件](POLICIES.md#完全仕様の必須条件) |
| 内製化の判断・差し替えの区分と工程 | [標準機能と内製化](CHARTER.md#標準機能と内製化の判断)、[差し替えの定義](CHARTER.md#内製crateの差し替え)、[工程](CHARTER.md#内製化と差し替えの工程)、[確認条件](POLICIES.md#内製化と差し替えの確認条件)、[実施手順](../AGENTS.md#内製化と差し替えの実施手順) |
| 判定区分・工程通過・憲章の発効 | [憲章第9条](CHARTER.md#第9条-適合判定の責任)、[第11条](CHARTER.md#第11条-再開と工程通過)、[第12条](CHARTER.md#第12条-制定改定と発効) |
| 製品全体の基本仕様と構成固有の定義 | [基本仕様](BASIC_SPEC.md)、[構成別仕様](#構成別仕様) |
| 機能名・概要・詳細の参照先 | [機能一覧](BASIC_SPEC.md#機能一覧) |
| 要求・受入条件・未達箇所の追跡 | [要求と受入条件](BASIC_SPEC.md#受入と追跡)、[受入検証・未達箇所の対応](BASIC_SPEC.md#要件別の受入未達参照)、[未確定事項の所在](BASIC_SPEC.md#未確定事項) |
| 内製機能とcrateの未確定部分 | [内製契約の所在](BASIC_SPEC.md#内製機能とcrateの未確定契約)、[現在の未達](ROADMAP.md#仕様充足の現在地) |
| 共通CLI・機種別操作の未確定部分 | [共通CLI・調査成果物](BASIC_SPEC.md#共通cliの未確定契約)、[ファームウェア](BASIC_SPEC.md#ファームウェアの未確定契約) |
| 計画・操作記録・排他・ログの未確定部分 | [共通管理記録](BASIC_SPEC.md#共通管理記録の未確定契約)、[更新計画](BASIC_SPEC.md#更新計画) |
| ポリシー設定・外部実行の未確定部分 | [バックアップ設定](POLICIES.md#バックアップ設定の未確定条件)、[セキュリティ設定](SECURITY_POLICY.md#セキュリティ設定の未確定条件)、[外部コマンド](BASIC_SPEC.md#外部コマンドの未確定契約) |
| VM・実機検証の適用境界 | [検証環境の未確定条件](SECURITY_POLICY.md#検証環境の未確定条件)、[テスト体系](BASIC_SPEC.md#テスト体系) |
| 配置先・本体の収録とビルド検証 | [管理ファイル一覧](BASIC_SPEC.md#管理ファイル一覧)、[組込み資材](BASIC_SPEC.md#組込み資材) |
| セキュリティ・収録禁止条件・署名鍵管理 | [セキュリティポリシー](SECURITY_POLICY.md)、[本体への収録制限](SECURITY_POLICY.md#本体への収録制限)、[リリース署名鍵](SECURITY_POLICY.md#リリース署名鍵) |
| 起動順・配布本体の検証 | [推奨起動順序](BASIC_SPEC.md#推奨起動順序)、[ターゲットとリンク](BASIC_SPEC.md#配布バイナリのターゲットとリンク) |
| CIの環境・実行区分と成果物の公開先・検証情報 | [CI/CD仕様](BASIC_SPEC.md#cicd)、[成果物保管](BASIC_SPEC.md#成果物保管) |
| 文書体系の移行・策定手順 | [ロードマップ](ROADMAP.md#文書体系の移行)、[作業ルール](../AGENTS.md#基本仕様と完全仕様の策定手順) |

フェーズ別完全仕様の格納先は[製品仕様の案内](#フェーズ別完全仕様)、作成状況と未達事項は上表の移行状況と[ロードマップ](ROADMAP.md)を参照し、仕様充足の判定と区別する。

## 憲章

| 文書 | 内容 |
|---|---|
| [CHARTER.md](CHARTER.md) | 開発・運用全般の根本義務と絶対準拠、権限と統制、適合判定、停止・是正・再開。発効条件は[第12条](CHARTER.md#第12条-制定改定と発効)を参照 |

## 案内・作業ルール

| 文書 | 内容 |
|---|---|
| [README.md](../README.md) | プロジェクト概要、文書と報告窓口への入口 |
| [AGENTS.md](../AGENTS.md) | 憲章に基づく作業・開発・文書管理の実施手順 |
| [DOCUMENT_INDEX.md](DOCUMENT_INDEX.md) | 文書の所在と担当正本 |

## 開発計画

| 文書 | 内容 |
|---|---|
| [ROADMAP.md](ROADMAP.md) | 開発フェーズ、内製化の段階、依存関係、完了条件、移行と未達事項、報告受付の準備状況、将来の基盤移行の時期と進捗 |

## 方針・ポリシー・例外管理

| 文書 | 定義する内容 |
|---|---|
| [PRINCIPLES.md](PRINCIPLES.md) | 製品・設計・開発・配布・運用に関する判断の方針 |
| [POLICIES.md](POLICIES.md) | セキュリティ以外の許可・禁止・適用条件と既定値 |
| [SECURITY_POLICY.md](SECURITY_POLICY.md) | セキュリティの要求、許可・禁止・適用条件、秘密情報、認証・権限、真正性検証、報告・検証環境・公開時の保護 |
| [DEPENDENCY_EXCEPTIONS.md](DEPENDENCY_EXCEPTIONS.md) | 外部依存の個別採用情報、用途・適用範囲、固定参照、検証・承認根拠、現在状態 |

## 製品仕様

| 文書 | 定義する内容 |
|---|---|
| [BASIC_SPEC.md](BASIC_SPEC.md) | 製品全体の基本仕様。目的・利用、対象範囲、実行環境、全体構成、運用方式、共通機能、情報管理、非機能、継続性、提供・保守、受入・追跡、未確定事項 |

### フェーズ別完全仕様

| 格納先 | 配置・命名の確認先 | 策定状況の確認先 |
|---|---|---|
| [complete/](complete/) | [作業ルール](../AGENTS.md#完全仕様書の配置) | [文書体系の移行](ROADMAP.md#文書体系の移行) |

`complete/.gitkeep`は格納先をGit管理に残す空ファイルであり、完全仕様書ではない。

| 本文案 | 担当する具体化 | 契約の確認先 |
|---|---|---|
| [PHASE_01.md](complete/PHASE_01.md) | 開発・検証基盤とCLI骨格、全案の適用範囲 | [契約と要件](complete/PHASE_01.md#要件と境界) |
| [PHASE_02.md](complete/PHASE_02.md) | 共通型・CLI・解析・構成選択・記録・外部実行 | [契約一覧](complete/PHASE_02.md#要件と契約の対応)、[CLI詳細案B2](complete/PHASE_02.md#p02-cli)、[解析詳細案B2](complete/PHASE_02.md#p02-parse) |
| [PHASE_03.md](complete/PHASE_03.md) | IaCの資源・観測・差分・計画 | [契約一覧](complete/PHASE_03.md#契約と共通参照) |
| [PHASE_04.md](complete/PHASE_04.md) | 本体管理・更新・保全・復旧・IaC適用 | [契約一覧](complete/PHASE_04.md#契約と依存) |
| [PHASE_05.md](complete/PHASE_05.md) | ISO・外部媒体・引継ぎ・ファームウェア | [契約一覧](complete/PHASE_05.md#契約と参照) |
| [PHASE_06.md](complete/PHASE_06.md) | サーバ設定・アプリとデータ保全 | [契約一覧](complete/PHASE_06.md#契約と共通参照) |
| [PHASE_07.md](complete/PHASE_07.md) | 採掘機器・設定・監視・復旧・更新 | [契約一覧](complete/PHASE_07.md#契約と共通参照) |
| [PHASE_08.md](complete/PHASE_08.md) | 物理マスター・台帳・同居・固有保全 | [契約一覧](complete/PHASE_08.md#契約と参照) |
| [PHASE_09.md](complete/PHASE_09.md) | SSH・要求配送・再接続・中断照合 | [契約一覧](complete/PHASE_09.md#契約と共通参照) |
| [PHASE_10.md](complete/PHASE_10.md) | 一時環境・OS導入・起動不能時復旧 | [契約一覧](complete/PHASE_10.md#契約と共通参照) |
| [PHASE_11.md](complete/PHASE_11.md) | 全体受入・成果物照合・リリース | [契約一覧](complete/PHASE_11.md#契約と依存範囲) |

案の位置付け・照合元・案版と承認済み版の区別は[案の適用範囲](complete/PHASE_01.md#案の適用範囲)、策定・実装・受入の状態は[フェーズ別の進捗](ROADMAP.md#フェーズ別の進捗)を参照する。

全フェーズの本文案B2には、型・API、状態/中断、依存/互換性、受入条件の充足照合を各本文の同名節へ収録する。共通参照の型は[Phase 2](complete/PHASE_02.md#型とapiの詳細案)、資材識別は[Phase 5](complete/PHASE_05.md#型とapiの詳細案)、全体の13必須項目と不足は[Phase 11](complete/PHASE_11.md#受入条件の充足照合案)へ案内し、本文を本索引へ複製しない。設定・保護条件の未採用案は[一般ポリシー](POLICIES.md#完全仕様化に伴うバックアップ設定の審査候補)、[セキュリティポリシー](SECURITY_POLICY.md#完全仕様化に伴う審査候補)、[例外候補](DEPENDENCY_EXCEPTIONS.md#追加の個別審査候補)の各担当箇所を参照する。

### 構成別仕様

構成固有の本文は以下の各個別仕様を正本とし、基本仕様へ複製しない。これらを構成別の基本仕様やフェーズ別完全仕様と扱わない。

| 文書 | 定義する内容 |
|---|---|
| [master/specification.md](master/specification.md) | マスター管理構成固有の統合、初期構築、設定・CLI、対象管理、SSH制御、中断照合と同居制御 |
| [mining/specification.md](mining/specification.md) | マイニング構成固有の対応機器、採掘方式、サービス・設定と共通機能との接続 |
| [server/specification.md](server/specification.md) | サーバ構成固有のDockerアプリ、運用設定、apps.ymlと共通機能との接続 |
