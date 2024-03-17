# ElasticSearch 设置账号密码

## 前言
7.0以后es把基础的安全模块免费集成了。很棒。现在试试设置吧。
官网安全模块文档，其中包含了一个安全入门教程。
下面基本都是以我的操作为例，实际上可以按照官网文档进行其它的尝试。
## 设置账号密码
单节点，在elasticsearch.yml文件里增加配置
```properties
xpack.security.enabled: true
xpack.security.transport.ssl.enabled: true
```
初始化密码需要在es启动的情况下进行设置，按照提示输入各个内置用户的密码。
```shell
[esuser@localhost elasticsearch-7.12.0]$./bin/elasticsearch -d
[esuser@localhost elasticsearch-7.12.0]$./bin/elasticsearch-setup-passwords interactive
```
