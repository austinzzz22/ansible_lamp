# 部署前准备
-  IP地址分配
| server1 | 192.168.60.10 | Ansible控制节点 |
| --- | --- | --- |
| server2 | 192.168.60.20 | Nginx负载均衡 |
| server3 | 192.168.60.30 | Apache网站服务 |
| server4 | 192.168.60.40 | Apache网站服务 |
| server5 | 192.168.60.50 | Mariadb数据库 |
| server6 | 192.168.60.60 | NFS网络文件系统 |

- 项目框架

![image.png](https://cdn.nlark.com/yuque/0/2023/png/21997074/1678194906024-d54d0bc7-2cdc-457d-9732-e6c303c3962a.png#averageHue=%23f8f8f8&clientId=u657a55bc-6575-4&from=paste&height=304&id=uf3f9407f&name=image.png&originHeight=380&originWidth=658&originalType=binary&ratio=1.25&rotation=0&showTitle=false&size=43752&status=done&style=none&taskId=u6c3f76a5-8c30-4571-9f92-955b2dc91fd&title=&width=526.4)

- 主节点的hosts文件
```bash
[root@server1 ~]# cat /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6

192.168.60.10 server1
192.168.60.20 server2
192.168.60.30 server3
192.168.60.40 server4
192.168.60.50 server5
192.168.60.60 server6
```

- 配置公钥和私钥，实现ansible控制节点的免密登录
```bash
[root@server1 ~]# ssh-keygen -P "" -t rsa
[root@server1 ~]# ssh-copy-id -i /root/.ssh/id_rsa.pub root@server1
[root@server1 ~]# ssh-copy-id -i /root/.ssh/id_rsa.pub root@server2
[root@server1 ~]# ssh-copy-id -i /root/.ssh/id_rsa.pub root@server3
[root@server1 ~]# ssh-copy-id -i /root/.ssh/id_rsa.pub root@server4
[root@server1 ~]# ssh-copy-id -i /root/.ssh/id_rsa.pub root@server5
[root@server1 ~]# ssh-copy-id -i /root/.ssh/id_rsa.pub root@server6
```

- ansible  hosts解析文件
```bash
[root@server1 roles]# cat /etc/ansible/hosts 
[all_ip]
192.168.60.10
192.168.60.20
192.168.60.30
192.168.60.40
192.168.60.50
192.168.60.60

[all_hostname]
server1
server2
server3
server4
server5
server6

[nginx]
server2

[apache]
server3
server4

[mariadb]
server5

[nfs]
server6
```

- 为所有主机生成hosts文件
```bash
[root@server1 ~]# cat template/hosts.j2 
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6

{% for host in groups.all_ip %}
{{hostvars[host].ansible_ens33.ipv4.address}} {{hostvars[host].ansible_hostname}}
{% endfor %}
[root@server1 ~]# cat playbook/hosts.yml 
- name: config host file
  hosts: all_ip
  remote_user: root

  tasks:
  - name: copy host.j2 to group servers
    template:
      src: ../template/hosts.j2
      dest: /etc/hosts`
[root@server1 ~]# ansible-playbook playbook/hosts.yml
```
## 为所有节点更换阿里云yum源

- 将本地的yum源拷贝到各个角色服务器上
```bash
[root@server1 roles]# ansible-galaxy init aliyum
[root@server1 ~]# cat /etc/ansible/roles/aliyum/tasks/main.yml 
---
# tasks file for aliyum
- name: find files in yum.repo.d
  find:
    paths: /etc/yum.repo.d/
    patterns: '*'
  register: files_to_delete

- name: remove original yum.repos.d/*
  file:
    path: "{{ item.path }}"
    state: absent
  with_items: "{{ files_to_delete.files }}"

- name: copy aliyun yum.repo to all nodes
  copy: 
    src: yum.repo
    dest: /etc/yum.repo.d/
[root@server1 ~]# cat /etc/ansible/roles/yum_repo_role_use.yml 
- name: update all nodes yum.repo file
  hosts: all_hostname

  roles:
  - aliyum
[root@server1 ~]# ansible-playbook /etc/ansible/roles/yum_repo_role_use.yml
```
# 使用ansible的角色进行部署
## 角色的创建
```bash
[root@server1 ~]# ansible-galaxy init /etc/ansible/roles/apache
[root@server1 ~]# ansible-galaxy init /etc/ansible/roles/nginx
[root@server1 ~]# ansible-galaxy init /etc/ansible/roles/mariadb
[root@server1 ~]# ansible-galaxy init /etc/ansible/roles/nfs
```
## nginx角色的部署

- 下载epel-release拓展源
- 下载nginx
```bash
[root@server1 ~]# cat /etc/ansible/roles/nginx/tasks/main.yml 
---
# tasks file for nginx
- name: yum epel
  yum: 
    name: epel-release.noarch
    state: present
- name: yum install nginx
  yum:
    name: nginx
    state: present

- name: start nginx
  service:
    name: nginx
    state: restarted
    enabled: yes
[root@server1 ~]# cat /etc/ansible/roles/nginx_install.yml 
- name: install nginx
  hosts: server2
  roles:
  - nginx
[root@server1 ~]# ansible-playbook /etc/ansible/roles/nginx_install.yml
```
## apache角色的部署

- 部署apache的同时部署php7
```bash
[root@server1 ~]# cat /etc/ansible/roles/apache/tasks/main.yml 
---
# tasks file for apache
- name: install lamp enviroment
  yum:
    name: httpd
    state: present
- name: install php7
  command: rpm -Uvh https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm
  ignore_errors: yes #多次执行会报错

- name: get webtatic download source
  command: rpm -Uvh https://mirror.webtatic.com/yum/el7/webtatic-release.rpm
  ignore_errors: yes #多次执行会报错

- name: install php7.2
  command: yum -y install php72w php72w-cli php72w-common php72w-devel php72w-embedded php72w-gd php72w-mbstring php72w-pdo php72w-xml php72w-fpm php72w-mysqlnd php72w-opcache php72w-redis 


- name: start and enable the httpd service
  service: 
    name: httpd
    state: started
    enabled: true

- name: start and enable the php-fpm service
  service:
    name: php-fpm
    state: started
    enabled: true
[root@server1 ~]# cat /etc/ansible/roles/apache_install.yml 
- name: prepare apache and php
  hosts: apache

  roles:
  - apache
[root@server1 ~]# ansible-playbook  /etc/ansible/roles/apache_install.yml
```
## Mariadb角色的部署

- 安装mariadb数据库
- 开启数据库服务
- 配置root用户密码
- 创建lamp用户
```bash
[root@server1 ~]# cat /etc/ansible/roles/mariadb/tasks/main.yml
---
# tasks file for mariadb
- name: yum install mariadb
  yum: 
    name: mariadb-server

- name: start mariadb
  service: 
    name: mariadb
    state: restarted
    enabled: true
- name: set password for root
  command: mysqladmin password '123456'
- name: create the lamp database
  command: mysql -uroot -p123456 -e "create database lamp;"

- name: grant privileges for lamp
  command: mysql -uroot -p123456 -e "grant SELECT, INSERT, UPDATE, DELETE, CREATE, REFERENCES, INDEX, ALTER, CREATE TEMPORARY TABLES, LOCK TABLES, EXECUTE, CREATE VIEW, SHOW VIEW, CREATE ROUTINE, ALTER ROUTINE, EVENT, TRIGGER on lamp.* to 'lamp'@'192.168.60.%';"

[root@server1 ~]# cat /etc/ansible/roles/mariadb_install.yml 
- name: install and disposition the mariadb role
  hosts: mariadb

  roles:
  - mariadb
[root@server1 ~]# ansible-playbook /etc/ansible/roles/mariadb_install.yml
```
## nginx负载均衡配置

- 准备负载均衡配置文件
- 拷贝配置文件到nginx负载均衡服务器
```bash
[root@server1 ~]# cat /etc/ansible/roles/nginx_lb/tasks/main.yml 
---
# tasks file for nginx_lb
- name: congigure nginx lb conf file
  template: 
    src: /etc/ansible/roles/nginx_lb/templates/lb.conf.j2
    dest: /etc/nginx/conf.d/lb.conf
- name: restart nginx
  service:
    name: nginx
    state: restarted
[root@server1 ~]# cat /etc/ansible/roles/nginx_lb/templates/lb.conf.j2
upstream websers{
server server3;
server server4;
}
server{
    listen 8080;
	server_name 192.168.60.20:8080;
	location / {
	    proxy_pass http://websers;
	}
	location ~ \.php$ {
	    proxy_pass http://websers;
		proxy_set_header Host $http_host;

		proxy_set_header X-Real-IP $remote_addr;

		proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

		proxy_set_header X-Forwarded-Proto $scheme;

	}
}
[root@server1 ~]# ansible-playbook /etc/ansible/roles/nginx_loadb.yml
```
## NFS用户的部署

- 下载nfs-utils
- 创建共享文件夹
- 配置nfs配置文件
- 提取网站源码
```bash
[root@server1 ~]# cat /etc/ansible/roles/nfs/tasks/main.yml 
---
# tasks file for nfs
- name: install nfs-utils
  yum: 
    name: nfs-utils
    state: present

- name: create the webdata directory
  file:
    path: /webdata
    state: directory

- name: copy the nfs configuration to the nfs server
  template:
    src: /etc/ansible/roles/nfs/templates/exports.j2
    dest: /etc/exports

- name: extract the web code to webdata
  unarchive:
    src: /etc/ansible/roles/nfs/files/latest-zh_CN.tar.gz
    dest: /webdata

- name: restart the nfs service
  service:
    name: nfs-server
    state: restarted
[root@server1 ~]# cat /etc/ansible/roles/nfs_config.yml 
- name: config the nfs role
  hosts: nfs

  roles:
  - nfs
[root@server1 ~]# ansible-playbook /etc/ansible/roles/nfs_config.yml
```
## apache网站源码目录的挂载

- 下载nfs-utils
- 挂载nfs目录
```bash
[root@server1 ~]# cat /etc/ansible/roles/mount/tasks/main.yml 
---
# tasks file for mount
- name: yum install nfs-utils
  yum: name=nfs-utils state=present
- name: mount the nfs file to apache server
  mount:
    src: 192.168.60.60:/webdata/wordpress 
    path: /var/www/html
    fstype: nfs
    state: mounted
[root@server1 ~]# cat /etc/ansible/roles/mount_nfs.yml 
- name: mount the nfs files
  hosts: apache

  roles:
  - mount
[root@server1 ~]# ansible-playbook /etc/ansible/roles/mount_nfs.yml
```
## 汇总为一个yml文件

- 将各个角色的任务汇总到一个文件
```bash
[root@server1 roles]# cat all.yml 
- name: config host file
  hosts: all_ip
  remote_user: root
  tasks:
  - name: copy host.j2 to group servers
    template:
      src: /root/template/hosts.j2
      dest: /etc/hosts
- name: disable all the firewalld
  hosts: all_ip
  remote_user: root
  tasks:
  - name: disable all the firewalld
    service:
      name: firewalld
      state: stopped

- name: update yum.repo file
  hosts: all_hostname
  remote_user: root
  tasks:
  - include_tasks: /etc/ansible/roles/aliyum/tasks/main.yml

- name: install nginx
  hosts: nginx
  remote_user: root
  tasks:
  - include_tasks: /etc/ansible/roles/nginx/tasks/main.yml

- name: install apache
  hosts: apache
  remote_user: root
  tasks:
  - include_tasks: /etc/ansible/roles/apache/tasks/main.yml

- name: install mariadb
  hosts: mariadb
  remote_user: root
  tasks:
  - include_tasks: /etc/ansible/roles/mariadb/tasks/main.yml

- name: config the nginx load balance role
  hosts: nginx
  remote_user: root
  tasks: 
  - include_tasks: /etc/ansible/roles/nginx_lb/tasks/main.yml

- name: config the nfs role
  hosts: nfs
  remote_user: root
  tasks:
  - include_tasks: /etc/ansible/roles/nfs/tasks/main.yml

- name: mount apache roles directories
  hosts: apache
  remote_user: root
  tasks: 
  - include_tasks: /etc/ansible/roles/mount/tasks/main.yml
[root@server1 roles]# ansible-playbook all.yml 
```
# 结果展示

- 成功安装

![image.png](https://cdn.nlark.com/yuque/0/2023/png/21997074/1678261925293-a1a2821e-0c70-428d-b1a9-be217a5c13c0.png#averageHue=%23daf8f6&clientId=ua128e9df-3d10-4&from=paste&height=753&id=u7c9c6679&name=image.png&originHeight=941&originWidth=1920&originalType=binary&ratio=1.25&rotation=0&showTitle=false&size=221884&status=done&style=none&taskId=uaad1ef31-c0d7-4df9-a597-9ed64babb94&title=&width=1536)
