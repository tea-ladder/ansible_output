# ansible performance turning

- ansible でのパフォーマンスチューニングに関して記載をする

## 概要

1. ネットワークリソースを最適化する
  1. パッケージマネージャのトランザクションをまとめる
2. asnible コントローラーとクライアントの接続を最適化する
  1. openssh の機能である「pipelining」を利用する
3. Default で有効化されているGathering Factの無効化

### パッケージマネージャのトランザクションをまとめる

- 引用[3]にある通りパッケージマネージャのタスクをなるべくまとめるようにする

```
[変更前]
- name: install the latest version of nginx
  yum:
    name: nginx
    state: latest

- name: install the latest version of postgresql
  yum:
    name: postgresql
    state: latest

- name: install the latest version of postgresql-server
  yum:
    name: postgresql-server
    state: latest
```

```
[変更後]
- name: Install a list of packages
  yum:
    name:
      - nginx
      - postgresql
      - postgresql-server
    state: present
```

## openssh の機能である「pipelining」を利用する

- ansible.cfgに対して、下記の通り設定することでpipelining機能を有効化することができる
  - ansibleは各taskを実行するごとにsshのセッションを確立させるが、pipeliningを有効にすることでセッションを使いまわすことが可能となるため、実行速度が速くなる


```
[defaults]
transport = ssh

[ssh_connection]
pipeline = true
ssh_args = -o ControlMaster=auto -o ControlPersist=300s
```

## Default で有効化されているGathering Factの無効化

- ansible.cfg に対して、下記の通り設定しGathering Factをデフォルトで実施しないようにする

```
[defaults]
gathering=explicit
```

- 各Playbook で必要な情報がある場合、Playbook内

### 引用

- [1]:https://www.ansible.com/blog/ansible-performance-tuning
- [2]:https://gryzli.info/2019/02/21/tuning-ansible-for-maximum-performance/
- [3]:https://www.linkedin.com/pulse/how-speed-up-ansible-playbooks-drastically-lionel-gurret
- [4]:https://blog.mosuke.tech/entry/2015/12/01/181304/
