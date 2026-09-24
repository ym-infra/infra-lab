# EFS に画像データを保管する

- 全体像 … [01-overview.md](./01-overview.md)
- ECS を動かす … [02-basics.md](./02-basics.md)
- ECS と RDS で WordPress を構築する … [03-rds-wordpress.md](./03-rds-wordpress.md)
- インメモリキャッシュ（Valkey/Redis） … [05-elasticache.md](./05-elasticache.md)
- Docker の基本 … [../docker/01-basics.md](../docker/01-basics.md)

## なぜEFSが必要か（課題）

WordPressのデータは2種類あり、保存先が違う。

| データ | 保存先 | 課題 |
| --- | --- | --- |
| 記事・設定・ユーザー | RDS（DB） | ○ 対応済み（永続化されている） |
| **画像・アップロードファイル** | **コンテナ内 `wp-content/uploads`** | × **コンテナは使い捨て → 再起動で消える** |

- Fargateのコンテナは**ephemeral（使い捨て）**。タスクが再起動・再デプロイされると、中のファイルは消える。
- さらに**タスクを複数**に増やすと、それぞれ別のファイルシステムを持つため「タスクAの画像がタスクBに無い」というバラつきも起きる。
- → **画像も「箱の外」の永続ストレージに逃がす**必要がある。それが **EFS**（or S3）。

---

## 考え方（イメージ・コンテナ・ストレージの関係）

```text
① 設計段階（Dockerfile の VOLUME 命令）
    「このフォルダは外部化する場所だよ」と"印"だけ付ける
    ※WordPress公式イメージは VOLUME /var/www/html を宣言済み
    ※明示的に繋がないと、Dockerが匿名ボリュームを勝手に作る（あのハッシュ名の正体）

② 接続先を決める段階 ← 環境ごとに違う
    素Docker → docker run の -v
    Compose  → compose.yml
    ECS      → タスク定義の volume + mountPoint   ★ここで決める

③ 結果
    そのパス（uploads）への読み書きが、コンテナ内ではなく EFS に向かう
```

**ポイント**：**イメージ自身はストレージを持たない/指さない**。イメージは「アプリ＋"ここは外部化する場所"の印」まで。**具体的な接続先（どのEFSか）はタスク定義で決める**。だから同じイメージのまま、繋ぎ先だけ差し替えできる（＝「箱は使い捨て、データは外」）。

---

## 全体構成（完成形）

```text
[ECS Fargate：WordPress（使い捨て）]
   ├─ 記事・設定 → RDS（3306）           ← 対応済み
   └─ 画像       → EFS（NFS 2049でマウント） ← これ
```

EFS＝**ネットワーク越しの共有ディスク**。`docker -v`（ボリューム）のAWSマネージド版。用語：**NFS（Network File System：ネットワーク越しにファイル共有する仕組み。ポート2049）**、**マウント（外部ディスクをコンテナの特定パスに繋ぐこと）**。

---

## 手順

**① EFSファイルシステムを作成**

- EFSコンソール → 作成 → **VPC は ECS タスクと同じものを選ぶ**（本記録では検証用に既定の VPC を使用。実運用では専用 VPC のプライベートサブネットに置く）
- ファイルシステムID（`<EFS_ID>`）をメモ

**② EFSのセキュリティグループで NFS(2049) を許可**

- EFSのマウントターゲットのSG → インバウンドに **NFS / 2049 / ソース＝ECSタスクのSG**
- ※RDSの3306と同じ発想。これが無いとマウントできずタスク起動失敗

**③ （推奨）アクセスポイントを作成**

- パス `/uploads`、**POSIXユーザーを UID/GID 33（www-data）** に設定
- ※WordPressは `www-data`(UID33) で動く。所有者を合わせないと**書き込み権限エラー**でアップロード失敗

**④ タスク定義に volume + mountPoint を追加**（イメージは触らない・新リビジョン）

```json
"volumes": [
  {
    "name": "wp-uploads",
    "efsVolumeConfiguration": {
      "fileSystemId": "<EFS_ID>",
      "transitEncryption": "ENABLED",
      "authorizationConfig": { "accessPointId": "<EFS_ACCESS_POINT_ID>", "iam": "DISABLED" }  // 検証用。実運用ではタスクロールに ClientMount/ClientWrite を付けて ENABLED にする
    }
  }
],
"containerDefinitions": [
  {
    "name": "wordpress",
    "mountPoints": [
      { "sourceVolume": "wp-uploads", "containerPath": "/var/www/html/wp-content/uploads" }
    ]
  }
]
```

**⑤ サービスを新リビジョンに更新**（新デプロイ強制）

- 新タスクが uploads を EFS にマウントして起動 → 画像がタスクを超えて残る＆複数タスクで共有

---

## ハマりどころ

- **権限エラー**：アクセスポイントで UID/GID を 33 にしないと uploads に書けない（「繋がったのに動かない」系）。
- **SGの2049許可漏れ**：無いとマウント失敗＝タスクが起動しない。
- **VPCを揃える**：EFSもECSと同じVPC。
- **Fargateのプラットフォームバージョン**：EFS対応は 1.4.0 以降（`LATEST`ならOK）。

---

## ローカル（Docker）との対応

| ローカル（Docker） | ECS + EFS |
| --- | --- |
| `docker volume create mydata` | EFSファイルシステム作成 |
| `-v mydata:/var/lib/mysql` | タスク定義の volume + mountPoint |
| （なし） | SGで NFS(2049) を許可（RDSの3306と同じ） |
| 箱を消してもデータ残る | 画像がタスクを超えて残る＋複数タスクで共有 |

**結論**：DBはRDS、ファイルはEFS。**コンテナには永続データを一切持たせない = ステートレスなコンテナ**。これが「箱は使い捨て、データは外」の最終形。

**エビデンス**：Amazon ECS「Amazon EFS ボリューム」 <https://docs.aws.amazon.com/ja_jp/AmazonECS/latest/developerguide/efs-volumes.html>
