# 永続化（PersistentVolume / PVC）

- 前: [03-service-configmap-secret.md](./03-service-configmap-secret.md) — Service / ConfigMap / Secret
- 次: [05-ingress.md](./05-ingress.md) — Ingress（外部公開）
- 関連: [06-rolling-update-auto-healing.md](./06-rolling-update-auto-healing.md) — Compose から Kubernetes への移行で PVC を使う例

本ページでは、Kubernetes におけるデータの永続化の仕組みである PersistentVolume（PV）と PersistentVolumeClaim（PVC）について整理する。

コンテナ（Pod）は使い捨てを前提とするため、Pod の内部に保存したデータは Pod の削除・再作成で失われる。消えては困るデータは、Pod の外部にあるストレージへ保存する必要がある。この「Pod の外部にストレージを用意して接続する」仕組みが PV / PVC であり、Docker のボリューム（`-v`）や ECS の EFS マウントに相当する。

---

## 1. PV と PVC の役割

| 用語 | 役割 |
| --- | --- |
| **PersistentVolume（PV）** | 実際のストレージ（ディスク）の実体 |
| **PersistentVolumeClaim（PVC）** | 「これだけのストレージが欲しい」という要求（申請） |

Pod は PVC をボリュームとしてマウントして利用する。PVC が PV に結び付く（Bound）ことで、Pod は永続ストレージを使えるようになる。

アプリケーション（Pod／PVC）は「必要な容量・アクセス方式」を要求するのみで、背後の実際のストレージが何であるか（EBS・EFS・ローカルディスク等）を意識しなくてよい。ストレージの実装とアプリケーションを分離できる点が、この 2 層構成の利点である。

---

## 2. StorageClass による動的プロビジョニング

PVC を作成すると、**StorageClass** の仕組みによって PV が自動的に作成され、PVC に結び付けられる（動的プロビジョニング）。PV を手動で用意する必要はない。

- **minikube** … 既定の StorageClass（`standard`）が PV を自動生成する。
- **EKS 等のクラウド** … StorageClass に応じて EBS や EFS が自動的に割り当てられる。

---

## 3. ハンズオン：Pod を削除してもデータが残ることの確認

**全体の流れ**：PVC を用意 → writer でデータを書く → writer を削除 → reader で同じ PVC をマウントして読む。writer と reader は別の Pod だが、**同じ PVC を指定する**ため、同じデータにアクセスできる。

### ① PVC（ストレージの要求）を作成する

`pvc.yaml`：

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mydata-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

```bash
kubectl apply -f pvc.yaml
kubectl get pvc          # STATUS が Bound になる（PV が割り当てられた）
kubectl get pv           # 自動生成された PV を確認
```

### ② writer Pod でデータを書き込む

`writer.yaml`：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: writer
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "echo '永続化テスト' > /data/test.txt && sleep 3600"]  # /data に書き込む
    volumeMounts:
    - name: vol
      mountPath: /data                 # PVC をこのパスにマウント
  volumes:
  - name: vol
    persistentVolumeClaim:
      claimName: mydata-pvc            # ① で作った PVC を指定
```

```bash
kubectl apply -f writer.yaml
kubectl exec writer -- cat /data/test.txt      # → 永続化テスト（書き込めたことを確認）
```

### ③ writer Pod を削除する

```bash
kubectl delete pod writer
```

Pod は削除されるが、データは Pod ではなく PVC／PV に保存されているため残る（PVC 自体は削除していない）。

### ④ reader Pod で同じ PVC をマウントして読み取る

`reader.yaml`（writer と同じ `claimName: mydata-pvc` を指定するのがポイント）：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: reader
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "sleep 3600"]
    volumeMounts:
    - name: vol
      mountPath: /data
  volumes:
  - name: vol
    persistentVolumeClaim:
      claimName: mydata-pvc            # ② の writer と同じ PVC を指定
```

```bash
kubectl apply -f reader.yaml
kubectl exec reader -- cat /data/test.txt      # → 永続化テスト（データが残っている）
```

writer を削除したにもかかわらず、reader で同じファイルを読み取れる。これは、writer と reader が **同じ PVC（`mydata-pvc`）** を指定しており、データが Pod ではなく PVC／PV に保存されているためである。

> 補足：writer が自分で書いて自分で読む（②）だけでは、通常のコンテナでも可能であり永続化の確認にはならない。**writer を削除した後、別の Pod（reader）で読み取れること（③④）** が、永続化の確認となる。

---

## 4. `kubectl get pvc` / `kubectl get pv` の読み方

| 項目 | 意味 |
| --- | --- |
| `STATUS: Bound` | PVC と PV が結び付き、利用可能な状態 |
| `STORAGECLASS: standard` | この StorageClass が PV を自動生成した |
| `CLAIM: default/mydata-pvc`（PV 側） | この PV がどの PVC に結び付いているか |
| `RECLAIM POLICY: Delete`（PV 側） | PVC を削除すると PV（およびデータ）も削除される |

### RECLAIM POLICY に関する注意

既定の `Delete` では、**PVC を削除すると PV とデータも削除される**。消えては困る本番データでは `Retain`（PVC を削除しても PV を残す）を選択するなど、データのライフサイクルに応じた設定が必要となる。

---

## 5. 本番環境での位置づけ

- データベースなど、状態を保持するアプリケーション（ステートフル）を Kubernetes で動かす際に用いる。Pod が再作成・入れ替わってもデータが残る。
- EKS では PV の実体として **EBS（ブロックストレージ）** や **EFS（ファイルストレージ）** が割り当てられる。今回 minikube で用いた `standard` クラスの、クラウド版にあたる。

---

## 6. Docker / ECS との対応

| Docker | ECS | Kubernetes |
| --- | --- | --- |
| `docker volume create` | EFS の作成 | PVC（PV が自動割り当て） |
| `-v mydata:/var/lib/mysql` | タスク定義の volume / mountPoint | Pod の `volumeMounts` ＋ `persistentVolumeClaim` |
| コンテナを削除してもデータが残る | タスクを削除してもデータが残る | Pod を削除してもデータが残る |

いずれも「コンテナは使い捨て、データは外部のストレージへ」という同じ考え方に基づく。繋ぎ方（マウント）と、実体のストレージが異なるのみである。
