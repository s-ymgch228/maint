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

公式では `EXTERNAL_URL=` に参照される URL を指定するように書かれているが、ここで指定する URL はインターネットから疎通性があるものでないといけない。
指定すると Let's encrypt で証明書が発行される。内向きの gitlab を立てるのでここでは指定しない。

```
% apt install -y gitlab-ee
```

## SSL 証明書配置

`/etc/gitlab/gitlab.rb` の `external_url` に `https://` から始まる URL を指定すると自己証明書が作られる。
が、これは gitlab-runner を動かす時に必要な SANs (Subject Alternative Names) がついていないので OpenSSL を使って自作する。

gitlab の証明書を入れておくパスは通常 `/etc/gitlab/ssl/<FQDN>.crt` 固定

```
% cd /etc/gitlab/ssl/
```

OpenSSL の　genrsa で秘密鍵を作る。パスワードを聞かれるので適当に設定する。このパスワードは `openssl req`, `openssl x509` で聞かれるものなのでわからなくなった時は作り直せばいい。
`genrsa` はパスワードありの鍵を生成するが nginx ではそのまま扱えないのでパスなしに変換する。（もしかすると初めからパスなしにする方法があるかも？)
gitlab の場合はパスなしの鍵を `<FQDN>.key` におく必要がある。

```
% openssl genrsa -aes256 -out ./gitlab.home.pem 2048
% openssl rsa -in gitlab.home.pem -out gitlab.home.key
```

req は作成する証明書に入れるパラメタを記述したやつ。これを実行すると初めに `genrsa` で設定したパスワードを求められる。
自己証明書なので国別コードと FQDN くらいは設定しておいて、残りは空欄にする。
```
% openssl req -new -key gitlab.home.key -out gitlab.home.csr
```

SANs (Subject Alternative Names) の内容は csr とは別にテキストで用意する。

```
echo "subjectAltName = DNS:gitlab.home" > gitlab.home.san
```

csr と san を組み合わせて X509 形式の証明書を作る。
```
% openssl x509 -days 3650 -req -signkey gitlab.home.key -in gitlab.home.csr -out gitlab.home.crt -extfile gitlab.home.san
```

作った証明書(gitlab.home.pem) の中身を見て `X509v3 Subject Alternative Name:` があれば成功。

```
% openssl x509 -text -in gitlab.home.crt -noout
```


## gitlab に https の設定を入れる

gitlab の設定ファイル(?) は `/etc/gitlab/gitlab.rb` にある。
このファイルのうち `external_url 'http://gitlab.example.com'` の行があるはずなのでここに FQDN を指定する。
指定する時に "http**s**://" のように指定すると SSL 周りの設定も自動で入る。

```
% grep -E ^external_url /etc/gitlab/gitlab.rb 
external_url 'https://gitlab.home'
%
```

反映には reconfigure コマンドが必要。このコマンドは結構時間がかかるので気長に待つ。
reconfigure 自体は事前に止めておく必要はないが、たまに nginx がうまく起動してこないことがあるので一旦止めてからやる。

```
% gitlab-ctl stop
% gitlab-ctl reconfigure
% gitlab-ctl start
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
