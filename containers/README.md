# コンテナ / Kubernetes 学習記録

AWS インフラエンジニアが、コンテナ未経験の状態から Docker・ECS・Kubernetes・監視までを手を動かして学んだ記録。

方針は「動いた」で終わらせないこと。各段階で**正常系を確認したあと、必ず壊して挙動を見る**ようにしている。そのため各ファイルには、設定手順だけでなく「なぜこれが必要なのか」「これが無いと何が困るのか」「実際に壊したらどうなったか」を残している。

## 学習の順番と、その理由

コンテナ技術は「下の層の課題を、上の層が解決する」積み重ねになっている。そのため下から順に、前の技術では何が困るのかを体感しながら進めた。

```text
Docker            アプリを「どこでも同じに動く箱」に閉じ込める
  ↓  箱が増えると手動起動が限界に
Docker Compose    複数の箱を1ファイルでまとめて起動
  ↓  1台では本番運用できない
  ├─ ECS (Fargate)   AWS に任せる道。サーバを意識せずコンテナを動かす
  └─ Kubernetes      業界標準の道。宣言した状態を保ち続ける仕組みを原理から
       ↓
     EKS / NKP      同じ K8s を「誰が運用するか」の違い
  ↓
Zabbix            全レイヤを横断して「壊れたら気づく」仕組み
```

監視を最後に置いているのは、監視対象を先に理解していないと、そもそも何を監視すべきか設計できないため。

## 目次

### [docker/](./docker/) — 土台

| ファイル | 内容 |
| --- | --- |
| [01-basics.md](./docker/01-basics.md) | イメージとコンテナの違い、ライフサイクル、Volume、環境変数、ネットワーク、よく使うコマンド |

### [ecs/](./ecs/) — AWS に寄せる道

| ファイル | 内容 |
| --- | --- |
| [01-overview.md](./ecs/01-overview.md) | 「外出し」の考え方。永続データ（RDS / S3 / EFS）とキャッシュ（Redis / CDN）の使い分け |
| [02-basics.md](./ecs/02-basics.md) | ECR への push からタスク定義、クラスター、サービスまで。ECS の各要素が Compose の何に対応するか |
| [03-rds-wordpress.md](./ecs/03-rds-wordpress.md) | ECS + RDS で WordPress を構築。Secrets Manager からの認証情報の注入 |
| [04-efs.md](./ecs/04-efs.md) | 画像データの永続化。コンテナが使い捨てであることの実際の影響 |
| [05-elasticache.md](./ecs/05-elasticache.md) | インメモリキャッシュ。「正本」と「写し」の役割の違い |

### [kubernetes/](./kubernetes/) — 標準技術の道

各リソースを「それが無ければ何が困るか」から順に積み上げている。

| ファイル | 内容 |
| --- | --- |
| [01-basics.md](./kubernetes/01-basics.md) | Kubernetes が解決する課題、Pod、クラスタの構成 |
| [02-replicaset-deployment.md](./kubernetes/02-replicaset-deployment.md) | Pod は自己修復しない → ReplicaSet。更新が煩雑 → Deployment |
| [03-service-configmap-secret.md](./kubernetes/03-service-configmap-secret.md) | Pod の IP は変わる → Service。設定と機密の外出し |
| [04-persistent-volume.md](./kubernetes/04-persistent-volume.md) | PV / PVC。Docker の Volume、ECS の EFS マウントに相当する仕組み |
| [05-ingress.md](./kubernetes/05-ingress.md) | L7 のルーティング。ECS の ALB に相当 |
| [06-rolling-update-auto-healing.md](./kubernetes/06-rolling-update-auto-healing.md) | 無停止更新とロールバック、自己修復、Compose から K8s への移行 |

### 監視（Zabbix）

コンテナに限らない監視の話なので、**別フォルダに分けている**。Docker 監視・SSL 証明書監視・Compose 構成に対する外形監視など、このフォルダの内容と地続きの部分が多い。

| ファイル | 内容 |
| --- | --- |
| [01-setup.md](../monitoring/zabbix/01-setup.md) | Zabbix サーバの構築。コンテナを使わず手動で組み、Server / Web(PHP) / DB / Agent の4部品を設定ファイル単位で把握する |
| [02-webui.md](../monitoring/zabbix/02-webui.md) | 設定ファイルと Web UI の分担、画面の基本操作 |
| [03-agent.md](../monitoring/zabbix/03-agent.md) | 監視対象の追加。パッシブとアクティブの違い |
| [04-http.md](../monitoring/zabbix/04-http.md) | HTTP 監視（Web シナリオ）とトリガー |
| [05-docker.md](../monitoring/zabbix/05-docker.md) | Docker 監視と LLD（ローレベルディスカバリ） |
| [06-cert.md](../monitoring/zabbix/06-cert.md) | SSL 証明書監視。前段での TLS 終端 |
| [99-troubleshooting.md](../monitoring/zabbix/99-troubleshooting.md) | つまずいた点と、切り分けに使った手法 |

## 一貫して出てくる考え方

学ぶ層が変わっても、同じ考え方が別の名前で繰り返し現れる。

| 考え方 | Docker | Compose | ECS | Kubernetes |
| --- | --- | --- | --- | --- |
| あるべき状態を宣言する | Dockerfile | compose.yml | タスク定義 | マニフェスト |
| 台数を保つ・自己修復 | なし | なし | Desired Count | replicas / Deployment |
| 設定の外出し | 環境変数 | `.env` | 環境変数 | ConfigMap |
| 機密の外出し | 環境変数 | Secret | Secrets Manager | Secret |
| データの永続化 | Volume | 名前付き Volume | EFS / RDS | PV / PVC |
| 外部への入口 | ポート公開 | ports | ALB | Service / Ingress |

コンテナは使い捨てであり、消えては困るものはすべて箱の外に出す。これがどの層でも共通している。

## 未着手

- EKS（マネージド K8s）、NKP（Nutanix 基盤での K8s）
- Zabbix の MariaDB 監視・ログ監視・通知設定
- 障害注入の通し検証（CPU 高負荷、ディスク逼迫、証明書期限切れ、K8s 異常）

## 注記

- **実際の IP アドレス・ホスト名・パスワード・AWS アカウント ID は記載していない。** `<SERVER_IP>` `<DB_PASSWORD>` `<AWS_ACCOUNT_ID>` などのプレースホルダに置き換えている。
- いずれも個人の学習用検証環境での記録であり、本番環境向けの設定ではない。自己署名証明書の使用、最小構成の DB、`.env` の扱いなど、本番では見直すべき箇所を含む。
