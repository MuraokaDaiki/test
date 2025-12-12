# 自作VPN手順書ｖ2.0

WireGuardは自宅にVPSサーバーを構築し、複数の端末にVPN接続を提供し安全な通信を実現する。前回の方法は鍵やパスワードの管理など煩雑な部分があったので、より簡便な方法として以下のように実行する。

## 環境

- Ubuntu　24.04.3 LTS
- WireGuard　Ver3.8

## 事前準備

### ✅ 1. **ホストOSがWireGuardをサポートしているか確認**

- Linuxカーネルが **5.6以上** であること（WireGuardが標準で含まれている）
- 確認コマンド：
    
    bash
    
    `uname -r`
    

### ✅ 2. `/lib/modules` **が存在しているか確認**

- コンテナがカーネルモジュールにアクセスするために必要
- 確認コマンド：
    
    bash
    
    `ls /lib/modules`
    

### ✅ 3. **IPフォワーディングを有効化（必要に応じて）**

- VPN経由でインターネットに出たい場合はこれが必要
- 一時的に有効にするには：
    
    bash
    
    `sudo sysctl -w net.ipv4.ip_forward=1`
    
- 永続化するには `/etc/sysctl.conf` に以下を追加：
    
    コード
    
    `net.ipv4.ip_forward=1`
    

### ✅ 4. **ファイアウォールとルーターのポート開放**

- 外部から接続するには、**UDP 51820番ポート** を開ける必要がある
- ルーターのポートフォワーディング設定で、ホストのIPに転送するように設定

### ✅ 5. `./config` **ディレクトリの準備**

- 初回起動時に自動生成されるので、**空の状態**にしておくとベスト
- すでに存在している場合は、バックアップしておくと安心

### ✅ 6. **DockerとDocker Composeのインストール確認**

- 確認コマンド：
    
    bash
    
    `docker -v docker compose version`
    

### ✅ 7. **グローバルIPの確認（**`SERVERURL=auto` **がうまくいかない場合）**

- 自分のグローバルIPを調べて、`SERVERURL=xxx.xxx.xxx.xxx` に手動で設定することもできるよ
- 確認方法：
    
    bash
    
    `curl ifconfig.me`
    

## Docker インストール

