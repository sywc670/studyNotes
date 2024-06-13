## linux命令

### jq

[ref](https://www.iplaysoft.com/tools/linux-command/c/jq.html)

```shell
kubectl get pods -A -o json | jq -r '.items[] | select(.status.phase != "Running") | "kubectl delete pods \(.metadata.name) -n \(.metadata.namespace) "' | xargs -n 1 bash -c
```

每次通过过滤，再管道符后，.会变成过滤后的条件，而select之后不会变

`select(foo)`: 如果foo返回true，则输入保持不变

`map(foo)`: 每个输入调用过滤器

`\(foo)`: 在字符串中插入值并进行运算

`keys`: 取出数组中的键

`length`: 计算一个值的长度

-r 输出原始字符串，而不是JSON文本：json输出有字符串类型，如果加上-r，输出的字符串类型不会有额外的双引号

### xargs

```shell
ls *.jpg | xargs -n1 -I {} cp {} /data/images # 复制
-I # 不仅是替换，还让每个参数都运行一次命令，注意会隐含-L，每次只读取一行
-p # 确认
cat url-list.txt | xargs wget -c # 多行下载
# 通过 xargs 的处理，换行和空白将被空格取代

xargs -n 1 bash -c
# 输入的每一条命令都执行一遍
```
### dd命令

对于 dd 命令来说，除了 if、of 两个选项之外，还应该掌握下面这两个重要选项：
bs=N：设置单次读入或单次输出的数据块（block）的大小为 N 个字节。当然也可以使用 ibs 和 obs 选项来分别设置。
ibs=N：单次读入的数据块（block）的大小为 N 个字节，默认为 512 字节。
obs=N：单次输出的数据块（block）的大小为 N 个字节，默认为 512 字节。
count=N：表示总共要复制 N 个数据块（block）。

if=file	输入文件名，缺省为标准输入。 从file读取，如if=/dev/zero
of=file	输出文件名，缺省为标准输出。 向file写出，可以写文件，可以写裸设备。如of=/dev/null
ibs=bytes	一次读入 bytes 个字节(即一个块大小为 bytes 个字节)
obs=bytes	一次写 bytes 个字节(即一个块大小为 bytes 个字节)
bs=bytes	同时设置读写块的大小为 bytes
seek=blocks	从输出文件开头跳过 blocks 个块后再开始复制
count=blocks	仅拷贝 blocks 个块，块大小等于 ibs 指定的字节数

原文链接：https://blog.csdn.net/sssssuuuuu666/article/details/106999198/

#### 使用dd复制硬盘

复制系统盘有很大风险，系统配置可能不同导致无法启动或者出错

sudo  fdisk -u -l
可以查看所有磁盘上的所有分区的尺寸和布局情况。
-u，让start和end中数字的单位是512字节，也就是一个sector扇区的大小。

[dd复制整个系统教程](https://blog.csdn.net/zhaominyong/article/details/123705337)

具体步骤
找一个U盘，安装UbuntuLive Cd系统。【具体如何制作U盘启动的UbuntuLive CD，可以参考Ubuntu官方网站的帮助。】
UbuntuLive Cd和WindowsPE系统类似，是光盘/U盘引导的Ubuntu操作系统，不需要安装就可以直接使用。
U盘启动，进入盘上的Ubuntu系统，打开命令行，执行：

sudo  fdisk -u -l /dev/sda
查看硬件的分区情况。
然后执行：

`dd   bs=512 count=[fdisk命令中最大的end数+1] if=/dev/sda of=/ghost.img`
这样，就可以把我需要的分区数据全部copy到ghost.img文件中。镜像制作完成了！
然后，我们就可以把U盘插到其他系统上，用U盘启动，进入UbuntuLiveCD，打开命令行，执行如下命令：

dd if=/ghost.img of=/dev/sda
完成后，拔掉U盘，启动计算机，就可以看到我们的Linux系统已经安装完毕了！

注意：
**不要直接在计算机上用本地磁盘启动系统后执行dd命令生成本地磁盘的镜像。而应该使用livecd启动计算机。因此计算机运行时会对系统盘产生大量写操作。 直接对运行中的系统盘生成的镜像，在恢复到其他硬盘上时，很可能会无法启动！**

一样适用于非Linux操作系统
在linux上用dd命令实现系统镜像备份和恢复，是不是很简单呢？

对于Windows系统，甚至Mac等等任意系统，其实都可以用dd命令实现系统镜像的备份和恢复。
因为，Linux的fdisk命令能够识别任意系统下的分区格式。fdisk并不关系分区上的文件系统，甚至有无文件系统都不关心。fdisk总是可以报告分区占用了哪些扇区。
dd命令也不关心磁盘的文件系统格式，它只是简单地按照要求从指定的位置，复制多少字节数据而已。
dd命令实现镜像备份和恢复，比Ghost软件简单和强大多了。使用ghost软件，依然需要用户进行复杂而危险的磁盘分区操作。
而使用fdisk和dd这两条命令，一切都免了！

压缩和解压缩
可能我们需要备份的分区很大，使用dd命令生成的镜像文件也就很大。存储和传输这些镜像不太方便。 我们也可以使用压缩程序压缩生成的镜像文件。 这里，我选择使用gzip程序，配合dd命令一起使用。

gzip参数：
-c 表示输出到stdout
-d 表示解压缩
-1 表示最快压缩
-9 表示最好压缩
默认使用的是-6压缩级别。

要使用 dd 和 gzip 生成压缩的镜像文件，可以执行命令：

`dd   bs=512 count=[fdisk命令中最大的end数+1] if=/dev/sda | gzip -6 > /ghost.img.gz`
还原时，可以执行下列命令：

gzip -dc /ghost.img.gz.gz | dd of=/dev/sda

提醒：
如果你把镜像恢复到另一台计算机上，你可能会发现你的网卡是eth1，而不是eth0。这是因为
/etc/udev/rules.d/70-persistent-net.rules 文件把你做镜像的计算机的网卡作为eth0登记了。
如果你的网络脚本对eth0进行了处理，而没有对eth1进行处理，那么不修改网络脚本，你可能就无法上网了。

也许你会希望在做镜像之前，先删除 /etc/udev/rules.d/70-persistent-net.rules 文件。这样你恢复镜像时，网卡的名字就是eth0。 就不会造成你在恢复后的计算机上无法上网的问题了。


#### 使用dd复制硬盘第二例（存有疑问）

[使用dd复制硬盘](https://blog.csdn.net/linuxshine/article/details/8849629)
这里是按照分区来复制的，更复杂，更容易出错

创建文件系统
root # mke2fs -j /dev/hdb1
root # mke2fs -j /dev/hdb2
root # mkswap /dev/hdb3

root # dd if=/dev/hda1 of=/dev/hdb1
root # dd if=/dev/hda2 of=/dev/hdb2

检查文件系统
root # e2ckfs -f /dev/hdb1
root # e2ckfs -f /dev/hdb2

磁盘容量调整
root # resize2fs /dev/hdb1
root # resize2fs /dev/hdb2

#### dd复制内存

同理，将内存中的数据整体备份，照样可以如法炮制：
`[root@roclinux ~]# dd if=/dev/mem of=/root/mem.img`

#### dd备份磁盘的 MBR

MBR，是 Master Boot Record，即硬盘的主引导记录，MBR 一旦损坏，分区表也就被破坏，数据大量丢失，系统就再也无法正常引导了，真是不堪设想啊！所以，对 MBR 的定期备份是十分必要的，在紧急关头，把它比喻成一颗救死扶伤的速效救心丸，也绝不为过。

一块磁盘的第一个扇区的 512 个字节所存储的正是这块磁盘的 MBR 信息，我们尝试用 dd 命令备份 MBR：
`[root@roclinux ~]# dd if=/dev/sda of=/root/sda_mbr.img count=1 bs=512`

如果未来遇到分区表损坏的情况，我们用曾经备份的 MBR 信息写回磁盘，就能起到立竿见影的效果。下面来一起看看如何将 MBR 写回硬盘：
`[root@roclinux ~]# dd if=/root/sda_mbr.img of=/dev/sda`

#### dd测试磁盘性能

向磁盘上写一个大文件, 来看写性能
`[root@roclinux ~]# dd if=/dev/zero bs=1024 count=1000000 of=/root/1Gb.file`
 
从磁盘上读取一个大文件, 来看读性能
`[root@roclinux ~]# dd if=/root/1Gb.file bs=64k | dd of=/dev/null`

`[root@roclinux ~]# time dd if=/dev/zero bs=1024 count=1000000 of=/root/1Gb.file`
`[root@roclinux ~]# time dd if=/dev/zero bs=2048 count=500000 of=/root/1Gb.file`
`[root@roclinux ~]# time dd if=/dev/zero bs=4096 count=250000 of=/root/1Gb.file`
`[root@roclinux ~]# time dd if=/dev/zero bs=8192 count=125000 of=/root/1Gb.file`

#### dd清除数据

dd if=/dev/urandom of=/dev/sda

### lsof
lsof -p $pid
找出pid打开的文件

lsof filename
找出使用filename的进程

假如由于误操作将/var/log/messages文件删除掉了，那么这时要将/var/log/messages文件恢复的方法如下：
首先使用lsof来查看当前是否有进程打开/var/logmessages文件，如下：

lsof |grep /var/log/messages
syslogd 1283 root 2w REG 3,3 5381017 1773647 /var/log/messages (deleted)
从上面的信息可以看到 PID 1283（syslogd）打开文件的文件描述符为 2。同时还可以看到/var/log/messages已经标记被删除了。因此我们可以在 /proc/1283/fd/2 （fd下的每个以数字命名的文件表示进程对应的文件描述符）中查看相应的信息，如下：

head -n 10 /proc/1283/fd/2

查看 /proc/8663/fd/2 就可以得到所要恢复的数据。如果可以通过文件描述符查看相应的数据，那么就可以使用 I/O 重定向将其复制到文件中，如:

cat /proc/1283/fd/2 > /var/log/messages
对于许多应用程序，尤其是日志文件和数据库，这种恢复删除文件的方法非常有用。

https://blog.csdn.net/demon7552003/article/details/117199334
https://www.iplaysoft.com/tools/linux-command/c/lsof.html
http://blog.itpub.net/31397003/viewspace-2147485

参数	作用
-a	列出打开文件存在的进程
-c <进程名>	列出指定进程所打开的文件
-p <进程号>	列出指定进程号所打开的文件
-g	列出GID号进程详情
-d <文件号>	列出占用该文件号的进程
+d <目录>	列出目录下被打开的文件
+D <目录>	递归列出目录下被打开的文件
-n <目录>	列出使用NFS的文件
-i <条件>	列出符合条件的进程。（4、6、协议、:端口、 @ip ）
-u	列出UID号进程详情
-h	显示帮助信息
-v	显示版本信息
————————————————
版权声明：本文为CSDN博主「lucky多多」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。
原文链接：https://blog.csdn.net/weixin_41948075/article/details/125558993

### ncat nc netcat

nc就是netcat，而ncat是nc的一个修改版本，功能更强大

-z 只是扫描，不连接，类似ping的效果
-l 监听
-p 端口
-k 只有ncat有，多个连接

```
ncat --recv-only IP PORT
nc -l -p -k [PORT] # 监听 持续多个连接
nc -v -z -w2 192.168.0.3 1-100  # 扫描端口 0交互 超时时间2s 默认TCP端口
```
更多功能：https://blog.csdn.net/xiao_yi_xiao/article/details/120458805


## 搜索文件和文件夹大小 

`sudo du -ahxd 1 .` 
一级一级查看哪个文件或目录占用最大

`du -ahx . | sort -rh | head -5` 
快速排查哪个文件或目录占用最大

`find . -xdev -type f -size +100M -print | xargs ls -lh | sort -k5,5 -h -r` 
在当前目录搜索并对文件大小排序，适用于找到对应目录后大文件很多情况，看不到目录内部大小

统计当前目录大小
`du -h --max-depth=1`
`du -h -d 1`

在当前目录中要搜索大小超过100MB的文件
`sudo find . -xdev -type f -size +100M`

传递find命令的输出到ls ，ls将打印已找到的每个文件的大小，然后将输出传递给sort命令，以根据文件大小对其进行排序
`find . -xdev -type f -size +100M -print | xargs ls -lh | sort -k5,5 -h -r`

打印占用最大磁盘空间的目录
`du -ahx . | sort -rh | head -5`会有重复情况
du -ahx .：估算当前目录（.）中的磁盘空间使用情况，包括文件和目录（a），以可读格式打印大小（h）并跳过不同文件系统上的目录（x）

加上x可以极大加速搜索时间

https://www.myfreax.com/find-large-files-in-linux/

## shell求值

`<()`是将命令的输出作为临时文件名传给其他命令，`$()`是将命令的输出作为字符串传给其他命令

比如bash就需要`<()`，bash -c需要`$()`

## 退出码

Linux 程序被外界中断时会发送中断信号，程序退出时的状态码为中断信号值加128

[ref](https://cloud.tencent.com/document/product/457/43125)

 打包Linux系统盘通常指的是创建一个系统镜像，这样你可以在其他计算机上恢复或部署相同的系统环境。这可以通过多种方法实现，以下是一些常见的方法：

## 导出硬盘文件aigc

1. **使用dd命令**：
   `dd` 是一个强大的Unix命令行工具，可以用来复制和转换文件。你可以使用它来创建一个完整的硬盘镜像。例如，如果你想要创建一个名为 `backup.img` 的镜像文件，你可以使用以下命令（请根据你的硬盘设备名称替换 `/dev/sda`）：

   ```bash
   sudo dd if=/dev/sda of=backup.img bs=4M status=progress
   ```

   这个命令会创建一个与 `/dev/sda` 硬盘大小相同的镜像文件。`bs=4M` 设置了块大小为4MB，这可以加快复制速度。`status=progress` 选项会在终端显示进度。

2. **使用Clonezilla**：
   Clonezilla 是一个流行的磁盘克隆和备份工具，它提供了一个用户友好的界面来创建和恢复系统镜像。你可以从官方网站下载Clonezilla Live CD/USB，然后从启动菜单中选择“device-image”来创建系统镜像。

3. **使用rsync**：
   `rsync` 是一个用于高效文件传输和同步的工具，也可以用来创建系统镜像。以下是一个基本的 `rsync` 命令示例，它会同步根目录到一个镜像文件：

   ```bash
   sudo rsync -aAXv --exclude={"/dev/*","/proc/*","/sys/*","/tmp/*","/run/*","/mnt/*","/media/*","/lost+found"} / /path/to/backup.img
   ```

   这个命令会创建一个名为 `backup.img` 的镜像文件，并且排除了一些不需要同步的系统目录。

4. **使用LVM快照**：
   如果你的系统使用了LVM（逻辑卷管理），你可以创建一个快照，然后导出为一个镜像文件。这通常需要一些LVM的知识，并且步骤可能会根据你的具体配置有所不同。

5. **使用专门的备份工具**：
   还有一些专门的备份工具，如Timeshift（主要用于Debian和Ubuntu系统），它们提供了更高级的备份和恢复功能。

在执行这些操作之前，请确保你有足够的权限，并且对系统进行了完整备份，以防万一出现问题。此外，创建系统镜像可能会占用大量磁盘空间，所以请确保你有足够的存储空间。如果你不熟悉这些命令或工具，建议在执行之前进行充分的研究或寻求专业人士的帮助。


## 新硬盘挂载流程

[ref](https://cloud.tencent.com/document/product/362/6734#Linux)

## tmux

[ref](https://www.ruanyifeng.com/blog/2019/10/tmux.html)

在 tmux 中使用鼠标滚轮滚动页面需要先开启 tmux 的鼠标模式。你可以在你的 tmux 配置文件（通常位于 ~/.tmux.conf）中加入以下一行：

set -g mode-mouse on

## session id 

查看进程各种id：

ps xao pid,ppid,pgid,sid,comm

介绍：

In Linux, every process has several IDs associated with it, including:

Process ID (PID)

This is an arbitrary number identifying the process. Every process has a unique ID, but after the process exits and the parent process has retrieved the exit status, the process ID is freed to be reused by a new process.

Parent Process ID (PPID)

This is just the PID of the process that started the process in question. If the parent process exits before the child does, the child's PPID is changed to another process (usually PID 1).

Process Group ID (PGID)

This is just the PID of the process group leader. If PID == PGID, then this process is a process group leader.

Session ID (SID)

This is just the PID of the session leader. If PID == SID, then this process is a session leader.

Sessions and process groups are just ways to treat a number of related processes as a unit. All the members of a process group always belong to the same session, but a session may have multiple process groups.

Normally, a shell will be a session leader, and every pipeline executed by that shell will be a process group. 

### setsid

[ref](https://stackoverflow.com/questions/45911705/why-use-os-setsid-in-python)

[ref2](https://blog.csdn.net/weixin_30336531/article/details/116609490)

setsid creates a new session id for the command you run using it, so that it does not depend on your shell session. If the shell session is closed the other command will continue to run. 

>session退出以后所有隶属于该session的进程组都会收到hup信号而挂起

shell创建的sid，而每次执行sudo就会创建新的pgid，不执行sudo执行python也会创建pgid，用sudo python的话sudo就相当于python的父进程，也是sudo的pgid

一个进程只要父进程退出，ppid就会变成1，不受ctrl+c影响，但是该shell退出会影响，setsid之后会有新的sid和pgid等于pid，不受影响

double fork的原因主要有:

如果只有一次fork，假如父进程并没有退出，而是需要继续运行其他代码，那么子进程daemon就不会被过继给init。在两次fork的情况下，第一个child process的作用只是用来产生daemon进程，fork完称以后可以直接exit，这样就能显式保证daemon在创建完成后就会被过继给init，而祖父进程仍然可以继续执行其他逻辑

如果只有一次fork，那么daemon因为setsid，所以会是session leader。这意味着daemon将有权限将这个新创建的session绑定到tty上。而两次fork产生的daemon，因为**第二个child process并没有setsid，所以并不是session leader**，从而杜绝了这种情况发生