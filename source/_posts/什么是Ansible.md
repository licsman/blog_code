---
title: 什么是 Ansible 
date: 2020-11-18 10:38:46
categories: linux运维
tags: [自动化安装部署] #文章标签，可空，多标签请用格式，注意:后面有个空格
description: Ansible 是一个简单，强大且无代理的自动化语言。
---



![image-20210118104822515](/images/image-20210118104822515.png)

<!--more-->

# 1. 什么是Ansible

Ansible 是一个简单，强大且无代理的自动化语言。

Ansible 的好处：

简单易读：基于 YAML 文本编写，易于阅读，非专业的开发人员也可以编写。

功能强大：它可以同于管理配置，软件安装，流程自动化

无代理：不需要在客户端安装额外的 agent

跨平台支持：支持 linux，Windows，Unix 和网络设备

# 2. Ansible 是如何工作的

Ansible 典型的工作方式是通过一个脚本文件（基于 YAML 格式构建的）去控制远端操作系统按照特定的顺序执行相关任务，我们称这个文件为 playbook；

## 2.1 架构

**节点：**Ansible 架构中拥有两种计算机类型，即控制节点和受控节点。Ansible 运行在控制节点上，并且只能运行在 linux 操作系统上，对于被控节点，可以是主机设备，也可以是网络设备，主机设备的操作系统，可以是 Windows，也可以是 linux。

![图片](/images/640.jpg)

**清单（inventory）：** 受控节点设备的列表。在这个列表中，你可以根据某些标准（如，作用，服务等）将拥有相同属性的计算机组织到一个组中。Ansible 清单，支持静态清单（一旦定义好，除非你修改配置文件，不然不会发生改变。），也支持动态清单（通过脚本从外部源获取清单，该清单可以随着环境的改变而改变。）。

**Playbook：** 需要在被控节点主机上运行的任务列表，从而让他们达到我们预期的状态。Playbook 中包含一个或多个 play，每个 play 会在一组或多组计算机上按顺序执行已经定义好的一系列 task，每个 task 都是一个执行模块。

**模块（Module）：** 用于确保主机处于某一个特定的状态，例如可以使用 **yum**（对于不同发行版本的 Linux，模块名可能有所不同个，如，在 Ubuntu 中与之对应的是 **apt** 模块。） 模块确保主机已经安装了某个软件，如果主机状态已经是预期的了（已经安装了该软件），那么就不会执行任何操作，执行下一个模块（task）。

**Plugin** 添加到 Ansible 中的代码段，用于扩展 Ansible 平台

# 3. 安装 Ansible

在 Ubuntu 上安装 ansible

```
it@workstation:~$ sudo apt install -y ansible
```

安装完成后，你可以通过 **ansible --version** 查看 ansible 的版本信息

```
it@workstation:~$ ansible --version
ansible 2.9.6
  config file = /etc/ansible/ansible.cfg
  configured module search path = ['/home/it/.ansible/plugins/modules', '/usr/share/ansible/plugins/modules']
  ansible python module location = /usr/lib/python3/dist-packages/ansible
  executable location = /usr/bin/ansible
  python version = 3.8.2 (default, Jul 16 2020, 14:00:26) [GCC 9.3.0]
```

# 4. 配置 Ansible

Ansible 提供的默认主机配置文件：

```
it@workstation:~$ ls -l /etc/ansible/ansible.cfg 
-rw-r--r-- 1 root root 19985 3月   5  2020 /etc/ansible/ansible.cfg
```

\* 只有 root 用户才有权限编辑。

创建新的主机配置文件：

一般情况，我们直接复制默认配置文件，然后根据需要修改复制过来的配置文件。

```
it@workstation:~$ mkdir ansible
it@workstation:~$ cp /etc/ansible/ansible.cfg ansible/
it@workstation:~$ ls -l ansible/
total 24
-rw-r--r-- 1 it it 19985 12月 29 15:03 ansible.cfg
```

\*  在创建新的配置文件之前，我们首先需要创建一个用于测试 Ansible 的工作目录，然后再创建配置文件。

当我们拥有多个配置文件时，配置文件生效的顺序为：

- 当前目录下：./ansible.cfg
- home 目录下：~/ansible.cfg
- 默认配置文件：/etc/ansible/ansible.cfg

复制完成后，我们可以通过 **ansible --version** 查看当前生效的配置文件

```
it@workstation:~$ cd ansible/
it@workstation:~/ansible$ ansible --version
ansible 2.9.6
  config file = /home/it/ansible/ansible.cfg
  configured module search path = ['/home/it/.ansible/plugins/modules', '/usr/share/ansible/plugins/modules']
  ansible python module location = /usr/lib/python3/dist-packages/ansible
  executable location = /usr/bin/ansible
  python version = 3.8.2 (default, Jul 16 2020, 14:00:26) [GCC 9.3.0]
```

\*  你需要切换到 Ansible 的工作目录，不然 Ansible 工作目录下的配置文件是无效的。

对于默认配置文件，我们当前需要了解的有两个模块：**defaults** 和 **privilege_escalation**。

defaults 模块：

**inventory：** 指定清单文件路径

**host_key_checking ：** 是否检查主机 key 是否可信

**remote_user：** 远程连接时使用的用户，默认使用当前用户

