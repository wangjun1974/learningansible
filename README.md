# learningansible
Learning Ansible

### 课程链接 
https://learning.redhat.com/course/view.php?id=1042

### Style Guide
Ansible Best Practise Style Guide
可以参考<br>
https://github.com/openshift/openshift-ansible/blob/master/docs/style_guide.adoc

### Ansible playbooks and roles examples

Red Hat Middleware Playbooks<br>
https://github.com/redhat-cop/ansible-middleware-playbooks

### Day1

ansible inventory
special group (all, ungrouped)
ansible-inventory -i <inventory> --graph
handler
block
rescue
always

### Ansible Playbook最佳实践

#### Inventory最佳实践
使用可读性强的名称构建inventory
```
Poor                       Better
10.1.2.75                   db1 ansible_host=10.1.2.75
10.1.5.45                   db2 ansible_host=10.1.5.45

w14301.acme.com             web1 ansible_host=w14301.acme.com
w17802.acme.com             web2 ansible_host=w17802.acme.com
```

inventory group 的例子
```
[db]              [east]          [dev]
db[1:4]           db1             db1
                  web1            web1
[web]             db3
web[1:4]          web3            [testing]
                                  db3
                  [west]          web3
                  db2
                  web2            [prod]
                  db4             db2
                  web4            web2
                                  db4
                                  web4
```

限制执行对象
```
$ ansible-playbook site.yml --limit 'web'         # Only group web
$ ansible-playbook site.yml --limit 'web,db3'     # Group web and db3
$ ansible-playbook site.yml --limit 'all:!prod'   # All non group prod
```

#### YAML 最佳实践

推荐的格式
```
- name: install telegraf
  yum:
    name: telegraf-{{ telegraf_version }}
    state: present
    update_cache: yes
    enablerepo: telegraf
  notify: restart telegraf

- name: configure telegraf
  template:
    src: telegraf.conf.j2
    dest: /etc/telegraf/telegraf.conf
  notify: restart telegraf
```

不推荐的格式
```
- name: install telegraf
  yum: name=telegraf-{{ telegraf_version }} state=present update_cache=yes enablerepo=telegraf
  notify: restart telegraf

- name: configure telegraf
  template: src=telegraf.conf.j2 dest=/etc/telegraf/telegraf.conf
  notify: restart telegraf
```

#### Plays and Tasks
* 使用name改善可读性
* 建议给plays, tasks清晰简洁且唯一性强的名称

```
- name: Installs and starts apache
  hosts: web

  tasks:

    - name: Install apache packages
      yum:
        name: httpd
        state: latest

    - name: Starts apache service
      service:
        name: httpd
        state: started
        enabled: yes
```

* 保持playbook简洁性，多个简单的playbook肯定比1个巨大的playbook好维护
* 分为多个playbook可降低维护复杂性
```
$ ls
acme_corp/
├── configure.yml
├── provision.yml
└── site.yml

$ cat site.yml
---
- import_playbook: provision.yml
- import_playbook: configure.yml
```

#### handlers最佳实践
* handler是inactive tasks
* handler需通过notify激活
* handler有全局唯一的名字
* handler通常情况下在active tasks执行完后执行
* handler如未被notify则不会执行
* handler如被notify将在play里的active tasks执行完后最后执行1次
* task meta: flush_handlers将触发handler立即执行
* handler可使用的模块与active task一样
* 常用的handler的使用场景是重启机器和重启服务

##### Handler的例子
```
tasks:

  - name: Configure demo for apache httpd
    template:
      src: demo.example.conf.j2
      dest: /etc/httpd/conf.d/demo.example.conf
    notify:
      - restart_apache

handlers:

  - name: restart_apache
    service:
      name: httpd
      state: restarted
```

#### Blocks最佳实践
* 在复杂playbook通常包含非常多tasks
* 这些tasks里有些task具有相关性
* 使用Block来把这些相关的tasks组织在一起
* Block的好处包括：改善可读性和在Block级别改善性能

