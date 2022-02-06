上一篇文章我们对余老师开发的分布式存储系统FastCFS做了一个简单的介绍，具体看这个链接：https://www.jianshu.com/p/f09953393b1e

今天我讲自己部署FastCFS集群并和k8s集群打通的全过程分享出来，希望能够帮助到希望尝试使用FastCFS的同学。

# 一、快速部署
如果你只是想简单快速上手体验 FastCFS，作为学习或者本地测试环境而非生产环境，你可以选择以下两种方式部署 FastCFS：

*   部署本地单节点
*   Docker部署

## 1. 单机部署
单机部署FastCFS的github和gitee上面已经提供了比较完整的文档说明。我这里摘抄一些，并做下说明：
>- 前提是安装的机器必须已经安装git客户端。
>- 仅支持Centos7、Centos8两个Linux发行版本，笔者在Centos7环境亲测成功。

一键搭建(包括部署和运行)单节点（需要root身份执行）：

```
git clone https://gitee.com/fastdfs100/FastCFS.git; 
cd FastCFS/
./helloWorld.sh

# 注意：helloWorld.sh将更改FastCFS相关配置文件，请不要在多节点集群上执行！

```

上述操作完成后，执行命令验证安装状态：

```
df -h /opt/fastcfs/fuse | grep fuse

```

如果可以看到FastCFS挂载的文件目录，说明安装成功，你可以把/opt/fastcfs/fuse 当作本地文件系统访问该目录。

