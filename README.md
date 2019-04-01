# 数据同步的问题——Canal
常常我们在一个大型分布式系统中会有redis、solr、mysql等各种数据库，往往nosql数据库都是从mysql总获取数据，那在系统运行中如何保证数据的同步呢？我们常常有以下三种
## 利用业务代码实现同步
往往在删除或者修改之后，执行逻辑代码实现同步。

**优点：操作简单**
**缺点：1. 业务耦合度太高 2. 执行效率变低**

## 利用定时器来实现同步(SpringTask、Quartz)
在数据库添加一个时间戳字段，然后每次定时执行任务，查询最后一个时间之后的数据变化然后同步到各个nosql数据库中。
**优点：和业务代码实现了解耦**
**缺点： 数据的实时性不高**
## 通过MQ实现同步
当执行完删除或者修改操作之后，发送消息到消息中间件，然后消费者接收到消息，执行同步逻辑
**优点：业务逻辑解耦，可以做到准实时**
**缺点： 还是要在业务代码中加入发送消息的代码，API耦合**

## 通过Canal来实现实时同步(阿里巴巴技术)
通过Canal来解析数据库的日志信息，**来检测数据库中表结构的数据变化**，从而更新Nosql数据库
**优点：业务逻辑解耦，可以做到准实时，API完全解耦**
# Canal深入学习
[介绍canal的github地址](https://github.com/alibaba/canal "github地址")
[点我去下载地址](https://github.com/alibaba/canal/releases?after=canal-1.0.26-preview-3 "具体下载地址")
## 内部组成
![canal内部组成](http://qiniuyun.lmxljc.xyz/canal%E5%86%85%E9%83%A8%E7%BB%84%E6%88%90.png "canal内部组成")
其中 server代表一个canal运行实例，对应一个jvm,Instance对应一个数据队列(可能有多个)

**接着instance下的三个子模块关系**
![instance下的是三个子模块的关系](http://qiniuyun.lmxljc.xyz/cana%E5%86%85%E9%83%A8l%E4%B8%89%E8%80%85%E5%85%B3%E7%B3%BB.png "instance下的是三个子模块的关系")

EventParser: 数据源接入，模拟slave协议和master进行交互,协议解析
EventSink: Parser和Store的链接器，进行数据的过滤、加工、分发的工作
EventStore: 数据的存储
## 进行canal搭建
canal原理利用mysql binlog技术，所以要开启Binlog写入功能，建议binlog模式为row；
```
mysql> show variables like 'binlog_format';
+---------------+-----------+
| Variable_name | Value     |
+---------------+-----------+
| binlog_format | STATEMENT |
+---------------+-----------+
1 row in set (0.00 sec)
```
**通过查询我们可以看出默认是STATEMENT,所以我们需要设置成row**

然后接着我们可以进行配置

```
1. 在linux中复制一份conf到/etc下
 cp/usr/share/mysql/my-default.cnf  /etc/my.cnf
2. 修改etc下的
 my.cnlog-bin=mysql-bin
 binlog_format=ROW
 server_id=1f
3.然后重启mysql
 service mysql restart
```
由于canal底层和主从相近，所以我们需要创建一个用户

```
1. 创建一个用户root1密码root1
CREATE USER root1@'localhost' IDENTIFIED BY 'root1';
2. 赋予查询权限(slave只复制读操作)
GRANT SELECT, REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO ' root1'@'localhost';
3. 刷新权限
FLUSH PRIVILEGES;
```
上传解压 canal.deployer-1.0.24.tar.gz，我解压到了/usr/local/canal下目录下
编辑 canal/conf/example/instance.properties :

![canal配置文件](http://qiniuyun.lmxljc.xyz/canal%E9%85%8D%E7%BD%AE%E6%96%87%E4%BB%B6.png "canal配置文件")
选项含义: 
1) canal.instance.mysql.slaveId : mysql 集群配置中的 serverId 概念，需要保证和当前 mysql 集群中 id 唯一;
2) canal.instance.master.address: mysql 主库链接地址;
3) canal.instance.dbUsername : mysql 数据库帐号
4) canal.instance.dbPassword : mysql 数据库密码;
5) canal.instance.defaultDatabaseName : mysql 链接时默认数据库;
6) canal.instance.connectionCharset : mysql 数据解析编码;
7) canal.instance.filter.regex : mysql 数据解析关注的表，Perl 正则表达

接下来我们启动canal
cd /usr/local/canal/bin/startup.sh
启动之后进入/usr/local/canal/logs/example，然后查询日志tail -f example.log

然后我们根据官方一个的测试工程
```java
package com.alibaba.otter.canal.example;

import java.net.InetSocketAddress;

import org.apache.commons.lang.exception.ExceptionUtils;

import com.alibaba.otter.canal.client.CanalConnector;
import com.alibaba.otter.canal.client.CanalConnectors;
import com.alibaba.otter.canal.common.utils.AddressUtils;


public class SimpleCanalClientTest extends AbstractCanalClientTest {

    public SimpleCanalClientTest(String destination){
        super(destination);
    }

    public static void main(String args[]) {
        // 根据ip，直接创建链接，无HA的功能
        String destination = "example";
        CanalConnector connector = CanalConnectors.newSingleConnector(new InetSocketAddress("192.168.13.130",
            11111), destination, "", "");

        final SimpleCanalClientTest clientTest = new SimpleCanalClientTest(destination);
        clientTest.setConnector(connector);
        clientTest.start();
        Runtime.getRuntime().addShutdownHook(new Thread() {

            public void run() {
                try {
                    logger.info("## stop the canal client");
                    clientTest.stop();
                } catch (Throwable e) {
                    logger.warn("##something goes wrong when stopping canal:\n{}", ExceptionUtils.getFullStackTrace(e));
                } finally {
                    logger.info("## canal client is down.");
                }
            }

        });
    }

}
```
InetSocketAddress修改自己服务器ip
然后创建了canaldb表之后

![新建表后canal变化](http://qiniuyun.lmxljc.xyz/canaldb%E6%9B%B4%E6%94%B9%E5%8F%98%E5%8C%96.png "添加数据后canal变化")
执行插入语句后
```java
INSERT INTO tb_book(NAME , author , publishtime , price , publishgroup) 
VALUES('白帽子讲安全协议 2','吴瀚请',NOW(),99.00,'电子工业出版社');
```
执行之后，控制台打印出对应的日志过程
![执行插入语句后](http://qiniuyun.lmxljc.xyz/canal%E5%9C%A8%E6%8F%92%E5%85%A5%E8%AF%AD%E5%8F%A5%E5%90%8E.png "执行插入语句后")
基本上使用上结束

## canal实现redis
工程已经在gitub上，大家可以download下来运行在idea上，[点我去下载](https://github.com/lmx110522/canal-redis-sync.git "点我去下载")




















