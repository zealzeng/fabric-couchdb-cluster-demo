# Hyperledger fabric peer数据膨胀解决方案探讨
#### 1. 问题场景
Fabric peer节点使用文件保存区块， 使用level db或couchdb数据库保存状态， 数据很多state db会膨胀, 我们探讨下一些解决方案。

#### 2. couchdb集群
couchdb2.x支持集群, 分片, 应该能把数据分散到集群的其它节点。先简单过一下如何安装。

##### 2.1 couchdb集群搭建
Fabric用到的couchdb镜像是自己打包的, 1.4对应的是hyperledger/fabric-couchdb:0.4.14, 不过很悲催, 笔者测试官方这个版本是有不少问题的:
**(1)开启couchdb持久化, 启动直接报错; 不开启持久化可跳过**
```yaml
    volumes:
      - ./couchdb1/data:/opt/couchdb/data
```
```
Could not open file ./data/_users.couch: permission denied
```

**(2)参考couchdb2.2, 2.3官方文档**, docker hub couchdb镜像的安装文档, 三个节点可以复制， 但是每个节点的状态是 single_node_enable, 而换成docker hub couchdb镜像则直接成功。笔者已将把问题提交到官方JIRA https://jira.hyperledger.org/browse/FABB-118 不知道会有人跟进不。

搭建代码分享在https://github.com/zealzeng/fabric-couchdb-cluster-demo
docker启动couchdb算简单,  不过参数有点多, 最少需要三个节点, 我们参考docker-compose-couchdb1.yaml
```yaml
#
# Copyright IBM Corp All Rights Reserved
#
# SPDX-License-Identifier: Apache-2.0
#
version: '2'

networks:
  basic:

services:
  couchdb1:
    container_name: couchdb
    #image: hyperledger/fabric-couchdb:0.4.14
    image: couchdb:2.2.0
    # Populate the COUCHDB_USER and COUCHDB_PASSWORD to set an admin user and password
    # for CouchDB.  This will prevent CouchDB from operating in an "Admin Party" mode.
    environment:
      - COUCHDB_USER=admin
      - COUCHDB_PASSWORD=adminpwd
      - NODENAME=192.168.31.86
      - ERL_FLAGS=-setcookie "brumbrum" -kernel inet_dist_listen_min 9100 -kernel inet_dist_listen_max 9100
      - COUCHDB_SECRET=1234567890
    ports:
      - 5984:5984
      #- 5986:5986
      - 4369:4369
      - 9100:9100
    volumes:
      - ./couchdb1/data:/opt/couchdb/data
    #  - ./couchdb1/etc/local.ini:/opt/couchdb/etc/local.ini
    #  - ./couchdb1/etc/vm.args:/opt/couchdb/etc/vm.args
    #networks:
    #  - basic
```

必须使用低版本些couchdb:2.2.0, 实际上fabric-couchdb：0.4.14用的是2.2版本的couchdb, 2.3.0已测试过, 无法启动。
NODENAME一般用IP或完整域名, 实际生成的节点唯一名字为couchdb@NODENAME.

ERL_FLAGS实际是couchdb启动参数, 会对应生成一个ini文件, setcookie也是一个通信密钥, 到官方文档查下https://docs.couchdb.org/en/2.2.0/cluster/setup.html?highlight=cluster.

端口5984是一个Couchdb Fauxton工具或http api端口, 要保证fabric peer能采访。
端口5986是内部管理任务的端口, 可不开放。
端口4369是erlang epmd机制的通信端口, 好像是要节点之间相互采访, 通过setcookie保证安全。
端口9100用于集群内节点通信, 说是随机生成, 只能限制端口范围, 不开放集群好像就起不来。

###### 安装步骤
**(1)启动三个couchdb节点**
192.168.31.86执行step1-start-couchdb1.sh
192.168.31.168执行step1-start-couchdb2.sh
192.168.31.121执行step1-start-couchdb1.sh

