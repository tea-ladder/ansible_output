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

- 各Playbook で必要な情報がある場合、Playbook内にsetup モジュールを用いて必要なgather subset / 特定のfacts を指定する
  - [ansible document](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/setup_module.html)

```
- name: Collect only facts returned by facter
  ansible.builtin.setup:
    gather_subset:
      - '!all'
      - '!any'
      - facter

- name: Collect only selected facts
  ansible.builtin.setup:
    filter:
      - 'ansible_distribution'
      - 'ansible_machine_id'
      - 'ansible_*_mb'
```

- 上記のようにすることで、各playbook内のgathering factに関する実行時間を改善することができる

### 引用

- [1] https://www.ansible.com/blog/ansible-performance-tuning
- [2] https://gryzli.info/2019/02/21/tuning-ansible-for-maximum-performance/
- [3] https://www.linkedin.com/pulse/how-speed-up-ansible-playbooks-drastically-lionel-gurret
- [4] https://blog.mosuke.tech/entry/2015/12/01/181304/



# ansible での2重ループについて

## 概要

- ループ処理はloop statement を用いて実行する

```
---
- name: debug a user account.
  hosts: all
  become: yes

  tasks:
    - name: user account
      debug:
        msg: "{{ item }}"
      loop:
        - taro
        - jiro
        - hanako
```

- 2重ループを実行する際は以下のようにtaskをincludeし、loopcontrolを利用する

```
- include_tasks: inner.yml
  loop:
    - taro
    - jiro
    - hanako
  loop_control:
    loop_var: outer_item 
```
```
- debug:
    msg:
      - "{{ outer_item }} {{ inner_item }}"
  loop:
    - yamada
    - satou
  loop_control:
    loop_var: inner_item
```

- 実行結果は以下のようになり、外側で指定した変数とimport_taskで指定した変数で２重ループされていることが確認できる

```
$ ansible-playbook -l webserver -i inventory.ini webserver.yml -v
Using /etc/ansible/ansible.cfg as config file

PLAY [webserver] ********************************************************************************************************************

TASK [Gathering Facts] **************************************************************************************************************
ok: [localhost]

TASK [double_loop : include_tasks] **************************************************************************************************
included: /home/ansible/ansible/roles/double_loop/tasks/inner.yml for localhost
included: /home/ansible/ansible/roles/double_loop/tasks/inner.yml for localhost
included: /home/ansible/ansible/roles/double_loop/tasks/inner.yml for localhost

TASK [double_loop : debug] **********************************************************************************************************
ok: [localhost] => (item=yamada) => {
    "msg": [
        "taro yamada"
    ]
}
ok: [localhost] => (item=satou) => {
    "msg": [
        "taro satou"
    ]
}

TASK [double_loop : debug] **********************************************************************************************************
ok: [localhost] => (item=yamada) => {
    "msg": [
        "jiro yamada"
    ]
}
ok: [localhost] => (item=satou) => {
    "msg": [
        "jiro satou"
    ]
}

TASK [double_loop : debug] **********************************************************************************************************
ok: [localhost] => (item=yamada) => {
    "msg": [
        "hanako yamada"
    ]
}
ok: [localhost] => (item=satou) => {
    "msg": [
        "hanako satou"
    ]
}

PLAY RECAP **************************************************************************************************************************
localhost                  : ok=7    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```


### 引用

- [1]  https://hutene.com/ansible_loop/
- [2]  https://zenn.dev/y_mrok/books/ansible-no-tsukaikata/viewer/chapter13
