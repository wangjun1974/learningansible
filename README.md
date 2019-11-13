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

### Day1 Labs - Best Practice Labs

#### 浏览环境

##### 列出全部主机
```
ansible all --list-hosts
```

##### 检查主机可ping通
```
ansible all -m ping
```

##### 检查app1可登陆
```
ssh app1.${GUID}.internal
```
#### 配置Cliet

##### 创建Client环境
```
export GUID=`hostname | awk -F"." '{print $2}'`
mkdir ~/.ssh
sudo cp /root/.ssh/${GUID}key.pem ~/.ssh
sudo chown `whoami` ~/.ssh/${GUID}key.pem
sudo chmod 400 ~/.ssh/${GUID}key.pem
sudo chmod 400 ~/.ssh/${GUID}key.pem
```

##### 检查实例可登陆
```
ssh -i ~/.ssh/${GUID}key.pem  ec2-user@app1.${GUID}.example.opentlc.com
exit
```
##### 创建inventory
```
cp /etc/ansible/hosts ~/myinventory.file
ansible -i myinventory.file all -m ping
```

#### 部署3层应用

##### 克隆bad-repository
```
git clone https://github.com/wangjun1974/bad-ansible
```

##### 拷贝文件
```
cd bad-ansible
cp /etc/yum.repos.d/open_three-tier-app.repo .
```

#### 执行bad-ansible playbook

##### 执行playbook，不带变量
```
ansible-playbook main.yml
```

##### 执行playbook，设置变量
```
ansible-playbook main.yml -e "GUID=${GUID}"
```

##### 检查网址可访问
```
curl http://frontend1.${GUID}.example.opentlc.com
```

#### 清理环境
```
ansible-playbook cleanup.yml
```

#### 重构playbook


##### 创建roles目录
```
mkdir -p roles/{ansible-roles-preconfig,ansible-roles-frontends,ansible-roles-apps,ansible-roles-db}
for i in ansible-roles-preconfig ansible-roles-frontends ansible-roles-apps ansible-roles-db
do
  mkdir -p roles/$i/{tasks,handlers,files,defaults,templates}
done
```

#### Roles - ansible-roles-preconfig

##### 生成roles/ansible-roles-preconfig/files/open_three-tier-app.repo
```
cat > roles/ansible-roles-preconfig/files/open_three-tier-app.repo << EOF
[rhel-7-server-rpms]
name=Red Hat Enterprise Linux 7
baseurl=http://admin.na.shared.opentlc.com/repos/ocp/3.6//rhel-7-server-rpms
enabled=1
gpgcheck=0

[rhel-7-server-rh-common-rpms]
name=Red Hat Enterprise Linux 7 Common
baseurl=http://admin.na.shared.opentlc.com/repos/ocp/3.6//rhel-7-server-rh-common-rpms
enabled=1
gpgcheck=0

[rhel-7-server-extras-rpms]
name=Red Hat Enterprise Linux 7 Extras
baseurl=http://admin.na.shared.opentlc.com/repos/ocp/3.6//rhel-7-server-extras-rpms
enabled=1
gpgcheck=0

[rhel-7-server-optional-rpms]
name=Red Hat Enterprise Linux 7 Optional
baseurl=http://admin.na.shared.opentlc.com/repos/ocp/3.6//rhel-7-server-optional-rpms
enabled=1
gpgcheck=0

[epel]
name=Extra Packages for Enterprise Linux 7 - $basearch
baseurl=http://download.fedoraproject.org/pub/epel/7/$basearch
mirrorlist=http://mirrors.fedoraproject.org/metalink?repo=epel-7&arch=$basearch
failovermethod=priority
enabled=1
gpgcheck=0
#gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-EPEL-7
EOF
```

##### 生成roles/ansible-roles-preconfig/tasks/mail.yml
```
cat > roles/ansible-roles-preconfig/tasks/mail.yml << EOF
---
- name: 'Enable repos'
  copy:
    src: open_three-tier-app.repo
    dest: /etc/yum.repos.d
    owner: root
    group: root
    mode: '0644'
EOF
```

