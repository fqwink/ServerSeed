# ServerSeed ルールブック

- 文書種別: リポジトリ運用ルールブック
- 対象リポジトリ: `fqwink/ServerSeed`

## 1. main運用

`main`ブランチへ直接pushしてはならない。

`main`への反映は、作業ブランチを作成し、GitHub上のPull Requestを経由して行う。

ローカル`main`は、GitHub上の`main`へ反映済みの内容を取得して同期するために使用する。

## 2. 作業ブランチ

変更作業は、必ず`main`以外の作業ブランチで行う。

作業ブランチ名は、内容が分かる名前にする。

```text
codex/add-repository-rulebook
codex/refine-master-spec
```

## 3. Pull Request

Pull Requestの向きは次を基本とする。

```text
作業ブランチ -> main
```

Pull Requestがマージされた後、ローカル`main`を`origin/main`と同期する。

## 4. remote設定

通常のpush先はSSH接続の`origin`とする。

```text
origin  git@github.com:fqwink/ServerSeed.git
```

バックアップ確認用のremoteはURL形式の`backup`とする。

```text
backup  https://github.com/fqwink/ServerSeed
```

`origin`は通常のfetchおよびpushに使用する。`backup`はURL確認、参照確認、緊急時の比較用とし、通常のpush先として使用しない。

## 5. 禁止事項

- `main`への直接push
- `main`上での通常作業
- `backup` remoteへの通常push
- PRを経由しない`main`反映
