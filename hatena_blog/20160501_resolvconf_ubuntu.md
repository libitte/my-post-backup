
# ubuntu 15.10 における resolvconf まわりの初期設定

私の環境だと、 ubuntu 15.10 におけるデフォルトの resolvconf 設定だと、
頻繁に google.com の名前解決ができなくなる現象が発生した。

これを解決するために、eth0 に固定アドレスを設定するようにした。
(`/etc/network/interfaces`, `/etc/resolvconf/resolv.conf/base` などを設定した)

例:

* /etc/network/interfaces

```
auto lo
iface lo inet loopback

auto eth0
iface eth0 inet static
    address 192.168.11.2
    netmask 255.255.255.0
    gateway 192.168.11.1
    dns-servers 192.168.11.1
```

結果として、 `/etc/resolv.conf` が生成される

設定を有効にするには、 `resolvconf` コマンドを実行すれば良い。

```
$ sudo resolvconf -u
```


## Trouble Shooting

> "ping: unknown host google.com" but IP's works fine

の時は `resolvconf` を入れなおした。

```
sudo apt-get remove --purge resolvconf && sudo apt-get install --reinstall resolvconf
```

## References

* http://www.tecmint.com/set-static-ip-address-in-ubuntu-15-10-server/
* http://www.asterisk-works.jp/wiki/index.php/%E3%83%8D%E3%83%83%E3%83%88%E3%83%AF%E3%83%BC%E3%82%AF%E8%A8%AD%E5%AE%9A(Ubuntu)
* http://d.hatena.ne.jp/zzkater/20140713/1405250096
* http://askubuntu.com/questions/368435/how-do-i-fix-dns-resolving-which-doesnt-work-after-upgrading-to-ubuntu-13-10-s
* http://askubuntu.com/questions/455338/ping-unknown-host-google-com-but-ips-works-fine
* http://askubuntu.com/questions/465729/ping-unknown-host-google-com-in-ubuntu-server
* https://forums.ubuntulinux.jp/viewtopic.php?id=17345
* http://geektrainee.hatenablog.jp/entry/2014/09/23/002443


