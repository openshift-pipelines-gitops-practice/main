# 開発者向けハンズオン：OpenShift Pipelines & GitOps

このハンズオンでは、開発者の視点から CI/CD パイプラインを体験します。
アプリケーションのコードを修正し、push するだけで自動的にビルド・テスト・デプロイが行われる流れを実際に確認します。

> **前提:** [セットアップガイド](setup-guide.md) の手順が完了し、Pipeline / PaC / Argo CD が稼働している環境で実施します。

---

## Step 1. 対象アプリケーションの構成理解

### 1-1. アプリケーションの概要

今回扱うアプリケーションは **FizzBuzz** です。
数値を入力すると、FizzBuzz のルールに基づいた結果を返します。

| ルール | 結果 |
|--------|------|
| 3 の倍数 | `Fizz` |
| 5 の倍数 | `Buzz` |
| 3 と 5 の両方の倍数 | `Fizz Buzz` |
| それ以外 | 入力した数値をそのまま返す |

このアプリケーションは **Backend（API）** と **Frontend（UI）** の 2 つのコンポーネントで構成されています。

### 1-2. 技術スタック

| コンポーネント | 技術 | 説明 |
|---------------|------|------|
| **Backend** | Java 17 / Spring Boot 2.7 / Maven | REST API (`GET /fizzbuzz/{number}`) を提供 |
| **Frontend** | Nuxt 4 / Vue 3 / TypeScript | ブラウザ UI + サーバーサイドで Backend にプロキシ |

### 1-3. 通信フロー

```
ブラウザ
  │
  │ HTTPS
  ▼
Frontend Route (port 3000)
  │
  │ Nuxt Server API (/api/fizzbuzz/:number)
  │ → サーバーサイドで Backend に HTTP リクエスト
  ▼
Backend Service (port 8080)
  │
  │ GET /fizzbuzz/{number}
  ▼
FizzBuzz ロジック → レスポンス
```

Frontend はブラウザからのリクエストを直接 Backend に転送するのではなく、**Nuxt のサーバーサイド API** (`server/api/fizzbuzz/[number].ts`) が Backend Service に対してリクエストを行います。これにより、Backend の URL はクラスタ内部の Service 名（`http://dev-handson-backend:8080`）で解決されます。

### 1-4. リポジトリ構成

このハンズオンで使用するリポジトリは 4 つです。

```
┌─────────────────────────────────┐
│  メインリポジトリ（本リポジトリ）  │  Pipeline 定義 / PaC 設定 / ArgoCD
│  → プラットフォームチームが管理    │
└─────────────────────────────────┘

┌──────────────────┐  ┌──────────────────┐
│  Backend リポジトリ │  │  Frontend リポジトリ │  ソースコード + .tekton/pipelinerun.yaml
│  → アプリチームが管理│  │  → アプリチームが管理 │
└────────┬─────────┘  └────────┬─────────┘
         │  push で CI 起動             │  push で CI 起動
         ▼                             ▼
┌─────────────────────────────────┐
│  Manifest リポジトリ (app-manifests)│  Kustomize マニフェスト
│  → CI パイプラインが自動更新        │
└─────────────────────────────────┘
```

### 1-5. Backend リポジトリの構成

```
backend-repo/
├── .tekton/
│   └── pipelinerun.yaml           ← PaC トリガー定義
├── pom.xml                        ← Maven プロジェクト定義
├── src/main/java/.../
│   ├── FizzbuzzApiApplication.java  ← Spring Boot メインクラス
│   ├── adapter/
│   │   ├── FizzBuzzController.java  ← REST コントローラ
│   │   └── AppConfig.java           ← 設定（min/max の範囲）
│   └── application/
│       ├── FizzBuzz.java            ← ★ FizzBuzz のコアロジック
│       └── FizzBuzzPolicy.java      ← ポリシーインターフェース
├── src/test/java/.../
│   ├── application/
│   │   └── FizzBuzzTest.java        ← ★ ユニットテスト
│   └── FizzbuzzApiApplicationIntegrationTest.java
└── src/main/resources/
    └── application.properties       ← min-number=1, max-number=100
```

