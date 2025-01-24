# Proxmox 関連作業

## install

普通にインストールした後リポジトリから有償版のものを抜く。

https://blog.nishi.network/2023/02/12/proxmox7-3-repository/

`https://enterprise.proxmox.com/debian/pve` の無効化と `deb http://download.proxmox.com/debian/pve buster pve-no-subscription
` を追加する。
今はどちらも WebUI 上からできる。

## PHOTO ディレクトリのマウント

samba/nfs をコンテナ側からマウントするのは手間がかかることが多いのでハイパーバイザ側でマウントしてそのディレクトリをコンテナ側に渡す。

今は /etc/fstab でやってる。そのうち systemd-mount に移行したい。

```
yaki-nasu.home:/mnt/datastore0/PHOTO/digicam /mnt/yaki-nasu/digicam nfs nofail 0 0
```

マウントしたディレクトリをコンテナ側に見せるコンフィグは WebUI から入れられないので `/etc/pve/lxc/<id>.conf` を直接書き換える。
また、コンテナ側の UID をホスト側にマッピングさせるために idmap の設定も入れる。
idmap は歯抜けの ID を描くのがめんどくさいらしく、3000 をマップする時は `0-2999`、`3000`, `3001-622534` の３つを書かないといけない。

```
arch: amd64
cores: 1
features: nesting=1
hostname: flashair
memory: 512
mp0: /mnt/yaki-nasu/digicam/,mp=/digicam
net0: name=eth0,bridge=vmbr0,firewall=1,hwaddr=BC:24:11:24:D4:94,ip=dhcp,ip6=auto,type=veth
ostype: ubuntu
rootfs: local-lvm:vm-104-disk-0,size=8G
swap: 512
unprivileged: 1
lxc.idmap: u    0 100000  3000
lxc.idmap: u 3000   3000     1
lxc.idmap: u 3001 103001 62534
lxc.idmap: g    0 100000  3000
lxc.idmap: g 3000   3000     1
lxc.idmap: g 3001 103001 62534
```
