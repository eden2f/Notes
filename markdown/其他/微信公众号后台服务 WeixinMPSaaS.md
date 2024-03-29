# 微信公众号后台服务 WeixinMPSaaS

### 介绍
微信公众号后台服务SaaS, 支持微信公众平台(订阅号、服务号、企业号、小程序).

- 支持多个微信公众号配置
- 动态更新配置

本项目仍在不断迭代开发中,遇到问题请提 issues , 谢谢配合.
> - [微信公众号开发官方文档](https://developers.weixin.qq.com/doc/)
> - [微信公众号开发SDK](https://github.com/Wechat-Group/WxJava)

### 代码仓库

- [Github项目路径: eden2f/weixin-mp-saas](https://github.com/eden2f/weixin-mp-saas)
- [Gitee项目路径: eden2f/weixin-mp-saas](https://gitee.com/eden2f/weixin-mp-saas)
### 软件架构
本项目基于 Spring Boot 构建, 对接微信公众号Api.
![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240310225028.png#id=XCWJT&originHeight=500&originWidth=770&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
### 安装教程

- 创建MySQL数据库
```
CREATE SCHEMA `weixin_mp_saas` DEFAULT CHARACTER SET utf8mb4 ;
```

- 生成可执行jar包
```shell
mvn clean package -Dmaven.test.skip=true
```

- 启动服务
```shell
java -jar weixin-mp-saas.jar
```

- 配置项 
   - server.port 服务端口号, 默认是 8080
   - mysql.host 默认是 127.0.0.1:3306
   - mysql.username 默认是 root
   - mysql.password 默认是 toor
   - mysql.database.name 数据库名称 weixin_mp_saas
```shell
java -jar -Dserver.port=8080 -Dmysql.host=localhost:3306 -Dmysql.username=root -Dmysql.password=toor -Dmysql.database.name=weixin_mp_saas weixin-mp-saas.jar
```
### 使用说明
**注意：需先在微信公众平台完成"IP白名单"设置**
#### 新增微信公众号配置

- 调用Http接口 
   - 开发者密码(AppSecret) : "secret"
   - 消息加解密密钥 : "aesKey"
   - 开发者ID(AppID) : "appid"
   - 令牌(Token) : "token"
   - 应用标识 : "appTag"

appTag与微信无关, 是本项目用于标识应用的, 因为本项目可支持多个微信公众号.
appTag 就是用来区别不同公众号的一个配置项.
假设 appTag 设置为 "WeixinMPSaaS", 那么在公众号基本配置页面配置如下 :
![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240310225100.png#id=FHL0A&originHeight=418&originWidth=1184&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
Http请求示例 : curl命令
```shell
curl --location --request POST 'http://127.0.0.1:8080/config/weixin/openapi/add' \
--header 'Content-Type: application/json' \
--data-raw '{
    "appTag":"xx",
    "secret":"xx",
    "aesKey":"xx",
    "appid":"xx",
    "token":"xx",
    "verifyValue":"xx",
    "verifyKey":"xx"
}'
```
#### 更新微信公众号配置
大多数请求参数与"新增微信公众号配置"接口一样，不再赘述.
openapiConfigVerify.secret 用于验证,当其等于原有配置的secret,才允许更新.
```shell
curl --location --request POST 'http://127.0.0.1:8080/config/weixin/openapi/update' \
--header 'Content-Type: application/json' \
--data-raw '{
    "appTag":"xx",
    "secret":"xx",
    "aesKey":"xx",
    "appId":"xx",
    "token":"xx",
    "verifyValue":"",
    "verifyKey":"",
    "openapiConfigVerify":{
        "secret":"xx"
    }
}'
```
### 效果演示
![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240310225405.png#id=MwrSE&originHeight=258&originWidth=258&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240310225420.png#id=WMmnB&originHeight=1552&originWidth=1116&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
### 其他配置
#### nignx 反向代理
```shell
server {

	# 极简配置, 需要其他配置项自行补上
	listen 80;

	location /weixin-mp-saas/ {
		proxy_pass http://127.0.0.1:8080/weixin-mp-saas/;
	}
}
```
#### 关闭跨公众号搜索能力
secret 用于验证,当其等于原有配置的secret,才允许更新.
```shell
curl --location --request POST 'http://127.0.0.1:8080/weixin-mp-saas/config/weixin/drainage/enable' \
--header 'Content-Type: application/json' \
--data-raw '{
    "appTag":"test",
    "secret":"testSecret",
    "enable" : "false"
}'
```