FizzBuzz のコアロジック（`FizzBuzz.java`）:

```java
public String fizzbuzz(int i) {
    if (divisibleBy3(i) && divisibleBy5(i)) {
        return "Fizz Buzz";
    } else if (divisibleBy3(i)) {
        return "Fizz";
    } else if (divisibleBy5(i)) {
        return "Buzz";
    } else {
        return String.valueOf(i);
    }
}
```

### 1-6. Frontend リポジトリの構成

```
frontend-repo/
├── .tekton/
│   └── pipelinerun.yaml           ← PaC トリガー定義
├── package.json                   ← Nuxt 4 / Vue 3 / Vitest
├── nuxt.config.ts                 ← API_BASE_URL の設定
├── app/
│   ├── app.vue                    ← ★ ルートコンポーネント
│   ├── components/
│   │   ├── FbFizzBuzz.vue           ← メインコンポーネント（API 呼び出し）
│   │   ├── FbEntry.vue              ← 入力フォーム（バリデーション付き）
│   │   ├── FbLogs.vue               ← 結果表示リスト
│   │   └── __tests__/
│   │       └── FbFizzBuzz.spec.ts   ← コンポーネントテスト
│   └── utils/
│       ├── validations.ts           ← バリデーションロジック
│       └── __tests__/               ← ユーティリティのテスト
└── server/api/fizzbuzz/
    └── [number].ts                ← Backend へのプロキシ API
```

### 1-7. Manifest リポジトリの構成

```
app-manifests/
├── base/
│   ├── kustomization.yaml
│   ├── backend/
│   │   ├── deployment.yaml    ← image: dev-handson-backend:latest
│   │   └── service.yaml       ← port: 8080（クラスタ内からのみ公開）
│   └── frontend/
│       ├── deployment.yaml    ← image: dev-handson-frontend:latest
│       ├── service.yaml       ← port: 3000
│       └── route.yaml         ← TLS edge termination
└── envs/
    ├── dev/
    │   └── kustomization.yaml   ← dev 環境（CI がイメージタグを自動更新）
    └── prod/
        └── kustomization.yaml   ← prod 環境（レプリカ数・リソース増強）
```

`envs/dev/kustomization.yaml` の `images` セクションで、Kustomize が base の Deployment のイメージを上書きします。CI パイプラインはここの `newTag` をコミットハッシュに書き換えて push します。

```yaml
images:
  - name: dev-handson-backend
    newName: image-registry.openshift-image-registry.svc:5000/dev-handson-app/backend-app
    newTag: "abc1234" # backend-image-tag
  - name: dev-handson-frontend
    newName: image-registry.openshift-image-registry.svc:5000/dev-handson-app/frontend-app
    newTag: "def5678" # frontend-image-tag
```

> `# backend-image-tag` / `# frontend-image-tag` のコメントは、パイプラインが `sed` で置換する際の目印です。

---

## Step 2. パイプラインの構成把握

### 2-1. Pipeline as Code (PaC) の仕組み

開発者がソースリポジトリに push すると、パイプラインが **自動的に** 起動します。
この自動化は **Pipeline as Code (PaC)** によって実現されています。

各ソースリポジトリの `.tekton/pipelinerun.yaml` に、以下のようなアノテーションが定義されています。

```yaml
metadata:
  annotations:
    pipelinesascode.tekton.dev/on-event: "[push]"
    pipelinesascode.tekton.dev/on-target-branch: "[main]"
    pipelinesascode.tekton.dev/max-keep-runs: "3"
```

| アノテーション | 意味 |
|---------------|------|
| `on-event: "[push]"` | push イベントで起動 |
| `on-target-branch: "[main]"` | `main` ブランチへの push が対象 |
| `max-keep-runs: "3"` | PipelineRun の履歴を最大 3 件保持 |

PaC は Git プロバイダの Webhook を受信し、リポジトリ内の `.tekton/pipelinerun.yaml` を読み取って PipelineRun を生成・実行します。

### 2-2. パイプラインのフロー

Backend / Frontend ともに同じ構成のパイプラインが順次実行されます。

