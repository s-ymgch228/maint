# Gitlab 関連の作業ログ

## インストール

proxmox 上のコンテナで動かすので Docker イメージは使わずにパッケージをインストールする。
最近は Redhat 系より Ubuntu の方が流行っているようなので Ubuntu ベースで進める。

https://about.gitlab.com/ja-jp/install/#ubuntu にある通り進める。コマンドの一部は読み替え(例: apt-get -> apt)。

```
% apt update -y && apt upgrade -y
% apt install -y curl openssh-server ca-certificates tzdata perl
```

Postfix は最初から入っていてインストールされないが一応コマンドは打っておく。

```
% apt install -y postfix
```

これはそのまま。

```
curl https://packages.gitlab.com/install/repositories/gitlab/gitlab-ee/script.deb.sh | sudo bash
```

EXTERNAL_URL に指定している `gitlab.home` はインターネット側から名前解決できないのでインストール時にエラーが出る。
エラーが出ると言っても最後でコケているだけで動作に影響はしていないようなのでスルーしていい(?)。
本来なら EXTERNAL_URL を指定せずに http を使う状態でインストールしてその後 https 化する方法もある。(が、試してはいない)

```
sudo EXTERNAL_URL="https://gitlab.home" apt install -y gitlab-ee
```

## SSL 置き換え

gitlab-runnner を動かす場合は SANs (Subject Alternative Names) 付きの証明書がいるらしいので作り直す。
作り直した証明書のパスは`/etc/gitlab/ssl/<FQDN>.crt` 固定

```
% cd /etc/gitlab/ssl/
% openssl req -new -key gitlab.home.key -out gitlab.home.csr
% echo "subjectAltName = DNS:gitlab.home" > gitlab.home.san
% openssl x509 -days 3650 -req -signkey gitlab.home.key -in gitlab.home.csr -out gitlab.home.crt -extfile gitlab.home.san
```

作った証明書(gitlab.home.pem) の中身を見て `X509v3 Subject Alternative Name:` があれば成功。

```
% openssl x509 -text -in gitlab.home.crt -noout
```

gitlab を再起動する

```
% gitlab-ctl restart
```

適当なホストで証明書を取得して反映されているかみる。
```
 % openssl s_client -showcerts -connect gitlab.home:443 -servername gitlab.home < /dev/null 2>/dev/null | openssl x509 -outform PEM > gitlab.home.crt
 % openssl x509 -text -in gitlab.home.crt -noout
```

## runner を追加する

自己証明書を使っていると登録時にエラーになる。

```
root@runner0:~# gitlab-runner register  --url https://gitlab.home  --token glrt-t1_RY8rtprFQBUDhCS2BSYZ
Runtime platform                                    arch=amd64 os=linux pid=6269 revision=3153ccc6 version=17.7.0
Running in system-mode.                            
                                                   
Enter the GitLab instance URL (for example, https://gitlab.com/):
[https://gitlab.home]: 
ERROR: Verifying runner... failed                   runner=t1_RY8rtp status=couldn't execute POST against https://gitlab.home/api/v4/runners/verify: Post "https://gitlab.home/api/v4/runners/verify": tls: failed to verify certificate: x509: certificate signed by unknown authority
PANIC: Failed to verify the runner.                
root@runner0:~# 
```

`/etc/gitlab-runner/certs` の下に `<FQDN>.crt` のファイル名でサーバから取得した証明書をおく。
パスとファイル名は固定で変更不可。

```
% mkdir /etc/gitlab-runner/certs
% openssl s_client -connect gitlab.home:443 -showcerts < /dev/null | openssl x509 -outform PEM > /etc/gitlab-runner/certs/gitlab.home.crt
depth=0 C = JP, ST = Tokyo, L = Kita-ku, O = home, CN = gitlab.home
verify error:num=18:self-signed certificate
verify return:1
depth=0 C = JP, ST = Tokyo, L = Kita-ku, O = home, CN = gitlab.home
verify return:1
DONE
% gitlab-runner register  --url https://gitlab.home  --token glrt-t1_RY8rtprFQBUDhCS2BSYZ
Runtime platform                                    arch=amd64 os=linux pid=6307 revision=3153ccc6 version=17.7.0
Running in system-mode.                            
                                                   
Enter the GitLab instance URL (for example, https://gitlab.com/):
[https://gitlab.home]: 
Verifying runner... is valid                        runner=t1_RY8rtp
Enter a name for the runner. This is stored only in the local config.toml file:
[runner0]: 
Enter an executor: instance, custom, parallels, docker, kubernetes, docker+machine, docker-autoscaler, shell, ssh, virtualbox, docker-windows:
shell
Runner registered successfully. Feel free to start it, but if it's running already the config should be automatically reloaded!
 
Configuration (with the authentication token) was saved in "/etc/gitlab-runner/config.toml" 
% 
```
