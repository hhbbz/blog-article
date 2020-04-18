---
title: Docker容器占用磁盘内存过大的问题排查
date: 2018-03-28 20:28:01
updated: 2018-03-29 22:13:52
categories: 
- 后端
- 运维
tags:
- Docker
- Linux
---
# 问题描述

同事在生产环境中使用Docker去部署ELK日志搜集系统，过程中没有将容器与数据卷挂载。于是持久化的数据都落在了/var/lib/docker/overlay2中。由于服务器需要清理服务器磁盘空间，所以要想无差错清理数据卷中的数据，需要对Docker的文件系统和存储驱动做了解和熟悉。

## 文件系统OverlayFS

　　OverlayFS是一种和AUFS很类似的文件系统，与AUFS相比，OverlayFS有以下特性：
　　　1) 更简单地设计；
　　　2) 从3.18开始，就进入了Linux内核主线；
　　　3) 可能更快一些。
　　因此，OverlayFS在Docker社区关注度提高很快，被很多人认为是AUFS的继承者。就像宣称的一样，OverlayFS还很年轻。所以，在生成环境使用它时，还是需要更加当心。
<!-- more -->
　　Docker的overlay存储驱动利用了很多OverlayFS特性来构建和管理镜像与容器的磁盘结构。
　　自从Docker1.12起，Docker也支持overlay2存储驱动，相比于overlay来说，overlay2在inode优化上更加高效。但overlay2驱动只兼容Linux kernel4.0以上的版本。

**`注意：自从OverlayFS加入kernel主线后，它在kernel模块中的名称就被从overlayfs改为overlay了。但是为了在本文中区别，我们使用OverlayFS代表整个文件系统，而overlay/overlay2表示Docker的存储驱动。`**

## 存储驱动overlay和overlay2

### OverlayFS（overlay）的镜像分层与共享

　　OverlayFS使用两个目录，把一个目录置放于另一个之上，并且对外提供单个统一的视角。这两个目录通常被称作层，这个分层的技术被称作union mount。术语上，下层的目录叫做lowerdir，上层的叫做upperdir。对外展示的统一视图称作merged。
　　如下图所示，Overlay在主机上用到2个目录，这2个目录被看成是overlay的层。 upperdir为容器层、lowerdir为镜像层使用联合挂载技术将它们挂载在同一目录(merged)下，提供统一视图。
{% asset_img overlay_constructs.jpg 图片 %}
　　注意镜像层和容器层是如何处理相同的文件的：容器层（upperdir）的文件是显性的，会隐藏镜像层（lowerdir）相同文件的存在。容器映射（merged）显示出统一的视图。
　　overlay驱动只能工作在两层之上。也就是说多层镜像不能用多层OverlayFS实现。替代的，每个镜像层在/var/lib/docker/overlay中用自己的目录来实现，使用硬链接这种有效利用空间的方法，来引用底层分享的数据。注意：Docker1.10之后，镜像层ID和/var/lib/docker中的目录名不再一一对应。
　　创建一个容器，overlay驱动联合镜像层和一个新目录给容器。镜像顶层是overlay中的只读lowerdir，容器的新目录是可写的upperdir。

#### overlay中镜像和容器的磁盘结构

　　下面的docker pull命令展示了Docker host下载一个由5层组成的镜像。

``` shell
$ docker pull ubuntu

Using default tag: latest
latest: Pulling from library/ubuntu
c62795f78da9: Pull complete
d4fceeeb758e: Pull complete
5c9125a401ae: Pull complete
0062f774e994: Pull complete
6b33fd031fac: Pull complete
Digest: sha256:c2bbf50d276508d73dd865cda7b4ee9b5243f2648647d21e3a471dd3cc4209a0
Status: Downloaded newer image for ubuntu:latests
```

此时，在路径/var/lib/docker/overlay/下，每个镜像层都有一个对应的目录，包含了该层镜像的内容，通过tree命令发现，每个镜像层下只包含一个root目录。 (因为docker1.10开始，使用基于内容的寻址，因此目录名和镜像层的id不一致)。

```shell
$ /home/hhbbz# tree -L 2 /var/lib/docker/overlay/

/var/lib/docker/overlay/
├── 3279a41fd358cdda798d99cc2da0425b4836a489083ae9db4aedc834f426915a
│   └── root
├── 33629fe3999e692ecbeb340f855a2d05ddf4e173ea915041be217e1c9cc8a48f
│   └── root
├── 6ab876fcc7ad584dd1eebae0d9304abb22561012f6adb87828f57906d799c33b
│   └── root
├── 77806a4b8257cd5508a1131d4d47c2d4b4d51703a1cc0dd8daddf0e86a68d492
│   └── root
└── ed271f2159e3c9de18ede980e4e2ecb0ef39b96927d446ba93deab0b6ff1a8e3
    └── root
```