```
                        Backend の場合
  ┌─────────────┐    ┌───────────┐    ┌─────────────┐
  │ fetch-source │ ─▶ │ unit-test │ ─▶ │ build-image │
  │ (git-clone)  │    │ (maven)   │    │ (s2i-java)  │
  └─────────────┘    └───────────┘    └──────┬──────┘
                                              │
       ┌──────────────┐    ┌─────────────────┐│   ┌──────────────┐
       │ sync-argocd   │ ◀─ │ update-manifest │◀──┘│ fetch-manifest│
       │ (argocd-task-  │    │ (git-cli)       │◀───│ (git-clone)   │
       │  sync-and-wait)│    └─────────────────┘    └──────────────┘
       └──────────────┘
```

### 2-3. 各ステップの解説

| # | ステップ | 使用 Task | 内容 |
|---|---------|-----------|------|
| 1 | **fetch-source** | `git-clone` | ソースリポジトリを clone |
| 2 | **unit-test** | `maven` (Backend) / インライン (Frontend) | テストを実行。失敗するとパイプライン全体が停止 |
| 3 | **build-image** | `s2i-java` / `s2i-nodejs` | S2I でコンテナイメージをビルドし、OpenShift 内部レジストリに push。**タグにはコミットハッシュ**を使用 |
| 4 | **fetch-manifest** | `git-clone` | Manifest リポジトリの `main` ブランチを clone |
| 5 | **update-manifest** | `git-cli` | `kustomization.yaml` のイメージタグをコミットハッシュに書き換えて push |
| 6 | **sync-argocd** | `argocd-task-sync-and-wait` | Argo CD に同期を指示し、デプロイ完了まで待機 |

> 使用する Tekton Task はすべて OpenShift Pipelines がプリセットで提供するものです（`openshift-pipelines` Namespace）。
> Pipeline の定義・メンテナンスはプラットフォームチームが担当しており、開発者が変更する必要はありません。

### 2-4. イメージタグの自動更新の仕組み

パイプラインの **update-manifest** ステップでは、以下のような `sed` コマンドが実行されます。

```bash
sed -i 's|newTag: ".*" # backend-image-tag|newTag: "a1b2c3d" # backend-image-tag|' envs/dev/kustomization.yaml
```

これにより、Manifest リポジトリの `kustomization.yaml` が更新され、新しいコミットが push されます。

### 2-5. Argo CD による自動デプロイ

Manifest リポジトリが更新されると、パイプラインの最後のステップで Argo CD に同期が指示されます。Argo CD は `envs/dev/` の変更を検知し、`dev-handson-app` Namespace のリソースを更新します。

```
Pipeline が push          Argo CD が同期        OpenShift に反映
 ┌──────────────┐        ┌──────────┐         ┌───────────────┐
 │ Manifest Repo │ ─────▶ │ Argo CD  │ ──────▶ │ dev-handson-app│
 │ (newTag 更新) │        │ (Sync)   │         │ Namespace      │
 └──────────────┘        └──────────┘         └───────────────┘
```

### 2-6. パイプラインの確認コマンド

現在の状態を確認するためのコマンドを覚えておきましょう。

```bash
# PipelineRun の一覧
oc get pipelinerun -n dev-handson

# 最新の PipelineRun のログをリアルタイム表示
tkn pipelinerun logs -n dev-handson --last -f

# PipelineRun をリアルタイム監視
oc get pipelinerun -n dev-handson -w

# デプロイされた Pod の確認
oc get pods -n dev-handson-app

# Route の確認
oc get route -n dev-handson-app
```

---

## Step 3. コードを修正してパイプラインを動かす

ここからは実際にコードを修正し、パイプラインの動作を体験します。

### 3-1. 事前確認：現在のアプリケーションの状態

まず、現在デプロイされているアプリケーションを確認します。

```bash
# Frontend の Route URL を取得
oc get route dev-handson-frontend -n dev-handson-app -o jsonpath='{.spec.host}'
```

ブラウザでこの URL にアクセスし、いくつかの数値を入力して FizzBuzz の動作を確認してください。