#### Roels - ansible-roles-frontends

##### 生成roles/ansible-roles-frontends/templates/haproxy.cfg.j2
```
cat > roles/ansible-roles-frontends/templates/haproxy.cfg.j2 << 'EOF'
global
  log /dev/log  local0
  log /dev/log  local1 notice
  stats socket /var/lib/haproxy/stats level admin
  chroot /var/lib/haproxy
  user haproxy
  group haproxy
  daemon

defaults
  log global
  mode  http
  option  httplog
  option  dontlognull
        timeout connect 5000
        timeout client 50000
        timeout server 50000

frontend hafrontend
    bind *:80
    mode http
    default_backend app-servers

backend app-servers
    mode http
    balance roundrobin
    option forwardfor
    server app1 app1.{{GUID}}.internal:8080 cookie app1 check
    server app2 app2.{{GUID}}.internal:8080 cookie app2 check
EOF
```

#### 生成roles/ansible-roles-frontends/handlers/main.yml
```
cat > roles/ansible-roles-frontends/handlers/main.yml << EOF
  
---
- name: enable and start haproxy
  systemd:
    name: haproxy
    enabled: yes
    state: restarted
    daemon_reload: yes
  become: yes
EOF
```

##### 生成roles/ansible-roles-frontends/tasks/main.yml
```
cat > roles/ansible-roles-frontends/tasks/main.yml << EOF
---
- name: 'Install packages'
  yum:
    name: "{{ item }}"
    state: latest
  with_items:
    - httpie
    - haproxy

- name: 'Configure haproxy'
  template:
    src: haproxy.cfg.j2
    dest: /etc/haproxy/haproxy.cfg
    owner: root
    group: root
    mode: 0644
  notify: enable and start haproxy

- name: 'Start haproxy'
  service:
    name: haproxy
    enabled: yes
    state: started
EOF    
```

#### Roles - ansible-roles-apps

##### roles/ansible-roles-apps/files/index.html.app1
```
cat > roles/ansible-roles-apps/files/index.html.app1 << EOF  
<!doctype html>

<html lang="en">
<head>
  <meta charset="utf-8">

  <title>Ansible Deployed Tomcat</title>
  <meta name="description" content="Ansible Created Content">
  <meta name="tony.g.kay@gmail.com" content="Ansible">

</head>

<body>
    <h1>Ansible has done its job - Welcome to Tomcat on App1</h1>
</body>
</html>
EOF
```

##### roles/ansible-roles-apps/files/index.html.app2
```
cat > roles/ansible-roles-apps/files/index.html.app2 << EOF
<!doctype html>

<html lang="en">
<head>
  <meta charset="utf-8">

  <title>Ansible Deployed Tomcat</title>
  <meta name="description" content="Ansible Created Content">
  <meta name="tony.g.kay@gmail.com" content="Ansible">

</head>

<body>
    <h1>Ansible has done its job - Welcome to Tomcat on App2</h1>
</body>
</html>
EOF
```

##### roles/ansible-roles-apps/handlers/main.yml
```
cat > << EOF
---
- name: enable and start tomcat and httpd
  systemd:
    name: "{{ item }}"
    enabled: yes
    state: restarted
    daemon_reload: yes
  with_items:
    - tomcat
    - httpd
  become: yes
EOF
```

##### roles/ansible-roles-apps/tasks/main.yml
```
cat > roles/ansible-roles-apps/tasks/main.yml << EOF
---
- name: 'Install tomcat'
  yum:
    name: "{{ item }}"
    state: latest
  with_items:
    - tomcat
    - httpd

- name: 'Create tomcat directory'
  file:
    path: /usr/share/tomcat/webapps/ROOT
    state: directory

- name: 'Copy template index.html.j2 to tomcat'
  template:
    src: index.html.j2
    dest: /usr/share/tomcat/webapps/ROOT/index.html
    mode: 0644
  notify: enable and start tomcat and httpd
EOF
```

