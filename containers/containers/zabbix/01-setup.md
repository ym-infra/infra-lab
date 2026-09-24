# Zabbix 構築（AlmaLinux 9 / Zabbix 7.0 LTS）

本ページでは、監視サーバ Zabbix を**コンテナを用いず手動で**構築した手順を整理する。

コンテナ（Docker Compose）による構築は別途行う予定であり、本ページはその前段にあたる。手動で組むと Zabbix が Server・Web（PHP）・DB・Agent の4部品で構成されていることが設定ファイル単位で把握でき、コンテナ版がそれを1ファイルに畳んだものだと理解しやすくなる。

## 構成

| 項目 | 内容 |
| --- | --- |
| OS | AlmaLinux 9.8（EC2 / t3.medium / 30GB） |
| Zabbix | 7.0 LTS（7.0.30） |
| データベース | MariaDB 10.5 |
| Web サーバ | nginx 1.20 + PHP-FPM（PHP 8.0） |

バージョンは 7.0 LTS を選定した。7.4 は標準リリースであり次期 LTS が出るまでの短期サポートとなるため、実運用を想定して LTS を採用している。

**OS は AlmaLinux 9 を用いる。Amazon Linux 2023 では構築できなかった。**
その理由と経緯は [99-troubleshooting.md](./99-troubleshooting.md) に記載する。

### 構成要素

| 役割 | 実体 | パッケージ |
| --- | --- | --- |
| 本体（収集・判定） | Zabbix Server | `zabbix-server-mysql` |
| データ保存 | MySQL / MariaDB / PostgreSQL | `mariadb-server` |
| Web サーバ | nginx または Apache | `zabbix-nginx-conf` |
| 画面（フロントエンド） | PHP | `zabbix-web-mysql` + `php-fpm` |
| 監視対象側 | Zabbix Agent | `zabbix-agent2` |

``` text
ブラウザ → nginx + PHP（画面）─┐
                               ├→ DB（設定も履歴もすべてここ）
Zabbix Server（収集・判定）────┘
       ↓
Zabbix Agent（監視対象）
```

Web 画面と Zabbix Server は直接接続しておらず、双方が DB を参照する構成である。画面で設定した内容は DB に書き込まれ、Zabbix Server がそれを読み取って動作する。

---

## 1. 構築手順

### 1-1. MariaDB の導入

``` bash
sudo dnf install -y mariadb-server
sudo systemctl enable --now mariadb
systemctl is-active mariadb
```

Amazon Linux 2023 では `mariadb105-server` だが、AlmaLinux 9 では `mariadb-server` である（中身は同じ 10.5 系）。

### 1-2. DB root の認証方式を決める

MariaDB 10.4 以降の RHEL 系パッケージでは、`root@localhost` の認証方式が既定で `unix_socket` になっている。パスワード照合ではなく、UNIX ドメインソケット経由で接続してきた**OSユーザーが誰かをカーネルに問い合わせて確認する**方式である。

``` bash
sudo mysql -uroot -e "SELECT user, host, plugin FROM mysql.user WHERE user='root';"
```

`plugin` 列に `unix_socket` と表示される。この状態では `sudo mysql -uroot` がパスワードなしで通る。設定漏れではなく、**漏洩する対象が存在しない分だけ安全**という設計である。

どちらの方式を採るかで以降のコマンドが変わるため、ここで決めておく。

#### 手順A：`unix_socket` のまま運用する

追加作業は不要である。以降、DB 操作は `sudo mysql -uroot` で行う。

- 利点：パスワードの保管・ローテーションが不要。誤ってコマンド履歴や設定ファイルに残す事故が起きない
- 性質：OS の root を取得されれば DB の root も同時に取得される。ただし同一ホストに DB が載る構成では元々そうであり、リスクは増加しない

#### 手順B：従来どおり root にパスワードを設定する

複数人で運用する場合や、組織のセキュリティ基準で root パスワードの設定が求められる場合はこちらを用いる。初期セキュリティ設定をまとめて実施できる `mysql_secure_installation` を使う。

``` bash
sudo mysql_secure_installation
```

対話では、root パスワードの設定、匿名ユーザーの削除、root のリモートログイン禁止、テスト用 DB の削除、権限テーブルの再読み込みを行う。