[https://docs.docker.com/engine/install/](https://docs.docker.com/engine/install/)

### 環境構築

```bash
# Add Docker's official GPG key:
sudo apt update
sudo apt install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Signed-By: /etc/apt/keyrings/docker.asc
EOF

sudo apt update
```

### 各種pluginをインストール

```jsx
 sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

### インストールされたかテスト

```jsx
sudo systemctl status docker  # dockerの現在の状態を確認する

sudo systemctl start docker  # 動いていない場合マニュアルで起動

sudo docker run hello-world  # hello-worldが無事出力されれば成功
```

## 作業ディレクトリを作成

`mkdir ~/wireguard2 && cd ~/wireguard2`

## docker-composeファイルを作成

`touch docker-compose.yaml` （名前は必ずdocker-composeとする。）

## docker compose upで以下yamlファイルを実行

docker-composeファイル

```yaml
version: '3.8'
services:
  wireguard:
    image: linuxserver/wireguard
    container_name: wireguard
    cap_add:
      - NET_ADMIN
      - SYS_MODULE
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Asia/Tokyo
      - SERVERURL=auto
      - SERVERPORT=51820
      - PEERS=3
      - INTERNAL_SUBNET=10.13.13.0
    volumes:
      - ./config:/config
      - /lib/modules:/lib/modules
    ports:
      - "51820:51820/udp"
    sysctls:
      - net.ipv4.conf.all.src_valid_mark=1
    restart: unless-stopped
```

### オプション変更の場合は以下で編集

```jsx
vi docker-compose.yaml
```

```jsx
nano docker-compose.yaml
```

### **WireGuardコンテナが起動**

- `linuxserver/wireguard` イメージが使われて、VPNサーバーが立ち上がります。
- `51820/udp` ポートがホストとバインドされ、外部からのVPN接続を受け付けます。
1. **初回起動時に設定ファイルが自動生成**
    - `./config` ディレクトリに以下のような構成ができます：
        
        コード
        
        `./config/
          ├── peer1/
          │   ├── peer1.conf
          │   └── peer1.png  ← QRコード（スマホ用）
          ├── peer2/
          ├── peer3/
          └── server/
              ├── privatekey
              └── publickey`
        

### **クライアント設定が3つ生成される**

- `PEERS=3` により、3人分のクライアント設定が作られます。
- 各クライアントは `.conf` ファイルを使ってPCから、または `.png` のQRコードをスマホで読み取って接続できます。
1. **WireGuardカーネルモジュールが必要**
    - ホスト側のLinuxカーネルがWireGuardをサポートしている必要があります（5.6以降が理想）。
    - `cap_add` と `/lib/modules` のマウントで、カーネルモジュールへのアクセスが確保されます。
2. `SERVERURL=auto` **がグローバルIPを検出**
    - 自動でホストのグローバルIPを取得して設定しますが、うまくいかない場合は明示的に `SERVERURL=xxx.xxx.xxx.xxx` にした方が安定します。

### 🌧️ 起動時に注意したいこと

- `./config` **がすでに存在していると再生成されない**
→ 初回起動時は空にしておくとスムーズです。
- **ホストのファイアウォールやルーターのポート開放**
→ 外部から接続するには、51820/UDP を開けておく必要があります。
- **IPフォワーディングとNATの設定（必要に応じて）**
→ クライアントがVPN経由でインターネットに出る場合、ホスト側で `iptables` の設定が必要です。

## WireGuard実行結果

```jsx
Uname info: Linux 8de1185ca1b5 6.14.0-36-generic #36~24.04.1-Ubuntu SMP PREEMPT_DYNAMIC Wed Oct 15 15:45:17 UTC 2 x86_64 GNU/Linux
wireguard  | **** As the wireguard module is already active you can remove the SYS_MODULE capability from your container run/compose. ****
wireguard  | ****     If your host does not automatically load the iptables module, you may still need the SYS_MODULE capability.     ****
wireguard  | **** Server mode is selected ****
wireguard  | **** SERVERURL var is either not set or is set to "auto", setting external IP to auto detected value of 60.144.210.189 ****
wireguard  | **** External server port is set to 51820. Make sure that port is properly forwarded to port 51820 inside this container ****
wireguard  | **** Internal subnet is set to 10.13.13.0 ****
wireguard  | **** AllowedIPs for peers 0.0.0.0/0, ::/0 ****
wireguard  | **** PEERDNS var is either not set or is set to "auto", setting peer DNS to 10.13.13.1 to use wireguard docker host's DNS. ****
wireguard  | **** No wg0.conf found (maybe an initial install), generating 1 server and 3 peer/client confs ****
wireguard  | PEER 1 QR code (conf file is saved under /config/peer1):

ーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーー
#peer1,peer2,peer3と三つ分のデバイスのQRコードとconf fileが生成されるのでWireGuardアプリで読み込む。
ーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーー

[custom-init] No custom files found, skipping...
wireguard  | maxprocs: Leaving GOMAXPROCS=4: CPU quota undefined
wireguard  | .:53
wireguard  | CoreDNS-1.12.1
wireguard  | linux/amd64, go1.24.1, 
wireguard  | **** Found WG conf /config/wg_confs/wg0.conf, adding to list ****
wireguard  | **** Activating tunnel /config/wg_confs/wg0.conf ****
wireguard  | [#] ip link add dev wg0 type wireguard
wireguard  | [#] wg setconf wg0 /dev/fd/63
wireguard  | [#] ip -4 address add 10.13.13.1 dev wg0
wireguard  | [#] ip link set mtu 1420 up dev wg0
wireguard  | [#] ip -4 route add 10.13.13.4/32 dev wg0
wireguard  | [#] ip -4 route add 10.13.13.3/32 dev wg0
wireguard  | [#] ip -4 route add 10.13.13.2/32 dev wg0
wireguard  | [#] iptables -A FORWARD -i wg0 -j ACCEPT; iptables -A FORWARD -o wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth+ -j MASQUERADE
wireguard  | **** All tunnels are now active ****
wireguard  | [ls.io-init] done.

```

## Docker停止→再起動

```jsx
docker stop linuxserver/wireguard

docker restart linuxserver/wireguard

```

## Docker消去→再起動

```jsx
docker rm linuxserver/wireguard

cd wireguard2

docker compose up -d docker-compose.yaml
```