# ReplicaSet と Deployment

- 前: [01-basics.md](./01-basics.md) — Kubernetes と Pod の基礎
- 次: [03-service-configmap-secret.md](./03-service-configmap-secret.md) — Service / ConfigMap / Secret
- 関連: [06-rolling-update-auto-healing.md](./06-rolling-update-auto-healing.md) — ローリングアップデートと自己修復の検証

本ページでは、Pod の次の段階にあたる ReplicaSet と Deployment について整理する。Kubernetes は「下位の仕組みでは解決できない課題を、上位の仕組みが引き受ける」という積み重ねで構成されており、Pod・ReplicaSet・Deployment はその代表例である。

``` text
Pod だけ        … 自己修復しない          → ReplicaSet が解決
ReplicaSet だけ … 更新・ロールバックが煩雑 → Deployment が解決
```

---

## 1. Pod だけでは不足する点

素の Pod（`kubectl run ... --restart=Never` などで作成した単体の Pod）は、削除するとそのまま消え、再作成されない。すなわち **Pod 自体には自己修復の仕組みがない**。

```bash
kubectl delete pod web
kubectl get pods            # 復活しない
```

本番環境では「常に一定数の Pod を稼働させ続けたい」という要件があるため、この性質では不十分である。これを担うのが ReplicaSet である。

---

## 2. ReplicaSet

### 役割

ReplicaSet は、**指定した数の Pod を常に維持する**コントローラーである。Pod が削除・停止して数が不足すると、自動的に新しい Pod を作成して指定数に戻す（自己修復）。

なお、自己修復を行うのは Pod 自身ではなく、Pod の外側にある ReplicaSet である。ReplicaSet が Pod の数を監視し、不足分を補う。

### マニフェスト例

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: web-rs
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: nginx
        image: nginx
```

| 項目 | 意味 |
| --- | --- |
| `kind: ReplicaSet` | 作成する対象が ReplicaSet であること |
| `spec.replicas` | 維持する Pod の数 |
| `spec.selector.matchLabels` | 管理対象とする Pod をラベルで指定する |
| `spec.template` | ReplicaSet が作成する Pod の設計図（テンプレート） |

1 つのマニフェストの中に、「管理役である ReplicaSet」と「作成される Pod の設計図（template）」の両方が含まれる点が特徴である。

### 適用と自己修復の確認

```bash
kubectl apply -f rs.yaml
kubectl get rs,pods                # ReplicaSet と Pod 3 個を確認
kubectl delete pod <Pod名>         # 1 個削除すると
kubectl get pods                   # 直ちに新しい Pod が作成され、3 個に戻る
```

削除した Pod の名前は消え、代わりに新しい名前の Pod（AGE が短いもの）が現れる。合計数は指定した `replicas` の値に保たれる。

---

## 3. ReplicaSet だけでは不足する点

ReplicaSet は「数の維持」は行うが、**アプリケーションのバージョン更新を安全に行う仕組みを持たない**。イメージのバージョンを上げる際、段階的な入れ替え（無停止更新）や、問題発生時に前のバージョンへ戻す操作（ロールバック）を自動では行えない。これを担うのが Deployment である。

---

## 4. Deployment

### 役割

Deployment は、**ReplicaSet を世代管理し、更新とロールバックを制御する**上位のコントローラーである。以下を提供する。

- **ローリングアップデート**：古い Pod を少しずつ新しい Pod へ置き換える（無停止での更新）
- **ロールバック**：問題が発生した場合、前のバージョンへ戻す
- **世代管理**：バージョンごとに ReplicaSet を作成し、履歴として保持する

構成は次の 3 階層になる。

``` text
Deployment（更新・世代管理）
   └─ ReplicaSet（Pod 数の維持）※バージョンごとに作成される
          └─ Pod（実体）
```

### マニフェスト例

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: nginx
        image: nginx:1.26
```

ReplicaSet のマニフェストと構造はほぼ同じで、`kind` が `Deployment` である点が異なる。

### 更新（ローリングアップデート）

```bash
kubectl apply -f deploy.yaml
kubectl get deploy,rs,pods                  # 3 階層を確認
kubectl set image deploy/web nginx=nginx:1.27
kubectl rollout status deploy/web           # 更新の進捗を確認
kubectl get rs                              # 新旧 2 つの ReplicaSet が確認できる
```

更新を行うと、Deployment は新しいバージョン用の ReplicaSet を新規に作成し、新しい ReplicaSet の Pod 数を増やしながら、古い ReplicaSet の Pod 数を減らしていく。`kubectl get rs` では、古い ReplicaSet（0 個）と新しい ReplicaSet（3 個）が並んで表示される。

### ロールバック

```bash
kubectl rollout undo deploy/web
kubectl rollout status deploy/web
kubectl get rs                              # 旧 ReplicaSet が再び 3 個に戻る
```

ロールバックが可能なのは、**バージョンごとの ReplicaSet が履歴として保持されている**ためである。前のバージョンの ReplicaSet がそのまま残っているため、その Pod 数を再び増やすことで、直ちに元の状態へ戻せる。保持される世代数は `revisionHistoryLimit`（既定値 10）で制御される。

---

## 5. 命令的操作と宣言的管理の違い（注意点）

`kubectl set image` や `kubectl rollout undo` は、稼働中の状態を直接変更する命令的な操作である。これらは手早いが、マニフェスト（YAML）や `kubectl apply` が記録する構成情報とは同期しないため、両者が乖離（ドリフト）する。

実際、`kubectl rollout undo` の実行時には次の警告が表示される。

``` text
Warning: resource deployments/web was previously managed with 'kubectl apply'.
Rolling back will not update the last-applied-configuration annotation ...
Consider using 'kubectl apply' with your previous configuration file instead.
```

これは「命令的な操作と宣言的な管理が乖離するため、マニフェストで管理することが望ましい」という Kubernetes 自身からの示唆である。

したがって、実務では次の方針が推奨される。

- **命令的操作**（`set image`／`rollout undo`） … 緊急時の応急処置として用いる
- **宣言的管理**（マニフェストを修正して `kubectl apply`） … 標準の運用手順とする。構成と実体を一致させ、Git で履歴・レビュー・ロールバックを管理できる

---

## 6. ECS との対応

| ECS | Kubernetes |
| --- | --- |
| サービス（常に N 個維持・ローリング更新・ロールバック） | Deployment（＋ ReplicaSet） |
| タスク定義 | Pod テンプレート（`spec.template`） |
| タスク | Pod |

Deployment が担う「台数維持・更新・ロールバック」は、ECS のサービスが担う役割とほぼ同じである。

---

## 7. まとめ

- 素の Pod は自己修復しない。**ReplicaSet が指定数の Pod を維持する**（自己修復は Pod の外側の ReplicaSet が担う）。
- ReplicaSet は数の維持のみを行い、更新・ロールバックは扱えない。**Deployment が ReplicaSet を世代管理し、ローリングアップデートとロールバックを提供する**。
- 構成は **Deployment → ReplicaSet → Pod** の 3 階層。
- ロールバックが可能なのは、**バージョンごとの ReplicaSet が履歴として保持されている**ため。
- 命令的操作は応急処置とし、標準は**マニフェストを修正して** `kubectl apply` する宣言的管理とする。
