# ServerSeed 開発ロードマップ

[文書一覧](DOCUMENT_INDEX.md)

現在はPhase 0の文書整備を進めている。実装コード、自動テストおよびCI定義は未作成で、Phase 1以降は未着手とする。

Phase 1ではCIの開発基盤を整備し、機能別の検証・配布ジョブは対応するフェーズで実装する。Phase 8で[CI/CD仕様](development/ci-cd.md#cicd)に定める初期CI全体を検証する。

## Phase 0: 仕様と文書体系

- 対象: [文書索引](DOCUMENT_INDEX.md)、責務別・構成別仕様、作業ルールの整理。
- 完了条件: 各仕様の定義先が明確で、文書一覧が実ファイルを網羅し、参照リンクとコード例の検証が完了している。

## Phase 1: CLIと開発基盤

- 依存: Phase 0。
- 対象: [CLI](interfaces/cli.md)、[実装方針](development/implementation.md#実装方針)、[CI/CD](development/ci-cd.md)の開発基盤。
- 完了条件: [Rustツールチェーン方針](development/implementation.md#rustツールチェーン方針)に従って採用条件が決まり、CLIの起動・ヘルプ・バージョン表示・共通出力を検証でき、[CIジョブ](development/ci-cd.md#cicd)の整形・静的解析・ビルド・unit testをローカルおよびCIから実行できる。

## Phase 2: 共通基盤と構成選択

- 依存: Phase 1。
- 対象: [共通基盤](common-platform.md)、[共通設定](interfaces/configuration.md)、[OS・アーキテクチャ判定](product-scope.md#対応範囲)。
- 完了条件: 本体導入状態と管理状態の分離、初回構成選択、構成の保存・整合性検査、操作状態、排他制御、実行記録を検証でき、読み取り専用操作と変更計画を実行できる。

## Phase 3: バイナリ導入と共通の更新・保護・復旧

- 依存: Phase 2。
- 対象: [バイナリ導入](development/implementation.md#バイナリ導入)、[self](interfaces/cli.md#self)、[更新・署名検証](operations/updates.md)、[セキュリティ](operations/security.md)、[バックアップ](operations/backups.md)。
- 完了条件: 初回の信頼確立、本体導入・再導入・更新・削除、前提条件による停止、署名不正、バックアップ未確認、中断からの復旧と切り戻しを隔離環境で検証できる。

## Phase 4: サーバ構成とマイニング構成の導入

- 依存: Phase 3。
- 対象: [サーバ構成](profiles/server.md)、[マイニング構成](profiles/mining.md)。
- 完了条件: 選択した構成の導入・再実行・検証が成立し、未選択構成へ変更しないことを確認できる。機器固有の導入結果は対応実機で確認できる。

## Phase 5: マイニング運転と自動保守

- 依存: Phase 4。
- 対象: マイニング構成の[監視と自動復旧](profiles/mining.md#監視と自動復旧)、[自動更新と再起動](profiles/mining.md#自動更新と再起動)、[更新失敗時の復旧](profiles/mining.md#更新失敗時の復旧)。
- 完了条件: 採掘停止、通信断、温度異常、再試行上限、更新失敗を検証でき、CPU・GPU別の運転と構成ポリシーに沿った復旧・更新が成立する。

## Phase 6: USB用ISOとファームウェア連携

- 依存: Phase 5。
- 対象: [USB導入](provisioning/usb.md)、[ISO生成](provisioning/usb.md#isoの作成)、[BIOS・UEFI](provisioning/firmware.md)。
- 完了条件: 共通バイナリとハッシュが一致するISOを生成し、VMでUSB・仮想メディア起動から導入・更新・復旧までを検証できる。対応実機では、ファームウェア設定の確認結果と復電起動の確認結果を区別して記録できる。

## Phase 7: Cloud導入とcloud-init

- 依存: Phase 6。
- 対象: [Cloud導入](provisioning/cloud.md)。SSHからの導入を先に整備し、cloud-initによる導入準備へ進む。
- 完了条件: ISOを取得せず共通バイナリを導入でき、既存の管理経路を保護し、cloud-initによる準備後も利用者が対話操作で初回構成を選択する流れを検証できる。

## Phase 8: 統合検証と初期リリース

- 依存: Phase 7。
- 対象: [テスト方針](development/implementation.md#テスト方針)、[リリース](development/ci-cd.md#リリース)、[アーティファクト検査](development/ci-cd.md#アーティファクト検査)。
- 完了条件: [配布・導入の受入条件](development/implementation.md#配布導入の受入条件)と手動VM試験を満たし、共通バイナリ・USB用ISO・検証情報を同一リリースとして生成・署名・検証して公開できる。
