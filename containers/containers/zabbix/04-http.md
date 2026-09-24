# HTTP 監視（Web シナリオ）

Zabbix サーバ自身が HTTP を投げて、Web サービスの応答を監視する。いわゆる外形監視。

- 監視対象の追加 … [03-agent.md](./03-agent.md)
- SSL 証明書の監視 … [06-cert.md](./06-cert.md)
- 画面の操作 … [02-webui.md](./02-webui.md)
- つまずいた点 … [99-troubleshooting.md](./99-troubleshooting.md)

## 1. 監視対象の準備

監視対象に Docker と Compose を導入し、WordPress + MariaDB を起動した。

Amazon Linux では `dnf install docker` に Compose が付属しないため、手動で導入する。

``` bash
sudo dnf install -y docker
sudo systemctl enable --now docker
sudo usermod -aG docker $USER
```

グループ追加はログアウト・再ログインまで反映されない。

``` bash
sudo mkdir -p /usr/local/lib/docker/cli-plugins
sudo curl -SL https://github.com/docker/compose/releases/latest/download/docker-compose-linux-x86_64 \
  -o /usr/local/lib/docker/cli-plugins/docker-compose
sudo chmod +x /usr/local/lib/docker/cli-plugins/docker-compose
docker compose version
```

`cli-plugins` に置くと `docker compose`（ハイフンなし）のサブコマンドとして認識される。古い `docker-compose`（ハイフンあり）は別物で、現在は非推奨。

`compose.yml`：

``` yaml
services:
  db:
    image: mariadb:11
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: ${DB_ROOT_PASSWORD}
      MYSQL_DATABASE: wordpress
      MYSQL_USER: wordpress
      MYSQL_PASSWORD: ${DB_PASSWORD}
    volumes:
      - dbdata:/var/lib/mysql

  wordpress:
    image: wordpress:latest
    depends_on:
      - db
    restart: always
    ports:
      - "80:80"
    environment:
      WORDPRESS_DB_HOST: db:3306
      WORDPRESS_DB_USER: wordpress
      WORDPRESS_DB_PASSWORD: ${DB_PASSWORD}
      WORDPRESS_DB_NAME: wordpress
    volumes:
      - wpdata:/var/www/html

volumes:
  dbdata:
  wpdata:
```

パスワードは `.env` に外出しする。ポートを 80 で公開しているのは、HTTP 監視の設定を素直にするため。

``` bash
docker compose up -d
docker compose ps
curl -I localhost
```

`302 Found` が返れば起動成功（WordPress のインストール画面へのリダイレクト）。

Zabbix サーバ側からも届くことを先に確認しておく。ここが通らないと、監視を設定しても失敗する。

``` bash
curl -I http://<TARGET_IP>/
```

## 2. Web シナリオの作成

Zabbix の外形監視は **Web シナリオ**という仕組みを使う。アイテムを個別に作るのではなく、「このURLをこの順で叩く」というシナリオを定義すると、複数のアイテムが自動生成される。

**Data collection → Hosts** → 対象ホストの行の右端 `Web` → **Create web scenario**

| 項目 | 値 |
| --- | --- |
| Name | `WordPress HTTP` |
| Update interval | `1m` |
| Attempts | `3` |

`Steps` タブ → `Add`

| 項目 | 値 |
| --- | --- |
| Name | `Home page` |
| URL | `http://<TARGET_IP>/` |
| Follow redirects | チェックを入れる |
| Required status codes | `200` |

リダイレクトを追う設定にしているため、最終的に `install.php` が返す `200` を見ることになる。

`Add` が2箇所にあって紛らわしい。**ステップ表の中の `Add` はステップを追加するリンク、画面下部の青い `Add` がシナリオの保存ボタン**である。

## 3. 通常のアイテムとの違い

### エージェントを経由しない

CPU やメモリは監視対象のエージェントに聞いて値をもらうが、Web シナリオは **Zabbix サーバ自身が HTTP を投げて**、返ってきたものを見る。

```
通常のアイテム   Zabbixサーバ →(10050)→ エージェント → OSに問い合わせ
Webシナリオ      Zabbixサーバ →(80)→ Webサーバ        ※エージェント不要
```

このため「プロセスは動いていてエージェントも応答するが、Web サーバが応答を返さない」という障害を検知できる。エージェント経由の監視では気づけない領域である。

エージェントが不要なので、自社管理外のサービス（他社 API、SaaS）の死活監視にも使える。

### 複数ステップとセッション維持

ステップを順に実行し、**Cookie が引き継がれる**。

``` text
Step 1  GET  /wp-login.php          Required string: Log In
Step 2  POST /wp-login.php          Post fields: log=..., pwd=...
Step 3  GET  /wp-admin/             Required string: Dashboard
```

