# virt-v2v

- [RHEL10を、KVMホストにする]()
- [ブリッジ作成する]()
- [virt-manager用マシンから、RHEL10-KVMホストに接続する]()


## RHEL10を、KVMホストにする

以前作ったCentOS6のように、qemu-kvm libvirt libvirt-client があれば、KVMホストの準備完了
```
[root@centos6-8300 ~]# rpm -q qemu-kvm libvirt virt-manager libvirt-client virt-install
qemu-kvm-0.12.1.2-2.503.el6_9.6.x86_64
libvirt-0.10.2-64.el6.x86_64
virt-manager-0.9.0-34.el6.x86_64
libvirt-client-0.10.2-64.el6.x86_64
パッケージ virt-install はインストールされていません。
[root@centos6-8300 ~]#
```

RHEL10の現在の状態
```
ubuntu@rhel10:~$ rpm -q qemu-kvm libvirt virt-manager libvirt-client virt-install
パッケージ qemu-kvm はインストールされていません
パッケージ libvirt はインストールされていません
パッケージ virt-manager はインストールされていません
libvirt-client-11.5.0-4.2.el10_1.x86_64
パッケージ virt-install はインストールされていません
ubuntu@rhel10:~$
```
最低、以下をインストールする
```
ubuntu@rhel10:~$ sudo dnf -y install qemu-kvm libvirt
```
インストールできた
```
ubuntu@rhel10:~$ rpm -q qemu-kvm libvirt libvirt-client
qemu-kvm-10.0.0-14.el10_1.x86_64
libvirt-11.5.0-4.2.el10_1.x86_64
libvirt-client-11.5.0-4.2.el10_1.x86_64
ubuntu@rhel10:~$
```

KVMホストとしての準備完了

libvirtdを有効化➕起動する
```
ubuntu@rhel10:~$ service libvirtd status
Redirecting to /bin/systemctl status libvirtd.service
○ libvirtd.service - libvirt legacy monolithic daemon
     Loaded: loaded (/usr/lib/systemd/system/libvirtd.service; disabled; preset: disabled)
     Active: inactive (dead)
TriggeredBy: ○ libvirtd-admin.socket
             ○ libvirtd-ro.socket
             ○ libvirtd.socket
       Docs: man:libvirtd(8)
             https://libvirt.org/
ubuntu@rhel10:~$ sudo systemctl enable --now libvirtd
Created symlink '/etc/systemd/system/multi-user.target.wants/libvirtd.service' → '/usr/lib/systemd/system/libvirtd.service'.
Created symlink '/etc/systemd/system/sockets.target.wants/libvirtd.socket' → '/usr/lib/systemd/system/libvirtd.socket'.
Created symlink '/etc/systemd/system/sockets.target.wants/libvirtd-ro.socket' → '/usr/lib/systemd/system/libvirtd-ro.socket'.
Created symlink '/etc/systemd/system/sockets.target.wants/libvirtd-admin.socket' → '/usr/lib/systemd/system/libvirtd-admin.socket'.
ubuntu@rhel10:~$ service libvirtd status
Redirecting to /bin/systemctl status libvirtd.service
● libvirtd.service - libvirt legacy monolithic daemon
     Loaded: loaded (/usr/lib/systemd/system/libvirtd.service; enabled; preset: disabled)
     Active: active (running) since Sun 2026-01-18 17:10:55 JST; 22s ago
...
```


## ブリッジ作成する

RHEL(CentOS)初期セットアップ時には、ブリッジはない　→ なので自分で作る必要ある

以下でCentOS6に対してやったことと、同じことをする

https://github.com/pkthom/centos_6.10/blob/main/README.md#%E3%83%96%E3%83%AA%E3%83%83%E3%82%B8%E4%BD%9C%E6%88%90

現在の状態 (CentOS6の時と同じく、virbr0以下に外から繋がらない)
```
ubuntu@rhel10:~$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host noprefixroute
       valid_lft forever preferred_lft forever
2: ens18: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether xx:xx:xx:xx:xx:xx brd ff:ff:ff:ff:ff:ff
    altname enp6s18
    altname enxbc2411411a76
    inet 10.20.30.202/24 brd 10.20.30.255 scope global noprefixroute ens18
       valid_lft forever preferred_lft forever
    inet6 fe80::be24:11ff:fe41:1a76/64 scope link noprefixroute
       valid_lft forever preferred_lft forever
3: virbr0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc htb state DOWN group default qlen 1000
    link/ether xx:xx:xx:xx:xx:xx brd ff:ff:ff:ff:ff:ff
    inet 10.20.122.1/24 brd 10.20.122.255 scope global virbr0
       valid_lft forever preferred_lft forever
ubuntu@rhel10:~$
```