#### Roles - ansible-roles-db

##### roles/ansible-roles-db/handlers/main.yml
```
cat > roles/ansible-roles-db/handlers/main.yml << EOF
---
- name: enable and start postgres
  systemd:
    name: postgresql
    enabled: yes
    state: restarted
    daemon_reload: yes
  become: yes
EOF
```

##### roles/ansible-roles-db/tasks/main.yml
```
cat > roles/ansible-roles-db/tasks/main.yml << EOF
---
- name: 'Install postgres'
  yum:
    name: ['postgresql-server']
    state: latest

- name: 'Check if data directory exist'
  shell: ls /var/lib/pgsql/data
  register: check_db_output
  ignore_errors: yes
  
- name: 'Initialize postgres'
  command: postgresql-setup initdb
  when: check_db_output.rc != 0

- name: 'Enable and start postgres'
  debug: 
    msg: "Enable and start postgres"
  notify: enable and start postgres
EOF
```

#### 创建playbook

##### 创建playbook preconfig.yml 
```
cat > preconfig.yml  << EOF
---
- name: config repos
  hosts: all
  gather_facts: false # remove later! speeds up testing
  become: true

  roles:
    - ansible-roles-preconfig 
EOF
```

##### 创建playbook frontends.yml
```
cat > frontends.yml << EOF
---
- name: config frontends
  hosts: frontends
  gather_facts: false # remove later! speeds up testing
  become: true

  roles:
    - ansible-roles-frontends
EOF
```

##### 创建playbook apps.yml
```
cat > apps.yml << EOF
---
- name: config apps
  hosts: apps
  gather_facts: false # remove later! speeds up testing
  become: true

  roles:
    - ansible-roles-apps
EOF
```

##### 创建playbook db.yml
```
cat > db.yml << EOF
---
- name: config db
  hosts: appdbs
  gather_facts: false # remove later! speeds up testing
  become: true

  roles:
    - ansible-roles-db
EOF
```

##### 创建playbook site.yml
```
cat > site.yml << EOF
---
- import_playbook: preconfig.yml
- import_playbook: frontends.yml
- import_playbook: apps.yml
- import_playbook: db.yml
EOF
```

#### 执行playbook
```
ansible-playbook cleanup.yml
ansible-playbook site.yml -e "GUID=${GUID}"
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

#### Day2 Labs - Windows Integration
##### 安装ansible 
```
yum install -y ansible
```

##### 生成ansible inventory文件
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

##### 列出全部主机
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

##### 安装Ansible管理Windows所需软件包
```
yum install -y python-devel krb5-devel krb5-libs krb5-workstation python-pip gcc
```

##### 安装版本大于等于0.2.2的pywinrm
```
pip install "pywinrm>=0.2.2"
```

##### 检查windows主机可访问
```
ansible windows -m win_ping
/usr/lib/python2.7/site-packages/urllib3/connectionpool.py:1004: InsecureRequestWarning: Unverified HTTPS request is being made. Adding certificate verification is strongly advised. See: https://urllib3.readthedocs.io/en/latest/advanced-usage.html#ssl-warnings
  InsecureRequestWarning,
