# Docker の基本

- 次: [../ecs/01-overview.md](../ecs/01-overview.md) — コンテナと各サービスの組み合わせ（全体像）
- 関連: [../kubernetes/01-basics.md](../kubernetes/01-basics.md) / [監視（Zabbix）](../../monitoring/zabbix/README.md)

## Docker よく使うコマンド集

> メモ：`<イメージ名>` `<コンテナ名>` `<ホスト側ポート>` `<箱の中ポート>` は自分の環境に置き換えて使う。  
> イメージ＝設計図（型）／コンテナ＝型から作った実物の箱。1つのイメージから箱は何個でも作れる。

| コマンド | 意味 / なぜやるか |
| --- | --- |
| `sudo dnf install -y docker` | Docker本体をインストール（最初に1回）。 |
| `sudo systemctl enable --now docker` | Dockerを今すぐ起動＋自動起動化。常駐していないと箱を操作できない。 |
| `sudo usermod -aG docker $USER` | 自分をdockerグループに追加＝sudoなしで使える。**再ログイン**（`newgrp docker`）で反映。 |
| `docker pull <イメージ名>` | イメージ（設計図）をダウンロードする。`run` でも自動取得される。 |
| `docker pull <イメージ名>:<タグ>` | バージョン指定で取得。実務は `latest` を避けてバージョン固定。 |
| `docker images` | 手元のイメージ一覧を表示（設計図の一覧）。 |
| `docker rmi <イメージ名>` | イメージを削除（remove image）。 |
| `docker build -t <イメージ名> .` | Dockerfileから自分だけのイメージを作る（`-t`＝名前、`.`＝今の場所のDockerfile）。 |
| `docker run <イメージ名>` | イメージから箱を作って起動。 |
| `docker run -d -p <ホスト側>:<箱の中> --name <コンテナ名> <イメージ名>` | 箱を起動。`-d`＝裏で動かす / `-p`＝ポートをつなぐ / `--name`＝名前を付ける。 |
| `docker run -it <イメージ名> bash` | 箱を作ってすぐ中に入る（`-it`＝対話モード、`bash`＝シェル）。 |
| `docker run --rm ...` | 箱が終了したら自動で削除。使い捨ての試し用に便利（ゴミが残らない）。 |
| `docker run -d -e <キー>=<値> --name <コンテナ名> <イメージ名>` | 環境変数を渡して起動。同じイメージでも `-e` の値で振る舞いが変わる。 |
| `docker ps` | 動いている箱の一覧。STATUSが `Up` なら稼働中。 |
| `docker ps -a` | 止まった箱も含めて全部表示（`-a`＝all）。 |
| `docker logs <コンテナ名>` | 箱のログ（記録）を見る。 |
| `docker exec -it <コンテナ名> bash` | 動いている箱の中に入る。抜けるときは `exit`（箱は動いたまま）。 |
| `docker stop <コンテナ名>` | 箱を止める（中身は残る、また起動できる）。まず SIGTERM を送り、既定10秒待っても止まらなければ SIGKILL する。**PID1が信号を無視しても最終的に強制終了されるので確実に止まる**（待たずに止めるなら `docker kill`）。 |
| `docker start <コンテナ名>` | 止めた箱を再び動かす。 |
| `docker restart <コンテナ名>` | 箱を再起動。 |
| `docker rm <コンテナ名>` | 箱を削除（先に `stop` が必要。`-f` で強制削除）。 |
| `docker system prune` | 使っていない箱・イメージ等をまとめて掃除。**消えるので注意**。 |

### 覚え方メモ

- `-d`（detach）＝裏で動かす / `-p`（port）＝つなぐ / `-it`＝中に入って対話 / `-a`（all）＝全部見せて
- `rm`＝箱を消す / `rmi`＝イメージを消す（`i` があるとイメージ側）
- `docker images`＝設計図一覧 ／ `docker ps`＝箱一覧（セットで覚える）

---

## ボリューム（データを箱の外に保存して永続化）

> なぜ：コンテナは削除すると中身も消える。**消えたら困るデータ（DB等）は箱の外（ボリューム）に逃がす**と、箱を消しても残る。  
> 考え方：**箱＝使い捨て／データ＝ボリュームで永続化**。

