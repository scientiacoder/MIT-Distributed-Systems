# MIT 6.824： Distributed Systems
Youtube 链接: https://www.youtube.com/watch?v=cQP8WApzIQQ&list=PLrw6a1wE39_tb2fErI4-WkMbsvGQk9_UB&index=1&ab_channel=MIT6.824%3ADistributedSystems  
  
Lab的地址: https://pdos.csail.mit.edu/6.824/  
  
某位大佬的博客: https://yuerblog.cc/2020/08/13/mit-6-824-distributed-systems-%e5%ae%9e%e7%8e%b0raft-lab2a/

## Lecture 1 Introduction
 1. 能用一台机器解决的问题，就不要用分布式
 2. Scalability - 2x Computers -> 2x Throughput
 3. Fault Tolerance:
     - Availability
     - Recoverability:
        - NV(non-volatile storage) Storage： 断电后能恢复，但是代价昂贵，expensive to update, 应该避免使用
        - Replication: identical servers
     - Strong Consistency: 非常expensive, 不推荐 10ms网络延迟communication cost, 能丢失百万条指令
     - **Weak** Consistency: 推荐
 4. MapReduce:
    - Reduce这一块要用到网络通信，网络通信要消耗带宽，如果有10TB文件要MapReduce，则Reduce阶段要在网络中发送10TB，最后产出10TB，网络消耗非常大
 5. 分布式系统要用到很多的网络通信，网络这一块很重要

## Lecture 2 RPC and Threads
 1. MIT推荐书籍**Effective Go**
 2. ```go run -race xxx.go```可以用来进行竞争检查
