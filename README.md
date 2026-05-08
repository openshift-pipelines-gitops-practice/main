# OpenShift Pipelines & GitOps ハンズオン

OpenShift Pipelines (Tekton) と OpenShift GitOps (Argo CD) を組み合わせた CI/CD パイプラインのハンズオン教材です。Pipeline as Code (PaC) による自動トリガー、S2I によるコンテナビルド、Kustomize + Argo CD による GitOps デプロイまでの一連のフローを体験できます。

## アーキテクチャ

```
  開発者が push
       │
       ▼
 ┌──────────────┐   Webhook    ┌──────────────────────────────────────────────┐
 │  Source Repo  │ ──────────▶ │  Pipeline as Code (PaC)                      │
 │  (Backend /   │             │    PipelineRun (.tekton/) をもとに起動        │
 │   Frontend)   │             └──────────┬───────────────────────────────────┘
 └──────────────┘                         │
                                          ▼
                              ┌──────────────────────┐
                              │  Tekton Pipeline      │
                              │  1. git-clone         │
                              │  2. unit-test         │
                              │  3. s2i build & push  │
                              │  4. manifest 更新     │
                              │  5. ArgoCD Sync       │
                              └──────────┬────────────┘
                                         │ イメージタグを
                                         │ git push で更新
                                         ▼
                              ┌──────────────────────┐       Auto Sync
                              │  Manifest Repo       │ ───────────────▶  Argo CD
                              │  (Kustomize)         │                     │
                              └──────────────────────┘                     ▼
                                                                   ┌────────────┐
                                                                   │ OpenShift  │
                                                                   │ Namespace  │
                                                                   └────────────┘
```

## リポジトリ構成

このハンズオンは **4 つの Git リポジトリ** で構成されています。

| リポジトリ | 役割 | 管理者 |
|-----------|------|--------|
| **本リポジトリ** (メイン) | Pipeline 定義・カスタム Task・PaC 設定・Argo CD リソースなどのプラットフォーム資材 | プラットフォームチーム |
| **Backend リポジトリ** | Java (Spring Boot) アプリケーション + `.tekton/pipelinerun.yaml` | アプリチーム |
| **Frontend リポジトリ** | Nuxt.js アプリケーション + `.tekton/pipelinerun.yaml` | アプリチーム |
| **Manifest リポジトリ** (app-manifests) | Kustomize マニフェスト（Backend: Deployment / Service、Frontend: Deployment / Service / Route）。イメージタグは CI が自動更新 | アプリチーム |

### 本リポジトリの構造

```
.
├── README.md
├── docs/
│   ├── setup-guide.md           … セットアップ手順書
│   └── developer-handson.md    … 開発者向けハンズオン
├── pipelines/
│   ├── task-maven.yaml                     … Backend 用 Maven Task（dev-handson に適用）
│   ├── task-s2i-java.yaml                  … Backend 用 S2I Java Task（dev-handson に適用）
│   ├── dev-handson-backend-pipeline.yaml … Backend 用 Pipeline
│   └── dev-handson-frontend-pipeline.yaml … Frontend 用 Pipeline
├── setup/
│   ├── pac-token-secret.yaml              … PaC 用 PAT Secret テンプレート
│   ├── pac-repository-backend.yaml        … PaC Repository (Backend)
│   └── pac-repository-frontend.yaml       … PaC Repository (Frontend)
└── argocd/
    ├── application.yaml        … Argo CD Application
    └── instance.yaml           … Argo CD インスタンス定義 (オプション)
```

### Manifest リポジトリの構造

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

### ソースリポジトリの構造（Backend / Frontend 共通）

```
source-repo/
├── .tekton/
│   └── pipelinerun.yaml   ← PaC が参照する PipelineRun 定義
├── (アプリケーションソースコード)
└── ...
```

**Backend** の `.tekton/pipelinerun.yaml` では、`maven-settings` Workspace に `emptyDir: {}` を指定するなど、クラスタ上の Pipeline 定義と Workspace を一致させる必要があります（リポジトリ内のサンプルを参照）。

## Namespace 構成

| Namespace | 用途 |
|-----------|------|
| `dev-handson` | Tekton Pipeline / PaC Repository / Secret、`maven`・`s2i-java` Task |
| `dev-handson-app` | Argo CD インスタンス・Application / デプロイされるアプリケーション |

## パイプラインのフロー

Backend・Frontend とも同じ構成のパイプラインが実行されます。

```
fetch-source → unit-test → build-image → fetch-manifest → update-manifest → sync-argocd
```

