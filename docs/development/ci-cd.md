# CI/CDとリリース仕様

[文書一覧](../DOCUMENT_INDEX.md)

## CI/CD

CI/CDはGitHub上で実行し、workflowを実行定義とする。リポジトリ運用は[AGENTS.md](../../AGENTS.md)に従う。

CIは、変更ブランチ、`main`ブランチ、リリース候補で実行できるものとする。CDは、管理者が明示的にリリース操作を実行した場合にのみ実行する。

ServerSeed固有の検査、バイナリのビルド、USB用ISOの生成、署名、GitHub Releases公開をworkflow上で実行する。

初期workflowは次とする。

| workflow | トリガー | 目的 |
|---|---|---|
| `ci.yml` | pull request、`main`へのpush | 通常CI |
| `release.yml` | `v*`タグ、手動実行 | リリース成果物作成 |
| `vm-test.yml` | 手動実行のみ | VMを使う破壊的操作の検証 |

`vm-test.yml`は通常CIの必須条件に含めない。実ディスク、実ホスト、利用者データを対象にしてはならない。

ISOを含む正式リリースでは、手動VM試験の成功を必須とし、試験した候補ISO・本体のハッシュを公開対象と照合する。通常CIの成功を起動・導入試験の代わりにしない。

初期CIで必須とするジョブは次のとおりとする。

