# Vault 構築手順 ① インストール〜ユーザー作成

HashiCorp Vault をシングルノード（Raft / HTTPS）で構築し、Root Token を使わない運用に切り替えるまでの手順。

| 項目 | 内容 |
|---|---|
| OS | AlmaLinux 9（AWS EC2） |
| Vault | v2.1.2（HashiCorp 公式 RPM） |
| ストレージ | Raft（統合ストレージ） |
| 通信 | HTTPS（自前 CA で署名した証明書） |
| 同居サービス | Zabbix / MariaDB / nginx / New Relic / Vuls |

---

## 0. 変数

以降のコマンドはすべて root で実行する。最初にこの変数を定義しておく。

```bash
export VAULT_IP=10.0.0.150                                          # サーバーのIP
export VAULT_FQDN=ip-10-0-0-150.ap-northeast-1.compute.internal     # hostname -f の結果
export VAULT_DNS=vault.internal                                     # 普段使う名前（任意）
export VAULT_USER=yuko                                              # 作成する管理者ユーザー名
```

---

## 1. 事前確認

```bash
cat /etc/os-release | grep -E '^(NAME|VERSION)='
hostname -f
ip -4 -br addr
getenforce                                     # Enforcing のままで進める
sudo ss -tlnp | grep -E ':8200|:8201'          # 何も出なければ空いている
free -h; df -h /
```

> firewalld は入っていないので、アクセス制御は AWS セキュリティグループで行う（TCP 8200 を VPN の CIDR から許可）。

---

## 2. OS の下準備

```bash
timedatectl set-timezone Asia/Tokyo
systemctl restart mariadb && systemctl restart zabbix-server php-fpm nginx   # 既存サービスのログ時刻をJSTに揃える（任意）
```

```bash
echo 'vm.swappiness = 1' > /etc/sysctl.d/99-vault.conf
sysctl -p /etc/sysctl.d/99-vault.conf          # swapを使われにくくする（本番ではswap無効が推奨）
```

---

## 3. Vault インストール

```bash
dnf install -y dnf-plugins-core
dnf config-manager --add-repo https://rpm.releases.hashicorp.com/RHEL/hashicorp.repo
dnf install -y vault
vault version
```

> RPM が vault ユーザー、`/opt/vault/data`、`/opt/vault/tls`（自己署名証明書）、systemd ユニットを自動で作成する。**この時点ではまだ起動しない。**

---

## 4. TLS 証明書（自前 CA）

自動生成の証明書は自己署名で SAN もないため使わず、自前の CA で署名した証明書に置き換える。

### 4-1. CA を作る

```bash
mkdir -p /root/vault-ca && chmod 700 /root/vault-ca && cd /root/vault-ca
```

```bash
cat > ca.ext <<'EOF'
basicConstraints = critical, CA:TRUE
keyUsage = critical, keyCertSign, cRLSign
subjectKeyIdentifier = hash
EOF
```

```bash
openssl genrsa -out ca.key 4096 && chmod 600 ca.key
openssl req -new -key ca.key -subj "/O=Clara Lab/CN=Vault Lab CA" -out ca.csr
openssl x509 -req -in ca.csr -signkey ca.key -days 3650 -sha256 -extfile ca.ext -out ca.crt
openssl x509 -in ca.crt -noout -subject -issuer -ext basicConstraints     # CA:TRUE を確認
```

> ⚠️ `ca.key` はどんな証明書でも発行できる鍵。Git・Confluence には絶対に載せない。

### 4-2. Vault のサーバー証明書を作る

```bash
cat > vault.ext <<EOF
basicConstraints = CA:FALSE
keyUsage = critical, digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth
subjectAltName = DNS:${VAULT_DNS}, DNS:${VAULT_FQDN}, DNS:localhost, IP:${VAULT_IP}, IP:127.0.0.1
EOF
```

```bash
openssl genrsa -out vault.key 2048 && chmod 600 vault.key
openssl req -new -key vault.key -subj "/O=Clara Lab/CN=${VAULT_DNS}" -out vault.csr
openssl x509 -req -in vault.csr -CA ca.crt -CAkey ca.key -CAcreateserial \
  -days 397 -sha256 -extfile vault.ext -out vault.crt
openssl verify -CAfile ca.crt vault.crt                                   # vault.crt: OK
openssl x509 -in vault.crt -noout -issuer -dates -ext subjectAltName      # SAN 5つを確認
```

### 4-3. 配置する

```bash
mv -n /opt/vault/tls/tls.crt /opt/vault/tls/tls.crt.rpm-default          # 元の証明書は退避
mv -n /opt/vault/tls/tls.key /opt/vault/tls/tls.key.rpm-default
install -o vault -g vault -m 600 vault.key /opt/vault/tls/vault.key      # mvではなくinstall（SELinuxラベル対策）
install -o vault -g vault -m 644 vault.crt /opt/vault/tls/vault.crt
ls -lZ /opt/vault/tls/
```