如果没有安装git客户端，也没有问题，只需要下载两个sh文件并放到同一个文件夹即可，它们是:
[helloWorld.sh](https://toscode.gitee.com/fastdfs100/FastCFS/raw/master/helloWorld.sh)
[fastcfs.sh](https://toscode.gitee.com/fastdfs100/FastCFS/raw/master/fastcfs.sh)（值得注意的是fastcfs.sh支持源码编译安装，但是个人觉得并不适合一键部署）

安装命令如下（有git客户端且已经按上一步安装成功的可以略过）
```
mkdir fastcfs
cd fashcfs
wget https://toscode.gitee.com/fastdfs100/FastCFS/raw/master/helloWorld.sh
wget https://toscode.gitee.com/fastdfs100/FastCFS/raw/master/fastcfs.sh
chmod +x helloWorld.sh
chmod +x fastcfs.sh
./helloWorld.sh


```

一键部署的详细说明，请参见这里[一键部署详细说明](https://github.com/happyfish100/FastCFS/blob/master/docs/Easy-install-detail-zh_CN.md)


## 2. Docker部署
官方暂时没有提供Docker镜像，为了方便大家快速体验，笔者制作了一个单机版本的镜像，已经push到Docker Hub，地址为：https://hub.docker.com/r/jerry6290/fastcfs
##### 启动方式：

```
docker run --name=fastCFS  --privileged -d jerry6290/fastcfs:v3.1.0
```

v3.1.0是版本号，可以根据实际情况修改成最新版本号。

注意：由于需要通过fuse把FastCFS作为目录挂载，所以在docker run时需要增加参数 **--privileged**，让容器真正有root权限。

##### 登录到容器验证fastCFS

  ```
docker exec -it fastCFS sh

# 执行 df -h ，应该能看到 fastCFS的fs pool 被挂载到 /opt/fastcfs/fuse 目录
df -h
Filesystem               Size  Used Avail Use% Mounted on
overlay                  500G  110G  391G  22% /
tmpfs                     64M     0   64M   0% /dev
tmpfs                     12G     0   12G   0% /sys/fs/cgroup
shm                       64M     0   64M   0% /dev/shm
/dev/mapper/centos-root  500G  110G  391G  22% /etc/hosts
/dev/fuse                342G     0  342G   0% /opt/fastcfs/fuse

# 写文件
echo 'Hello FastCFS' > /opt/fastcfs/fuse/FastCFS.txt

# 查看文件内容
cat /opt/fastcfs/fuse/FastCFS.txt
Hello FastCFS

```
当然，你也可以通过-v的方式把/opt/fastcfs/fuse目录挂载出来，具体做法就不赘述了。

该Dockerfile开源在github，并通过github Actions自动打包并push到docker hub，项目地址：https://github.com/jerry6290/dockerImage-fastCFS-Server



# 二、集群部署
## 1. 软硬件环境准备
FastCFS作为一款开源云原生分布式存储系统，可以很好的部署和运行在 Intel 架构服务器环境和主流虚拟化环境，并支持绝大多数的主流硬件和网络， 支持主流的 Linux 操作系统环境。

 ### 1.1 操作系统发行版本要求：
| Linux 操作系统平台 | 版本 |
| -----------                   | ----------- |
|Red Hat Enterprise Linux    |7.x及以上版本 版本 |
|CentOS	                             | 7.x 及以上版本版本 |
|Ubuntu LTS	                     |16.04 及以上的版本|

### 1.2 服务器建议配置
-------
FastCFS有三大组件，FastDIR，FastStore，FastAuth，支持部署和运行在 Intel x86-64 架构的 64 位通用硬件服务器平台。

对于开发，测试，及生产环境的服务器硬件配置有以下要求和建议：

#### 开发及测试环境：
| 组件       | CPU      | 内存     |  硬盘     | 网络 | 实例数量 |
| ----------- | -----------|-----------| -----------|----------| ----------- |
|  FastDIR| 4核+    | 8G+  | 无特殊要求 | 千兆网卡 | 1（可和FastStore，FastAuth同机器） |
|  FastStore| 4核+    | 8G+  | 无特殊要求，最好是SSD，容量大一些 | 千兆网卡 | 1（可和FastDIR，FastAuth同机器） |
|  FastAuth| 4核+    | 8G+  | 无特殊要求 | 千兆网卡 | 1（可和FastStore，FastDIR同机器） |

#### 生产环境：
| 组件       | CPU         | 内存     |  硬盘     | 网络 | 实例数量 |
| ----------- | -------------|-----------| -----------|----------| ----------- |
|  FastDIR| 8核+    | 16G+  | 无特殊要求 | 千兆网卡 | 3及以上，最好是奇数（可和FastStore，FastAuth同机器） |
|  FastStore| 8核+    | 16G+  | 最好是SSD，容量大一些 | 千兆网卡 | 6及以上，可根据需求的容量而定（可和FastDIR，FastAuth同机器） |
|  FastAuth| 8核+    | 16G+  | 无特殊要求 | 千兆网卡 | 3及以上，最好是奇数（可和FastStore，FastDIR同机器） |

> 注意：
> - FastStore对服务器和数据均采用分组方式，服务器分组简称 SG，组内的数据是冗余关系（服务器数即数据副本数）。一个SG可以容纳多个数据分组DG，引入DG的主要目的是方便扩容时做数据迁移，因此最好预设得大一些，生产环境至少配置 256，开发测试环境至少配置16个。
> - 生产环境建议建2个以上SG，每个SG有3台服务器，即3个数据副本，所以存储数据的组件FastStore建议至少2*3=6台服务器，以保证数据完整性。
> - FastAuth是可选，如果需要通过CSI集成到k8s，则需要开启存储池或访问权限控制，需要部署FastAuth认证集群。
> - FastDIR启用存储插件的话，最好配置SSD。
> - 如果对性能和可靠性有更高的要求，fastDIR，fastStore，fastAuth三大组件应尽可能分开部署。

## 2. 环境及配置准备
### 2.1 SSH免密登录
找一个服务器当中控机，比如：192.168.0.201，以root用户登录到中控机，执行以下命令。将 192.168.0.204 替换成你的受控机器 IP，按提示输入受控机器root用户密码，执行成功后即创建好 SSH 互信，其他机器同理。
```
ssh-keygen -t rsa
# 一路回车
ssh-copy-id 192.168.0.204
```
验证ssh免密是否成功，在中控机，通过 ssh 的方式登录受控机器 IP。如果不需要输入密码并登录成功，即表示 SSH 互信配置成功，如果没有成功请检查受控机器的sshd配置和相关安全策略。
```
ssh 192.168.0.204
```
### 2.2 端口准备
- fastDIR
  - 默认集群端口 11011
  - 默认服务端口 11012
- fastAuth
  - 默认集群端口 31011
  - 默认服务端口 31012
- fastStore
  - 默认集群端口 21014
  - 默认副本端口 21015
  - 默认服务端口 21016

需要保证每台服务器上述的端口是相互能通的，如果是在redhat7、centos7版本及以上版本，可以通过firewall-cmd命令打开服务器之间端口通信。比如在网段192.168.0.1/24，可以通过一下命令打开端口，其中网段和--zone=public需要自行根据自己的zone进行调整：
```
firewall-cmd --permanent --zone=public --add-rich-rule='rule family="ipv4" source address="192.168.0.1/24" port protocol=tcp port=11011-11012 accept'
firewall-cmd --permanent --zone=public --add-rich-rule='rule family="ipv4" source address="192.168.0.1/24" port protocol=tcp port=21014-21016 accept'
firewall-cmd --permanent --zone=public --add-rich-rule='rule family="ipv4" source address="192.168.0.1/24" port protocol=tcp port=31011-31012 accept'
firewall-cmd --reload
```

## 3. 集群拓扑规划 
FastCFS支持大规模的集群，下面以最为典型的最小化集群拓扑，1个SG，3个DG（3个数据副本）。更大规模的安装方式可参照此过程扩展安装，服务器的数量及配置参数按上文所述。

本次安装教程，笔者手头上的机器有限，准备了4台机器，已经安装好centos7.9，对应的端口防火墙已经打开,，集群拓扑规划如下：
- 3个FastStore节点、3个FastDir节点、3个FastAuth节点，一个fuse客户端节点
- FastStore、FastDir、FastAuth公用3个节点。

| 组件 | 服务器IP | 个数|
| -----------                   | ----------- | ----------- |
|FastDir    | 192.168.0.201，192.168.0.204，192.168.0.205| 3|
|FastStore | 192.168.0.201，192.168.0.204，192.168.0.205| 3|
|FastAuth  |192.168.0.201，192.168.0.204，192.168.0.205| 3|
|Fast-fuse客户端  |192.168.0.203| 1|

## 4. 集群安装与配置
### 4.1 yum方式安装

#### 4.1.1 FastOS.repo yum源
需要在每个节点安装，命令：
```
rpm -ivh http://www.fastken.com/yumrepo/el7/x86_64/FastOSrepo-1.0.0-1.el7.centos.x86_64.rpm
```
#### 4.1.2 安装FastDIR
分别在192.168.0.201,204,205安装
```
yum install fastDIR-server -y
```
安装完毕后，在/etc/fastcfs 下可看到FastDIR的配置文件，在/usr/bin/目录看到fdir_*相关的程序
```
$ ll /etc/fastcfs/fdir/
total 16
-rw-r--r--. 1 root root  134 Jan 20 23:24 client.conf
-rw-r--r--. 1 root root  291 Jan 20 23:24 cluster.conf
-rw-r--r--. 1 root root 2160 Jan 20 23:24 server.conf
-rw-r--r--. 1 root root  726 Jan 20 23:24 storage.conf


ll /usr/bin/ |grep fdir_*
-rwxr-xr-x.   1 root root      11336 Jan 20 20:51 fdir_cluster_stat
-rwxr-xr-x.   1 root root      11472 Jan 20 20:51 fdir_getxattr
-rwxr-xr-x.   1 root root      11344 Jan 20 20:51 fdir_list
-rwxr-xr-x.   1 root root       7200 Jan 20 20:51 fdir_list_servers
-rwxr-xr-x.   1 root root      11344 Jan 20 20:51 fdir_mkdir
-rwxr-xr-x.   1 root root      11312 Jan 20 20:51 fdir_remove
-rwxr-xr-x.   1 root root      11312 Jan 20 20:51 fdir_rename
-rwxr-xr-x.   1 root root     234208 Jan 20 20:51 fdir_serverd
-rwxr-xr-x.   1 root root      11384 Jan 20 20:51 fdir_service_stat
-rwxr-xr-x.   1 root root      11328 Jan 20 20:51 fdir_setxattr
-rwxr-xr-x.   1 root root      11328 Jan 20 20:51 fdir_stat
```


#### 4.1.2 安装FastStore
分别在192.168.0.201,204,205安装
```
yum install faststore-server -y
```
安装完毕后，在/etc/fastcfs 下可看到FastStore的配置文件，在/usr/bin/目录看到fs_*相关的程序
```
ll /etc/fastcfs/fstore/
total 16
-rw-r--r--. 1 root root  147 Jan 20 23:24 client.conf
-rw-r--r--. 1 root root 1739 Jan 20 23:24 cluster.conf
-rw-r--r--. 1 root root 1274 Jan 20 23:24 server.conf
-rw-r--r--. 1 root root  673 Jan 20 23:24 storage.conf



ll /usr/bin/ |egrep " fs_"
-rwxr-xr-x.   1 root root      11488 Jan 13 10:32 fs_cluster_stat
-rwxr-xr-x.   1 root root      11440 Jan 13 10:32 fs_delete
-rwxr-xr-x.   1 root root      11480 Jan 13 10:32 fs_read
-rwxr-xr-x.   1 root root     447336 Jan 13 10:32 fs_serverd
-rwxr-xr-x.   1 root root      11424 Jan 13 10:32 fs_service_stat
-rwxr-xr-x.   1 root root      11456 Jan 13 10:32 fs_write

```
#### 4.1.3 安装FastAuth
分别在192.168.0.201,204,205安装
```
yum install FastCFS-auth-server -y
```
安装完毕后，在/etc/fastcfs 下可看到FastAuth的配置文件
```
ll /etc/fastcfs/auth/
total 20
-rw-r--r--. 1 root root  411 Jan 21 00:47 auth.conf
-rw-r--r--. 1 root root  134 Jan 21 00:47 client.conf
-rw-r--r--. 1 root root  148 Jan 21 19:18 cluster.conf
drwxr-xr-x. 2 root root   51 Jan 21 15:47 keys
-rw-r--r--. 1 root root 1627 Jan 21 00:47 server.conf
-rw-r--r--. 1 root root  145 Jan 21 00:47 session.conf

```
#### 4.1.3 安装Fast-fused客户端
客户端只需要在192.168.0.203安装
```
yum remove fuse -y
yum install FastCFS-fused -y
```
> ***说明：***
>   * centos版本中的fuse为老版本的包（fuse2.x），需要卸载才可以成功安装FastCFS-fused依赖的fuse3；
 >  * 第一次安装才需要卸载fuse包，以后就不用执行了。

安装完毕后，在/etc/fastcfs 下可看到fcfs的配置文件
```
 ll /etc/fastcfs/
total 0
drwxr-xr-x 3 root root 113 Feb  3 14:38 auth
drwxr-xr-x 2 root root  23 Feb  3 14:38 fcfs
drwxr-xr-x 2 root root  84 Feb  3 14:38 fdir
drwxr-xr-x 2 root root  84 Feb  3 14:38 fstore
```

#### 4.1.4 集群配置
FastCFS并没有统一的配置中心，需要在各个节点上单独部署配置文件。配置文件分为三大类：集群配置文件、服务配置文件、客户端配置文件。

1.  **集群配置文件：** 指的是描述 FastDir 、 FastStore、FastAuth 的配置文件，入口文件名称为cluster.conf . 该配置文件中设定是集群的参数，如服务节点的IP、服务端口号、集群同步端口号，服务节点的拓扑结构等。 cluster.conf 文件全局统一，各个节点上的内容是相同的。

2.  **服务配置文件：** 指的是服务本身的配置文件，入口文件名称server.conf 如线程数量、链接数量、缓冲区大小、存储配置、日志配置等。 服务配置文件的内容，可以全局不统一。不过从集群运维方便的角度考虑，服务器配置最好是统一的。

3.  **客户端配置文件：** 指的是fuse或者其他客户端的配置，比如fuse客户端需要知道fdir、fstore、fauth的集群情况，所以客户端需要fdir、fstore和fauth的server服务集群的配置。

下面针对三大组件和客户端的配置进行详细说明：
##### fdir 配置
用root登录主控机192.168.0.201
- 修改fdir集群配置文件
修改/etc/fastcfs/fdir/cluster.conf 文件，修改为上面提到的三个IP地址（修改成你自己对应的IP）,[sever-1]为192.168.0.201，[sever-2]为192.168.0.204，[sever-3]为192.168.0.205，如果你有更多fdir节点，增加配置[server-N]即可，修改后的内容如下：
```
# config the auth config filename
auth_config_filename = ../auth/auth.conf

[group-cluster]
# the default cluster port
port = 11011

[group-service]
# the default service port
port = 11012

[server-1]
host = 192.168.0.201 #节点1

[server-2]
host = 192.168.0.204 #节点2

[server-3]
host = 192.168.0.205 #节点3
```
- 修改fdir服务配置文件
修改/etc/fastcfs/fdir/server.conf 文件，内容如下：
```
# the base path to store log files
# this path must be exist
base_path = /opt/fastcfs/fdir

# the path to store data files
# can be an absolute path or a relative path
# the relative path for sub directory under the base_path
# this path will be created auto when not exist
# default value is data
data_path = data

# max concurrent connections this server support
# you should set this parameter larger, eg. 10240
# default value is 256
max_connections = 10240

# the data thread count
# these threads deal CUD (Create, Update, Delete) operations
# dispatched by the hash code of the namespace
# if you have only one namespace, you should config this parameter to 1,
# because it is meaningless to configure this parameter greater than 1 in this case
# default value is 1
data_threads = 1


# the cluster id for generate inode
# must be natural number such as 1, 2, 3, ...
#
## IMPORTANT NOTE: do NOT change the cluster id after set because the 64 bits
##                 inode includes the cluster id, and the inode disorder maybe
##                 lead to confusion
cluster_id = 1

# config cluster servers
cluster_config_filename = cluster.conf

# session config filename for auth
session_config_filename = ../auth/session.conf


[storage-engine]
# if enable the storage engine
### false: use binlog directly
### true: use storage engine for massive files
# default value is false，如果设置为true，还需要配置storage.conf
enabled = false

# the config filename for storage
storage_config_filename = storage.conf

# the path to store the data files
# can be an absolute path or a relative path
# the relative path for sub directory under the base_path
# this path will be created auto when not exist
# default value is db
data_path = db

# the interval for lru elimination
# <= 0 for never eliminate
# unit: seconds
# default value is 1
eliminate_interval = 1

# the memory limit ratio for dentry
# the valid limit range is [1%, 99%]
# default value is 80%
memory_limit = 80%


[cluster]
# the listen port
port = 11011

# the network thread count
# these threads deal network io
# dispatched by the incoming socket fd
# default value is 4
work_threads = 2

[service]
port = 11012
work_threads = 4
```
server.conf基本上用默认配置即可，如果需要开启存储插件，需要把[storage-engine]下面的enable=true，同时配置storage.conf。

- 复制cluster文件到其他节点
fdir的cluster.conf,server.conf配置完成后，把cluster.conf配置文件通过scp命令复制到其他节点（包括客户端节点），server.conf不需要复制：
```
scp /etc/fastcfs/fdir/cluster.conf 192.168.0.204:/etc/fastcfs/fdir/cluster.conf
scp /etc/fastcfs/fdir/cluster.conf 192.168.0.205:/etc/fastcfs/fdir/cluster.conf
scp /etc/fastcfs/fdir/cluster.conf 192.168.0.203:/etc/fastcfs/fdir/cluster.conf
```



##### fstore 配置
用root登录主控机192.168.0.201
- 修改fstore集群配置文件
> ### 非常重要：
> fstore是存储数据的核心组件，修改cluster配置文件，一定要了解fstore存储的基本原理。了解SG（服务器分组），DG（数据分组），DGC（数据分组数）等几个名词的相互关系。上面的拓扑规划时已经简单描述过，更详细查看作者余大的技术文章：https://my.oschina.net/u/3334339/blog/4870261

修改/etc/fastcfs/fstore/cluster.conf 文件，让fstore的集群拓扑为：1个SG，SG里面包含三台服务器，DG为2，DGC为128，修改后的内容如下：
```
# the group count of the servers / instances
server_group_count = 1 # SGC=1

# all data groups must be mapped to the server group(s) without omission.
# once the number of data groups is set, it can NOT be changed, otherwise
# the data access will be confused!
data_group_count = 128 #DGC

# config the auth config filename
auth_config_filename = ../auth/auth.conf

[group-cluster]
# the default cluster port
port = 21014

[group-replica]
# the default replica port
port = 21015

[group-service]
# the default service port
port = 21016

[server-group-1]
server_ids = [1, 3]
data_group_ids = [1, 64]
data_group_ids = [65, 128]

[server-1]
host = 192.168.0.201

[server-2]
host = 192.168.0.204

[server-3]
host = 192.168.0.205

```
server.conf基本上用默认配置即可，如果需要修改存储目录，需要修改storage.conf，如果有多个数据盘可以配置多个目录，充分利用硬盘空间
```
# the write thread count per store path
# the default value is 1
write_threads_per_path = 1

# the read thread count per store path
# the default value is 1，如果有多个盘可以配置多个目录，充分利用硬盘空间
read_threads_per_path = 1

# usually one store path for one disk
# each store path is configurated in the section as: [store-path-$id],
# eg. [store-path-1] for the first store path, [store-path-2] for
#     the second store path, and so on.
store_path_count = 1

# reserved space of each disk for system or other applications.
# the value format is XX%
# the default value is 10%
reserved_space_per_disk = 10%


#### store paths config #####

[store-path-1]
# the path to store the file，如果有多个盘可以配置多个目录，充分利用硬盘空间
path = /opt/faststore/data

```
- 复制cluster文件到其他节点
fstore的cluster.conf,server.conf,storage.conf配置完成后，把cluster.conf配置文件通过scp命令复制到其他节点，server.conf不需要复制：
```
scp /etc/fastcfs/fstore/cluster.conf 192.168.0.204:/etc/fastcfs/fstore/cluster.conf
scp /etc/fastcfs/fstore/cluster.conf 192.168.0.205:/etc/fastcfs/fstore/cluster.conf
```

##### fauth 配置
用root登录主控机192.168.0.201
- 修改fauth集群配置文件
修改/etc/fastcfs/auth/cluster.conf 文件，修改为上面提到的三个IP地址（修改成你自己对应的IP）,[sever-1]为192.168.0.201，[sever-2]为192.168.0.204，[sever-3]为192.168.0.205，如果你有更多fauth节点，增加配置[server-N]即可，修改后的内容如下：
```
[group-cluster]
# the default cluster port
port = 31011

[group-service]
# the default service port
port = 31012

[server-1]
host = 192.168.0.201

[server-2]
host = 192.168.0.204

[server-3]
host = 192.168.0.205
```
- 修改auth.conf开启认证
把 /etc/fastcfs/auth/auth.conf 文件里面的auth_enabled = true ，修改后内容：
```
# enable / disable authentication
# default value is false
auth_enabled = true

# the username for login
# default value is admin
username = admin

# the secret key filename of the user
# variable ${username} will be replaced with the value of username
# default value is keys/${username}.key
secret_key_filename = keys/${username}.key

# the config filename of auth client
client_config_filename = client.conf
```
- 复制cluster文件到其他节点
fauth的cluster.conf,auth.conf配置完成后，把cluster.conf、auth.conf配置文件通过scp命令复制到其他节点，server.conf不需要复制：
```
scp /etc/fastcfs/auth/cluster.conf 192.168.0.204:/etc/fastcfs/auth/cluster.conf
scp /etc/fastcfs/auth/cluster.conf 192.168.0.205:/etc/fastcfs/auth/cluster.conf
scp /etc/fastcfs/auth/cluster.conf 192.168.0.203:/etc/fastcfs/auth/cluster.conf
scp /etc/fastcfs/auth/auth.conf 192.168.0.204:/etc/fastcfs/auth/auth.conf
scp /etc/fastcfs/auth/auth.conf 192.168.0.205:/etc/fastcfs/auth/auth.conf
scp /etc/fastcfs/auth/auth.conf 192.168.0.203:/etc/fastcfs/auth/auth.conf
```


#####  集群启动
集群的配置文件在各个节点已经配置和分发完毕，下面可以开始启动集群了。
启动顺序如下：
1. fdir
2. fauth
3. fstore
用root用户分别登录到3个节点，执行如下命令：
```
systemctl restart fastdir
systemctl restart fastauth
systemctl restart faststore
```
如果启动有问题，请检查配置文件，具体启动日志可以查看/opt/fastcfs/下对应的auth、fdir、fstore三个目录里面的logs目录。


### 客户端启动
如果三个节点的所有组件启动没有错误，在客户端节点可以启动客户端程序，把fastCFS的默认pool fs挂载到相应目录。
以本次部署为例，root用户登录到客户端节点192.168.0.203，执行命令：
```
systemctl restart fastcfs
```
查看日志文件/opt/fastcfs/fcfs/logs/fcfs_fused.log看下启动是否有误，如果无误可以通过df -h查看挂载的目录 /opt/fastcfs/fuse
```
df -h
Filesystem               Size  Used Avail Use% Mounted on
overlay                  500G  110G  391G  22% /
tmpfs                     64M     0   64M   0% /dev
tmpfs                     12G     0   12G   0% /sys/fs/cgroup
shm                       64M     0   64M   0% /dev/shm
/dev/mapper/centos-root  500G  110G  391G  22% /etc/hosts
/dev/fuse                342G     0  342G   0% /opt/fastcfs/fuse
```

### 4.2 部署工具fastcfs.sh安装
运维工具fastcfs.sh方式的安装，请参考官方文档，这里不赘述：https://github.com/happyfish100/FastCFS/blob/master/docs/fcfs-ops-tool-zh_CN.md
### 4.3 Ansible方式安装（推荐、未完成）
### 4.4 K8S Operator方式安装（未完成）

## 5. 验证集群状态
### 5.1 fdir集群状态
查询fdir整个集群状态
```
fdir_cluster_stat
# 输出如下
server_id: 1, host: 192.168.0.201:11012, status: 23 (ACTIVE), is_master: 0
server_id: 2, host: 192.168.0.204:11012, status: 23 (ACTIVE), is_master: 0
server_id: 3, host: 192.168.0.205:11012, status: 23 (ACTIVE), is_master: 1

server count: 3

```
查询fdir某个节点状态，1表示是server编号，和cluster.conf里面的server-N相对应
```
fdir_service_stat 1

# 输出如下
        server_id: 1
        host: 192.168.0.201:11012
        status: 23 (ACTIVE)
        is_master: false
        connection : {current: 4, max: 4}
        binlog : {current_version: 48}
        dentry : {current_inode_sn: 5000020, ns_count: 2, dir_count: 10, file_count: 4}
```
### 5.2 fstore集群状态
显示所有状态
```
fs_cluster_stat
# 输出如下：
data_group_id: 1
        server_id: 1, host: 192.168.0.201:21016, status: 5 (ACTIVE), is_preseted: 0, is_master: 0, data_version: 0
        server_id: 2, host: 192.168.0.204:21016, status: 5 (ACTIVE), is_preseted: 1, is_master: 1, data_version: 0
        server_id: 3, host: 192.168.0.205:21016, status: 5 (ACTIVE), is_preseted: 0, is_master: 0, data_version: 0
.... 省略N个
data_group_id: 127
        server_id: 1, host: 192.168.0.201:21016, status: 5 (ACTIVE), is_preseted: 0, is_master: 0, data_version: 0
        server_id: 2, host: 192.168.0.204:21016, status: 5 (ACTIVE), is_preseted: 1, is_master: 1, data_version: 0
        server_id: 3, host: 192.168.0.205:21016, status: 5 (ACTIVE), is_preseted: 0, is_master: 0, data_version: 0

data_group_id: 128
        server_id: 1, host: 192.168.0.201:21016, status: 5 (ACTIVE), is_preseted: 0, is_master: 0, data_version: 0
        server_id: 2, host: 192.168.0.204:21016, status: 5 (ACTIVE), is_preseted: 0, is_master: 0, data_version: 0
        server_id: 3, host: 192.168.0.205:21016, status: 5 (ACTIVE), is_preseted: 1, is_master: 1, data_version: 0

data server count: 384
```


显示某个sg状态，比如sg=1
```
fs_cluster_stat -g 1

#输出如下：
data_group_id: 1
        server_id: 1, host: 192.168.0.201:21016, status: 5 (ACTIVE), is_preseted: 1, is_master: 1, data_version: 0
        server_id: 2, host: 192.168.0.204:21016, status: 5 (ACTIVE), is_preseted: 0, is_master: 0, data_version: 0
        server_id: 3, host: 192.168.0.205:21016, status: 5 (ACTIVE), is_preseted: 0, is_master: 0, data_version: 0

data server count: 3
```
更多查看
```
fs_cluster_stat -help
```
### 5.1 fauth集群状态
```
fauth_cluster_stat

#输出如下：
server_id: 1, host: 192.168.0.201:31012, is_online: 1, is_master: 0
server_id: 2, host: 192.168.0.204:31012, is_online: 1, is_master: 0
server_id: 3, host: 192.168.0.205:31012, is_online: 1, is_master: 1

server count: 3

```
## 6. 集群管理
### 6.1 fdir相关操作
fdir有fdir_list、fdir_mkdir、fdir_rename、fdir_remove、fdir_getxattr、fdir_setxattr等，还有fcfs_pool管理、查看pool相关
比如查看pool fs，可以通过fdir_list，更多的信息查看用-help 查看
```
fdir_list -n fs /
```
查看pool列表
```
fcfs_pool plist
```
### 6.2 fstore相关操作
fs_read,fs_write,fs_delete等，相关命令还不熟，研究中...
### 6.3 fauth相关操作
fcfs_user 查看，新增，删除，设置用户权限等
查看用户列表
```
fcfs_user list
```
更多操作
```
Usage: fcfs_user [-c config_filename=/etc/fastcfs/auth/client.conf]
        [-u admin_username=admin]
        [-k admin_secret_key_filename=/etc/fastcfs/auth/keys/${username}.key]
        [-p priviledges=pool]
        <operation> [username]
        [user_secret_key_filename=keys/${username}.key]

        the operations and parameters are:
          create <username> [user_secret_key_filename]
          passwd | secret-key <username> [user_secret_key_filename] [-y]: regenerate user's secret key
          grant <username>, the option <-p priviledges> is required
          delete | remove <username>
          list [username]

        [user_secret_key_filename]: specify the filename to store the generated secret key of the user
        [priviledges]: the granted priviledges seperate by comma, priviledges:
          user: user management
          pool: create storage pool
          cluster: monitor cluster
          session: subscribe session for FastDIR and FastStore server side
          *: for all priviledges

```
## 7. 测试集群性能

更多性能测试查看官方测试结果：https://github.com/happyfish100/FastCFS/blob/master/docs/benchmark.md

## 8. Kubernetes CSI安装与配置
终于到了对k8s的支持，作为云原生分布式存储，对k8s的支持肯定是少不了的。
### 8.1 用户和pool准备
CSIDriver必须要求FastCFS 启用验证模块auth_enabled = true，因为k8s的CSI要求支持卷支持定义容量，不同卷是相互独立的。pool就是为此设计的，在CSI Driver中一个卷就是一个pool，而pool是属于用户进行管理的。

为CSI单独创建一个用户：k8s，当然可以用admin用户，但是不推荐。
```
fcfs_user create k8s
create user k8s success, secret key store to file: keys/k8s.key
```
新建k8s用户成功后会在当前目录下keys目录生成k8s.key文件，这个文件的内容后面会用到。
查看用户：
```
fcfs_user list
  No.                         username                      priviledges
   1.                            admin                                *
   2.                           admin1                                *
   3.                              k8s                             pool

```

### 8.2 配置准备
FastCFS CSI是利用fused客户端来实现的卷的创建、挂载等操作。上面已经说到FastCFS没有统一的配置中心，而客户端又需要FastCFS集群相关的配置信息，所以需要把集群的配置让CSI读取到。
CSI现在实现的方式是通过http uri来读取，因此需要把配置文件通过web服务器暴露处理，可以通过nginx、apache等web服务器。
> 现在CSI需要依赖一个web服务器，其实可以通过configMap挂载文件的方式来实现，已经给官方提供建议并被采纳，不依赖uri的新CSI版本应该很快和大家见面。

笔者使用的是nginx，把192.168.0.201的/etc/fastcfs目录通过nginx暴露出来，具体细节请参考nginx文档，比如我这里的地址是：http://192.168.0.201:1808/
- ConfigMap准备
创建fastcfs-csi-cm.yml文件，并填入以下内容，其中configURL需要修改成你自己的可以获取fastcfs配置的web服务器地址：
```
apiVersion: v1
kind: ConfigMap
data:
  config.json: |-
    [
      {
        "clusterID": "virtual-cluster-id-1",
        "configURL": "http://192.168.0.201:1808/"
      }
    ]
metadata:
  name: fcfs-csi-config
```
执行 
```
kubectl apply -f fastcfs-csi-cm.yml
```

- Secret准备
Secret用来保存刚才创建的用户的用户名和密钥，还是以k8s为例，查看keys/k8s.key文件
```
cat keys/k8s.key
407f1379637afba188d31a795d0224e8
```
创建fastcfs-csi-secret.yml,并填入内容，407f1379637afba188d31a795d0224e8 改成你自己的密钥。
```
---
apiVersion: v1
kind: Secret
metadata:
  name: csi-fcfs-secret
  namespace: default
stringData:
  # 其实这里并需要admin用户，先填k8s用户和密钥
  adminName: k8s
  adminSecretKey: 407f1379637afba188d31a795d0224e8 
  # use static pv for user
  userName: k8s
  userSecretKey: 407f1379637afba188d31a795d0224e8 
```
执行 
```
kubectl apply -f fastcfs-csi-secret.yml
```
### 8.3 Helm3准备
添加 fastcfs-csi Helm 存储库：
```
helm repo add fastcfs-csi https://happyfish100.github.io/fastcfs-csi
helm repo update
```
### 8.4 安装FastCFS CSI
- 使用helm chart 安装驱动程序的版本

> 注意helm中容器用到了 k8s.gcr.io 下面的image，如果没有条件，docker hub上面已经有人做了镜像，可以按下面的步骤修改：
```
helm pull fastcfs-csi/fcfs-csi-driver
tar -xzvf fcfs-csi-driver-0.3.0.tgz
vi fcfs-csi-driver/values.yaml
```
把里面相关 k8s.gcr.io/sig-storage开头的6个image地址修改成如下内容：
```
sidecars:
  provisionerImage:
    repository: opsdockerimage/sig-storage-csi-provisioner
    tag: "v2.1.1"
  attacherImage:
    repository: opsdockerimage/sig-storage-csi-attacher
    tag: "v3.1.0"
  snapshotterImage:
    repository: opsdockerimage/sig-storage-csi-snapshotter
    tag: "v3.0.3"
  livenessProbeImage:
    repository: opsdockerimage/sig-storage-livenessprobe
    tag: "v2.2.0"
  resizerImage:
    repository: opsdockerimage/sig-storage-csi-resizer
    tag: "v1.0.0"
  nodeDriverRegistrarImage:
    repository: opsdockerimage/sig-storage-csi-node-driver-registrar
    tag: "v2.1.0"

```
最后执行安装操作：
```
helm upgrade --install fastcfs-csi ./fcfs-csi-driver
```

如果你可以连接，可忽略上面的步骤，直接执行：
```
helm upgrade --install fastcfs-csi fastcfs-csi/fcfs-csi-driver
```
- 创建静态卷
创建文件fastcfs-static-pv.yml
```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: test-pv
spec:
  capacity:
    storage: 1Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  storageClassName: ""
  csi:
    driver: fcfs.csi.vazmin.github.io
    # volumeHandle should be same as FastCFS pool name
    volumeHandle: test-pv
    nodeStageSecretRef:
      # node stage secret name
      name: csi-fcfs-secret
      # node stage secret namespace where above secret is created
      namespace: default
    volumeAttributes:
      # Required options from storage class parameters need to be added in volumeAttributes
      "clusterID": "virtual-cluster-id-1"
      "static": "true"
  persistentVolumeReclaimPolicy: Retain
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: fcfs-claim
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: ""
  resources:
    requests:
      storage: 1Gi
  volumeName: test-pv
---
```
执行：
```
kubectl apply -f fastcfs-static-pv.yml
```
- 创建storageClass和动态卷
创建fastcfs-storageClass.yml
```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: csi-fcfs-sc
provisioner: fcfs.csi.vazmin.github.io
reclaimPolicy: Delete
volumeBindingMode: Immediate
allowVolumeExpansion: true
parameters:

  # The secrets have to contain admin credentials.
  csi.storage.k8s.io/provisioner-secret-name: csi-fcfs-secret
  csi.storage.k8s.io/provisioner-secret-namespace: default
  csi.storage.k8s.io/controller-expand-secret-name: csi-fcfs-secret
  csi.storage.k8s.io/controller-expand-secret-namespace: default
  csi.storage.k8s.io/node-stage-secret-name: csi-fcfs-secret
  csi.storage.k8s.io/node-stage-secret-namespace: default
  csi.storage.k8s.io/node-publish-secret-name: csi-fcfs-secret
  csi.storage.k8s.io/node-publish-secret-namespace: default

  clusterID: virtual-cluster-id-1
```
创建fastcfs-dynamic-pv.yml
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: csi-fcfs-claim
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: csi-fcfs-sc
  resources:
    requests:
      storage: 1Gi
```
执行
```
kubectl apply -f fastcfs-storageClass.yml
kubectl apply -f fastcfs-dynamic-pv.yml
```
查看生成的pool，说明新建CSI Driver集成成功！
```
fcfs_pool plist k8s
  No.                                          pool_name      quota (GiB)
   1.   csi-vol-pvc-5722b1ea-548e-4154-95bf-fbef9c4fab8a                1

```

# 三、总结
FastCFS支持大规模的集群，对服务器要求不高，但是由于没有统一配置中心，如果节点比较多的话，配置会稍显麻烦，虽然官方也可以fcfs.sh运维工具，但是还是不够好用，可以考虑用ansible等工具来安装，后来笔者会写一个ansible的playbook来简化安装FastCFS集群。