| ジョブ | 内容 |
|---|---|
| `format` | `cargo fmt --all -- --check`およびMarkdown整形確認 |
| `lint` | `cargo clippy --all-targets --locked -- -D warnings`および危険な外部コマンド実行の検査 |
| `test-unit` | `cargo test --lib --locked`によるunit test |
| `test-integration` | `cargo test --tests --locked`によるUbuntu 24.04 LTS上のCLI契約と、使い捨てコンテナ内での本体管理の検査 |
| `build` | `cargo build --release --locked --target TARGET`による`serverseed`バイナリ作成。`TARGET`は[配布ターゲット](implementation.md#配布バイナリのターゲットとリンク)に従う |
| `package-iso` | 検証済みビルド成果物を使うUSB用ISOの作成と内容検査 |
| `spec-check` | [文書索引の製品仕様](../DOCUMENT_INDEX.md#製品仕様)と実装上の固定値の矛盾検査 |
| `cli-contract` | サブコマンド、共通オプション、JSON出力契約の検査 |

CIで実行してはならない処理は次のとおりとする。

- 実ディスクの消去
- 実ホストのパーティション変更
- UEFIまたはBIOS設定変更
- 実ホストのSSH設定変更
- 実ホストのDockerサービス再起動
- `apt upgrade`などCI実行環境自体を変更する更新
- 許可されていないリモートリポジトリへの自動push
- 対象サーバへの自動デプロイ

VMを使用する破壊的操作の検証は、管理者が明示的に起動する専用CIジョブでのみ実行する。専用CIジョブは、テスト用ディスクイメージ以外を対象にしてはならない。

ServerSeedのCIは、GitHub workflowから少なくとも次のローカルスクリプトを呼び出せるようにする。

```bash
ci/run-format
ci/run-lint
ci/run-unit
ci/run-integration
ci/run-build
ci/run-package
ci/run-spec-check
ci/run-cli-contract
ci/run-all
```

各CIコマンドはローカル開発環境でも実行できるようにし、workflowから共通利用する。

### GitHubの役割

GitHubは次の役割を担当する。

- CI/CD workflowの起動
- CI結果の表示
- リリースタグ管理
- リリース成果物の公開または公開入口

GitHub Actionsを使用する場合、workflowは`.github/workflows/`に配置する。重複を避けるため、必要に応じて`ci/`および`release/`配下の補助スクリプトを呼び出す。

### CI実行環境

CIの標準実行環境はUbuntu 24.04 LTSコンテナまたはUbuntu 24.04 LTS仮想マシンとする。

CI実行環境には、[Rustツールチェーン方針](implementation.md#rustツールチェーン方針)に従うコンパイラー・Cargo・rustfmt・Clippy、[ISO生成](../provisioning/usb.md#isoの作成)のツールおよび署名検証ツールを事前に導入する。ビルドターゲットに必要な標準ライブラリ、リンカーおよびシステムライブラリもCI用イメージに含める。

CI実行中にホストOSを変更してはならない。依存関係の導入が必要な場合は、CI用イメージの定義を更新する。変更理由の記載は[文書の更新](../../AGENTS.md#文書の更新)に従う。

CI用イメージは、再現可能な定義ファイルから作成する。イメージ定義はリポジトリ内で管理する。

### リリース

リリースはGit tagで管理する。タグ形式は`vMAJOR.MINOR.PATCH`とする。

管理者がリリース操作を実行した場合、CDは次を生成する。

- Linux amd64用`serverseed`バイナリ
- Linux arm64用`serverseed`バイナリ
- USB起動用amd64 ISO
- USB起動用arm64 ISO
- 本体用と媒体用のマニフェスト
- 本体ビルドメタデータ
- SHA-256チェックサム
- OpenPGP detached signature

リリース成果物は、署名検証とチェックサム検証に成功したものだけを公開対象とする。

リリース成果物名は次を基本とする。

```text
serverseed_VERSION_linux_amd64
serverseed_VERSION_linux_arm64
serverseed_VERSION_usb_amd64.iso
serverseed_VERSION_usb_arm64.iso
serverseed_VERSION_build.json
serverseed_VERSION_manifest.json
serverseed_VERSION_manifest.json.asc
serverseed_VERSION_media.json
serverseed_VERSION_media.json.asc
serverseed_VERSION_checksums.txt
serverseed_VERSION_checksums.txt.asc
```

`VERSION`はGit tagの先頭`v`を除いたSemantic Versioning値とする。例として、タグ`v0.1.0`の場合は`serverseed_0.1.0_linux_amd64`とする。

リリース操作は、少なくとも次のコマンドで実行できるようにする。

```bash
release/build
release/build-iso
release/sign
release/verify
release/publish
```

`release/publish`は、署名済みかつ検証済みの成果物だけを公開する。

### リリース工程

1. 固定タグ・コミットから両アーキテクチャの本体を一度だけビルドし、[受入条件](implementation.md#配布導入の受入条件)を検証する。
2. 本体のビルドメタデータと[更新マニフェスト](../operations/updates.md#更新マニフェスト)を確定し、本番鍵でマニフェストへ署名する。これ以降、本体・メタデータ・マニフェストを変更しない。
3. `release/build-iso`に固定版の本体と署名付き情報を渡す。既存リリースから取得する場合も同じ検証を行う。ISO内のバイナリと候補バイナリをハッシュで照合し、別ビルドに置き換えない。
4. [ISOの作成](../provisioning/usb.md#isoの作成)と手動VM試験を完了し、生成したISOの情報を[媒体マニフェスト](../provisioning/usb.md#媒体マニフェスト)に記載して署名する。
5. 全配布ファイルのSHA-256一覧を作成し、一覧にも署名する。一覧自身とその署名は一覧の対象外とする。`release/verify`で相互参照とすべての必要な成果物を検査する。
6. 検証済み一式を非公開のdraft releaseへ配置し、取得後の再照合に成功した場合のみ同一タグの正式リリースとして公開する。部分失敗、成果物不足、ハッシュ変更がある場合は公開しない。

公開済みの同名成果物やタグを差し替えない。修正が必要な場合は新しいタグで生成・検証する。通常CIの成果物やテスト署名を正式版として扱わない。

### 署名鍵の扱い

リリース署名鍵はCIログに出力してはならない。

本番鍵による署名処理はリリース操作に限定する。通常CIでは本番署名鍵へアクセスできないようにする。署名機能の試験に使用する鍵は本番と明確に分離し、テスト署名の成果物は公開しない。

初期実装では署名用の実行環境を通常CIと分離する。署名鍵の保管、利用権限、ローテーション手順をリリース手順書に明記する。

### アーティファクト検査

本体の導入・更新・削除は、少なくとも次を検査する。

- [配置と権限](implementation.md#初回の取得と配置)が正しく、設定・状態・ログが保護されること
- 構成未選択のまま本体だけを導入でき、`doctor --json`がその状態を区別すること
- 再導入、署名不正、競合配置、中断および復旧を[受入条件](implementation.md#配布導入の受入条件)どおり扱うこと

生成したバイナリは、少なくとも次を検査する。

- `serverseed version`が成功すること
- `serverseed inspect --json`がJSONのみを標準出力へ出すこと
- `serverseed --help`が成功すること
- `serverseed doctor --json`がJSONのみを標準出力へ出すこと
- `serverseed plan update --json`が変更なしで計画またはブロック理由を出すこと

実行環境と実行時依存の検証は[配布バイナリのターゲットとリンク](implementation.md#配布バイナリのターゲットとリンク)に従う。新しいUbuntu環境で、RustやCargoを導入せず配布本体を実行できることも確認する。

ISOは[媒体マニフェスト](../provisioning/usb.md#媒体マニフェスト)の収録版・ハッシュと[起動互換性](../provisioning/firmware.md#isoの起動互換性)を検査する。生成成功だけで合格にせず、構成選択前の非破壊性とUSB・仮想メディア双方の起動を確認する。

### 成果物保管

CI成果物は一時保管とし、リリース成果物と区別する。

リリース成果物は、GitHub Releasesを標準の公開先とする。必要に応じて静的ファイル配布場所にも保存できる。どの公開先を使用する場合でも、成果物のチェックサム、署名、ビルドメタデータを併せて公開する。

既定の取得先は`fqwink/ServerSeed`の`vVERSION`リリースとする。ミラーやローカル搬入でもファイル名と署名対象の内容を変更しない。チェックサム一覧だけを信頼せず、[検証手順](../operations/updates.md#署名およびチェックサム)に従う。

成果物には、ビルド元commit、タグ、ビルド日時、ビルド環境識別子、チェックサム、署名情報を紐づける。

### 本体ビルドメタデータ

`serverseed_VERSION_build.json`は次の形式とする。値は形式例であり、実際の採用版・ビルド結果へ置き換える。

```json
{
  "schema_version": 1,
  "serverseed_version": "0.1.0",
  "tag": "v0.1.0",
  "commit": "commit-sha",
  "built_at": "RFC3339",
  "rustc_version": "adopted-version",
  "cargo_version": "adopted-version",
  "cargo_lock_sha256": "hex",
  "artifacts": [
    {
      "architecture": "amd64",
      "target": "x86_64-unknown-linux-gnu",
      "build_environment": "image-digest",
      "link_mode": "dynamic",
      "elf_interpreter": "/lib64/ld-linux-x86-64.so.2",
      "libraries": [
        {"soname": "libc.so.6", "required_versions": ["GLIBC_2.39"]}
      ],
      "linked_crates": []
    }
  ]
}
```

アーキテクチャごとに一件を記載し、版・タグ・コミットを本体用マニフェストとビルド対象へ照合する。`link_mode`は`static`または`dynamic`、静的リンク時の`elf_interpreter`は`null`とする。`libraries`は実際のELF依存と要求するABIの集合を記録し、`linked_crates`は組み込んだcrate名・バージョンを持つオブジェクトの配列とする。依存を推測で省略しない。

導入時は必要なローダー・ライブラリ・要求ABIをすべて照合する。条件を判定できない場合は実行可能と扱わない。本体用マニフェストの署名と、このファイルへのSHA-256参照で真正性を確認し、未検証のメタデータで依存確認を省略しない。

### デプロイ方針

CI/CDから対象端末へ自動デプロイしない。

CDが行うのは、成果物のビルド、検査、署名、公開までとする。端末側への適用方法は[サーバ構成](../profiles/server.md#更新と復旧)と[自動更新と再起動](../profiles/mining.md#自動更新と再起動)に従う。

自動更新の前提条件と確認方式も、マイニング構成の定義を参照する。
