## linux命令

### jq

[ref](https://www.iplaysoft.com/tools/linux-command/c/jq.html)

```shell
kubectl get pods -A -o json | jq -r '.items[] | select(.status.phase != "Running") | "kubectl delete pods \(.metadata.name) -n \(.metadata.namespace) "' | xargs -n 1 bash -c
```

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