「サイトが表示される」ではなく「**ログインして管理画面まで到達できる**」を監視できる。`Required string` は応答本文を見るため、「200 は返るが中身がエラーページ」も検知できる。

Nagios の `check_http` は単発のリクエストであり、この点が大きく異なる。

認証情報をシナリオに平文で書くのは避け、ホストの `Macros` タブで **Secret text 型のマクロ**に入れて参照する。Secret text は画面上で伏せ字になり、API 経由でも読み出せない。

### ステップごとに指定できる項目

| 項目 | 内容 |
| --- | --- |
| URL | 叩き先 |
| Post fields | フォームに送る値（name / value） |
| Variables | シナリオ内で使い回す変数 |
| Headers | 任意のHTTPヘッダ |
| Required string | 応答本文に含まれているべき文字列（正規表現可） |
| Required status codes | 期待する応答コード |
| Timeout | この手順の制限時間 |

### 他の方式との使い分け

| 方式 | 叩く側 | 分かること |
| --- | --- | --- |
| `net.tcp.port[,80]`（エージェント） | 対象自身 | ポートが開いているか |
| Simple check `net.tcp.service[http]` | Zabbixサーバ | 外からポートに繋がるか |
| **Web シナリオ** | Zabbixサーバ | 応答コード・時間・本文の内容・一連の操作 |
| `http_agent` アイテム | Zabbixサーバ | 単発リクエスト。JSON を取って解析するのに向く |

下に行くほど分かることが増えるが、その分設定も重くなる。

## 4. 自動生成されるアイテム

| アイテム | 内容 |
| --- | --- |
| `web.test.fail[シナリオ名]` | 失敗したステップ番号。正常なら 0 |
| `web.test.error[シナリオ名]` | エラーメッセージ |
| `web.test.rspcode[シナリオ名,ステップ名]` | 応答コード |
| `web.test.time[シナリオ名,ステップ名,resp]` | 応答時間 |
| `web.test.in[シナリオ名,ステップ名,bps]` | ダウンロード速度 |

結果は **Monitoring → Hosts** → 対象ホスト → `Web` で確認する。

**`Monitoring → Web` というメニューは 6.0 で廃止されている**（5.x までは存在した）。ホスト名をクリックして出るメニューからも `Web` を選べる。

## 5. トリガーの作成

**アイテムは値を集めるだけで、異常の判定はしない。** トリガーを作らないと Problems には上がらない。

``` text
アイテム   値を集めるだけ
トリガー   集めた値を判定する
アクション 判定結果を通知する
```

実際、WordPress コンテナを停止した状態で Web シナリオは失敗を記録していたが、トリガーが無いため Problems には何も表示されなかった。実務でも「監視しているはずなのにアラートが来ない」の原因として多いパターンである。どの段で止まっているかを切り分ける。

**Data collection → Hosts** → 対象 → **Triggers** → **Create trigger**

| 項目 | 値 |
| --- | --- |
| Name | `WordPress: Web scenario "WordPress HTTP" failed` |
| Severity | `High` |
| Expression | `last(/<ホスト名>/web.test.fail[WordPress HTTP])>0` |

書式は `関数(/ホスト名/アイテムキー)`。

- `last(...)` … 最新の値を取る
- `web.test.fail[...]` … 何ステップ目で失敗したかを返すアイテム
- `>0` … 0 より大きい＝どこかで失敗している

`web.test.fail` を使うと、**ステップが増えても式を変更せずに済む**。

トリガー名は Zabbix 標準の `<対象>: <事象>` という書式に揃えている（例：`Linux: Zabbix agent is not available`）。

## 6. 検知テスト

``` bash
docker compose stop wordpress
```

Web シナリオの Status に以下が表示された。

``` text
Step "Home page" [1 of 1] failed: Failed to connect to <TARGET_IP> port 80: Connection refused
```

**`Connection refused` と `timed out` は意味が異なる。**

| エラー | 意味 | 疑う場所 |
| --- | --- | --- |
| Connection refused | パケットは届いたが、誰も待ち受けていない | サービスが停止している |
| timed out | パケットが破棄されて応答がない | セキュリティグループ、firewalld |

復旧させると、トリガーは自動的に解消される。

``` bash
docker compose start wordpress
```

Zabbix はトリガーの式を評価し直し、条件を満たさなくなった時点で解決済みとして扱う。Nagios のように回数（`max_check_attempts`）で確定させる方式とは異なり、**式の評価結果がそのまま状態になる**。

## 未実施

- WordPress のインストールを完了させたうえでの、ログイン監視（複数ステップのシナリオ）
- HTTPS 化後のシナリオの張り替え（現在は 80 番を直接叩いている）

SSL 証明書の期限監視は [06-cert.md](./06-cert.md) に分けて記録した。
