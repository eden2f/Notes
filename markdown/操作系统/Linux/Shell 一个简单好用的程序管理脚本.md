# Shell 一个简单好用的程序管理脚本

## 使用示例
### 启动程序
```shell
./monitor.sh start
```
### 停止程序
```shell
./monitor.sh stop
```
### 重启程序
```shell
./monitor.sh restart
```
### 查看程序运行状态
```shell
./monitor.sh status
```
## Shell脚本模板
根据自己环境调整，大部分人应该只需修改Jar包路径
```shell
#!/bin/bash

# export GOOGLE_APPLICATION_CREDENTIALS="配置服务需要的系统环境变量"

#配置jar包所在路径，替换为自己的路径
APP_NAME=/xxxxx/xxxxxxx.jar

#执行 sh restart.sh 会打印的帮助信息
usage() {
    echo "Usage: sh zxjava.sh [start|stop|restart|status]"
    exit 1
}

#判断服务是否存在
is_exist(){
  pid=`ps -ef|grep $APP_NAME|grep -v grep|awk '{print $2}'`
  #如果不存在返回1，存在返回0   
  if [ -z "${pid}" ]; then
   return 1
  else
    return 0
  fi
}

#启动程序
start(){
  is_exist
  if [ $? -eq 0 ]; then
    echo "${APP_NAME} is already running. pid=${pid}"
  else
    nohup java -Xmx512m -Dspring.profiles.active=test -jar ${APP_NAME} &
  fi
}

#停止程序
stop(){
  is_exist
  if [ $? -eq "0" ]; then
    kill -9 $pid
  else
    echo "${APP_NAME} is not running"
  fi
}

#查看程序运行状态
status(){
  is_exist
  if [ $? -eq "0" ]; then
    echo "${APP_NAME} is running. Pid is ${pid}"
  else
    echo "${APP_NAME} is NOT running."
  fi
}

#重启程序
restart(){
  stop
  sleep 5
  start
}

#根据输入，执行相应动作
case "$1" in
  "start")
    start
    ;;
  "stop")
    stop
    ;;
  "status")
    status
    ;;
  "restart")
    restart
    ;;
  *)
    usage
    ;;
esac
```
