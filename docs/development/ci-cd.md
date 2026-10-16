# CI/CDとリリース仕様

[文書一覧](../DOCUMENT_INDEX.md)

## CI/CD

実行基盤と責任分担は[CI/CD](../PRINCIPLES.md#cicd)、リポジトリ運用は[AGENTS.md](../../AGENTS.md)を参照する。

実行の許可条件は[CIとリリースの実行許可](../POLICIES.md#ciとリリースの実行許可)を参照する。

workflowごとの処理は次の定義に従う。

初期workflowは次とする。

| workflow | トリガー | 目的 |
|---|---|---|
| `ci.yml` | pull request、`main`へのpush | 通常CI |
| `release.yml` | `v*`タグ、手動実行 | リリース成果物作成 |
| `vm-test.yml` | 手動実行のみ | VMを使う破壊的操作の検証 |

通常CIとVM試験の適用条件は[CIの隔離](../POLICIES.md#ciの隔離)を参照する。

正式公開の検証条件は[公開前の検証](../POLICIES.md#公開前の検証)を参照する。

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

実行環境での禁止操作は[CIの隔離](../POLICIES.md#ciの隔離)を参照する。

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

標準環境は[CI実行基盤](../PRINCIPLES.md#ci実行基盤)を参照する。

CI実行環境には、[Rustツールチェーン方針](../PRINCIPLES.md#rustツールチェーン)に従うコンパイラー・Cargo・rustfmt・Clippy、[ISO生成](../provisioning/usb.md#isoの作成)のツールおよび署名検証ツールを事前に導入する。ビルドターゲットに必要な標準ライブラリ、リンカーおよびシステムライブラリもCI用イメージに含める。

環境変更の条件は[CI環境の変更](../POLICIES.md#ci環境の変更)に従う。

イメージ定義の管理は[CI実行基盤](../PRINCIPLES.md#ci実行基盤)を参照する。

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

公開対象の選定は[公開前の検証](../POLICIES.md#公開前の検証)に従う。

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

`release/publish`の許可条件も[公開前の検証](../POLICIES.md#公開前の検証)に従う。

### リリース工程

1. 固定タグ・コミットから両アーキテクチャの本体を一度だけビルドし、[受入条件](implementation.md#配布導入の受入条件)を検証する。
2. 本体のビルドメタデータと[更新マニフェスト](../operations/updates.md#更新マニフェスト)を確定し、本番鍵でマニフェストへ署名する。これ以降、本体・メタデータ・マニフェストを変更しない。
3. `release/build-iso`に固定版の本体と署名付き情報を渡す。既存リリースから取得する場合も同じ検証を行う。ISO内のバイナリと候補バイナリをハッシュで照合し、別ビルドに置き換えない。
4. [ISOの作成](../provisioning/usb.md#isoの作成)と手動VM試験を完了し、生成したISOの情報を[媒体マニフェスト](../provisioning/usb.md#媒体マニフェスト)に記載して署名する。
5. 全配布ファイルのSHA-256一覧を作成し、一覧にも署名する。一覧自身とその署名は一覧の対象外とする。`release/verify`で相互参照とすべての必要な成果物を検査する。
6. 検証済み一式を非公開のdraft releaseへ配置し、取得後の再照合に成功した場合のみ同一タグの正式リリースとして公開する。部分失敗、成果物不足、ハッシュ変更がある場合は公開しない。

公開後の扱いは[公開物の固定](../POLICIES.md#公開物の固定)を参照する。

### 署名鍵の扱い

鍵の分離・保管・利用は[リリース署名鍵](../POLICIES.md#リリース署名鍵)を参照する。

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

成果物の保管区分と公開先は[成果物の配布](../PRINCIPLES.md#成果物の配布)を参照する。

既定の取得先とミラーの条件は[更新元](../POLICIES.md#更新元)を参照する。

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

### 端末側への適用

端末側への適用責任は[CI/CD](../PRINCIPLES.md#cicd)、自動デプロイの禁止は[CIの隔離](../POLICIES.md#ciの隔離)、自動更新の許可条件は[マイニングの自動更新と再起動](../POLICIES.md#マイニングの自動更新と再起動)を参照する。