| ステップ | 内容 | 使用 Task | Namespace |
|---------|------|-----------|-----------|
| **fetch-source** | ソースコードを clone | `git-clone` | `openshift-pipelines` |
| **unit-test** | テスト実行 | `maven`（Backend） / `taskSpec` 内の npm（Frontend） | Backend: `dev-handson` / Frontend: （インライン） |
| **build-image** | S2I でコンテナイメージをビルド（タグ = コミットハッシュ） | `s2i-java` / `s2i-nodejs` | Backend: `dev-handson` / Frontend: `openshift-pipelines` |
| **fetch-manifest** | Manifest リポジトリを clone | `git-clone` | `openshift-pipelines` |
| **update-manifest** | `kustomization.yaml` のイメージタグを書き換えて push | `git-cli` | `openshift-pipelines` |
| **sync-argocd** | Argo CD に同期を指示 | `argocd-task-sync-and-wait` | `openshift-pipelines` |

**Backend** では、`maven` と `s2i-java` を Red Hat 提供の Task 定義どおり **`dev-handson` Namespace に事前適用**します（`pipelines/task-maven.yaml`、`pipelines/task-s2i-java.yaml`）。ビルダーイメージや Maven プロジェクトの相対パスは Pipeline パラメータ（`s2i-builder-image`、`maven-subdirectory` など）で調整できます。

**Frontend** のユニットテストは UBI Node.js イメージ上で `npm install` → `nuxt prepare` → `npm run test:unit` を実行するインライン `taskSpec` です。`s2i-nodejs` はクラスタ標準の `openshift-pipelines` Namespace を参照します。

## 認証方式

| 用途 | 認証方式 | Secret 名 |
|------|----------|-----------|
| PaC (Webhook / コミットステータス更新) | Personal Access Token | `dev-handson-pac-token` |
| Pipeline 内の Git clone / push | PaC 自動生成トークン (`{{ git_auth_secret }}`) | 自動 |

## プラットフォームチーム vs アプリチームの責任範囲

```
┌─────────────────────────────────────────────────────────────────────┐
│  プラットフォームチーム                                              │
│  ・Pipeline 定義の作成・メンテナンス (pipelines/*.yaml)               │
│  ・Backend 用 Task（maven / s2i-java）の適用 (task-maven / task-s2i-java) │
│  ・PaC Repository / Secret の管理 (setup/)                          │
│  ・Argo CD Application の管理 (argocd/)                             │
│  ・Namespace / RBAC の管理                                          │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│  アプリチーム                                                        │
│  ・ソースコードの開発                                                │
│  ・.tekton/pipelinerun.yaml の配置（テンプレートから URL を変更）     │
│  ・Manifest リポジトリの初期構築                                     │
└─────────────────────────────────────────────────────────────────────┘
```

アプリチームは Pipeline のロジックを意識する必要はなく、`.tekton/pipelinerun.yaml` 内のリポジトリ URL パラメータを設定するだけで CI/CD が有効になります。

## 前提条件

- OpenShift Container Platform へのアクセス（`oc` CLI ログイン済み）
- **OpenShift Pipelines Operator** がインストール済み
- **OpenShift GitOps Operator** がインストール済み
- 本教材（メイン）リポジトリに加え、**Backend・Frontend・Manifest 用に別々の Git リポジトリ**（計 3 つ）を用意できること

## セットアップ

詳細な手順は **[セットアップガイド](docs/setup-guide.md)** を参照してください。ガイドの Pipeline 適用より **先に**、`pipelines/task-maven.yaml` と `pipelines/task-s2i-java.yaml` を `dev-handson` に適用してください（下記概要 Step 3）。

概要:

1. Namespace の作成（`dev-handson`, `dev-handson-app`）
2. PaC 用 PAT Secret の作成
3. **Backend 用 Task と Pipeline の適用** — `task-maven.yaml`・`task-s2i-java.yaml` を先に `oc apply` し、続けて両 Pipeline を適用
4. PaC Repository の登録
5. ソースリポジトリに `.tekton/pipelinerun.yaml` を配置（Backend は `maven-settings` Workspace を含める）
6. Manifest リポジトリの準備
7. Argo CD Application の作成

## 変更が必要なパラメータ

セットアップ時に、以下のプレースホルダーを自分の環境に合わせて変更してください。

| プレースホルダー | ファイル | 説明 |
|------------------|----------|------|
| `<YOUR_GIT_PERSONAL_ACCESS_TOKEN>` | `setup/pac-token-secret.yaml` | PaC 用 Personal Access Token |
| リポジトリ URL (HTTPS) | `setup/pac-repository-backend.yaml` | Backend リポジトリの URL |
| リポジトリ URL (HTTPS) | `setup/pac-repository-frontend.yaml` | Frontend リポジトリの URL |
| Manifest リポジトリ URL (HTTPS) | `argocd/application.yaml` | Argo CD が監視するリポジトリ |
| Manifest リポジトリ URL (HTTPS) | 各 `.tekton/pipelinerun.yaml` | Pipeline が更新するリポジトリ |

## 関連ドキュメント

- [セットアップガイド](docs/setup-guide.md) — 環境構築の詳細手順
- [開発者向けハンズオン](docs/developer-handson.md) — アプリ修正から CI/CD デプロイまでの体験
