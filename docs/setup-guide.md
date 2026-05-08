# セットアップガイド：OpenShift Pipelines & GitOps ハンズオン

## 前提条件

- OpenShift Container Platform へのアクセス（`oc` CLI ログイン済み）
- OpenShift Pipelines Operator がインストール済み
- OpenShift GitOps Operator がインストール済み
- Git リポジトリが 3 つ用意されていること
  - **Backend リポジトリ** — Java Spring Boot ソースコード
  - **Frontend リポジトリ** — Nuxt.js ソースコード
  - **Manifest リポジトリ** — Kustomize マニフェスト一式（app-manifests）

> リポジトリ構成やアーキテクチャの全体像は [README.md](../README.md) を参照してください。

---

## Step 1. Namespace の作成

このハンズオンでは 2 つの Namespace を使用します。

| Namespace | 用途 |
|-----------|------|
| `dev-handson` | Tekton Pipeline / PaC Repository / Secret |
| `dev-handson-app` | Argo CD インスタンス・Application / デプロイされるアプリケーション |

```bash
oc new-project dev-handson
oc new-project dev-handson-app
```

> `oc new-project` の代わりに `oc create namespace` でも構いません。

---

## （オプション）OpenShift Dev Spaces 用ワークスペースイメージのビルド

本リポジトリの `devspaces/Dockerfile` は、`registry.redhat.io/devspaces/udi-rhel9` をベースに **OpenJDK 25** と **Node.js 24** を追加したカスタム Universal Developer Image (UDI) です。Dev Spaces の devfile からこのイメージを参照すると、ハンズオン用 Backend（Java 25）および Frontend（Nuxt）を同一ワークスペースで開発しやすくなります。

ビルド定義は `setup/buildconfig-udi-workspace-image.yaml` にあります。**ImageStream と BuildConfig はいずれも `openshift` Namespace に配置する想定**です。適用・ビルド開始には **クラスタ管理者権限** が必要になることが多いです。

### 編集が必要な項目

| 変更箇所 | 説明 |
|----------|------|
| `BuildConfig.spec.source.git.uri` | **メイン教材リポジトリ**（本リポジトリ、`devspaces/Dockerfile` がルート直下にある構成）の HTTPS URL |

Git の clone ルートが **`main/` サブディレクトリだけ**（モノレポなど）の場合は、`spec.source` に `contextDir: main` を追加してください。

ベース UDI のタグをクラスタの Dev Spaces バージョンに合わせて固定したい場合は、`dockerStrategy.buildArgs` の `UDI_TAG` を変更します。

### registry.redhat.io の認証

Docker ビルド時に `registry.redhat.io/devspaces/udi-rhel9` を pull できる必要があります。クラスタ既定で足りない場合は、`openshift` Namespace の **`builder` ServiceAccount** に Red Hat Container Registry 用の pull Secret をリンクします。

```bash
oc create secret docker-registry redhat-registry-pull-secret \
  --docker-server=registry.redhat.io \
  --docker-username='<Red Hat アカウントまたはトークン用ユーザー名>' \
  --docker-password='<registry 用パスワードまたはトークン>' \
  -n openshift
oc secrets link builder redhat-registry-pull-secret --for=pull -n openshift
```

別の方法として、`BuildConfig.spec.strategy.dockerStrategy.pullSecret` に同じ Secret を指定することもできます。

### 適用とビルド

```bash
oc apply -f setup/buildconfig-udi-workspace-image.yaml -n openshift
oc start-build udi-rhel9-java25-node24 -n openshift
```

ビルド完了後、Internal Registry 上の参照例は次のとおりです（クラスタのレジストリホスト名は環境に合わせて読み替えてください）。

```
image-registry.openshift-image-registry.svc:5000/openshift/udi-rhel9-java25-node24:latest
```

Dev Spaces の devfile のコンテナ `image` は、この URL（または外向きレジストリの同等パス）を指定します。ワークスペース Pod が **その Namespace のイメージを pull できること**（既定では認証付き pull が必要な場合があります）を管理者側で確認してください。

---

## Step 2. PaC 用トークン Secret の作成

Pipeline as Code (PaC) は Webhook の受信やコミットステータスの更新に Git プロバイダの API を使用します。
また、PaC はこのトークンをもとに Pipeline 内の Git 操作用の認証情報（`basic-auth` Secret）を **自動生成** します。

つまり、このトークンが **CI パイプライン全体の Git 認証を担う唯一の認証情報** です。

### 2-1. Personal Access Token (PAT) の発行

**GitHub の場合:**

