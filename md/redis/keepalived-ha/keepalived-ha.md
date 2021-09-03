# Redis Keepalived HA

>乞丐版哨兵， Redis + Keepalived 高可用方案，曲线救国。



## 测试环境

| IP            | OS         | VIP           | 中间件                              |
| ------------- | ---------- | ------------- | ----------------------------------- |
| 172.18.40.198 | CentOS 7.6 | 172.18.40.222 | redis (master)，keepalived (master) |
| 172.18.40.199 | CentOS 7.6 | 172.18.40.222 | redis (slave)，keepalived (backup)  |



## 功能验证

### 架构图

![image-20201202111456911](./img/image-20201202111456911.png)



### 测试过程

先试试守护，停止 redis 主节点（198） 。

```bash
service redis stop
```

![image-20201202111915130](./img/image-20201202111915130.png)



触发 keepalived 检测脚本，查看脚本日志显示 redis 成功自重启。

```bash
vim /app/log/keepalived/redis-state.log
```

![image-20201202111959465](./img/image-20201202111959465.png)



接下来同时停止 redis 主节点（198）与 keepalived 主节点（198），模拟宕机。

```bash
service redis stop && service keepalived stop
```

![image-20201202112946665](./img/image-20201202112946665.png)



查看 keepalived 从节点（199）系统日志，显示已切换为 `MASTER` 状态。

```bash
journalctl -xe
```

![image-20201202113150524](./img/image-20201202113150524.png)



VIP 发生漂移。

```bash
ip a
```

![image-20201202113455156](./img/image-20201202113455156.png)



查看此时的 keepalived 主节点（199）的脚本日志，显示原 redis 从节点（199）已成功切换为主节点。

![image-20201202115357542](./img/image-20201202115357542.png)



不妨检查下 199 服务器 redis 的 info 信息，确实已经提升为主节点了。

```bash
/app/soft/redis-5.0.5/bin/redis-cli -a powersi@redis info replication
```

![image-20201202134159811](./img/image-20201202134159811.png)



然后再启动原主节点 (198)上已停止的 keepalived 服务，模拟恢复。

```bash
service keepalived start
```

![image-20201202135426172](./img/image-20201202135426172.png)



查看 198 服务器系统日志，显示 keepalived 节点已通过检测，且重新提权成为主节点。

![image-20201202135337356](./img/image-20201202135337356.png)



VIP 漂回 198 服务器。

![image-20201202135644410](./img/image-20201202135644410.png)



查看 198 脚本日志，显示 keepalived 首先尝试重启了 198 服务器的 redis，再执行 `SYNC` 从当前 redis 主节点 (199) 进行数据同步，最后将 198 的 redis 重新提升为主节点，完成切回。

![image-20201202135810072](./img/image-20201202135810072.png)



于此同时，检查 199 服务器的脚本日志，显示其在等待对方 `SYNC` 同步后，切回了 redis 从节点。

![image-20201202140259559](./img/image-20201202140259559.png)



## 数据丢失验证

### Java demo

```java
package com.xzq.keepalived;

import redis.clients.jedis.Jedis;
import redis.clients.jedis.exceptions.JedisConnectionException;
import redis.clients.jedis.exceptions.JedisDataException;

/**
 * 计数器
 *
 * @author xzq
 */
public class IncrDemo {

    public static void main(String[] args) throws InterruptedException {

        // 从VIP获得redis连接
        Jedis client = new Jedis("172.18.40.222", 6379);
        client.auth("powersi@redis");
        // 总请求数
        int qry = 0;
        // 总异常数
        int err = 0;

        while (true) {
            try {
                // 发送连续请求
                int pos = client.incr("data").intValue();
                System.out.println("成功写入数据，当前data自增至" + pos);
                // 存在数据丢失 打印相关信息
                if (qry != pos - 1) {
                    System.out.println("数据丢失，原请求数" + qry + "，现自增后结果" + pos);
                }
                // 更新请求数
                qry = pos;
                Thread.currentThread().sleep(50);
            } catch (JedisConnectionException e1) {
                err++;
                System.out.println("第" + err + "次连接异常，无法访问redis");
                try {
                    // 尝试重新获取redis连接
                    client = new Jedis("172.18.40.222", 6379);
                    client.auth("powersi@redis");
                    System.out.println("重新获取redis连接成功");
                } catch (Exception e2) {
                    System.out.println("第" + err + "次尝试获取redis连接失败");
                }
            } catch (JedisDataException e3) {
                System.out.println("当前redis节点不可写");
            }
        }
    }
}

```



### 容灾切换

