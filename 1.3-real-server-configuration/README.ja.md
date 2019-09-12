# 演習 1.3 - Real Serverの追加

## 目次

- [本演習の目的](#本演習の目的)
- [Real Serverを構成するPlaybookの作成](#Real Serverを構成するPlaybookの作成)
- [Real Serverを構成するPlaybookの実行](#Real Serverを構成するPlaybookの実行)

# 本演習の目的

本演習では、`a10_slb_server`モジュールを利用し、サーバー負荷分散の対象となる実サーバー（Real Server）の設定を行います。

# Real Serverを構成するPlaybookの作成

Real Serverを設定するために、Ansible実行用サーバーのplaybookディレクトリで、`a10_slb_servers_create.yaml`という名前でPlaybookを作成します。
このPlaybookでは、Ansibleモジュールとして`a10_slb_server`を利用します。

```
[root@ansible playbook]# vi a10_slb_servers_create.yaml
```

2台のWebサーバーの設定を連続して実施し、構成変更後に構成の保存をするためのタスクを追加して連続実行するようにします。

``` 
---
- hosts: 10.255.0.1
  connection: local
  gather_facts: no

  vars:
    a10_host: "10.255.0.1"
  tasks:
  - name: Configure real server
    a10_slb_server:
      a10_host: "{{ a10_host }}"
      a10_port: "{{ a10_port }}"
      a10_username: "{{ a10_username }}"
      a10_password: "{{ a10_password }}"
      name: "{{ item.name }}"
      host: "{{ item.host }}"
      port_list:
        - port_number: "{{ item.port_number }}"
          protocol: "{{ item.protocol }}"
      state: present
      partition: shared
    with_items:
      - { name: "s1", host: "192.168.2.1", port_number: 80, protocol: "tcp" }
      - { name: "s2", host: "192.168.2.2", port_number: 80, protocol: "tcp" }

  - name: Write memory
    a10_write_memory:
      a10_host: "{{ a10_host }}"
      a10_username: "{{ a10_username }}"
      a10_password: "{{ a10_password }}"
      a10_port: "{{ a10_port }}"
      state: present
      partition: all

```

- `name: "{{ item.name }}"`は、モジュールのパラメーターで、`a10_slb_server`で設定するReal serverの名前を指定します。
- `host: "{{ item.host }}"`は、モジュールのパラメーターで、`a10_slb_server`で設定するReal serverのIPアドレスを指定します。
- `port_list:`は、リスト形式のモジュールのパラメーターで、`a10_slb_server`で設定するReal serverのサービスがListenするポート番号を`port_number`、プロトコルを`protocol`で指定します。

ここまで記述したところで、Playbookを保存しコマンドラインに戻ります。

# Real Serverを構成するPlaybookの実行

このPlaybookを実行すると、以下のようになります。

```
[root@ansible playbook]# ansible-playbook -i hosts a10_slb_servers_create.yaml

PLAY [10.255.0.1] *********************************************************************************************************************************

TASK [Configure real server] **********************************************************************************************************************
changed: [10.255.0.1] => (item={'name': 's1', 'host': '192.168.2.1', 'port_number': 80, 'protocol': 'tcp'})
changed: [10.255.0.1] => (item={'name': 's2', 'host': '192.168.2.2', 'port_number': 80, 'protocol': 'tcp'})

TASK [Write memory] *******************************************************************************************************************************
changed: [10.255.0.1]

PLAY RECAP ****************************************************************************************************************************************
10.255.0.1                 : ok=2    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

```

2つのタスクが連続実行され、1つ目のタスク（Configure　real server）でserver s1とs2が構成され、2つ目のタスク（Write memory）で変更した構成が起動用の設定として保存されたことがわかります。

vThunderの設定を確認してみましょう。

```
vThunder#show running-config
!Current configuration: 385 bytes
!Configuration last updated at 15:16:42 IST Thu Sep 12 2019
!Configuration last saved at 15:16:45 IST Thu Sep 12 2019
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
slb server s1 192.168.2.1
  port 80 tcp
!
slb server s2 192.168.2.2
  port 80 tcp
!
sflow setting local-collection
!
sflow collector ip 127.0.0.1 6343
!
!
end
!Current config commit point for partition 0 is 0 & config mode is classical-mode
```

slb serverとしてs1とs2が設定され、それぞれport 80 tcpでListenしています。

ここで、`show slb server`をvThunderで実行します。

```
vThunder#show slb server
Total Number of Servers configured: 2
Total Number of Services configured: 2
                   Current = Current Connections, Total = Total Connections
                   Fwd-pkt = Forward packets, Rev-pkt = Reverse packets
Service                   Current    Total      Fwd-pkt    Rev-pkt    Peak-conn  State
---------------------------------------------------------------------------------------
s1:80/tcp                 0          0          0          0          0          Up
s1: Total                 0          0          0          0          0          Up

s2:80/tcp                 0          0          0          0          0          Up
s2: Total                 0          0          0          0          0          Up
```

ヘルスチェックの結果、どちらのサーバーもそのポートもStateがUpになっています。

再度同じPlaybookを実行してみます。
```
[root@ansible playbook]# ansible-playbook -i hosts a10_slb_servers_create.yaml

PLAY [10.255.0.1] *********************************************************************************************************************************

TASK [Configure real server] **********************************************************************************************************************
ok: [10.255.0.1] => (item={'name': 's1', 'host': '192.168.2.1', 'port_number': 80, 'protocol': 'tcp'})
ok: [10.255.0.1] => (item={'name': 's2', 'host': '192.168.2.2', 'port_number': 80, 'protocol': 'tcp'})

TASK [Write memory] *******************************************************************************************************************************
changed: [10.255.0.1]

PLAY RECAP ****************************************************************************************************************************************
10.255.0.1                 : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

```

Real Serverを設定する部分は冪等性が保たれていることがわかります。

これで、Real Serverの追加が完了しました。
次の演習では、サーバー負荷分散のためのService Groupの設定を行います。

本演習は以上となります。  [トレーニングガイドに戻る](../README.ja.md)
