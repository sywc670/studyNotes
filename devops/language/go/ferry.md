### ferry

运维工单平台后端

启动：cobra、viper、pflag

viper可以自动判断配置文件类型取值
viper有一个全局实例，可以添加多个子实例

cobra添加两个子命令，一个启动服务器，一个初始化数据库数据

初始化数据库、启动参数、JWT、SSL、日志

#### jwt使用

jwt的生成是在login接口上，通过authenticator，后续访问api接口，加入鉴权中间件，包含authorizator


#### 任务的调度执行

使用`github.com/RichardKnop/machinery/v1`

用redis作为broker新建machinery.Server，创建machinery.Worker，启动broker的消费