ad1.d7da.internal | SUCCESS => {
    "changed": false, 
    "ping": "pong"
}
```

##### 创建github项目，克隆项目
```
git clone https://github.com/wangjun1974/windowsad.git
cd windowsad
```

##### 创建目录
```
mkdir -p roles/win_ad_install/tasks
mkdir -p roles/win_ad_install/defaults
```

##### 生成roles默认变量roles/win_ad_install/defaults/main.yml
```
cat > roles/win_ad_install/defaults/main.yml << EOF
---
ad_domain_name: ad1.${GUID}.example.opentlc.com
ad_safe_mode_password: "{{ansible_password}}"
ad_admin_user: "admin@{{ad_domain_name}}"
ad_admin_password: "{{ansible_password}}"
EOF
```

##### 生成roles/win_ad_install/tasks/main.yml
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
cat << EOF > roles/win_service_config/defaults/main.yml
---
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
mkdir -p roles/win_ad_user/{tasks,defaults}
cat << EOF > roles/win_ad_user/defaults/main.yml
---
# vars file for roles/win_ad_user
user_info:
  - { name: 'james', firstname: 'James', surname: 'Jockey', password: 'redhat@123', group_name: 'dev', group_scope: 'domainlocal'}
  - { name: 'bill', firstname: 'Bill', surname: 'Gates', password: 'redhat@123', group_name: 'dev', group_scope: 'domainlocal'}
  - { name: 'mickey', firstname: 'Mickey', surname: 'Mouse', password: 'redhat@123', group_name: 'qa', group_scope: 'domainlocal'}
  - { name: 'donald', firstname: 'Donald', surname: 'Duck', password: 'redhat@123', group_name: 'qa', group_scope: 'domainlocal'}
EOF
```

##### 创建tasks
```
cat << EOF > roles/win_ad_user/tasks/main.yml
---
# tasks file for roles/win_ad_user
- name: Create windows domain group
  win_domain_group:
    name: "{{ item.group_name }}"
    scope: "{{ item.group_scope }}"
    state: present
  loop: "{{ user_info }}"

- name: Create AD User
  win_domain_user:
    name: "{{ item.name }}"
    firstname: "{{item.firstname }}"
    surname: "{{ item.surname }}"
    password: "{{ item.password }}"
    groups: "{{ item.group_name }}"
    state: present
    email: '"{{ item.name }}"@ad1.${GUID}.example.opentlc.com'
  loop: "{{ user_info }}"
EOF
```

##### 创建Playbook
```
cat << EOF > ad_user_group_create.yml
---
- name: add active directory users
  hosts: windows
  roles:
    - win_ad_user
EOF
```

##### 执行Playbook
```
ansible-playbook ad_user_group_create.yml
```

##### 登陆mickey用户
```
kinit mickey@AD1.${GUID_CAP}.EXAMPLE.OPENTLC.COM
```

##### 确认SSH服务工作正常
```
ssh mickey@ad1.${GUID}.example.opentlc.com
```


### Day3

#### Labs 3.1

##### 下载并解包最新版ansible-tower
```
ansible localhost -m unarchive -a "src=https://releases.ansible.com/ansible-tower/setup/ansible-tower-setup-latest.tar.gz dest=/root/ remote_src=yes"
```

##### 生成inventory文件 
```
cat > /root/ansible-tower-setup-*/inventory << EOF
[tower]
tower1.d7da.internal
tower2.d7da.internal
tower3.d7da.internal

[database]
support1.d7da.internal

[all:vars]
ansible_become=true                  #### 执行时切换为root
admin_password='redhat123'

pg_host='support1.d7da.internal'
pg_port='5432'
pg_database='awx'
pg_username='awx'
pg_password='redhat123'

rabbitmq_port=5672
rabbitmq_vhost=tower
rabbitmq_username=tower
rabbitmq_password='redhat123'
rabbitmq_cookie=cookiemonster
rabbitmq_use_long_name=true          #### RabbitMQ使用FQDN
EOF
```

#### 执行ansible tower安装
```
/root/ansible-tower-setup-*/setup.sh
```

#### 访问Tower界面

##### Labs 3.2

https://www.ansible.com/blog/ansible-tower-feature-spotlight-instance-groups-and-isolated-nodes

安装带isolation group的Ansible Tower，把bastion设置为isolated_group_ThreeTierApp的member

在Three Tier APPs bastion
```
wget http://www.opentlc.com/download/ansible_bootcamp/openstack_keys/openstack.pub
cat openstack.pub  >> /home/ec2-user/.ssh/authorized_keys
```