>- 前提：初始状态。
>- 事件：198 服务器宕机，199 服务器 redis 从节点提升为主节点，并继续对外提供服务。

运行测试 demo，持续 `incr`。

![image-20201203091843421](./img/image-20201203091843421.png)



模拟 198 服务器宕机。

![image-20201202185443195](./img/image-20201202185443195.png)



redis 客户端连接中断，在尝试 2 次重连后成功，继续写入数据，自增 Key 连续，无丢失数据日志打印。容灾切换过程无数据丢失。

![image-20201203091933935](./img/image-20201203091933935.png)



### 恢复切回

>- 前提：198 服务器宕机，199 服务器 redis 提升为主节点且持续对外提供服务。
>- 事件：198 服务器 keepalived 恢复，redis 主节点切回，同时 199 服务器 redis 降级为从节点。

模拟 198 服务器宕机。

![image-20201202185443195](./img/image-20201202185443195.png)



运行测试 demo，持续 `incr`。

![image-20201203091817758](./img/image-20201203091817758.png)



恢复 198 服务器 keepalived。

![image-20201202152019580](./img/image-20201202152019580.png)



keepalived 尝试重启 198 服务器 redis，之后进行了 10s 数据同步，最后将 198 的 redis 提升为主节点。

![image-20201202193039487](./img/image-20201202193039487.png)



keepalived 切回时，redis 客户端连接中断，在尝试1次重连后成功，之后10s的同步数据时间均不可写。

![image-20201203092356130](./img/image-20201203092356130.png)



数据同步完成，198 服务器 redis 提升为主节点，对外提供服务，继续写入数据，自增 Key 连续，无丢失数据日志打印。恢复切回过程无数据丢失。

![image-20201203092439135](./img/image-20201203092439135.png)

注意，这里的数据无丢失结论是片面的，只能说明 10s 内将数据同步完成了。假如数据量很大，这时 198 服务器恢复，10s 的时间内可能并未同步完所有数据，导致数据丢失。



### 特殊情况

>- 前提：恢复切回后，数据刚同步完成，198 服务器 redis 提升为主节点。
>- 事件：5s 内，再次发生容灾切换，即 198 服务器 keepalived 宕机。

运行测试 demo，持续 `incr`。

![image-20201203091817758](./img/image-20201203091817758.png)



198 服务器宕机，容灾切换发生，重新获得 redis 连接后，自增 Key 连续，无数据丢失。

![image-20201203103209502](./img/image-20201203103209502.png)



198 服务器 keepalived 恢复。恢复切回，重新获取 redis 连接。

![image-20201203103525622](./img/image-20201203103525622.png)



198 服务器redis数据同步完成，自增 Key 连续，无数据丢失。

![image-20201203103554169](./img/image-20201203103554169.png)



5s 内，198 服务器再次宕机，VIP 漂移至 199 服务器，此时 199 服务器 redis 仍处于 `redis_backup.sh` 脚本中等待对方同步的状态 (199 服务器 redis 等待同步睡眠 15s，而 198 服务器 redis 同步只用 10s)，其状态仍然为 redis 主节点，即还未向当前对外提供服务的 198 服务器同步这 5s 内的最新数据 ，造成数据丢失情况。下图自增 Key 为 421，正好对应 198 服务器恢复后写入的第一份数据。

![image-20201203104053645](./img/image-20201203104053645.png)



## 相关问题

### keepalived 当前 state 获取方法

使用 `notify` 命令指定 keepalived 状态文件。

![image-20201202195240247](./img/image-20201202195240247.png)



定义通知脚本 `notify.sh`。

![image-20201202195330377](./img/image-20201202195330377.png)



查看实时打印的 keepalived 状态日志。

![image-20201202195504271](./img/image-20201202195504271.png)



截取当前 keepalived 状态。

```bash
awk 'END {print $8}' /app/log/keepalived/keepalived.log | sed 's/,//g'
```

![image-20201202195625945](./img/image-20201202195625945.png)



### keepalived 初始化 BACKUP 状态

重启 keepalived 主节点。

![image-20201202195831674](./img/image-20201202195831674.png)



查看系统日志。

![image-20201202200015330](./img/image-20201202200015330.png)

可见 keepalived 启动流程如下：

- 初始化当前节点为 `BACKUP`，执行 `notify_backup` 所指定的通知脚本。
- 运行 `VRRP_Script` 配置块中指定的检测脚本，节点权值发生转变。
- 若当前节点权值满足成为 `MASTER`，则切换为 `MASTER`，同时执行 `notify_master` 所指定的通知脚本。



