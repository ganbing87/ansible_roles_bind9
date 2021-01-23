# 项目说明
- 通过ansbile一键自动化部署bind9 dns服务，项目其包含两大功能：初始化系统操作（如关闭防火墙及selinux、安装jdk、修改ulimit参数、修改sysctl参数等）、安装docker及启动bind服务容器。
- 适用平台centos、redhat系统。
- 本项目是基于ansible roles角色的结构来实现的。

# ansible roles说明
  角色（roles）是ansible自1.2版本开始引入的新特性，用于层次性，结构化地组织playbook.  
  
  roles能够根据层次型结构自动装载变量文件、tasks以及handlers等。要使用roles只需要在playbook中使用include指令即可。简单的说，roles就是通过分别将变量、文件、任务、模块及处理器放置于单独的目录中、并可以便捷地include他们的一种机制。角色一般用于基于主机构建服务的场景中、但也可以是用于构建守护进程等场景中。  
  
# 目录结构说明
- group_vars：存放变量的目录
- inventories：主机清单目录
- bind.yml: ansbile-playbook调用的入口文件
- roles：角色目录
  - env_init：自定义的初始化系统角色目录，此角色里面包含了创建用户、安装jdk、修改resolve.conf、关闭防火墙及selinux、修改ulimit等功能的其它角色；
    - createdir: 创建安装目录角色
    - createuser： 创建用户的目录角色
    - jdk：安装jdk 1.8的目录角色
    - modify_resolv.conf：修改dns配置的目录角色
    - system：设置系统防火墙、sysctl、ulimit参数等配置的目录角色
    - tasks： 至少应该包含一个名为main.yml的文件，其定义了此角色的任务列表；此文件可以使用include包含其他的位于此目录中（如引用createdir、createuser等）的task文件；
  - bind9：bind服务角色目录
    - defaults： 为当前角色设定默认变量时使用此目录；应当包含一个main.yml文件；
    - tasks：至少应该包含一个名为main.yml的文件，其定义了此角色的任务列表；此文件可以使用include包含其他的位于此目录中的task文件；
    - templates：templates模块会自动在此目录中寻找Jinja2模板文件；
 
***

# 安装步骤
以下安装以centos 7为例进行操作，以下操作均在ansible机器上操作.  
## 1、准备ansilbe环境
### 1.1 安装ansible
```
#备份旧yum源
mv /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.backup

#下载新的CentOS-Base.repo 到/etc/yum.repos.d/
wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo
wget -O /etc/yum.repos.d/epel.repo http://mirrors.aliyun.com/repo/epel-7.repo
yum clean all

#安装依赖包和工具包
yum -y python-setuptools python2-pip bash-completion bind-utils net-tools iproute telnet curl wget nmap vim jq gettext e2fsprogs

#安装ansible
yum -y install ansible
```

### 1.2 安装community.general模块
```
#安装ansible通用模块，后面用调用docker相关的模块，这是在线安装
ansible-galaxy collection install community.general
```
安装完成后，会把模块生成到/root/.ansible/collections/ansible_collections目录下.  
模块安装参说明，参考相关网站：
  - https://docs.ansible.com/ansible/latest/collections/community/general/docker_container_info_module.html
  - https://www.cnblogs.com/lisenlin/p/10919068.html


## 2、创建安装包目录、上传软件包
```
# mkdir -p /data/packages

# ll
总用量 515280
-rw-r--r--. 1 root root 255131648 1月  22 20:11 bind-9.16.tar
-rw-r--r--. 1 root root  43834194 1月  22 20:10 docker-18.06.2-ce.tgz
-rw-r--r--. 1 root root 194042837 1月  22 20:12 jdk-8u202-linux-x64.tar.gz
-rw-r--r--. 1 root root  34632262 1月  22 20:12 python-virtualenv.tar.gz
[root@localhost packages]# 
```

> 软件包请联系笔者下载（QQ 312683629）

## 3、下载此项目
```
git clone https://github.com/ganbing87/ansible_roles_bind9.git
```

## 4、修改ansible.cfg
修改host_key_checking = False ，其作用是取消对此的注释以禁用SSH密钥主机检查，如果ansbile管理主机和被管理端没有做ssh_key信任，把此参数打开，可以用密码进行管理主机
```
# vim /etc/ansible/ansible.cfg 
host_key_checking = False
```

## 5、修改inventories/hosts文件
```
# vim inventories/hosts
[bind9]
192.168.245.129 ansible_port=22 ansible_ssh_user=root ansible_ssh_pass=123456

[nginx]
192.168.245.129 ansible_port=22 ansible_ssh_user=root ansible_ssh_pass=123456

[mysql]
mysql-master ansible_host=192.168.245.129 ansible_port=22 ansible_ssh_user=root ansible_ssh_pass=123456
```
说明：此项目是安装bind，但是也多了nginx、mysql这两个主机的配置，主要是后面安装好bind之后，会关联这两个主机的信息写入到DNS记录中，便于测试用，上面主要修改IP、密码即可进行测试。这3台主机信息的IP可以同时指向一台机器，也可以指向不同的机器。

