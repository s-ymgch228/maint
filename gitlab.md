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
% openssl req -new -key gitlab.home.key -out gitlab.home.crt
% echo "subjectAltName = DNS:gitlab.home" > san.txt
% openssl x509 -days 3650 -req -signkey gitlab.home.key -in gitlab.home.crt -out gitlab.home.crt -extfile san.txt
```

作った証明書(gitlab.home.pem) の中身を見て `X509v3 Subject Alternative Name:` があれば成功。

```
% openssl x509 -text -in gitlab.home.crt -noout
```

gitlab を再起動する

```
% gitlab-ctl restart
```