# サーバ構成仕様

[文書一覧](../DOCUMENT_INDEX.md)

## 責任範囲

サーバ構成はDocker Engine・Compose、サーバアプリ、アプリ固有のバックアップ・検証・復旧を担当する。

構成間の適用制約は[構成の分離](../POLICIES.md#構成の分離)を参照する。

Docker・Composeおよびアプリ固有の外部実行物は[外部依存の例外一覧](../DEPENDENCY_EXCEPTIONS.md#外部依存の例外一覧)を参照する。

## 更新と復旧

実行許可、再起動および自動修復の制約は[サーバ構成](../POLICIES.md#サーバ構成)を参照する。

共通の処理は[更新](../core/operations/lifecycle.md)と[バックアップ](../core/operations/backups.md)に従う。

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

`backup.required`による更新停止条件は[バックアップの成功条件](../POLICIES.md#バックアップの成功条件)に従う。

`apps.yml`が存在しない場合、管理対象Dockerアプリなしとして扱う。空の`apps`配列は有効とする。

アプリ名は英数字、ハイフン、アンダースコアのみを許可する。`compose_file`は絶対パスとし、`/srv/`配下を推奨する。相対パス、空文字、親ディレクトリ参照を含むパスは設定エラーとする。

`healthcheck.type`は初期実装では`http`または`command`とする。`backup.command`は絶対パスとし、シェル文字列ではなく実行ファイルと引数配列へ分解できる形式にする。