設定後は、DB 操作を以下のように読み替える。

| unix_socket | パスワード認証 |
| --- | --- |
| `sudo mysql -uroot` | `mysql -uroot -p` |
| `sudo mysql -uroot -e "..."` | `mysql -uroot -p -e "..."` |

確認：

``` bash
mysql -uroot -p -e "SELECT user, host, plugin FROM mysql.user WHERE user='root';"
```

`plugin` 列が `mysql_native_password` になっていれば切り替え完了である。

### 1-3. データベースとユーザーの作成

> **注意:** 以下の `<ZABBIX_DB_PASSWORD>` は実際のパスワードへ置き換える。GitHub には実値を記載しない。

``` sql
CREATE DATABASE zabbix CHARACTER SET utf8mb4 COLLATE utf8mb4_bin;
CREATE USER 'zabbix'@'localhost' IDENTIFIED BY '<ZABBIX_DB_PASSWORD>';
GRANT ALL PRIVILEGES ON zabbix.* TO 'zabbix'@'localhost';
SET GLOBAL log_bin_trust_function_creators = 1;
```

| 設定 | 理由 |
| --- | --- |
| `utf8mb4` / `utf8mb4_bin` | Zabbix の要件。照合順序が異なるとスキーマ投入時にエラーとなる |
| `log_bin_trust_function_creators = 1` | スキーマ投入時にストアドファンクションを作成するために一時的に必要。投入後に 0 へ戻す |

`zabbix` ユーザーには、root の認証方式にかかわらず**必ずパスワードを設定する**。Zabbix サーバのプロセスや PHP は OS ユーザー名と無関係に接続してくるため、`unix_socket` 方式では認証できないからである。

確認：

``` bash
sudo mysql -uroot -e "SHOW DATABASES;"
```

### 1-4. Zabbix リポジトリの登録

``` bash
sudo rpm -Uvh https://repo.zabbix.com/zabbix/7.0/rhel/9/x86_64/zabbix-release-latest-7.0.el9.noarch.rpm
sudo dnf clean all
sudo dnf repolist | grep -i zabbix
```

`zabbix` と `zabbix-non-supported` の2つが表示されれば成功である。後者には `fping` など RHEL 標準リポジトリに存在しない補助パッケージが含まれており、サーバの依存解決に用いられる。

導入時の `NOKEY` 警告は正常であり、GPG 鍵は初回の `dnf install` 時に取り込まれる。

### 1-5. パッケージの導入

``` bash
sudo dnf install -y \
  zabbix-server-mysql \
  zabbix-web-mysql \
  zabbix-nginx-conf \
  zabbix-sql-scripts \
  zabbix-selinux-policy \
  zabbix-agent2
```

`zabbix-selinux-policy` は、SELinux が Enforcing の環境で必要となるルール一式を提供するパッケージである。これを含めておくことで、後段での手作業による SELinux 調整がほぼ不要になる。

インストール前に依存関係が解決できるかだけを確認したい場合は `--assumeno` を付ける。

``` bash
sudo dnf install --assumeno zabbix-server-mysql zabbix-sql-scripts zabbix-agent2
```

### 1-6. スキーマ投入

``` bash
zcat /usr/share/zabbix-sql-scripts/mysql/server.sql.gz \
  | mysql --default-character-set=utf8mb4 -uzabbix -p zabbix
```

パスワード入力を求められたら、Zabbix DB ユーザーのパスワードを入力する。

30秒〜1分程度、出力なしで経過する。完了後、一時設定を戻す。

``` bash
sudo mysql -uroot -e "SET GLOBAL log_bin_trust_function_creators = 0;"
mysql -uzabbix -p zabbix -e "SHOW TABLES;" | wc -l
```

本構築では 204（ヘッダ行を含むため実質 203 テーブル）であった。

### 1-7. Zabbix Server の設定と起動

`/etc/zabbix/zabbix_server.conf` に DB 接続情報を設定する。

``` text
DBName=zabbix
DBUser=zabbix
DBPassword=<ZABBIX_DB_PASSWORD>
```

`zabbix_server.conf` には DB パスワードが平文で記載されるため、パーミッションを確認しておく（既定で `640` / `root:zabbix`）。