**ask_pass：** 连接时是否询问输入密码（如果没有配置密钥登录，需要配置该选项为 true。）

privilege_escalation 模块：sudo 提权相关的配置

**become：** 是否开启切换用户

**become_method：** 如何切换用户

**become_user：** 切换到那个用户

**become_ask_pass：** 是否提示输入密码

更改配置文件

```
it@workstation:~/ansible$ vim ansible.cfg 
it@workstation:~/ansible$ grep -vE '^$|#' ansible.cfg 
[defaults]
inventory      = /home/it/ansible/hosts
host_key_checking = False
remote_user = it
ask_pass      = True
[inventory]
[privilege_escalation]
become=True
become_method=sudo
become_user=root
become_ask_pass=True
[paramiko_connection]
[ssh_connection]
[persistent_connection]
[accelerate]
[selinux]
[colors]
[diff]
```

# 3. 清单（Inventory）

Ansible 提供了一个示例清单文件，并给我们提供了一些常规的示例:

**EX 1：** 将未分组的主机放在所有组的前面的；

**EX 2：** 通过 **[ ]** 指定主机组的名称，如，webservers，下面直到下一个组名（dbservers）前结束的都是属于该组的主机，即使主机与主机之间存在空行，但如没有到下一个组名，他们依然属于同一个主机组；如果某些主机之间有某些顺序关系，你可以通过简写，将他们放到同一行，如示例中的 “www[001:006].example.com”，分别表示 www001.example.com、www002.example.com、www003.example.com、www004.example.com、www005.example.com 和 www006.example.com。

**EX 3：** 提供了另一种主机范围的示例

```
it@workstation:~$ cat /etc/ansible/hosts 
# This is the default ansible 'hosts' file.
#
# It should live in /etc/ansible/hosts
#
#   - Comments begin with the '#' character
#   - Blank lines are ignored
#   - Groups of hosts are delimited by [header] elements
#   - You can enter hostnames or ip addresses
#   - A hostname/ip can be a member of multiple groups

# Ex 1: Ungrouped hosts, specify before any group headers.

#green.example.com
#blue.example.com
#192.168.100.1
#192.168.100.10

# Ex 2: A collection of hosts belonging to the 'webservers' group

#[webservers]
#alpha.example.org
#beta.example.org
#192.168.1.100
#192.168.1.110

# If you have multiple hosts following a pattern you can specify
# them like this:

#www[001:006].example.com

# Ex 3: A collection of database servers in the 'dbservers' group

#[dbservers]
#
#db01.intranet.mydomain.net
#db02.intranet.mydomain.net
#10.25.1.56
#10.25.1.57

# Here's another example of host ranges, this time there are no
# leading 0s:

#db-[99:101]-node.example.com

it@workstation:~$ 
```

创建自己的主机清单

```
it@workstation:~$ vim ansible/hosts
it@workstation:~$ cat ansible/hosts
serverb

[web]
servera

[prod:children]
web
```

在该清单中，我们使用组嵌套，这个是前面示例中没有的，web 组是 prod 组的子组。

我们可以通过 **ansible** 命令查看主机清单内容：

```
it@workstation:~/ansible$ ansible web --list-host
SSH password: 
BECOME password[defaults to SSH password]: 
  hosts (1):
    servera
it@workstation:~/ansible$ ansible prod --list-host
SSH password: 
BECOME password[defaults to SSH password]: 
  hosts (1):
    servera
```

我们可以通过 **ansible** 命令来验证我们的主机清单文件

同时 Ansible 还有两个默认组：

**all：** 表示所有组件

**ungrouped：** 表示所有未分组的主机

```
it@workstation:~/ansible$ ansible all --list-host
SSH password: 
BECOME password[defaults to SSH password]: 
  hosts (2):
    serverb
    servera
it@workstation:~/ansible$ ansible ungrouped --list-host
SSH password: 
BECOME password[defaults to SSH password]: 
  hosts (1):
    serverb
```

\*  在 Ansible 中，主机可以同时属于多个组。

Ansible 中存在一个隐藏的主机 localhost，即 ansible 本身，当主机指定为 localhost 时，ansible 会自动忽略掉 remote_user 配置；

# 4. 运行 ansible 临时命令

Ansible 临时命令可以快速执行单个 ansible 任务，而不需编写 playbook，这对测试和排错很有帮助。如，你可以使用临时命令去检查，某个软件包在主机上的状态是什么，是否可用。

通过临时命令查看连接到远程主机的用户信息

```
it@workstation:~/ansible$ ansible servera -m shell -a "id"
SSH password: 
BECOME password[defaults to SSH password]: 
servera | CHANGED | rc=0 >>
uid=0(root) gid=0(root) groups=0(root)
```

\*  由于我们没有配置免密（密钥），所以这里需要我们输入输入两次密码，一次时 ssh 连接的密码，一次是 sudo 提权的密码；

\*  如果使用 ssh 密码方式运行 ansible，你还需要安装 sshpass，不然会有报错;

```
it@workstation:~/ansible$ ansible servera -m shell -a "id"
SSH password: 
BECOME password[defaults to SSH password]: 
servera | FAILED | rc=-1 >>
to use the 'ssh' connection type with passwords, you must install the sshpass program
```

安装 sshpass

```
it@workstation:~/ansible$ sudo apt install sshpass
```