| 入力 | 期待される結果 |
|------|--------------|
| 1 | `1` |
| 3 | `Fizz` |
| 5 | `Buzz` |
| 15 | `Fizz Buzz` |
| 7 | `7` |

Backend API を直接呼び出すこともできます。

```bash
BACKEND_URL=$(oc get route dev-handson-backend -n dev-handson-app -o jsonpath='{.spec.host}')
curl -s "https://${BACKEND_URL}/fizzbuzz/3"
# => Fizz
```

---

### 3-2. Backend の修正：FizzBuzz に新しいルールを追加する

FizzBuzz のロジックを拡張して、**7 の倍数のときに `"Whiz"` を返す** ルールを追加します。

#### 修正対象ファイル

| ファイル | 変更内容 |
|----------|----------|
| `src/main/java/.../application/FizzBuzz.java` | 7 の倍数の判定を追加 |
| `src/test/java/.../application/FizzBuzzTest.java` | 新ルールのテストケースを追加 |

#### (1) FizzBuzz.java を修正

`FizzBuzz.java` の `fizzbuzz` メソッドに 7 の倍数の判定を追加します。

**変更前:**

```java
public String fizzbuzz(int i) {
    if (i < policy.getMinNumber() || i > policy.getMaxNumber()) {
        throw new IllegalArgumentException(
            String.format("fizzbuzz only accepts integer %d ... %d",
                          policy.getMinNumber(), policy.getMaxNumber()));
    }

    if (divisibleBy3(i) && divisibleBy5(i)) {
        return "Fizz Buzz";
    } else if (divisibleBy3(i)) {
        return "Fizz";
    } else if (divisibleBy5(i)) {
        return "Buzz";
    } else {
        return String.valueOf(i);
    }
}
```

**変更後:**

```java
public String fizzbuzz(int i) {
    if (i < policy.getMinNumber() || i > policy.getMaxNumber()) {
        throw new IllegalArgumentException(
            String.format("fizzbuzz only accepts integer %d ... %d",
                          policy.getMinNumber(), policy.getMaxNumber()));
    }

    if (divisibleBy3(i) && divisibleBy5(i)) {
        return "Fizz Buzz";
    } else if (divisibleBy3(i)) {
        return "Fizz";
    } else if (divisibleBy5(i)) {
        return "Buzz";
    } else if (divisibleBy7(i)) {
        return "Whiz";
    } else {
        return String.valueOf(i);
    }
}

private boolean divisibleBy7(int i) {
    return i % 7 == 0;
}
```

#### (2) FizzBuzzTest.java にテストを追加

新しいルールのテストケースを追加します。

```java
@Test void fizzbuzzShouldReturnWhiz() {
    assertThat(fizzbuzz.fizzbuzz(7)).isEqualTo("Whiz");
    assertThat(fizzbuzz.fizzbuzz(7 * 2)).isEqualTo("Whiz");
    assertThat(fizzbuzz.fizzbuzz(7 * 4)).isEqualTo("Whiz");
}
```

> **注意:** `7 * 3 = 21` は 3 の倍数でもあるため `"Fizz"` が返ります。テストケースでは 3 や 5 の倍数と重ならない値を選んでください。

#### (3) ローカルでテストを実行

```bash
cd backend-repo/
./mvnw test
```

```
[INFO] Tests run: XX, Failures: 0, Errors: 0, Skipped: 0
[INFO] BUILD SUCCESS
```

テストが通ることを確認します。

#### (4) commit して push

```bash
git add -A
git commit -m "feat: 7の倍数でWhizを返すルールを追加"
git push origin main
```

#### (5) パイプラインの動作を観察する

push 後、PaC が Webhook を受信してパイプラインが起動します。

**ターミナル 1: PipelineRun の監視**

```bash
oc get pipelinerun -n dev-handson -w
```

```
NAME                             SUCCEEDED   REASON    STARTTIME   COMPLETIONTIME
dev-handson-backend-push-xxxxx   Unknown     Running   5s
```

**ターミナル 2: ログのリアルタイム確認**

```bash
tkn pipelinerun logs -n dev-handson --last -f
```

