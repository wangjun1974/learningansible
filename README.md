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

##### 变量和变量名
变量
* Separate logic (tasks) from variables and reduce repetitive patterns

变量名
* Makes plays more readable and helps avoid variable name conflicts
* Use descriptive, unique, human-meaningful names for variables
* Prefix role variables with role names
```
haproxy_max_keepalive: 25
haproxy_port: 80
tomcat_port: 8080
```

## Day2

### Windows Integration

https://docs.ansible.com/ansible/latest/user_guide/windows_winrm.html
https://docs.ansible.com/ansible/latest/user_guide/windows_setup.html
https://digitalist.global/talks/winrmansible/
https://www.unixarena.com/2019/04/ansible-configure-windows-servers-as-ansible-client-winrm.html/

PowerShell脚本用来准备Windows主机
https://github.com/ansible/ansible/blob/devel/examples/scripts/ConfigureRemotingForAnsible.ps1

为Ansible准备Windows主机
https://docs.ansible.com/ansible/2.5/user_guide/windows_setup.html

最新的Windows Ansible Module
http://docs.ansible.com/ansible/latest/list_of_windows_modules.html

#### Labs
安装ansible
```
yum install -y ansible
```

生成ansible inventory文件
```
cat > /etc/ansible/hosts << EOF

[GenericExample:vars]

###########################################################################
### Ansible Vars
###########################################################################
timeout=60
ansible_become=yes
ansible_user=ec2-user


[GenericExample:children]
towers
windows
support

[towers]
## These are the towers
tower1.d7da.internal public_host_name=tower1.d7da.example.opentlc.com ssh_host=tower1.d7da.internal
tower2.d7da.internal public_host_name=tower2.d7da.example.opentlc.com ssh_host=tower2.d7da.internal
tower3.d7da.internal public_host_name=tower3.d7da.example.opentlc.com ssh_host=tower3.d7da.internal


[windows]
## These are the activedirectory servers
ad1.d7da.internal ssh_host=ad1.d7da.example.opentlc.com ansible_password=jVMijRwLbI02gFCo2xkjlZ9lxEA7bm7zgg==


## These are the supporthosts
[support]
support1.d7da.internal ssh_host=support1.d7da.internal
support2.d7da.internal ssh_host=support2.d7da.internal

[windows:vars]
ansible_connection=winrm
ansible_user=Administrator
ansible_winrm_server_cert_validation=ignore
ansible_become=false

EOF
```

检查
```
ansible all --list-hosts
  hosts (6):
    tower1.d7da.internal
    tower2.d7da.internal
    tower3.d7da.internal
    ad1.d7da.internal
    support1.d7da.internal
    support2.d7da.internal
```

安装Ansible管理Windows所需软件包
```
yum install -y python-devel krb5-devel krb5-libs krb5-workstation python-pip gcc
```

安装版本大于等于0.2.2的pywinrm
```
pip install pywinrm==0.2.2
```

检查windows主机可访问
```
ansible windows -m win_ping
/usr/lib/python2.7/site-packages/urllib3/connectionpool.py:1004: InsecureRequestWarning: Unverified HTTPS request is being made. Adding certificate verification is strongly advised. See: https://urllib3.readthedocs.io/en/latest/advanced-usage.html#ssl-warnings
  InsecureRequestWarning,
ad1.d7da.internal | SUCCESS => {
    "changed": false, 
    "ping": "pong"
}
```

创建github项目，克隆项目
```
git clone https://github.com/wangjun1974/windowsad.git
cd windowsad
```

创建目录
```
mkdir -p roles/win_ad_install/tasks
mkdir -p roles/win_ad_install/defaults
```

生成roles默认变量
```
cat > roles/win_ad_install/defaults/main.yml << EOF
---
ad_domain_name: ad1.${GUID}.example.opentlc.com
ad_safe_mode_password: "{{ansible_password}}"
ad_admin_user: "admin@{{ad_domain_name}}"
ad_admin_password: "{{ansible_password}}"
EOF
```