### 4-4. サーバーに CA を信頼させる

```bash
cp /root/vault-ca/ca.crt /etc/pki/ca-trust/source/anchors/vault-lab-ca.crt
update-ca-trust
trust list | grep -A3 'Vault Lab CA'                                      # trust: anchor を確認
```

> 証明書の更新（1年ごと）は 4-2 と 4-3 をやり直し、`systemctl reload vault`。

---

## 5. vault.hcl

```bash
cp -p /etc/vault.d/vault.hcl /etc/vault.d/vault.hcl.rpm-default
```

```bash
cat > /etc/vault.d/vault.hcl <<EOF
ui            = true
cluster_name  = "vault-lab"
disable_mlock = true

api_addr     = "https://${VAULT_IP}:8200"
cluster_addr = "https://${VAULT_IP}:8201"

storage "raft" {
  path    = "/opt/vault/data"
  node_id = "vault-01"
}

listener "tcp" {
  address         = "0.0.0.0:8200"
  tls_cert_file   = "/opt/vault/tls/vault.crt"
  tls_key_file    = "/opt/vault/tls/vault.key"
  tls_min_version = "tls12"
}
EOF
```

> `disable_mlock = true` は Raft 使用時の公式推奨（mlock を有効にするとメモリ不足で落ちやすい）。

```bash
sudo -u vault vault operator diagnose -config=/etc/vault.d/vault.hcl     # 必ず vault ユーザーで実行
chmod 700 /opt/vault/data                                                # diagnose の権限 warning 対策
```

> rootで diagnose を実行すると、データフォルダにroot所有のファイルができて起動に失敗する。
> 初期化前は `Raft Quorum: 0 voters` の warning が出るが、問題ない。

---

## 6. 起動・初期化・アンシール

```bash
systemctl enable --now vault
systemctl status vault --no-pager
```

```bash
export VAULT_ADDR=https://127.0.0.1:8200       # https。CAをOSに登録済みなので VAULT_CACERT は不要
vault status                                   # Initialized: false / Sealed: true
```

### 初期化（鍵は画面に出さずファイルへ）

```bash
(umask 077; set -C; vault operator init -key-shares=5 -key-threshold=3 -format=json > /root/vault-init.json)
ls -l /root/vault-init.json                    # -rw------- を確認
```

> `set -C` で、誤って2回実行したときに鍵ファイルが空で上書きされるのを防ぐ。
> ⚠️ `vault-init.json` の中身はチャット・Git・Confluence・Slack に貼らない。

### アンシール（3回）

```bash
vault operator unseal "$(jq -r '.unseal_keys_b64[0]' /root/vault-init.json)" | grep -E 'Sealed|Progress'
vault operator unseal "$(jq -r '.unseal_keys_b64[1]' /root/vault-init.json)" | grep -E 'Sealed|Progress'
vault operator unseal "$(jq -r '.unseal_keys_b64[2]' /root/vault-init.json)" | grep -E 'Sealed|HA Mode'
```

```bash
sleep 5 && vault status | grep -E 'Sealed|HA Mode'    # Sealed: false / HA Mode: active
```

> アンシール直後は `standby`。数秒で Raft のリーダー選挙が終わり `active` になる。

---

## 7. Root Token → 管理者ユーザーへ切り替え

### 7-1. Root Token でログイン（最初で最後）

```bash
vault login -no-print "$(jq -r '.root_token' /root/vault-init.json)"
vault operator raft list-peers                 # vault-01 / leader / true
```

### 7-2. 管理者ポリシー

```bash
vault policy write admin - <<'EOF'
# 学習用の管理者ポリシー：すべてのパスに対して全操作を許可
path "*" {
  capabilities = ["create", "read", "update", "delete", "list", "sudo"]
}
EOF
```

### 7-3. userpass 有効化・ユーザー作成

```bash
vault auth enable userpass
```

```bash
read -rs -p "${VAULT_USER}のパスワード: " P && echo && \
  vault write auth/userpass/users/${VAULT_USER} password="$P" token_policies=admin token_ttl=8h token_max_ttl=24h; unset P
```

> `read -rs` でパスワードを画面にもコマンド履歴にも残さない。

### 7-4. 作ったユーザーでログインし直す

```bash
vault login -no-print -method=userpass username=${VAULT_USER}
```

### 7-5. Root Token を無効化

```bash
vault token revoke "$(jq -r '.root_token' /root/vault-init.json)"
VAULT_TOKEN="$(jq -r '.root_token' /root/vault-init.json)" vault token lookup >/dev/null 2>&1 \
  && echo "まだ有効です" || echo "Root Tokenは無効になりました"
```

> 再び Root が必要になったら、Unseal Key 3つで `vault operator generate-root` を使って作り直す。

---

## 便利コマンド

```bash
# 今だれでログインしているか
vault token lookup -format=json | jq -r '.data | "ログイン中: \(.display_name) / 権限: \(.policies | join(",")) / 残り: \(.ttl)秒"'
```

