# Service / ConfigMap / Secret

- 前: [02-replicaset-deployment.md](./02-replicaset-deployment.md) — ReplicaSet と Deployment
- 次: [04-persistent-volume.md](./04-persistent-volume.md) — 永続化（PersistentVolume / PVC）
- 関連: [05-ingress.md](./05-ingress.md) — Service の外側にあたる外部公開の仕組み

本ページでは、Service（Pod への安定した入口）と、ConfigMap / Secret（設定と機密の外出し）について整理する。あわせて、ECS や他のストレージとの関係で混同しやすい点も末尾にまとめる。

---

## 1. Service ―― Pod への安定した入口

### 背景となる課題

Pod には IP が割り当てられるが、**Pod の IP は固定ではない**。Pod が再作成される（ローリングアップデート、ノード障害、削除など）たびに、新しい IP が割り当てられる。したがって、特定の Pod の IP を宛先として通信することはできない。

```bash
kubectl get pods -o wide      # Pod の IP を確認
kubectl delete pod <Pod名>     # 削除すると
kubectl get pods -o wide      # 再作成された Pod は別の IP になる
```

### Service の役割

Service は、**変化し続ける Pod 群に対する、変化しない固定の入口**である。固定の IP と DNS 名を持ち、ラベルによって現在稼働している Pod 群へ自動的に振り分ける（ロードバランシング）。Pod が入れ替わって IP が変わっても、Service のアドレスは変わらない。

### 作成と確認

```bash
kubectl expose deployment web --port=80 --target-port=80 --name web-svc
kubectl get svc                       # web-svc の固定 IP（CLUSTER-IP）を確認
kubectl get endpoints web-svc         # 振り分け先の Pod 群（IP）を確認
```

- `--port` … Service が受けるポート
- `--target-port` … 転送先の Pod のポート
- 振り分け対象の Pod は、Deployment のラベル（`app: web`）で自動的に選択される

### 動作確認（Service 名でアクセス）

```bash
kubectl run tmp --rm -it --image=busybox -- wget -qO- web-svc
```

Pod の IP ではなく **Service 名**（`web-svc`）でアクセスすると、Kubernetes 内蔵の DNS が Service の固定 IP に変換し、現在の Pod 群へ振り分ける。Pod を削除・再作成しても、Service 名でのアクセスは変わらず成功する。これにより、**Pod の入れ替わりは利用者から見えない**（Service が固定の入口となり、複数レプリカが処理を継続するため）。

### Service の種類

| 種類 | アクセス範囲 |
| --- | --- |
| **ClusterIP**（既定） | クラスタ内部のみ |
| **NodePort** | ノードのポート経由で外部からアクセス可能 |
| **LoadBalancer** | クラウドのロードバランサー経由で外部公開 |

クラウド（EKS 等）では `type: LoadBalancer` を指定すると、AWS が実際のロードバランサー（ALB 等）を作成し、公開用の DNS 名を払い出す。これは ECS における ALB／ターゲットグループに相当する。

なお、minikube のようなローカル環境では外部ロードバランサーが存在しないため、`type: LoadBalancer` を指定しても `EXTERNAL-IP` は `<pending>` のままになる。NodePort も、ノードの実体が Docker コンテナであるため、待ち受けるのはクラスタ内部のアドレスになる。手元の PC から参照するには、`kubectl port-forward` とセキュリティグループ、または SSH ポートフォワードで別途「橋」を架ける必要がある。

### ECS との対応

| ECS | Kubernetes |
| --- | --- |
| ALB／ターゲットグループ（固定の入口・振り分け） | Service（特に LoadBalancer 型） |
| 内部での名前解決 | Service（ClusterIP 型） |

---

## 2. ConfigMap / Secret ―― 設定と機密の外出し

### 役割

アプリケーションの設定や機密情報をイメージに埋め込まず、外部から注入するための仕組みである。

| 種類 | 用途 | 格納される内容 |
| --- | --- | --- |
| **ConfigMap** | 設定（非機密）の外出し | 環境名、接続先、ログレベル、設定ファイル等 |
| **Secret** | 機密の外出し | パスワード、API キー、証明書等 |

### 作成

```bash
kubectl create configmap appcfg --from-literal=MODE=prod
kubectl create secret generic dbsec --from-literal=PASS=<DB_PASSWORD>
```

### 重要な注意：Secret は暗号化されていない

Secret に格納した値は暗号化ではなく **Base64 エンコード**されているのみである。次のコマンドで容易に平文へ戻せる。

```bash
kubectl get secret dbsec -o jsonpath='{.data.PASS}' | base64 -d
# → <DB_PASSWORD>（平文で表示される）
```

したがって「Secret」という名称であっても、それ自体で機密が保護されるわけではない。本番環境では以下の対策を併用する必要がある。