每一层都包含了"该层独有的文件"以及"和其低层共享的数据的硬连接"，如

``` shell
$ ls -i /var/lib/docker/overlay/3279a41fd358cdda798d99cc2da0425b4836a489083ae9db4aedc834f426915a/root/bin/ls

405241 /var/lib/docker/overlay/3279a41fd358cdda798d99cc2da0425b4836a489083ae9db4aedc834f426915a/root/bin/ls

$ /home/hhbbz# ls -i /var/lib/docker/overlay/33629fe3999e692ecbeb340f855a2d05ddf4e173ea915041be217e1c9cc8a48f/root/bin/ls

405241 /var/lib/docker/overlay/33629fe3999e692ecbeb340f855a2d05ddf4e173ea915041be217e1c9cc8a48f/root/bin/ls
```

每一层都使用一个硬链接来指向实际ls命令的inode号(405241)。使用刚刚拉取的ubuntu镜像创建一个容器，用docker ps命令查看：

```shell
$ /home/hhbbz# docker ps

CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
8963ffb422fa        ubuntu              "/bin/bash"         32 minutes ago      Up 52 seconds                           container_1
```

创建容器时，实际上是在已有的镜像层上创建了一层容器层，容器层在路径/var/lib/docker/overlay下也存在对应的目录：

```shell

$ /home/hhbbz# ls /var/lib/docker/overlay/

3279a41fd358cdda798d99cc2da0425b4836a489083ae9db4aedc834f426915a
33629fe3999e692ecbeb340f855a2d05ddf4e173ea915041be217e1c9cc8a48f
6ab876fcc7ad584dd1eebae0d9304abb22561012f6adb87828f57906d799c33b
6abc47fcf668b6820a2a9a8b0d0c6150f9df61ab5a323783630fac3fd282144d       //创建容器后新增的目录
6abc47fcf668b6820a2a9a8b0d0c6150f9df61ab5a323783630fac3fd282144d-init  //创建容器后新增的目录
77806a4b8257cd5508a1131d4d47c2d4b4d51703a1cc0dd8daddf0e86a68d492
ed271f2159e3c9de18ede980e4e2ecb0ef39b96927d446ba93deab0b6ff1a8e3
```

“6abc47……”和“6abc47…..-init”为创建容器后新增的目录。查看这两个目录的内容:

```shell
$ /home/hhbbz# tree -L 3 /var/lib/docker/overlay/6abc47fcf668b6820a2a9a8b0d0c6150f9df61ab5a323783630fac3fd282144d*/

/var/lib/docker/overlay/6abc47fcf668b6820a2a9a8b0d0c6150f9df61ab5a323783630fac3fd282144d/
├── lower-id
├── merged
├── upper
│   ├── dev
│   │   └── console
│   ├── etc
│   │   ├── hostname
│   │   ├── hosts
│   │   ├── mtab -> /proc/mounts
│   │   └── resolv.conf
│   └── root
└── work
    └── work

/var/lib/docker/overlay/6abc47fcf668b6820a2a9a8b0d0c6150f9df61ab5a323783630fac3fd282144d-init/
├── lower-id
├── merged
├── upper
│   ├── dev
│   │   └── console
│   └── etc
│       ├── hostname
│       ├── hosts
│       ├── mtab -> /proc/mounts
│       └── resolv.conf
└── work
    └── work
```

“6abc47……”为读写层，“6abc47…..-init”为初始层。 初始层中大多是初始化容器环境时，与容器相关的环境信息， 如容器主机名，主机host信息以及域名服务文件等。所有对容器做出的改变都记录在读写层。

文件lower-id用来索引该容器使用的镜像层，upper目录包含了容器层的内容，每当启动一个容器时，会将lower-id指向的镜像层目录以及upper目录联合挂载到merged目录，因此，容器内的视角就是merged目录下的内容。而work目录则是用来完成如copy-on_write的操作。 看看容器使用到了哪一层镜像层：

```shell
$ /home/hhbbz# cat /var/lib/docker/overlay/6abc47fcf668b6820a2a9a8b0d0c6150f9df61ab5a323783630fac3fd282144d/lower-id

ed271f2159e3c9de18ede980e4e2ecb0ef39b96927d446ba93deab0b6ff1a8e3

$ /home/hhbbz# cat /var/lib/docker/overlay/6abc47fcf668b6820a2a9a8b0d0c6150f9df61ab5a323783630fac3fd282144d-init/lower-id

ed271f2159e3c9de18ede980e4e2ecb0ef39b96927d446ba93deab0b6ff1a8e3
```

在刚才创建的容器中创建一个文件：

```shell
root@8963ffb422fa:/# touch file

root@8963ffb422fa:/# echo "hello world" > file

root@8963ffb422fa:/# ls
bin   dev  file  lib    media  opt   root  sbin  sys  usr
boot  etc  home  lib64  mnt    proc  run   srv   tmp  var
```

