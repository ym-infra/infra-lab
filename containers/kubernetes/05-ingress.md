# Ingress（外部公開）

- 前: [04-persistent-volume.md](./04-persistent-volume.md) — 永続化（PersistentVolume / PVC）
- 次: [06-rolling-update-auto-healing.md](./06-rolling-update-auto-healing.md) — ローリングアップデート・自己修復・Compose からの移行
- 関連: [03-service-configmap-secret.md](./03-service-configmap-secret.md) — Ingress の振り分け先となる Service

本ページでは、外部からの HTTP アクセスを受け付ける Ingress について整理する。

Ingress は、**外部からの HTTP アクセスを、ホスト名や URL のパスに基づいて適切な Service へ振り分ける、上位の入口**である。ECS における ALB（ホスト／パスベースのルーティング）や、リバースプロキシに相当する。

---

## 1. Service だけでは不足する点

Service（NodePort / LoadBalancer 型）でも外部公開はできるが、次の制約がある。

- サービスごとに入口（ロードバランサーやポート）が必要になり、アプリケーションが増えるほど入口も増える。
- URL のパス（`/shop`、`/blog` 等）による振り分けができない。
- ホスト名（`app1.example.com`、`app2.example.com` 等）による振り分けができない。

これらを解決し、**1 つの入口で複数の Service に振り分ける**のが Ingress である。

---

## 2. Ingress の役割と全体構成

Ingress は、1 つの入口で受けた HTTP アクセスを、ホスト名やパスに応じて各 Service へ振り分ける。

``` text
外部からのアクセス
      ↓
  [Ingress]（1 つの入口・振り分け）
      ├─ web.local        → Service A → Pod
      ├─ blog.local       → Service B → Pod
      └─ /api             → Service C → Pod
```

Ingress は Service の前段に位置する。

``` text
外部 → Ingress（振り分け）→ Service（各アプリへ）→ Pod
```

|  | 役割 | レベル |
| --- | --- | --- |
| **Service** | 1 つのアプリ（Pod 群）への入口 | IP・ポート（L4） |
| **Ingress** | 複数の Service を束ねる HTTP の入口。ホスト名／パスで振り分け | HTTP（L7） |

---

## 3. 準備：Ingress Controller の有効化

Ingress（振り分けルール）を実際に機能させるには、振り分けを実行する **Ingress Controller** が必要である。minikube ではアドオンで有効化する。

```bash
minikube addons enable ingress
kubectl get pods -n ingress-nginx     # ingress-nginx-controller が Running になるまで待つ
```

Ingress（ルールの定義）と Ingress Controller（ルールを実行する実体）はセットで用いる。

---

## 4. ハンズオン：ホスト名で振り分ける

前提として、振り分け先の Service（例：`web-svc`。nginx の Deployment に対する Service）が存在するものとする。

### ① Ingress リソースを作成する

`ingress.yaml`：

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-ingress
spec:
  rules:
  - host: web.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-svc
            port:
              number: 80
```

| 項目 | 意味 |
| --- | --- |
| `host: web.local` | このホスト名宛のアクセスを対象とする |
| `path: /` ／ `pathType: Prefix` | このパス（`/` 以下すべて）を対象とする |
| `backend.service.name: web-svc` | 振り分け先の Service |
| `port.number: 80` | その Service のポート |

```bash
kubectl apply -f ingress.yaml
kubectl get ingress          # HOSTS（web.local）と ADDRESS（minikube の IP）を確認
```

### ② 動作確認

```bash
minikube ip                                        # minikube の IP を確認（例：<MINIKUBE_IP>）
curl -H "Host: web.local" http://$(minikube ip)/   # web.local 宛として送信
```

nginx の Welcome ページが返れば成功。`web.local` 宛の HTTP が、Ingress → web-svc → nginx と振り分けられている。

- `-H "Host: web.local"` は「web.local というホスト名で来たことにする」指定。実際のドメインを用意しなくても、ホスト名ベースの振り分けを確認できる。

---

## 5. Ingress の利点

同じ 1 つの入口に対して、ホスト名やパスごとのルールを追加することで、複数の Service を 1 つの入口で束ねられる。

``` text
1 つの入口（例：<MINIKUBE_IP>）
   ├─ web.local   → web-svc
   ├─ blog.local  → blog-svc
   └─ /api        → api-svc
```

サービスごとにロードバランサーを用意する必要がなく、URL による振り分けも行える。

---

## 6. ECS との対応

| ECS | Kubernetes |
| --- | --- |
| ALB（ホスト／パスベースのルーティング） | Ingress ＋ Ingress Controller |
| ターゲットグループ | Service |
| タスク | Pod |

Ingress が担う「HTTP をホスト名・パスで振り分ける入口」は、ECS の ALB に相当する。

---

## 7. まとめ

- **Ingress は、外部の HTTP アクセスをホスト名・パスで振り分ける、1 つの上位入口**である。
- Service だけではサービスごとに入口が必要になり URL による振り分けもできないが、**Ingress は 1 つの入口で複数の Service を束ねる**。
- Ingress（ルール）と **Ingress Controller（実行役）** をセットで用いる。
- 構成は **外部 → Ingress → Service → Pod**。ECS の ALB に相当する。
