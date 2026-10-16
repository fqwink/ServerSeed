# サーバ構成仕様

[文書一覧](../DOCUMENT_INDEX.md)

## 責任範囲

サーバ構成はDocker Engine・Compose、サーバアプリ、アプリ固有のバックアップ・検証・復旧を担当する。

サーバ構成に採掘ソフトや採掘用の自動再起動ポリシーを適用しない。

## 更新と復旧

サーバ構成への適用は管理者が`serverseed update`を明示的に実行する。

サーバ構成では無許可の自動再起動を行わず、再起動要否を明示する。

サーバ構成の自動修復は初期実装の対象外とする。修復が必要な場合は、`doctor`、`verify`、`plan`、`report`で原因、影響、推奨操作を表示する。

共通の処理は[更新](../operations/updates.md)と[バックアップ](../operations/backups.md)に従う。

## `apps.yml`

`apps.yml`はサーバ構成が管理するDockerアプリを定義する。マイニング構成では適用しない。

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

アプリ定義にバックアップが必要と指定されている場合、更新前にバックアップ成功状態を確認できなければ更新しない。

`apps.yml`が存在しない場合、管理対象Dockerアプリなしとして扱う。空の`apps`配列は有効とする。

アプリ名は英数字、ハイフン、アンダースコアのみを許可する。`compose_file`は絶対パスとし、`/srv/`配下を推奨する。相対パス、空文字、親ディレクトリ参照を含むパスは設定エラーとする。

`healthcheck.type`は初期実装では`http`または`command`とする。`backup.command`は絶対パスとし、シェル文字列ではなく実行ファイルと引数配列へ分解できる形式にする。