#### Block错误处理的例子
```
tasks:
  - name: Attempt and gracefully roll back demo
    block:
      - debug:
          msg: "I execute normally"
      - command: /bin/false
      - debug:
          msg: "I never execute, due to the above task failing"
    rescue:
      - debug:
          msg: "I caught an error"
      - command: /bin/false
      - debug:
          msg: "I also never execute :-("
    always:
      - debug:
          msg: "this always executes"
```
* block的debug msg: "I execute normally"首先执行
* block的command '/bin/false'执行完后，msg: "I never execute, due to the above task failing"不会被执行
* 由于有rescue和always的存在，紧接着将执行debug msg: "I caught an error"
* rescue的command '/bin/false'执行完后，msg: "I also never execute :-("不会执行
* 周后会执行always的debug msg: "this always executes"

#### Block与Handler处理
```
 tasks:

   - name: Attempt and gracefull roll back demo
     block:
       - debug:
           msg: "I execute normally"
         notify: run me even after an error
       - command: /bin/false
     rescue:
       - name: make sure all handlers run
         meta: flush_handlers
  handlers:
    - name: run me even after an error
      debug:
        msg: "this handler runs even on error"
```
说明：
* notify设置未来执行handler(inactive tasks)'run me even after an error'
* command /bin/false导致block执行失败，此时如没有rescue或always则playbook在此终止
* 因为有rescue的存在，继续执行meta: flush_handlers
* meta: flush_handlers，触发handler 'run me even after an error'执行

#### Roles最佳实践
* 与playbook一样，为了实现易于维护，请保持roles的简洁性
* 简化roles的依赖
* 通过roles子目录保存项目内开发的roles
* 用ansible-galaxy init开发可共享的roles
* 使用ansible-galaxy安装roles
* 使用roles的文件如requirements.yml定义项目的外部依赖的roles
* 使用roles的特定版本

##### 外部roles的例子
```
myapp/
├── config.yml
├── provision.yml
├── roles
│   └── requirements.yml
└── setup.yml

$ ansible-galaxy install -r requirements.yml

$ cat requirements.yml

# from galaxy
- src: yatesr.timezone

# from GitHub
- src: https://github.com/bennojoy/nginx
  version: v1.4

# from a webserver, where the role is packaged in a tar.gz
- src: https://some.webserver.example.com/files/master.tar.gz
  name: http-role
```

##### Roles的执行顺序
* 默认roles在playbook的tasks之前执行
* 可在pre_tasks处执行所有roles还未执行的tasks
* 可在post_tasks处执行所有roles执行完之后的tasks

不同阶段的tasks的执行顺序
```
---
- hosts: remote.example.com
  pre_tasks:
    - shell: echo 'hello'
  roles:
    - role1
    - role2
  tasks:
    - shell: echo 'still busy'
  post_tasks:
    - shell: echo 'goodbye'
```

##### Templates
Templates可用于
* 保存未知在templates目录下
* 主要功能是变量替换
* 基于条件设置变量
* 简单条件判断
* 简单循环迭代

在使用Templates时需避免
* 在Templates里定义/管理变量
* 复杂条件
* 基于主机名设置条件

##### Ansible-Vault
* 使用ansible-vault加密重要数据
```
Create: ansible-vault create secret.yml
Encrypt: ansible-vault encrypt secret.yml
View: ansible-vault view secret.yml
Decrypt: ansible-vault decrypt secret.yml
```

* 运行使用ansible-vault加密的playbook时使用以下参数
```
--ask-vault-pass
--vault-password-file
```
例子<br>
https://www.digitalocean.com/community/tutorials/how-to-use-vault-to-protect-sensitive-ansible-data-on-ubuntu-16-04<br>
https://schoolofdevops.github.io/ultimate-ansible-bootcamp/ansible-vault/


### Memo
|hostname|ipaddr|
|---|---|
|test.example.com|192.168.0.1|
