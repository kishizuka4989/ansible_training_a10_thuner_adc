# 演習 1.5 - NATプールの設定

## 目次

- [本演習の目的](#本演習の目的)
- [NATプールを構成するPlaybookの作成](#NATプールを構成するPlaybookの作成)
- [NATプールを構成するPlaybookの実行](#NATプールを構成するPlaybookの実行)

# 本演習の目的

本演習では、`a10_ip_nat_pool`モジュールを利用し、Serverにアクセスする際のSNATで利用するNATプールの設定を行います。

# NATプールを構成するPlaybookの作成

NATプールを設定するために、Ansible実行用サーバーのplaybookディレクトリで、`a10_ip_nat_pool_create.yaml`という名前でPlaybookを作成します。
このPlaybookでは、Ansibleモジュールとして`a10_ip_nat_pool`を利用します。

```
[root@ansible playbook]# vi a10_ip_nat_pool_create.yaml
```

NATプールとして192.168．2.100のアドレス1つを割り当て、構成変更後に構成の保存をするためのタスクを追加して連続実行するようにします。

``` 
---
- hosts: 10.255.0.1
  connection: local
  gather_facts: no

  vars:
    a10_host: "10.255.0.1"
  tasks:
  - name: Configure NAT Pool
    a10_ip_nat_pool:
      a10_host: "{{ a10_host }}"
      a10_port: "{{ a10_port }}"
      a10_username: "{{ a10_username }}"
      a10_password: "{{ a10_password }}"
      a10_protocol: "{{ a10_protocol }}"
      pool_name: "p1"
      start_address: "192.168.2.100"
      end_address: "192.168.2.100"
      netmask: "/24"
      state: present

  - name: Write memory
    a10_write_memory:
      a10_host: "{{ a10_host }}"
      a10_port: "{{ a10_port }}"
      a10_username: "{{ a10_username }}"
      a10_password: "{{ a10_password }}"
      a10_protocol: "{{ a10_protocol }}"
      state: present
      partition: all
```

- `pool_name: "p1"`は、モジュールのパラメーターで、`a10_ip_nat_pool`で設定するNATプールの名前を指定します。
- `start_address: "192.168.2.100"`は、モジュールのパラメーターで、`a10_ip_nat_pool`で利用するIPアドレス帯の開始点となるIPアドレスを指定します。
- `end_address: "192.168.2.100"`は、モジュールのパラメーターで、`a10_ip_nat_pool`で利用するIPアドレス帯の終了点となるIPアドレスを指定します。
- `netmask: "/24"`は、モジュールのパラメーターで、`a10_ip_nat_pool`で利用するIPアドレス帯のサブネットマスクを指定します。

ここまで記述したところで、Playbookを保存しコマンドラインに戻ります。

# NATプールを構成するPlaybookの実行

このPlaybookを実行すると、以下のようになります。

```
[root@ansible playbook]# ansible-playbook -i hosts a10_ip_nat_pool_create.yaml

PLAY [10.255.0.1] *********************************************************************************************************************************

TASK [Configure NAT Pool] *************************************************************************************************************************
changed: [10.255.0.1]

TASK [Write memory] *******************************************************************************************************************************
changed: [10.255.0.1]

PLAY RECAP ****************************************************************************************************************************************
10.255.0.1                 : ok=2    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

```

2つのタスクが連続実行され、1つ目のタスク（Configure　NAT Pool）でNATプールが構成され、2つ目のタスク（Write memory）で変更した構成が起動用の設定として保存されたことがわかります。

vThunderの設定を確認してみましょう。

```
vThunder#show running-config
!Current configuration: 441 bytes
!Configuration last updated at 16:09:55 IST Thu Sep 12 2019
!Configuration last saved at 16:09:58 IST Thu Sep 12 2019
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
ip nat pool p1 192.168.2.100 192.168.2.100 netmask /24
!
slb server s1 192.168.2.1
  port 80 tcp
!
slb server s2 192.168.2.2
  port 80 tcp
!
slb service-group sg1 tcp
  member s1 80
  member s2 80
!
sflow setting local-collection
!
sflow collector ip 127.0.0.1 6343
!
!
end
!Current config commit point for partition 0 is 0 & config mode is classical-mode
```

新たに`ip nat pool p1 192.168.2.100 192.168.2.100 netmask /24`が設定されています。

再度同じPlaybookを実行してみます。
```
[root@ansible playbook]# ansible-playbook -i hosts a10_ip_nat_pool_create.yaml

PLAY [10.255.0.1] *********************************************************************************************************************************

TASK [Configure NAT Pool] *************************************************************************************************************************
ok: [10.255.0.1]

TASK [Write memory] *******************************************************************************************************************************
changed: [10.255.0.1]

PLAY RECAP ****************************************************************************************************************************************
10.255.0.1                 : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

```

NATプールを設定する部分は冪等性が保たれていることがわかります。

これで、NATプールの追加が完了しました。
次の演習では、サーバー負荷分散のためのVirtual-Serverの設定を行います。

本演習は以上となります。  [トレーニングガイドに戻る](../README.ja.md)