生成roles/win_ad_install/tasks/main.yml
```
cat > roles/win_ad_install/tasks/main.yml << EOF
- name: Install AD-Domain-Services feature
  win_feature:
    name: AD-Domain-Services
    include_management_tools: yes
    include_sub_features: yes

- name: Setup Active Directory Controller
  win_domain:
    dns_domain_name: "{{ ad_domain_name }}"
    safe_mode_password: "{{ ad_safe_mode_password }}"
  register: active_directory_controllers

- name: Reboot once DC created
  win_reboot:
  when: active_directory_controllers.reboot_required

- name: List Domain Controllers in domain
  win_shell: "nltest /dclist:{{ ad_domain_name }}"
  register: domain_list

- debug:
   var: domain_list
EOF
```

##### 创建playbook
```
cat << EOF > setup_ad.yml
---
- name: install and configure active directory
  hosts: windows
  gather_facts: false

  roles:
    - win_ad_install
EOF
```

##### 执行playbook
```
ansible-playbook setup_ad.yml
```

#### Controller配置Kerberose
##### 生成环境变量
```
export GUID=`hostname | awk -F"." '{print $2}'`
export GUID_CAP=`echo ${GUID} | tr 'a-z' 'A-Z'`
```

##### 创建kerberos配置
```
cat << EOF > /etc/krb5.conf.d/ansible.conf

[realms]

 AD1.${GUID_CAP}.EXAMPLE.OPENTLC.COM = {

 kdc = ad1.${GUID}.example.opentlc.com
 }

[domain_realm]
 .ad1.${GUID}.example.opentlc.com = AD1.${GUID_CAP}.EXAMPLE.OPENTLC.COM
EOF
```

##### 确认kerberos工作正常
```
kinit administrator@AD1.${GUID_CAP}.EXAMPLE.OPENTLC.COM
```

##### 列出Kerberos principle and tickets
```
klist
```

#### 配置OpenSSH on Windows Server
##### 创建Roles
```
mkdir -p roles/win_service_config/{tasks,vars}
cat << EOF > roles/win_service_config/tasks/main.yml
---
- name: Install Windows package
  win_chocolatey:
    name: "{{ package_name }}"
    params: "{{ parameters }}"
    state: latest
  when: ansible_distribution == "Microsoft Windows Server 2012 R2 Standard"

- name: Start windows service
  win_service:
    name: "{{ service_name }}"
    state: started
    start_mode: auto
  when: ansible_distribution == "Microsoft Windows Server 2012 R2 Standard"

- name: Add win_firewall_rule
  win_firewall_rule:
    name: "{{ service_name }}"
    localport: "{{ local_port }}"
    action: allow
    direction: in
    protocol: "{{ protocol_name }}"
    state: present
    enabled: yes
EOF
```

##### 创建Playbook vars
```
cat << EOF > ssh_var.yml
package_name: openssh
parameters: /SSHServerFeature
service_name: SSHD
local_port: 22
protocol_name: tcp
EOF
```

##### 创建Playbook
```
cat << EOF > win_ssh_server.yml
- hosts: windows
  vars_files:
    - ./ssh_var.yml
  roles:
    - win_service_config
EOF
```

##### 执行Playbook
```
ansible-playbook win_ssh_server.yml
```

#### 创建Windows AD Users/Groups
##### 创建vars
```
cat << EOF > roles/win_ad_user/defaults/main.yml
# vars file for roles/win_ad_user
user_info:
  - { name: 'james', firstname: 'James', surname: 'Jockey', password: 'redhat@123', group_name: 'dev', group_scope: 'domainlocal'}
  - { name: 'bill', firstname: 'Bill', surname: 'Gates', password: 'redhat@123', group_name: 'dev', group_scope: 'domainlocal'}
  - { name: 'mickey', firstname: 'Mickey', surname: 'Mouse', password: 'redhat@123', group_name: 'qa', group_scope: 'domainlocal'}
  - { name: 'donald', firstname: 'Donald', surname: 'Duck', password: 'redhat@123', group_name: 'qa', group_scope: 'domainlocal'}
EOF
```

##### 创建



### Memo
|hostname|ipaddr|
|---|---|
|test.example.com|192.168.0.1|