在Ansible Tower bastion
```
export APP_GUID="ada0"
ansible all -i bastion.${APP_GUID}.example.opentlc.com, --private-key=~/.ssh/openstack.pem -u ec2-user -m ping
```


```
export GUID="d7da"
export APP_GUID="ada0"

cat << EOF > /root/ansible-tower-setup-*/inventory
[tower]
tower1.${GUID}.internal
tower2.${GUID}.internal
tower3.${GUID}.internal
[database]
support1.${GUID}.internal
[isolated_group_ThreeTierApp]
bastion.${APP_GUID}.example.opentlc.com ansible_user='ec2-user' ansible_ssh_private_key_file='~/.ssh/openstack.pem'
[isolated_group_ThreeTierApp:vars]
controller=tower
[all:vars]
ansible_become=true
admin_password='redhat123'
pg_host='support1.${GUID}.internal'
pg_port='5432'
pg_database='awx'
pg_username='awx'
pg_password='redhat123'
rabbitmq_port=5672
rabbitmq_vhost=tower
rabbitmq_username=tower
rabbitmq_password='redhat123'
rabbitmq_cookie=cookiemonster
rabbitmq_use_long_name=true
EOF
```

执行安装
```
/root/ansible-tower-setup-*/setup.sh
```

在Three Tier bastion
```
export APP_GUID="ada0"
cat << EOF > ./inventory/hosts
[frontends]
frontend1.${APP_GUID}.internal ansible_ssh_host=frontend1.${APP_GUID}.example.opentlc.com

[apps]
app1.${APP_GUID}.internal ansible_ssh_host=app1.${APP_GUID}.example.opentlc.com
app2.${APP_GUID}.internal ansible_ssh_host=app2.${APP_GUID}.example.opentlc.com

[appdbs]
appdb1.${APP_GUID}.internal ansible_ssh_host=appdb1.${APP_GUID}.example.opentlc.com
EOF
``` 

#### Lab 3.3

##### 为Ansible Tower准备HA PostgreSQL
PostgreSQL灾备切换<br>
参见：
https://severalnines.com/database-blog/postgresql-replication-disaster-recovery<br>
https://scalegrid.io/blog/getting-started-with-postgresql-streaming-replication/<br>


在Ansible Tower bastion, 编辑support1服务器上的postgresql.conf
```
ansible support1.${GUID}.internal -m lineinfile -a "line='include_dir = 'conf.d'' path=/var/lib/pgsql/9.6/data/postgresql.conf"
```

创建目录/var/lib/pgsql/9.6/data/conf.d
```
ansible support1.${GUID}.internal -m file -a 'path=/var/lib/pgsql/9.6/data/conf.d state=directory'
```

创建并拷贝文件
```
cat << EOF > tower-postgresql.conf
wal_level = hot_standby
synchronous_commit = local
archive_mode = on
archive_command = 'cp %p /var/lib/pgsql/9.6/data/archive/%f'
max_wal_senders = 2
wal_keep_segments = 10
synchronous_standby_names = 'awx'
EOF

ansible support1.${GUID}.internal -m copy -a "src=/root/tower-postgresql.conf dest=/var/lib/pgsql/9.6/data/conf.d/tower-postgresql.conf"
```

创建复制用户
https://stackoverflow.com/questions/17429040/creating-user-with-encrypted-password-in-postgresql

```
psql
# CREATE USER replica LOGIN REPLICATION PASSWORD ’redhat123’;
```

配置数据库服务器接受复制服务器连接
https://www.postgresql.org/docs/9.1/auth-pg-hba-conf.html
```
ansible support1.${GUID}.internal  -m lineinfile -a "line='host    replication replica     0.0.0.0/0        md5' path=/var/lib/pgsql/9.6/data/pg_hba.conf"
```

重启数据库服务器
```
ansible support1.${GUID}.internal -m service -a"name=postgresql-9.6 state=restarted"
```