<details>
  <summary>不要だったが、ちなみにRHEL10では、NIC情報は /etc/sysconfig/network-scripts/ ではなくなっている</summary>


```
root@rhel10:/etc/NetworkManager/system-connections# cat ens18.nmconnection
[connection]
id=ens18
uuid=xxxx
type=ethernet
autoconnect-priority=-999
interface-name=ens18
timestamp=1767263405

[ethernet]

[ipv4]
address1=xxx.xxx.xxx.xxx/24
dns=8.8.8.8;1.1.1.1;
gateway=xxx.xxx.xxx.1
method=manual

[ipv6]
addr-gen-mode=eui64
method=auto

[proxy]
root@rhel10:/etc/NetworkManager/system-connections#
```

</details>

ブリッジのNIC(br0)を追加する
```
root@rhel10:/etc/NetworkManager/system-connections# nmcli connection add type bridge con-name br0 ifname br0
接続 'br0' (574b321e-65a3-42ae-ad03-2b6cce887da0) が正常に追加されました。
root@rhel10:/etc/NetworkManager/system-connections# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host noprefixroute
       valid_lft forever preferred_lft forever
2: ens18: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether xx:xx:xx:xx:xx:xx brd ff:ff:ff:ff:ff:ff
    altname enp6s18
    altname enxbc2411411a76
    inet 10.20.30.202/24 brd 10.20.30.255 scope global noprefixroute ens18
       valid_lft forever preferred_lft forever
    inet6 fe80::be24:11ff:fe41:1a76/64 scope link noprefixroute
       valid_lft forever preferred_lft forever
3: virbr0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc htb state DOWN group default qlen 1000
    link/ether xx:xx:xx:xx:xx:xx brd ff:ff:ff:ff:ff:ff
    inet 10.20.122.1/24 brd 10.20.122.255 scope global virbr0
       valid_lft forever preferred_lft forever
4: br0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default qlen 1000
    link/ether xx:xx:xx:xx:xx:xx brd ff:ff:ff:ff:ff:ff
root@rhel10:/etc/NetworkManager/system-connections#
```
br0の詳細設定
```
root@rhel10:/etc/NetworkManager/system-connections# nmcli connection modify br0 ipv4.addresses 10.20.30.202/24 ipv4.gateway 10.20.30.1 ipv4.dns "8.8.8.8,1.1.1.1" ipv4.method manual
```

ens18をbr0に従属させる設定を追加
```
root@rhel10:/etc/NetworkManager/system-connections# nmcli connection add type ethernet slave-type bridge con-name br0-port1 ifname ens18 master br0
接続 'br0-port1' (a2bb2995-f363-4981-9da4-72a23b0c687f) が正常に追加されました。
```

古い設定を削除し、ブリッジを起動
```
root@rhel10:/etc/NetworkManager/system-connections# nmcli connection down ens18 && nmcli connection del ens18 && nmcli connection up br0
client_loop: send disconnect: Connection reset
```

ブリッジ完成
```
ubuntu@rhel10:~$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host noprefixroute
       valid_lft forever preferred_lft forever
2: ens18: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel master br0 state UP group default qlen 1000
    link/ether xx:xx:xx:xx:xx:xx brd ff:ff:ff:ff:ff:ff
    altname enp6s18
    altname enxbc2411411a76
3: virbr0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc htb state DOWN group default qlen 1000
    link/ether xx:xx:xx:xx:xx:xx brd ff:ff:ff:ff:ff:ff
    inet 10.20.122.1/24 brd 10.20.122.255 scope global virbr0
       valid_lft forever preferred_lft forever
4: br0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether xx:xx:xx:xx:xx:xx brd ff:ff:ff:ff:ff:ff
    inet 10.20.30.202/24 brd 10.20.30.255 scope global noprefixroute br0
       valid_lft forever preferred_lft forever
    inet6 fe80::3e8b:8773:9199:c84f/64 scope link noprefixroute
       valid_lft forever preferred_lft forever
ubuntu@rhel10:~$
```

## virt-manager用マシンから、RHEL10-KVMホストに接続する

virt-manager用マシン（デスクトップUbuntu24）から、RHEL10に、パスワードなしでSSHできる必要がある

<img width="634" height="450" alt="image" src="https://github.com/user-attachments/assets/06727a1b-9d53-44a4-a1cd-5f122c87db25" />

鍵をコピー
```
ssh-copy-id rhel10
```
→ パスワードなしでrhel10へSSHできるようになる

RHEL10（KVMホスト）へ接続

<img width="1146" height="640" alt="image" src="https://github.com/user-attachments/assets/405e07d5-8568-41c2-be40-fbf36cd4dd9e" />

接続完了

<img width="522" height="529" alt="image" src="https://github.com/user-attachments/assets/7f363d06-2405-4b89-87e2-f1764b91b558" />
