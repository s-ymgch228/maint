# Proxmox 用 https proxy を立てる

 - proxmox は 8006/tcp を使っていて変更不可
 - 443/tcp に変えるには reverse proxy が必要

 # 自己証明書

 openssl コマンドで適当に作る。

    mkdir /etc/nginx/ssl
    openssl genrsa -out /etc/nginx/ssl/secret.key 4096
    openssl req -new -key /etc/nginx/ssl/secret.key  -out /etc/nginx/ssl/server.csr
    openssl x509 -days 3650 -req -signkey /etc/nginx/ssl/secret.key -in /etc/nginx/ssl/server.csr -out /etc/nginx/ssl/server.crt
 
適当な秘密鍵(secret.key) を作った後 SSL に必要なパラメタを書いた仕様書(server.csr)を作って、csr, key から公開鍵(server.crt) を作る

# nginx を設定する

適当に作った自己証明書を `ssl_certicficate*` に設定すればいい。
proxy_pass はホスト名(prox.home) にした方が良さそうに見えるが、 プロセス立ち上げ時に名前解決できなければいけなくなるのであえてアドレス直打ち

```/etc/nginx/conf.d/sever.conf 
server{
    listen 443 ssl;
    ssl on;
    ssl_certificate     /etc/nginx/ssl/server.crt;
    ssl_certificate_key /etc/nginx/ssl/secret.key;
    proxy_redirect off;

    location / {
          proxy_set_header X-Real-IP $remote_addr;
          proxy_set_header X-Forwarded-Proto $scheme;
          proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
          proxy_set_header Host $host;
          proxy_set_header Upgrade $http_upgrade;
          proxy_set_header Connection "Upgrade";
          proxy_buffering off;
          client_max_body_size 0;
          proxy_connect_timeout 3600s;
          proxy_read_timeout 3600s;
          proxy_send_timeout 3600s;
          send_timeout 3600s;
          proxy_pass https://192.168.1.95:8006;
    }
}
```

仮に名前解決できないと `host not found in upstream ~` なログが出て異常終了する。

```
Jan 04 14:43:25 pxgate systemd[1]: Starting nginx.service - A high performance web server and a reverse proxy server...
Jan 04 14:43:25 pxgate nginx[12675]: 2025/01/04 14:43:25 [warn] 12675#12675: the "ssl" directive is deprecated, use the "listen ... ssl" directive instead in /etc/nginx/conf.d/sever.co>
Jan 04 14:43:25 pxgate nginx[12675]: 2025/01/04 14:43:25 [emerg] 12675#12675: host not found in upstream "prox.home" in /etc/nginx/conf.d/sever.conf:21
Jan 04 14:43:25 pxgate nginx[12675]: nginx: configuration file /etc/nginx/nginx.conf test failed
Jan 04 14:43:25 pxgate systemd[1]: nginx.service: Control process exited, code=exited, status=1/FAILURE
Jan 04 14:43:25 pxgate systemd[1]: nginx.service: Failed with result 'exit-code'.
Jan 04 14:43:25 pxgate systemd[1]: Failed to start nginx.service - A high performance web server and a reverse proxy server.
```