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
        - Replication: identical servers 推荐
     - Strong Consistency: 非常expensive, 不推荐 10ms网络延迟communication cost, 能丢失百万条指令
     - **Weak** Consistency: 推荐
 4. MapReduce:
    - Reduce这一块要用到网络通信，网络通信要消耗带宽，如果有10TB文件要MapReduce，则Reduce阶段要在网络中发送10TB，最后产出10TB，网络消耗非常大
 5. 分布式系统要用到很多的网络通信，网络这一块很重要

## Lecture 2&3 RPC and Threads & GFS
 1. MIT推荐书籍**Effective Go**
 2. ```go run -race xxx.go```可以用来进行竞争检查
 ```go
 ch := make(chan []string)
 for urls := range ch{
    update ch // 这个for可能永远都不会停止
 }
 ```
 ![bad-repl](./imgs/bad-repl.png)  
 S1和S2处理C1、C2请求的(时间)顺序不同，出现不同的结果，没有consistency，所以是bad design  
 1. GFS是在一个Data center里，而不是全球都有around the world
 2. GFS是Big sequential access, 就是GB、TB大文件，而不是random的  
 3. GFS是基于Master Worker的，Master DATA见下图
 4. version 表示最新的版本 write的时候要比较，然后在最新的版本append
 
 ![master-data](./imgs/master-data.png)
   
 以下是GFS write时的操作，左边INCREMENT V#是指Master increment, 右边如果primary returns "no" to client, 则client会继续发起这个写请求
 直到成功为止，Google好像并没有提不成功的情况  
 ![gfs-write](./imgs/gfs-write.png)  
  - **GFS没有Strong Consistency, 所以存的东西可能丢失、重复！**
  - 如果Pimary挂了，在Secondary选举为新的Primary后，要进行同步Sync, 因为可能有Last set operation有的做了有的没做
  - GFS使用单一节点作为Master，其实是有问题的，出现Out of Memory RAM，所有东西都在内存里，加内存不够
   
 ## Lecture 4 Primary-Backup Replication 主从复制
  1. **State transfer**: Primary sends state to the secondary(Sync) e.g Send memory to the backup.
      - sends **memory**
  2. **Replicated State Mache**: Assuming state transfer is deterministic unless there are external factors. Thus, Primary do not send state, instead , it sends
  external factors to the secondary (backup).
      - sends **operations from client**
  3. What state? 主从复制考虑的问题
      - Primary/Backup Sync 主从同步
      - Cut-over 主从转换，比如主机挂了需要提升一个新的主机
      - Anomalies 外界不应该感知到主机挂了
      - New replicas 备份从挂了，也需要一个新的备份
  4. 一般情况下是Primary收到一条指令(packet?)，然后发给Backup也执行这条指令(packet)，但是有一些指令是weird instructions比如获取当前时间，获得进程ID，随机数之类的，
  在主和从会产生不同的结果，这种情况下从需要等主执行完后将结果告诉它
  5. **Output Rule**: Primary can not generate output(ACK) to the client until all the backups generate output to the primary.
  6. 一般情况下，client和Primary以及Secondary连接使用**TCP协议**，Primary把TCP包转给Secondary，回复ack时会把**tcp Sequence number**带上，所以
  如果client收到两份ack(primary and secondary）通过查看Seq，会把第二个Secondary发的给drop掉
  7. **Split Brain**(脑裂)的解决：让第三方authority来确定谁是Primary, 有一个TEST-AND-SET Server，让左脑和右脑同时发送test-and-set命令给这个Server，
  Server先收到谁的命令就让谁set为真正的Primary.
   
 ## Lecture 5 Concurrency in Go
```go
// 在for中使用goroutine的时候，如果用到i，要用go func(x int)的形式将i传进去
// 因为for会把i的值改掉，因此当goroutine运行到这一行sendRPC(x or i)这一行时
// i会变成for更改后的值``
func main() {
	var wg sync.WaitGroup
	for i := 0; i < 5; i++ {
		wg.Add(1)
		go func(x int) {
			sendRPC(x)
			wg.Done()
		}(i)
	}
	wg.Wait()
}

func sendRPC(i int) {
	println(i)
}

```
接下来是用go做一些周期性的事情, periodic
```go
// periodic模型
var done bool
var mu sync.Mutex

func main() {
	time.Sleep(1 * time.Second)
	println("started")
	go periodic()
	time.Sleep(5 * time.Second)
	mu.Lock()
	done = true
	mu.Unlock()
	println("cancelled")
	time.Sleep(3 * time.Second) // observe no output
}

func periodic() {
	for {
		println("tick")
		time.Sleep(1 * time.Second)
		mu.Lock()
		if done {
			// 这是个bug, 因为return前没有Unlock()!加上
			mu.Unlock()
			return
		}
		mu.Unlock()
	}
}

```
defer小知识
```go
func main() {
	println("started")
	defer println("1")
	defer println("2")
	defer println("3")
	defer println("4")
	defer println("5")
}
// 这段代码会打印 defer小知识：按defer的顺序理解为入栈，最后出栈
started
5
4
3
2
1
```
Busy waiting以及解决(sleep等待或者使用condition)
```go
// 首先来看这一段代码, 这段代码为什么不行，就是因为它一直在忙等待(busy waiting)，for一直尝试获取锁
// 这样会带来大量的cpu消耗，所以一定要避免忙等busy waiting
for {
    mu.Lock()
    if count >=  || finished == 10{
    	break
    }
    mu.Unlock()
}
// do something
mu.Unlock()
```
第一种解决方案：加入time.Sleep()等待
```go
// 这种方法会有magic number，但是可以解决忙等的问题
for {
    mu.Lock()
    if count >=  || finished == 10{
    	break
    }
    mu.Unlock()
    time.Sleep(50 * time.Millisecond)
}
```
第二种解决方案：使用Condition(推荐)
```go
// Condition broadcast wait类似于signal和wait, 区别是signal用于唤醒一个
// cond.Wait()的时候会讲这个加入到一个wait list里然后等待broadcast
// cond要绑定一个mutex
cond := sync.NewCond(&mu)
for i := 0 ...{
    go func(){
    	vote := requestVote()
	mu.Lock()
	defer mu.Unlock()
	if vote{
	    count++
	}
	finished++
	cond.Broadcast() // broadcast一定要在Unlock()操作之前
    }()
}
mu.Lock()
for count < 5 && finished != 10{ // 这里要是false的判断
    cond.Wait()
}
// do something
mu.Unlock()
```
Condition模版：
```go
mu.Lock()
// do something that might affect the condition
cond.Broadcast()
mu.Unlock()

----
mu.Lock()
while condition == false{
    cond.Wait()
}
// new condition is true, and we have the lock
// do something
mu.Unlock()
```
