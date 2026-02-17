# Kubernetes Practice Repository

ArgoCD + kind を使った Kubernetes 練習環境。本番 EKS を想定したノード分離構成を学ぶ。

## プロジェクト構成

```
kind-config.yaml                  # kind クラスタ定義（worker ノード × 3）
argocd/
  install/values.yaml             # ArgoCD Helm values
  applicationset-echoserver-group.yaml  # echoserver + httpbin グループ（ApplicationSet）
  application-nginx.yaml                # nginx（通常の Application）
apps/
  <name>/
    manifests/
      deployment.yaml             # Deployment（nodeSelector + tolerations）
      service.yaml                # Service
docs/
  design-comparison.md            # App of Apps vs ApplicationSet の比較
```

## 管理アプリ

| アプリ | イメージ | Namespace | ノード |
|--------|---------|-----------|--------|
| nginx | nginx:1.27 | practice | app=nginx |
| httpbin | kennethreitz/httpbin | practice | app=httpbin |
| echoserver | ealen/echo-server | practice | app=echoserver |

## アーキテクチャ

### GitOps（ApplicationSet パターン）

- `echoserver-group` ApplicationSet: Git Directory Generator で echoserver + httpbin の Application を自動生成
- `nginx` は通常の Application として個別管理（1アプリなので ApplicationSet 不要）
- 全アプリは `practice` Namespace に統一デプロイ
- syncPolicy: automated + prune + selfHeal + CreateNamespace=true

### ノード分離戦略

nodeSelector と Taints/Tolerations の組み合わせでアプリごとにノードを専有する。

- **nodeSelector**: Pod を特定ラベルのノードに配置
- **Taint** (`app=<name>:NoSchedule`): Toleration を持たない Pod の配置を拒否
- **Toleration**: 対応する Taint を許容し配置を許可

この構成は本番 EKS でのデプロイ範囲の最小化・リソース競合回避の練習。

## クラスタ操作

### 初回セットアップ / 再作成

```bash
kind create cluster --name argocd-practice --config kind-config.yaml

# ArgoCD インストール（Helm）
helm repo add argo https://argoproj.github.io/argo-helm
helm install argocd argo/argo-cd -n argocd --create-namespace -f argocd/install/values.yaml

# ApplicationSet + Application 適用
kubectl apply -f argocd/applicationset-echoserver-group.yaml
kubectl apply -f argocd/application-nginx.yaml
```

### kind の制約

- Taint やラベルの変更はクラスタ再作成が必要（`kind delete cluster` → `kind create cluster`）
- 再作成後は ArgoCD の再インストール + ApplicationSet / Application の再適用が必要

## Git リポジトリ

- リモート: `https://github.com/kentap/practice_kubernetes.git`
- メインブランチ: `main`
- コミットメッセージ: 英語、動詞始まり（例: `Add nodeSelector...`）

## アプリ追加手順

1. `apps/<name>/manifests/` に deployment.yaml, service.yaml を作成
2. 対象グループの ApplicationSet に `- path: apps/<name>/manifests` を追加
3. `kind-config.yaml` に worker ノードを追加（labels + taints）
4. クラスタを再作成して反映