### 存在双向同步情况

>- 前提：初始状态。
>- 事件：keepalived 主节点 (198) 重启。

重启 198 服务器 keepalived。

![image-20201202152920705](./img/image-20201202152920705.png)



198 服务器系统日志。

![image-20201202202615396](./img/image-20201202202615396.png)

>**说明**
>
>- 16:59，keepalived 重启，初始化 `BACKUP`，执行 `redis_backup.sh`。
>- 17:03，keepalived 重启 4s 后 (keepalived 检测间隔)，通过检测脚本，提权为 `MASTER`，执行 `redis_master.sh`。
>- 17:03 -- 17:13，`redis_master.sh` 脚本执行期间，发现 199 服务器 redis 存活，尝试同步。此时 198 服务器 redis 为从节点。
>- 17:13，`redis_master.sh` 脚本同步数据开始 10s 后 (脚本中同步时间设置的 10s)，198 服务器 redis 成为主节点。
>- 17:14，`redis_backup.sh` 开始执行 15s 后 (睡眠 15s)，发现当前 keepalived 节点为 `MASTER`，不执行降级从节点动作。无日志打印。



199服务器系统日志。

![image-20201202202744815](./img/image-20201202202744815.png)

>**说明**
>
>- 16:59，198 服务器 keepalived 主节点死亡，199 服务器 keepalived 立即被提升为 `MASTER`，同时触发 `redis_master.sh`。
>- 17:03，198 服务器 keepalived 重启 4s 后，进行了第一次脚本检测且通过，其权值恢复且被提升为 `MASTER`。同时，199 服务器 keepalived 切换为 `BACKUP`，触发 `redis_backup.sh`。
>- 16:59 -- 17:09，`redis_master.sh` 脚本执行期间，发现 198 服务器 redis 存活，尝试同步。此时 199 服务器 redis 为从节点。
>- 17:09，`redis_master.sh` 脚本同步数据开始 10s 后 (脚本中同步时间设置的 10s)，199 服务器 redis 成为主节点。
>- 17:18，`redis_backup.sh` 开始执行 15s 后 (睡眠 15s)，发现当前 keepalived 节点为 `BACKUP`，降级从节点。



可见，17:03 -- 17:09 这段时间内，198 与 199 服务器 redis 均为从节点且相互同步数据。



## 附页

### redis_check.sh (检测)

```bash
#!/usr/bin/env bash

PORT=$1
PASSWORD=$2
. /home/app/.bashrc

REDISCLI="redis-cli -p $PORT -a $PASSWORD"
LOGFILE="/app/log/keepalived/redis-state.log"

function log() {
    local LEVEL=$1
    local MESSAGE=$2
    local TS=`date '+%Y-%m-%d %H:%M:%S'`

    echo "$TS [$LEVEL] CHECK [$PORT] $MESSAGE" >> $LOGFILE
}

RESP=`$REDISCLI PING`
if [ "$RESP"x != "PONG"x ]; then
	log WARNING "Redis server PING request: No Response, Now restart..."
        service redis restart
	sleep 3
	log INFO "Redis server restarted"
	if [ "`$REDISCLI PING`"x != "PONG"x ]; then
		log ERROR "Redis server has crashed" 
		exit 1
	else 
		log INFO "Redis server has been recovered"
	fi
	# 获取当前keepalived状态，MASTER，BACKUP，STOP
	STATE=`awk 'END {print $8}' /app/log/keepalived/keepalived.log | sed 's/,//g'`
	if [ "$STATE"x == "MASTER"x ]; then
		$REDISCLI SLAVEOF NO ONE
		if [ $? -ne 0 ]; then
			log ERROR "Switch redis[$PORT] to master failed"
		else
			log INFO "Switch redis[$PORT] to master succeed"
		fi
	fi
	exit 0
else
	exit 0
fi
```



### redis_master.sh (提升主节点)

