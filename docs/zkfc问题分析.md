# zkfc报错：
```
 2019-09-17 05:04:12,618 FATAL org.apache.hadoop.hdfs.tools.DFSZKFailoverController: Got a fatal error, exiting now
java.lang.RuntimeException: ZK Failover Controller failed: Received stat error from Zookeeper. code:CONNECTIONLOSS. Not retrying further znode monitoring connection errors.
        at org.apache.hadoop.ha.ZKFailoverController.mainLoop(ZKFailoverController.java:369)
        at org.apache.hadoop.ha.ZKFailoverController.doRun(ZKFailoverController.java:238)
        at org.apache.hadoop.ha.ZKFailoverController.access$000(ZKFailoverController.java:61)
        at org.apache.hadoop.ha.ZKFailoverController$1.run(ZKFailoverController.java:172)
        at org.apache.hadoop.ha.ZKFailoverController$1.run(ZKFailoverController.java:168)
        at org.apache.hadoop.security.SecurityUtil.doAsLoginUserOrFatal(SecurityUtil.java:412)
        at org.apache.hadoop.ha.ZKFailoverController.run(ZKFailoverController.java:168)
        at org.apache.hadoop.hdfs.tools.DFSZKFailoverController.main(DFSZKFailoverController.java:181)
```

  <br>  <br>
# zk报错：
```
2019-09-17 05:04:12,043 [myid:3] - WARN  [QuorumPeer[myid=3]/0.0.0.0:2181:LearnerHandler@702] - Closing connection to peer due to transaction timeout.
```


  <br>  <br>
# 猜测在leader选举的时候造成zkfc挂掉，分析如下：
 
>注：
（1）LearnerHandler：Learner服务器的管理者，主要负责Follower/Observer服务器和Leader服务器之间的一系列网络通信，包括数据同步、请求转发和Proposal提议的投票等。Leader服务器中保存了所有Follower/Observer对应的LearnerHandler。
（2）Learner：细分为Follower和Observer,是Follower和Observer的父类
（3）SyncLimitCheck：控制leader等待当前learner给proposal回复ACK的时间
（4）ping：检测leader与learner是否有proposal超时了

## 根据zk报错看zk源码，在 LearnerHandler 类中
在向 Learner 发送ping消息之前, 首先会通过 syncLimitCheck 来检查 （在Leader上有个while loop会遍历 LearnerHandler 然后ping Learner，以检测leader与learner是否有 proposal 超时）
```
public void ping() {
    long id;
    if (syncLimitCheck.check(System.nanoTime())) {
        synchronized(leader) {
            id = leader.lastProposed;
        }
        QuorumPacket ping = new QuorumPacket(Leader.PING, id, null, null);
        queuePacket(ping);
    } else {
        LOG.warn("Closing connection to peer due to transaction timeout.");
        shutdown();
    }
}
```
  <br>  <br>
## 在zk的LearnerHandler类的内部类SyncLimitCheck的方法check中,当 (time - currentTime) / 1000000 大于等于 tickTime*syncLimit时表示proposal超时，看 currentTime （最久一次更新了但是没有收到ack的proposal的时间） 更新情况
```
// 用“当前时间与最久一次没收到ack的proposal的时间差”是否小于“tickTime * syncLimit”以判断 leader 与 learner 是否有proposal超时了(超过指定时长没有收到ack)
public synchronized boolean check(long time) {
    if (currentTime == 0) {
        return true;
    } else {
        long msDelay = (time - currentTime) / 1000000; //当前时间与最久一次没收到ack的proposal的时间差
        return (msDelay < (leader.self.tickTime * leader.self.syncLimit));
    }
}
```
<br>  <br>
## 在zk的LearnerHandler类的内部类SyncLimitCheck中,由两个方法可以更新currentTime：

在updateAck中，经查看日志后没有发现“ACK for xxx received before ACK for...”,排除："nextZxid == zxid"时更新currentTime，确认是“currentZxid == zxid”时更新currentTime

