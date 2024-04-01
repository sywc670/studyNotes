- [ansible 命令](#ansible-命令)
  - [ansible查看插件命令](#ansible查看插件命令)
- [ansible 知识](#ansible-知识)
  - [自定义模块](#自定义模块)
  - [ansible\_runner](#ansible_runner)
  - [ansible modules plugin](#ansible-modules-plugin)
    - [import\_tasks与include\_tasks模块](#import_tasks与include_tasks模块)
  - [jinjia2模板](#jinjia2模板)
  - [role](#role)
  - [ansible ad-hoc传入变量](#ansible-ad-hoc传入变量)
  - [ansible常用模块](#ansible常用模块)
  - [ansible常用结构](#ansible常用结构)
  - [ansible内置变量](#ansible内置变量)
  - [role](#role-1)
- [ansible 设置](#ansible-设置)
  - [ansible cfg 性能优化设置](#ansible-cfg-性能优化设置)
  - [ansible第一次连接不输入yes](#ansible第一次连接不输入yes)
  - [ansible 主机清单设置](#ansible-主机清单设置)
    - [ansible hosts设置yaml格式](#ansible-hosts设置yaml格式)
- [ansible报错](#ansible报错)
  - ["msg": "to use the 'ssh' connection type with passwords, you must install the sshpass program"](#msg-to-use-the-ssh-connection-type-with-passwords-you-must-install-the-sshpass-program)
  - [undefined variable报错](#undefined-variable报错)
- [ansible脚本案例](#ansible脚本案例)
  - [更新服务包](#更新服务包)
  - [初始化centos7模板](#初始化centos7模板)
  - [远程设置密码](#远程设置密码)
- [参考](#参考)

# ansible 命令

```
ansible-playbook --syntax-check /testdir/ansible/test.yml
ansible-playbook --check test.yml
```

## ansible查看插件命令

```
ansible-doc -t lookup -l
ansible-doc -t lookup dict
```

# ansible 知识

## 自定义模块

[ref](https://blog.csdn.net/qq_37326531/article/details/130119675)

自定义模块可以用go\python\shell等语言实现，其实就是拷贝过去执行

## ansible_runner 

[ref](https://ansible.readthedocs.io/projects/runner/en/latest/)

## ansible modules plugin

### import_tasks与include_tasks模块

它们的不同之处在于，”import_tasks”是静态的，”include_tasks”是动态的。

那么动态和静态又是什么意思呢，它们有什么不同呢？动态和静态的主要区别在于被include的文件的加载时机不同。

“静态”的意思就是被include的文件在playbook被加载时就展开了（是预处理的）。

“动态”的意思就是被include的文件在playbook运行时才会被展开（是实时处理的）。

由于”include_tasks”是动态的，所以，被include的文件的文件名可以使用任何变量替换。

由于”import_tasks”是静态的，所以，被include的文件的文件名不能使用动态的变量替换。

我们知道，当使用when关键字对include文件添加了条件判断时，只有条件满足后，include文件中的任务列表才会被执行，其实，when关键字对”include_tasks”和”import_tasks”的实际操作有着本质区别，区别如下：

当对”include_tasks”使用when进行条件判断时，when对应的条件只会应用于”include_tasks”任务本身，当执行被包含的任务时，不会对这些被包含的任务重新进行条件判断。

当对”import_tasks”使用when进行条件判断时，when对应的条件会应用于被include的文件中的每一个任务，当执行被包含的任务时，会对每一个被包含的任务进行同样的条件判断。

与”include_tasks”不同，当为”import_tasks”添加标签时，tags是针对被包含文件中的所有任务生效的，与”include”关键字的效果相同。

https://www.zsythink.net/archives/2977

## jinjia2模板

{{      }}  ：用来装载表达式，比如变量、运算表达式、比较表达式等。

{%   %}   ：用来装载控制语句，比如 if 控制结构，for循环控制结构。

{#    #}   ：用来装载注释，模板文件被渲染后，注释不会包含在最终生成的文件中。

## role 

调用角色会查找的目录为：

同级目录中。

同级目录中的roles目录中。

当前系统用户的家目录中的.ansible/roles目录，即 ~/.ansible/roles目录中。

## ansible ad-hoc传入变量

所有传入的变量都是字符串形式

## ansible常用模块

yum package file template lineinfile command shell blockinfile copy fetch service apt script find replace cron user group yum_repository 
debug setup set_fact unarchive
mysql_db mysql_user
get_url
fail
include_vars
include_tasks import_tasks

## ansible常用结构

gather_facts register ignore_errors
vars_prompt vars_files
handlers meta 
block rescue always
with_list with_together with_cartesian with_indexed_items with_items with_sequence with_random_choice with_dict with_supplements with_file with_fileglob
loop loop_control
when "is exists" failed_when changed_when
include_playbook
roles
delegate_to connection: local run_once: true

## ansible内置变量

hostvars play_hosts ansible_host

## role

tasks目录：角色需要执行的主任务文件放置在此目录中，默认的主任务文件名为main.yml，当调用角色时，默认会执行main.yml文件中的任务，你也可以将其他需要执行的任务文件通过include的方式包含在tasks/main.yml文件中。

handlers目录：当角色需要调用handlers时，默认会在此目录中的main.yml文件中查找对应的handler

defaults目录：角色会使用到的变量可以写入到此目录中的main.yml文件中，通常，defaults/main.yml文件中的变量都用于设置默认值，以便在你没有设置对应变量值时，变量有默认的值可以使用，定义在defaults/main.yml文件中的变量的优先级是最低的。

vars目录：角色会使用到的变量可以写入到此目录中的main.yml文件中，看到这里你肯定会有疑问，vars/main.yml文件和defaults/main.yml文件的区别在哪里呢？区别就是，defaults/main.yml文件中的变量的优先级是最低的，而vars/main.yml文件中的变量的优先级非常高，如果你只是想提供一个默认的配置，那么你可以把对应的变量定义在defaults/main.yml中，如果你想要确保别人在调用角色时，使用的值就是你指定的值，则可以将变量定义在vars/main.yml中，因为定义在vars/main.yml文件中的变量的优先级非常高，所以其值比较难以覆盖。

meta目录：如果你想要赋予这个角色一些元数据，则可以将元数据写入到meta/main.yml文件中，这些元数据用于描述角色的相关属性，比如 作者信息、角色主要作用等等，你也可以在meta/main.yml文件中定义这个角色依赖于哪些其他角色，或者改变角色的默认调用设定，在之后会有一些实际的示例，此处不用纠结。

templates目录： 角色相关的模板文件可以放置在此目录中，当使用角色相关的模板时，如果没有指定路径，会默认从此目录中查找对应名称的模板文件。

files目录：角色可能会用到的一些其他文件可以放置在此目录中，比如，当你定义nginx角色时，需要配置https，那么相关的证书文件即可放置在此目录中。

 
当然，上述目录并不全是必须的，也就是说，如果你的角色并没有相关的模板文件，那么角色目录中并不用包含templates目录，同理，其他目录也一样，一般情况下，都至少会有一个tasks目录。

# ansible 设置

## ansible cfg 性能优化设置

```
mkdir /etc/ansible/
cat << EOF > /etc/ansible/ansible.cfg
[defaults]
host_key_checking=False
pipelining=True
forks=100
EOF
```

默认情况下 Ansible 执行过程中会把生成好的本地 python 脚本文件 PUT 到 远端机器。如果我们开启了 ssh 的 pipelining 特性，这个过程就会在 SSH 的会话中进行

## ansible第一次连接不输入yes

```
vi /etc/ansible/ansible.cfg
host_key_checking = False 
```

## ansible 主机清单设置

**ansible_port** 用于配置对应主机上的sshd服务端口号，在实际的生产环境中，各个主机的端口号通常不会使用默认的22号端口，所以用此参数指定对应端口。

**ansible_user** 用于配置连接到对应主机时所使用的用户名称。

**ansible_ssh_pass** 用于配置对应用户的连接密码。

### ansible hosts设置yaml格式

```yaml
all:
  children:
    lo:
      hosts:
        local:
          ansible_host: 127.0.0.1
    app:
      hosts:
        app1:
          ansible_host: 192.168.60.4
          ansible_port: 22
        app2:
          ansible_host: 192.168.60.5
          ansible_port: 22
        app3:
          ansible_host: 192.168.60.6
          ansible_port: 22     
      vars:
        ansible_ssh_user: vagrant
        ansible_ssh_private_key_file: /home/will/.vagrant.d/insecure_private_key  
        #ansible_ssh_pass: xx
        #ansible_python_interpreter: /usr/bin/python3
```

# ansible报错

## "msg": "to use the 'ssh' connection type with passwords, you must install the sshpass program"

yum install sshpass -y

## undefined variable报错

如果使用--check 那么shell返回值得来的变量就无法执行会报错
且shell返回值取一个field，得用`var.field`而不是`var[field]`



# ansible脚本案例

## 更新服务包

```yaml
---
# 存在的问题：copy会将文件夹移动到对应目录内
# 未来升级点：
# 配置文件使用template提供
# 添加其他服务，xxx启动过程
- hosts: server100 # 在/etc/ansible/hosts中设置 不允许以数字开头
  become: yes
  vars_prompt: 
    - name: "xxx_package"
      prompt: "where xxx package lies"
  vars: 
    - name: 
  tasks: 
    - name: produce date
      shell: "date +%F"
      register: date
      tags: always
    - name: debug msg
      debug:
        msg: /opt/onebackup/data-{{date.stdout}}
      tags: always

    - name: backup data
      copy:
        remote_src: yes
        src: /opt/know/nodeServer/data
        dest: /opt/onebackup/data-{{date.stdout}}
      tags: data
    - name: deploy data
      copy:
        remote_src: yes
        src: /opt/know/tmp/{{one_package}}/data-develop
        dest: /opt/know/nodeServer/data
      tags: data
    - name: revise data
      copy:
        remote_src: yes
        src: /opt/onebackup/data-{{date.stdout}}/config.js 
        dest: /opt/know/nodeServer/data/config.js
      tags: data

    - name: backup one
      copy:
        remote_src: yes
        src: /opt/know/apache-tomcat-8.0.30/webapps/one
        dest: /opt/onebackup/one-{{date.stdout}}
      tags: one
    - name: deploy one
      copy:
        remote_src: yes
        src: /opt/tmp/{{one_package}}/one-develop
        dest: /opt/apache-tomcat-8.0.30/webapps/one
      tags: one
    - name: revise one
      copy:
        remote_src: yes
        src: /opt/{{date.stdout}}/js/config.js 
        dest: /optapache-tomcat-8.0.30/webapps/one/js/config.js
      tags: one

      
```



## 初始化centos7模板

```ini
[aliBase]
name=aliBase
baseurl=https://mirrors.aliyun.com/centos/$releasever/os/$basearch/
enabled=1
gpgcheck=1
gpgkey=https://mirrors.aliyun.com/centos/$releasever/os/$basearch/RPM-GPG-KEY-CentOS-$releasever
[aliEpel]
name=aliEpel
baseurl=https://mirrors.aliyun.com/epel/$releasever\Server/$basearch/
enabled=1
gpgcheck=0
```
```yaml
#centos7
#固定IP功能
---
- hosts: all
  become: yes
  tasks: 
    - name: download aliyun yum repofile 
      get_url:
        dest: /etc/yum.repos.d/CentOS-Base.repo
        url: https://mirrors.aliyun.com/repo/Centos-7.repo
        force: yes 
    - name: install packages
      yum:
        name: 
          - curl
          - bash-completion
          - net-tools 
          - tree
          - mlocate
          - vim-enhanced
          - epel-release
          - bind-utils
          - strace
          - sysstat
          - pv
          - git
          - tcpdump
          - yum-utils
          - proxychains-ng
          - iproute2
        state: present

    - name: vim conf edit
      blockinfile:
        path: /etc/vimrc
        block: |
          set showcmd
          set encoding=utf-8
          set wrap
          set tabstop=4
        marker: "\" {mark} ANSIBLE BLOCK"
```

## 远程设置密码
需要pip install passlib
```yaml
---
- hosts: test70
  remote_user: root
  vars_prompt:
    - name: "user_name"
      prompt: "Enter user name"
      private: no
    - name: "user_password"
      prompt: "Enter user password"
      encrypt: "sha512_crypt"
      confirm: yes
  tasks:
   - name: create user
     user:
      name: "{{user_name}}"
      password: "{{user_password}}"
```

# 参考

[ref-ansible](https://www.zsythink.net/archives/category/%e8%bf%90%e7%bb%b4%e7%9b%b8%e5%85%b3/ansible)