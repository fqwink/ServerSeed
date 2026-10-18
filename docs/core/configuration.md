# 共通設定とデータ形式

[文書一覧](../DOCUMENT_INDEX.md)

## ファイル形式

人が編集するServerSeedの構成定義はYAML形式とし、ファイル拡張子は`.yml`に統一する。外部ツールが指定する受け渡しファイル名は、[USBセットアップ](usb.md#セットアップ)など該当する外部契約に従う。

機械処理を主目的とするマニフェストおよび状態スナップショットはJSON形式とし、追記型の実行履歴およびイベントログは1行1JSONのJSONL形式とする。

YAMLの読み込み制約は[設定入力の安全条件](../POLICIES.md#設定入力の安全条件)に従う。

秘密情報の扱いは[セキュリティ仕様](../POLICIES.md#秘密情報)に従う。

## 構成定義の解析

YAML・JSONの解析はCoreの共通処理とし、各設定・マニフェスト・記録の型と意味は担当仕様へ照合する。宣言型管理へ渡す情報の区分は[入力と状態](declarative-configuration.md#入力と状態の区分)、資源の項目と管理範囲は[資源定義の未確定契約](declarative-configuration.md#未確定契約)を参照する。解析できたことを、適用可能性や権限の確認の代わりにしない。

対応するYAML・JSONの版と構文範囲、文字コード、文字列・数値・真偽値・nullの解釈、重複キー・複数文書・参照の扱い、入力サイズ・深さ・解析時間・メモリーの上限と、制限超過・不正入力時の出力は未確定とする。対応しない構文の扱いも契約として確定し、形式名だけで完全互換を保証しない。[設定入力の安全条件](../POLICIES.md#設定入力の安全条件)と、各担当仕様の未知項目・型不正時の扱いは維持する。

本節では新しい設定キー、資源定義ファイルや保存形式を追加しない。既存設定との接続・版・移行を含む解析契約と受入条件が確定するまで、依存する解析・適用処理を[実装着手可能](../POLICIES.md#実装着手と仕様充足の判定)と判定しない。

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

## 管理ファイル一覧

| パス | 定義先 |
|---|---|
| `/etc/serverseed/config.yml` | [基本設定](#configyml) |
| `/etc/serverseed/policy.yml` | [共通ポリシー](#policyyml) |
| `/etc/serverseed/apps.yml` | [サーバ構成のアプリ定義](../server/specification.md#appsyml) |
| `/etc/serverseed/profiles/server.yml` | [サーバ機能の構成固有設定](../server/specification.md#構成固有設定) |
| `/etc/serverseed/profiles/mining.yml` | [採掘サービスと設定](../mining/specification.md#採掘サービスと設定) |
| `/etc/serverseed/managed` | [管理マーカー](common-platform.md#管理マーカー) |
| `/etc/serverseed/machine-id` | [管理マーカーとの照合](common-platform.md#管理マーカー) |
| `/etc/serverseed/version` | [共通の管理情報](common-platform.md#共通状態判定) |
| `/etc/serverseed/trustedkeys.gpg` | [署名検証用キーリング](lifecycle.md#署名およびチェックサム) |
| `/var/lib/serverseed/state.json` | [状態スナップショット](common-platform.md#状態スナップショット) |
| `/var/lib/serverseed/operation.json` | [操作状態](common-platform.md#操作状態) |
| `/var/lib/serverseed/history.jsonl` | [実行履歴](common-platform.md#実行履歴) |
| `/var/lib/serverseed/install.json` | [本体導入情報](common-platform.md#本体導入と構成管理) |
| `/var/lib/serverseed/release/` | [本体の置換と復旧](lifecycle.md#本体の置換と復旧) |
| `/var/lib/serverseed/self-transaction/` | [本体の置換と復旧](lifecycle.md#本体の置換と復旧) |
| `/var/lib/serverseed/previous/` | [本体の置換と復旧](lifecycle.md#本体の置換と復旧) |

共通ログの保存先は次のとおりとする。記録形式と秘匿化の参照先は[ログ形式](common-platform.md#ログ形式)に従う。

```text
/var/log/serverseed/serverseed.log
/var/log/serverseed/events.jsonl
```

## 構成固有設定

構成固有設定の配置は[管理ファイル一覧](#管理ファイル一覧)、適用範囲は[構成の分離](../POLICIES.md#構成の分離)を参照する。

構成固有の形式・項目と未確定事項は[マスター管理構成](../master/specification.md#設定と管理情報)、[サーバ機能](../server/specification.md#構成固有設定)と[マイニング機能](../mining/specification.md#採掘サービスと設定)を参照する。許可条件・既定値は各担当仕様から[運用ポリシー](../POLICIES.md#構成別の運用)へ対応付ける。同居しても機能別の設定を別形式へ複製しない。

## `config.yml`

`config.yml`はServerSeed全体の基本設定を定義する。

生成時期と既存設定の扱いは[初回導入の保護条件](../POLICIES.md#初回導入の保護条件)に従う。以下は形式例であり、未指定時の値は[基本設定の既定値](../POLICIES.md#基本設定の既定値)を参照する。

```yaml
serverseed:
  config_revision: 1
  environment: cloud
  profile: server
  log_level: info
  allow_reboot: false
  update_channel: stable
```

`environment`は導入経路を記録し、`usb`または`cloud`とする。ISOは媒体形式であり、独立した実行環境値にしない。ISOを外部媒体から起動して物理PCへ導入する経路は`usb`、マスターからSSHでCloud対象へ導入する経路は`cloud`とする。後者はOS導入の有無に依存せず、SSH用に仮想メディアを使用するだけで`usb`へ変えない。この値だけを根拠に、現在の実行環境を判定したりディスク操作を許可したりしない。

`config.yml`の必須項目は次とする。

| 項目 | 型 | 必須 |
|---|---|---|
| `serverseed.config_revision` | integer | はい |
| `serverseed.environment` | string | はい |
| `serverseed.profile` | string | はい |
| `serverseed.log_level` | string | いいえ |
| `serverseed.allow_reboot` | boolean | いいえ |
| `serverseed.update_channel` | string | いいえ |

`log_level`は`debug`、`info`、`warn`、`error`のいずれかとする。`update_channel`は初期実装では`stable`のみ有効とする。

`profile`は単一のstringとし、許容値と対応する構成は次のとおりとする。配列や複数値の連結文字列は受け付けない。値の確定・保存と管理マーカーとの照合は[初期構成選択と継続管理](../POLICIES.md#構成選択と継続管理)に従う。

| 値 | 構成 |
|---|---|
| `master` | マスター管理構成 |
| `mining` | マイニング構成 |
| `server` | サーバ構成 |

未知のトップレベルキーは設定エラーとする。未知の下位キーは警告ではなく設定エラーとし、変更操作を開始しない。

## `policy.yml`

`policy.yml`の項目と型は次のとおりとする。以下の値は形式例であり、既定値と許可条件は[共通ポリシーの既定値](../POLICIES.md#共通ポリシーの既定値)を参照する。

```yaml
policy:
  require_backup_before_update: true
  require_ssh_recovery_path: true
  allow_password_auth_disable: true
  allow_unattended_security_updates: true
  allow_destructive_disk_setup: false
```

ディスク消去の許可条件は[ディスク操作](../POLICIES.md#ディスク操作)を参照する。

`policy.yml`の必須項目は次とする。

| 項目 | 型 | 必須 |
|---|---|---|
| `policy.require_backup_before_update` | boolean | いいえ |
| `policy.require_ssh_recovery_path` | boolean | いいえ |
| `policy.allow_password_auth_disable` | boolean | いいえ |
| `policy.allow_unattended_security_updates` | boolean | いいえ |
| `policy.allow_destructive_disk_setup` | boolean | いいえ |

型が一致しない場合は設定エラーとする。

## 構成記録の互換性

`config.yml`と管理記録の対応は[共通基盤](common-platform.md#初期構成選択と継続管理)、内包機能の対応は[構成の分離](../POLICIES.md#構成の分離)を参照する。既存の構成値の意味は維持し、マスター固有の設定形式は[担当仕様](../master/specification.md#設定と管理情報)へ集約する。

本書の値追加だけで、旧二択形式を扱う本体との読み書き互換性を保証しない。構成設定・管理記録・更新情報の版の扱い、構成リビジョンと互換範囲、旧形式からの移行・検証・切り戻しの詳細は未確定とする。既存例の版番号を、移行や新形式の受入検証が完了した根拠にしない。依存する実装は[仕様充足](../POLICIES.md#実装着手と仕様充足の判定)を確認してから行う。
