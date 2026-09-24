# Kubernetes の基礎

- 次: [02-replicaset-deployment.md](./02-replicaset-deployment.md) — ReplicaSet と Deployment による台数維持
- 関連: [03-service-configmap-secret.md](./03-service-configmap-secret.md) / [04-persistent-volume.md](./04-persistent-volume.md) / [05-ingress.md](./05-ingress.md) / [06-rolling-update-auto-healing.md](./06-rolling-update-auto-healing.md)
- 前提知識: [../docker/01-basics.md](../docker/01-basics.md)

---

## 1. Kubernetes（クバネティス）とは

**たくさんのコンテナを、自動で配置・維持・復旧・スケールする"管理役"のソフトウェア**。

### 位置づけ

コンテナが1〜2個なら手で起動すれば足りる。しかし本番環境では、コンテナが数十個、複数のサーバにまたがって動く。これを人手で「どのサーバに置くか・落ちたら立て直す・混雑したら増やす」と運用するのは現実的でない。この運用を丸ごと自動化するのが Kubernetes であり、いわば **コンテナ専用の司令塔（オーケストレーター）** にあたる。

### 主な機能

- **配置（スケジューリング）**：どのノード（サーバ）でどのコンテナを動かすかを決めて配置する
- **維持（自己修復）**：「常に3個動かす」と宣言すれば、落ちても自動で3個に戻す
- **スケール**：負荷に応じてコンテナを増減する
- **入口の管理**：外部からのアクセスを適切なコンテナへ振り分ける

これらは AWS の ECS と役割が似ている。

---

## 2. なぜ必要か（Docker だけでは何が不足するか）

| 段階 | できること | 不足すること |
| --- | --- | --- |
| **Docker** | コンテナを1個動かす | 複数サーバへの配置・自動復旧を人手で行うのは困難 |
| **Kubernetes** | コンテナを**自動で**配置・維持・復旧・スケール | （Docker の不足を補う） |

**Docker はコンテナを作って動かす道具、Kubernetes はそのコンテナを大規模かつ安定して運用する道具**。役割が異なり、K8s は Docker の上位で動作する。

---

## 3. ECS との対応

Kubernetes と ECS はほぼ同じ役割を果たす。違いは **「AWS 専用か、どこでも動くか」** に集約される。

- **ECS** … AWS 専用のコンテナ管理サービス
- **Kubernetes** … 業界標準のコンテナ管理基盤（AWS・Azure・オンプレミス、どこでも動作）

EKS（AWS 上の K8s）や NKP（オンプレの K8s）も中身は Kubernetes であり、K8s を理解すればそれらへ応用できる。

### ECS ↔ Kubernetes 対応表

| ECS | Kubernetes | 役割 |
| --- | --- | --- |
| クラスタ | **クラスタ** | コンテナを動かす土台 |
| タスク | **Pod（ポッド）** | 動く最小単位（コンテナの入れ物） |
| タスク定義 | Deployment など | 設計図 |
| サービス（常にN個維持） | **Deployment＋ReplicaSet** | 台数維持・自己修復 |
| ALB／ターゲットグループ | **Service／Ingress** | 外部からの入口・振り分け |
| 環境変数／Secrets Manager | **ConfigMap／Secret** | 設定・機密の外出し |

名称が異なるだけで、同じような役割を持っている。

---

## 4. 用語

| 用語 | 説明 |
| --- | --- |
| **クラスタ** | コンテナを動かす土台の全体。この中で Pod が動く |
| **ノード** | クラスタを構成する1台1台のサーバ（マシン） |
| **Pod** | K8s で動かす最小単位。中に1個以上のコンテナが入る |
| **kubectl** | K8s を操作するコマンド（Docker における `docker` コマンドに相当） |
| **minikube／kind** | 1台のマシンで学習用の小さな K8s クラスタを作る道具 |

### 道具の関係

``` text
minikube または kind  →  クラスタを「作る」道具（土台を用意）
        ↓
kubectl              →  そのクラスタを「操作する」道具（Pod 起動など）
```

- **kubectl は必須**（操作コマンド。クラスタ自体は作らない）
- **minikube と kind はどちらか一方でよい**（どちらも「1台でミニ K8s」を作る、役割は同じ）
    - kind … 軽量・シンプル
    - minikube … 多機能（ダッシュボードあり）

---