1. [Settings > Developer settings > Personal access tokens](https://github.com/settings/tokens) を開く
2. **Generate new token (classic)** をクリック
3. 以下のスコープを付与:
   - `repo` — ソースリポジトリおよびマニフェストリポジトリへのアクセス
4. トークンを生成しコピー

> **注意:** トークンには、Backend・Frontend・Manifest の **3 つのリポジトリすべて** にアクセスできる権限が必要です。

### 2-2. Secret の作成

`setup/pac-token-secret.yaml` のプレースホルダーを編集します。

| プレースホルダー | 設定値 |
|------------------|--------|
| `<YOUR_GIT_PERSONAL_ACCESS_TOKEN>` | 発行した PAT |

```bash
oc apply -f setup/pac-token-secret.yaml
```

確認:

```bash
oc get secret dev-handson-pac-token -n dev-handson
```

### 認証の仕組み

```
                                PaC が自動生成
  ┌────────────────────┐      ┌──────────────────────────────────┐
  │ dev-handson-pac-token │ ──▶ │ {{ git_auth_secret }}             │
  │ (PAT を格納)          │      │ basic-auth Secret (一時的)        │
  └────────────────────┘      │ → Pipeline 内の git clone / push │
                              └──────────────────────────────────┘
```

PaC が PipelineRun を起動する際、PAT から一時的な `basic-auth` Secret を自動生成し、Pipeline の `basic-auth` Workspace にマウントします。これにより、ソースリポジトリの clone やマニフェストリポジトリへの push が可能になります。

---

## Step 3. Pipeline リソースのデプロイ（プラットフォームチーム）

プラットフォームチームが管理する Pipeline 定義をクラスタに適用します。

```bash
oc apply -f pipelines/dev-handson-backend-pipeline.yaml
oc apply -f pipelines/dev-handson-frontend-pipeline.yaml
```

確認:

```bash
oc get pipeline -n dev-handson
```

```
NAME                              AGE
dev-handson-backend-pipeline      10s
dev-handson-frontend-pipeline     10s
```

---

## Step 4. Pipeline as Code (PaC) Repository の登録

PaC が Webhook イベントを受けてパイプラインを起動できるよう、Repository リソースを作成します。

### 4-1. リポジトリ URL の編集

`setup/pac-repository-backend.yaml` と `setup/pac-repository-frontend.yaml` の `spec.url` を自分のリポジトリに変更します。

| ファイル | 変更箇所 | 設定値 |
|----------|----------|--------|
| `setup/pac-repository-backend.yaml` | `spec.url` | Backend リポジトリの HTTPS URL |
| `setup/pac-repository-frontend.yaml` | `spec.url` | Frontend リポジトリの HTTPS URL |

> **注意:** PaC Repository CR の `spec.url` は API アクセス用のため **HTTPS 形式** で指定します。

### 4-2. git_provider の設定確認

Repository CR には `git_provider.secret` ブロックが含まれています。Step 2 で作成した `dev-handson-pac-token` Secret を参照するようになっていることを確認してください。

```yaml
spec:
  url: "https://github.com/YOUR_ORG/your-app.git"
  git_provider:
    secret:
      name: dev-handson-pac-token
      key:
        token: token
```

> `pac-repository-backend.yaml` では `git_provider` ブロックがコメントアウトされている場合があります。コメントを解除してください。

### 4-3. 適用

```bash
oc apply -f setup/pac-repository-backend.yaml
oc apply -f setup/pac-repository-frontend.yaml
```

確認:

```bash
oc get repository -n dev-handson
```

---

## Step 5. ソースリポジトリに PipelineRun を配置（アプリチーム）

各ソースリポジトリの `.tekton/pipelinerun.yaml` が、PaC がパイプラインを起動するためのトリガー定義です。

### Backend リポジトリ

Backend リポジトリのルートに `.tekton/pipelinerun.yaml` を配置します。

```
backend-repo/
├── .tekton/
│   └── pipelinerun.yaml   ← PaC トリガー定義
├── pom.xml
└── src/
```

### Frontend リポジトリ

Frontend リポジトリも同様の構成です。

```
frontend-repo/
├── .tekton/
│   └── pipelinerun.yaml   ← PaC トリガー定義
├── package.json
└── app/
```

### アプリチームが変更する箇所

`.tekton/pipelinerun.yaml` 内の以下のパラメータを環境に合わせて編集します。

| パラメータ | 説明 |
|-----------|------|
| `manifest-repo-url` | Manifest リポジトリの HTTPS URL |

> `repo-url` と `revision` は PaC テンプレート変数（`{{ repo_url }}`, `{{ revision }}`）で自動設定されるため、変更不要です。

Pipeline のロジック自体はプラットフォームチーム側で管理されており、アプリチームは意識する必要がありません。

---

## Step 6. Manifest リポジトリの準備

Manifest リポジトリ（app-manifests）は **本リポジトリとは別の Git リポジトリ** です。
Kustomize 構成の Kubernetes マニフェストを格納し、CI パイプラインがイメージタグを自動更新します。

### リポジトリの構造

```
app-manifests/
├── base/
│   ├── kustomization.yaml
│   ├── backend/
│   │   ├── deployment.yaml
│   │   └── service.yaml
│   └── frontend/
│       ├── deployment.yaml
│       ├── service.yaml
│       └── route.yaml
└── envs/
    ├── dev/
    │   └── kustomization.yaml    ← dev 環境（CI がイメージタグを自動更新）
    └── prod/
        └── kustomization.yaml    ← prod 環境（レプリカ数・リソース増強）
```

### イメージタグの更新マーカー

`envs/dev/kustomization.yaml` には、パイプラインがイメージタグを書き換えるためのコメントマーカーが必要です。

```yaml
images:
  - name: dev-handson-backend
    newName: image-registry.openshift-image-registry.svc:5000/dev-handson-app/backend-app
    newTag: "initial" # backend-image-tag
  - name: dev-handson-frontend
    newName: image-registry.openshift-image-registry.svc:5000/dev-handson-app/frontend-app
    newTag: "initial" # frontend-image-tag
```

`# backend-image-tag` / `# frontend-image-tag` コメントが `sed` による置換の目印になります。これらのコメントを削除しないでください。

---

## Step 7. Argo CD のセットアップ

### 7-1. Argo CD インスタンスの作成（オプション）

Namespace スコープの Argo CD インスタンスを使用する場合は、以下を適用します。

```bash
oc apply -f argocd/instance.yaml
```

> クラスタスコープの OpenShift GitOps（`openshift-gitops` Namespace）を使用する場合はこのステップは不要です。
> その場合は `argocd/application.yaml` の `metadata.namespace` をクラスタの Argo CD Namespace に変更してください。

確認:

```bash
oc get argocd -n dev-handson-app
```

### 7-2. Argo CD Application の作成

`argocd/application.yaml` の Manifest リポジトリ URL を編集します。

| 変更箇所 | 設定値 |
|----------|--------|
| `spec.source.repoURL` | Manifest リポジトリの HTTPS URL |

```bash
oc apply -f argocd/application.yaml
```

確認:

```bash
oc get application -n dev-handson-app
```

> Argo CD は Manifest リポジトリの `envs/dev/` を監視し、変更を検知すると `dev-handson-app` Namespace にデプロイします。

---

## 動作確認

### PaC Webhook の設定

PaC Repository を登録した後、Git プロバイダ側で Webhook を設定する必要があります。

**GitHub の場合:**

PaC コントローラの Webhook URL を確認します。

```bash
echo "https://$(oc get route -n openshift-pipelines pipelines-as-code-controller -o jsonpath='{.spec.host}')"
```

Backend・Frontend の各リポジトリの **Settings > Webhooks** にこの URL を設定します。

- **Content type:** `application/json`
- **Events:** `Pushes` と `Pull requests`

> PaC が GitHub App として設定されている場合は、Webhook の手動設定は不要です。

### CI パイプラインの確認（PaC 経由）

Backend または Frontend リポジトリの `main` ブランチに push すると、パイプラインが自動起動します。

```bash
oc get pipelinerun -n dev-handson -w
```

パイプラインのフロー:

```
fetch-source → unit-test → build-image → fetch-manifest → update-manifest → sync-argocd
```

1. **fetch-source** — ソースコードを clone
2. **unit-test** — テスト実行（Maven / npm）
3. **build-image** — S2I でコンテナイメージをビルド（タグ = コミットハッシュ）
4. **fetch-manifest** — Manifest リポジトリを clone
5. **update-manifest** — `kustomization.yaml` のイメージタグを更新して push
6. **sync-argocd** — Argo CD に同期を指示し完了を待機

ログの確認:

```bash
tkn pipelinerun logs -n dev-handson --last
```

### CD デプロイの確認

パイプラインが Manifest リポジトリを更新すると、Argo CD が同期を実行します。

デプロイされた Pod を確認:

```bash
oc get pods -n dev-handson-app
```

アプリケーションの Route を確認:

```bash
oc get route -n dev-handson-app
```

---

## 変更が必要なパラメータ一覧

| 変更内容 | ファイル | 説明 |
|----------|----------|------|
| `<YOUR_GIT_PERSONAL_ACCESS_TOKEN>` | `setup/pac-token-secret.yaml` | PaC 用 Personal Access Token |
| Backend リポジトリ URL (HTTPS) | `setup/pac-repository-backend.yaml` | PaC が監視するリポジトリ |
| Frontend リポジトリ URL (HTTPS) | `setup/pac-repository-frontend.yaml` | PaC が監視するリポジトリ |
| Manifest リポジトリ URL (HTTPS) | `argocd/application.yaml` | Argo CD が監視するリポジトリ |
| Manifest リポジトリ URL (HTTPS) | 各ソースリポジトリの `.tekton/pipelinerun.yaml` | Pipeline が更新するリポジトリ |
| メイン教材リポジトリ URL (HTTPS) | `setup/buildconfig-udi-workspace-image.yaml` の `spec.source.git.uri` | Dev Spaces 用 UDI ビルドのソース取得（オプション） |
