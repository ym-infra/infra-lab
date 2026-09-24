# 監視対象の追加（エージェント導入・ホスト登録）

Zabbix サーバに対して、**別ホストを監視対象として追加**する手順。

- サーバの構築 … [setup.md](./setup.md)
- 画面の操作 … [webui.md](./webui.md)
- つまずいた点 … [troubleshooting.md](./troubleshooting.md)

設定した監視の種類は以下を参照。

- [HTTP 監視（Web シナリオ）](./http.md)
- [Docker 監視](./docker.md)

## 構成

| 役割 | OS | 用途 |
| --- | --- | --- |
| Zabbix サーバ | AlmaLinux 9 | 監視する側 |
| 監視対象 | Amazon Linux 2023 | 監視される側。Docker で WordPress + MariaDB が稼働 |

監視サーバと監視対象を分けている。同一ホストで兼ねると、障害注入の演習で Web サーバや DB を停止した際に Zabbix の画面自体が落ちてしまい、検知結果を確認できなくなるため。

セキュリティグループは以下のとおり設定した。ソースに IP ではなくセキュリティグループを指定すると、IP 変更時に追随が不要になる。

| 対象 | 方向 | ポート | ソース | 用途 |
| --- | --- | --- | --- | --- |
| Zabbix サーバ | In | 10051 | 監視対象の SG | エージェントからの能動送信（アクティブ） |
| 監視対象 | In | 10050 | Zabbix サーバの SG | サーバからの取得（パッシブ） |
| 監視対象 | In | 80 | Zabbix サーバの SG | HTTP 監視 |

---

## 1. エージェントの導入

監視対象側で実行する。Amazon Linux 2023 でも **エージェントは導入できる**（サーバは導入できない。理由は [troubleshooting.md](./troubleshooting.md) を参照）。

``` bash
sudo rpm -Uvh https://repo.zabbix.com/zabbix/7.0/rhel/9/x86_64/zabbix-release-latest-7.0.el9.noarch.rpm
sudo dnf clean all
sudo dnf install -y zabbix-agent2
```

## 2. 設定ファイルの編集

``` bash
sudo cp /etc/zabbix/zabbix_agent2.conf /etc/zabbix/zabbix_agent2.conf.$(date +%Y%m%d)
sudo vi /etc/zabbix/zabbix_agent2.conf
```

3箇所を変更する。いずれも既定値が入った状態で存在しているため、新規追加ではなく値の変更となる。

| 既定値 | 変更後 |
| --- | --- |
| `Server=127.0.0.1` | `Server=<ZABBIX_SERVER_IP>` |
| `ServerActive=127.0.0.1` | `ServerActive=<ZABBIX_SERVER_IP>` |
| `Hostname=Zabbix server` | `Hostname=<任意のホスト名>` |

vi 内では行頭を指定して検索すると早い（`/^Server=` など）。`Server` だけで検索すると `ServerActive` やコメント行も引っかかる。

### Server と ServerActive の違い

**同じ IP を書くが、意味が異なる。**

| | `Server` | `ServerActive` |
| --- | --- | --- |
| 方式 | パッシブ | アクティブ |
| 接続する側 | Zabbix サーバ | エージェント |
| 使うポート | 10050 | 10051 |
| 設定値の意味 | **許可リスト**（このIPからの問い合わせに答える） | **接続先**（ここへ送りに行く） |

```
パッシブ   Zabbixサーバ ──(10050)──→ エージェント
           「CPU使用率を教えて」→「45%です」

アクティブ Zabbixサーバ ←──(10051)── エージェント
           「監視する項目をください」→ 値をまとめて送信
```

どちらを使うかはアイテムのタイプで決まる。

| アイテムのタイプ | 方式 |
| --- | --- |
| `Zabbix agent` | パッシブ |
| `Zabbix agent (active)` | アクティブ |

テンプレートも `Linux by Zabbix agent` がパッシブ版、`Linux by Zabbix agent active` がアクティブ版に分かれている。**両方貼ると同じ項目が二重に収集される**ため、通常はどちらか一方を使う。

**ログ監視（`log[]`）はアクティブでのみ動作する。** どこまで読んだかをエージェント側が保持する必要があるため。

アクティブが有利なのは、NAT やファイアウォールの内側にある監視対象を扱う場合と、台数が多い場合である。前者はエージェントが外に出ていくだけで済み、後者は Zabbix サーバが多数のホストへ接続しに行く負荷を避けられる。

