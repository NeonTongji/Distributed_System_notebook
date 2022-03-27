# 1. Intro

### 1.1 why

1. 通过并行扩容 to increase capacity via parallel processing
2. 通过副本保证容错性 to tolerate faults via replication
3. to match distribution of physical devices e.g. sensors
4.  通过隔离保证安全性 to achieve security via isolation

    ![image-20220327165418436](https://tva1.sinaimg.cn/large/e6c9d24ely1h0ojkac78jj21ht0u0jvx.jpg)

### 1.2 History

* 1980s local area networks
* 1990s Database, big webset, websearch, shopping
* 2000s cloud computing, 不在本地计算，去远程云端计算
* Now, blooming!

### 1.3 Challenges

* many concurrent parts
* must deal with partial failure
* triky realize the performance potential

### 1.4 Labs

* Lab1: 分布式大数据框架 distributed big-data framework (like MapReduce)
* Lab2: 利用副本保证容错性 fault tolerance library using replication (Raft)
* Lab3: 实现简单的可容错的数据库 a simple fault-tolerant database
* Lab4: 通过分片实现可扩容的数据库 scalable database performance via sharding

### 1.5 Goal

将分布式系统的复杂性对用户隐藏起来

### 1.6 Topic

#### 1.6.1 容错性 Fault Tolerance

```
// 通过副本集实现高可用1000s of servers, big network -> always something broken    We'd like to hide these failures from the application.    "High availability": service continues despite failures  Big idea: replicated servers.    If one server crashes, can proceed using the other(s).    Labs 2 and 3
```

#### 1.6.2 一致性 Consistency

```
// 副本很难保证一致性general-purpose infrastructure needs well-defined behavior.    E.g. "Get(k) yields the value from the most recent Put(k,v)."  Achieving good behavior is hard!    "Replica" servers are hard to keep identical.
```

#### 1.6.3 高性能 performance

```
The goal: scalable throughput    Nx servers -> Nx total throughput via parallel CPU, disk, net.  Scaling gets harder as N grows:    Load imbalance.    Slowest-of-N latency.    Some things don't speed up with N: initialization, interaction.  Labs 1, 4
```

#### 1.6.4 协调 Tradeoff

容错性、一致性和高性能有悖，容错性和一致性需要集群间的通信，通信就会导致很慢，很多设计通过弱一致性来保证速度

```
Fault-tolerance, consistency, and performance are enemies.  Fault tolerance and consistency require communication    e.g., send data to backup    e.g., check if my data is up-to-date    communication is often slow and non-scalable  Many designs provide only weak consistency, to gain speed.    e.g. Get() does *not* yield the latest Put()!    Painful for application programmers but may be a good trade-off.  We'll see many design points in the consistency/performance spectrum.
```

#### 1.6.5 实现 Implementation

```
RPC, threads, concurrency control, configuration.  The labs...
```

### 1.7 Case Study: MapReduce

*   MapReduce概览

    * input
    * splitting 划分多个子任务
    * map 为KV
    * shuffling
    * sort 排序
    * reduce

    ![image-20220327165830494](https://tva1.sinaimg.cn/large/e6c9d24ely1h0ojnykqf3j21g50u0436.jpg)

    ![image-20220327165950043](https://tva1.sinaimg.cn/large/e6c9d24ely1h0ojpblsu0j21if0u0439.jpg)

    ![image-20220327170156636](https://tva1.sinaimg.cn/large/e6c9d24ely1h0ojrrcwmdj21fu0u0q6e.jpg)

    * Map: 并行初步处理转化数据
    * Reduce：并行对Map结果聚合汇总
    * Application Master: 追踪每个MapRduce任务
    * Node Mananger: 追踪每个map任务

    ![image-20220327170417193](https://tva1.sinaimg.cn/large/e6c9d24ely1h0oju8gie3j21hk0u0go3.jpg)

    * 提交任务
    * 任务分布： master把任务分给不同的node节点
    * Node协调
    * 任务重提交：如果DataNode执行失败，则提交给另一个可用node
    * 状态：ResourceManager收集最终信息，告诉客户端任务执行成功与否
*   MapReduce隐藏的细节

    ```
      （1）给服务器发送代码 sending app code to servers  （2）跟踪已经完成的任务 tracking which tasks have finished  （3）把Maps处理后的中间数据给Reduces "shuffling" intermediate data from Maps to Reduces  （4）对各服务器负载均衡 balancing load over servers  （5）错误后恢复 recovering from failures
    ```

### 1.8 限制MR性能的是什么？

networks，数据在不同cluster之间传输损耗了性能

### 1.9 MR如何减小网络使用？

Map任务就在储存数据的本机执行，Reduces才从网络中获取Maps产生的数据

```
Coordinator tries to run each Map task on GFS server that stores its input.    All computers run both GFS and MR workers    So input is read from local disk (via GFS), not over network.  Intermediate data goes over network just once.    Map worker writes to local disk.    Reduce workers read from Map worker disks over the network.    Storing it in GFS would require at least two trips over the network.  Intermediate data partitioned into files holding many keys.    R is much smaller than the number of keys.    Big network transfers are more efficient.
```

### 1.10 MR 如何实现负载均衡?

木桶的容量取决于最短的那跟，如果1个服务器还没有完成计算，其他服务器都要等他。怎么解决呢？

协调者将新的任务分配给已经完成之前计算的服务器

```
Wasteful and slow if N-1 servers have to wait for 1 slow server to finish.  But some tasks likely take longer than others.  Solution: many more tasks than workers.    Coordinator hands out new tasks to workers who finish previous tasks.    So no task is so big it dominates completion time (hopefully).    So faster servers do more tasks than slower ones, finish abt the same time.
```

### 1.11 MR如何实现容错性

中途失败怎么办，MR需要重头计算吗？

答案是否定的，MR只用重新计算失败的这次

```
 I.e. what if a worker crashes during a MR job?  We want to hide failures from the application programmer!  Does MR have to re-run the whole job from the beginning?    Why not?  MR re-runs just the failed Map()s and Reduce()s.    Suppose MR runs a Map twice, one Reduce sees first run's output,      another Reduce sees the second run's output?    Correctness requires re-execution to yield exactly the same output.    So Map and Reduce must be pure deterministic functions:      they are only allowed to look at their arguments/input.      no state, no file I/O, no interaction, no external communication.  What if you wanted to allow non-functional Map or Reduce?    Worker failure would require whole job to be re-executed,      or you'd need to roll back to some kind of global checkpoint.
```

### 1.12 Crash Recovery

Map失败了，发现map服务器ping不通后，map任务应该被重新在其他机器上执行

Reduce失败了 在其他机器上重新完成reduce任务

```
 * a Map worker crashes:    coordinator notices worker no longer responds to pings    coordinator knows which Map tasks ran on that worker      those tasks' intermediate output is now lost, must be re-created      coordinator tells other workers to run those tasks    can omit re-running if all Reduces have fetched the intermediate data  * a Reduce worker crashes:    finished tasks are OK -- stored in GFS, with replicas.    coordinator re-starts worker's unfinished tasks on other workers.
```

\
