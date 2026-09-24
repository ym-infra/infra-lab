# Docker 監視

コンテナごとの状態・リソース使用量を監視する。あわせて、Zabbix の自動検出機能（LLD）の仕組みを整理する。

- 監視対象の追加 … [03-agent.md](./03-agent.md)
- HTTP 監視 … [04-http.md](./04-http.md)
- SSL 証明書の監視 … [06-cert.md](./06-cert.md)
- 画面の操作 … [02-webui.md](./02-webui.md)

## 1. 何を監視するのか

Linux 監視で取得できるのは**ホスト OS 全体の数値**である。コンテナはホストから見ると中身の見えない箱であり、どのコンテナがどれだけリソースを使っているかは Docker の API に問い合わせる必要がある。

``` text
ホストOSの監視    CPU 80%  → 誰が使っているか分からない
Docker監視        wordpress 75% / db 5%  → 特定できる
```

Nagios の `check_procs -C httpd` のようなプロセス名による監視は、コンテナでは機能しない。コンテナは増減し名前も変わるため、固定の監視項目を手で作る運用が成り立たない。

主な取得項目は以下のとおり。

| 単位 | 項目 |
| --- | --- |
| Docker エンジン全体 | コンテナ数（稼働中/停止中）、イメージ数、ディスク使用量、デーモンの応答 |
| コンテナごと | 状態、CPU 使用率、メモリ使用量・上限、ネットワーク I/O、再起動回数、起動時刻 |

### 実務で効くもの

**再起動の繰り返し** — `restart: always` を設定していると異常終了しても自動で起動し直すため、落ちては起動を繰り返す状態が外から見えにくくなる。HTTP 監視では「たまに応答する」ので見逃しかねないが、再起動回数を監視していれば気づける。

**メモリ上限への張り付き** — コンテナにメモリ制限を付けると、超えた瞬間に OOM Killer で停止される。ホストのメモリには余裕があるのに特定のコンテナだけ死ぬ、という障害はホスト監視では捉えられない。

## 2. 権限の設定

`zabbix-agent2` は `/var/run/docker.sock` 経由で Docker API を叩くため、権限が必要になる。

``` bash
sudo usermod -aG docker zabbix
sudo systemctl restart zabbix-agent2
```

グループ追加はプロセスの再起動まで反映されない。`permission denied` が出る場合は再起動が効いていない。

なお `docker` グループへの追加は、`docker run -v /:/host` のような形でホストの全ファイルに到達できるため、**実質的に root 相当の権限を与えることに等しい**。監視用途で許容するかは、対象ホストの位置づけに応じて判断する。

Docker 監視は `zabbix-agent2` に最初から組み込まれている。agent から agent2 へ置き換わった理由のひとつがこれで、追加のプラグイン導入は不要。

## 3. 疎通確認

Zabbix サーバ側で実行する。

``` bash
zabbix_get -s <TARGET_IP> -k docker.info
zabbix_get -s <TARGET_IP> -k docker.containers.discovery
```

`docker.info` は Docker エンジン全体の情報を返す。

``` json
{"Containers":2,"ContainersRunning":2,"ContainersStopped":0,"Images":2,
 "Driver":"overlay2","ServerVersion":"25.0.x","NCPU":2,"MemTotal":4022308864, ...}
```

`docker.containers.discovery` が **LLD の入力**である。

``` json
[{"{#ID}":"e7e7464d...","{#NAME}":"/wp-compose-wordpress-1"},
 {"{#ID}":"04eaedb0...","{#NAME}":"/wp-compose-db-1"}]
```

`{#NAME}` `{#ID}` の `{#...}` 形式が **LLD マクロ**である。

## 4. テンプレートの適用

**Data collection → Hosts** → 対象 → `Templates` 欄に `Docker by Zabbix agent 2` を追加して `Update`。

`Linux by Zabbix agent` は残したまま、2つ並んだ状態にする。テンプレートは複数適用できる。

すでにリンク済みのテンプレートは候補一覧に表示されない。上部に `Unlink` のリンク付きで表示されていれば適用済みである。

---

## 5. LLD（ローレベルディスカバリ）

### なぜ必要か

監視項目には、**事前に数が分からないもの**がある。

| 対象 | 固定できない理由 |
| --- | --- |
| ファイルシステム | サーバごとに `/` だけだったり `/var` `/boot` があったり |
| ネットワークインターフェース | `eth0` `ens5` `docker0` … 環境で違う |
| コンテナ | 増えたり減ったり名前が変わったりする |
| ディスク、CPUコア | 台数・構成で変わる |

