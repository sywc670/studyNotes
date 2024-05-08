# ferry

运维工单平台后端

启动：cobra、viper、pflag

viper可以自动判断配置文件类型取值
viper有一个全局实例，可以添加多个子实例

cobra添加两个子命令，一个启动服务器，一个初始化数据库数据

初始化数据库、启动参数、JWT、SSL、日志