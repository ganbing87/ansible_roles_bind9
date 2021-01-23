# 项目说明
- 通过ansbile一键自动化部署bind9 dns服务。
- 适用平台centos、redhat系统。
- 本项目是基于ansible roles角色的结构来实现的。

# ansible roles说明
  角色（roles）是ansible自1.2版本开始引入的新特性，用于层次性，结构化地组织playbook.  
  
  roles能够根据层次型结构自动装载变量文件、tasks以及handlers等。要使用roles只需要在playbook中使用include指令即可。简单的说，roles就是通过分别将变量、文件、任务、模块及处理器放置于单独的目录中、并可以便捷地include他们的一种机制。角色一般用于基于主机构建服务的场景中、但也可以是用于构建守护进程等场景中。  
  
# 目录结构说明
- group_vars：存放变量的目录
- bind9.yml: ansbile-playbook调用的入口文件
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
以下安装以centos 7为例进行操作.  
## 1、准备ansilbe环境
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


## 2、创建安装包上传目录
```
# mkdir -p /data/packages
```
