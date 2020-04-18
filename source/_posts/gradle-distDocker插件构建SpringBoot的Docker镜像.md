---
title: gradle-distDocker插件构建SpringBoot的Docker镜像
date: 2017-11-07 18:11:24
updated: 2017-11-08 17:16:01
categories: 
- 后端
- 运维
tags:
- Docker
- Gradle
- Java
---
通常我们使用 Dockerfile 来构建项目的Docker 镜像，但是也有需求希望使用 gralde 在编译项目的时候一起把镜像给构建并上传，所以该教程讲解了在 gradle中 编写配置 Dockerfile 并生成镜像的过程。
####  **1. 添加依赖**
教程使用[gradle-docker](https://github.com/Transmode/gradle-docker)插件来实现，在 Gradle 的脚本里配置 dockerfile 的构建镜像功能。

只需要在dependencies添加依赖就能使用 docker 插件。
<!-- more -->
build.gradle中的配置如下，其他配置省略：
```
buildscript {
   ...
    dependencies {
        //添加gradle-docker 依赖，版本1.2
        classpath 'se.transmode.gradle:gradle-docker:1.2'
    }
}

```
####  **2. 加入插件**
将插件加入配置文件build.gradle 
```
apply plugin: 'application' //可选
apply plugin: 'docker'
```
如果添加了application插件的话，默认gradle-docker插件会添加一个distDocker的 gralde task，用来构建一个包含所有程序文件的 docker 镜像。

####  **3. 指定Dockerfile所需**
Dockerfiles 包含一些有关image相应的指令要求。
Dockerfile关键词|gradle-task方法
:-:|:-:|:-
ADD | addFile(Closure copySpec)
 &nbsp;| addFile(String source, String dest)
  &nbsp;| addFile(File source, String dest)
CMD | defaultCommand(List cmd)
ENTRYPOINT|entryPoint(List entryPoint)
ENV|setEnvironment(String key, String val)
EXPOSE|exposePort(Integer port)
&nbsp;|exposePort(String port)
RUN	|runCommand(String cmd)
USER|switchUser(String userNameOrUid)
VOLUME|volume(String... paths)
WORKDIR|workingDir(String dir)

这里的springBoot项目只需要指定字符编码和时区：
```
    distDocker {
        runCommand("echo \"Asia/shanghai\" > /etc/timezone;")
        setEnvironment("LANG", "zh_CN.UTF-8;")
    }
```
依次可以想象，在插件中加入addFile实际是Docker的ADD指令，而runCommand会对应aRUN，而setEnvironment会创建一个ENV指令。


####  **4. 构建运行**
控制台中执行命令：gradle distDocker
等待出现BUILD SUCCESSFUL就证明编译成功了。
使用docker images命令就可以看到新生成的镜像了。