``` bash
ls -l /etc/zabbix/zabbix_server.conf

sudo systemctl enable --now zabbix-server
sudo tail -20 /var/log/zabbix/zabbix_server.log
```

`current database version` が出力されていれば、スキーマ投入も正しく完了している。

### 1-8. フロントエンド（nginx + PHP-FPM）

nginx のリッスン設定を有効化する。既定ではコメントアウトされており、ポートも 8080 になっている。

``` bash
sudo sed -i \
  -e 's|^#\s*listen\s*8080;|        listen          80;|' \
  -e 's|^#\s*server_name\s*example.com;|        server_name     _;|' \
  /etc/nginx/conf.d/zabbix.conf
```

PHP のタイムゾーンを設定する。本環境では `/etc/php-fpm.d/zabbix.conf` に `date.timezone` の行が存在しなかったため、PHP 本体側に設定した。

``` bash
sudo sed -i 's|^;date.timezone =.*|date.timezone = Asia/Tokyo|' /etc/php.ini
```

なお `/etc/php-fpm.d/zabbix.conf` の `listen.acl_users = apache,nginx` は既定で設定済みであり、変更不要である。これは nginx が PHP-FPM のソケットへアクセスするための許可設定である。

``` bash
sudo nginx -t

sudo systemctl enable --now nginx php-fpm
sudo systemctl enable --now zabbix-agent2
sudo systemctl restart zabbix-server nginx php-fpm

systemctl is-active zabbix-server zabbix-agent2 nginx php-fpm
curl -sI -4 http://127.0.0.1/
```

最後の `curl` が `302 Found` と `Location: setup.php` を返せば、初期セットアップ待ちの状態で正常に動作している。

### 1-9. Web UI での初期設定

ブラウザで以下へアクセスするとセットアップウィザードが起動する。

``` text
http://<ZABBIX_SERVER_IP>/
```

| 画面 | 内容 |
| --- | --- |
| Welcome | 表示言語の選択 |
| Check of pre-requisites | PHP のバージョン・拡張・設定値の適合確認。全項目 OK であること |
| Configure DB connection | DB 接続情報を設定 |
| Settings | サーバ名、Default time zone（Asia/Tokyo） |
| Pre-installation summary | 設定内容を確認 |
| Install | `/etc/zabbix/web/zabbix.conf.php` が生成される |

DB 接続画面で入力するのは **DB の root ではなく `zabbix` ユーザー**である。root にパスワードを設定していても、この画面には関係しない。

初回ログイン後は、Web UI の初期パスワードを必ず変更する。

### 1-10. 自ホストの監視を有効化する

Zabbix は初期状態で `Zabbix server` という自ホスト用のホストを1件持っている。**Status が Disabled になっている場合があるため、有効化しないとデータを収集しない。**

まずエージェント側を確認する。

``` bash
systemctl is-active zabbix-agent2
sudo ss -tlnp | grep 10050
```

Web UI を開く前に、サーバ → エージェントの取得経路をコマンドで確認しておくと切り分けが楽になる。

``` bash
sudo dnf install -y zabbix-get
zabbix_get -s 127.0.0.1 -k agent.ping
zabbix_get -s 127.0.0.1 -k system.cpu.load[all,avg1]
```

`1` とロードアベレージの数値が返れば、通信経路は問題ない。以降で値が入らない場合、原因はネットワークやエージェントではなく Web UI の設定側だと切り分けられる。

Web UI の **Data collection → Hosts** で `Zabbix server` を開き、次を確認する。

- **Status** が `Enabled` になっているか
- **Templates** に `Linux by Zabbix agent` が入っているか
- **Interface** が `127.0.0.1:10050` になっているか

**Monitoring → Hosts** で **ZBX** アイコンが緑になれば疎通できている。

本構築では、テンプレートを1つ適用しただけで **アイテム153件・グラフ16件・ダッシュボード4件** が一度に作成された。

---

## 補足

本ページは Zabbix Server の手動構築を中心にまとめている。

以下は別ページに分けて整理する。

- Web UI の基本操作
- ホスト・テンプレート・アイテム・トリガー・アクション
- 通知設定
- トラブルシューティング
- Amazon Linux 2023 で構築できなかった件
- nginx / IPv4 / IPv6 の切り分け
- コンテナを利用した Zabbix 構築・監視
