# 演習 2.1 - SSL/TLSテンプレートの作成

## 目次

- [本演習の目的](#本演習の目的)
- [SSL/TLSテンプレートを作成するPlaybookの作成](#SSL/TLSテンプレートを作成するPlaybookの作成)
- [SSL/TLSテンプレートを作成するPlaybookの実行](#SSL/TLSテンプレートを作成するPlaybookの実行)

# 本演習の目的

本演習では、Virtual-ServerでSSL/TLSを終端するために、`a10_slb_template_client_ssl`モジュールを利用して、インポートした自己証明書と秘密鍵を元にSSL/TLSテンプレートを作成します。


# SSL/TLSテンプレートを作成するPlaybookの作成

SSL/TLSテンプレートを作成するために、Ansible実行用サーバーのplaybookディレクトリで、`a10_slb_template_client_ssl_create.yaml`という名前でPlaybookを作成します。
このPlaybookでは、Ansibleモジュールとして`a10_slb_template_client_ssl`を利用します。

```
[root@ansible playbook]# vi a10_slb_template_client_ssl_create.yaml
```

インポートした証明書と秘密鍵を元にテンプレートを構成し、構成変更後に構成の保存をするためのタスクを追加して連続実行するようにします。

``` 
---
- hosts: 10.255.0.1
  connection: local
  gather_facts: no

  vars:
    a10_host: "10.255.0.1"
  tasks:
  - name: Configure client-ssl template
    a10_slb_template_client_ssl:
      a10_host: "{{ a10_host }}"
      a10_port: "{{ a10_port }}"
      a10_username: "{{ a10_username }}"
      a10_password: "{{ a10_password }}"
      a10_protocol: "{{ a10_protocol }}"
      name: "ssl1"
      cert_str: "server.crt"
      key_str: "server.key"
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

- `name: "ssl1"`は、モジュールのパラメーターで、`a10_slb_template_client_ssl`で構成されるSSL/TLSテンプレートの名前を指定します。
- `cert_str: "server.crt"`は、モジュールのパラメーターで、`a10_slb_template_client_ssl`で構成されるSSL/TLSテンプレートで利用する証明書のThunder上でのファイル名を指定します。
- `key_str: "server.key"`は、モジュールのパラメーターで、`a10_slb_template_client_ssl`で構成されるSSL/TLSテンプレートで利用する証明書に対応する秘密鍵のThunder上でのファイル名を指定します。

ここまで記述したところで、Playbookを保存しコマンドラインに戻ります。

# SSL/TLSテンプレートを作成するPlaybookの実行

このPlaybookを実行すると、以下のようになります。

```
[root@ansible playbook]# ansible-playbook -i hosts a10_slb_template_client_ssl_create.yaml

PLAY [10.255.0.1] *********************************************************************************************************************************

TASK [Configure client-ssl template] **************************************************************************************************************
 [WARNING]: Module did not set no_log for forward_passphrase

 [WARNING]: Module did not set no_log for key_alt_passphrase

 [WARNING]: Module did not set no_log for fp_alt_passphrase

 [WARNING]: Module did not set no_log for key_passphrase

 [WARNING]: Module did not set no_log for key_shared_passphrase

changed: [10.255.0.1]

TASK [Write memory] *******************************************************************************************************************************
changed: [10.255.0.1]

PLAY RECAP ****************************************************************************************************************************************
10.255.0.1                 : ok=2    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

```

1つ目のタスク（Configure client-ssl template）でSSL/TLSテンプレートが作成され、2つ目のタスク（Write memory）で構成が保存されます。

vThunderの設定を確認してみましょう。

```
vThunder#show running-config
!Current configuration: 508 bytes
!Configuration last updated at 18:13:58 IST Thu Sep 12 2019
!Configuration last saved at 18:14:02 IST Thu Sep 12 2019
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
slb template client-ssl ssl1
  cert server.crt
  key server.key
!
slb virtual-server vip1 192.168.1.100
  port 80 http
    source-nat pool p1
    service-group sg1
!
sflow setting local-collection
!
sflow collector ip 127.0.0.1 6343
!
!
end
!Current config commit point for partition 0 is 0 & config mode is classical-mode
```

新たにslb template client-ssl　ssl1として、SSL/TLSテンプレートが作成されています。
次の演習では、Virtual-ServerにSSL/TLSテンプレートを紐づけ、HTTPSでのサービス提供を実現します。

本演習は以上となります。  [トレーニングガイドに戻る](../README.ja.md)
