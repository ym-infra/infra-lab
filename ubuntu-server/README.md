# Ubuntu Server Home Lab

使わなくなったノートPCを再利用し、
Ubuntu Serverをインストールして自宅検証サーバーを構築しました。

## 環境

- OS: Ubuntu Server 26.04.1 LTS
- CPU: Intel Core i7-8565U（4コア / 8スレッド）
- Memory: 16GB
- Storage:
  - SSD 128GB（Ubuntu Server）
  - HDD 1TB（既存データのため未使用）

## 構築内容

### 1. 起動用USBの作成

Ubuntu ServerのISOイメージをダウンロードし、
Rufusを使用して起動用USBを作成。

### 2. USBから起動

BIOS / UEFIのBoot設定を変更し、
USBメモリを起動順位の1番目に設定。

USBからUbuntu Serverのインストーラーを起動。

### 3. Ubuntu Serverのインストール

インストール先として128GB SSDを選択。

ストレージはLVM構成とし、
1TB HDDは既存データがあるため変更せず残しました。

### 4. ネットワーク設定

Ubuntu Server起動後、以下のコマンドで
ネットワークインターフェースを確認。

    ip addr

Wi-Fiインターフェース `wlo1` は認識されていましたが、
IPアドレスが付与されていなかったためNetplanを設定。

設定ファイル：

    /etc/netplan/00-installer-config.yaml

設定例：

    network:
      version: 2
      wifis:
        wlo1:
          dhcp4: true
          access-points:
            "<SSID>":
              password: "<PASSWORD>"

設定を反映。

    sudo netplan generate
    sudo netplan apply

ルーター側ではDHCP予約を設定し、
Ubuntu Serverに同じIPアドレスが払い出されるようにしました。

### 5. SSH接続

SSHサービスが起動していることを確認。

    sudo systemctl status ssh

別のPCからUbuntu ServerへSSH接続。

    ssh <USER>@<SERVER_IP>

SSH経由でUbuntu Serverを操作できることを確認しました。

## 今後やりたいこと

- Dockerの導入
- Webサーバーの構築
- コンテナ環境の検証
- Linux / ネットワークの学習
