# インメモリキャッシュ（Valkey/Redis）

- 全体像 … [01-overview.md](./01-overview.md)
- ECS を動かす … [02-basics.md](./02-basics.md)
- ECS と RDS で WordPress を構築する … [03-rds-wordpress.md](./03-rds-wordpress.md)
- EFS に画像データを保管する … [04-efs.md](./04-efs.md)
- Docker の基本 … [../docker/01-basics.md](../docker/01-basics.md)

## インメモリキャッシュ（Redis/Valkey）とは

- **一般名**：インメモリキャッシュ（＝メモリ上で動くキャッシュ）。中身の技術は **Redis / Memcached / Valkey**。
- **AWSのサービス名**：**ElastiCache**（Redis/Memcached/Valkey を運用代行するマネージドサービス）。
- **RDSとの対比**：RDS＝MySQL等のマネージド版／**ElastiCache＝Redis等のマネージド版**。どちらも「定番OSS技術をAWSが運用代行」。

> Redis＝**メモリ(RAM)上のkey-value保管庫**。超高速だが**揮発性**（プロセス/箱が死ぬと消える）。

---

## キャッシュの基本概念（7つ）

1. **Redis＝メモリ上のkey-value保管庫**。速いが揮発性。
2. **キャッシュ＝"写し"で高速化**。**本物(正本)はRDS**。Redisが消えてもRDSから作り直せる。
3. **キーで範囲を分ける**：全員共通（`product:999`）／ユーザー別（`user:123:cart`）／セッション別（`session:abc`）。
4. **TTL（有効期限）で保管期間を決める**：付ければ自動失効、無ければ消すまで／メモリ満杯時の挙動は追い出しポリシー次第（Redis 素の既定は `noeviction` で書き込みがエラーになる。ElastiCache の既定は `volatile-lru` で TTL 付きのものから追い出す）／箱が死ねば全消え。
5. **速さ vs 鮮度のトレードオフ**：何を・どのくらいの期間キャッシュするかを設計。
6. **層で使い分け**：CDN（エッジ/静的）／Redis（アプリ/動的）／RDS（正本）。
7. **アプリがコードで"使いにいく"／インフラが"用意する"**（役割の境界）。

---

## redis-cli で触った例（EC2でDocker起動）

```bash
docker run -d --name redis -p 6379:6379 redis
docker exec -it redis redis-cli --raw     # --raw で日本語も読める形で表示
```

```text
SET greeting "こんにちは"          # 保存
GET greeting                       # 取り出す
INCR hits                          # カウンタ +1（helloappのアクセス数の中身）

SET user:123:cart "商品A,商品B"    # ユーザー別キャッシュ
SET user:456:cart "商品C"          # キーが違えば別物
GET user:123:cart

SET session:abc "ログイン中" EX 10 # TTL 10秒
TTL session:abc                    # 残り秒数（-1=無期限, -2=消えた）
# 10秒後 → GET session:abc は (nil)＝自動失効
```

- **箱を消す（`docker rm -f redis`）と全データ消える** → メモリ上＝揮発性の証拠。だからキャッシュは"消えても平気"なデータ向き。

---

## キャッシュ設計の勘所

**何を・どの範囲で・どのくらいの鮮度でキャッシュするか**を、TTLとキーで設計する。

| データ | キャッシュ範囲 | TTL目安 | 備考 |
| --- | --- | --- | --- |
| 商品データ（名前・説明） | 全員共通 | 数分〜数十分 | あまり変わらない |
| 在庫数（表示） | 全員共通 | 数秒〜十数秒 | 変わりやすい。ざっくり表示（在庫あり/残りわずか/なし）も有効 |
| カート・お気に入り | ユーザー別 | セッション越えで保持 | `user:123:cart` |
| ログイン状態 | セッション別 | 一定時間で失効 | TTLで自動ログアウト |

**在庫の注意**：**表示**はキャッシュOK（短TTL）だが、**購入時の在庫減算は必ずRDSで正確に**（キャッシュを信じると売り越し）。「見せるのは速く、買うのは正確に」。

---

## CDN と Redis の違い（別の層・組み合わせて使う）

| 観点 | CDN | Redis |
| --- | --- | --- |
| キャッシュする場所 | エッジ（ユーザーの近く） | アプリの内側 |
| 対象 | 出来上がったページ・**画像**（HTTP出力） | DBの**データ**・クエリ結果 |
| 得意 | **全員に同じもの**（静的・共通） | **ユーザーごと**に違うもの（動的） |
| サーバの関与 | **飛ばせる**（サーバ抜きで返す） | アプリは動く（データ取得が速くなる） |

- **同じURL（`/api/cart`）でも中身が人ごとに違う** → CDNは出し分け不可、**RedisはキーにユーザーIDを入れて出し分け可**。
- 実際のECは **CDN（静的）＋ Redis（動的）＋ RDS（正本）** を層で重ねる。

---

## ストレージ全体マップ（データの種類ごとに保管先が違う）

| データの種類 | 保管場所（本物 or 写し） | AWSサービス名 |
| --- | --- | --- |
| 構造化データ（記事・注文・ユーザー） | 正本・永続 | **RDS** |
| ファイル（画像・動画・PDF） | 正本・永続 | **S3**（or **EFS**） |
| データのキャッシュ | 速い写し・揮発 | **ElastiCache(Redis)** |
| ファイルのキャッシュ（エッジ配信） | 速い写し・エッジ | **CloudFront(CDN)** |

- **画像の本物はS3（またはEFS）**。**CDNはその写しをキャッシュ**するだけ。
- **DB(RDS)には画像そのものは入れず、"画像の場所(URL/パス)"だけ**を持つ（例：`s3://bucket/images/999.jpg`）。

---

## 一般名 ↔ 各社サービス名（マルチクラウド提案用）

**"役割(一般名)"で設計を考えると、AWS以外でも応用が効く**。

| 役割（一般名） | AWS | Azure | 自前/オンプレ |
| --- | --- | --- | --- |
| リレーショナルDB | RDS | Azure SQL Database | MySQL/PostgreSQL |
| オブジェクトストレージ | S3 | Blob Storage | MinIO・Ceph RGW |
| 共有ファイルシステム(NFS) | EFS | Azure Files | NFSサーバ |
| インメモリキャッシュ | ElastiCache | Azure Cache for Redis | Redis/Memcached |
| CDN | CloudFront | Azure CDN / Front Door | Cloudflare等 |

> 「S3 を使う」ではなく「オブジェクトストレージ（AWS なら S3、Azure なら Blob）」と役割名で捉えると、クラウドが変わっても設計を再利用できる。

---

## エビデンス（参考）

- Amazon ElastiCache：<https://docs.aws.amazon.com/ja_jp/AmazonElastiCache/latest/dg/WhatIs.html>
- Amazon CloudFront（CDN）：<https://docs.aws.amazon.com/ja_jp/AmazonCloudFront/latest/DeveloperGuide/Introduction.html>
- Amazon S3：<https://docs.aws.amazon.com/ja_jp/AmazonS3/latest/userguide/Welcome.html>
