# Zabbix トラブルシューティング

構築手順は [Zabbix 構築（AlmaLinux 9 / Zabbix 7.0 LTS）](./setup.md)、Web UI の操作は [Zabbix Web UI](./webui.md) を参照。

## 1. Amazon Linux 2023 では Zabbix Server を構築できない

### 症状

Zabbix 公式リポジトリを登録したうえでインストールを試みると、依存関係が解決できず失敗する。

``` text
nothing provides liblber.so.2(OPENLDAP_2.200)(64bit) needed by zabbix-server-mysql-7.0.30-release1.el9.x86_64
nothing provides libldap.so.2(OPENLDAP_2.200)(64bit) needed by zabbix-server-mysql-7.0.30-release1.el9.x86_64
nothing provides libOpenIPMI.so.0()(64bit) needed by zabbix-server-mysql-7.0.30-release1.el9.x86_64
nothing provides libOpenIPMIposix.so.0()(64bit) needed by zabbix-server-mysql-7.0.30-release1.el9.x86_64
```

### 原因

不足しているのは PHP ではなく、より低位の C 共有ライブラリ2系統である。

| 不足ライブラリ | 用途 | AL2023 での状況 |
| --- | --- | --- |
| `liblber.so.2` / `libldap.so.2`（`OPENLDAP_2.200`） | LDAP 認証 | 同梱されているのは OpenLDAP 2.4 系。シンボルのバージョンが一致しない |
| `libOpenIPMI.so.0` / `libOpenIPMIposix.so.0` | IPMI 監視 | リポジトリにパッケージが存在しない |

Zabbix の el9 パッケージは **RHEL 9 のライブラリ構成を前提にビルドされている**。Amazon Linux 2023 は CentOS 9 Stream 等から構成部品を取り込んでいるため「RHEL 9 系寄り」ではあるが、同一ではなく、この差が埋まらない。

この件は AWS の公式リポジトリに Issue が起票されている（`amazonlinux/amazon-linux-2023` #752、2024年7月起票）。本手順の実施時点でも未解決であり、待機しても解消しない。

`--skip-broken` は有効ではない。壊れているのが `zabbix-server-mysql` 本体であるため、スキップすると何も導入されない。

### 注意点

**Zabbix エージェントは AL2023 でも導入できる。** エージェントは上記ライブラリに依存しないためである。

Web 上には「Amazon Linux 2023 に Zabbix を導入した」旨の記事が複数存在するが、**対象がエージェントのみである場合がある**。サーバ構築の可否を判断する材料としては、記事の対象範囲を確認する必要がある。

### 対応

RHEL 9 完全互換かつ Zabbix 公式サポート対象である **AlmaLinux 9** を選定し、別インスタンスで構築した。Rocky Linux 9 でも同じ手順が通る。

結果として監視サーバと監視対象が別ホストになり、実務構成に近い形になった。エージェント〜サーバ間の通信設計を扱えるという副次的な利点も得られた。

### 事前確認の方法

インストールせずに依存関係の解決可否だけを確認できる。OS を決める前にこれを実行しておくと、無駄な作業を避けられる。

``` bash
sudo dnf install --assumeno zabbix-server-mysql zabbix-sql-scripts zabbix-agent2
```

---

## 2. 画面が Zabbix ではなく nginx の初期ページになる

### 症状

フロントエンドを起動したあと、`curl http://localhost/` が Zabbix ではなく OS のテストページを返す。

``` text
HTTP/1.1 200 OK
Server: nginx/1.20.1
Content-Type: text/html
Content-Length: 5760
Last-Modified: Mon, 24 Mar 2025 16:15:24 GMT
ETag: "67e1851c-1680"
```

`nginx -t` の時点で警告も出ている。

``` text
nginx: [warn] conflicting server name "_" on 0.0.0.0:80, ignored
```

### 原因

**IPv6 である。**

`/etc/nginx/nginx.conf` の既定サーバ定義は IPv4 と IPv6 の両方を待ち受けている。

``` text
listen       80;
listen       [::]:80;
```

一方、Zabbix 側（`/etc/nginx/conf.d/zabbix.conf`）は `listen 80;` のみで IPv4 しか待ち受けていない。

`localhost` は IPv6（`::1`）を先に解決するため、IPv6 で来たリクエストを受けられる既定サーバ側が応答していた。

### 確認

IPv4 を明示すると結果が変わる。

``` bash
curl -sI -4 http://127.0.0.1/
```

``` text
HTTP/1.1 302 Found
X-Powered-By: PHP/8.0.30
Location: setup.php
```

Zabbix は正常に動作しており、初期セットアップ待ちの 302 を返していた。パブリック IP は IPv4 であるため、外部からのアクセスには影響しない。

### 対処

紛らわしいため、既定サーバ定義を無効化する。行番号は環境により異なるため、先に確認してから実行する。

