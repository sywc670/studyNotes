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