```

//在proposal leader的时候调用System.nanoTime()更新
public synchronized void updateProposal(long zxid, long time) {
    if (!started) {
        return;
    }
    if (currentTime == 0) {
        currentTime = time;
        currentZxid = zxid;
    } else {
        nextTime = time;
        nextZxid = zxid;
    }
}


//learner 投票后发送ack给 Leader时更新currentTime
public synchronized void updateAck(long zxid) {
	 //如果是刚刚发送的ack
     if (currentZxid == zxid) {
     	 //传递到下一个记录
         currentTime = nextTime;
         currentZxid = nextZxid;
         nextTime = 0;
         nextZxid = 0;
     } else if (nextZxid == zxid) { //如果旧的ack还没收到 但是收到了 新的ack
         LOG.warn("ACK for " + zxid + " received before ACK for " + currentZxid + "!!!!");
         nextTime = 0;
         nextZxid = 0;
     }
}
```


  <br>  <br>

## 经 <a href="https://github.com/zhanghuang03/zkfc_monitor">zkfc_monitor</a> 日志中发现在连接zk时失败：
```
2019-09-17 05:03:55,403 -connection:connection.py-L637-INFO: Connecting to x.x.x.x:2181, use_ssl: False
2019-09-17 05:03:55,525 -client:client.py-L480-INFO: Zookeeper connection established, state: CONNECTED
2019-09-17 05:04:03,691 -connection:connection.py-L608-WARNING: Connection dropped: outstanding heartbeat ping not received
2019-09-17 05:04:03,691 -connection:connection.py-L612-WARNING: Transition to CONNECTING
2019-09-17 05:04:03,691 -client:client.py-L490-INFO: Zookeeper connection lost
2019-09-17 05:04:03,691 -connection:connection.py-L637-INFO: Connecting to x.x.x.x:2181, use_ssl: False
2019-09-17 05:04:03,895 -connection:connection.py-L610-WARNING: Connection time-out: socket time-out during read
2019-09-17 05:04:03,895 -connection:connection.py-L637-INFO: Connecting to x.x.x.x:2181, use_ssl: False
2019-09-17 05:04:03,905 -client:client.py-L480-INFO: Zookeeper connection established, state: CONNECTED
2019-09-17 05:04:07,083 -connection:connection.py-L608-WARNING: Connection dropped: outstanding heartbeat ping not received
2019-09-17 05:04:07,083 -connection:connection.py-L612-WARNING: Transition to CONNECTING
2019-09-17 05:04:07,083 -client:client.py-L490-INFO: Zookeeper connection lost
2019-09-17 05:04:07,083 -connection:connection.py-L637-INFO: Connecting to x.x.x.x:2181, use_ssl: False
2019-09-17 05:04:07,087 -connection:connection.py-L608-WARNING: Connection dropped: socket connection broken
2019-09-17 05:04:07,087 -connection:connection.py-L637-INFO: Connecting to x.x.x.x:2181, use_ssl: False
2019-09-17 05:04:07,287 -connection:connection.py-L610-WARNING: Connection time-out: socket time-out during read
2019-09-17 05:04:08,294 -connection:connection.py-L637-INFO: Connecting to x.x.x.x:2181, use_ssl: False
2019-09-17 05:04:08,495 -connection:connection.py-L610-WARNING: Connection time-out: socket time-out during read
2019-09-17 05:04:08,495 -connection:connection.py-L637-INFO: Connecting to x.x.x.x:2181, use_ssl: False
2019-09-17 05:04:08,697 -connection:connection.py-L610-WARNING: Connection time-out: socket time-out during read
2019-09-17 05:04:08,697 -connection:connection.py-L637-INFO: Connecting to x.x.x.x:2181, use_ssl: False
2019-09-17 05:04:08,897 -connection:connection.py-L610-WARNING: Connection time-out: socket time-out during read
```


  <br>  <br>  <br>  <br>

## 综上所述，zfkc挂掉是由leader选举时  ack消息超时引起的，应检查follower和leader之间的网络是否稳定或调整 ticktime 和 syncLimit
