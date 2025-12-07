# 自作VPN手順書

## Dockerでipsec-vpn-serverイメージを立ち上げる

**Note:** All the variables to this image are optional, which means you don't have to type in any variable, and you can have an IPsec VPN server out of the box! To do that, create an empty `env` file using `touch vpn.env`, and skip to the next section.

**注意:** このイメージのすべての変数はオプションです。つまり、変数を一切入力する必要がなく、そのままIPsec VPNサーバーを立ち上げることができます。そのためには、`touch vpn.env`で空の`env`ファイルを作成し、次のセクションに進んでください。

[envファイル（環境変数格納）](https://www.notion.so/env-2bfaf040ebde809789a0d74493523acb?pvs=21)

```jsx
docker run \
    --name ipsec-vpn-server \
    --env-file ./vpn.env \
    --restart=always \
    -v ikev2-vpn-data:/etc/ipsec.d \
    -v /lib/modules:/lib/modules:ro \
    -p 500:500/udp \
    -p 4500:4500/udp \
    -d --privileged \
    hwdsl2/ipsec-vpn-server
```

[Docker操作](https://www.notion.so/Docker-2bfaf040ebde80d69714f90f411e86f7?pvs=21)

[自宅WiFi](https://www.notion.so/WiFi-2bfaf040ebde80c8a71bc96f1dfb1add?pvs=21)

## docker compose upで以下compose.yamlファイルを実行

```jsx
services:
  vpnserver:
    image: hwdsl2/ipsec-vpn-server
    ports:
      - 500:500/udp
      - 4500:4500/udp
    volumes:
      - ./ikev2-vpn-data:/etc/ipsec.d
      - /lib/modules:/lib/modules:ro
    privileged: true
    environment:
      - VPN_IKEV2_ONLY=yes
      - VPN_XAUTH_NET=192.168.43.0/24
      - VPN_XAUTH_POOL=192.168.43.10-192.168.43.250
    restart: unless-stopped

```