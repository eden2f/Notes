# 持久化延时任务框架 common-schedule

## 概述
本项目是基于 ScheduledExecutorService 实现一个支持持久化的延时任务框架.
说来有趣, 由于在生产环境已经用着"前辈"自己实现的延时任务在跑着了, 改着改着就有了这个项目.
至于为什么不用各种知名的开源的社区资源丰富的框架, 因为业务需求简单, 当前实现的功能已经满足了, 重点是很简单, 容易上手.
主要提供了以下特性:

- 支持任务持久化, down机重启可继续执行
- 在分布式集群环境下保证任务最多被执行一次
- 将任务持久化处理业务与具体的持久化实现方式分离, 提高拓展性
- 支持取消已提交的任务
### 代码仓库

- [Github项目路径 : common-schedule-all](https://github.com/eden2f/common-schedule-all)
- [Gitee项目路径 : common-schedule-all](https://gitee.com/eden2f/common-schedule-all)
## 项目依赖
| 组件名 | 版本号 |
| --- | --- |
| JDK | 1.8 |
| Lombok | 1.18.8 |
| slf4j-api | 1.7.7 |

## 项目设计
## 使用示例项目
common-schedule-example
### 项目集成框架技术
| 组件名 | 版本号 |
| --- | --- |
| Spring Boot | * |
| Spring Data Jpa | * |
| MySql | * |
| JDK | 1.8 |
| Lombok | * |
| Swagger | * |

### 功能测试
数据库初始化
```
CREATE SCHEMA `common_schedule_example` DEFAULT CHARACTER SET utf8mb4 ;
```
启动入口 : com.eden.common.schedule.example.CommonScheduleExampleApplication
测试页面 : [http://127.0.0.1:8080/common-schedule-example/swagger-ui.html#/](http://127.0.0.1:8080/common-schedule-example/swagger-ui.html#/)

![](https://gitee.com/eden2f/ImageHosting/raw/master/imgs/20210514104054.png#id=rgw61&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