##### 准备复制数据库服务器
在第二台数据库服务器上配置repo和repo key
```
ansible support2.${GUID}.internal -m get_url -a "url=http://www.opentlc.com/download/ansible_bootcamp/repo/pgdg-96-centos.repo dest=/etc/yum.repos.d/pgdg-96-centos.repo"
ansible support2.${GUID}.internal -m get_url -a "url=http://www.opentlc.com/download/ansible_bootcamp/repo/RPM-GPG-KEY-PGDG-96 dest=/etc/pki/rpm-gpg/RPM-GPG-KEY-PGDG-96"
```

在第二台数据库服务器上安装postgresql96-server
```
ansible support2.${GUID}.internal -m yum -a "name=postgresql96-server state=present"
```

在第二台数据库服务器上复制数据
```
ansible support2.${GUID}.internal -m shell -a "export PGPASSWORD=redhat123 && pg_basebackup -h support1.${GUID}.internal -U replica -D /var/lib/pgsql/9.6/data/ -P --xlog" --become-user=postgres
```

修改第二台数据库服务器的/var/lib/pgsql/9.6/data/postgresql.conf，设置hot_standby为on
```
ansible support2.${GUID}.internal -m lineinfile -a "line='hot_standby = on' path=/var/lib/pgsql/9.6/data/postgresql.conf"
```

拷贝recovery.conf文件到第二台数据库服务器上
```
cat << EOF > recovery.conf
restore_command = 'scp support1.${GUID}.internal:/var/lib/pgsql/9.6/data/archive/%f %p'
standby_mode = on
primary_conninfo = 'host=support1.${GUID}.internal port=5432 user=replica password=redhat123 application_name=awx'
EOF

ansible support2.${GUID}.internal -m copy -a "src=/root/recovery.conf dest=/var/lib/pgsql/9.6/data/recovery.conf"
```

启用并启动第二台数据库服务器的postgresql-9.6服务
```
ansible support2.${GUID}.internal -m service -a "name=postgresql-9.6 state=started enabled=true"
```

##### 用Sam Doran Role 配置 Replication

在bastion节点上
```
cd /root/ansible-tower-setup-*
ansible-galaxy install samdoran.pgsql_replication -p roles
```

修改roles/samdoran.pgsql_replication/tasks/master.yml文件的Task 'Add trust in pg_hba.conf'，重新定义pgsqlrep_replica_address变量
```
loop: "{{ pgsqlrep_replica_address }}"
vars:
  pgsqlrep_replica_address: "{{ groups[pgsqlrep_group_name] | map('extract', hostvars, 'ansible_all_ipv4_addresses') | flatten }}"
notify: restart postgresql
```

生成inventory
```
cat << EOF > pg_inventory
[tower]
tower1.d7da.internal
tower2.d7da.internal
tower3.d7da.internal
[database]
support1.d7da.internal pgsqlrep_role=master

[database_replica]
support2.d7da.internal pgsqlrep_role=replica
EOF
```