手で作ると台数分の作業が発生し、しかもコンテナは日々変わるため、作った端から古くなる。

### 4つの部品

``` text
1. 検出ルール     何を探すか。JSONを返すアイテム
      ↓           例: docker.containers.discovery
2. LLDマクロ      検出結果に付く変数
      ↓           例: {#NAME} = /wp-compose-db-1
3. プロトタイプ   雛形。マクロを埋め込んで書く
      ↓           例: docker.container_stats[{#NAME}]
4. 実体           検出された数だけ生成される
```

テンプレートにはアイテムが直接書かれているのではなく、**アイテムプロトタイプ**（雛形）が入っている。

``` text
プロトタイプ   docker.container_stats[{#NAME}]
                        ↓ 検出結果を当てはめる
実体1          docker.container_stats[/wp-compose-wordpress-1]
実体2          docker.container_stats[/wp-compose-db-1]
```

プロトタイプはアイテムだけでなく、**トリガー・グラフ・ホストにも存在する**。「見つかったコンテナごとに、使用率の項目とグラフと、停止時に発火するトリガーを作る」がまとめて実現される。

### Cacti の Data Query との違い

Cacti で `Get Mounted Partitions` を追加すると、グラフ候補にパーティションが出てくる。これが Data Query であり、同じ発想にあたる。

違いは自動化の度合いである。Cacti は候補を出すところまでで、グラフを作るのは手動だった。LLD は**生成まで自動**で、**定期的に再実行して増減に追従する**。

### 検出ルールの設定項目

| 項目 | 本環境の値 | 意味 |
| --- | --- | --- |
| Key | `docker.containers.discovery[false]` | `false` = 稼働中のコンテナのみ検出。`true` で停止中も含む |
| Update interval | `15m` | 検出を実行する間隔 |
| Disable lost resources | `Immediately` | 消えたら即座に収集を停止 |
| Delete lost resources | `After 7d` | 7日後に削除 |
| Filters | 2件 | 検出結果の絞り込み条件 |

**停止と削除が分かれている**のがポイントである。コンテナが一時的に消えただけの可能性があるため、いきなり履歴を捨てない設計になっている。再起動中のコンテナの履歴が毎回消えては困る。

Docker の検出間隔が `15m` なのに対し、Linux 側の `Block devices discovery` や `Network interface discovery` は `1h` である。コンテナは増減が早いため、追従を速くしてある。

### 手動で実行する

待たずに実行する場合は `Execute now` を使う。

**Data collection → Hosts** → 対象 → **Discovery rules**

一覧画面から実行する場合は、**チェックボックスを選択して初めて画面下部のボタンが有効になる**。チェックを入れる前はグレーアウトしていて押せない。

検出ルールを開いた詳細画面の下部にも `Execute now` がある。

## 6. 結果

`Item prototypes` が 42 のため、コンテナ2つで **42 × 2 = 84項目**が生成された。

``` text
container: /wp-compose-db-1         42
container: /wp-compose-wordpress-1  42
```

生成された項目の例。

``` text
Container /wp-compose-db-1: CPU percent usage
Container /wp-compose-db-1: Current PIDs count
Container /wp-compose-db-1: Dead
Container /wp-compose-db-1: Exit code
Container /wp-compose-db-1: Health failing streak
```

タグも自動で付与される。

| タグ | 用途 |
| --- | --- |
| `container: /wp-compose-db-1` | そのコンテナの項目だけを抽出 |
| `component: cpu` / `memory` / `network` / `system` | 種別ごとに絞り込み |

### Latest data での確認に注意

**Monitoring → Latest data** で Name に `docker` と入力しても1件しか表示されない。

公式テンプレートのアイテム名は `Container /wp-compose-db-1: CPU percent usage` のような形式で、**`docker` という文字列を含まないものが大半**である。コンテナ名（`wp-compose` など）で絞り込むか、Name を空にして一覧するほうが確実である。

生成されたかどうかの確認は、**Data collection → Hosts** の `Items` の数字を見るのが最も早い。

---

## 監視のレイヤ

同じ障害でも、監視の層によって見える内容が異なる。

| 障害 | Linux 監視 | HTTP 監視 | Docker 監視 |
| --- | --- | --- | --- |
| DB コンテナの停止 | 変化なし | WordPress が応答しない | **db コンテナが停止** |

外形監視は「WordPress が応答しない」としか言えないが、Docker 監視は原因まで指し示す。層を重ねることで切り分けが速くなる。

## 未実施

- コンテナ停止時のトリガー確認（Docker テンプレートに `Trigger prototypes` として標準で含まれる）
- MariaDB 監視、ログ監視
