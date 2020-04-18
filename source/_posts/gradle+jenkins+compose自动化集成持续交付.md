---
title: gradle+jenkins+compose自动化集成持续交付
date: 2017-12-06 16:32:41
updated: 2017-12-06 19:04:12
categories: 
- 后端
- 运维
tags:
- Docker
- Linux
- Gradle
- Jenkins
- Shell
- 分布式
---
## 简介
本文将介绍如何使用本地搭建的jenkins自动化持续集成工具，配合Docker swarm去为Docker容器进行编排、提供集群服务，线上发布、部署。

以下操作全部在阿里云容器服务中进行。使用阿里云容器服务的原因是：可以省去一些运维、维护的人力成本。

## 背景
公司开发产品使用的是spring cloud微服务技术栈，线上需要使用Docker对各个单独的模块服务进行容器化部署，Docker技术发展为微服务落地提供了更加便利的环境,而Docker Swarm  
<!--more-->
是 Docker 官方三剑客项目之一，提供 Docker 容器集群服务，是 Docker 官方对容器云生态进行支持的核心方案，便于我们日后对应用进行垂直扩展，弹性伸缩。

## 难题

 1. 部署步骤繁琐。 -->  **jenkins**
需要将项目编译打包，构建镜像、登陆Docker仓库，tag镜像，最后再push上去。过程中还会参杂着其他的备份，替换等各种命令。


2. 编排时，容器服务中的配置信息的统一/差异化。 --> **Docker Compose**
由于我们使用Docker swarm对容器进行编排，集群部署，将多个 Docker 主机封装为单个大型的虚拟 Docker 主机，于是需要根据业务对容器中的配置信息进行统一/差异化。比如一组容器服务由三个容器集群组成，但是我们其中一个容器里面的应用的DataBase配置需要跟其他两个不一样。


3. 热更新困难。 --> **蓝绿发布**
 版本迭代的时候，需要对线上应用进行热发布更新，即更新过程中不能对用户产生影响或影响可忽略。

## 操作步骤
0. 在Jenkins中新建项目

1. 打开 Jenkins

   进入项目配置页面

   {% asset_img 1.png 图片 %}

2.   更改镜像版本号(如不更改，则原镜像版本会变为null)

   {% asset_img 2.png 图片 %}

3.   点击立即构建

   {% asset_img 3.png 图片 %}

4.   进入阿里云容器服务控制台,点击order-service的变更配置

  {% asset_img 4.png 图片 %}

5.  更新前的编排文件：

   {% asset_img 5.png 图片 %}

6.   更新后的编排文件:

   {% asset_img 6.png 图片 %}

7.  点击确定

   进入应用详情。

   {% asset_img 7.png 图片 %}

   如果发现新发布服务失败，则点击该失败服务的变更配置。勾选restart:always，然后更新即可。

   查看新应用的容器列表里面的端口映射，可在路由列表，将流量全部切换到新版本进行初步测试。

   {% asset_img 8.png 图片 %}

   查看节点公网ip：

  {% asset_img 9.png 图片 %}

   测试完成后选择确认蓝绿发布新应用

   {% asset_img 10.png 图片 %}
8. 注意事项

   docker相关都在/var/lib/docker

   定期 df -sh *，free一下，关注磁盘和内存状态，及时清理无用的docker挂载数据卷。
 
   >``for i in `find . -name "*.log"`; do cat /dev/null >$i; done``
 
   清空文件名以log为后缀的日志

   >docker rm $(docker ps -a -q)
  
   删除所有未运行 Docker 容器

   >docker images|grep none|awk '{print $3}'|xargs docker rmi

   批量删除 tag为none的镜像