両方書いておけば、テンプレート側がどちらを使っても動く。**片方しか書かないと、そのテンプレートを貼ったときに片側の項目だけ値が入らない**という分かりにくい状態になる。

### 起動

``` bash
grep -E '^(Server|ServerActive|Hostname)=' /etc/zabbix/zabbix_agent2.conf
sudo systemctl enable --now zabbix-agent2
systemctl is-active zabbix-agent2
sudo ss -tlnp | grep 10050
```

## 3. 疎通確認

Web UI を触る前に、コマンドで経路だけを確認しておく。Zabbix サーバ側で実行する。

``` bash
sudo dnf install -y zabbix-get
zabbix_get -s <TARGET_IP> -k agent.ping
zabbix_get -s <TARGET_IP> -k agent.hostname
```

`1` とホスト名が返れば、セキュリティグループを含めて経路が通っている。ここが通っていれば、以降で値が入らない場合の原因は Web UI の設定側に絞られる。

- `system.hostname` は **OS のホスト名**を返す
- `agent.hostname` は **設定ファイルの `Hostname=` の値**を返す

Web UI に登録するホスト名と一致させる必要があるのは後者である。

`zabbix_get` は Zabbix サーバと同じ動きで 10050 に問い合わせるコマンドのため、**パッシブチェックの確認にしか使えない**。

## 4. ホストの登録

**Data collection → Hosts → Create host**

| 項目 | 値 |
| --- | --- |
| Host name | エージェントの `Hostname=` と同じ値 |
| Templates | `Linux by Zabbix agent` |
| Host groups | `Linux servers` |
| Interfaces | `Add → Agent` → 監視対象の IP / Port `10050` |

### テンプレートの付け忘れに注意

**Host groups は必須項目のため未入力だとエラーで止まるが、Templates は任意項目なので黙って登録が通る。** その結果「ホストは作れたのに値が入らない」となる。

一覧画面で判別できる。

| 状態 | テンプレートあり | テンプレートなし |
| --- | --- | --- |
| Tags | `class: os` などが自動で付く | 空 |
| Graphs / Dashboards | 件数付きのリンク | 灰色 |
| ZBX | 緑 | 灰色のまま |

後から追加する場合は、ホスト名をクリックして `Templates` 欄に入力し `Update` する。保存した時点でアイテム・トリガー・グラフがコピーされ、1〜2分で ZBX が緑になる。

### 確認

**Monitoring → Hosts** で ZBX アイコンの色を見る。

| 色 | 状態 |
| --- | --- |
| 緑 | 疎通できている |
| 赤 | エージェントに到達していない。マウスオーバーでエラー内容が表示される |
| 灰 | 判定前。1分程度で色がつく |

本環境では、`Linux by Zabbix agent` を1つ適用しただけで **50項目**が生成された。その後ローレベルディスカバリが動き、ファイルシステムとネットワークインターフェースを検出して **77項目**まで増加した。LLD の詳細は [docker.md](./docker.md) を参照。

## 5. 導入直後に上がる通知

監視開始後、以下の Warning が上がることがある。

``` text
Linux: Number of installed packages has been changed
```

Docker やエージェントを導入したことによる、**正しい検知**である。タグに `scope: notice` が付いており、「壊れた」ではなく「変化したので知らせる」種類のトリガーである。

| scope | 意味 |
| --- | --- |
| `availability` | 落ちている |
| `performance` | 遅い・重い |
| `capacity` | 容量が足りない |
| `security` | セキュリティ上の問題 |
| `notice` | 状態が変わった、という通知 |

変更管理の文脈で有効である。誰も作業していないサーバでパッケージが増えていれば、侵入か申請外の作業を疑える。同系統に `/etc/passwd has been changed` がある。

このトリガーは条件が元に戻らないため**自動では解消しない**。`Update` から `Close problem` で手動クローズする。コメントを残せるので、作業の記録として使える。

同様に、インスタンスを起動した直後は `Linux: ... has been restarted (uptime < 10m)` が上がる。こちらは10分経過すると自動的に解消される。

``` text
last(/<ホスト名>/system.uptime)<10m
```

知らないうちに再起動していた事実を検知するための標準トリガーであり、異常ではない。