- etcd（Kubernetes の設定保存領域）の保存時暗号化
- RBAC による、Secret を参照できる主体の制限
- 外部シークレット管理サービスとの連携（例：AWS Secrets Manager と External Secrets Operator の連携）

なお、ECS で用いた AWS Secrets Manager は、暗号化・権限管理・ローテーションに対応した機密管理サービスであり、Kubernetes からも連携して利用できる。

### Pod への注入方法

注入方法には 2 種類あり、いずれも実務で用いられる。

**① 環境変数として渡す**

```bash
kubectl set env --from=configmap/appcfg deploy/web
kubectl set env --from=secret/dbsec deploy/web
kubectl exec deploy/web -- env | grep -E 'MODE|PASS'   # MODE=prod / PASS=<DB_PASSWORD> を確認
```

**② ファイルとしてマウントする**（設定ファイルや証明書に適する）

ConfigMap を nginx の表示ファイルとしてマウントする例。

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: web-html
data:
  index.html: |
    <h1>ConfigMap から表示（バージョン1）</h1>
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web2
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web2
  template:
    metadata:
      labels:
        app: web2
    spec:
      containers:
      - name: nginx
        image: nginx
        volumeMounts:
        - name: html
          mountPath: /usr/share/nginx/html
      volumes:
      - name: html
        configMap:
          name: web-html
```

適用後、ConfigMap の内容を書き換えて Pod を再作成すると、**イメージを変更していないにもかかわらず、表示内容が変わる**。

```bash
kubectl create configmap web-html --from-literal=index.html='<h1>設定を変更（バージョン2）</h1>' --dry-run=client -o yaml | kubectl apply -f -
kubectl rollout restart deployment web2
```

これは「設定をイメージに焼き込まず、外部から差し込む」ことの利点を端的に示す。すなわち、**同一のイメージを、環境ごとに異なる設定で動作させられる**（開発・検証・本番で ConfigMap／Secret を切り替える）。

### ECS との対応

| ECS | Kubernetes |
| --- | --- |
| 環境変数（非機密） | ConfigMap |
| Secrets Manager（機密） | Secret（本番では Secrets Manager 連携が望ましい） |

WordPress を ECS で構築した際の構成（`WORDPRESS_DB_HOST` 等を環境変数、`WORDPRESS_DB_PASSWORD` を Secrets Manager から注入）は、Kubernetes では ConfigMap と Secret に対応する。設定と機密を分離して外出しするという考え方は共通である。

---

## 3. 補足：混同しやすい点の整理

### ECS のクラスタと Kubernetes のクラスタの違い

いずれも「コンテナを動かす管理システム（オーケストレーター）」の土台であり、役割は同じである。異なるのは実装（製品）である。

|  | ECS のクラスタ | Kubernetes のクラスタ（minikube） |
| --- | --- | --- |
| 種別 | AWS 専用のコンテナ管理サービス | 業界標準のコンテナ管理基盤 |
| 稼働 | AWS が運用 | 自分で minikube 上に構築 |
| 動作範囲 | AWS 内 | どこでも（AWS・Azure・オンプレ） |

用語も対応する（タスク↔Pod、サービス↔Deployment/Service）。役割が同じであるため似て見えるが、別の製品である。

### 外出しの「種類」と繋ぎ方の違い

外出しする対象によって、繋ぎ方（渡し方）が異なる。

| 外出しする対象 | 繋ぎ方 | AWS の例 | Kubernetes の例 |
| --- | --- | --- | --- |
| 設定（非機密） | 環境変数 | 環境変数 | ConfigMap |
| 機密 | 環境変数／参照 | Secrets Manager | Secret |
| ファイル・ディスク | **マウント** | EFS・EBS | **PersistentVolume / PVC** |
| データベース | **ネットワーク接続** | RDS | クラスタ外の RDS 等に接続 |

### PersistentVolume / PVC と EFS・RDS の関係

- **PersistentVolume（PV）／PersistentVolumeClaim（PVC）** … Kubernetes において、ストレージ（ディスク）を Pod にマウントするための仕組み。Docker のボリューム（`-v`）や ECS の EFS マウントに相当する。PVC は「必要なストレージを要求する申請」、PV は「実際に割り当てられるストレージ」である。
- **EFS** … 実体としてのファイルストレージ。EKS では EFS を PV の実体として利用できる（PV/PVC と関連する）。
- **RDS** … マネージドのデータベース。マウントするものではなく、ネットワーク経由で接続する。したがって PV/PVC とは別のもの（上表の「データベース」の行にあたる）。

要点として、PV/PVC は「ファイル・ディスクをマウントする」仕組みであり EFS・EBS が実体となる。一方 RDS はネットワーク接続で利用するデータベースであり、マウントの対象ではない。詳細は [04-persistent-volume.md](./04-persistent-volume.md) にまとめる。