## 5. Pod とは／なぜ必要か

**Pod はコンテナを包む"入れ物"**。K8s はコンテナを直接ではなく、この Pod 単位で扱う。

### Pod が存在する理由

1. **一緒に動かすコンテナをまとめられる**
多くの場合は 1 Pod＝1 コンテナ。ただし「アプリ本体＋ログ収集役（サイドカー）」のように、必ずセットで動かしたい複数コンテナが存在する場合がある。Pod はそれらを **同じIP・同じストレージ・同じライフサイクル** でまとめる（ECS の「1タスクに複数コンテナ」と同じ発想）。
2. **管理する単位を1つに統一できる**
K8s は「Pod を配置する・Pod を N 個維持する」と一貫して扱える。単位が統一されることで管理が単純になる。
3. **特定のコンテナ技術に依存しない**
Pod という抽象を挟むことで、内部のコンテナ実行環境が何であっても同じように扱える。

> Pod は「一緒に動かすコンテナのまとまり＋K8s の管理単位」。ECS のタスクに相当する。

---

## 6. 構築手順（ハンズオン）

### 前提

- コンテナを動かすホスト（例：EC2 / Amazon Linux 2023）に **Docker が導入済み**であること。

### 道具の導入

```bash
# kubectl（操作コマンド）
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install kubectl /usr/local/bin/kubectl

# minikube（クラスタ作成）
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
```

### クラスタの作成

```bash
minikube start --driver=docker
```

- `minikube start` … ミニ K8s クラスタを起動する
- `--driver=docker` … 既存の Docker を使い、K8s のノードを Docker コンテナとして動かす（ホストに Docker がある場合に適する）
- 起動後 `docker ps` を実行すると `minikube` コンテナが確認できる（= K8s ノードの実体は Docker コンテナ）

### クラスタと Pod の確認・起動

```bash
kubectl get nodes                                # ノード一覧。minikube が Ready なら成功
kubectl run web --image=nginx --restart=Never    # Pod（nginx）を起動
kubectl get pods -o wide                         # Pod の状態・IP を確認
kubectl describe pod web                         # Pod の詳細（Events 含む）
```

- `kubectl run web --image=nginx` … 「web という名前の Pod を nginx イメージで起動する」
- `--restart=Never` … 単発の素の Pod（Deployment との対比のため）
- STATUS が `Running` になれば成功

### Pod の操作（Docker コマンドの K8s 版）

```bash
kubectl logs web              # ログ表示（docker logs に相当）
kubectl exec -it web -- bash  # Pod 内に入る（docker exec に相当）。exit で退出
```

---

## 7. `kubectl describe pod` の中身

``` text
Name:             web
Namespace:        default
Priority:         0
Service Account:  default
Node:             minikube/<NODE_IP>
Start Time:       Tue, 08 Sep 2026 23:55:43 +0000
Labels:           run=web
Annotations:      <none>
Status:           Running
IP:               <POD_IP>
IPs:
  IP:  <POD_IP>
Containers:
  web:
    Container ID:   containerd://ddfa32a0db37b04e2be05c24c163411a8287a689e477adb544e27a60aef216bc
    Image:          nginx
    Image ID:       docker.io/library/nginx@sha256:05b8cb60c354a44ab824ea6e7dc69b46d50762cdbe728a347a5b656e6fb3d7c4
    Port:           <none>
    Host Port:      <none>
    State:          Running
      Started:      Tue, 08 Sep 2026 23:55:49 +0000
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-k5d9j (ro)
Conditions:
  Type                        Status
  PodReadyToStartContainers   True
  Initialized                 True
  Ready                       True
  ContainersReady             True
  PodScheduled                True
Volumes:
  kube-api-access-k5d9j:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    Optional:                false
    DownwardAPI:             true
QoS Class:                   BestEffort
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  13m   default-scheduler  Successfully assigned default/web to minikube
  Normal  Pulling    13m   kubelet            spec.containers{web}: Pulling image "nginx"
  Normal  Pulled     13m   kubelet            spec.containers{web}: Successfully pulled image "nginx" in 5.631s (5.631s including waiting). Image size: 66343926 bytes.
  Normal  Created    13m   kubelet            spec.containers{web}: Container created
  Normal  Started    13m   kubelet            spec.containers{web}: Container started
```

Pod の詳細情報のうち、特に重要な項目は以下。