此时再观察“6abc47……”目录(读写层)：

```shell
$ /home/hhbbz# tree -L 3 /var/lib/docker/overlay/6abc47fcf668b6820a2a9a8b0d0c6150f9df61ab5a323783630fac3fd282144d/  

/var/lib/docker/overlay/6abc47fcf668b6820a2a9a8b0d0c6150f9df61ab5a323783630fac3fd282144d/
├── lower-id
├── merged
├── upper
│   ├── dev
│   │   └── console
│   ├── etc
│   │   ├── hostname
│   │   ├── hosts
│   │   ├── mtab -> /proc/mounts
│   │   └── resolv.conf
│   ├── file
│   └── root
└── work
    └── work
```

发现upper目录下多出了一个file文件，就是刚才在容器中创建的文件。

### OverlayFS（overlay2）的镜像分层与共享

和overlay为了实现“两个目录反映多层镜像“而使用硬链接不同，overlay2驱动天生支持多层。(最多128)

因此，overlay2在使用docker层相关的命令时，能提供更好的性能(如：docker build、docker commit)。而且overlay2消耗的inode节点更少。

#### overlay2中镜像和容器的磁盘结构

docker pull ubuntu拉取完一个5层的Ubuntu镜像后，/var/lib/docker/overlay2下可以看到6个目录:

