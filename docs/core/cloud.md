# Cloud環境への導入

[文書一覧](../DOCUMENT_INDEX.md)

## ServerSeed Cloud

### 役割

ServerSeed Cloudは、VPSまたはクラウド事業者が準備した対応OSへServerSeedを導入し、共通定義に基づいて構成、更新および検証する。

導入後の操作入口は`serverseed`とする。

### 前提

OS準備とServerSeedの責任境界は[Cloud導入](../PRINCIPLES.md#cloud導入)を参照する。

本体の導入経路は次をサポートし、VPSへISOを取得・接続することを必須にしない。

- cloud-init
- SSHからのセットアップ
- ローカルへ搬入した署名付きバイナリと検証情報

cloud-initの役割は[Cloud導入](../PRINCIPLES.md#cloud導入)に従う。初回の構成選択は[初期構成選択と継続管理](../POLICIES.md#構成選択と継続管理)に従い、利用者がSSHまたはコンソールから行う。選択完了後に構成適用を開始する。

### 本体の導入

利用者が事業者のUbuntuイメージを選び、SSHまたはコンソールの管理経路を確認した後、[初回の取得と配置](lifecycle.md#初回の取得と配置)に従って本体を導入する。コンパイラーやCargoを対象VPSへ導入することは要求しないが、[実行時依存](ci-cd.md#配布バイナリのターゲットとリンク)は確認する。

非対話での準備と既存設定の扱いは[cloud-init](../POLICIES.md#cloud-init)に従う。

準備完了後にSSHまたはコンソールで`serverseed setup`を実行する。構成選択の条件は[構成選択と継続管理](../POLICIES.md#構成選択と継続管理)に従う。

### 初期セットアップ

`ADOPTABLE`状態の対応OSに対して、次を行う。

1. 利用者による構成の手動選択と、OS、systemd、ネットワークおよび空き容量の検査
2. 現在のSSH接続および管理ユーザーの保護
3. 基本セキュリティ設定
4. 選択済み構成に必要なパッケージおよび実行基盤の導入
5. バックアップ、ログおよび監視設定
6. ServerSeed管理情報の作成
7. 最終検証

既存管理経路とルートディスクの保護は[Cloudの管理経路](../POLICIES.md#cloudの管理経路)に従う。

### 更新と起動不能時の復旧

通常の構成・OS更新は[更新処理](lifecycle.md#更新処理)、本体だけの更新は[self](cli.md#self)を使用する。ISOとOS再導入の扱いは[Cloudの管理経路](../POLICIES.md#cloudの管理経路)を参照する。

OSを起動できない場合は、利用者が事業者のレスキュー環境または対応Ubuntuの再作成を用意し、管理経路を回復してから、検証済み本体の導入と[バックアップからの復元確認](../POLICIES.md#復元確認)を行う。OS再作成と事業者APIの扱いは[Cloudの管理経路](../POLICIES.md#cloudの管理経路)に従う。

独自ISOを接続できる環境では[USB用ISO](usb.md)を仮想メディアとして使用できる。再導入と未対応環境の停止条件は[Cloudの管理経路](../POLICIES.md#cloudの管理経路)に従う。