| 項目 | 意味 |
| --- | --- |
| `Status: Running` / `Ready: True` | Pod が起動し正常に動作している |
| `Node: minikube` | 配置されたノード（サーバ）。**K8s のスケジューラが決定する** |
| `IP: <POD_IP>` | Pod の IP（動く単位に IP が割り当てられる） |
| `Restart Count: 0` | 再起動回数（0＝一度も落ちていない） |
| `Events`（末尾） | 起動過程の時系列ログ。**トラブル時の第一の確認先** |

### Events の流れ（Pod 起動プロセス）

``` text
Scheduled  → K8s が Pod を配置するノードを決定
Pulling    → イメージのダウンロード中
Pulled     → 取得完了
Created    → コンテナ作成
Started    → コンテナ起動
```

ECS のタスク状態遷移（PROVISIONING→RUNNING）や、`docker logs` で見る起動ログの K8s 版にあたる。Pod が起動しない場合は、この Events で原因を特定する。

---

## 8. まとめ・注意点

- **Kubernetes は ECS の業界標準版**。どこでも動くコンテナ管理基盤で、用語は異なるが役割は対応する。
- **kubectl は操作コマンド（必須）、minikube・kind はクラスタ作成の道具（どちらか一方）**。
- **Pod はコンテナの入れ物であり K8s の最小単位**。ECS のタスクに相当する。
- `describe` の Events が診断の要（ECS ログ・Docker ログと同じ発想）。

### 環境上の注意点

- **minikube はメモリ消費が大きい**。小さめのインスタンス（例：メモリ4GB）では警告が出ることがある。不安定な場合は `minikube start --driver=docker --memory=2200mb` のように割り当てを下げて作り直す。
- **学習で多くのイメージを取得するとディスクが逼迫しやすい**。`docker system prune -a -f` で未使用イメージ・停止コンテナを削除して空きを確保する（実行中の minikube のイメージは削除されない）。

---

## 9. ダッシュボード

### ダッシュボードとは

クラスタの状態（Pod・Deployment・Node など）を **Web 画面で見られる** ツール。`kubectl get pods` の視覚版にあたる。

### 開き方

- `minikube dashboard --url` → `http://127.0.0.1:PORT/...` の URL が出る
- ただし `127.0.0.1` **＝そのマシン自身**なので、**EC2 の中からしか開けない**。手元 PC から見る場合は、**SSH ポートフォワードでトンネルを掘る**。

``` text
# 手元 PC で実行（EC2 の 8001 を手元の 8001 に転送する）
ssh -L 8001:127.0.0.1:8001 <USER>@<EC2_HOST>

# EC2 側で実行（ループバックにだけ待ち受ける）
kubectl proxy --port=8001

▼手元のブラウザからのアクセス先
http://127.0.0.1:8001/api/v1/namespaces/kubernetes-dashboard/services/http:kubernetes-dashboard:/proxy/
```

`kubectl proxy` は**認証を通した状態で API サーバへ中継する**ため、`--address='0.0.0.0'` や `--accept-hosts='^.*$'` で外部に開いてはいけない。到達できた時点でクラスタを操作できるのと同じであり、セキュリティグループだけを防壁にするのは危険である。

### できること／位置づけ

- Pod・ログ・Node などを画面で確認、**作成もフォーム／YAML で可能**
- ただし実務の標準は **YAML ＋** `kubectl apply`（宣言的・再現可能）。ダッシュボードは **「見る・確認する」用**

### ダッシュボードで作成しない理由

ダッシュボードからも作成（フォーム入力／YAML 貼り付け）はできるが、実務では非推奨。理由は以下。

- **記録が残らない・再現できない**：フォーム操作は「何をしたか」が残らず、同じ構成を作り直せない。
- **バージョン管理できない**：YAML ファイルなら Git で履歴・レビュー・ロールバックができるが、画面操作は追えない。
- **手作業ミス・属人化**：入力は人によってバラつき、打ち間違いも起きる。
- **自動化できない**：CI/CD は YAML を `kubectl apply` で適用する。画面操作は組み込めない。
- **監査できない**：本番では「いつ・誰が・何を変えたか」の追跡が必要。
- **セキュリティ**：作成権限を持つダッシュボードを外部公開すると攻撃対象になりうる（過去に乗っ取り事例あり）。
