# Ubuntu Server 構築手順

## 1. 起動用USBの作成

### Ubuntu Server ISOの取得
Ubuntu Server 26.04.1 LTS のISOイメージを取得。

### Rufusで起動用USBを作成
ISOファイルをUSBメモリへコピーするだけでは起動用USBにはならないため、
OSSのRufusを使用してISOイメージを書き込む。

---

## 2. USBからUbuntu Serverを起動

PCをシャットダウンし、作成した起動用USBを接続。

Windowsが起動する前にBIOS / UEFIのBoot設定画面を開き、
起動順位を以下のように変更。

1. USB Memory
2. HDD / SSD

設定を保存して再起動し、USBからUbuntu Serverのインストーラーを起動。

---

## 3. Ubuntu Serverのインストール

インストーラーに従って以下を設定。

- Language
- Keyboard
- Storage
- User / Hostname
- SSH

### ストレージ

Ubuntu Serverのインストール先として128GB SSDを選択。

- SSD 128GB：Ubuntu Server
- HDD 1TB：既存データがあるため未使用

ストレージ管理にはLVMを使用。

OSをSSDへ配置し、論理ボリュームの容量を後から変更しやすい構成とした。

インストール完了後、USBメモリを取り外して再起動。

---

## 4. ネットワーク設定

Ubuntu Serverへログイン後、ネットワークインターフェースを確認。

    ip addr

以下のインターフェースを確認。

- eno2：有線LAN
- wlo1：Wi-Fi

Wi-Fiインターフェース `wlo1` は認識されていたが、
IPアドレスが付与されていなかったためNetplanを設定。

### Netplan設定

設定ファイル：

    /etc/netplan/00-installer-config.yaml

Wi-Fi設定を追加。

    network:
      version: 2
      wifis:
        wlo1:
          dhcp4: true
          access-points:
            "<SSID>":
              password: "<PASSWORD>"

※ 実際のSSID・パスワードは記載しない。

設定内容を確認。

    sudo netplan generate

設定を反映。

    sudo netplan apply

再度ネットワーク状態を確認。

    ip addr

`wlo1` にIPアドレスが付与されたことを確認。

外部ネットワークへの疎通も確認。

    ping -c 3 8.8.8.8

---

## 5. DHCP予約

SSH接続時に接続先IPアドレスが変わらないよう、
ルーター側でDHCP予約を設定。

Ubuntu Server側ではDHCPを使用したまま、
ルーターから常に同じIPアドレスが払い出されるようにした。

---

## 6. SSH接続

SSHサービスの状態を確認。

    sudo systemctl status ssh

`active (running)` であることを確認。

別のWindows PCからUbuntu ServerへSSH接続。

    ssh <USER>@<SERVER_IP>

ログイン後、Ubuntu Serverのプロンプトが表示されることを確認。

    <USER>@home-lab:~$

これにより、Ubuntu Serverを直接操作せず、
別PCからリモートで操作できる状態となった。

---

## 7. 構築時にハマったポイント

### ISOファイルはUSBへコピーするだけでは起動できない

ISOイメージから起動可能なインストールメディアを作成する必要がある。
今回はRufusを使用して起動用USBを作成した。

### キーボード配列

インストール後、キーボード配列が想定と異なり、
記号を正しく入力できなかった。

日本語（JIS）キーボードへ設定を変更して解消。

### NetplanのYAML

Netplanの設定ファイルはYAML形式のため、
インデントや `:` の記述ミスによってエラーが発生した。

    sudo netplan generate

で設定内容を確認しながら修正した。

---

## 8. 今後

構築したUbuntu Serverを自宅検証環境として利用する。

今後は以下を検証予定。

- Docker
- Webサーバー
- コンテナ環境
- Linux / ネットワーク
