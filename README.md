redis cluster集群构建流程4部：
1.准备节点（docker-compose 编排构建redis主从节点）

2.节点握手（通过Gssip[流氓协议]握手）-->（cluster meet IP port-->cluster meet 192.168.1.9 6391）-->(redis-cli客户端执行)

3.配置节点的主从关系(redis-cli -h ip port[从节点端口] cluster replicate 主节点id)(redis-cli -h 192.168.1.9 6394 cluster replicate fe74467be98b)

4.为主节点分配槽（redis-cli -h 192.168.1.9  -p 6391[主节点端口]  cluster addslots {0..5461}）


==========================================

创建目录
[root@localhost ~]# mkdir -p /usr/local/docker-compose-redis-cluster 
[root@localhost ~]# cd /usr/local/docker-compose-redis-cluster

1.构建redis镜像=====================
[root@localhost docker-compose-redis-cluster]# docker build -t rediscluster .



2.docom-compose执行编排=============

1.构建：
[root@localhost docker-compose-redis-cluster]# docker-compose up -d

2.查看构建容器：
[root@localhost docker-compose-redis-cluster]# docker-compose ps

3.进入容器：
[root@localhost docker-compose-redis-cluster]# docker exec -it redis-slave1 bash

4.进入redis-slave执行握手操作：
[root@67b9fea495ed config]# redis-cli -p 6394
127.0.0.1:6394>cluster meet 192.168.1.9 6391  [此处利用局域网或公网执行握手]
127.0.0.1:6394>ok

127.0.0.1:6394>cluster meet 192.168.1.9 6392
127.0.0.1:6394>ok

127.0.0.1:6394>cluster meet 192.168.1.9 6393
127.0.0.1:6394>ok

127.0.0.1:6394>cluster meet 192.168.1.9 6395
127.0.0.1:6394>ok

127.0.0.1:6394>cluster meet 192.168.1.9 6396
127.0.0.1:6394>ok

5.CLUSTER nodes：列出集群当前已知的所有节点（node）的相关信息
127.0.0.1:6394> cluster nodes
9292e93a4154a22b687849db479fa6b1a68d19c1 172.30.0.1:6392@16392 master - 0 1574690790000 2 connected
d9e7f06869e8ade7acea87c1241094525154fb78 172.30.0.1:6391@16391 master - 0 1574690790212 3 connected
5cd14eb016dc49374092da300742fc8a4ab85018 172.30.0.2:6394@16394 myself,master - 0 1574690789000 1 connected
f6258f82ff53b4f2ee2b07f583fc4216aae4636c 172.30.0.1:6393@16393 master - 0 1574690791221 0 connected
d14924fb04a16ceda18c5ef13468828f7b853764 172.30.0.1:6395@16395 master - 0 1574690789000 4 connected
038078f9d50fd190e4f6d8f26697a6a91759c9df 172.30.0.1:6396@16396 master - 0 1574690788193 5 connected


6.进入/var/lib/redis查看配置文件或用于唯一标识集群内一个节点ID
生产的文件是:nodes-6379.conf

7.打印集群的信息
127.0.0.1:6394>cluster info

8.设置从节点
[root@67b9fea495ed config]# redis-cli -h 192.168.1.9 -p 6394 cluster replicate d9e7f06869e8ade7acea87c1241094525154fb78[主节点的集群唯一标识Id]
[root@67b9fea495ed config]# redis-cli -h 192.168.1.9 -p 6395 cluster replicate 9292e93a4154a22b687849db479fa6b1a68d19c1
[root@67b9fea495ed config]# redis-cli -h 192.168.1.9 -p 6396 cluster replicate f6258f82ff53b4f2ee2b07f583fc4216aae4636c



9.为主节点分配槽
[root@67b9fea495ed config]# redis-cli -h 192.168.1.9 -p 6391 cluster addslots {0..5461}
OK
[root@67b9fea495ed config]# redis-cli -h 192.168.1.9 -p 6392 cluster addslots {5462..10922}
OK
[root@67b9fea495ed config]# redis-cli -h 192.168.1.9 -p 6393 cluster addslots {10923..16383}

查看节点信息:
127.0.0.1:6394> cluster nodes
9292e93a4154a22b687849db479fa6b1a68d19c1 172.30.0.1:6392@16392 master - 0 1574693537000 2 connected 5462-10922
d9e7f06869e8ade7acea87c1241094525154fb78 172.30.0.1:6391@16391 master - 0 1574693536000 3 connected 0-5461
5cd14eb016dc49374092da300742fc8a4ab85018 172.30.0.2:6394@16394 myself,slave d9e7f06869e8ade7acea87c1241094525154fb78 0 1574693538000 1 connected
f6258f82ff53b4f2ee2b07f583fc4216aae4636c 172.30.0.1:6393@16393 master - 0 1574693537454 0 connected 10923-16383
d14924fb04a16ceda18c5ef13468828f7b853764 172.30.0.1:6395@16395 slave 9292e93a4154a22b687849db479fa6b1a68d19c1 0 1574693538464 4 connected
038078f9d50fd190e4f6d8f26697a6a91759c9df 172.30.0.1:6396@16396 slave f6258f82ff53b4f2ee2b07f583fc4216aae4636c 0 1574693534000 5 connected



10.操作redis执行set操作【没有指定集群模式】：


在redis-master1 6391 执行set操作
127.0.0.1:6391> set name 111
(error) MOVED 5798 172.50.0.1:6392--》提示该key需要存储到5798的槽中
127.0.0.1:6391> get name
(error) MOVED 5798 172.50.0.1:6392
127.0.0.1:6391> 
【提示该key需要到6392的节点执行操作】

在redis-master2 6392 执行set操作
127.0.0.1:6392> set name 111
OK
127.0.0.1:6392> get name
"111"
【name信息只有在6392的节点才能操作】

set，get操作都得到指定的槽中操作，不然key无法执行set，get操作
【说明：数据插入一定要到到对应的槽区执行数据插入的操作】


11.集群模式下获取数据：

[root@d8b6958531ad redis]# redis-cli -c -h 192.168.1.9 -p 6391 set name 123456
OK
[root@d8b6958531ad redis]# redis-cli -c -h 192.168.1.9 -p 6391 get name       
"123456"


[root@d8b6958531ad redis]# redis-cli -c -h 192.168.1.9 -p 6391 set nameof 1111 
OK
[root@d8b6958531ad redis]# redis-cli -c -h 192.168.1.9 -p 6391 get nameof     
"1111"

【注意：集群模式下的操作无需指定到对应的槽节点执行数据操作】













 