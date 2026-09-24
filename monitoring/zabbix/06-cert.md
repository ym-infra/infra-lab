# SSL 証明書の監視

HTTPS の証明書の有効期限と検証結果を監視する。あわせて、コンテナ構成における証明書の置き場所を整理する。

- 監視対象の追加 … [03-agent.md](./03-agent.md)
- HTTP 監視 … [04-http.md](./04-http.md)
- Docker 監視 … [05-docker.md](./05-docker.md)
- つまずいた点 … [99-troubleshooting.md](./99-troubleshooting.md)

## 1. なぜ監視するのか

証明書の期限切れは、**予告なく起きる障害ではなく、日付が分かっている障害**である。それにもかかわらず落ちるのは、更新を忘れるからである。CPU やディスクと違って前兆がなく、期限を過ぎた瞬間にブラウザが全面警告を出してサービスが使えなくなる。

監視で見るのは主に次の3点。

| 項目 | 内容 |
| --- | --- |
| 残り日数 | 期限までの日数。閾値を切ったら通知 |
| 検証結果 | 信頼できる証明書として成立しているか |
| 内容の変化 | 発行者や有効期限が変わっていないか（差し替えの検知） |

## 2. 証明書をどこに置くか

**結論：アプリのコンテナではなく、前段で TLS を終端させる。**

当初の構成は WordPress コンテナが 80 番で直接受けていた。ここに証明書を入れようとすると、公式イメージの中身を書き換えることになり、イメージを更新するたびに作業が消える。

そこで `nginx:alpine` のコンテナを前段に立て、そこだけが 443 を受け、後ろの WordPress へは HTTP で渡す構成にした。

``` text
変更前  クライアント ──(80)──→ wordpress

変更後  クライアント ──(443/TLS)──→ proxy ──(80)──→ wordpress
                                    ↑
                              ここだけが証明書を持つ
```

この形は実務の構成と同じである。

| 環境 | TLS を終端する場所 |
| --- | --- |
| 本環境 | nginx コンテナ（proxy） |
| AWS | ALB |
| Kubernetes | Ingress |

いずれも「**前段が証明書を持ち、後ろは平文で受ける**」という考え方であり、アプリ側は HTTPS を意識しない。証明書の更新も1箇所で済む。

証明書の実体はホスト OS 側に置き、コンテナには読み取り専用でマウントする。

| 種類 | 置き場所 | 権限 |
| --- | --- | --- |
| 証明書 | `/etc/pki/tls/certs/` | 644 |
| 秘密鍵 | `/etc/pki/tls/private/` | **600** |

コンテナの中に焼き込まないのは、イメージを作り直すと消えることと、イメージに秘密鍵が含まれてしまうためである。

## 3. 証明書の作成

自己署名証明書を作成する。検証の挙動を見るために、期限を **30日**と短くしてある。

``` bash
sudo openssl req -x509 -nodes -days 30 -newkey rsa:2048 \
  -keyout /etc/pki/tls/private/selfsigned.key \
  -out /etc/pki/tls/certs/selfsigned.crt \
  -subj "/C=JP/ST=Tokyo/O=Study/CN=<ホスト名>" \
  -addext "subjectAltName=DNS:<ホスト名>,DNS:localhost,IP:127.0.0.1"
sudo chmod 600 /etc/pki/tls/private/selfsigned.key
```

| オプション | 意味 |
| --- | --- |
| `-x509` | 署名要求（CSR）ではなく証明書そのものを出力する＝自己署名 |
| `-nodes` | 秘密鍵にパスフレーズを付けない。付けると nginx 起動時に入力を求められる |
| `-days 30` | 有効期限 |
| `-subj` | 対話入力を省略して一括指定 |
| `-addext subjectAltName` | **SAN。必須** |

### SAN を必ず入れる

`CN` にホスト名を書いただけでは、**現在のクライアントはホスト名の一致を認めない**。`-addext "subjectAltName=..."` が必要である。

省略すると、検証時に次のエラーになる。

``` text
x509: certificate relies on legacy Common Name field, use SANs instead
```

