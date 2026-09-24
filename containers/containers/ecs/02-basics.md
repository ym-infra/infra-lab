# ECS を動かす

- 全体像 … [01-overview.md](./01-overview.md)
- ECS と RDS で WordPress を構築する … [03-rds-wordpress.md](./03-rds-wordpress.md)
- EFS に画像データを保管する … [04-efs.md](./04-efs.md)
- インメモリキャッシュ（Valkey/Redis） … [05-elasticache.md](./05-elasticache.md)
- Docker の基本 … [../docker/01-basics.md](../docker/01-basics.md)

## 全体の流れ（ざっくり）

```text
自作イメージ（helloapp:v1）
   ↓ ① ECRにpush（AWSの倉庫へ）
ECR（helloapp:v1）
   ↓ ② タスク定義（設計図：どのイメージ・ポート・環境変数）
   ↓ ③ クラスター（土台）＋ サービス（常に動かす）
ECS Fargate でタスク（コンテナ）が起動
   ↓ ④ パブリックIP or ALB でアクセス
ブラウザで表示
```

> 対応イメージ：**タスク定義＝Composeのyml**、**タスク＝コンテナ**、**サービス＝常に動かす管理係**。

---

## ① 前提：EC2にECR権限（IAMロール）

- EC2からAWSを操作するには **IAMロール** が必要。
- 確認：`aws sts get-caller-identity` が成功すればOK（`Account`が自分のID）。
- 失敗（`Unable to locate credentials`）→ EC2に **ECR権限付きIAMロール** をアタッチ（コンソール：EC2→アクション→セキュリティ→IAMロールを変更）。
- **鍵はEC2に置かず、ロールで一時発行**が鉄則。

## ② ECRにイメージをpush

```bash
# 変数にまとめると楽
ACCT=<AWS_ACCOUNT_ID>
REGION=ap-northeast-1
ECR=$ACCT.dkr.ecr.$REGION.amazonaws.com

# 倉庫を作る（コンソールでもOK）
aws ecr create-repository --repository-name helloapp --region $REGION

# ECRにログイン（一時トークンをdocker loginに渡す）
aws ecr get-login-password --region $REGION | docker login --username AWS --password-stdin $ECR

# タグ付け（手元イメージに、ECR宛の名札を貼る）
docker tag helloapp:v1 $ECR/helloapp:v1

# アップロード
docker push $ECR/helloapp:v1

# 確認
aws ecr describe-images --repository-name helloapp --region $REGION
```

**ポイント**

- 上げるのは **フォルダやDockerfileではなくイメージ**（`docker images` に出るやつ）。
- `docker tag` ＝宛名書き、`docker push` ＝発送。**tag→pushの順**。
- **倉庫名・タグ（`:v1`）は例示なので、実際に作った名前に読み替える**。

## ③ ECSで動かす

**クラスター（土台）**：AWS Fargate（サーバーレス。サーバ管理不要）で作成。

**タスク定義（設計図）**：

| 項目 | 値 | Compose対応 |
| --- | --- | --- |
| イメージURI | `…/helloapp:v1`（`:v1`まで正確に） | `image:` |
| コンテナポート | **5000**（アプリの待ち受けポート。80のままは×） | `ports:` |
| 環境変数 | `MESSAGE=…` | `environment:` |
| タスク実行ロール | `ecsTaskExecutionRole`（**ECRから取得＋ログ出力に必須**） | （ECS特有） |
| ログ収集 | ON推奨（CloudWatchでデバッグ用） | - |

> ロールは2種類。**タスクロール＝アプリがAWSを叩く用（今回空）／タスク実行ロール＝ECSがイメージ取得する用（必須）**。

**サービス（運用ルール）**：

- 起動タイプ：Fargate／必要なタスク数：**1**（増やせば同じ箱を複数＝自己修復・冗長化）。
- ネットワーキング：**ECSと同じVPCを選ぶ**／パブリックIP：オン（直接アクセスするなら）。
- セキュリティグループ：アクセス元から **5000番** を許可。

## ④ アクセス

- **タスクのパブリックIP**：ECS→サービス→タスク→タスクをクリック→ネットワークの **パブリックIP** → `http://<TASK_PUBLIC_IP>:5000`
- **ALB経由**（本番向け）：固定URLで、複数タスクに振り分け・ヘルスチェック。タスクは非公開(プライベート)にできる。

---

## メモ

- **タスク定義を作っただけでは動かない**。サービス（またはRun Task）で初めて起動＝「イメージとコンテナ」の関係と同じ。
- **VPCを揃える**：ALB・タスク・RDSは**全部同じVPC**。
- **ポートは5000**：アプリの待ち受けと合わせる。80のままだと「起動してるのに繋がらない」。
- `Reachability may be impacted` / 504 / unhealthy：ALB→タスクが通ってないサイン。**タスクSGでALBからの5000を許可**、**ターゲット/ヘルスチェックのポートが5000**か確認。
- **タグは正確に(`:v1`)**：省略すると`latest`を探して失敗。