**(2)选择一个操作节点192.168.31.86**
假设ssh登录86, 默认使用127.0.0.1， 也可以使用86采访
```shell
curl -X POST -H "Content-Type: application/json" http://admin:adminpwd@127.0.0.1:5984/_cluster_setup -d '{"action": "enable_cluster", "bind_address":"0.0.0.0", "username": "admin", "password":"adminpwd", "port": 5984, "node_count": "3", "remote_node": "192.168.31.168", "remote_current_user": "admin", "remote_current_password": "adminpwd" }'
curl -X POST -H "Content-Type: application/json" http://admin:adminpwd@127.0.0.1:5984/_cluster_setup -d '{"action": "add_node", "host":"192.168.31.168", "port": 5984, "username": "admin", "password":"adminpwd"}'
curl -X POST -H "Content-Type: application/json" http://admin:adminpwd@127.0.0.1:5984/_cluster_setup -d '{"action": "enable_cluster", "bind_address":"0.0.0.0", "username": "admin", "password":"adminpwd", "port": 5984, "node_count": "3", "remote_node": "192.168.31.121", "remote_current_user": "admin", "remote_current_password": "adminpwd" }'
curl -X POST -H "Content-Type: application/json" http://admin:adminpwd@127.0.0.1:5984/_cluster_setup -d '{"action": "add_node", "host":"192.168.31.121", "port": 5984, "username": "admin", "password":"adminpwd"}'
curl -X POST -H "Content-Type: application/json" http://admin:adminpwd@127.0.0.1:5984/_cluster_setup -d '{"action": "finish_cluster"}'
```

要确保86能够采访168, 121其它两个节点, 防火墙之类要保证通信。
最后验证一下集群安装.
````shell
curl http://admin:adminpwd@127.0.0.1:5984/_cluster_setup
````
返回内容要保证是cluster_finished,如果是single_node_enabled, single_node_disabled多数要重来。
{"state":"cluster_finished"}

**(3)测试下集群**
http://192.168.31.86:5984/_utils/
http://192.168.31.168:5984/_utils/
http://192.168.31.121:5984/_utils/

登录fauxton, 在一个节点创建或更新文档或数据,  在其它节点能够看到变化数据就对了。 一个节点停止后， 其它节点应该也能采访。

##### 2.2 peer连接couchdb
参考docker-compose.yaml, 假设启动一个peer节点。
```yaml
      - CORE_LEDGER_STATE_STATEDATABASE=CouchDB
      #- CORE_LEDGER_STATE_COUCHDBCONFIG_COUCHDBADDRESS=couchdb:5984
      - CORE_LEDGER_STATE_COUCHDBCONFIG_COUCHDBADDRESS=192.168.31.86:5984
      # The CORE_LEDGER_STATE_COUCHDBCONFIG_USERNAME and CORE_LEDGER_STATE_COUCHDBCONFIG_PASSWORD
      # provide the credentials for ledger to connect to CouchDB.  The username and password must
      # match the username and password set for the associated CouchDB.
      - CORE_LEDGER_STATE_COUCHDBCONFIG_USERNAME=admin
      - CORE_LEDGER_STATE_COUCHDBCONFIG_PASSWORD=adminpwd
```

默认只能接入一个couchdb节点。couchdb貌似是提倡前置一个均衡负载HAPROXY或NGINX。
执行step2-start-fabric.sh就可以启动了。

登录cli可查询下, 也可以到fauxton查询下, 数据都是同步的。
peer chaincode query -C mychannel -n fabcar -c '{"Args":["queryAllCars"]}'

一些链码更新操作的同步测试这里就跳过了。 

##### 2.3 小结
couchdb的集群, 复制, 分片是否默认真的解决我们的问题 这里需要进一步研究下couchdb。但一个原则是如果能保证不是所有数据都放要给couchdb节点就好， 如果单纯只是每个节点都保存所有数据， 每个节点只是复制备份这样就没撒用了。 分片参考下
https://docs.couchdb.org/en/2.2.0/cluster/sharding.html?highlight=shard


#### 3.使用网络存储
每个peer节点对应一个couchdb, couchdb使用网络存储NFS, NAS等扩容, fabric原本也是分布式记账本, 怕一个peer节点挂， 就多建两个peer冗余就好。

现在有不少区块链分片的实现， 闪电网络，侧链，迅雷的同构多链出现，实际上也是各玩各的，没有一个标准，fabric的路还长。

[![](https://www.javatree.cn/file-server/e/20190219/t_9f4a284840c74b9ea2cf907848fb5490.png)](https://www.javatree.cn/file-server/e/20190219/t_9f4a284840c74b9ea2cf907848fb5490.png)
















