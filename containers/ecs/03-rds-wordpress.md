# ECS と RDS で WordPress を構築する

- 全体像 … [01-overview.md](./01-overview.md)
- ECS を動かす … [02-basics.md](./02-basics.md)
- EFS に画像データを保管する … [04-efs.md](./04-efs.md)
- インメモリキャッシュ（Valkey/Redis） … [05-elasticache.md](./05-elasticache.md)
- Docker の基本 … [../docker/01-basics.md](../docker/01-basics.md)

## ゴール構成

```text
ブラウザ
   ↓
[ECS Fargate：WordPress]  ← 自作イメージ(ECR)
   ├─ 記事・設定 → RDS(MariaDB) 3306で接続
   │                 認証情報はSecrets Managerから注入(valueFrom)
   └─ 画像・メディア → ※今はコンテナ内=使い捨て。本番はEFS/S3に入れる必要あり。
```

> 考え方：**コンテナは使い捨て。消えたら困るデータは外へ保管しなければいけない。**
>
> DB に入るデータ（記事・設定）は RDS で永続化できているが、画像などのファイルはコンテナのエフェメラルストレージに置かれたままで、タスクの再起動で失われる。→ [04-efs.md](./04-efs.md) で対応する。

---

## 事前準備（手動でやること）

| 準備 | 内容 |
| --- | --- |
| WPイメージ | `wordpress` を ECR(`wordpress:v1`) に push（下記コマンド参照） |
| RDS作成 | MariaDB / **ECSと同じVPC** / **パブリックアクセス なし** / 認証情報は **Secrets Managerで管理** |
| `wp` DB作成 | **RDSは自動作成しない**ので手動で `CREATE DATABASE wp;`（WPは器=DBを作らない。テーブルは作る） |
| SG(RDS) | インバウンド **3306** を ECSタスクのSG（or VPCのCIDR）から許可 |
| 実行ロール | `ecsTaskExecutionRole` に **シークレット読取(`secretsmanager:GetSecretValue`)** を付与 |

WPイメージを ECR に push する手順：

```bash
# 変数（自分の環境に合わせる）
ACCT=<AWS_ACCOUNT_ID>
REGION=ap-northeast-1
ECR=$ACCT.dkr.ecr.$REGION.amazonaws.com

# ① 公式WordPressイメージを取得（手元に無ければ）
docker pull wordpress:latest

# ② ECRに倉庫を作る
aws ecr create-repository --repository-name wordpress --region $REGION

# ③ ECRにログイン
aws ecr get-login-password --region $REGION | docker login --username AWS --password-stdin $ECR

# ④ タグ付け（公式イメージにECR宛の名札を貼る）→ push
docker tag wordpress:latest $ECR/wordpress:v1
docker push $ECR/wordpress:v1

# ⑤ 確認
aws ecr describe-images --repository-name wordpress --region $REGION
```

---

## WordPress タスク定義

| 項目 | 値 |
| --- | --- |
| イメージURI | `…/wordpress:v1` |
| コンテナポート | **80**（WPはApache=80。helloappの5000とは違う） |
| CPU/メモリ | 0.5 vCPU / 1 GB |
| 環境変数 `WORDPRESS_DB_HOST` | RDSのエンドポイント（`<RDS_ENDPOINT>`） |
| 環境変数 `WORDPRESS_DB_NAME` | `wp` |
| 環境変数 `WORDPRESS_DB_USER` | `admin` |
| **シークレット** `WORDPRESS_DB_PASSWORD` | **タイプ** `valueFrom` で `<SECRET_ARN>:password::` |

> HOST/NAME/USERは平文(`value`)、**パスワードだけ `valueFrom`** でSecrets Managerから注入＝機密分離。

> **DB ユーザーについて** — ここでは検証のため DB の管理ユーザー（`root` / `admin`）をそのままアプリに使っている。実運用では、対象の DB にだけ権限を持つ専用ユーザーを作って割り当てる。管理ユーザーを共有すると、アプリが侵害されたときの影響が DB 全体に及ぶ。


---

## 詰まったところ

### 1. アクセスがタイムアウト（ERR_TIMED_OUT）

- **ポート違い**：WPは**80**。SGで80をマイIPから許可。helloappの5000のままにしない。
- **タスク再起動でIPが変わる**：クラッシュ再起動を繰り返すと毎回パブリックIPが変わる。古いIPを見てた。

### 2. Error establishing a database connection（DB接続エラー）

- **RDSがSSL必須**：`mysql`で繋ぐと `ERROR 3159 ... require_secure_transport=ON`。WPはデフォルトSSL無しなので弾かれる。
    - 対処（学習）：**カスタムパラメータグループ**を作り `require_secure_transport=0` → RDSに適用 → **再起動**（デフォルトのパラメータグループは編集不可。カスタムを作って切り替える）。
    - 本番：RDSはON維持のまま、**WP側をSSL対応**（`WORDPRESS_CONFIG_EXTRA`＋CA証明書）。セキュリティは下げずアプリを合わせる。
- **RDSのSGがWPタスクを許可してない**：EC2からは繋がるがWPタスクは別SG。3306を許可（VPCのCIDR `<VPC_CIDR>` で一括でも可）。
- **パスワードの注入ミス**：`WORDPRESS_DB_PASSWORD` のタイプが `value`（平文） になっていて、**ARNの文字列がそのままパスワード扱い**になっていた。
    - → タイプを `valueFrom` に変更（＝ARNの住所から中身を取得して注入）。**同じARNでも `value` か `valueFrom` かで「文字列そのもの」か「中身取得」かが変わる**。

### 3. mysqlクライアント関連

- Amazon Linuxに `sudo dnf install -y mariadb105` でクライアント導入。
- 接続コマンドは `mysql`（MariaDBクライアント）。**MySQLの `--ssl-mode` は使えない**、MariaDBは `--ssl`。
- 証明書検証するなら `curl -o global-bundle.pem https://truststore.pki.rds.amazonaws.com/global/global-bundle.pem` を落として `--ssl-ca=global-bundle.pem`。
- パスワード入力は**画面に表示されない**（正常）。値は**Secrets Managerの `password`** から。

---

## 気付き

- **タスク定義は直せる＝新リビジョンを作るだけ**（全部作り直し不要）。過去リビジョンも残るのでロールバック可＝宣言的・バージョン管理の便利さ。
- **「起動している ≠ 正しく動く」**：ARN文字列でもタスクは起動する。中身が違うだけでDB接続時に失敗。切り分けが命。
- **VPC・SG・ポートを揃える**が全ての土台（ALB/ECS/RDS/EFSすべて同じVPC、必要ポートをSGで許可）。
- マネージドサービスは**便利な自動化が効かない部分がある**（ローカルのmariadbコンテナは`MYSQL_DATABASE`でDB自動作成→RDSは手動）。
- **画像などのファイルは未対応**（コンテナ内=使い捨て）。本番は **EFSをマウント**（`wp-content/uploads`）or S3プラグイン。EFSは`docker -v`のAWS版、SGでNFS(2049)を許可。