## 6、修改全局变量文件all.yml
```
# vim group_vars/all.yml
#--------------------Global Configuration-------------------------
install_dir: "/export/server"     # 被管理主机，安装程序软件的目录
softlink_dir: "/export"           # 被管理主机，软件程序软链接目录
local_packages_dir: "/data/packages"        # ansilbe主机，本地安装包上传目录，需要手动创建此目录
remote_packages_dir: "/export/packages"     # 被管理主机，安装包存放目录
ssh_user: "root"                # 用于ansible主机ssh到被管理主机的用户
appuser: "zhangsan"             # 用于自动在被管理机器上创建的普通用户
limit_nofile: "65536"           # 用于下面system configuratio配置中引用的变量
limit_nproc: "131072"           # 用于下面system configuratio配置中引用的变量
domain: "ganbing.cnn"           # 定义一个测试的一级域名
test01_fqdn_hostname: "test01"    # 定义一个测试的二级域名1
test02_fqdn_hostname: "test02"    # 定义一个测试的二级域名2
mysql_fqdn_hostname: "mysql"      # 定义一个mysql测试的二级域名

#--------------------jdk configuration-------------------------
jdk8_version: "202"   # jdk的版本号，需要核实下jdk包的版本是否和这一致

#-------------------Bind Configuration-------------------------
bind_version: "9.16"                # bind镜像包的版本号
bind9_work_dir: "/export/bind9"     # 定义bind安装后的目录

#--------------------docker configuration-------------------------
docker_version: 18.06.2-ce            # docker镜像包的版本号
docker_data_dir: /export/lib/docker   # 定义docker安装后的目录

#--------------------system configuration-------------------------
kernel_config:
   - name: net.ipv4.ip_forward
     value: 1
   - name: net.ipv4.tcp_tw_recycle
     value: 0
   - name: vm.swappiness
     value: 0
   - name: vm.overcommit_memory
     value: 1
   - name: vm.panic_on_oom
     value: 0
   - name: vm.max_map_count
     value: 262144
   - name: fs.inotify.max_user_instances
     value: 8192
   - name: fs.inotify.max_user_watches
     value: 1048576
   - name: fs.file-max
     value: 52706963
   - name: net.ipv6.conf.all.disable_ipv6
     value: 1
   - name: net.ipv6.conf.default.disable_ipv6
     value: 1
   - name: net.ipv6.conf.lo.disable_ipv6
     value: 1

ulmit_config:
  - limit_domain: "*"
    limit_type: "soft"
    limit_item: "nofile"
    limit_value: "{{ limit_nofile }}"
  - limit_domain: "*"
    limit_type: "hard"
    limit_item: "nofile"
    limit_value: "{{ limit_nofile }}"
  - limit_domain: "*"
    limit_type: "soft"
    limit_item: "nproc"
    limit_value: "{{ limit_nproc }}"
  - limit_domain: "*"
    limit_type: "hard"
    limit_item: "nproc"
    limit_value: "{{ limit_nproc }}"

```
说明：主要修改-Global Configuratio 相关的配置信息，


## 7、执行ansible-playbook命令
说明：执行命令前，先检查端口TCP和UDP的53是否被占用，如被占用，需杀掉相关进程，或者修改roles/bind9/tasks/start.yml文件，把53端口改为未被其它程序占用的端口.  
```
ansible-playbook -i inventories/hosts  bind.yml  -v
```

## 8、验证运行结果
进入被管理主机通过以下方式进行结果验证：
```
# ping配置的域名是否通
ping mysql.ganbing.cnn

# 查看docker信息，如bind镜像，bind容器是否启动
docker ps
docker images

# 查看/etc/sysctl.conf文件，是否写入了相关配置
cat /etc/sysctl.conf

# 查看limits.conf文件，是否写入了相关配置
cat /etc/security/limits.conf

# 查看selinux是否禁用
getenforce

# 查看防火墙是否关闭
systemctl status firewalld
```
### 小结
> 如果您下载此项目，测试过后，觉得还有用，请支持一下辛苦的工作，多少随您的心意，您的支持将是我继续开源的动力，感谢提出宝贵的意见!
![支付宝收款码](文档：个人收钱.note
链接：http://note.youdao.com/noteshare?id=d99e0464ab502736f2c48a171a663020&sub=F229386AF6C04EDDB05A9A7B58D552FB)