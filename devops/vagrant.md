# vagrant安装启动流程
vagrant up用hyperv总是会报错，所以无法使用
wget https://app.vagrantup.com/centos/boxes/7/versions/2004.01/providers/virtualbox.box
vagrant add box centos/7 virtualbox.box #这里名字必须为官方名字
vagrant init centos/7
vagrant up

**注意ansible无法装在windows系统**

# ubuntu安装vagrant

```
wget -O- https://apt.releases.hashicorp.com/gpg | gpg --dearmor | sudo tee /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install vagrant
```

# vagrant wsl virtualbox windows报错
```
There was an error when attempting to rsync a synced folder.
Please inspect the error message below for more info.

Host path: /mnt/c/Users/will/projects/centos/
Guest path: /vagrant
Command: "rsync" "--verbose" "--archive" "--delete" "-z" "--copy-links" "--no-owner" "--no-group" "--rsync-path" "sudo rsync" "-e" "ssh -p 2222 -o LogLevel=FATAL   -o ControlMaster=auto -o ControlPath=/tmp/vagrant-rsync-20220731-3359-1vdatu9 -o ControlPersist=10m  -o IdentitiesOnly=yes -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null -i '/mnt/c/Users/will/projects/centos/.vagrant/machines/default/virtualbox/private_key'" "--exclude" ".vagrant/" "/mnt/c/Users/will/projects/centos/" "vagrant@172.26.176.1:/vagrant"
Error: vagrant@172.26.176.1: Permission denied (publickey,gssapi-keyex,gssapi-with-mic).
rsync: connection unexpectedly closed (0 bytes received so far) [sender]
rsync error: unexplained error (code 255) at io.c(235) [sender=3.1.3]

解决方案：
export PATH="$PATH:/mnt/c/Program Files/Oracle/VirtualBox"
export VAGRANT_WSL_ENABLE_WINDOWS_ACCESS="1"
export VAGRANT_HOME="/home/<user>/.vagrant.d"
Vagrantfile中修改
config.ssh.insert_key = false

参考：
https://blog.thenets.org/how-to-run-vagrant-on-wsl-2/
https://github.com/Karandash8/virtualbox_WSL2
https://stackoverflow.com/questions/46150672/vagrant-wsl-rsync-and-ssh-permission-error/46154824#46154824
https://zhuanlan.zhihu.com/p/259833884
```

# 搭建vagrant+virtualbox环境

vagrant plugin install vagrant-vbguest

By default, Vagrant shares your project directory (the one containing the Vagrantfile) to the /vagrant directory in your guest machine.