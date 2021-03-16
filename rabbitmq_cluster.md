## RabbitMQ集群安装

本次安装操作系统为`CentOS Linux release 7.9.2009`，软件版本为：`3.7.16`

**规划机器**

```
172.18.0.10 rabbitmq-10
172.18.0.20 rabbitmq-20
172.18.0.30 rabbitmq-30
```

**设置主机名**

```
~]# hostnamectl  set-hostname rabbitmq-10
~]# hostnamectl  set-hostname rabbitmq-20
~]# hostnamectl  set-hostname rabbitmq-30
```

**为每台机器创建rabbitmq账号，并设置密码**

```
~]# groupadd  -g 2000 rabbitmq
~]# useradd  -g 2000 -u 2000 rabbitmq
~]# echo rabbitmq:rabbitmq@1234 |chpasswd
~]# echo "rabbitmq    ALL=(ALL)NOPASSWD:ALL" >> /etc/sudoers
```

**下载程序包**

```
~]$ wget http://www.rpmfind.net/linux/centos/7.9.2009/os/x86_64/Packages/socat-1.7.3.2-2.el7.x86_64.rpm
~]$ wget https://github.com/rabbitmq/rabbitmq-server/releases/download/v3.7.16/rabbitmq-server-3.7.16-1.el7.noarch.rpm
~]$ wget wget https://github.com/rabbitmq/erlang-rpm/releases/download/v22.2/erlang-22.2-1.el7.x86_64.rpm
```

**安装程序包**

```
~]# sudo yum install socat-1.7.3.2-2.el7.x86_64.rpm erlang-22.2-1.el7.x86_64.rpm  rabbitmq-server-3.7.16-1.el7.noarch.rpm
```

**启动服务**

```
~]$ sudo systemctl  start rabbitmq-server && sudo systemctl enable rabbitmq-server
```

**同步`erlang.cookie`**

```
~]$ ls /home/rabbitmq/.erlang.cookie  -l
-r-------- 1 rabbitmq rabbitmq 20 3月  12 00:00 /home/rabbitmq/.erlang.cookie
~]$ sudo  scp ~/.erlang.cookie rabbitmq-20:/home/rabbitmq/
~]$ sudo  scp ~/.erlang.cookie rabbitmq-30:/home/rabbitmq/
~]$ sudo chown rabbitmq.rabbitmq .erlang.cookie
```

**再次启动服务**

```
~]$ sudo systemctl  start rabbitmq-server
~]$ sudo systemctl  status  rabbitmq-server
```

**创建集群**

以`rabbitmq-10`为基准，将`rabbitmq-20`、`rabbitmq-30`加入到集群中，在`rabbitmq-20/30`上执行如下命令

```
~]$ sudo rabbitmqctl  stop_app
~]$ sudo rabbitmqctl  reset
~]$ sudo rabbitmqctl  join_cluster rabbit@rabbitmq-10
~]$ sudo rabbitmqctl  start_app
```

**检查集群的状态**

```
~]$ sudo rabbitmqctl  cluster_status
Cluster status of node rabbit@rabbitmq-10 ...
[{nodes,[{disc,['rabbit@rabbitmq-10','rabbit@rabbitmq-20',
                'rabbit@rabbitmq-30']}]},
 {running_nodes,['rabbit@rabbitmq-30','rabbit@rabbitmq-20',
                 'rabbit@rabbitmq-10']},
 {cluster_name,<<"rabbit@rabbitmq-10">>},
 {partitions,[]},
 {alarms,[{'rabbit@rabbitmq-30',[]},
          {'rabbit@rabbitmq-20',[]},
          {'rabbit@rabbitmq-10',[]}]}]
```

**管理操作**

```
启动管理插件：~]$ sudo rabbitmq-plugins enable rabbitmq_management
添加账号：~]$ sudo rabbitmqctl  add_user admin admin
设置权限：~]$ sudo rabbitmqctl set_permissions -p / admin ".*" ".*" ".*"
赋权：~]$ sudo rabbitmqctl set_user_tags admin administrator
访问控制台：http://ip:15672
```

**设置镜像队列**

针对每一个镜像队列都包含一个master和多个slave节点。如果master不工作，那么镜像队列最早的slave升级为master。

```
~]$ sudo rabbitmqctl  set_policy -p / --priority 0 --apply-to queues mirror_queue "^[0-9a-zA-Z]" '{"ha-mode": "all", "ha-sync-mode":"automatic"}
```

## 其他

故障节点剔除：`rabbitmqctl -n rabbit@rabbitmq-10 forget_cluster_node rabbit@rabbitmq-20 ` 

重置节点数据：`rm -rf /var/lib/rabbitmq/mnesia`