CN によるホスト名検証は Go 1.15 で廃止されており、`zabbix-agent2` は Go 製のため影響を受ける。ブラウザも同様に拒否する。**CN は表示用の名前であり、ホスト名の照合には使われない**と考えるのが正しい。

IP アドレスで接続する場合は `IP:` 形式で、名前で接続する場合は `DNS:` 形式で、**実際に接続に使う値を列挙する**。

## 4. proxy の設定

`proxy.conf`：

``` nginx
server {
    listen 443 ssl;
    server_name _;
    ssl_certificate     /etc/nginx/certs/server.crt;
    ssl_certificate_key /etc/nginx/certs/server.key;

    location / {
        proxy_pass http://wordpress:80;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-Proto https;
    }
}
```

`compose.yml` に追加するサービス：

``` yaml
  proxy:
    image: nginx:alpine
    restart: always
    depends_on:
      - wordpress
    ports:
      - "443:443"
    volumes:
      - /etc/pki/tls/certs/selfsigned.crt:/etc/nginx/certs/server.crt:ro
      - /etc/pki/tls/private/selfsigned.key:/etc/nginx/certs/server.key:ro
      - ./proxy.conf:/etc/nginx/conf.d/default.conf:ro
```

`:ro` を付けて読み取り専用でマウントする。コンテナ側から書き換えられないようにするため。

`proxy_pass http://wordpress:80` の `wordpress` は Compose のサービス名である。Compose が作るネットワーク内では**サービス名が名前解決できる**ため、IP を書く必要がない。

### X-Forwarded-Proto が必要な理由

**これが無いとリダイレクトループになる。**

WordPress は自身が HTTP で受けたと判断すると、設定された URL（`https://...`）へリダイレクトを返す。proxy から見ると後ろは HTTP なので、WordPress は毎回リダイレクトを返し続ける。

``` text
ヘッダなし   client →(443) proxy →(80) wordpress
                         ←「httpsへ行け」← ずっと繰り返す

ヘッダあり   proxy が「元はhttpsだった」と伝える
                         → wordpress は通常の応答を返す
```

ALB や Ingress でも同じヘッダが使われる。**TLS を終端すると、後段は「元の接続がどちらだったか」を知る手段がなくなる**ため、ヘッダで明示的に渡す必要がある。

### 起動と確認

``` bash
docker compose up -d
curl -kI https://localhost/
openssl s_client -connect localhost:443 -servername <ホスト名> </dev/null 2>/dev/null \
  | openssl x509 -noout -subject -dates -ext subjectAltName
```

`curl` の `-k` は証明書の検証を省略するオプション。自己署名のため付けている。`Location: https://...` が返り、`200` で終われば構成として正しい。

## 5. Zabbix 側の設定

### 疎通確認

Web UI を触る前に、コマンドで値が取れることを確認する。Zabbix サーバ側で実行する。

``` bash
zabbix_get -s <TARGET_IP> -k 'web.certificate.get[<ホスト名>,443,<TARGET_IP>]'
```

キーの引数は `[ホスト名, ポート, 接続先IP]` である。

**第3引数を指定するのが実務上重要**で、これが無いとエージェントはホスト名を自分で名前解決しようとする。DNS に載っていない証明書（内部向けやテスト環境）では `/etc/hosts` を書く必要が出てしまう。第3引数で接続先を直接指定し、ホスト名は SNI と検証に使わせる、という分担にできる。

JSON が返る。

``` json
{"x509":{"version":3,"serial_number":"...",
  "signature_algorithm":"SHA256-RSA",
  "issuer":"CN=<ホスト名>,O=Study,ST=Tokyo,C=JP",
  "subject":"CN=<ホスト名>,O=Study,ST=Tokyo,C=JP",
  "not_before":{"value":"..."},"not_after":{"value":"..."},
  "alternative_names":{"dns":["<ホスト名>","localhost"],"ip":["127.0.0.1"]}},
 "result":{"value":"valid-but-self-signed","message":"..."}}
```

`issuer` と `subject` が同一なのは自己署名の特徴である。認証局が発行した証明書では `issuer` に CA 名が入る。

### 検証結果は3値である

