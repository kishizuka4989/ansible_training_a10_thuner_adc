# 演習 1.2 - データポートのenable

## 目次

- [本演習の目的](#本演習の目的)
- [データポートをenableするPlaybookの作成](#データポートをenableするPlaybookの作成)
- [データポートをenableするPlaybookの実行](#データポートをenableするPlaybookの実行)
- [クライアントとWebサーバーからのvThunderへの疎通確認](#クライアントとWebサーバーからのvThunderへの疎通確認)

# 本演習の目的

本演習では、`a10_interface_ethernet`モジュールを利用し、vThunderのデータポートを利用可能にする（enableする）設定を行います。
データポートをenableにすることで、クライアントとVE 10、WebサーバーとVE 20との間でネットワークの疎通確認をすることができます。

データポートがenableになっていないことを確認するには、vThunderで以下のコマンドを実行します。

```
vThunder#show interfaces brief
Port    Link  Dupl  Speed  Trunk Vlan MAC             IP Address          IPs  Name
------------------------------------------------------------------------------------
mgmt    Up    Full  1000   N/A   N/A  2cc2.6064.4ec4  10.255.0.1/16         1
1       Disb  None  None   none  10   2cc2.603c.b0de  0.0.0.0/0             0
2       Disb  None  None   none  20   2cc2.6044.0a45  0.0.0.0/0             0
ve10    Down  N/A   N/A    N/A   10   2cc2.603c.b0de  192.168.1.254/24      1
ve20    Down  N/A   N/A    N/A   20   2cc2.6044.0a45  192.168.2.254/24      1

Global Throughput:0 bits/sec (0 bytes/sec)
Throughput:0 bits/sec (0 bytes/sec)
```

Port 1も2もDisb（disable）になっており、ve10もve20もDownしていることが確認できます。


# データポートをenableするPlaybookの作成

vThunderのデータポートを利用可能にするために、Ansible実行用サーバーのplaybookディレクトリで、`a10_interfaces_ethernet_enable.yaml`という名前でPlaybookを作成します。
このPlaybookでは、Ansibleモジュールとして`a10_interface_ethernet`を利用します。

```
[root@ansible playbook]# vi a10_interfaces_ethernet_enable.yaml
```

vThunderの持つ2つのデータポートを連続してenableにし、構成変更後に構成の保存をするためのタスクを追加して連続実行するようにします。

``` 
---
- hosts: 10.255.0.1
  connection: local
  gather_facts: no

  vars:
    a10_host: "10.255.0.1"
  tasks:
  - name: Enable ethernet interfaces
    a10_interface_ethernet:
      a10_host: "{{ a10_host }}"
      a10_port: "{{ a10_port }}"
      a10_username: "{{ a10_username }}"
      a10_password: "{{ a10_password }}"
      ifnum: "{{ item.ifnum }}"
      action: enable
      state: present
      partition: shared
    with_items:
      - { ifnum: 1 }
      - { ifnum: 2 }

  - name: Write memory
    a10_write_memory:
      a10_host: "{{ a10_host }}"
      a10_username: "{{ a10_username }}"
      a10_password: "{{ a10_password }}"
      a10_port: "{{ a10_port }}"
      state: present
      partition: all

```

- `ifnum: "{{ item.ifnum }}"`は、モジュールのパラメーターで、`a10_interface_ethernet`で設定するインターフェース（ethernet）のID番号を指定します。
- `action: enable`は、モジュールのパラメーターで、`a10_interface_ethernet`で設定するインターフェースに対する操作を指定します。

ここまで記述したところで、Playbookを保存しコマンドラインに戻ります。

# データポートをenableするPlaybookの作成

このPlaybookを実行すると、以下のようになります。

```
[root@ansible playbook]# ansible-playbook -i hosts a10_interfaces_ethernet_enable.yaml

PLAY [10.255.0.1] *********************************************************************************************************************************

TASK [Enable ethernet interfaces] *****************************************************************************************************************
changed: [10.255.0.1] => (item={'ifnum': 1})
changed: [10.255.0.1] => (item={'ifnum': 2})

TASK [Write memory] *******************************************************************************************************************************
changed: [10.255.0.1]

PLAY RECAP ****************************************************************************************************************************************
10.255.0.1                 : ok=2    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

```

2つのタスクが連続実行され、1つ目のタスク（Enable ethernet interfaces）でethernet 1と2がenableになり、2つ目のタスク（Write memory）で変更した構成が起動用の設定として保存されたことがわかります。

vThunderの設定を確認してみましょう。

```
vThunder#show running-config
!Current configuration: 385 bytes
!Configuration last updated at 14:44:38 IST Thu Sep 12 2019
!Configuration last saved at 14:44:41 IST Thu Sep 12 2019
!64-bit Advanced Core OS (ACOS) version 4.1.4-GR1, build 78 (Jan-18-2019,16:02)
!
multi-config enable
!
terminal idle-timeout 0
!
vlan 10
  untagged ethernet 1
  router-interface ve 10
!
vlan 20
  untagged ethernet 2
  router-interface ve 20
!
interface management
  ip address 10.255.0.1 255.255.0.0
  ip default-gateway 10.255.255.1
!
interface ethernet 1
  enable
!
interface ethernet 2
  enable
!
interface ve 10
  ip address 192.168.1.254 255.255.255.0
!
interface ve 20
  ip address 192.168.2.254 255.255.255.0
!
!
sflow setting local-collection
!
sflow collector ip 127.0.0.1 6343
!
!
end
!Current config commit point for partition 0 is 0 & config mode is classical-mode
```

interface ethernet 1と2がenableになっています。

ここで、先ほどと同じコマンド`show interfaces brief`をvThunderで実行します。

```
vThunder#show interfaces brief
Port    Link  Dupl  Speed  Trunk Vlan MAC             IP Address          IPs  Name
------------------------------------------------------------------------------------
mgmt    Up    Full  1000   N/A   N/A  2cc2.6064.4ec4  10.255.0.1/16         1
1       Up    Full  10000  none  10   2cc2.603c.b0de  0.0.0.0/0             0
2       Up    Full  10000  none  20   2cc2.6044.0a45  0.0.0.0/0             0
ve10    Up    N/A   N/A    N/A   10   2cc2.603c.b0de  192.168.1.254/24      1
ve20    Up    N/A   N/A    N/A   20   2cc2.6044.0a45  192.168.2.254/24      1

Global Throughput:0 bits/sec (0 bytes/sec)
Throughput:0 bits/sec (0 bytes/sec)
```
ethernet 1と2がUpになり、ve10とve20もUpになっていることが確認できます。

再度同じPlaybookを実行してみます。
```
[root@ansible playbook]# ansible-playbook -i hosts a10_interfaces_ethernet_enable.yaml

PLAY [10.255.0.1] *********************************************************************************************************************************

TASK [Enable ethernet interfaces] *****************************************************************************************************************
ok: [10.255.0.1] => (item={'ifnum': 1})
ok: [10.255.0.1] => (item={'ifnum': 2})

TASK [Write memory] *******************************************************************************************************************************
changed: [10.255.0.1]

PLAY RECAP ****************************************************************************************************************************************
10.255.0.1                 : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

```

データポートをenableする部分は冪等性が保たれていることがわかります。

# クライアントとWebサーバーからのvThunderへの疎通確認

データポートが利用可能になったので、クライアントからvThunderへの疎通確認を行うことができます。
CentOSクライアントにログインし、vThunderのve10（192.168．1.254）に対して`ping`を実行すると応答が返ってくるので、疎通を確認できます（Windows 10クライアントからでも確認できます）。

```
[root@client ~]# ping 192.168.1.254
PING 192.168.1.254 (192.168.1.254) 56(84) bytes of data.
64 bytes from 192.168.1.254: icmp_seq=1 ttl=64 time=22.4 ms
64 bytes from 192.168.1.254: icmp_seq=2 ttl=64 time=13.1 ms
64 bytes from 192.168.1.254: icmp_seq=3 ttl=64 time=11.2 ms
64 bytes from 192.168.1.254: icmp_seq=4 ttl=64 time=13.2 ms
^C
--- 192.168.1.254 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3005ms
rtt min/avg/max/mdev = 11.265/15.020/22.418/4.345 ms
```

同様に、Webサーバーにログインし、vThunderのve20（192.168．2.254）に対して`ping`を実行すると応答が返ってくるので、疎通を確認できます。

```
[root@server01 ~]# ping 192.168.2.254
PING 192.168.2.254 (192.168.2.254) 56(84) bytes of data.
64 bytes from 192.168.2.254: icmp_seq=1 ttl=64 time=23.0 ms
64 bytes from 192.168.2.254: icmp_seq=2 ttl=64 time=17.2 ms
64 bytes from 192.168.2.254: icmp_seq=3 ttl=64 time=15.0 ms
64 bytes from 192.168.2.254: icmp_seq=4 ttl=64 time=13.0 ms
^C
--- 192.168.2.254 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3005ms
rtt min/avg/max/mdev = 13.073/17.118/23.055/3.740 ms
```

vThunderからも`ping`を実行すると応答が返ってくるので、疎通を確認できます（Windows 10クライアントは応答を返さない設定になっていますのでご注意ください）。

```
vThunder#ping 192.168.1.2
PING 192.168.1.2 (192.168.1.2) 56(84) bytes of data.
64 bytes from 192.168.1.2: icmp_seq=1 ttl=64 time=15.5 ms
64 bytes from 192.168.1.2: icmp_seq=2 ttl=64 time=21.9 ms
64 bytes from 192.168.1.2: icmp_seq=3 ttl=64 time=12.9 ms
64 bytes from 192.168.1.2: icmp_seq=4 ttl=64 time=15.6 ms
64 bytes from 192.168.1.2: icmp_seq=5 ttl=64 time=17.7 ms

--- 192.168.1.2 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 4005ms
rtt min/avg/max/mdev = 12.916/16.758/21.935/3.009 ms
vThunder#ping 192.168.2.2
PING 192.168.2.2 (192.168.2.2) 56(84) bytes of data.
64 bytes from 192.168.2.2: icmp_seq=1 ttl=64 time=29.7 ms
64 bytes from 192.168.2.2: icmp_seq=2 ttl=64 time=12.6 ms
64 bytes from 192.168.2.2: icmp_seq=3 ttl=64 time=11.5 ms
64 bytes from 192.168.2.2: icmp_seq=4 ttl=64 time=13.8 ms
64 bytes from 192.168.2.2: icmp_seq=5 ttl=64 time=15.7 ms

--- 192.168.2.2 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 4005ms
rtt min/avg/max/mdev = 11.573/16.722/29.750/6.661 ms
```

これで、データポートのenableが完了しました。
次の演習では、サーバー負荷分散のためのReal Serverの設定を行います。

本演習は以上となります。  [トレーニングガイドに戻る](../README.ja.md)
