# ansible  個人メモ

- このページはansibleの個人的な知識をメモするためのページ

## ansilbe とは

- 構成管理ツールと呼ばれるもの
  - 似たプロダクトにchef,pappet などがある

## ansibleのディレクトリ構成

- [公式ドキュメントのベストプラクティス](https://docs.ansible.com/ansible/2.9/user_guide/playbooks_best_practices.html#directory-layout)に従った構成で作成する

```
production                # inventory file for production servers
staging                   # inventory file for staging environment

group_vars/
   group1.yml             # here we assign variables to particular groups
   group2.yml
host_vars/
   hostname1.yml          # here we assign variables to particular systems
   hostname2.yml

library/                  # if any custom modules, put them here (optional)
module_utils/             # if any custom module_utils to support modules, put them here (optional)
filter_plugins/           # if any custom filter plugins, put them here (optional)

site.yml                  # master playbook
webservers.yml            # playbook for webserver tier
dbservers.yml             # playbook for dbserver tier

roles/
    common/               # this hierarchy represents a "role"
        tasks/            #
            main.yml      #  <-- tasks file can include smaller files if warranted
        handlers/         #
            main.yml      #  <-- handlers file
        templates/        #  <-- files for use with the template resource
            ntp.conf.j2   #  <------- templates end in .j2
        files/            #
            bar.txt       #  <-- files for use with the copy resource
            foo.sh        #  <-- script files for use with the script resource
        vars/             #
            main.yml      #  <-- variables associated with this role
        defaults/         #
            main.yml      #  <-- default lower priority variables for this role
        meta/             #
            main.yml      #  <-- role dependencies
        library/          # roles can also include custom modules
        module_utils/     # roles can also include custom module_utils
        lookup_plugins/   # or other types of plugins, like lookup in this case

    webtier/              # same kind of structure as "common" was above, done for the webtier role
    monitoring/           # ""
    fooapp/               # ""
```

## ansibleのコンポーネントについて

- ここでは大まかにansibleの主要なコンポーネントについて記載する
  - inventory : ansibleで管理する対象ホストやホストをまとめるグループを定義
  - host_vars/group_vars : host/groupの変数を定義
  - playbook : ansibleで実行する処理を定義

- 以下からはlocalhostにapacheをインストールする処理をサンプルに各ファイルを記載する

### inventory ファイル

```
[webserver]      # group の指定
localhost        # host の指定
```

### group_vars ファイル

- 検証したローカルサーバー上でインストール可能なhttpdのバージョンは以下の2つである

```
[ansible ~]$ sudo yum --showduplicates search httpd 2>/dev/null | grep 'Apache HTTP Server' | grep '^httpd-[0-9]'
httpd-2.4.6-95.el7.centos.x86_64 : Apache HTTP Server
httpd-2.4.6-97.el7.centos.x86_64 : Apache HTTP Server
[ansible ~]$ 
```

- ここではgroup_varsの検証のため、2.4.6を明示的にインストールするようgroup_varsにapacheのバージョンを定義する

```
httpd_version: 2.4.6
```


### playbook

- ここでは、httpdをインストールするroleを作成し、それをインポートするplaybookを作成

```
- hosts: webserver
  tasks:
    - import_role:
        name: apache
```

### role apache

- apache roleの処理は以下の通り

```
---
# tasks file for roles/apache
- name: apache install    # taskの処理名を任意に記載する
  become: yes             # admin 権限が必要な際、yesを指定する
  yum:                    # ansibleのbuiltin module であるyumを利用する。['公式ドキュメント'](https://docs.ansible.com/ansible/2.9/modules/yum_module.html)
    name: 'httpd-{{ httpd_version }}'   # group_varsの変数を用いてhttpd-2.4.6を指定
    state: present
```

### ansible の実行

- ansible を実行する際、コマンドで実施する
  - オプションの詳細は```ansible-playbook --help```を参照

```
[ansible ansible]$ ansible-playbook -l webserver -i inventory.ini webserver.yml -v
Using /etc/ansible/ansible.cfg as config file

PLAY [webserver] ***********************************************************************************************************************************

TASK [Gathering Facts] *****************************************************************************************************************************
ok: [localhost]

TASK [apache install] ******************************************************************************************************************************
changed: [localhost] => {"changed": true, "changes": {"installed": ["httpd-2.4.6"]}, "msg": "https://packages.cloud.google.com/yum/repos/cloud-sdk-el7-x86_64/repodata/repomd.xml: [Errno -1] repomd.xml signature could not be verified for google-cloud-sdk\nTrying other mirror.\n", "rc": 0, "results": ["Loaded plugins: fastestmirror\nRetrieving key from https://packages.cloud.google.com/yum/doc/yum-key.gpg\nRetrieving key from https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg\nLoading mirror speeds from cached hostfile\n * base: ftp.riken.jp\n * elrepo: ftp.ne.jp\n * epel: nrt.edge.kernel.org\n * extras: ftp.riken.jp\n * updates: ftp.riken.jp\nResolving Dependencies\n--> Running transaction check\n---> Package httpd.x86_64 0:2.4.6-97.el7.centos will be installed\n--> Processing Dependency: httpd-tools = 2.4.6-97.el7.centos for package: httpd-2.4.6-97.el7.centos.x86_64\n--> Processing Dependency: /etc/mime.types for package: httpd-2.4.6-97.el7.centos.x86_64\n--> Processing Dependency: libaprutil-1.so.0()(64bit) for package: httpd-2.4.6-97.el7.centos.x86_64\n--> Processing Dependency: libapr-1.so.0()(64bit) for package: httpd-2.4.6-97.el7.centos.x86_64\n--> Running transaction check\n---> Package apr.x86_64 0:1.4.8-7.el7 will be installed\n---> Package apr-util.x86_64 0:1.5.2-6.el7 will be installed\n---> Package httpd-tools.x86_64 0:2.4.6-97.el7.centos will be installed\n---> Package mailcap.noarch 0:2.1.41-2.el7 will be installed\n--> Finished Dependency Resolution\n\nDependencies Resolved\n\n================================================================================\n Package           Arch         Version                     Repository     Size\n================================================================================\nInstalling:\n httpd             x86_64       2.4.6-97.el7.centos         updates       2.7 M\nInstalling for dependencies:\n apr               x86_64       1.4.8-7.el7                 base          104 k\n apr-util          x86_64       1.5.2-6.el7                 base           92 k\n httpd-tools       x86_64       2.4.6-97.el7.centos         updates        93 k\n mailcap           noarch       2.1.41-2.el7                base           31 k\n\nTransaction Summary\n================================================================================\nInstall  1 Package (+4 Dependent packages)\n\nTotal download size: 3.0 M\nInstalled size: 10 M\nDownloading packages:\n--------------------------------------------------------------------------------\nTotal                                              2.6 MB/s | 3.0 MB  00:01     \nRunning transaction check\nRunning transaction test\nTransaction test succeeded\nRunning transaction\n  Installing : apr-1.4.8-7.el7.x86_64                                       1/5 \n  Installing : apr-util-1.5.2-6.el7.x86_64                                  2/5 \n  Installing : httpd-tools-2.4.6-97.el7.centos.x86_64                       3/5 \n  Installing : mailcap-2.1.41-2.el7.noarch                                  4/5 \n  Installing : httpd-2.4.6-97.el7.centos.x86_64                             5/5 \n  Verifying  : httpd-2.4.6-97.el7.centos.x86_64                             1/5 \n  Verifying  : apr-1.4.8-7.el7.x86_64                                       2/5 \n  Verifying  : mailcap-2.1.41-2.el7.noarch                                  3/5 \n  Verifying  : httpd-tools-2.4.6-97.el7.centos.x86_64                       4/5 \n  Verifying  : apr-util-1.5.2-6.el7.x86_64                                  5/5 \n\nInstalled:\n  httpd.x86_64 0:2.4.6-97.el7.centos                                            \n\nDependency Installed:\n  apr.x86_64 0:1.4.8-7.el7                     apr-util.x86_64 0:1.5.2-6.el7    \n  httpd-tools.x86_64 0:2.4.6-97.el7.centos     mailcap.noarch 0:2.1.41-2.el7    \n\nComplete!\n"]}

PLAY RECAP *****************************************************************************************************************************************
localhost                  : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   

[ansible ansible]$ rpm -qa | grep httpd
httpd-2.4.6-97.el7.centos.x86_64
httpd-tools-2.4.6-97.el7.centos.x86_64
[ansible ansible]$ 

```

## ansible のシステム変数について

- ansible では対象ホストのシステム変数を利用することができる
  - [公式ドキュメント](https://docs.ansible.com/ansible/2.9/user_guide/playbooks_variables.html#variables-discovered-from-systems-facts)
  - 下記コマンドで指定したhostのシステム変数の一覧を取得することができる

```
ansible localhost -m setup 
[サンプル]
[ansible ansible]$ ansible localhost -m setup 
〜省略〜
        "ansible_distribution": "CentOS", 
        "ansible_distribution_file_parsed": true, 
        "ansible_distribution_file_path": "/etc/redhat-release", 
        "ansible_distribution_file_variety": "RedHat", 
        "ansible_distribution_major_version": "7", 
        "ansible_distribution_release": "Core", 
        "ansible_distribution_version": "7.4", 
```

## ansible の処理の場合分けについて

- ansible では実行する処理(task)を、when ステートメントにより処理をスキップしたりすることができる
  - [公式ドキュメント](https://docs.ansible.com/ansible/2.9/user_guide/playbooks_conditionals.html#the-when-statement)

### apache role の場合分け

- 先ほどのapache role に対して、when ステートメント及びシステム変数を利用した場合分けをするよう修正する
  - 以下では、Centosならyumモジュールを利用し、Ubuntuであればaptモジュールを利用しapacheをインストールする

```
---
# tasks file for roles/apache
- name: apache install for CentOS
  become: yes
  yum:
    name: 'httpd-{{ httpd_version }}'
    state: present
  when: ansible_distribution == 'CentOS'

- name: apache install for Ubuntu
  become: yes
  apt:
    name: 'apache2'
    state: present
	when: ansible_distribution == 'Ubuntu'
```


- 実行するとtask 「apache install for Ubuntu」がスキップされていることがわかる

```
[ansible ansible]$ ansible-playbook -l webserver -i inventory.ini webserver.yml -v
Using /etc/ansible/ansible.cfg as config file

PLAY [webserver] ***********************************************************************************************************************************

TASK [Gathering Facts] *****************************************************************************************************************************
ok: [localhost]

TASK [apache install for CentOS] *******************************************************************************************************************
ok: [localhost] => {"changed": false, "msg": "", "rc": 0, "results": ["httpd-2.4.6-97.el7.centos.x86_64 providing httpd-2.4.6 is already installed"]}

TASK [apache install for Ubuntu] *******************************************************************************************************************
skipping: [localhost] => {"changed": false, "skip_reason": "Conditional result was False"}

PLAY RECAP *****************************************************************************************************************************************
localhost                  : ok=2    changed=0    unreachable=0    failed=0    skipped=1    rescued=0    ignored=0   

[ansible ansible]$
```