``` text
valid                    信頼できる
valid-but-self-signed    形式は正しいが自己署名
invalid                  検証に失敗（期限切れ、SAN無し、ホスト名不一致など）
```

自己署名は `invalid` ではなく**独立した値**として扱われる。標準トリガーの条件は `like "invalid"` という部分一致のため、`valid-but-self-signed` では発火しない。「自己署名を承知で使っている環境で毎日アラートが鳴る」ことを避ける設計になっている。

### テンプレートの適用

**Data collection → Hosts** → 対象 → `Templates` に `Website certificate by Zabbix agent 2` を追加。

構造は Docker テンプレートとは異なり、**LLD は無く、1つの親アイテムと12個の依存アイテム**で構成されている。

``` text
親    web.certificate.get[...]        JSONをまとめて取得
        ↓ JSONPath で切り出す
依存  Validation result               result.value
依存  Expires on                      x509.not_after
依存  Subject                         x509.subject
依存  Issuer                          x509.issuer  …（計12個）
```

証明書は取得が1回で全項目が分かるため、この形が効率的である。**1回の通信で済み、監視対象への負荷が小さい**。

反面、**親が失敗すると依存アイテムが全滅する**。個別に見ても原因は分からないので、`Not supported` が並んだときは親アイテムのエラーを読む。

### マクロの設定

同じホストの `Macros` タブで設定する。

| マクロ | 既定値 | 設定値 | 意味 |
| --- | --- | --- | --- |
| `{$CERT.WEBSITE.HOSTNAME}` | **なし** | `<ホスト名>` | 必須。未設定ではアイテムが動かない |
| `{$CERT.EXPIRY.WARN}` | `7` | `60` | 残り日数の警告閾値 |

`{$CERT.WEBSITE.HOSTNAME}` に既定値が無いのは、監視対象ごとに異なる値であり推測できないためである。**テンプレートを貼っただけでは動かない**テンプレートにあたる。

`{$CERT.EXPIRY.WARN}` を `60` にしたのは、今回の証明書が30日物のため。既定の `7` では発火しない。**閾値をホスト側で上書きする**という、テンプレート運用の基本形の確認になっている。実運用でも「本番は90日前、検証環境は7日前」のような使い分けをする。

### 生成されるトリガー

| トリガー | 条件 | 深刻度 |
| --- | --- | --- |
| SSL certificate expires soon | 残り日数 < `{$CERT.EXPIRY.WARN}` | Warning |
| SSL certificate has expired | 期限切れ | Average |
| SSL certificate is invalid | 検証結果に `invalid` を含む | Average |

## 6. 結果

`Monitoring → Latest data` で確認できた値。

``` text
Validation result      valid-but-self-signed
Expires on             （30日後の日付）
Subject                CN=<ホスト名>,O=Study,ST=Tokyo,C=JP
Issuer                 CN=<ホスト名>,O=Study,ST=Tokyo,C=JP
Version                3
Signature algorithm    SHA256-RSA
```

`{$CERT.EXPIRY.WARN}` を `60` にしてあるため、`SSL certificate expires soon` が Warning で上がった。狙った動作である。

取得間隔は既定で `15m` と長い。証明書は分単位で変わるものではないため妥当だが、**設定直後に値を見たい場合は待つことになる**。アイテム一覧の行の `···` から `Execute now` で即時実行できる。

## 監視のレイヤ（再掲）

| 障害 | Linux 監視 | HTTP 監視 | Docker 監視 | 証明書監視 |
| --- | --- | --- | --- | --- |
| DB コンテナの停止 | 変化なし | WordPress が応答しない | db コンテナが停止 | 変化なし |
| 証明書の期限切れ | 変化なし | （`-k` 相当なら通る） | 変化なし | **期限切れを検知** |

証明書の期限切れは、**他のどの層からも見えない**。HTTP 監視も、証明書検証をしない設定なら通ってしまう。層を分けて持つ必要がある代表例である。

## 未実施

- 認証局が発行した証明書での `valid` の確認
- 中間証明書の欠落（チェーン不備）の検知
- 期限切れ直前・直後の挙動確認（`-days 1` などでの再現）