创建新playbook
```
cat > pgsql_replication.yml << 'EOF'
- name: Configure PostgreSQL streaming replication
  hosts: database_replica

  tasks:
    - name: Find recovery.conf
      find:
        paths: /var/lib/pgsql
        recurse: yes
        patterns: recovery.conf
      register: recovery_conf_path

    - name: Remove recovery.conf
      file:
        path: "{{ item.path }}"
        state: absent
      loop: "{{ recovery_conf_path.files }}"

    - name: Add replica to database group
      add_host:
        name: "{{ inventory_hostname }}"
        groups: database
      tags:
        - always

    - import_role:
        name: nginx
      vars:
        nginx_exec_vars_only: yes

    - import_role:
        name: repos_el
      when: ansible_os_family == "RedHat"

    - import_role:
        name: packages_el
      vars:
        packages_el_install_tower: no
        packages_el_install_postgres: yes
      when: ansible_os_family == "RedHat"

    - import_role:
        name: postgres
      vars:
        postgres_allowed_ipv4: "0.0.0.0/0"
        postgres_allowed_ipv6: "::/0"
        postgres_username: "{{ pg_username }}"
        postgres_password: "{{ pg_password }}"
        postgres_database: "{{ pg_database }}"
        max_postgres_connections: 1024
        postgres_shared_memory_size: "{{ (ansible_memtotal_mb*0.3)|int }}"
        postgres_work_mem: "{{ (ansible_memtotal_mb*0.03)|int }}"
        postgres_maintenance_work_mem: "{{ (ansible_memtotal_mb*0.04)|int }}"
      tags:
        - postgresql_database

    - import_role:
        name: firewall
      vars:
        firewalld_http_port: "{{ nginx_http_port }}"
        firewalld_https_port: "{{ nginx_https_port }}"
      tags:
        - firewall
      when: ansible_os_family == 'RedHat'

- name: Configure PSQL master server
  hosts: database[0]

  vars:
    pgsqlrep_master_address: "{{ hostvars[groups[pgsqlrep_group_name_master][0]].ansible_all_ipv4_addresses[-1] }}"
    pgsqlrep_replica_address: "{{ hostvars[groups[pgsqlrep_group_name][0]].ansible_all_ipv4_addresses[-1] }}"

  tasks:
    - import_role:
        name: samdoran.pgsql_replication

- name: Configure PSQL replica
  hosts: database_replica

  vars:
    pgsqlrep_master_address: "{{ hostvars[groups[pgsqlrep_group_name_master][0]].ansible_all_ipv4_addresses[-1] }}"
    pgsqlrep_replica_address: "{{ hostvars[groups[pgsqlrep_group_name][0]].ansible_all_ipv4_addresses[-1] }}"

  tasks:
    - import_role:
        name: samdoran.pgsql_replication
EOF
```

执行playbook
```
ansible-playbook -b -i pg_inventory pgsql_replication.yml -e pgsqlrep_password=redhat123 -e pg_username='awx' -e pg_database='awx' -e pg_password='redhat123'
```

检查复制
```
ansible support1.${GUID}.internal -m shell -a "psql -c 'select application_name, state, sync_priority, sync_state from pg_stat_replication;'" --become-user postgres
```

https://www.tutorialdba.com/2017/09/postgresql-replication-sync-to-async.html

https://pgdash.io/blog/monitoring-postgres-replication.html

##### 切换

生成切换Playbook
```
cat << 'EOF' > postgres_failover.yml
- name: Gather facts
  hosts: all
  become: yes

- name: Failover PostgreSQL
  hosts: database_replica
  become: yes

  tasks:
    - name: Get the current PostgreSQL Version
      import_role:
        name: samdoran.pgsql_replication
        tasks_from: pgsql_version.yml

    - name: Promote secondary PostgreSQL server to primary
      command: /usr/pgsql-{{ pgsql_version }}/bin/pg_ctl promote
      become_user: postgres
      environment:
        PGDATA: /var/lib/pgsql/{{ pgsql_version }}/data
      ignore_errors: yes

- name: Update Ansible Tower database configuration
  hosts: tower
  become: yes

  tasks:
    - name: Update Tower postgres.py
      lineinfile:
        dest: /etc/tower/conf.d/postgres.py
        regexp: "^(.*'HOST':)"
        line: "\\1 '{{ hostvars[groups['database_replica'][0]].ansible_default_ipv4.address }}',"
        backrefs: yes
      notify: restart tower

  handlers:
    - name: restart tower
      command: ansible-tower-service restart
EOF
```

关闭当前primary数据库服务器
```
ansible support1.$GUID.internal -m shell -a 'shutdown -h now'
```

切换数据库
```
ansible-playbook -b -i pg_inventory postgres_failover.yml -e pgsqlrep_password=r3dh4t1!
```

检查tower是否工作正常，访问https://tower1.d7da.example.opentlc.com/