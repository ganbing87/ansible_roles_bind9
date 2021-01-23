# 项目说明
- 通过ansbile一键自动化部署bind9 dns服务。
- 适用平台centos、redhat系统。
- 本项目是基于ansible roles角色的结构来实现的。
***

# ansilb roles说明
  角色（roles）是ansible自1.2版本开始引入的新特性，用于层次性，结构化地组织playbook.  
  roles能够根据层次型结构自动装载变量文件、tasks以及handlers等。要使用roles只需要在playbook中使用include指令即可。简单的说，roles就是通过分别将变量、文件、任务、模块及处理器放置于单独的目录中、并可以便捷地include他们的一种机制。角色一般用于基于主机构建服务的场景中、但也可以是用于构建守护进程等场景中。  
  
