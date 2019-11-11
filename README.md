# learningansible
Learning Ansible

### 课程链接 
https://learning.redhat.com/course/view.php?id=1042

### Day1
Ansible Best Practise Style Guide
可以参考<br>
https://github.com/openshift/openshift-ansible/blob/master/docs/style_guide.adoc<br>

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


### Memo
|hostname|ipaddr|
|---|---|
|test.example.com|192.168.0.1|
