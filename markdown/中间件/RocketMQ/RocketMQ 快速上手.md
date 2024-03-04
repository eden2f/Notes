# RocketMQ 快速上手

## 系统要求
1. 64bit 的 Linux、 Unix 或 Mac (Windows 也支持)
2. 64bit JDK 1.8+;
3. Maven 3.2.x;
4. Git;
5. 4g+ free disk for Broker server
## 官网下载链接
[下载当前最新版本:我使用的是4.7.1版](http://rocketmq.apache.org/dowloading/releases/)
```shell
> unzip rocketmq-all-4.7.1-source-release.zip
> cd rocketmq-all-4.7.1/
> mvn -Prelease-all -DskipTests clean install -U
> cd distribution/target/rocketmq-4.7.1/rocketmq-4.7.1
```
## Linux
### Start Name Server
```shell
> nohup sh bin/mqnamesrv &
> tail -f ~/logs/rocketmqlogs/namesrv.log
> The Name Server boot success...
```
### Start Broker
```shell
> nohup sh bin/mqbroker -n localhost:9876 &
> tail -f ~/logs/rocketmqlogs/broker.log 
> The broker[%s, 172.30.30.233:10911] boot success...
```
### Send & Receive Messages
Before sending/receiving messages, we need to tell clients the location of name servers. RocketMQ provides multiple ways to achieve this. For simplicity, we use environment variable `NAMESRV_ADDR`
```shell
> export NAMESRV_ADDR=localhost:9876
> sh bin/tools.sh org.apache.rocketmq.example.quickstart.Producer
 SendResult [sendStatus=SEND_OK, msgId= ...

> sh bin/tools.sh org.apache.rocketmq.example.quickstart.Consumer
 ConsumeMessageThread_%d Receive New Messages: [MessageExt...
```
### Shutdown Servers
```shell
> sh bin/mqshutdown broker
The mqbroker(36695) is running...
Send shutdown request to mqbroker(36695) OK

> sh bin/mqshutdown namesrv
The mqnamesrv(36664) is running...
Send shutdown request to mqnamesrv(36664) OK
```
## Windows(10+)
### 配置系统环境变量
```properties
ROCKETMQ_HOME="D:\rocketmq"
NAMESRV_ADDR="localhost:9876"
```
### Start Name Server
设置正确的环境变量后，打开新的powershell窗口。然后将目录更改为环境变量ROCKETMQ_HOME指定的路径后运行:
```powershell
.\bin\mqnamesrv.cmd
```
### Start Broker
设置正确的环境变量后，打开新的powershell窗口。然后将目录更改为环境变量ROCKETMQ_HOME指定的路径后运行:
```powershell
.\bin\mqbroker.cmd -n localhost:9876 autoCreateTopicEnable=true
```
### Send & Receive Messages
#### Send Messages
设置正确的环境变量后，打开新的powershell窗口。然后将目录更改为环境变量ROCKETMQ_HOME指定的路径后运行:
```powershell
.\bin\tools.cmd  org.apache.rocketmq.example.quickstart.Producer
```
#### Receive Messages
在发送消息成功后, 我们可以尝试消费者信息。
设置正确的环境变量后，打开新的powershell窗口。然后将目录更改为环境变量ROCKETMQ_HOME指定的路径后运行:
```powershell
.\bin\tools.cmd  org.apache.rocketmq.example.quickstart.Consumer
```
### Shutdown Servers
通常，您可以直接关闭这些powershell窗口。(请勿在生产环境中使用)