| コマンド | 意味 / なぜやるか |
| --- | --- |
| `docker volume create <ボリューム名>` | データ置き場（外付けディスク的なもの）を作る。 |
| `docker run -v <ボリューム名>:<箱の中のパス> <イメージ名>` | 箱の中の指定パスを、ボリュームに繋ぐ（＝そこへの書き込みは箱の外に保存）。 |
| `docker volume ls` | ボリューム一覧を表示。 |
| `docker volume rm <ボリューム名>` | ボリュームを削除（データも消えるので注意）。 |

**実験メモ（データが箱を超えて残るのを確認）**

```bash
docker volume create mydata
docker run -d --name db1 -e MYSQL_ROOT_PASSWORD=<PASSWORD> -v mydata:/var/lib/mysql mariadb
# ↑ 準備に十数秒かかる。docker logs db1 に "ready for connections" が出たらOK
docker exec -it db1 mariadb -uroot -p<PASSWORD> -e 'CREATE DATABASE test1;'   # DB作成
docker rm -f db1                                                             # 箱を削除
docker run -d --name db2 -e MYSQL_ROOT_PASSWORD=<PASSWORD> -v mydata:/var/lib/mysql mariadb  # 同じmydataで新しい箱
docker exec -it db2 mariadb -uroot -p<PASSWORD> -e 'SHOW DATABASES;'         # test1 が残っている＝永続化成功
```

**めも**

- MariaDBの接続コマンドは `mysql` → `mariadb` に名前が変わった（`mysql not found` はこれが原因）。
- DBは起動してから受付開始まで**数秒〜十数秒**かかる。直後に繋ぐと `Can't connect ... socket` になる → 少し待つ。
- **1つのボリュームを2つのDBが同時に使うと壊れる**（`Unable to lock ibdata1`）。データ1つにDBサーバは1つ。

---

## ネットワーク（箱同士を名前で通信させる）

> なぜ：実務のアプリは「Web＋DB＋キャッシュ」など複数の箱が連携する。**同じネットワークに入れると箱同士が「名前」で通信できる**（DNS内蔵、IP不要）。  
> ポイント：`-p` は外向けの橋。箱同士の内部通信には `-p` は不要。DBは外に晒さず、内側からだけ名前でアクセスできる＝安全。

| コマンド | 意味 / なぜやるか |
| --- | --- |
| `docker network create <ネットワーク名>` | 箱をつなぐ専用ネットワーク（部屋）を作る。 |
| `docker run --network <ネットワーク名> --name <コンテナ名> <イメージ名>` | 箱をそのネットワークに入れて起動。 |
| `docker network ls` | ネットワーク一覧を表示。 |
| `docker network inspect <ネットワーク名>` | ネットワークの詳細（所属する箱・IPなど）を見る。 |

**実験メモ（名前だけで箱同士がつながるのを確認）**

```bash
docker network create appnet
docker run -d --name api --network appnet nginx           # apiという名前でappnetに入れる（-pなし＝外には出さない）
docker run --rm --network appnet curlimages/curl curl -s http://api
# ↑ 同じappnet内なので http://api（箱の名前）でnginxに届く。IPを調べなくてよい
```

**メモ**

- `api` は `-p` が無いので**外（ブラウザ）からは見えない**が、**同じネットワークの箱からは名前で見える**。「外部公開」と「内部通信」は別物。
- この「名前で繋ぐ」を自動化するのが **Docker Compose**、本番規模でやるのが **ECS / Kubernetes のサービスディスカバリ**。

---

## Docker Compose（EC2環境）で WordPress を立ててみた

> 手作業（`docker run`）で「ネットワーク作成→DB起動→WordPress起動」を毎回打つのは面倒で壊れやすい。**構成を1ファイル**（`docker-compose.yml`）に「宣言」して、コマンド1つで起動・停止できる。

### 手順

**① 作業フォルダとファイルを用意**

```bash
mkdir -p ~/wp-compose && cd ~/wp-compose
echo 'DB_PASSWORD=<DB_PASSWORD>' > .env    # パスワードは.envに外出し（機密はコードに書かない）
```

**②** `docker-compose.yml` を作成

