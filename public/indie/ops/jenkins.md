
# jenkins

[ref](https://www.cnblogs.com/wangxu01/p/11077785.html)

The built-in documentation can be found globally at `${YOUR_JENKINS_URL}/pipeline-syntax`.

### jenkins agent

ssh模式，master主动连接agent

jnlp模式，agent主动连接master

### pipeline

分为声明式和脚本式，声明式：pipeline，脚本式：node

agent指令用于声明式，指明分配一个executor和work space

params env currentBuild 这几个变量是全局变量，有各自用途

单引号双引号区别和shell一样，单引号不会翻译出变量

### 重要的插件

Pipeline Groovy Libraries 让jenkinsfile可以以仓库形式存在，各个文件的组织形式像代码一样，方便复用
docker pipeline


#### kubernetes

node(POD_LABEL)的POD_LABEL是运行时会生成一个独特的值好像是creates a pod template with a generated unique label (available as POD_LABEL) 

Commands will be executed by default in the jnlp container, where the Jenkins agent is running. 

container("label")可以通过label来选择一个容器来进入执行命令，默认是jnlp容器里面


### blue ocean

是jenkins的一个UI，可以代替手工pipeline编写，变成可视化编写，也可以可视化流水线运行的过程

### jenkins更改国内插件源

https://mirrors.tuna.tsinghua.edu.cn/jenkins/updates/update-center.json

还要更改default.json文件才行