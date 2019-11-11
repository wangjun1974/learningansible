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
使用可读性强的名称构建inventory
```
Poor                       Better
10.1.2.75                   db1 ansible_host=10.1.2.75
10.1.5.45                   db2 ansible_host=10.1.5.45

w14301.acme.com             web1 ansible_host=w14301.acme.com
w17802.acme.com             web2 ansible_host=w17802.acme.com
```







### Memo
|hostname|ipaddr|
|---|---|
|test.example.com|192.168.0.1|
