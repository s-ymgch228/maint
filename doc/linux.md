## Linux 一般

### NFS mount

|       |  | 
|-------|--|
| いま   | /etc/systemd/system/ に *.mount でファイルを作る
| むかし | /etc/fstab に書く。 nofail をつけておくと起動時に繋げなくてもいい。

```
[Unit]
Description= Mount digicam directory

[Mount]
What=yaki-nasu.home:/mnt/datastore0/PHOTO/digicam
Where=/mnt/digicam
Options=vers=4
Type=nfs
TimeoutSec=10

[Install]
WantedBy=multi-user.target
```

- ファイル名はマウント先のパスを '/' -> '-' する
  - `/mnt/digicam` にマウントするなら `mnt-digicam.mount` を `/etc/systemd/system` の下に作る
- マウント先のパスに '-' を含んでいると動かない(はず)
