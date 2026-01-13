# Gitea を lxc に入れる

## Ubuntu の lxc を立てる

初めは update
```
apt update -y && sudo apt upgrade -y
reboot

```

日本語環境を入れる

```
apt install -y language-pack-ja-base language-pack-ja
locale-gen ja_JP.UTF-8
```

## gitea を入れる

git ユーザを作っておく

```
adduser \
   --system \
   --group \
   --shell /bin/bash \
   --create-home \
   --home /home/git \
   --gecos 'Git Version Control' \
   git
```

必要なパッケージを入れておく
```
apt update
apt install git sqlite3 -y
```

gitea はバイナリを持ってくる
```
 cd /usr/local/bin/
 wget -O gitea https://dl.gitea.com/gitea/1.25.3/gitea-1.25.3-linux-amd64
 chmod -x gitea 
 ```

ディレクトリ構成を作る
```
mkdir -p /var/lib/gitea/{custom,data,log}
chown -R git:git /var/lib/gitea/
chmod -R 750 /var/lib/gitea/
mkdir /etc/gitea
chown root:git /etc/gitea
chmod 770 /etc/gitea
```

systemctl のスクリプトは自分で書かないといけない
```
cat > /etc/systemd/system/gitea.service <<EOS
> [Unit]
Description=Gitea (Git with a cup of tea)
After=syslog.target
After=network.target

[Service]
RestartSec=2s
Type=simple
User=git
Group=git
WorkingDirectory=/var/lib/gitea/
ExecStart=/usr/local/bin/gitea web --config /etc/gitea/app.ini
Restart=always
Environment=USER=git HOME=/home/git GITEA_WORK_DIR=/var/lib/gitea

[Install]
WantedBy=multi-user.target
EOS
systemctl daemon-reload
```

https 化するために nginx を立てる
```
sudo apt update
sudo apt install nginx -y
cat > /etc/nginx/sites-available/gitea <<'EOS'
server {
    listen 80;
    server_name _;
    return 301 https://$host$request_uri; # HTTPに来たら強制的にHTTPSへ飛ばす
}

server {
    listen 443 ssl;
    server_name _;

    ssl_certificate     /etc/nginx/ssl/gitea.crt;
    ssl_certificate_key /etc/nginx/ssl/gitea.key;

    location / {
        proxy_pass http://localhost:3000; # Giteaのポート
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
EOS
```

nginx の設定チェック

```
sudo ln -s /etc/nginx/sites-available/gitea /etc/nginx/sites-enabled/
sudo rm /etc/nginx/sites-enabled/default  # デフォルト設定と競合しないよう削除
sudo nginx -t                             # 構文チェック
```

gitea の設定を入れる

```
cat > /etc/gitea/app.ini <<'EOS'
[server]
PROTOCOL = http
DOMAIN   = gitea-mx0.home
ROOT_URL = https://gitea-mx0.home/
HTTP_ADDR = 127.0.0.1
HTTP_PORT = 80
EOS
chown git /etc/gitea/app.ini
```