```shell
$ /home/hhbbz# tree -L 2 /var/lib/docker/overlay2

/var/lib/docker/overlay2
├── 1d18eac18e483e5af46d52673e254cb89996041d91356c6dd1f0ac1518ba130a
│   ├── diff
│   └── link   //文件
├── 28702e0cca9ff9a3245ce7e61326d098788e855ea599bf160465f537756a56a6
│   ├── diff
│   ├── link   //文件
│   ├── lower   //文件
│   ├── merged
│   └── work
├── 299397034eb96f22d544bf544c9ef993fa32571f466bd8a881aec0fc5a94a8df
│   ├── diff
│   ├── link   //文件
│   ├── lower   //文件
│   ├── merged
│   └── work
├── 8c0de7df4581df4787b08a28356cc8d7ba7b420c70dc36cc0615afdafb6ee15a
│   ├── diff
│   ├── link   //文件
│   ├── lower   //文件
│   ├── merged
│   └── work
├── f21a47541026214e519fccdf8b838ac2c4e11a1a2dd6a5febc061381a1972ad7
│   ├── diff
│   ├── link   //文件
│   ├── lower   //文件
│   ├── merged
│   └── work
└── l
    ├── 5DJFQNGXSA5CVOC6NA6HPUCXXB -> ../299397034eb96f22d544bf544c9ef993fa32571f466bd8a881aec0fc5a94a8df/diff
    ├── EGQNS3O24ONBL5BJSGNTX4NP6E -> ../1d18eac18e483e5af46d52673e254cb89996041d91356c6dd1f0ac1518ba130a/diff
    ├── JWB63ZJDZOTK5N22OMR5BKUVMG -> ../8c0de7df4581df4787b08a28356cc8d7ba7b420c70dc36cc0615afdafb6ee15a/diff
    ├── UZHRLLTPQMODQQYBLN6NOT6N2K -> ../28702e0cca9ff9a3245ce7e61326d098788e855ea599bf160465f537756a56a6/diff
    └── X76VPZDDKIW3GLWBFUHKDFBEPE -> ../f21a47541026214e519fccdf8b838ac2c4e11a1a2dd6a5febc061381a1972ad7/diff

24 directories, 9 files

```
”l“目录包含一些符号链接作为缩短的层标识符. 这些缩短的标识符用来避免挂载时超出页面大小的限制。
```
$ /home/hhbbz# ls -l /var/lib/docker/overlay2/l/
    
总用量 20
lrwxrwxrwx 1 root root 72  4月 20 17:31 5DJFQNGXSA5CVOC6NA6HPUCXXB -> ../299397034eb96f22d544bf544c9ef993fa32571f466bd8a881aec0fc5a94a8df/diff
lrwxrwxrwx 1 root root 72  4月 20 17:31 EGQNS3O24ONBL5BJSGNTX4NP6E -> ../1d18eac18e483e5af46d52673e254cb89996041d91356c6dd1f0ac1518ba130a/diff
lrwxrwxrwx 1 root root 72  4月 20 17:31 JWB63ZJDZOTK5N22OMR5BKUVMG -> ../8c0de7df4581df4787b08a28356cc8d7ba7b420c70dc36cc0615afdafb6ee15a/diff
lrwxrwxrwx 1 root root 72  4月 20 17:31 UZHRLLTPQMODQQYBLN6NOT6N2K -> ../28702e0cca9ff9a3245ce7e61326d098788e855ea599bf160465f537756a56a6/diff
lrwxrwxrwx 1 root root 72  4月 20 17:31 X76VPZDDKIW3GLWBFUHKDFBEPE -> ../f21a47541026214e519fccdf8b838ac2c4e11a1a2dd6a5febc061381a1972ad7/diff
```
最底层包含”link”文件(不包含lower文件，因为是最底层)，在上面的结果中“1d8e….”为最底层。 这个文件记录着作为标识符的更短的符号链接的名字、最底层还有一个”diff”目录(包含实际内容)。
```
$ /home/hhbbz# cat /var/lib/docker/overlay2/1d18eac18e483e5af46d52673e254cb89996041d91356c6dd1f0ac1518ba130a/link 
    
EGQNS3O24ONBL5BJSGNTX4NP6E
    
root@hhbbz-MS-7823:/home/hhbbz# ls /var/lib/docker/overlay2/1d18eac18e483e5af46d52673e254cb89996041d91356c6dd1f0ac1518ba130a/diff/
    
bin   dev  home  lib64  mnt  proc  run   srv  tmp  var
boot  etc  lib   media  opt  root  sbin  sys  usr
```
从第二层开始，每层镜像层包含”lower“文件，根据这个文件可以索引构建出整个镜像的层次结构。 ”diff“文件(层的内容)。还包含”merged“和”work”目录，用途和overlay一样。
```
$ /home/hhbbz# cat /var/lib/docker/overlay2/28702e0cca9ff9a3245ce7e61326d098788e855ea599bf160465f537756a56a6/link 
    
UZHRLLTPQMODQQYBLN6NOT6N2K
    
$ /home/hhbbz# ls /var/lib/docker/overlay2/28702e0cca9ff9a3245ce7e61326d098788e855ea599bf160465f537756a56a6/diff/
    
etc  sbin  usr  var
    
$ /home/hhbbz# cat /var/lib/docker/overlay2/28702e0cca9ff9a3245ce7e61326d098788e855ea599bf160465f537756a56a6/lower
    
l/EGQNS3O24ONBL5BJSGNTX4NP6E
```
再来看看容器层，容器层的文件构成和镜像层类似(这点和overlay不同)，使用刚刚拉取的ubuntu镜像创建一个容器，/var/lib/docker/overlay2目录下新增2个目录：
```
├── 2558ed8dd7051c1e78ea980590a228100ec3d299c01b35e9b7fa503723593d19
│   ├── diff
│   ├── link
│   ├── lower
│   ├── merged
│   └── work
├── 2558ed8dd7051c1e78ea980590a228100ec3d299c01b35e9b7fa503723593d19-init
│   ├── diff
│   ├── link
│   ├── lower
│   ├── merged
│   └── work
└── l
    ├── 7H7MUMX4IOAVLIJ2YPLR5MZOO5 -> ../2558ed8dd7051c1e78ea980590a228100ec3d299c01b35e9b7fa503723593d19-init/diff
    ├── EBSBKJBLA2WYBHTMHKSMCEEWNP -> ../2558ed8dd7051c1e78ea980590a228100ec3d299c01b35e9b7fa503723593d19/diff

$ /home/hhbbz# cat /var/lib/docker/overlay2/2558ed8dd7051c1e78ea980590a228100ec3d299c01b35e9b7fa503723593d19/lower 
    
l/7H7MUMX4IOAVLIJ2YPLR5MZOO5:l/5DJFQNGXSA5CVOC6NA6HPUCXXB:l/JWB63ZJDZOTK5N22OMR5BKUVMG:l/X76VPZDDKIW3GLWBFUHKDFBEPE:l/UZHRLLTPQMODQQYBLN6NOT6N2K:l/EGQNS3O24ONBL5BJSGNTX4NP6E
```
### 容器的读写
对于读，考虑下列3种场景：

- 读的文件不在容器层：如果读的文件不在容器层，则从镜像层进行读
- 读的文件只存在在容器层：直接从容器层读
- 读的文件在容器层和镜像层：读容器层中的文件，因为容器层隐藏了镜像层同名的文件

对于写，考虑下列场景：

- 写的文件不在容器层，在镜像层：由于文件不在容器层，因此overlay/overlay2存储驱动使用copy_up操作从镜像层拷贝文件到容器层，然后将写入的内容写入到文件新的拷贝中。
- 删除文件和目录：删除镜像层的文件，会在容器层创建一个whiteout文件来隐藏它；删除镜像层的目录，会创建opaque目录，它和whiteout文件有相同的效果
- 重命名目录：对一个目录调用rename(2)仅仅在资源和目的地路径都在顶层时才被允许，否则返回EXDEV。

## 问题解决
　　最后通过在/var/lib/docker/overlay2目录下执行
```du -sh *```
找到占用磁盘空间最大的几个文件夹，进行审查清理。

