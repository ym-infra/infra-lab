# ローリングアップデート・自己修復・Compose から Kubernetes への移行

- 前: [05-ingress.md](./05-ingress.md) — Ingress（外部公開）
- 基礎: [01-basics.md](./01-basics.md) / [02-replicaset-deployment.md](./02-replicaset-deployment.md) / [03-service-configmap-secret.md](./03-service-configmap-secret.md) / [04-persistent-volume.md](./04-persistent-volume.md)
- 移行元の Compose 構成: [../docker/01-basics.md](../docker/01-basics.md)

## 概要

本ページでは、Deployment（デプロイメント：アプリのあるべき状態を宣言し、Pod の世代管理を担うリソース）が提供する二つの機能 **Rolling Update（ローリングアップデート：無停止でのバージョン更新）** と **Auto Healing（オートヒーリング：自己修復）** を扱い、あわせて **Docker Compose 構成を Kubernetes へ移行する**実践（WordPress + MariaDB）を記録する。いずれも「宣言的管理（望ましい状態を宣言し、コントローラが現状との差分を埋め続ける仕組み）」を土台とする。記載のコマンドと出力は、実際のハンズオンで実行・観測したものである。

## 1. 前提：Deployment・ReplicaSet・Pod の三層構造

本ページの機能は、いずれも次の三層の関係から生まれる。上位が下位に対して「あるべき状態」を指示し、下位はそれを保とうとする。

| 階層 | 役割 |
| --- | --- |
| **Deployment** | アプリのあるべき状態（イメージ・レプリカ数など）を宣言し、更新や切り戻しといった**世代管理**を担う。 |
| **ReplicaSet** | 指定された数の Pod を**常に維持**する。Deployment は更新のたびに新しい ReplicaSet を作り、世代を切り替える。 |
| **Pod** | コンテナが実際に動く最小単位。使い捨てであり、落ちれば作り直される。 |

すなわち **Deployment → ReplicaSet → Pod** という指示系統である。Rolling Update は「Deployment が新旧 ReplicaSet を切り替える動き」、Auto Healing は「ReplicaSet が Pod の数を保つ動き」として説明できる。

## 2. 使用したマニフェスト（deploy.yaml）

nginx を 3 レプリカで動かす Deployment `web` を用いた。

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 3                 # あるべき Pod 数（＝この数を常に維持する）
  selector:
    matchLabels:
      app: web                # 管理対象の Pod を選ぶラベル条件
  template:                   # ここから下が「作る Pod の設計図」
    metadata:
      labels:
        app: web              # selector と一致させる
    spec:
      containers:
      - name: nginx
        image: nginx:1.26     # このイメージ指定を更新の対象にする
```

```bash
# 適用し、3 階層を確認
kubectl apply -f deploy.yaml
kubectl get deploy,rs,pods    # Deployment 1・ReplicaSet 1・Pod 3 が見える
```

## 3. Rolling Update（ローリングアップデート）

### 概念

Rolling Update とは、**稼働中の Pod を一度に全部止めるのではなく、新しい Pod を少しずつ立ち上げながら古い Pod を順に置き換えていく更新方式**である。Deployment はこの更新を、新旧二つの ReplicaSet の世代交代として管理する。常に応答できる Pod が残るため、更新中もサービスが停止しない。

### 実行したコマンド

```bash
# イメージを nginx:1.26 → nginx:1.27 に更新
kubectl set image deploy/web nginx=nginx:1.27

# 更新の進捗を確認
kubectl rollout status deploy/web

