# App of Apps vs ApplicationSet — 設計方針比較

## 1. 現在の構成（App of Apps パターン）

```
argocd/app-of-apps.yaml          # ルート Application（apps/ を再帰監視、**/application.yaml を include）
apps/<name>/application.yaml     # 各アプリごとに手書きの Application 定義
apps/<name>/manifests/           # ワークロードマニフェスト（Deployment, Service など）
```

管理中のアプリ: nginx, httpbin, echoserver（計3アプリ）

各アプリの `application.yaml` は個別に手書きされており、repoURL / path / namespace / syncPolicy を個別に制御できる。現在は3アプリとも同一の syncPolicy（automated + prune + selfHeal + CreateNamespace=true）を使用している。

## 2. 仕組みの違い

### App of Apps

- ルート Application（`argocd/app-of-apps.yaml`）が `apps/` 配下の `**/application.yaml` を Git から読み取り、子 Application としてクラスタに適用する
- 各 Application は独立した Kubernetes リソースとして存在し、個別に設定可能
- アプリ追加時: `apps/<name>/` ディレクトリ作成 + `application.yaml` を手書き + `manifests/` にワークロード定義を配置

### ApplicationSet

- ApplicationSet Controller がテンプレートとジェネレータから Application リソースを自動生成する
- Git Directory Generator を使う場合、ディレクトリ構造から動的に Application を生成できる
- アプリ追加時: `apps/<name>/manifests/` ディレクトリにワークロード定義を配置するだけ（`application.yaml` 不要）

## 3. 業務利用時の判断ポイント

| 観点 | App of Apps | ApplicationSet |
|---|---|---|
| **アプリごとの個別設定** | 容易（各 application.yaml を自由に編集） | テンプレートのオーバーライドが必要で複雑化しがち |
| **アプリ追加の手間** | application.yaml の手書きが必要（コピペ＋修正） | ディレクトリ追加だけで自動生成 |
| **大量アプリ管理（50+）** | application.yaml の重複が管理負荷に | テンプレート1つで統一管理でき効率的 |
| **マルチクラスタ展開** | クラスタごとに application.yaml を書く | Cluster Generator で動的に展開可能 |
| **学習コスト・デバッグ** | Application リソースが直接見えるのでわかりやすい | 生成ロジックの理解が必要、トラブルシュート時に間接層がある |

## 4. 推奨の使い分け

### App of Apps が適するケース

- アプリ数が少ない（〜20程度）
- アプリごとに syncPolicy や project を細かく変えたい
- チームが ArgoCD に不慣れで、明示的なリソース定義のほうが理解しやすい

### ApplicationSet が適するケース

- アプリ数が多い（20+）
- 設定が均一（syncPolicy, project などが共通）
- マルチクラスタ展開がある
- アプリ追加の頻度が高く、手書きのコストを下げたい

### 併用パターン

App of Apps でカテゴリ（infra / backend / frontend）を分け、各カテゴリ内は ApplicationSet で管理する。これにより、カテゴリレベルでは個別制御を維持しつつ、カテゴリ内のアプリ追加は自動化できる。

## 5. 現在の構成からの移行パス

現構成（App of Apps）を ApplicationSet に移行する場合の変更点:

1. `apps/<name>/application.yaml` を全削除（nginx, httpbin, echoserver）
2. `argocd/app-of-apps.yaml` を `argocd/applicationset.yaml`（ApplicationSet リソース）に置き換え
3. Git Directory Generator で `apps/*/manifests` を自動検出するテンプレートを記述
4. `apps/<name>/manifests/` 配下のワークロードマニフェストはそのまま利用可能

### ApplicationSet の YAML 例（現構成の移行先イメージ）

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: apps
  namespace: argocd
spec:
  goTemplate: true
  goTemplateOptions: ["missingkey=error"]
  generators:
    - git:
        repoURL: https://github.com/kentap/practice_kubernetes.git
        revision: main
        directories:
          - path: apps/*/manifests
  template:
    metadata:
      name: "{{ index .path.segments 1 }}"
    spec:
      project: default
      source:
        repoURL: https://github.com/kentap/practice_kubernetes.git
        targetRevision: main
        path: "{{ .path.path }}"
      destination:
        server: https://kubernetes.default.svc
        namespace: "{{ index .path.segments 1 }}"
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
```

この ApplicationSet は、`apps/*/manifests` にマッチするディレクトリごとに Application を自動生成する。ディレクトリ名（`nginx`, `httpbin`, `echoserver`）がアプリ名および namespace として使われる。
