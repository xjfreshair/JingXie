---
layout: post
title: performance test report
date: 2017-11-26 15:00:00.000000000 +09:00
tags: Jekyll Github
---

# 1.概述
## 1.1项目背景
yaf属于超轻量级的c级别裸框架，只集成了基本的视图和路由控制，如果要完成一个中等以上规模的网站的话，还需要集成非常多的第三方类库，如果在线上运营环境中使用，会因为php加载大量的第三方类库而影响到性能。此次在框架中集成mysql和redis的连接池。
## 1.2测试目的
验证接口重构之后性能表现
1. 以及基于用户中心接口业务，找出系统能稳定响应的最大tps数值。
2. 在80%最大tps数值请求下，系统稳定性。

## 1.3测试内容
  本次性能测试选取接口进行测试。
  | 序号
## 1.4术语表
 TPS： 每秒钟事务处理速率
 QPS:  数据库每秒查询率(Query Per Second)
 VUS： 虚拟用户数
# 2.测试环境
## 2.1 服务器端硬件环境
本次测试在阿里云

| 服务器 | system       | cpu    | memory | disk     | ip            |
| ------ | ------------ | ------ | ------ | -------- | ------------- |
| PHP    | Centeros 6.6 | 4 core | 8g     | 200g ssd | 172.22.33.227 |
| MySql  | Centeros 6.6 | 4 core | 8g     | 200g ssd | 172.22.33.228 |
| Redis  | Centeros 6.6 | 4 core | 8g     | 200g ssd | 172.22.33.228 |

## 2.2客户端硬件环境

| 服务器 | 数量 | cpu/memory                                      | 型号/操作系统 | IP                                                          |
| ------ | ---- | ----------------------------------------------- | ------------- | ----------------------------------------------------------- |
| 压力机 | 6台  | Intel(R) Xeon® CPU E5-2650 v2 @ 2.60GHz  4核/8G | Centos 6.5    | 172.22.33.229-172.22.33.232     172.22.33.224-172.22.33.226 |

## 2.3软件环境

| 资源               | 描述                                                      |
| ------------------ | --------------------------------------------------------- |
| 操作系统           | Centos 6.6                                                |
| 中间件（编译安装） | php-7.1.6  yaf-3.0.4 swoole-1.9.13 gcc-4.8.5 Tengin-2.0.3 |
| 数据库             | mysql-5.7 rdis-2.8.17                                     |
| 测试工具           | Jmeter-3.1                                                |
| 监控工具           | Server-Agent-2.2.1                                        |

## 2.4网络拓扑图
![](https://github.com/xjfreshair/JingXie/raw/master/images/0.png)
# 3测试方法
## 3.1 通过标准

- 响应时间：稳定性测试场景下，服务响应时间3s之内。

- 业务成功率：稳定性测试场景下，业务成功率>99%。

- 服务器资源：服务器运行稳定，主要资源90%以下。

- 在压力场景下服务不宕机。

## 3.2 测试场景

- 测试场景

| 序号 | 测试场景              | 接口                                   | 测试步骤                              |
| ---- | --------------------- | -------------------------------------- | ------------------------------------- |
| 1    | 压力测试              | ucenter/role/getRoleTemplateByOrgroot  | 30000线程300秒之内启动                |
| 2    | 80%最大压力稳定性测试 | /ucenter/role/getRoleTemplateByOrgroot | 根据压力测试中80%tps(3000)，运行1小时 |

- 测试数据

| 数据类别 | 表/数据类型                   | 数据量（条） | 备注             |
| -------- | ----------------------------- | ------------ | ---------------- |
| Mysql    | Ucenter_role                  | 75976        | 和线上数据量一致 |
| Mysql    | Ucenter_role_quotes_resources | 97           | 和线上数据量一致 |
| Mysql    | Ucenter_roletemplate_resource | 22           | 和线上数据量一致 |

## 3.3测试执行

本次测试采用Jmeter性能测试工具，模拟报文向服务器发送请求，测试过程中采取由少到多梯度加压的方式。

### 3.3.1 场景1
#### 3.3.1.1 场景说明
getRoleTemplateByOrgroot接口压力测试，30000线程，300秒内启动。找出系统tps拐点。基于此拐点80%tps，作为稳定性测试标准。

观察：
      1. 在压力测试场景下，服务是否停止响应。
      2. 根据TPS找出性能拐点。
### 3.3.1.2 结果图形说明

![](https://github.com/xjfreshair/JingXie/raw/master/images/1.png)

 ** 事务数和线程关系 **

![ ](https://github.com/xjfreshair/JingXie/raw/master/images/2.png)

** TPS随时间变化 **

![](https://github.com/xjfreshair/JingXie/raw/master/images/3.png)

** 响应时间随时间变化 **

![](https://github.com/xjfreshair/JingXie/raw/master/images/4.png)

** Nginx告警日志 **

017/08/15 11:51:01 [alert] 23488#0: *748199 10240 worker_connections are not enough while connecting to upstream, client: 172.22.33.230, server: ucenter.com, request: "GET /service.php?method=role.getRoleTemplateByOrgroot&orgroot=200NOI HTTP/1.1", upstream: "fastcgi://127.0.0.1:9001", host: "ucenter.com"

** PHP慢日志 **

无

## 3.3.1.3 结果分析
** 结论 **
1.  3000线程300秒内启动，TPS峰值3000。在6分钟左右，出现错误。随时时间增加请求积压，错误增加，tps数量下降。

2. 系统在出现性能拐点之后，系统未崩溃。

** 测试通过 **

** 建议： **
1. 主要错误出现在nginx转发请求fpm中，连接数不够错误。线上部署两台服务器，tps数量将高于测试环境的性能。

### 3.3.2 场景 2
#### 3.3.2.1场景说明
按照2000线程，控制TPS为2400

** 观察 **
1. 系统响应时间是否稳定
2. 服务器资源占用是否稳定。
#### 3.3.2.2 结果图形说明

![](https://github.com/xjfreshair/JingXie/raw/master/images/5.png)

- 网络流量和线程数的关系

![](https://github.com/xjfreshair/JingXie/raw/master/images/6.png)

- TPS随时间变化

![](https://github.com/xjfreshair/JingXie/raw/master/images/7.png)

- 响应时间随时间变化

![](https://github.com/xjfreshair/JingXie/raw/master/images/8.png)

- 响应时间和请求百分比变化

![](https://github.com/xjfreshair/JingXie/raw/master/images/9.png)

- 聚合测试报告

**PHP服务器资源监控**
![](https://github.com/xjfreshair/JingXie/raw/master/images/10.png)

**MySql服务器资源监控**
![](https://github.com/xjfreshair/JingXie/raw/master/images/11.png)

**PHP慢查询日志**

无

**PHP错误日志**

无

**MySql慢日志**

无

#### 3.3.2.3 结果分析
**结论：**

1.  2000线程，2400TPS压力情况下，系统响应时间在3S之类
2.  服务器主要资源占用在90%以下。

**测试通过**


# 4.测试问题说明

# 5.结论和建议

## 6.1测试结论
  **性能测试通过**
## 6.2风险和建议

| 序号 | 风险描述                                                     | 风险影响                                   | 严重程度 | 建议             |
| ---- | ------------------------------------------------------------ | ------------------------------------------ | -------- | ---------------- |
| 1    | 本次测试环境与生产环境存在差异，不能完全准确反映生产环境情况 | 线上采用阿里RDS和redis，性能比测试环境更好 | 低       | 生产环境加强监控 |
