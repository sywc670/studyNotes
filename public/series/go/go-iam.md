- [go 安装配置](#go-安装配置)
- [MVC](#mvc)
- [go配置](#go配置)
- [iam 项目架构](#iam-项目架构)
- [iam项目部署流程](#iam项目部署流程)
  - [部署文档问题](#部署文档问题)
  - [生成证书](#生成证书)
- [开源协议](#开源协议)
- [代码规范 commit angular](#代码规范-commit-angular)
- [目录规范](#目录规范)
- [测试技术](#测试技术)
- [工具](#工具)
- [设计模式](#设计模式)
  - [策略模式](#策略模式)
  - [模板模式](#模板模式)
  - [工厂方式模式](#工厂方式模式)
  - [代理模式](#代理模式)
  - [选项模式](#选项模式)
- [API](#api)
  - [REST](#rest)
  - [RPC](#rpc)
- [OAuth](#oauth)
- [gorm](#gorm)
- [pflag](#pflag)
- [viper](#viper)
  - [优先级](#优先级)
  - [环境变量](#环境变量)
  - [读取配置](#读取配置)
- [cobra](#cobra)
- [设置必选标志](#设置必选标志)
  - [非选项参数验证](#非选项参数验证)
  - [hook](#hook)
- [iam项目阅读代码心得](#iam项目阅读代码心得)
  - [优雅关停](#优雅关停)
  - [sdk设计](#sdk设计)

## go 安装配置

```shell

mkdir -p $HOME/go
tar -xvzf /tmp/go1.18.3.linux-amd64.tar.gz -C $HOME/go
mv $HOME/go/go $HOME/go/go1.18.3

tee -a $HOME/.bashrc <<'EOF'
# Go envs
export GOVERSION=go1.18.3 # Go 版本设置
export GO_INSTALL_DIR=$HOME/go # Go 安装目录
export GOROOT=$GO_INSTALL_DIR/$GOVERSION # GOROOT 设置
export GOPATH=$WORKSPACE/golang # GOPATH 设置
export PATH=$GOROOT/bin:$GOPATH/bin:$PATH # 将 Go 语言自带的和通过 go install 安装的二进制文件加入到 PATH 路径中
export GO111MODULE="on" # 开启 Go moudles 特性
export GOPROXY=https://goproxy.cn,direct # 安装 Go 模块时，代理服务器设置
export GOPRIVATE=
export GOSUMDB=off # 关闭校验 Go 依赖包的哈希值
EOF
```

## MVC

MVC 架构的好处是通过`控制器层`将`视图层`和`模型层`分离之后，当更改视图层代码后时，我们就不需要重新编译控制器层和模型层的代码了。

通过 Controller 层将 Model 层和 View 层解耦，从而使代码更容易维护和扩展

## go配置

在使用模块的时候，`$GOPATH` 是无意义的，不过它还是会把下载的依赖储存在 `$GOPATH/pkg/mod` 目录中，也会把 go install 的二进制文件存放在 `$GOPATH/bin` 目录中。


## iam 项目架构

![](https://static001.geekbang.org/resource/image/0a/42/0a5f6fd67af1eda1c690c8216dc5e042.jpg?wh=3197*2063)
![](https://static001.geekbang.org/resource/image/6c/71/6cdbde36255c7fb2d4f2e718c9077a71.jpeg?wh=1920*1043)
![](https://static001.geekbang.org/resource/image/e6/f2/e68c21e1991c74becc4b8a6a8bf5a8f2.jpeg?wh=1818*496)

## iam项目部署流程

省略数据库安装等

### 部署文档问题

1. GOWORK用于.bashrc中，应换成GOSRC或者其他变量
2. go.work没有使用iam的go.mod 应当加上命令go work use iam

### 生成证书

我们可以使用 CloudFlare 的 PKI 工具集 `cfssl` 来创建所有的`证书`。安装 cfssl 工具集。我们可以直接安装 cfssl 已经编译好的二进制文件，cfssl 工具集中包含很多工具，这里我们需要安装 cfssl、cfssljson、cfssl-certinfo，功能如下。

cfssl：证书签发工具。
cfssljson：将 cfssl 生成的证书（json 格式）变为文件承载式证书。

```shell
cd $IAM_ROOT
./scripts/install/install.sh iam::install::install_cfssl

cd $IAM_ROOT
tee ca-config.json << EOF
{
  "signing": {
    "default": {
      "expiry": "87600h"
    },
    "profiles": {
      "iam": {
        "usages": [
          "signing",
          "key encipherment",
          "server auth",
          "client auth"
        ],
        "expiry": "876000h"
      }
    }
  }
}
EOF
```
上面的 JSON 配置中，有一些字段解释如下。
signing：表示该证书可用于签名其它证书（生成的 ca.pem 证书中 CA=TRUE）。
server auth：表示 client 可以用该证书对 server 提供的证书进行验证。
client auth：表示 server 可以用该证书对 client 提供的证书进行验证。
expiry：876000h，证书有效期设置为 100 年。

```shell

cd $IAM_ROOT
tee ca-csr.json << EOF
{
  "CN": "iam-ca",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "BeiJing",
      "L": "BeiJing",
      "O": "marmotedu",
      "OU": "iam"
    }
  ],
  "ca": {
    "expiry": "876000h"
  }
}
EOF
```

上面的 JSON 配置中，有一些字段解释如下。
C：Country，国家。ST：State，省份。L：Locality (L) or City，城市。CN：Common Name，iam-apiserver 从证书中提取该字段作为请求的用户名 (User Name) ，浏览器使用该字段验证网站是否合法。O：Organization，iam-apiserver 从证书中提取该字段作为请求用户所属的组 (Group)。OU：Company division (or Organization Unit – OU)，部门 / 单位。

除此之外，还有两点需要我们注意。不同证书 csr 文件的 CN、C、ST、L、O、OU 组合必须不同，否则可能出现 PEER'S CERTIFICATE HAS AN INVALID SIGNATURE 错误。后续创建证书的 csr 文件时，CN、OU 都不相同（C、ST、L、O 相同），以达到区分的目的。

```shell
cd $IAM_ROOT
source scripts/install/environment.sh
cfssl gencert -initca ca-csr.json | cfssljson -bare ca
ls ca*
# ca-config.json  ca.csr  ca-csr.json  ca-key.pem  ca.pem
sudo mv ca* ${IAM_CONFIG_DIR}/cert # 需要将证书文件拷贝到指定文件夹下（分发证书），方便各组件引用
```

上述命令会创建运行 CA 所必需的文件 `ca-key.pem`（私钥）和 `ca.pem`（证书），还会生成 `ca.csr`（证书签名请求），用于交叉签名或重新签名。

创建完之后，我们可以通过 `cfssl certinfo` 命名查看 cert 和 csr 信息：
```shell
cfssl certinfo -cert ${IAM_CONFIG_DIR}/cert/ca.pem # 查看 cert(证书信息)
cfssl certinfo -csr ${IAM_CONFIG_DIR}/cert/ca.csr # 查看 CSR(证书签名请求)信息
```

最后配置hosts

```shell
sudo tee -a /etc/hosts <<EOF
127.0.0.1 iam.api.marmotedu.com
127.0.0.1 iam.authz.marmotedu.com
EOF
```

之后的操作

api-server
```
cd $IAM_ROOT
source scripts/install/environment.sh
tee iam-apiserver-csr.json <<EOF
{
  "CN": "iam-apiserver",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "BeiJing",
      "L": "BeiJing",
      "O": "marmotedu",
      "OU": "iam-apiserver"
    }
  ],
  "hosts": [
    "127.0.0.1",
    "localhost",
    "iam.api.marmotedu.com"
  ]
}
EOF


cfssl gencert -ca=${IAM_CONFIG_DIR}/cert/ca.pem \
  -ca-key=${IAM_CONFIG_DIR}/cert/ca-key.pem \
  -config=${IAM_CONFIG_DIR}/cert/ca-config.json \
  -profile=iam iam-apiserver-csr.json | cfssljson -bare iam-apiserver
sudo mv iam-apiserver*pem ${IAM_CONFIG_DIR}/cert # 将生成的证书和私钥文件拷贝到配置文件目录

cd $IAM_ROOT
source scripts/install/environment.sh
make build BINS=iam-apiserver
sudo cp _output/platforms/linux/amd64/iam-apiserver ${IAM_INSTALL_DIR}/bin


./scripts/genconfig.sh scripts/install/environment.sh configs/iam-apiserver.yaml > iam-apiserver.yaml
sudo mv iam-apiserver.yaml ${IAM_CONFIG_DIR}


./scripts/genconfig.sh scripts/install/environment.sh init/iam-apiserver.service > iam-apiserver.service
sudo mv iam-apiserver.service /etc/systemd/system/


sudo systemctl daemon-reload
sudo systemctl enable iam-apiserver
sudo systemctl restart iam-apiserver
systemctl status iam-apiserver # 查看 iam-apiserver 运行状态，如果输出中包含 active (running)字样说明 iam-apiserver 成功启动


curl -s -XPOST -H'Content-Type: application/json' -d'{"username":"admin","password":"Admin@2021"}' http://127.0.0.1:8080/login | jq -r .token
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJhdWQiOiJpYW0uYXBpLm1hcm1vdGVkdS5jb20iLCJleHAiOjE2MTc5MjI4OTQsImlkZW50aXR5IjoiYWRtaW4iLCJpc3MiOiJpYW0tYXBpc2VydmVyIiwib3JpZ19pYXQiOjE2MTc4MzY0OTQsInN1YiI6ImFkbWluIn0.9qztVJseQ9XwqOFVUHNOtG96-KUovndz0SSr_QBsxAA


# 创建用户
$ curl -s -XPOST -H'Content-Type: application/json' -H'Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJhdWQiOiJpYW0uYXBpLm1hcm1vdGVkdS5jb20iLCJleHAiOjE2MTc5MjI4OTQsImlkZW50aXR5IjoiYWRtaW4iLCJpc3MiOiJpYW0tYXBpc2VydmVyIiwib3JpZ19pYXQiOjE2MTc4MzY0OTQsInN1YiI6ImFkbWluIn0.9qztVJseQ9XwqOFVUHNOtG96-KUovndz0SSr_QBsxAA' -d'{"password":"User@2021","metadata":{"name":"colin"},"nickname":"colin","email":"colin@foxmail.com","phone":"1812884xxxx"}' http://127.0.0.1:8080/v1/users

# 列出用户
$ curl -s -XGET -H'Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJhdWQiOiJpYW0uYXBpLm1hcm1vdGVkdS5jb20iLCJleHAiOjE2MTc5MjI4OTQsImlkZW50aXR5IjoiYWRtaW4iLCJpc3MiOiJpYW0tYXBpc2VydmVyIiwib3JpZ19pYXQiOjE2MTc4MzY0OTQsInN1YiI6ImFkbWluIn0.9qztVJseQ9XwqOFVUHNOtG96-KUovndz0SSr_QBsxAA' 'http://127.0.0.1:8080/v1/users?offset=0&limit=10'

# 获取 colin 用户的详细信息
$ curl -s -XGET -H'Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJhdWQiOiJpYW0uYXBpLm1hcm1vdGVkdS5jb20iLCJleHAiOjE2MTc5MjI4OTQsImlkZW50aXR5IjoiYWRtaW4iLCJpc3MiOiJpYW0tYXBpc2VydmVyIiwib3JpZ19pYXQiOjE2MTc4MzY0OTQsInN1YiI6ImFkbWluIn0.9qztVJseQ9XwqOFVUHNOtG96-KUovndz0SSr_QBsxAA' http://127.0.0.1:8080/v1/users/colin

# 修改 colin 用户
$ curl -s -XPUT -H'Content-Type: application/json' -H'Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJhdWQiOiJpYW0uYXBpLm1hcm1vdGVkdS5jb20iLCJleHAiOjE2MTc5MjI4OTQsImlkZW50aXR5IjoiYWRtaW4iLCJpc3MiOiJpYW0tYXBpc2VydmVyIiwib3JpZ19pYXQiOjE2MTc4MzY0OTQsInN1YiI6ImFkbWluIn0.9qztVJseQ9XwqOFVUHNOtG96-KUovndz0SSr_QBsxAA' -d'{"nickname":"colin","email":"colin_modified@foxmail.com","phone":"1812884xxxx"}' http://127.0.0.1:8080/v1/users/colin

# 删除 colin 用户
$ curl -s -XDELETE -H'Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJhdWQiOiJpYW0uYXBpLm1hcm1vdGVkdS5jb20iLCJleHAiOjE2MTc5MjI4OTQsImlkZW50aXR5IjoiYWRtaW4iLCJpc3MiOiJpYW0tYXBpc2VydmVyIiwib3JpZ19pYXQiOjE2MTc4MzY0OTQsInN1YiI6ImFkbWluIn0.9qztVJseQ9XwqOFVUHNOtG96-KUovndz0SSr_QBsxAA' http://127.0.0.1:8080/v1/users/colin

# 批量删除用户
$ curl -s -XDELETE -H'Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJhdWQiOiJpYW0uYXBpLm1hcm1vdGVkdS5jb20iLCJleHAiOjE2MTc5MjI4OTQsImlkZW50aXR5IjoiYWRtaW4iLCJpc3MiOiJpYW0tYXBpc2VydmVyIiwib3JpZ19pYXQiOjE2MTc4MzY0OTQsInN1YiI6ImFkbWluIn0.9qztVJseQ9XwqOFVUHNOtG96-KUovndz0SSr_QBsxAA' 'http://127.0.0.1:8080/v1/users?name=colin&name=mark&name=john'


# 创建 secret0 密钥
$ curl -s -XPOST -H'Content-Type: application/json' -H'Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJhdWQiOiJpYW0uYXBpLm1hcm1vdGVkdS5jb20iLCJleHAiOjE2MTc5MjI4OTQsImlkZW50aXR5IjoiYWRtaW4iLCJpc3MiOiJpYW0tYXBpc2VydmVyIiwib3JpZ19pYXQiOjE2MTc4MzY0OTQsInN1YiI6ImFkbWluIn0.9qztVJseQ9XwqOFVUHNOtG96-KUovndz0SSr_QBsxAA' -d'{"metadata":{"name":"secret0"},"expires":0,"description":"admin secret"}' http://127.0.0.1:8080/v1/secrets

# 列出所有密钥
$ curl -s -XGET -H'Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJhdWQiOiJpYW0uYXBpLm1hcm1vdGVkdS5jb20iLCJleHAiOjE2MTc5MjI4OTQsImlkZW50aXR5IjoiYWRtaW4iLCJpc3MiOiJpYW0tYXBpc2VydmVyIiwib3JpZ19pYXQiOjE2MTc4MzY0OTQsInN1YiI6ImFkbWluIn0.9qztVJseQ9XwqOFVUHNOtG96-KUovndz0SSr_QBsxAA' http://127.0.0.1:8080/v1/secrets

# 获取 secret0 密钥的详细信息
$ curl -s -XGET -H'Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJhdWQiOiJpYW0uYXBpLm1hcm1vdGVkdS5jb20iLCJleHAiOjE2MTc5MjI4OTQsImlkZW50aXR5IjoiYWRtaW4iLCJpc3MiOiJpYW0tYXBpc2VydmVyIiwib3JpZ19pYXQiOjE2MTc4MzY0OTQsInN1YiI6ImFkbWluIn0.9qztVJseQ9XwqOFVUHNOtG96-KUovndz0SSr_QBsxAA' http://127.0.0.1:8080/v1/secrets/secret0

# 修改 secret0 密钥
$ curl -s -XPUT -H'Content-Type: application/json' -H'Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJhdWQiOiJpYW0uYXBpLm1hcm1vdGVkdS5jb20iLCJleHAiOjE2MTc5MjI4OTQsImlkZW50aXR5IjoiYWRtaW4iLCJpc3MiOiJpYW0tYXBpc2VydmVyIiwib3JpZ19pYXQiOjE2MTc4MzY0OTQsInN1YiI6ImFkbWluIn0.9qztVJseQ9XwqOFVUHNOtG96-KUovndz0SSr_QBsxAA' -d'{"metadata":{"name":"secret0"},"expires":0,"description":"admin secret(modified)"}' http://127.0.0.1:8080/v1/secrets/secret0

# 删除 secret0 密钥
$ curl -s -XDELETE -H'Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJhdWQiOiJpYW0uYXBpLm1hcm1vdGVkdS5jb20iLCJleHAiOjE2MTc5MjI4OTQsImlkZW50aXR5IjoiYWRtaW4iLCJpc3MiOiJpYW0tYXBpc2VydmVyIiwib3JpZ19pYXQiOjE2MTc4MzY0OTQsInN1YiI6ImFkbWluIn0.9qztVJseQ9XwqOFVUHNOtG96-KUovndz0SSr_QBsxAA' http://127.0.0.1:8080/v1/secrets/secret0


# 创建策略
$ curl -s -XPOST -H'Content-Type: application/json' -H'Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJhdWQiOiJpYW0uYXBpLm1hcm1vdGVkdS5jb20iLCJleHAiOjE2MTc5MjI4OTQsImlkZW50aXR5IjoiYWRtaW4iLCJpc3MiOiJpYW0tYXBpc2VydmVyIiwib3JpZ19pYXQiOjE2MTc4MzY0OTQsInN1YiI6ImFkbWluIn0.9qztVJseQ9XwqOFVUHNOtG96-KUovndz0SSr_QBsxAA' -d'{"metadata":{"name":"policy0"},"policy":{"description":"One policy to rule them all.","subjects":["users:<peter|ken>","users:maria","groups:admins"],"actions":["delete","<create|update>"],"effect":"allow","resources":["resources:articles:<.*>","resources:printer"],"conditions":{"remoteIPAddress":{"type":"CIDRCondition","options":{"cidr":"192.168.0.1/16"}}}}}' http://127.0.0.1:8080/v1/policies

# 列出所有策略
$ curl -s -XGET -H'Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJhdWQiOiJpYW0uYXBpLm1hcm1vdGVkdS5jb20iLCJleHAiOjE2MTc5MjI4OTQsImlkZW50aXR5IjoiYWRtaW4iLCJpc3MiOiJpYW0tYXBpc2VydmVyIiwib3JpZ19pYXQiOjE2MTc4MzY0OTQsInN1YiI6ImFkbWluIn0.9qztVJseQ9XwqOFVUHNOtG96-KUovndz0SSr_QBsxAA' http://127.0.0.1:8080/v1/policies

# 获取 policy0 策略的详细信息
$ curl -s -XGET -H'Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJhdWQiOiJpYW0uYXBpLm1hcm1vdGVkdS5jb20iLCJleHAiOjE2MTc5MjI4OTQsImlkZW50aXR5IjoiYWRtaW4iLCJpc3MiOiJpYW0tYXBpc2VydmVyIiwib3JpZ19pYXQiOjE2MTc4MzY0OTQsInN1YiI6ImFkbWluIn0.9qztVJseQ9XwqOFVUHNOtG96-KUovndz0SSr_QBsxAA' http://127.0.0.1:8080/v1/policies/policy0

# 修改 policy0 策略
$ curl -s -XPUT -H'Content-Type: application/json' -H'Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJhdWQiOiJpYW0uYXBpLm1hcm1vdGVkdS5jb20iLCJleHAiOjE2MTc5MjI4OTQsImlkZW50aXR5IjoiYWRtaW4iLCJpc3MiOiJpYW0tYXBpc2VydmVyIiwib3JpZ19pYXQiOjE2MTc4MzY0OTQsInN1YiI6ImFkbWluIn0.9qztVJseQ9XwqOFVUHNOtG96-KUovndz0SSr_QBsxAA' -d'{"metadata":{"name":"policy0"},"policy":{"description":"One policy to rule them all(modified).","subjects":["users:<peter|ken>","users:maria","groups:admins"],"actions":["delete","<create|update>"],"effect":"allow","resources":["resources:articles:<.*>","resources:printer"],"conditions":{"remoteIPAddress":{"type":"CIDRCondition","options":{"cidr":"192.168.0.1/16"}}}}}' http://127.0.0.1:8080/v1/policies/policy0

# 删除 policy0 策略
$ curl -s -XDELETE -H'Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJhdWQiOiJpYW0uYXBpLm1hcm1vdGVkdS5jb20iLCJleHAiOjE2MTc5MjI4OTQsImlkZW50aXR5IjoiYWRtaW4iLCJpc3MiOiJpYW0tYXBpc2VydmVyIiwib3JpZ19pYXQiOjE2MTc4MzY0OTQsInN1YiI6ImFkbWluIn0.9qztVJseQ9XwqOFVUHNOtG96-KUovndz0SSr_QBsxAA' http://127.0.0.1:8080/v1/policies/policy0

```

iamctl

```

cd $IAM_ROOT
source scripts/install/environment.sh
cat > admin-csr.json <<EOF
{
  "CN": "admin",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "BeiJing",
      "L": "BeiJing",
      "O": "marmotedu",
      "OU": "iamctl"
    }
  ],
  "hosts": []
}
EOF


cfssl gencert -ca=${IAM_CONFIG_DIR}/cert/ca.pem \
  -ca-key=${IAM_CONFIG_DIR}/cert/ca-key.pem \
  -config=${IAM_CONFIG_DIR}/cert/ca-config.json \
  -profile=iam admin-csr.json | cfssljson -bare admin
mkdir -p $(dirname ${CONFIG_USER_CLIENT_CERTIFICATE}) $(dirname ${CONFIG_USER_CLIENT_KEY}) # 创建客户端证书存放的目录
mv admin.pem ${CONFIG_USER_CLIENT_CERTIFICATE} # 安装 TLS 的客户端证书
mv admin-key.pem ${CONFIG_USER_CLIENT_KEY} # 安装 TLS 的客户端私钥文件


cd $IAM_ROOT
source scripts/install/environment.sh
make build BINS=iamctl
cp _output/platforms/linux/amd64/iamctl $HOME/bin



./scripts/genconfig.sh scripts/install/environment.sh configs/iamctl.yaml> iamctl.yaml
mkdir -p $HOME/.iam
mv iamctl.yaml $HOME/.iam

iamctl user list

```

iam-authz-server

```

cd $IAM_ROOT
source scripts/install/environment.sh
tee iam-authz-server-csr.json <<EOF
{
  "CN": "iam-authz-server",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "BeiJing",
      "L": "BeiJing",
      "O": "marmotedu",
      "OU": "iam-authz-server"
    }
  ],
  "hosts": [
    "127.0.0.1",
    "localhost",
    "iam.authz.marmotedu.com"
  ]
}
EOF


cfssl gencert -ca=${IAM_CONFIG_DIR}/cert/ca.pem \
  -ca-key=${IAM_CONFIG_DIR}/cert/ca-key.pem \
  -config=${IAM_CONFIG_DIR}/cert/ca-config.json \
  -profile=iam iam-authz-server-csr.json | cfssljson -bare iam-authz-server
sudo mv iam-authz-server*pem ${IAM_CONFIG_DIR}/cert # 将生成的证书和私钥文件拷贝到配置文件目录


cd $IAM_ROOT
source scripts/install/environment.sh
make build BINS=iam-authz-server
sudo cp _output/platforms/linux/amd64/iam-authz-server ${IAM_INSTALL_DIR}/bin


./scripts/genconfig.sh scripts/install/environment.sh configs/iam-authz-server.yaml > iam-authz-server.yaml
sudo mv iam-authz-server.yaml ${IAM_CONFIG_DIR}


./scripts/genconfig.sh scripts/install/environment.sh init/iam-authz-server.service > iam-authz-server.service
sudo mv iam-authz-server.service /etc/systemd/system/


sudo systemctl daemon-reload
sudo systemctl enable iam-authz-server
sudo systemctl restart iam-authz-server
systemctl status iam-authz-server # 查看 iam-authz-server 运行状态，如果输出中包含 active (running)字样说明 iam-authz-server 成功启动。


token=`curl -s -XPOST -H'Content-Type: application/json' -d'{"username":"admin","password":"Admin@2021"}' http://127.0.0.1:8080/login | jq -r .token`

curl -s -XPOST -H"Content-Type: application/json" -H"Authorization: Bearer $token" -d'{"metadata":{"name":"authztest"},"policy":{"description":"One policy to rule them all.","subjects":["users:<peter|ken>","users:maria","groups:admins"],"actions":["delete","<create|update>"],"effect":"allow","resources":["resources:articles:<.*>","resources:printer"],"conditions":{"remoteIPAddress":{"type":"CIDRCondition","options":{"cidr":"192.168.0.1/16"}}}}}' http://127.0.0.1:8080/v1/policies


curl -s -XPOST -H"Content-Type: application/json" -H"Authorization: Bearer $token" -d'{"metadata":{"name":"authztest"},"expires":0,"description":"admin secret"}' http://127.0.0.1:8080/v1/secrets
{"metadata":{"id":23,"name":"authztest","createdAt":"2021-04-08T07:24:50.071671422+08:00","updatedAt":"2021-04-08T07:24:50.071671422+08:00"},"username":"admin","secretID":"ZuxvXNfG08BdEMqkTaP41L2DLArlE6Jpqoox","secretKey":"7Sfa5EfAPIwcTLGCfSvqLf0zZGCjF3l8","expires":0,"description":"admin secret"}


curl -s -XPOST -H'Content-Type: application/json' -H'Authorization: Bearer eyJhbGciOiJIUzI1NiIsImtpZCI6Ilp1eHZYTmZHMDhCZEVNcWtUYVA0MUwyRExBcmxFNkpwcW9veCIsInR5cCI6IkpXVCJ9.eyJhdWQiOiJpYW0uYXV0aHoubWFybW90ZWR1LmNvbSIsImV4cCI6MTYxNzg0NTE5NSwiaWF0IjoxNjE3ODM3OTk1LCJpc3MiOiJpYW1jdGwiLCJuYmYiOjE2MTc4Mzc5OTV9.za9yLM7lHVabPAlVQLCqXEaf8sTU6sodAsMXnmpXjMQ' -d'{"subject":"users:maria","action":"delete","resource":"resources:articles:ladon-introduction","context":{"remoteIPAddress":"192.168.0.5"}}' http://127.0.0.1:9090/v1/authz
{"allowed":true}


```

iam-pump

```

cd $IAM_ROOT
source scripts/install/environment.sh
make build BINS=iam-pump
sudo cp _output/platforms/linux/amd64/iam-pump ${IAM_INSTALL_DIR}/bin


./scripts/genconfig.sh scripts/install/environment.sh configs/iam-pump.yaml > iam-pump.yaml
sudo mv iam-pump.yaml ${IAM_CONFIG_DIR}


./scripts/genconfig.sh scripts/install/environment.sh init/iam-pump.service > iam-pump.service
sudo mv iam-pump.service /etc/systemd/system/


sudo systemctl daemon-reload
sudo systemctl enable iam-pump
sudo systemctl restart iam-pump
systemctl status iam-pump # 查看 iam-pump 运行状态，如果输出中包含 active (running)字样说明 iam-pump 成功启动。

curl http://127.0.0.1:7070/healthz
```

man

```

cd $IAM_ROOT
./scripts/update-generated-docs.sh

sudo cp docs/man/man1/* /usr/share/man/man1/

man iam-apiserver
```

iamctl version -o yaml


## 开源协议

![](../../reference/pic/open.webp)

## 代码规范 commit angular

git rebase 的最大作用是它可以**重写历史**。我们通常会通过 `git rebase -i <commit ID>`使用 git rebase 命令，-i 参数表示交互（interactive），该命令会进入到一个交互界面中，其实就是 Vim 编辑器。在该界面中，我们可以对里面的 commit 做一些操作

`git rebase -i <commid ID>`这里的一定要是需要合并 commit 中最旧 commit 的父 commit ID。

这个交互界面会首先列出给定`<commit ID>`之前（不包括，越下面越新）的所有 commit，每个 commit 前面有一个操作命令，默认是 pick。我们可以选择不同的 commit，并修改 commit 前面的命令，来对该 commit 执行不同的变更操作。

git commit --amend：修改最近一次 commit 的 message；
git rebase -i：修改某次 commit 的 message。

![git rebase选项](https://static001.geekbang.org/resource/image/5f/f2/5f5a79a5d2bde029d4de9d98026ef3f2.png?wh=629*393)

今天我向你介绍了 `Commit Message` 规范，主要讲了业界使用最多的 `Angular` 规范。Angular 规范中，Commit Message 包含三个部分：Header、Body 和 Footer。Header 对 commit 做了`高度概括`，Body 部分是对本次 commit 的更`详细描述`，Footer 部分主要用来说明本次 commit 导致的`后果`。格式如下：

```
<type>[optional scope]: <description>
// 空行
[optional body]
// 空行
[optional footer(s)]
```

![](https://static001.geekbang.org/resource/image/89/27/89c618a7415c0c38b09d86d7f882a427.png?wh=726*511)

## 目录规范

[目录规范](https://github.com/golang-standards/project-layout/blob/master/README_zh.md)

`/internal`存放私有应用和库代码。如果一些代码，你不希望在其他应用和库中被导入，可以将这部分代码放在/internal 目录下。在引入其它项目 internal 下的包时，Go 语言会在编译时报错

## 测试技术

当我们的代码可测之后，就可以借助一些工具来 `Mock` 需要的接口了。常用的 Mock 工具，有这么几个：

golang/mock，是官方提供的 Mock 框架。它实现了基于 interface 的 Mock 功能，能够与 Golang 内置的 testing 包做很好的集成，是最常用的 Mock 工具。golang/mock 提供了 mockgen 工具用来生成 interface 对应的 Mock 源文件。

sqlmock，可以用来模拟数据库连接。数据库是项目中比较常见的依赖，在遇到数据库依赖时都可以用它。

httpmock，可以用来 Mock HTTP 请求。

bouk/monkey，猴子补丁，能够通过替换函数指针的方式来修改任意函数的实现。如果 golang/mock、sqlmock 和 httpmock 这几种方法都不能满足我们的需求，我们可以尝试通过猴子补丁的方式来 Mock 依赖。可以这么说，猴子补丁提供了单元测试 Mock 依赖的最终解决方案。

当我们编写了可测试的代码之后，接下来就需要编写足够的`测试用例`，用来提高项目的单元测试覆盖率。

这里我有以下两个建议供你参考：

使用 `gotests` 工具自动生成单元测试代码，减少编写单元测试用例的工作量，将你从重复的劳动中解放出来。

定期检查单元测试覆盖率。你可以通过以下方法来检查：

```
go test -race -cover  -coverprofile=./coverage.out -timeout=10m -short -v ./...
go tool cover -func ./coverage.out
```

## 工具

```
$ cd $IAM_ROOT
$ make tools.install
```

## 设计模式

### 策略模式

```go

package strategy

// 策略模式

// 定义一个策略类
type IStrategy interface {
  do(int, int) int
}

// 策略实现：加
type add struct{}

func (*add) do(a, b int) int {
  return a + b
}

// 策略实现：减
type reduce struct{}

func (*reduce) do(a, b int) int {
  return a - b
}

// 具体策略的执行者
type Operator struct {
  strategy IStrategy
}

// 设置策略
func (operator *Operator) setStrategy(strategy IStrategy) {
  operator.strategy = strategy
}

// 调用策略中的方法
func (operator *Operator) calculate(a, b int) int {
  return operator.strategy.do(a, b)
}
```

### 模板模式

```go

package template

import "fmt"

type Cooker interface {
  fire()
  cooke()
  outfire()
}

// 类似于一个抽象类
type CookMenu struct {
}

func (CookMenu) fire() {
  fmt.Println("开火")
}

// 做菜，交给具体的子类实现
func (CookMenu) cooke() {
}

func (CookMenu) outfire() {
  fmt.Println("关火")
}

// 封装具体步骤
func doCook(cook Cooker) {
  cook.fire()
  cook.cooke()
  cook.outfire()
}

type XiHongShi struct {
  CookMenu
}

func (*XiHongShi) cooke() {
  fmt.Println("做西红柿")
}

type ChaoJiDan struct {
  CookMenu
}

func (ChaoJiDan) cooke() {
  fmt.Println("做炒鸡蛋")
}
```

### 工厂方式模式

```go

type Person struct {
  name string
  age int
}

func NewPersonFactory(age int) func(name string) Person {
  return func(name string) Person {
    return Person{
      name: name,
      age: age,
    }
  }
}

newBaby := NewPersonFactory(1)
baby := newBaby("john")

newTeenager := NewPersonFactory(16)
teen := newTeenager("jill")
```

### 代理模式

```go

package proxy

import "fmt"

type Seller interface {
  sell(name string)
}

// 火车站
type Station struct {
  stock int //库存
}

func (station *Station) sell(name string) {
  if station.stock > 0 {
    station.stock--
    fmt.Printf("代理点中：%s买了一张票,剩余：%d \n", name, station.stock)
  } else {
    fmt.Println("票已售空")
  }

}

// 火车代理点
type StationProxy struct {
  station *Station // 持有一个火车站对象
}

func (proxy *StationProxy) sell(name string) {
  if proxy.station.stock > 0 {
    proxy.station.stock--
    fmt.Printf("代理点中：%s买了一张票,剩余：%d \n", name, proxy.station.stock)
  } else {
    fmt.Println("票已售空")
  }
}
```

### 选项模式

```go

package options

import (
  "time"
)

type Connection struct {
  addr    string
  cache   bool
  timeout time.Duration
}

const (
  defaultTimeout = 10
  defaultCaching = false
)

type options struct {
  timeout time.Duration
  caching bool
}

// Option overrides behavior of Connect.
type Option interface {
  apply(*options)
}

type optionFunc func(*options)

func (f optionFunc) apply(o *options) {
  f(o)
}

func WithTimeout(t time.Duration) Option {
  return optionFunc(func(o *options) {
    o.timeout = t
  })
}

func WithCaching(cache bool) Option {
  return optionFunc(func(o *options) {
    o.caching = cache
  })
}

// Connect creates a connection.
func NewConnect(addr string, opts ...Option) (*Connection, error) {
  options := options{
    timeout: defaultTimeout,
    caching: defaultCaching,
  }

  for _, o := range opts {
    o.apply(&options)
  }

  return &Connection{
    addr:    addr,
    cache:   options.caching,
    timeout: options.timeout,
  }, nil
}
```

## API

### REST

这里强调下 `REST` 和 `RESTful` API 的区别：REST 是一种规范，而 RESTful API 则是满足这种规范的 API 接口。

资源名使用名词而不是动词，并且用名词复数表示。
资源分为 Collection 和 Member 两种。
Collection：一堆资源的集合。例如我们系统里有很多用户（User）, 这些用户的集合就是 Collection。Collection 的 URI 标识应该是 域名/资源名复数, 例如https:// iam.api.marmotedu.com/users。
Member：单个特定资源。例如系统中特定名字的用户，就是 Collection 里的一个 Member。Member 的 URI 标识应该是 域名/资源名复数/资源名称, 例如https:// iam.api.marmotedu/users/admin。
URI 结尾不应包含/。URI 中不能出现下划线 _，必须用中杠线 -代替（有些人推荐用 _，有些人推荐用 -，统一使用一种格式即可，我比较推荐用 -）。
URI 路径用小写，不要用大写。避免层级过深的 URI。超过 2 层的资源嵌套会很乱，建议将其他资源转化为?参数，比如：
/schools/tsinghua/classes/rooma/students/zhang # 不推荐
/students?school=qinghua&class=rooma # 推荐

在实际的 API 开发中，可能你会发现`有些操作不能很好地映射为一个 REST 资源`，这时候，你可以参考下面的做法。将一个操作变成资源的一个属性，比如想在系统中暂时禁用某个用户，可以这么设计 URI：/users/zhangsan?active=false。

将操作当作是一个资源的嵌套资源，比如一个 GitHub 的加星操作：
PUT /gists/:id/star # github star action
DELETE /gists/:id/star # github unstar action

如果以上都不能解决问题，有时可以打破这类规范。比如登录操作，登录不属于任何一个资源，URI 可以设计为：/login。

在设计 API 时，经常会有`批量删除`的需求，需要在请求中携带多个需要删除的资源名，但是 HTTP 的 DELETE 方法不能携带多个资源名，这时候可以通过下面三种方式来解决：发起多个 DELETE 请求。操作路径中带多个 id，id 之间用分隔符分隔, 例如：DELETE /users?ids=1,2,3  。直接使用 POST 方式来批量删除，body 中传入需要删除的资源列表。其中，第二种是我最推荐的方式，因为使用了匹配的 DELETE 动词，并且不需要发送多次 DELETE 请求。你需要注意的是，这三种方式都有各自的使用场景，你可以根据需要自行选择。如果选择了某一种方式，那么整个项目都需要统一用这种方式。

随着时间的推移、需求的变更，一个 API 往往满足不了现有的需求，这时候就需要对 API 进行修改。对 API 进行修改时，不能影响其他调用系统的正常使用，这就要求 API 变更做到向下兼容，也就是新老版本共存。但在实际场景中，很可能会出现同一个 API 无法向下兼容的情况。这时候最好的解决办法是从一开始就引入` API 版本机制`，当不能向下兼容时，就引入一个新的版本，老的版本则保留原样。这样既能保证服务的可用性和安全性，同时也能满足新需求。API 版本有不同的标识方法，在 RESTful API 开发中，通常将版本标识放在如下 3 个`位置`：

URL 中，比如/v1/users。
HTTP Header 中，比如Accept: vnd.example-com.foo+json; version=1.0。
Form 参数中，比如/users?version=v1。

我们这门课中的版本标识是放在 URL 中的，比如/v1/users，这样做的好处是很直观，GitHub、Kubernetes、Etcd 等很多优秀的 API 均采用这种方式。

API 通常的`命名方式`有三种，分别是驼峰命名法 (serverAddress)、蛇形命名法 (server_address) 和脊柱命名法 (server-address)。

REST 资源的查询接口，通常情况下都需要实现`分页、过滤、排序、搜索功能`，因为这些功能是每个 REST 资源都能用到的，所以可以实现为一个公共的 API 组件。下面来介绍下这些功能。
分页：在列出一个 Collection 下所有的 Member 时，应该提供分页功能，例如/users?offset=0&limit=20（limit，指定返回记录的数量；offset，指定返回记录的开始位置）。引入分页功能可以减少 API 响应的延时，同时可以避免返回太多条目，导致服务器 / 客户端响应特别慢，甚至导致服务器 / 客户端 crash 的情况。
过滤：如果用户不需要一个资源的全部状态属性，可以在 URI 参数里指定返回哪些属性，例如/users?fields=email,username,address。
排序：用户很多时候会根据创建时间或者其他因素，列出一个 Collection 中前 100 个 Member，这时可以在 URI 参数中指明排序参数，例如/users?sort=age,desc。
搜索：当一个资源的 Member 太多时，用户可能想通过搜索，快速找到所需要的 Member，或着想搜下有没有名字为 xxx 的某类资源，这时候就需要提供搜索功能。搜索建议按模糊匹配来搜索。

API 的`域名设置`主要有两种方式：
https://marmotedu.com/api，这种方式适合 API 将来不会有进一步扩展的情况，比如刚开始 marmotedu.com 域名下只有一套 API 系统，未来也只有这一套 API 系统。
https://iam.api.marmotedu.com，如果 marmotedu.com 域名下未来会新增另一个系统 API，这时候最好的方式是每个系统的 API 拥有专有的 API 域名，比如：storage.api.marmotedu.com，network.api.marmotedu.com。腾讯云的域名就是采用这种方式。

### RPC

在 Go 项目开发中，如果业务对`性能要求`比较高，并且需要提供给多种编程语言调用，这时候就可以考虑使用 RPC API 接口。RPC 在 Go 项目开发中用得也非常多，需要我们认真掌握。

RPC（Remote Procedure Call），即远程过程调用，是一个计算机通信协议。该协议允许运行于一台计算机的程序调用另一台计算机的子程序，而程序员不用额外地为这个交互作用编程。

`通俗来讲`，就是服务端实现了一个函数，客户端使用 RPC 框架提供的接口，像调用本地函数一样调用这个函数，并获取返回值。RPC 屏蔽了底层的网络通信细节，使得开发人员无需关注网络编程的细节，可以将更多的时间和精力放在业务逻辑本身的实现上，从而提高开发效率。

![](../../reference/pic/rpc.webp)

RPC 调用具体流程如下：
Client 通过本地调用，调用 Client Stub。
Client Stub 将参数打包（也叫 Marshalling）成一个消息，然后发送这个消息。
Client 所在的 OS 将消息发送给 Server。
Server 端接收到消息后，将消息传递给 Server Stub。
Server Stub 将消息解包（也叫 Unmarshalling）得到参数。
Server Stub 调用服务端的子程序（函数），处理完后，将最终结果按照相反的步骤返回给 Client。

这里需要注意，`Stub` 负责调用参数和返回值的流化（serialization）、参数的打包和解包，以及网络层的通信。Client 端一般叫 Stub，Server 端一般叫 Skeleton。

在 gRPC 的框架中，`Protocol Buffers` 主要有三个作用。

第一，可以用来`定义数据结构`。举个例子，下面的代码定义了一个 SecretInfo 数据结构：
```

// SecretInfo contains secret details.
message SecretInfo {
    string name = 1;
    string secret_id  = 2;
    string username   = 3;
    string secret_key = 4;
    int64 expires = 5;
    string description = 6;
    string created_at = 7;
    string updated_at = 8;
}
```
第二，可以用来`定义服务接口`。下面的代码定义了一个 Cache 服务，服务包含了 ListSecrets 和 ListPolicies 两个 API 接口。
```

// Cache implements a cache rpc service.
service Cache{
  rpc ListSecrets(ListSecretsRequest) returns (ListSecretsResponse) {}
  rpc ListPolicies(ListPoliciesRequest) returns (ListPoliciesResponse) {}
}
```

第三，可以通过 protobuf `序列化和反序列化`，提升传输效率。

假定希望用RPC作为内部API的通讯，同时也想对外提供RESTful API，又不想写两套，可以使用`gRPC Gateway` 插件，在生成RPC的同时也生成RESTful web  server。

```protobuf

syntax = "proto3";

option go_package = "github.com/marmotedu/gopractise-demo/apistyle/greeter/helloworld";

package helloworld;

// The greeting service definition.
service Greeter {
  // Sends a greeting
  rpc SayHello (HelloRequest) returns (HelloReply) {}
}

// The request message containing the user's name.
message HelloRequest {
  string name = 1;
}

// The response message containing the greetings
message HelloReply {
  string message = 1;
}
```

在 helloworld.proto 定义文件中，`option` 关键字用来对.proto 文件进行一些设置，其中 `go_package` 是必需的设置，而且 go_package 的值必须是包导入的路径。package 关键字指定生成的.pb.go 文件所在的包名。我们通过 `service` 关键字定义服务，然后再指定该服务拥有的 RPC 方法，并定义方法的请求和返回的结构体类型：

gRPC 支持定义 4 种类型的`服务方法`，分别是简单模式、服务端数据流模式、客户端数据流模式和双向数据流模式。

`简单模式`（Simple RPC）：是最简单的 gRPC 模式。客户端发起一次请求，服务端响应一个数据。定义格式为 rpc SayHello (HelloRequest) returns (HelloReply) {}。

`服务端数据流模式`（Server-side streaming RPC）：客户端发送一个请求，服务器返回数据流响应，客户端从流中读取数据直到为空。定义格式为 rpc SayHello (HelloRequest) returns (stream HelloReply) {}。

`客户端数据流模式`（Client-side streaming RPC）：客户端将消息以流的方式发送给服务器，服务器全部处理完成之后返回一次响应。定义格式为 rpc SayHello (stream HelloRequest) returns (HelloReply) {}。

`双向数据流模式`（Bidirectional streaming RPC）：客户端和服务端都可以向对方发送数据流，这个时候双方的数据可以同时互相发送，也就是可以实现实时交互 RPC 框架原理。定义格式为 rpc SayHello (stream HelloRequest) returns (stream HelloReply) {}。

接下来，我们需要根据.proto 服务定义生成 gRPC 客户端和服务器接口。我们可以使用 `protoc` 编译工具，并指定使用其 Go 语言插件来生成
```shell
protoc -I. --go_out=plugins=grpc:$GOPATH/src helloworld.proto
ls
helloworld.pb.go  helloworld.proto
```
-I意思是include，是proto文件目录，可以选择多个目录

--go-grpc_out= 与--go_out=plugins=grpc:**区别**

在使用 Protocol Buffers（protobuf）和 gRPC 生成代码时，有两个相关的选项：`--go-grpc_out` 和 `--go_out=plugins=grpc`。这两个选项用于生成 gRPC 相关的代码，但有一些细微的区别。

1. `--go_out=plugins=grpc`：这是 gRPC 官方提供的选项，用于生成 gRPC 服务端和客户端的代码。它指示编译器生成与 gRPC 相关的代码，包括服务端接口的实现和客户端的调用代码。使用此选项后，编译器将根据 `.proto` 文件生成对应的 gRPC 接口和消息类型的 Go 代码。

示例：
```
protoc --go_out=plugins=grpc:. example.proto
```

2. `--go-grpc_out`：这是由 gRPC Go 生态系统提供的选项，用于生成 gRPC 相关的代码，包括服务端和客户端的代码。它与 `--go_out=plugins=grpc` 的功能基本相同，但由于使用了单独的插件 `protoc-gen-go-grpc`，因此在使用该选项时需要确保该插件已正确安装。

示例：
```
protoc --go-grpc_out=. example.proto
```

综上所述，两个选项都可以用于生成 gRPC 相关的代码，但 `--go-grpc_out` 是一个第三方插件，而 `--go_out=plugins=grpc` 是官方提供的选项。在选择使用哪个选项时，可以根据个人偏好和项目需求进行决定。


## OAuth

OAuth（开放授权）是一个开放的授权标准，允许用户让第三方应用访问该用户在某一 Web 服务上存储的私密资源（例如照片、视频、音频等），而无需将用户名和密码提供给第三方应用。OAuth 目前的版本是 2.0 版。OAuth2.0 一共分为四种授权方式，分别为密码式、隐藏式、凭借式和授权码模式。

接下来，我们就具体介绍下每一种授权方式。

第一种，`密码式`。密码式的授权方式，就是用户把用户名和密码直接告诉给第三方应用，然后第三方应用使用用户名和密码换取令牌。所以，使用此授权方式的前提是无法采用其他授权方式，并且用户高度信任某应用。

认证流程如下：网站 A 向用户发出获取用户名和密码的请求；用户同意后，网站 A 凭借用户名和密码向网站 B 换取令牌；网站 B 验证用户身份后，给出网站 A 令牌，网站 A 凭借令牌可以访问网站 B 对应权限的资源。

第二种，`隐藏式`。这种方式适用于前端应用。认证流程如下：A 网站提供一个跳转到 B 网站的链接，用户点击后跳转至 B 网站，并向用户请求授权；用户登录 B 网站，同意授权后，跳转回 A 网站指定的重定向 redirect_url 地址，并携带 B 网站返回的令牌，用户在 B 网站的数据给 A 网站使用。这个授权方式存在着“中间人攻击”的风险，因此只能用于一些安全性要求不高的场景，并且令牌的有效时间要非常短。

第三种，`凭借式`。这种方式是在命令行中请求授权，适用于没有前端的命令行应用。认证流程如下：应用 A 在命令行向应用 B 请求授权，此时应用 A 需要携带应用 B 提前颁发的 secretID 和 secretKey，其中 secretKey 出于安全性考虑，需在后端发送；应用 B 接收到 secretID 和 secretKey，并进行身份验证，验证通过后返回给应用 A 令牌。

第四种，`授权码模式`。这种方式就是第三方应用先提前申请一个授权码，然后再使用授权码来获取令牌。相对来说，这种方式安全性更高，前端传送授权码，后端存储令牌，与资源的通信都是在后端，可以避免令牌的泄露导致的安全问题。认证流程如下：A 网站提供一个跳转到 B 网站的链接 +redirect_url，用户点击后跳转至 B 网站；用户携带向 B 网站提前申请的 client_id，向 B 网站发起身份验证请求；用户登录 B 网站，通过验证，授予 A 网站权限，此时网站跳转回 redirect_url，其中会有 B 网站通过验证后的授权码附在该 url 后；网站 A 携带授权码向网站 B 请求令牌，网站 B 验证授权码后，返回令牌即 access_token。

## gorm

GORM 支持连接池，底层是用 database/sql 包来维护连接池的，连接池设置如下：
```
sqlDB, err := db.DB()
sqlDB.SetMaxIdleConns(100)              // 设置MySQL的最大空闲连接数（推荐100）
sqlDB.SetMaxOpenConns(100)             // 设置MySQL的最大连接数（推荐100）
sqlDB.SetConnMaxLifetime(time.Hour)    // 设置MySQL的空闲连接最大存活时间（推荐10s）
```

## pflag

使用Get获取参数的值。可以使用Get来获取标志的值，代表 Pflag 所支持的类型。例如：有一个 pflag.FlagSet，带有一个名为 flagname 的 int 类型的标志，可以使用`GetInt()`来获取 int 值。需要注意 flagname 必须存在且必须是 int，例如：
`i, err := flagset.GetInt("flagname")`

在定义完标志之后，可以调用`pflag.Parse()`来解析定义的标志。
解析后，可通过`pflag.Args()`返回所有的非选项参数，通过`pflag.Arg(i)`返回第 i 个非选项参数。参数下标 0 到 `pflag.NArg()` - 1。

指定了选项但是没有指定选项值时的默认值。

创建一个 Flag 后，可以为这个 Flag 设置`pflag.NoOptDefVal`。如果一个 Flag 具有 NoOptDefVal，并且该 Flag 在命令行上没有设置这个 Flag 的值，则该标志将设置为 NoOptDefVal 指定的值。例如：
```go
var ip = pflag.IntP("flagname", "f", 1234, "help message")
pflag.Lookup("flagname").NoOptDefVal = "4321"
```

Pflag 可以**弃用标志或者标志的简写**。弃用的标志或标志简写在帮助文本中会被隐藏，并在使用不推荐的标志或简写时打印正确的用法提示。例如，弃用名为 logmode 的标志，并告知用户应该使用哪个标志代替：
```go
// deprecate a flag by specifying its name and a usage message
pflag.CommandLine.MarkDeprecated("logmode", "please use --log-mode instead")
```

## viper

### 优先级

按`优先级`从高到低排列如下：
通过 viper.Set 函数显示设置的配置
命令行参数
环境变量
配置文件
Key/Value存储
默认值

设置默认值：viper.SetDefault(key, value)

### 环境变量

Viper 还支持`环境变量`，通过如下 5 个函数来支持环境变量：
AutomaticEnv()
BindEnv(input …string) error
SetEnvPrefix(in string)
SetEnvKeyReplacer(r *strings.Replacer)
AllowEmptyEnv(allowEmptyEnv bool)

这里要注意：Viper 读取环境变量是**区分大小写**的。
Viper 提供了一种机制来确保 Env 变量是唯一的。
通过使用 SetEnvPrefix，可以告诉 Viper 在读取环境变量时使用前缀。
BindEnv 和 AutomaticEnv 都将使用此前缀。
比如，我们设置了 viper.SetEnvPrefix(“VIPER”)，
当使用 viper.Get(“apiversion”) 时，实际读取的环境变量是VIPER_APIVERSION。

`BindEnv` 需要一个或两个参数。第一个参数是键名，第二个是环境变量的名称，环境变量的名称区分大小写。
如果未提供 Env 变量名，则 Viper 将假定 Env 变量名为：环境变量前缀_键名全大写。
例如：前缀为 VIPER，key 为 username，则 Env 变量名为VIPER_USERNAME。
**当显示提供 Env 变量名（第二个参数）时，它不会自动添加前缀**。
例如，如果第二个参数是 ID，Viper 将查找环境变量 ID。

在使用 Env 变量时，需要注意的一件重要事情是：每次访问该值时都将读取它。
Viper 在调用 BindEnv 时不固定该值。

还有一个魔法函数 `SetEnvKeyReplacer`，SetEnvKeyReplacer 允许你使用 strings.Replacer 对象来重写 Env 键。
如果你想在 Get() 调用中使用-或者.，但希望你的环境变量使用_分隔符，可以通过 SetEnvKeyReplacer 来实现。

```go
// 使用环境变量
os.Setenv("VIPER_USER_SECRET_ID", "QLdywI2MrmDVjSSv6e95weNRvmteRjfKAuNV")
os.Setenv("VIPER_USER_SECRET_KEY", "bVix2WBv0VPfrDrvlLWrhEdzjLpPCNYb")

viper.AutomaticEnv()                                             // 读取环境变量
viper.SetEnvPrefix("VIPER")                                      // 设置环境变量前缀：VIPER_，如果是viper，将自动转变为大写。
viper.SetEnvKeyReplacer(strings.NewReplacer(".", "_", "-", "_")) // 将viper.Get(key) key字符串中'.'和'-'替换为'_'
viper.BindEnv("user.secret-key")
viper.BindEnv("user.secret-id", "USER_SECRET_ID") // 绑定环境变量名到key
```

### 读取配置

Viper 提供了如下方法来读取配置：
```
Get(key string) interface{}
Get<Type>(key string) <Type>
AllSettings() map[string]interface{}
IsSet(key string) : bool
```

每一个 Get 方法在找不到值的时候都会返回零值。为了检查给定的键是否存在，可以使用 IsSet() 方法。`<Type>`可以是 Viper 支持的类型，首字母大写：Bool、Float64、Int、IntSlice、String、StringMap、StringMapString、StringSlice、Time、Duration。例如：GetInt()。

Viper 可以支持将所有或特定的值解析到结构体、map 等。可以通过两个函数来实现：
```
Unmarshal(rawVal interface{}) error
UnmarshalKey(key string, rawVal interface{}) error
```

## cobra

cobra和viper都可以调用pflag

```go
func init() {
  rootCmd.PersistentFlags().StringVar(&author, "author", "YOUR NAME", "Author name for copyright attribution")
  viper.BindPFlag("author", rootCmd.PersistentFlags().Lookup("author"))
}
```

## 设置必选标志

默认情况下，标志是可选的，我们也可以设置标志为必选，当设置标志为必选，但是没有提供标志时，Cobra 会报错。

```go
rootCmd.Flags().StringVarP(&Region, "region", "r", "", "AWS region (required)")
rootCmd.MarkFlagRequired("region")
```

### 非选项参数验证

在命令的过程中，经常会传入非选项参数，并且需要对这些非选项参数进行验证，Cobra 提供了机制来对非选项参数进行验证。可以使用 Command 的 `Args` 字段来验证非选项参数。Cobra 也内置了一些验证函数：

NoArgs：如果存在任何非选项参数，该命令将报错。
ArbitraryArgs：该命令将接受任何非选项参数。
OnlyValidArgs：如果有任何非选项参数不在 Command 的 ValidArgs 字段中，该命令将报错。
MinimumNArgs(int)：如果没有至少 N 个非选项参数，该命令将报错。
MaximumNArgs(int)：如果有多于 N 个非选项参数，该命令将报错。
ExactArgs(int)：如果非选项参数个数不为 N，该命令将报错。
ExactValidArgs(int)：如果非选项参数的个数不为 N，或者非选项参数不在 Command 的 ValidArgs 字段中，该命令将报错。
RangeArgs(min, max)：如果非选项参数的个数不在 min 和 max 之间，该命令将报错。

### hook

在运行 Run 函数时，我们可以运行一些`钩子函数`，比如 PersistentPreRun 和 PreRun 函数在 Run 函数之前执行，PersistentPostRun 和 PostRun 在 Run 函数之后执行。如果子命令没有指定`Persistent*Run`函数，则子命令将会继承父命令的`Persistent*Run`函数。这些函数的运行顺序如下：
1. PersistentPreRun
2. PreRun
3. Run
4. PostRun
5. PersistentPostRun

注意，父级的 PreRun 只会在父级命令运行时调用，子命令是不会调用的。

## iam项目阅读代码心得

通过cobra、viper、pflag构建整体框架，pflag.FlagSet有name字段，据此分组输出

iamauthserver项目认证过程：
1. apiserver的login接口使用basic认证，以配置中的secret生成token返回
2. 利用token访问apiserver来获取secretID和secretKEY
3. 使用iamctl jwt sign利用secretID和secretKEY生成访问authserver的token
4. 使用该token访问authserver

iamapiserver项目认证两种方法：
1. apiserver的login接口的basic认证获取token，再利用token去访问其他接口
2. 直接使用basic认证去访问其他接口，会被autostrategy自动判断认证


cobra、viper、pflag结合读取配置文件：
1. 创建读取config文件的pflag
2. viper读取该pflag配置内容
3. cobra运行函数中pflagset值绑定到viper
4. cobra运行函数中将viper读取内容unmarshal到options中
5. 默认的options创建是默认值，由pflag值修改

猜想：viper中存在优先级，pflag优先级最高，所以使用viper的配置时，需要先BindPFlags同步pflag，再使用Unmarshal，这样保证优先级一致
其中pflag的内容没有viper中的全，但也没有必要更新pflag的内容，后续需要配置一律从viper中读取即可

cobra外部包装一层app，会先运行cobra命令的RunE:主要作用就是根据app的参数初始化，再运行app的RunFunc:主要作用创建server并运行


iam项目业务架构

![](../../reference/pic/iam.webp)

Controller层是在gin的handleFunc中定义的，调用service，再调用repository

![](../../reference/pic/iam-controller.webp)

### 优雅关停

函数返回一个channel阻塞，该函数使用singal.Notify捕获信号
通过关闭一个channel来解除阻塞
```go
var onlyOneSignalHandler = make(chan struct{})

var shutdownHandler chan os.Signal

func SetupSignalHandler() <-chan struct{} {
    close(onlyOneSignalHandler) // panics when called twice

    shutdownHandler = make(chan os.Signal, 2)

    stop := make(chan struct{})

    signal.Notify(shutdownHandler, shutdownSignals...)

    go func() {
        <-shutdownHandler
        close(stop)
        <-shutdownHandler
        os.Exit(1) // second signal. Exit directly.
    }()

    return stop
}

func (s *grpcAPIServer) Run(stopCh <-chan struct{}) {
    listen, err := net.Listen("tcp", s.address)
    if err != nil {
        log.Fatalf("failed to listen: %s", err.Error())
    }

    log.Infof("Start grpc server at %s", s.address)

    go func() {
        if err := s.Serve(listen); err != nil {
            log.Fatalf("failed to start grpc server: %s", err.Error())
        }
    }()

    <-stopCh

    log.Infof("Grpc server on %s stopped", s.address)
    s.GracefulStop()
}
```

### sdk设计

client-go的设计模式是分多层流式调用的，在每一层都抽象出一个接口

marmotedu-sdk-go 提供了**两类客户端**，分别是 RESTClient 客户端和基于 RESTClient 封装的客户端。

RESTClient：Raw 类型的客户端，可以通过指定 HTTP 的请求方法、请求路径、请求参数等信息，直接发送 HTTP 请求，例如 
client.Get().AbsPath("/version").Do().Into() 。

基于 RESTClient 封装的客户端：例如 AuthzV1Client、APIV1Client 等，执行特定 REST 资源、特定 API 接口的请求，方便开发者调用。