# ReplicaSet の世代を確認
kubectl get rs
```

### 実際の出力（kubectl get rs）

更新後は、新旧二つの ReplicaSet が並び、DESIRED（あるべき数）が入れ替わっていた。

``` text
NAME             DESIRED   CURRENT   READY   AGE
web-68cdd476b5   0         0         0       35s    ← 旧世代（nginx:1.26）が 0 に
web-69c6f74b8b   3         3         3       15s    ← 新世代（nginx:1.27）が 3 に
```

二つの ReplicaSet は、そのまま **二つのバージョン（1.26 と 1.27）** に対応する。Deployment が Pod を新旧 ReplicaSet の間で移し替えることで、更新が成立している。

### ロールバック（切り戻し）と、そのとき出た警告

直前の世代へ戻すには `rollout undo` を使う。実行すると次のように世代が入れ替わり、旧バージョンへ戻った。

```bash
kubectl rollout undo deploy/web
kubectl rollout status deploy/web
kubectl get rs
```

``` text
NAME             DESIRED   CURRENT   READY   AGE
web-68cdd476b5   3         3         3       60s    ← 旧世代（1.26）が再び 3 に
web-69c6f74b8b   0         0         0       40s    ← 新世代（1.27）が 0 に
```

このとき、Kubernetes から次の警告が表示された。

``` text
Warning: resource deployments/web was previously managed with 'kubectl apply'.
Rolling back will not update the kubectl.kubernetes.io/last-applied-configuration
annotation, which may cause unexpected behavior on future 'kubectl apply' operations.
Consider using 'kubectl apply' with your previous configuration file instead.
```

### この警告の意味（命令的操作と宣言的管理のズレ）

この警告は、**「命令的な操作」と「宣言的な定義（YAML）」がズレていることを Kubernetes 自身が知らせている**ものである。

- `kubectl set image`（命令的）でイメージを変更した際、**手元の deploy.yaml は書き換えていない**。
- `kubectl rollout undo`（命令的）で戻した際も、**YAML とは同期しない**。
- その結果、「YAML に記録された状態」と「実際の状態」がズレる。Kubernetes は「次に `apply` すると予期しない挙動になるおそれがあるため、以前の設定ファイルで `apply` し直すこと」を勧めている。

**教訓**：`set image` や `rollout undo` は速いが応急処置であり、YAML とズレる。正攻法は **deploy.yaml のイメージを書き換えて** `kubectl apply` **する**こと。こうすれば YAML（定義）と実物が常に一致し、この警告も出ない。Kubernetes 自身が「宣言的（YAML）で管理せよ」と促している点が、宣言的管理の正しさの裏づけとなる。

|  | 命令的（応急） | 宣言的（正攻法） |
| --- | --- | --- |
| 更新 | `kubectl set image ...` | deploy.yaml を編集 → `kubectl apply -f deploy.yaml` |
| 切り戻し | `kubectl rollout undo ...` | deploy.yaml を元に戻す → `kubectl apply -f deploy.yaml` |
| 特徴 | 速いが YAML とズレる（警告が出る） | 定義と実物が一致し、履歴も Git で残せる |

**補足：更新の刻み幅**：「一度に何個ずつ置き換えるか」は Deployment の `strategy.rollingUpdate` にある `maxSurge`（あるべき数を超えて一時的に増やせる Pod 数）と `maxUnavailable`（更新中に欠けてよい Pod 数）で決まる。今回のマニフェストでは明示しておらず、既定値（各 25%）で動作した。

## 4. Auto Healing（自己修復）

### 概念

Auto Healing とは、**Pod が落ちたり削除されたりしても、あるべき数まで自動的に作り直される仕組み**である。Kubernetes は `replicas: 3` といった**望ましい状態（desired state）**を常に監視し、現状がそれを下回ると差分を埋めるように Pod を補充する。

### 実際に観測できた自己修復

学習環境（minikube）を停止・再起動した後も、Deployment `web` は指定どおり 3 つの Pod を保っていた。各 Pod の RESTARTS が増えているのは、再起動時に一度落ちた Pod が自動で立て直された跡である。

``` text
NAME                       READY   STATUS    RESTARTS      AGE
pod/web-7d5875d96b-4lfrp   1/1     Running   1 (23s ago)   6d19h
pod/web-7d5875d96b-j9t8b   1/1     Running   1 (23s ago)   6d19h
pod/web-7d5875d96b-qbbth   1/1     Running   1 (23s ago)   6d19h

deployment.apps/web        3/3     3            3           6d22h
```

### 試し方（Pod を手動で削除してみる）

```bash
# 現在の Pod を確認（3 つ）
kubectl get pods

# 1 つ手動で削除して障害を模擬する
kubectl delete pod <Pod名>

# 直後に確認すると、新しい Pod が自動生成され、再び 3 つに戻る
kubectl get pods
```

削除直後は一時的に 2 つに減るが、数秒で新しい Pod（名前の末尾ハッシュが変わり、AGE が短い）が作られ、3 つに戻る。

**なぜ自動で復旧するのか**：ReplicaSet が「望ましい数」と「現在の数」を絶えず突き合わせ、差分があれば埋めようとするため。Rolling Update と同じく、宣言的管理（あるべき状態を宣言し、コントローラが差分を解消し続ける）が土台になっている。

## 5. Docker Compose → Kubernetes 移行（WordPress + MariaDB）

これまで Docker Compose で動かしていた WordPress + MariaDB 構成を、Kubernetes のマニフェストへ書き換えて移行した。Compose の各要素が Kubernetes のどのリソースに対応するのかを押さえるのが要点となる。

### 対応関係（Compose と Kubernetes）

| Docker Compose | Kubernetes |
| --- | --- |
| `services:` の各サービス | **Deployment**（コンテナを動かす）＋ **Service**（名前解決・接続先） |
| `image:` | Deployment の `image` |
| `environment:`（平文） | `env` の `value` |
| `.env` のパスワードなど機密情報 | **Secret** ＋ `env` の `valueFrom: secretKeyRef` |
| `volumes:`（データ永続化） | **PVC（PersistentVolumeClaim）** ＋ `volumeMounts` |
| サービス名で相互接続（例：`db`） | **Service 名**で名前解決（例：`WORDPRESS_DB_HOST: db`） |

### 使用したマニフェスト（wp-k8s.yaml）

一つのファイルに `---` 区切りで複数リソース（Secret・PVC・Deployment・Service）を記述した。

```yaml
# --- パスワード（Secret）---
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
stringData:
  password: <DB_PASSWORD>
