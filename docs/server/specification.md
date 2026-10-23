# サーバ構成仕様

[文書一覧](../DOCUMENT_INDEX.md)

## 責任範囲

サーバ構成はDocker Engine・Compose、サーバアプリ、アプリ固有のバックアップ・検証・復旧を担当する。

構成間の適用制約は[構成の分離](../POLICIES.md#構成の分離)を参照する。

本書の機能をマスターへ組み込む手順と、管理・採掘との責任境界は[機能の統合](../master/specification.md#機能の統合)を参照する。

Docker・Composeおよびアプリ固有の外部実行物は[外部依存の例外一覧](../DEPENDENCY_EXCEPTIONS.md#外部依存の例外一覧)を参照する。

## 更新と復旧

実行許可、再起動および自動修復の制約は[サーバ構成](../POLICIES.md#サーバ構成)を参照する。

共通の処理は[更新](../BASIC_SPEC.md#本体の導入更新復旧)と[バックアップ](../BASIC_SPEC.md#バックアップとデータ保護)に従う。

## 構成固有設定

サーバ機能の運用設定の配置は[管理ファイル一覧](../BASIC_SPEC.md#管理ファイル一覧)の`server.yml`を参照する。アプリ単位の定義は[apps.yml](#appsyml)へ分離し、構成別の適用範囲と実行許可は[構成の分離](../POLICIES.md#構成の分離)と[サーバ構成の運用](../POLICIES.md#サーバ構成)に従う。

`server.yml`の形式・項目、型・必須性・許容範囲、省略・欠損・不正時の扱い、版と互換性は未確定とする。`apps.yml`の定義だけで運用設定まで確定した扱いにせず、依存する実装の着手可否は[仕様充足の判定](../POLICIES.md#実装着手と仕様充足の判定)に従う。

## `apps.yml`

操作を明示する計画生成は[AppOperation/plan_server](../complete/PHASE_06.md#内製単位api案)、親計画への結合は[P03の構築/封印](../complete/PHASE_03.md#依存と受入の確定候補)を審査候補の定義先とする。アプリ計画の配列だけを配送可能なPlanにせず、同一対象/親identity/担当stepへ結合する。サーバ機能の受入は[担当lane](../complete/PHASE_11.md#設計行から実行scopeへの束縛)へ照合し、mining専用構成やCloud masterへ成功経路を生成しない。型/操作名/数値の本文はここへ複製しない。

サーバ固有契約の採用審査用候補は[Phase 6](../complete/PHASE_06.md#p06-config)、型・APIは[型の詳細](../complete/PHASE_06.md#型とapiの詳細案)、方式別の互換境界は[依存と互換性](../complete/PHASE_06.md#依存と互換性の詳細案)を参照する。本書の既存定義は維持し、候補の存在をserver.yml・apps.ymlの受理形式の切替や採用承認と扱わない。

| 担当する追加具体案 | 定義先 |
|---|---|
| 全設定/計画/結果の型とHTTP adapter | [型と結果の対応](../complete/PHASE_06.md#型とapiの詳細案)、[health adapter](../complete/PHASE_06.md#health-urlと外部adapter) |
| Compose projectと全サービス・版別データ方法 | [Composeの範囲](../complete/PHASE_06.md#composeの範囲とデータ方法行)、[実container同定](../complete/PHASE_06.md#実containerへの束縛)、[P06-DATA](../complete/PHASE_06.md#p06-data) |
| 元稼働集合と停止/復帰、全アプリの親予算 | [データ整合](../complete/PHASE_06.md#p06-data)、[P06-APP](../complete/PHASE_06.md#p06-app) |

新形式の全キーと旧形式の扱いは[入力/保存形式](../complete/PHASE_06.md#入力と保存形式の詳細案)、停止取得/health試行の中断は[実行/復旧](../complete/PHASE_06.md#実行と復旧の確定候補)の候補へ集約する。既存backup.requiredの保護を維持し、旧backup.commandから復元/隔離検査を自動生成しない。新形式の採用・互換判断までは、下記の例を新形式に対応済みという根拠にしない。

候補の型付き計画/全サービス集合と固定CommandSpecへの束縛は[データ契約](../complete/PHASE_06.md#データ契約の具体化案)、HTTPの応答完了/符号化/全量上限は[依存境界](../complete/PHASE_06.md#依存と互換性の詳細案)、完了印と外部応答の分離は[health adapter](../complete/PHASE_06.md#health-urlと外部adapter)、アプリ版別の隔離復元は[P06-DATA](../complete/PHASE_06.md#p06-data)を参照する。本書で数値や型を複製せず、health成功とデータ復元、空アプリの正常と未選定アプリの対応を区別する。

適用後の部分結果は[AppResult列と後続未開始](../complete/PHASE_06.md#状態と中断の詳細案)、healthの試行集約は[実行/復旧](../complete/PHASE_06.md#実行と復旧の確定候補)へ参照する。先行アプリの失敗、後続の未開始、親操作の記録未完了を別に保持し、空結果や他アプリの成功へ置換しない。試験の実行割当は[Phase 11](../complete/PHASE_11.md#全ケースの実行割当案)へ対応付け、実アプリの方法行が未選定なら未達のままにする。

`apps.yml`は本書のサーバ機能が管理するDockerアプリを定義する。読み込む構成の範囲は[構成の分離](../POLICIES.md#構成の分離)に従う。

```yaml
apps:
  - name: example
    compose_file: /srv/example/docker-compose.yml
    healthcheck:
      type: http
      url: http://127.0.0.1:8080/health
    backup:
      required: true
      command: /usr/local/bin/example-backup
```

`backup.required`による更新停止条件は[バックアップの成功条件](../POLICIES.md#バックアップの成功条件)に従う。

`apps.yml`が存在しない場合、管理対象Dockerアプリなしとして扱う。空の`apps`配列は有効とする。

アプリ名は英数字、ハイフン、アンダースコアのみを許可する。`compose_file`は絶対パスとし、`/srv/`配下を推奨する。相対パス、空文字、親ディレクトリ参照を含むパスは設定エラーとする。

`healthcheck.type`は初期実装では`http`または`command`とする。`backup.command`は絶対パスとし、シェル文字列ではなく実行ファイルと引数配列へ分解できる形式にする。