各ステップが順番に実行される様子を確認してください。

| 確認ポイント | 確認方法 |
|------------|---------|
| Pipeline が起動したか | `oc get pipelinerun -n dev-handson` |
| unit-test が通ったか | `tkn pipelinerun logs` で Maven テスト結果を確認 |
| イメージがビルドされたか | ログで `s2i-java` の完了を確認 |
| Manifest が更新されたか | Manifest リポジトリの Git ログを確認 |
| Argo CD が同期したか | ログで `sync-argocd` の完了を確認 |

#### (6) デプロイ結果を確認する

パイプラインが完了したら、Pod が更新されていることを確認します。

```bash
# Pod のローリングアップデートを確認
oc get pods -n dev-handson-app

# Backend API で新しいルールをテスト
BACKEND_URL=$(oc get route dev-handson-backend -n dev-handson-app -o jsonpath='{.spec.host}')
curl -s "https://${BACKEND_URL}/fizzbuzz/7"
# => Whiz

curl -s "https://${BACKEND_URL}/fizzbuzz/14"
# => Whiz

curl -s "https://${BACKEND_URL}/fizzbuzz/3"
# => Fizz（既存ルールは変わらない）
```

ブラウザで Frontend にアクセスし、`7` を入力して `Whiz` が表示されることも確認してください。

---

### 3-3. Frontend の修正：UI のタイトルを変更する

次は Frontend のコードを修正して、同様にパイプラインが動くことを確認します。

#### 修正対象ファイル

| ファイル | 変更内容 |
|----------|----------|
| `app/app.vue` | ヘッダーのタイトルテキストを変更 |

#### (1) app.vue を修正

**変更前:**

```vue
<template>
  <header>
    <h1>Fizz Buzz</h1>
  </header>
  <main>
    <FbFizzBuzz />
  </main>
</template>
```

**変更後:**

```vue
<template>
  <header>
    <h1>Fizz Buzz Game</h1>
  </header>
  <main>
    <FbFizzBuzz />
  </main>
</template>
```

#### (2) ローカルでテストを実行

```bash
cd frontend-repo/
npm run test:unit
```

テストが通ることを確認します。

> 今回の変更はテストで検証している箇所に影響しないため、既存テストはそのまま通ります。

#### (3) commit して push

```bash
git add -A
git commit -m "feat: タイトルを Fizz Buzz Game に変更"
git push origin main
```

#### (4) パイプラインの動作を観察する

Backend と同様に、PipelineRun の起動とログを確認します。

```bash
oc get pipelinerun -n dev-handson -w
```

```bash
tkn pipelinerun logs -n dev-handson --last -f
```

今回は **`dev-handson-frontend-pipeline`** が実行されることに注目してください。
Backend と Frontend で **別々の Pipeline** が使われていますが、フローの構成は同じです。

| Backend Pipeline | Frontend Pipeline |
|-----------------|-------------------|
| `maven` でテスト | インライン Task で `npm run test:unit` |
| `s2i-java` でビルド | `s2i-nodejs` (Node.js 22) でビルド |

#### (5) デプロイ結果を確認する

```bash
oc get pods -n dev-handson-app
```

ブラウザで Frontend にアクセスし、ヘッダーが **「Fizz Buzz Game」** に変わっていることを確認してください。

---

## まとめ

このハンズオンを通じて、以下の流れを体験しました。

```
コード修正 → git push → PaC が検知 → Pipeline 起動
  → テスト → イメージビルド → Manifest 更新 → ArgoCD Sync → デプロイ完了
```

開発者として意識するポイント:

- **普段のコード開発と push だけで CI/CD が動く** — Pipeline の定義はプラットフォームチームが管理しており、開発者が変更する必要はありません
- **テストが通らないとデプロイされない** — unit-test ステップで失敗するとパイプライン全体が停止します
- **イメージタグはコミットハッシュ** — どのコミットがデプロイされているかをいつでも追跡できます
- **GitOps による宣言的デプロイ** — Manifest リポジトリの状態がクラスタの状態と一致するよう Argo CD が自動的に同期します