```yaml
services:
  db:
    image: mariadb
    environment:
      MYSQL_ROOT_PASSWORD: ${DB_PASSWORD}   # .envのDB_PASSWORDを参照
      MYSQL_DATABASE: wp                     # wpというDBを自動作成
    volumes:
      - dbdata:/var/lib/mysql               # DBデータを永続化

  wordpress:
    image: wordpress
    ports:
      - "8083:80"                            # ホスト8083 → 箱の中80
    environment:
      WORDPRESS_DB_HOST: db                  # 接続先はサービス名"db"（名前解決）
      WORDPRESS_DB_NAME: wp
      WORDPRESS_DB_USER: root
      WORDPRESS_DB_PASSWORD: ${DB_PASSWORD}
    depends_on:
      - db                                   # dbを先に起動

volumes:
  dbdata:
```

**③ 起動・確認・停止**

```bash
docker compose up -d          # ファイルの通りに全部まとめて起動
docker compose ps             # db と wordpress が running か確認
docker compose logs -f wordpress   # ログ追跡（Ctrl+Cで抜ける、箱は止まらない）
curl -I localhost:8083        # HTTP/1.1 302 Found なら成功（DB接続OK）
docker compose down           # 箱もネットワークもまとめて削除（ボリュームは残る）
```

### `docker run`（手作業）と Compose の対応

| 手作業の `docker run` | Composeの項目 |
| --- | --- |
| イメージ名 | `image:` |
| `-p 8083:80` | `ports:` |
| `-e キー=値` | `environment:` |
| `-v dbdata:/var/lib/mysql` | `volumes:` |
| `--name db` | サービス名（`db:`） |
| 起動順を手で調整 | `depends_on:` |
| `docker network create` | **不要**（Composeが自動作成し、サービス名で名前解決） |

### Compose のコマンド

| コマンド | 意味 |
| --- | --- |
| `docker compose up -d` | 宣言した状態にする（差分だけ反映。何度打ってもOK＝冪等） |
| `docker compose down` | 箱・ネットワークを削除（**ボリュームは残る**） |
| `docker compose down -v` | ボリュームごと削除（**データも消える。要注意**） |
| `docker compose ps` | このプロジェクトの箱の状態 |
| `docker compose logs <サービス名>` | ログを見る |
| `docker compose exec <サービス名> <コマンド>` | 動いてる箱の中でコマンド実行 |

### ハマりどころ・気づきメモ

> **DB ユーザーについて** — ここでは検証のため DB の管理ユーザー（`root`）をそのままアプリに使っている。実運用では、対象の DB にだけ権限を持つ専用ユーザーを作って割り当てる。管理ユーザーを共有すると、アプリが侵害されたときの影響が DB 全体に及ぶ。

- **Composeは別途プラグインが必要**：Amazon Linuxで `dnf install docker` してもComposeは付いてこない。`~/.docker/cli-plugins/` にインストールが必要（`docker compose version` で確認）。`unknown shorthand flag: 'd'` はこれが原因。
- `.env` は「置くだけ」では効かない：ymlで `${DB_PASSWORD}` と**参照**して初めて使われる。ファイル名は `.env` だと自動で読まれる（別名は `--env-file` で指定）。
- **DBパスワードは初回初期化時だけ有効**：ボリュームが残った状態で `.env` のパスワードを変えても、DB側は古いまま → **WordPressが** `Error establishing a database connection`（認証エラー）。変えるなら `down -v` でボリュームをリセットして作り直す。
- `up` は宣言的：`.env` やymlを変えて `up` すると、**違う箇所だけ作り直す**（`Started`/`Recreated`）。変わってなければ何もしない（`Running`）。手作業の `docker run` と違い、同じ名前のコンテナが既にあってもエラーにならず、Compose の管理下のものとして作り直してくれる（ただし**ホストポートが他のプロセスと衝突している場合は** `port is already allocated` **で失敗する**）。
- **「Up」＝「正常」ではない**：認証エラー時も箱・Apacheは `Up` のまま。ブラウザで開いて初めてエラーが分かる。**起動している ≠ ちゃんと使える**。
- **アプリ更新とデータは別**：ymlの `image:` を新バージョンにして `up` するとアプリは更新されるが、**記事などのデータはボリュームに残る**（Composeは触らない）。本番の「無停止でアプリだけ更新」に繋がる。

### 確認できたこと

`http://localhost:8083` をブラウザで開き、WordPress の初期セットアップ画面が表示されることまで確認した。