```bash
#!/usr/bin/env bash

MASTER_IP=$1
MASTER_PORT=$2
PASSWORD=$3
. /home/app/.bashrc

REDISCLI="redis-cli -p $MASTER_PORT -a $PASSWORD"
LOGFILE="/app/log/keepalived/redis-state.log"

function log() {
    local LEVEL=$1
    local MESSAGE=$2
    local TS=`date '+%Y-%m-%d %H:%M:%S'`

    echo "$TS [$LEVEL] MASTER [$MASTER_PORT] $MESSAGE" >> $LOGFILE
}

# 判断待同步主节点是否存活
RESP=`$REDISCLI -h $1 PING`
if [ "$RESP"x = "PONG"x ]; then
    log INFO  "Sync data from original master: SLAVEOF $MASTER_IP $MASTER_PORT"
    # slaveof当前主节点，sync同步数据
	$REDISCLI SLAVEOF $MASTER_IP $MASTER_PORT
	if [ $? -ne 0 ]; then
		log ERROR "Sync data from original master failed"
	else
		log INFO "Sync data from original master ..."
	fi
    # 睡眠10s，等待数据同步
	sleep 10
fi

log INFO "Assuming data synchronization is complete, now switch Redis[$MASTER_PORT] to Master"
# 提升当前slave为master
$REDISCLI SLAVEOF NO ONE
if [ $? -ne 0 ]; then
    log ERROR "Switch redis[$MASTER_PORT] to master failed"
else
    log INFO "Switch redis[$MASTER_PORT] to master succeed"
fi
```



### redis_backup.sh (降级从节点)

```bash
#!/usr/bin/env bash

MASTER_IP=$1
MASTER_PORT=$2
PASSWORD=$3
. /home/app/.bashrc

REDISCLI="redis-cli -p $MASTER_PORT -a $PASSWORD"
LOGFILE="/app/log/keepalived/redis-state.log"

function log() {
    local LEVEL=$1
    local MESSAGE=$2
    local TS=`date '+%Y-%m-%d %H:%M:%S'`

    echo "$TS [$LEVEL] BACKUP [$MASTER_PORT] $MESSAGE" >> $LOGFILE
}

log INFO "Waiting data sync..."

sleep 15

# 当前keepalived状态，MASTER，BACKUP，STOP
STATE=`awk 'END {print $8}' /app/log/keepalived/keepalived.log | sed 's/,//g'`

# 若当前keepalived状态为MASTER，则不将当前redis降级为从
if [ "$STATE"x != "MASTER"x ]; then
	log INFO "Switching redis server $MASTER_PORT to slave mode"
	$REDISCLI SLAVEOF $MASTER_IP $MASTER_PORT
	if [ $? -ne 0 ]; then
		log ERROR "Switch redis server $MASTER_PORT to slave mode failed"
	else
		log INFO "Switch redis server $MASTER_PORT to slave mode succeed"
	fi
fi
```



### keepalived.conf

- MASTER

```conf
! Configuration File for keepalived
global_defs {
   router_id REDIS-HA
   script_user root
}

vrrp_script chk_tcp_port {
    script "/etc/keepalived/scripts/redis_check.sh 6379 powersi@redis"		# 检测脚本
    weight -10		# script 返回值非 0 时，priority 减少 10
    interval 4		# 检测间隔
    rise 3			# 成功 3 次，服务器被标记为成功
    fall 2			# 失败 2 次，服务器被标记为失败
}

vrrp_instance VI_1 {
   state MASTER		# 初始状态
   interface ens192
   virtual_router_id 222
   priority 100		# 权值
   advert_int 1
   authentication {
     auth_type PASS
     auth_pass 1111
   }
   virtual_ipaddress {
     172.18.40.222
   }
   track_script {
     chk_tcp_port
   }
   preempt_delay 5
   notify /etc/keepalived/scripts/notify.sh
   
   # 当前 keepalived 节点切换为 MASTER 时，触发 redis_master.sh (提升主节点脚本)
   notify_master "/etc/keepalived/scripts/redis_master.sh 172.18.40.199 6379 powersi@redis"	
   # 当前 keepalived 节点切换为 BACKUP 时，触发 redis_backup.sh (降级从节点脚本)
   notify_backup "/etc/keepalived/scripts/redis_backup.sh 172.18.40.199 6379 powersi@redis"
}

```

- BACKUP

```conf
! Configuration File for keepalived
global_defs {
   router_id REDIS-HA
   script_user root
}

vrrp_script chk_tcp_port {
    script "/etc/keepalived/scripts/redis_check.sh 6379 powersi@redis"
    weight -10
    interval 4
    rise 3
    fall 2
}

vrrp_instance VI_1 {
   state BACKUP
   interface ens192
   virtual_router_id 222
   priority 95
   advert_int 1
   authentication {
     auth_type PASS
     auth_pass 1111
   }
   virtual_ipaddress {
     172.18.40.222
   }
   track_script {
     chk_tcp_port
   }
   preempt_delay 5
   notify /etc/keepalived/scripts/notify.sh

   notify_master "/etc/keepalived/scripts/redis_master.sh 172.18.40.198 6379 powersi@redis"
   notify_backup "/etc/keepalived/scripts/redis_backup.sh 172.18.40.198 6379 powersi@redis"
}
```


