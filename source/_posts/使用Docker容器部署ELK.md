---
title: 使用Docker容器部署ELK
date: 2018-07-03 23:11:29
categories: 
- 后端
- 运维
tags:
- Docker
- Shell
- Linux
---

# 选择镜像

选择docker images(在hub.docker.com 搜索 elk 选择 start或pulls比较多的镜像) 本次安装选择的是 sebp/elk,默认本地已安装docker环境
docker pull sebp/elk

1. 选择docker镜像登录 hub.docker.com 搜索 'elk', 选择stars 和 pulls 比较多的镜像

2. 下载镜像docker pull sebp/elk

# 启动docker容器

```shell
docker run --ulimit nofile=65536:65536 -p 5601:5601 -p 9200:9200 -p 5044:5044 -p 5045:5045 -p 5046:5046 -d --restart=always -v /etc/logstash:/etc/logstash -v /etc/localtime:/etc/localtime --name elk sebp/elk
```

## 参数介绍

```txt
--ulimit  来修改容器的ulimit参数(该镜像默认的ulimit值为4096。 不带该参数，启动容器会出现类似 “max file descriptors [4096] for elasticsearch process is too low, increase to at least [65536]” 错误)
    -p  指定容器和宿主机映射端口
        5601：  kibana服务端口 HTTP  (web访问)
        9200： Elasticsearch 开发端口 HTTP,保存数据到Elasticsearch中使用
        5044： logstash  收集日志端口 TCP
    -v  挂载目录 可以将logstash 的配置文件挂载在宿主机的目录上，方便随时修改，修改后的配置文件会同步到容器中。
        挂载 /etc/localtime 该目录是为了保证容器和宿主机的时区相同。
        通过-v参数，冒号前为宿主机目录，必须为绝对路径，冒号后为镜像内挂载的路径。
        现在镜像内就可以共享宿主机里的文件了。
```

# 配置logstash

创建以下路径，并在其中创建logstash配置文件 /etc/logstash/conf.d/logstash.conf

```conf
input {
    beats {
        port => 5044
    }
}
output {
    elasticsearch {
        hosts => ["127.0.0.1:9200"]
        index => "test" ##对应es索引名
    }
}
```

修改完配置文件后，执行如下命令来重启logstash  
docker exec elk /etc/init.d/logstash restart

# 客户端服务器

1. 安装filebeat  

Debian/Ubuntu:

```shell
curl -L -O https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-5.3.2-amd64.deb
sudo dpkg -i filebeat-5.3.2-amd64.deb
```

Redhat/Centos/Fedora:

```shell
curl -L -O https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-5.3.2-x86_64.rpm
sudo rpm -vi filebeat-5.3.2-x86_64.rpm
```

2. 配置filebeat （默认文件路径 /etc/filebeat/filebeat.yml）

```yml
filebeat.prospectors:
    - input_type: log
document_type: info
paths:
    - /data/logs/info.log
output.logstash:
    hosts: ["18.18.18.18:5044"]
```

paths 需要收集的日志文件路径  
hosts: logstash 服务IP和端口

3. 测试配置文件语法是否正确

```shell
/usr/bin/filebeat.sh -configtest /etc/filebeat/filebeat.yml
```

修改完成后，重启filebeat

```shell
/etc/init.d/filebeat restart
```

# 访问kibana页面

1. 在已经安装filebeat的客户端服务器上，测试日志收集

```shell
echo 'nihao' >> /data/logs/info.log
```

2. 在浏览器输入 18.18.18.18:5601（IP修改为ELK所在服务器的IP地址）创建 index pattern，选择刚才创建的“test“索引即可，可以在web上看到输入的测试日志。

3. 在 http://localhost:9200/_search?pretty 可看到es中的全部索引数据。

4. 在 http://localhost:9200/_cat/indices?v 可看到es中的全部索引。