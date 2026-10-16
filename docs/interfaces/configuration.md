# 共通設定とデータ形式

[文書一覧](../DOCUMENT_INDEX.md)

## ファイル形式

人が編集するServerSeedの構成定義はYAML形式とし、ファイル拡張子は`.yml`に統一する。外部ツールが指定する受け渡しファイル名は、[USBセットアップ](../provisioning/usb.md#セットアップ)など該当する外部契約に従う。

機械処理を主目的とするマニフェストおよび状態スナップショットはJSON形式とし、追記型の実行履歴およびイベントログは1行1JSONのJSONL形式とする。

YAMLの読み込み制約は[設定入力の安全条件](../POLICIES.md#設定入力の安全条件)に従う。

秘密情報の扱いは[セキュリティ仕様](../POLICIES.md#秘密情報)に従う。

## 管理ファイル一覧

| パス | 定義先 |
|---|---|
| `/etc/serverseed/config.yml` | [基本設定](#configyml) |
| `/etc/serverseed/policy.yml` | [共通ポリシー](#policyyml) |
| `/etc/serverseed/apps.yml` | [サーバ構成のアプリ定義](../profiles/server.md#appsyml) |
| `/etc/serverseed/profiles/server.yml` | [構成固有設定](#構成固有設定) |
| `/etc/serverseed/profiles/mining.yml` | [マイニング構成のポリシー](../POLICIES.md#マイニングの自動更新と再起動) |
| `/etc/serverseed/managed` | [管理マーカー](../common-platform.md#管理マーカー) |
| `/etc/serverseed/machine-id` | [管理マーカーとの照合](../common-platform.md#管理マーカー) |
| `/etc/serverseed/version` | [共通の管理情報](../common-platform.md#共通状態判定) |
| `/etc/serverseed/trustedkeys.gpg` | [署名検証用キーリング](../operations/updates.md#署名およびチェックサム) |
| `/var/lib/serverseed/state.json` | [状態スナップショット](../common-platform.md#状態スナップショット) |
| `/var/lib/serverseed/operation.json` | [操作状態](../common-platform.md#操作状態) |
| `/var/lib/serverseed/history.jsonl` | [実行履歴](../common-platform.md#実行履歴) |
| `/var/lib/serverseed/install.json` | [本体導入情報](../common-platform.md#本体導入と構成管理) |
| `/var/lib/serverseed/release/` | [本体の置換と復旧](../operations/updates.md#本体の置換と復旧) |
| `/var/lib/serverseed/self-transaction/` | [本体の置換と復旧](../operations/updates.md#本体の置換と復旧) |
| `/var/lib/serverseed/previous/` | [本体の置換と復旧](../operations/updates.md#本体の置換と復旧) |

## 構成固有設定

構成固有設定の配置は[管理ファイル一覧](#管理ファイル一覧)、適用範囲は[構成の分離](../POLICIES.md#構成の分離)を参照する。

構成固有の定義は[サーバ構成](../profiles/server.md#appsyml)と[マイニング構成](../profiles/mining.md)を参照する。

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

`environment`は導入経路を記録し、`usb`または`cloud`とする。ISOは媒体形式であり、独立した実行環境値にしない。ISOをUSBまたは仮想メディアから起動する導入は`usb`、既存OSへの直接導入は`cloud`とする。この値だけを根拠に、現在USB起動中と判定したりディスク操作を許可したりしない。

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

`profile`の値は`server`または`mining`とする。値の確定・保存と管理マーカーとの照合は[初期構成選択と継続管理](../POLICIES.md#構成選択と継続管理)に従う。

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