``` bash
awk 'NR>=36 && NR<=58 {print NR": "$0}' /etc/nginx/nginx.conf

sudo cp /etc/nginx/nginx.conf /etc/nginx/nginx.conf.bak
sudo sed -i '38,54s/^/#/' /etc/nginx/nginx.conf
sudo nginx -t && sudo systemctl reload nginx
```

失敗した場合は `sudo cp /etc/nginx/nginx.conf.bak /etc/nginx/nginx.conf` で戻せる。

### 切り分けの要点

応答ヘッダに `Last-Modified` と `ETag` が付いていれば**静的ファイルの応答**であり、PHP が処理した結果ではない。この2つの有無で、どちらのサーバ定義が応答しているかを判別できる。

---

## 3. firewall-cmd が存在しない

### 症状

ファイアウォールを開放しようとすると、コマンド自体が見つからない。

``` text
sudo: firewall-cmd: command not found
```

### 原因と対応

AlmaLinux の AWS 用イメージには firewalld が導入されていない。この場合、OS 側のファイアウォールは存在せず、通信可否はセキュリティグループのみで決まる。

AMI によって差があるため、**開放作業の前に `firewall-cmd` の有無を確認する**のが確実である。導入されている場合は以下を実施する。

``` bash
sudo firewall-cmd --add-service=http --permanent
sudo firewall-cmd --add-port=10051/tcp --permanent
sudo firewall-cmd --reload
```

80 は Web UI、10051 はエージェントがアクティブチェックでデータを送る受け口である。

---

## 4. セットアップウィザードで日本語が選択できない

### 症状

言語選択のプルダウンで、日本語がグレーアウトして選べない。

``` text
You are not able to choose some of the languages, because locales for them are not installed on the web server.
```

### 原因と対応

OS に日本語ロケールが導入されていないため。ウィザードを進める前に導入しておくと、最初から選択できる。

``` bash
sudo dnf install -y glibc-langpack-ja
sudo systemctl restart php-fpm
```

---

## 5. ホストを登録したのに値が入らない

### 症状

`Data collection → Hosts` にホストは存在し、Status も `Enabled` だが、`Monitoring → Latest data` に何も表示されない。ZBX アイコンも灰色のまま。

### 原因と対応

**テンプレートが適用されていない**可能性が高い。Host groups は必須項目のため未入力だとエラーで止まるが、Templates は任意項目なので、未設定のまま登録が通ってしまう。

一覧画面で判別できる。

| 状態 | テンプレートあり | テンプレートなし |
| --- | --- | --- |
| Tags | `class: os` などが自動で付く | 空 |
| Graphs / Dashboards | 件数付きのリンク | 灰色 |
| ZBX | 緑 | 灰色のまま |

ホスト名をクリックして設定画面を開き、`Templates` 欄に `Linux by Zabbix agent` などを入力して `Update` する。保存した時点でアイテム・トリガー・グラフがコピーされ、1〜2分で ZBX が緑になる。

### 通信経路の切り分け

Web UI を触る前に、サーバ → エージェントの経路だけをコマンドで確認できる。

``` bash
sudo dnf install -y zabbix-get
zabbix_get -s <監視対象のIP> -k agent.ping
zabbix_get -s <監視対象のIP> -k agent.hostname
```

`1` とホスト名が返れば、ファイアウォールやエージェントの問題ではない。原因は Web UI の設定側に絞られる。

`zabbix_get` は Zabbix サーバと同じ動きで 10050 に問い合わせるコマンドである。そのため**パッシブチェックの確認にしか使えない**。アクティブチェック（エージェントからサーバへ 10051）の確認には使えない点に注意する。

なお `system.hostname` は OS のホスト名を返すため、設定ファイルの `Hostname=` を確認したい場合は `agent.hostname` を用いる。Web UI に登録するホスト名と一致させる必要があるのは後者である。

---

## 6. Web UI のパスワードを忘れた

Zabbix 7.0 のパスワードは bcrypt でハッシュ化されている。ハッシュを生成してから DB を直接更新する。

``` bash
php -r 'echo password_hash("<NEW_PASSWORD>", PASSWORD_BCRYPT), PHP_EOL;'
```

出力された `$2y$10$...` を使って更新する。

``` sql
UPDATE users SET passwd='<生成したハッシュ>' WHERE username='Admin';
```

ハッシュには `$` が含まれるため、シェル経由で流す場合は変数展開されないよう注意する。

---

## 切り分けで使えた手法

| 手法 | 用途 |
| --- | --- |
| `dnf install --assumeno` | インストールせずに依存関係の解決可否だけを確認する |
| `zabbix_get -s <IP> -k agent.ping` | Web UI を介さず、サーバ → エージェントの取得経路だけを確認する |
| 応答ヘッダの `Last-Modified` / `ETag` | 付いていれば静的ファイル、無ければ PHP が処理した結果 |
| `curl -4` / `curl -6` | IPv4 と IPv6 のどちらで問題が起きているかを切り分ける |
| `ss -tlnp \| grep <ポート>` | 待ち受けているプロセスとポートを確認する |
