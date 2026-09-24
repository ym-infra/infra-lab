# Zabbix Web UI

Zabbixでは、設定ファイルはデーモンの動作を設定し、
実際の監視内容はWeb UIから設定する。

## 1. 設定ファイルとWeb UIの分担

### zabbix_server.conf

Zabbix Server自体の動作を設定する。

主な設定内容：

- DB接続先
- 待受ポート
- プロセス数
- ログ出力先

### zabbix_agent2.conf

Zabbix Agent側の動作を設定する。

主な設定内容：

- 通信するZabbix Server
- 自身のホスト名

### Web UI

監視に関する設定を行う。

- ホスト
- アイテム
- トリガー
- アクション
- 通知
- ユーザー
- 権限

監視対象を追加する場合、基本的にZabbix Server側の
設定ファイルを変更する必要はなく、Web UIから設定する。


## 2. Web UIのメニュー

### Monitoring

現在の監視状態を確認する。

主に以下を確認できる。

- Problems
- Latest data
- Hosts
- Graphs

### Data collection

監視設定を行う。

- Hosts
- Templates
- Items
- Triggers
- Maintenance

### Alerts

通知に関する設定を行う。

- Media types
- Actions

### Reports

可用性などのレポートを確認する。

### Users

ユーザーや権限を設定する。

### Administration

Zabbix全体のシステム設定を行う。

Housekeepingなどの設定もここから行う。


## 3. 監視状態の確認

### Problems

`Monitoring → Problems`

現在発生している障害を確認する。


### Latest data

`Monitoring → Latest data`

各ホストから取得した実測値を確認する。

ホストやアイテム名でフィルタできる。

`Execute now` を実行すると、
次の監視間隔を待たずにデータを取得できる。

設定変更後の確認などで利用できる。


### Graphs

`Monitoring → Hosts → 対象ホスト → Graphs`

テンプレートに含まれているグラフを確認できる。


## 4. ホストの追加

`Data collection → Hosts → Create host`

主に以下を設定する。

- Host name
- Templates
- Host groups
- Interfaces

Agentを利用する場合、
Interfaceには監視対象サーバのIPアドレスとポート10050を設定する。

Host nameはAgent側の`Hostname`と一致させる。


## 5. テンプレート

テンプレートをホストに適用することで、
複数の監視設定をまとめて追加できる。

テンプレートには主に以下が含まれる。

- Items
- Triggers
- Graphs

今回の検証では、

`Linux by Zabbix agent`

を利用した。


## 6. アイテム

アイテムは、Zabbixが収集するデータを定義する。

例：

- CPU使用率
- メモリ使用量
- ディスク使用量
- Load Average

収集された値は、

`Monitoring → Latest data`

から確認できる。


## 7. トリガー

トリガーは、アイテムで取得した値をもとに
障害と判断する条件を設定する。

テンプレート由来のトリガーは、
ホスト側から直接変更するのではなく、
テンプレート側の設定が利用される。


## 8. マクロ

ホストごとにしきい値を変更したい場合は、
Macrosを利用する。

`Data collection → Hosts → 対象ホスト → Macros`

テンプレートでは、しきい値が以下のような
マクロで設定されている。

`{$CPU.UTIL.CRIT}`

ホスト側に同名のマクロを設定することで、
そのホストだけ値を変更できる。

これにより、テンプレート自体を変更せずに
ホスト単位でしきい値を調整できる。


## 9. Maintenance

`Data collection → Maintenance`

計画作業などで一時的に通知を止めたい場合に利用する。

メンテナンス期間を登録することで、
作業中の不要なアラート通知を抑制できる。


## 10. 通知

通知には主に以下の3つの設定が必要になる。

### Media types

`Alerts → Media types`

通知方法を設定する。

例：

- Email
- Slack
- Webhook

### User Media

`Users → 対象ユーザー → Media`

通知先を設定する。

### Actions

`Alerts → Actions`

どのような条件で通知するかを設定する。

通知が飛ばない場合は、
Media typesが有効になっているか確認する。


## 11. 監視設定の流れ

Zabbixの監視設定は、以下の流れで構成される。

ホスト  
↓  
テンプレート  
↓  
アイテム  
↓  
トリガー  
↓  
アクション

- ホスト：監視対象
- テンプレート：監視設定のまとまり
- アイテム：値を取得
- トリガー：取得した値から障害を判定
- アクション：条件に応じて通知などを実行

この流れを理解しておくと、
Web UI上でどの設定を変更すればよいか判断しやすくなる。
