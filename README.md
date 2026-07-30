# cicddemo-Petclinic

OpenShift 上で Spring PetClinic アプリケーションの CI/CD パイプラインを構築するデモです。
Tekton Pipelines でビルド・テスト・デプロイを自動化し、ArgoCD (OpenShift GitOps) で Kubernetes マニフェストを GitOps 管理します。

## 構成概要

```
cicddemo-Petclinic/
├── tekton/                          # Tekton Pipeline 定義
│   ├── pipeline.yaml                # 11ステップの CI/CD パイプライン
│   ├── pipelinerun.yaml             # PipelineRun (パイプライン実行)
│   └── dockerfile-configmap.yaml    # Dockerfile を格納する ConfigMap
├── manifests/base/                  # Kubernetes マニフェスト (ArgoCD が同期)
│   ├── kustomization.yaml           # Kustomize 構成
│   ├── namespace.yaml               # petclinic Namespace
│   ├── deployment.yaml              # Deployment (UBI9 + OpenJDK 17)
│   ├── service.yaml                 # Service (port 8080)
│   └── route.yaml                   # OpenShift Route (TLS edge)
├── argocd/
│   └── application.yaml             # ArgoCD Application (自動同期 + セルフヒール)
└── slides/                          # デモ用プレゼンテーション資料
```

## パイプライン (11ステップ)

Tekton Pipeline `petclinic-build` は以下の順序で実行されます:

```
 1. fetch-source     ソースコード取得 (git-clone)
        │
 2. unit-test        単体テスト (Maven: test)
        │
 3. code-analysis    コード品質チェック (Checkstyle, Compiler Warnings, Dependency Tree)
        │
 4. build-app        アプリケーションビルド (Maven: package)
        │
 5. copy-dockerfile  Dockerfile を ConfigMap からワークスペースにコピー
        │
 6. build-image      コンテナイメージビルド (Buildah)
        │
 7. image-scan       イメージ脆弱性スキャン (Trivy: HIGH/CRITICAL)
        │
 8. deploy-wait      デプロイ & ロールアウト完了待ち (oc rollout)
        │
 9. smoke-test       スモークテスト (HTTP 200 チェック、最大10回リトライ)
        │
10. api-test         API テスト (/, /owners/find, /vets.html, /owners/new)
        │
11. load-test        負荷テスト (50リクエスト, 5並列, エラー率20%以下で合格)
```

## 前提条件

- OpenShift 4.x クラスタ
- OpenShift Pipelines Operator (Tekton) インストール済み
- OpenShift GitOps Operator (ArgoCD) インストール済み
- OpenShift 内部イメージレジストリが有効

## セットアップ

### 1. Tekton リソースの作成

```sh
oc apply -f tekton/dockerfile-configmap.yaml
oc apply -f tekton/pipeline.yaml
```

### 2. ArgoCD Application の作成

```sh
oc apply -f argocd/application.yaml
```

ArgoCD が `manifests/base/` を自動同期し、Namespace・Deployment・Service・Route を作成します。

### 3. パイプラインの実行

```sh
oc create -f tekton/pipelinerun.yaml
```

`generateName` を使用しているため `oc create` で実行します（`oc apply` ではなく）。

## GitOps フロー

```
[GitHub リポジトリ]
       │
       ├── tekton/          ──→ Tekton Pipeline がビルド & テスト & デプロイ
       │                          │
       │                          ▼
       │                    OpenShift 内部レジストリにイメージ Push
       │
       └── manifests/base/  ──→ ArgoCD が監視 & 自動同期
                                  │
                                  ▼
                             petclinic Namespace にマニフェスト適用
```

- **Tekton**: ソースコードのビルド、テスト、イメージ作成、デプロイを担当
- **ArgoCD**: `manifests/base/` の変更を検知し、クラスタへ自動反映（prune + selfHeal 有効）

## アプリケーション構成

| 項目 | 値 |
|---|---|
| アプリケーション | [Spring PetClinic](https://github.com/spring-projects/spring-petclinic) |
| ベースイメージ | `registry.access.redhat.com/ubi9/openjdk-17:latest` |
| Namespace | `petclinic` |
| レプリカ数 | 1 |
| リソース | CPU: 250m〜500m / Memory: 512Mi〜1Gi |
| ヘルスチェック | Readiness: 30s後開始 / Liveness: 60s後開始 |
| Route | TLS edge termination (HTTP → HTTPS リダイレクト) |
