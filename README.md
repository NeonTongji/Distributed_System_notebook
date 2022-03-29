# Lab detail

## 1 实现分布式MapReduce

包含两个programs：coordinator , workers

本项目不需要部署在多个机器上，部署在自己电脑上就可以。

* [ ] works通过RPC和coordinator通信
* [ ] 每个worker向coordinator索要task，通过读取1个或多个文件来执行任务, 并将输出写入到1个或多个文件
* [ ] coordinator需要监视每个worker是否在合理时间内完成任务（本次lab为10s），如果超时，则将任务分配给其他机器执行

<!---->

* main/mrcoordinator.go ： 实现master
* main/mrworker.go: 实现slave
* mr/rpc.go:&#x20;

## 2 提示

* 修改mr/worker.go的worker()方法，让worker给coordinator发送RPC索要任务，修改后的coordinator对woker（）用一个还没开始的任务的文件名作出确认，worker会读取该文件名，然后调用Map方法，即mrsequential.go
* .so文件会在运行时加载Map、Reduce方法
* 如果修改了mr/目录，则MapReduce插件需要重新build，命令为

```bash
go build -race -buildmode=plugin ../mrapps/wc.go
```

* 所有的worker共享一个文件系统。如果是部署在多台机器上时，则需要GFS的文件系统
* 中间文件名称规范为：mr-X-Y, X代表Map任务数量，Y代表Reduce任务数量
* worker的map代码需要存放中间kv对，之后作为reduce任务的输入。可以用Go语言的encoding/json包, 对文件写入json格式代码为

```go
  enc := json.NewEncoder(file)
  for _, kv := ... {
    err := enc.Encode(&kv)
```

读取代码为

```go
 dec := json.NewDecoder(file)
  for {
    var kv KeyValue
    if err := dec.Decode(&kv); err != nil {
      break
    }
    kva = append(kva, kv)
  }
```

* worker中的map方法可以用其中的ihash(key)方法为给定的key选取一个reduce任务
* mrsequential.go中有代码可以用来读取map输入文件，还有代码可以对map和reduce中间文件进行排序，并将reduce的输出结果储存到文件中
* coordinator， 充当了一个RPC服务器，需要支持并发，所以要对共享数据进行上锁
* go语言race detector命令可以测试

```
go build -race
go run -race
test-mr.sh
```

* workers经常需要等待其他worker，因为reduce任务必须要在所有的map任务完成后才能进行。wokers可以周期性地给coordinator发送请求索要任务，每次请求之间需要睡眠一段时间。或者是或者是每个线程RPC handler处于waiting，而coordinator中还在处理RPC
* 超过10s任务还没有完成，就假设worker已经宕机（虽然可能没有宕机），需要把任务教给别的worker处理
* 为了保证写文件到一半时发生宕机，MapReduce paper中用temporary文件暂存，并在文件完全写完后原子性地对其重命名。可以用ioutil.Templfie创建temporary文件，并用os.Rename对其原子性重命名
* 如果出错了或者想要查看中间文件或是输出结果，可以查看test-mr.sh
* test-mr-many.sh输入测试次数，多次进行测试。如果并行使用test-mr.sh容易产生端口冲突
* Go RPC只会发送首字母大写的结构体，子结构体的字段也要大写
* 把指针指向RPC的reply struct时，\*reply内存地址应该为空

```
  reply := SomeType{}
  call(..., &reply)
```

## 3 Rules

* map阶段需要把中间keys分到n个buckets中，即nReduce, reduce task的任务数量，也是main/mrcoordinates传给MakeCoordinator()的参数。所以每个mapper需要创建n个中间文件给reducer消费
* worker中需要把第X个reduce task的结果写到mr-out-X文件中
* 一个mr-out-X文件中每一行都是一个Reduce结果，每行需要用go中“%v %v”格式输出，并要用键值对的形式展示。
* worker应该把Map的中间结果存在当前目录的文件里，之后再读取作为Reduce的输入
* mr/coordinator.go应该实现Done（）方法，当MapReduce任务完成的时候返回true，然后mrcoordinator就要退出
* 当job完全结束，worker进程应该退出。实现方法是使用call()的返回值：如果worker未能和coordinator建立联系，它可以假设coordinator已经完成工作并退出，因此worker可以终止。