### application.yml DEMO
``` yaml
server:
  port: ${ADDITIONAL_TOOL_SERVER_PORT}
eureka:
  client:
    service-url:
      defaultZone: ${ADDITIONAL_EUREKA_SERVER_LIST}
logging.level.project.user.UserClient: INFO
spring:
   datasource:
     url: ${DATABASE_URL}
     username:  ${DATABASE_USERNAME}
     password:  ${DATABASE_PASSWORD}
   redis:
    database: ${REDIS_DATABASE}
    host:  ${REDIS_HOST}
    password: ${REDIS_PASSWORD}
    port: ${REDIS_PORT}
    timeout: ${REDIS_TIMEOUT}
    pool:
      max-idle: ${REDIS_MAX_IDLE}
      min-idle: ${REDIS_MIN_IDLE}
      max-active: ${REDIS_MAX_ACTIVE}
      max-wait: ${REDIS_MAX_WAIT}

```
### docker-compose.yml DEMO
``` yaml
version: '2'
services:
  tool-app1:
    image: registry.cn-shenzhen.aliyuncs.com/zwx_cloud/tool-app:1.0.0
    container_name: tool-app1
    ports:
       - "10086:10086"
    restart: always
    labels:
      aliyun.routing.port_10086: tool-app
    environment:
      - TZ=Asia/Shanghai
      - ADDITIONAL_TOOL_SERVER_PORT=10106
      - ADDITIONAL_EUREKA_SERVER_LIST=http://eureka1:10011/eureka/,http://eureka2:10012/eureka/,http://eureka3:10013/eureka/
      - DATABASE_URL=jdbc:mysql://123
      - DATABASE_USERNAME=123
      - DATABASE_PASSWORD=123
      - REDIS_DATABASE=1
      - REDIS_HOST=123
      - REDIS_PASSWORD=123
      - REDIS_PORT=123
      - REDIS_TIMEOUT=500
      - REDIS_MAX_IDLE=20
      - REDIS_MIN_IDLE=8
      - REDIS_MAX_ACTIVE=20
      - REDIS_MAX_WAIT=-1
  tool-app2:
    image: registry.cn-shenzhen.aliyuncs.com/zwx_cloud/tool-app:1.0.0
    container_name: tool-app2
    ports:
       - "10096:10096"
    restart: always
    labels:
      aliyun.routing.port_10096: tool-app
    environment:
      - TZ=Asia/Shanghai
      - ADDITIONAL_TOOL_SERVER_PORT=10106
      - ADDITIONAL_EUREKA_SERVER_LIST=http://eureka1:10011/eureka/,http://eureka2:10012/eureka/,http://eureka3:10013/eureka/
      - DATABASE_URL=jdbc:mysql://123
      - DATABASE_USERNAME=123
      - DATABASE_PASSWORD=123
      - REDIS_DATABASE=1
      - REDIS_HOST=123
      - REDIS_PASSWORD=123
      - REDIS_PORT=123
      - REDIS_TIMEOUT=500
      - REDIS_MAX_IDLE=20
      - REDIS_MIN_IDLE=8
      - REDIS_MAX_ACTIVE=20
      - REDIS_MAX_WAIT=-1
  tool-app3:
    image: registry.cn-shenzhen.aliyuncs.com/zwx_cloud/tool-app:1.0.0
    container_name: tool-app3
    ports:
       - "10106:10106"
    restart: always
    labels:
      aliyun.routing.port_10106: tool-app
    environment:
      - TZ=Asia/Shanghai
      - ADDITIONAL_TOOL_SERVER_PORT=10106
      - ADDITIONAL_EUREKA_SERVER_LIST=http://eureka1:10011/eureka/,http://eureka2:10012/eureka/,http://eureka3:10013/eureka/
      - DATABASE_URL=jdbc:mysql://123
      - DATABASE_USERNAME=123
      - DATABASE_PASSWORD=123
      - REDIS_DATABASE=1
      - REDIS_HOST=123
      - REDIS_PASSWORD=123
      - REDIS_PORT=123
      - REDIS_TIMEOUT=500
      - REDIS_MAX_IDLE=20
      - REDIS_MIN_IDLE=8
      - REDIS_MAX_ACTIVE=20
      - REDIS_MAX_WAIT=-1

```