---
# --- MariaDB用の永続化（PVC）---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: db-pvc
spec:
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 1Gi
---
# --- MariaDB（Deployment）---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: db
spec:
  replicas: 1
  selector:
    matchLabels: { app: db }
  template:
    metadata:
      labels: { app: db }
    spec:
      containers:
      - name: mariadb
        image: mariadb
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef: { name: db-secret, key: password }
        - name: MYSQL_DATABASE
          value: wp
        volumeMounts:
        - { name: data, mountPath: /var/lib/mysql }
      volumes:
      - name: data
        persistentVolumeClaim: { claimName: db-pvc }
---
# --- MariaDB（Service：名前解決用）---
apiVersion: v1
kind: Service
metadata:
  name: db
spec:
  selector: { app: db }
  ports:
  - { port: 3306 }
---
# --- WordPress（Deployment）---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress
spec:
  replicas: 1
  selector:
    matchLabels: { app: wordpress }
  template:
    metadata:
      labels: { app: wordpress }
    spec:
      containers:
      - name: wordpress
        image: wordpress
        env:
        - name: WORDPRESS_DB_HOST
          value: db                # ← Service名"db"で繋ぐ（K8sの名前解決）
        - name: WORDPRESS_DB_NAME
          value: wp
        - name: WORDPRESS_DB_USER
          value: root
        - name: WORDPRESS_DB_PASSWORD
          valueFrom:
            secretKeyRef: { name: db-secret, key: password }
        ports:
        - { containerPort: 80 }
---
# --- WordPress（Service）---
apiVersion: v1
kind: Service
metadata:
  name: wordpress
spec:
  selector: { app: wordpress }
  ports:
  - { port: 80 }
```

```bash
# 複数リソースを一括で適用
kubectl apply -f wp-k8s.yaml
```

### ポイント（3つの要素）

> **DB ユーザーについて** — ここでは検証のため DB の管理ユーザー（`root`）をそのままアプリに使っている。実運用では、対象の DB にだけ権限を持つ専用ユーザーを作って割り当てる。管理ユーザーを共有すると、アプリが侵害されたときの影響が DB 全体に及ぶ。

- **Secret（機密情報）**：パスワードは `Secret` に切り出し、参照側は `valueFrom: secretKeyRef` で注入する。`stringData` に平文で書けるが、保存時に Base64 で符号化されるだけで**暗号化ではない**点に注意。値をそのまま `value` に書くのではなく `valueFrom` で参照するのが定石である。
- **PVC（永続化）**：MariaDB のデータ領域 `/var/lib/mysql` を PVC にマウントする。「**箱（コンテナ）は使い捨て、データは外（PVC）に置く**」という原則により、Pod が入れ替わってもデータは残る。
- **Service 名で名前解決**：WordPress は `WORDPRESS_DB_HOST: db` と指定し、MariaDB の Service 名 `db` へ接続する。Compose で「サービス名で相互接続」していたのと同じ感覚で、Kubernetes では Service 名が接続先になる。

### 移行後の確認

両方の Pod が Running になり、WordPress が MariaDB へ接続できることを確認した。

``` text
$ kubectl get pods
NAME                         READY   STATUS    RESTARTS       AGE
db-9f94d74c8-nv9vl           1/1     Running   1 (7m1s ago)   7m19s
wordpress-86b68674cc-77cxt   1/1     Running   0              22s
```

```bash
# 使い捨て Pod から WordPress にアクセスし、接続を確認
kubectl run tmp --rm -it --image=busybox -- wget -qO- -S wordpress
```

``` text
HTTP/1.1 302 Found
Location: http://wordpress/wp-admin/install.php
X-Redirect-By: WordPress
```

`/wp-admin/install.php` へのリダイレクト（302）が返ったことから、**WordPress が Service 名** `db` **経由で MariaDB に到達し、初期セットアップ画面まで進んでいる**ことが確認できる。

## 6. まとめ

| 項目 | 内容 | 支えている仕組み |
| --- | --- | --- |
| Rolling Update | 無停止でのバージョン更新（新旧 Pod を順に置き換え） | Deployment による新旧 ReplicaSet の世代交代 |
| Auto Healing | Pod が落ちても自動で再生成し、数を維持 | ReplicaSet による望ましい状態（replicas）の常時監視 |
| Compose→K8s 移行 | Compose 構成を Deployment / Service / Secret / PVC に置き換え | 宣言的マニフェストと Service 名による名前解決 |

共通する考え方は **「望ましい状態を宣言し、コントローラが現状との差分を埋め続ける」** という宣言的管理である。Rolling Update では `rollout undo` の警告を通じて**命令的操作が宣言的定義とズレる様子**を実地に確認できた。更新も切り戻しも移行も、**YAML を書き換えて** `apply` **する**ことを基本とすると、定義と実物が一致し、トラブルの切り分けもしやすくなる。

**関連ページ**：ReplicaSet と Deployment の基本構造は [02-replicaset-deployment.md](./02-replicaset-deployment.md)、Service・ConfigMap・Secret は [03-service-configmap-secret.md](./03-service-configmap-secret.md)、永続化（PV/PVC）は [04-persistent-volume.md](./04-persistent-volume.md) を参照。
