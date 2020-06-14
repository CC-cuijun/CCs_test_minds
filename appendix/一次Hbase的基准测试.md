# Hbase基准测试

## Hbase测试工具

### hbase自带测试工具类说明

org.apache.hadoop.hbase.PerformanceEvaluation类是hbase官方提供的测试工具类，提供多种测试方式，使用方式如下：
<pre>
Usage: java org.apache.hadoop.hbase.PerformanceEvaluation \
  <OPTIONS> [-D<property=value>]* <command> <nclients>

Options:
nomapred        Run multiple clients using threads (rather than use mapreduce)
rows            Rows each client runs. Default: One million
size            Total size in GiB. Mutually exclusive with --rows. Default: 1.0.
sampleRate      Execute test on a sample of total rows. Only supported by randomRead. Default: 1.0
traceRate       Enable HTrace spans. Initiate tracing every N rows. Default: 0
table           Alternate table name. Default: 'TestTable'
multiGet        If >0, when doing RandomRead, perform multiple gets instead of single gets. Default: 0
compress        Compression type to use (GZ, LZO, ...). Default: 'NONE'
flushCommits    Used to determine if the test should flush the table. Default: false
writeToWAL      Set writeToWAL on puts. Default: True
autoFlush       Set autoFlush on htable. Default: False
oneCon          all the threads share the same connection. Default: False
presplit        Create presplit table. Recommended for accurate perf analysis (see guide).  Default: disabled
inmemory        Tries to keep the HFiles of the CF inmemory as far as possible. Not guaranteed that reads are always served from memory.  Default: false
usetags         Writes tags along with KVs. Use with HFile V3. Default: false
numoftags       Specify the no of tags that would be needed. This works only if usetags is true.
filterAll       Helps to filter out all the rows on the server side there by not returning any thing back to the client.  Helps to check the server side performance.  Uses FilterAllFilter internally.
latency         Set to report operation latencies. Default: False
bloomFilter      Bloom filter type, one of [NONE, ROW, ROWCOL]
valueSize       Pass value size to use: Default: 1024
valueRandom     Set if we should vary value size between 0 and 'valueSize'; set on read for stats on size: Default: Not set.
valueZipf       Set if we should vary value size between 0 and 'valueSize' in zipf form: Default: Not set.
period          Report every 'period' rows: Default: opts.perClientRunRows / 10
multiGet        Batch gets together into groups of N. Only supported by randomRead. Default: disabled
addColumns      Adds columns to scans/gets explicitly. Default: true
replicas        Enable region replica testing. Defaults: 1.
splitPolicy     Specify a custom RegionSplitPolicy for the table.
randomSleep     Do a random sleep before each get between 0 and entered value. Defaults: 0
columns         Columns to write per row. Default: 1
caching         Scan caching to use. Default: 30

Note: -D properties will be applied to the conf used.
  For example:
   -Dmapreduce.output.fileoutputformat.compress=true
   -Dmapreduce.task.timeout=60000

Command:
filterScan      Run scan test using a filter to find a specific row based on it's value (make sure to use --rows=20)
randomRead      Run random read test
randomSeekScan  Run random seek and scan 100 test
randomWrite     Run random write test
scan            Run scan test (read every row)
scanRange10     Run random seek scan with both start and stop row (max 10 rows)
scanRange100    Run random seek scan with both start and stop row (max 100 rows)
scanRange1000   Run random seek scan with both start and stop row (max 1000 rows)
scanRange10000  Run random seek scan with both start and stop row (max 10000 rows)
sequentialRead  Run sequential read test
sequentialWrite Run sequential write test

Args:
nclients        Integer. Required. Total number of clients (and HRegionServers)
                 running: 1 <= value <= 500
Examples:
To run a single evaluation client:
$ bin/hbase org.apache.hadoop.hbase.PerformanceEvaluation sequentialWrite 1
</pre>

### 该工具类存在的问题
   经查源码，该工具中randomWrite（随机写入）方式实际为rowkey的写入条数随机取，实际上会比期望的插入条数少很多，且生成的rowkey为顺序递增，不能均衡写入各个region，存在热点问题，不能满足我们的业务模拟；
   sequentialWrite（顺序写入）方式rowkey的生成方式为顺序递增，不能均衡写入各个region，同样存在热点问题，不能满足我们的业务模拟；
### 工具类定制化修改
   对该工具类中sequentialWrite方式的rowkey生成方式做了修改，修改rowkey的生成规则为vin码+纳秒时间戳，为了消除热点问题，只选择1000个vin码，循环生成rowkey（模拟1000个车辆上报），为了不影响存储时间的统计，将整个rowkey预先放入内存中，在for循环内部直接读取rowkey，提高效率。
   该工具基于hbase 1.1.2版本工具类修改，不同的版本之间可能存在差异，需要针对不同的版本做定制化修改；
### 工具类的部署：
   将该工具包hbase-server-1.1.2-tests.jar,放到hbase的lib目录中，替换原有的hbase-server-1.1.2-tests.jar包，建议在单独的一个服务器上部署该hbase包；
### 工具类的使用：
使用方式与原命令保持一致，注意修改的是sequentialWrite方式，命令如下：

> ./hbase org.apache.hadoop.hbase.PerformanceEvaluation --nomapred --rows=10000000 --table=test1 --valueSize=20 --compress=LZO  --flushCommits=false --columns=8 sequentialWrite 1

### 命令解读：
<pre>
-- nomapred 使用线程，不使用mapreduce，模拟实际业务的使用方式
-- rows=10000000 插入1000万条数据
-- table=test1 插入的表名
-- valueSize=20 设置values值为20bytes，模拟业务的数据量
-- compress=LZO 设置开启LZO，模拟实际业务的使用方式
-- flushCommits=false 关闭flushcommits（这里有一个坑：官方给的使用说明中默认写的false，但经查代码实际给的是true，所以此处一定要设置为false）
-- columns=8 设置column为8，模拟实际业务的使用方式
-- sequentialWrite 设置为顺序写入（实际已变为随机写入）1 为开启一个hbase-client
</pre>
* 工具注意事项，由于将rowkey事先放入了内存中，该工具对内存的需求很大，在使用该工具包时注意观察客户端服务器的内存消耗情况（设置三个hbase-client每个client写入1000万条数据，会占用6个g的内存），及时调整测试策略，建议部署在内存充足的服务器上。  

### 测试服务器配置信息  

|节点|服务器类型|作用|cpu |内存|
| :----: | :----: | :----: |:----: | :----: |
|master1|     物理机 | NameNode/SecondaryNameNode/HMaster | CPU*8| 16GB|
|master2|     物理机 | DataNode/HRegionServer             |CPU*8| 16GB|
|slave1 |     物理机 | DataNode/HRegionServer             |CPU*8| 16GB|
|slave3 |     物理机 | DataNode/HRegionServer                      |CPU*8| 16GB|

### 原生hbase-1.1.2版本测试结果  

> ./hbase org.apache.hadoop.hbase.PerformanceEvaluation --nomapred --rows=10000000 --table=test1 --valueSize=20 --compress=LZO  --flushCommits=false --columns=8 sequentialWrite 1  

测试结果:  
|保存条数  |花费时间ms|平均写入速度|99.99%写入耗时ms|DataNode个数|Region划分策略|memstore配置|writeToWAL|CPU |其他配置 |备注|
| :----: | :----: | :----: |:----: | :----: |:----: |:----: |:----: |:----: |:----: |:----: |
|10000000*3|  453985   |   22k/s*3  |   51.078       |            3|            10g|        512MB|      true  |  75% | 3 clients||  

Hbase配置项：
```
region划分策略：10g
memstore：512MB
hbase.hstore.blockingStoreFiles：100
flush上下限值：0.75
```

### hdp2.5.3.0-37版本测试

> ./hbase org.apache.hadoop.hbase.PerformanceEvaluation --nomapred --rows=10000000 --table=test1 --valueSize=20 --compress=LZO  --flushCommits=false --columns=8 sequentialWrite 1

测试结果：
|保存条数|花费时间ms|平均写入速度|99.99%写入耗时ms|DataNode个数|Region划分策略|memstore配置|writeToWAL|CPU|其他配置|备注|
| :----: | :----: | :----: |:----: | :----: |:----: |:----: |:----: |:----: |:----: |:----: |
|10000000*3|  465524   |   21k/s*3  |   61.391       |            3|            10g|        255MB|      true  |  70% | 3 clients||

## 测试过程记录

### 第一轮测试

跑一段时间后，Hbase报错：
```
2017-11-13 15:43:34.205 [htable-pool331929-t1] INFO  o.a.hadoop.hbase.client.AsyncProcess - #331929, table=LocationData, attempt=10/35 failed=200ops, last exception: org.apache.hadoop.hbase.RegionTooBusyException: org.apache.hadoop.hbase.RegionTooBusyException: Above memstore limit, regionName=LocationData,XXXL3BXXXH6128349_\x7F\xFF\xFE\xA0K\xD7\xB8\xA2_POSITIONANDSPEED_,1510557386775.b50b404296ab0f2493f33d1b7751de2b., server=tslave3.tsptest,16020,1510496758359, memstoreSize=537936880, blockingMemStoreSize=536870912
    at org.apache.hadoop.hbase.regionserver.HRegion.checkResources(HRegion.java:3587)
    at org.apache.hadoop.hbase.regionserver.HRegion.batchMutate(HRegion.java:2792)
    at org.apache.hadoop.hbase.regionserver.HRegion.batchMutate(HRegion.java:2743)
    at org.apache.hadoop.hbase.regionserver.RSRpcServices.doBatchOp(RSRpcServices.java:692)
    at org.apache.hadoop.hbase.regionserver.RSRpcServices.doNonAtomicRegionMutation(RSRpcServices.java:654)
    at org.apache.hadoop.hbase.regionserver.RSRpcServices.multi(RSRpcServices.java:2031)
    at org.apache.hadoop.hbase.protobuf.generated.ClientProtos$ClientService$2.callBlockingMethod(ClientProtos.java:32213)
    at org.apache.hadoop.hbase.ipc.RpcServer.call(RpcServer.java:2114)
    at org.apache.hadoop.hbase.ipc.CallRunner.run(CallRunner.java:101)
    at org.apache.hadoop.hbase.ipc.RpcExecutor.consumerLoop(RpcExecutor.java:130)
    at org.apache.hadoop.hbase.ipc.RpcExecutor$1.run(RpcExecutor.java:107)
    at java.lang.Thread.run(Thread.java:745)
on tslave3.tsptest,16020,1510496758359, tracking started null, retrying after=10071ms, replay=200ops
2017-11-13 15:43:34.225 [htable-pool331930-t1] INFO  o.a.hadoop.hbase.client.AsyncProcess - #331930, table=LocationData, attempt=10/35 failed=200ops, last exception: org.apache.hadoop.hbase.RegionTooBusyException: org.apache.hadoop.hbase.RegionTooBusyException: Above memstore limit, regionName=LocationData,XXXL3BXXXH6128349_\x7F\xFF\xFE\xA0K\xD7\xB8\xA2_POSITIONANDSPEED_,1510557386775.b50b404296ab0f2493f33d1b7751de2b., server=tslave3.tsptest,16020,1510496758359, memstoreSize=537936880, blockingMemStoreSize=536870912
    at org.apache.hadoop.hbase.regionserver.HRegion.checkResources(HRegion.java:3587)
    at org.apache.hadoop.hbase.regionserver.HRegion.batchMutate(HRegion.java:2792)
    at org.apache.hadoop.hbase.regionserver.HRegion.batchMutate(HRegion.java:2743)
    at org.apache.hadoop.hbase.regionserver.RSRpcServices.doBatchOp(RSRpcServices.java:692)
    at org.apache.hadoop.hbase.regionserver.RSRpcServices.doNonAtomicRegionMutation(RSRpcServices.java:654)
    at org.apache.hadoop.hbase.regionserver.RSRpcServices.multi(RSRpcServices.java:2031)
    at org.apache.hadoop.hbase.protobuf.generated.ClientProtos$ClientService$2.callBlockingMethod(ClientProtos.java:32213)
    at org.apache.hadoop.hbase.ipc.RpcServer.call(RpcServer.java:2114)
    at org.apache.hadoop.hbase.ipc.CallRunner.run(CallRunner.java:101)
    at org.apache.hadoop.hbase.ipc.RpcExecutor.consumerLoop(RpcExecutor.java:130)
    at org.apache.hadoop.hbase.ipc.RpcExecutor$1.run(RpcExecutor.java:107)
    at java.lang.Thread.run(Thread.java:745)
on tslave3.tsptest,16020,1510496758359, tracking started null, retrying after=10081ms, replay=200ops
2017-11-13 15:43:34.285 [pool-8-thread-2] INFO  o.a.hadoop.hbase.client.AsyncProcess - #331929, waiting for some tasks to finish. Expected max=0, tasksInProgress=10
2017-11-13 15:43:34.289 [pool-8-thread-4] INFO  o.a.hadoop.hbase.client.AsyncProcess - #331930, waiting for some tasks to finish. Expected max=0, tasksInProgress=10
```
> 调整regions划分策略为1g后，不再出现该错误

使用HBase自带测试工具，测试情况如下：

> ./hbase org.apache.hadoop.hbase.PerformanceEvaluation --nomapred --rows=6000000 --table=test1 --valueSize=50 --compress=LZO --flushCommits=true --autoFlush=true --columns=6 sequentialWrite 1

> 插入速度约15000条/s

Hbase配置调整：
```
memstore=512MB
regionsplit=5G
```
> 插入速度约18000条/s

Hbase配置调整：
```
flush上下限值：0.75
```
> 插入速度约 18000条/s  

脚本参数调整：
```
valueSize=20bytes
```
> 插入速度约：20000条/s

脚本策略调整：
```
4个client同时插入（4个节点每个启一个client）
```
> 插入数度约：8000*4条/s（rowkey存在重复）
```
> 4个client同时插入（1个节点启4个clinet）
```
> 插入数度约：7500*4t/s

Hbase配置项：
```
memstore=512MB
regionsplit=1G
flush上下限值：0.75
```

### 优化测试工具（hbase自带测试工具）：
#### 拆分region，情况如下：
测试策略：
> ./hbase org.apache.hadoop.hbase.PerformanceEvaluation --nomapred --rows=10000000 --table=test1 --valueSize=20 --compress=LZO  --flushCommits=true --autoFlush=true --columns=8 sequentialWrite 1 2>&1|tee result1.log

```
2017-11-20 16:13:56,218 INFO  [TestClient-0] hbase.PerformanceEvaluation: save total time: 390622ms
2017-11-20 16:13:56,218 INFO  [TestClient-0] hbase.PerformanceEvaluation: SequentialWriteTest latency log (microseconds), on 10000000 measures
2017-11-20 16:13:56,236 INFO  [TestClient-0] hbase.PerformanceEvaluation: SequentialWriteTest Min      = 3.0
2017-11-20 16:13:56,236 INFO  [TestClient-0] hbase.PerformanceEvaluation: SequentialWriteTest Avg      = 38.0374684
2017-11-20 16:13:56,236 INFO  [TestClient-0] hbase.PerformanceEvaluation: SequentialWriteTest StdDev   = 1037.8337000940983
2017-11-20 16:13:56,236 INFO  [TestClient-0] hbase.PerformanceEvaluation: SequentialWriteTest 50th     = 4.0
2017-11-20 16:13:56,236 INFO  [TestClient-0] hbase.PerformanceEvaluation: SequentialWriteTest 75th     = 4.0
2017-11-20 16:13:56,236 INFO  [TestClient-0] hbase.PerformanceEvaluation: SequentialWriteTest 95th     = 6.0
2017-11-20 16:13:56,236 INFO  [TestClient-0] hbase.PerformanceEvaluation: SequentialWriteTest 99th     = 9.0
2017-11-20 16:13:56,236 INFO  [TestClient-0] hbase.PerformanceEvaluation: SequentialWriteTest 99.9th   = 13092.974000000278
2017-11-20 16:13:56,236 INFO  [TestClient-0] hbase.PerformanceEvaluation: SequentialWriteTest 99.99th  = 43247.52429998759
2017-11-20 16:13:56,237 INFO  [TestClient-0] hbase.PerformanceEvaluation: SequentialWriteTest 99.999th = 75894.2680304132
2017-11-20 16:13:56,237 INFO  [TestClient-0] hbase.PerformanceEvaluation: SequentialWriteTest Max      = 824383.0
2017-11-20 16:13:56,237 INFO  [TestClient-0] hbase.PerformanceEvaluation: SequentialWriteTest valueSize after 0 measures
2017-11-20 16:13:56,237 INFO  [TestClient-0] hbase.PerformanceEvaluation: SequentialWriteTest Min      = 0.0
2017-11-20 16:13:56,237 INFO  [TestClient-0] hbase.PerformanceEvaluation: SequentialWriteTest Avg      = 0.0
2017-11-20 16:13:56,237 INFO  [TestClient-0] hbase.PerformanceEvaluation: SequentialWriteTest StdDev   = 0.0
2017-11-20 16:13:56,237 INFO  [TestClient-0] hbase.PerformanceEvaluation: SequentialWriteTest 50th     = 0.0
2017-11-20 16:13:56,237 INFO  [TestClient-0] hbase.PerformanceEvaluation: SequentialWriteTest 75th     = 0.0
2017-11-20 16:13:56,237 INFO  [TestClient-0] hbase.PerformanceEvaluation: SequentialWriteTest 95th     = 0.0
2017-11-20 16:13:56,237 INFO  [TestClient-0] hbase.PerformanceEvaluation: SequentialWriteTest 99th     = 0.0
2017-11-20 16:13:56,237 INFO  [TestClient-0] hbase.PerformanceEvaluation: SequentialWriteTest 99.9th   = 0.0
2017-11-20 16:13:56,237 INFO  [TestClient-0] hbase.PerformanceEvaluation: SequentialWriteTest 99.99th  = 0.0
2017-11-20 16:13:56,237 INFO  [TestClient-0] hbase.PerformanceEvaluation: SequentialWriteTest 99.999th = 0.0
2017-11-20 16:13:56,237 INFO  [TestClient-0] hbase.PerformanceEvaluation: SequentialWriteTest Max      = 0.0
2017-11-20 16:13:56,329 INFO  [TestClient-0] client.ConnectionManager$HConnectionImplementation: Closing zookeeper sessionid=0x15fb964db511289
2017-11-20 16:13:56,339 INFO  [TestClient-0] zookeeper.ZooKeeper: Session: 0x15fb964db511289 closed
2017-11-20 16:13:56,339 INFO  [TestClient-0-EventThread] zookeeper.ClientCnxn: EventThread shut down
2017-11-20 16:13:56,441 INFO  [TestClient-0] hbase.PerformanceEvaluation: Finished class org.apache.hadoop.hbase.PerformanceEvaluation$SequentialWriteTest in 390844ms at offset 0 for 10000000 rows (5.51 MB/s)
2017-11-20 16:13:56,442 INFO  [TestClient-0] hbase.PerformanceEvaluation: Finished TestClient-0 in 390844ms over 10000000 rows
```
#### --writeToWAL=false情况如下：
```
2017-11-20 16:27:15,396 INFO  [TestClient-0] hbase.PerformanceEvaluation: save total time: 289756ms
2017-11-20 16:27:15,396 INFO  [TestClient-0] hbase.PerformanceEvaluation: SequentialWriteTest latency log (microseconds), on 10000000 measures
2017-11-20 16:27:15,425 INFO  [TestClient-0] hbase.PerformanceEvaluation: SequentialWriteTest Min      = 3.0
2017-11-20 16:27:15,426 INFO  [TestClient-0] hbase.PerformanceEvaluation: SequentialWriteTest Avg      = 27.9457851
2017-11-20 16:27:15,426 INFO  [TestClient-0] hbase.PerformanceEvaluation: SequentialWriteTest StdDev   = 723.3358991881652
2017-11-20 16:27:15,426 INFO  [TestClient-0] hbase.PerformanceEvaluation: SequentialWriteTest 50th     = 4.0
2017-11-20 16:27:15,426 INFO  [TestClient-0] hbase.PerformanceEvaluation: SequentialWriteTest 75th     = 4.0
2017-11-20 16:27:15,426 INFO  [TestClient-0] hbase.PerformanceEvaluation: SequentialWriteTest 95th     = 6.0
2017-11-20 16:27:15,426 INFO  [TestClient-0] hbase.PerformanceEvaluation: SequentialWriteTest 99th     = 10.0
2017-11-20 16:27:15,426 INFO  [TestClient-0] hbase.PerformanceEvaluation: SequentialWriteTest 99.9th   = 8402.983000000182
2017-11-20 16:27:15,426 INFO  [TestClient-0] hbase.PerformanceEvaluation: SequentialWriteTest 99.99th  = 29340.777399996296
2017-11-20 16:27:15,426 INFO  [TestClient-0] hbase.PerformanceEvaluation: SequentialWriteTest 99.999th = 75652.0573401018
2017-11-20 16:27:15,426 INFO  [TestClient-0] hbase.PerformanceEvaluation: SequentialWriteTest Max      = 335803.0
2017-11-20 16:27:15,427 INFO  [TestClient-0] hbase.PerformanceEvaluation: SequentialWriteTest valueSize after 0 measures
2017-11-20 16:27:15,427 INFO  [TestClient-0] hbase.PerformanceEvaluation: SequentialWriteTest Min      = 0.0
2017-11-20 16:27:15,427 INFO  [TestClient-0] hbase.PerformanceEvaluation: SequentialWriteTest Avg      = 0.0
2017-11-20 16:27:15,427 INFO  [TestClient-0] hbase.PerformanceEvaluation: SequentialWriteTest StdDev   = 0.0
2017-11-20 16:27:15,427 INFO  [TestClient-0] hbase.PerformanceEvaluation: SequentialWriteTest 50th     = 0.0
2017-11-20 16:27:15,427 INFO  [TestClient-0] hbase.PerformanceEvaluation: SequentialWriteTest 75th     = 0.0
2017-11-20 16:27:15,427 INFO  [TestClient-0] hbase.PerformanceEvaluation: SequentialWriteTest 95th     = 0.0
2017-11-20 16:27:15,427 INFO  [TestClient-0] hbase.PerformanceEvaluation: SequentialWriteTest 99th     = 0.0
2017-11-20 16:27:15,427 INFO  [TestClient-0] hbase.PerformanceEvaluation: SequentialWriteTest 99.9th   = 0.0
2017-11-20 16:27:15,427 INFO  [TestClient-0] hbase.PerformanceEvaluation: SequentialWriteTest 99.99th  = 0.0
2017-11-20 16:27:15,427 INFO  [TestClient-0] hbase.PerformanceEvaluation: SequentialWriteTest 99.999th = 0.0
2017-11-20 16:27:15,427 INFO  [TestClient-0] hbase.PerformanceEvaluation: SequentialWriteTest Max      = 0.0
2017-11-20 16:27:15,482 INFO  [TestClient-0] client.ConnectionManager$HConnectionImplementation: Closing zookeeper sessionid=0x25fb36413092426
2017-11-20 16:27:15,490 INFO  [TestClient-0] zookeeper.ZooKeeper: Session: 0x25fb36413092426 closed
2017-11-20 16:27:15,490 INFO  [TestClient-0-EventThread] zookeeper.ClientCnxn: EventThread shut down
2017-11-20 16:27:15,593 INFO  [TestClient-0] hbase.PerformanceEvaluation: Finished class org.apache.hadoop.hbase.PerformanceEvaluation$SequentialWriteTest in 289952ms at offset 0 for 10000000 rows (7.43 MB/s)
2017-11-20 16:27:15,593 INFO  [TestClient-0] hbase.PerformanceEvaluation: Finished TestClient-0 in 289952ms over 10000000 rows
```
#### 开启两个client
```2017-11-20 16:45:06,137 INFO  [TestClient-0] hbase.PerformanceEvaluation: save total time: 525269ms
2017-11-20 16:45:06,138 INFO  [TestClient-0] hbase.PerformanceEvaluation: SequentialWriteTest latency log (microseconds), on 10000000 measures
2017-11-20 16:45:06,165 INFO  [TestClient-0] hbase.PerformanceEvaluation: SequentialWriteTest Min      = 3.0
2017-11-20 16:45:06,165 INFO  [TestClient-0] hbase.PerformanceEvaluation: SequentialWriteTest Avg      = 51.4780589
2017-11-20 16:45:06,165 INFO  [TestClient-0] hbase.PerformanceEvaluation: SequentialWriteTest StdDev   = 1423.7366785986983
2017-11-20 16:45:06,165 INFO  [TestClient-0] hbase.PerformanceEvaluation: SequentialWriteTest 50th     = 4.0
2017-11-20 16:45:06,165 INFO  [TestClient-0] hbase.PerformanceEvaluation: SequentialWriteTest 75th     = 4.0
2017-11-20 16:45:06,165 INFO  [TestClient-0] hbase.PerformanceEvaluation: SequentialWriteTest 95th     = 6.0
2017-11-20 16:45:06,166 INFO  [TestClient-0] hbase.PerformanceEvaluation: SequentialWriteTest 99th     = 9.0
2017-11-20 16:45:06,166 INFO  [TestClient-0] hbase.PerformanceEvaluation: SequentialWriteTest 99.9th   = 16839.989000000118
2017-11-20 16:45:06,166 INFO  [TestClient-0] hbase.PerformanceEvaluation: SequentialWriteTest 99.99th  = 56513.950499991886
2017-11-20 16:45:06,166 INFO  [TestClient-0] hbase.PerformanceEvaluation: SequentialWriteTest 99.999th = 151244.40662014997
2017-11-20 16:45:06,166 INFO  [TestClient-0] hbase.PerformanceEvaluation: SequentialWriteTest Max      = 720978.0
2017-11-20 16:45:06,166 INFO  [TestClient-0] hbase.PerformanceEvaluation: SequentialWriteTest valueSize after 0 measures
2017-11-20 16:45:06,166 INFO  [TestClient-0] hbase.PerformanceEvaluation: SequentialWriteTest Min      = 0.0
2017-11-20 16:45:06,166 INFO  [TestClient-0] hbase.PerformanceEvaluation: SequentialWriteTest Avg      = 0.0
2017-11-20 16:45:06,166 INFO  [TestClient-0] hbase.PerformanceEvaluation: SequentialWriteTest StdDev   = 0.0
2017-11-20 16:45:06,166 INFO  [TestClient-0] hbase.PerformanceEvaluation: SequentialWriteTest 50th     = 0.0
2017-11-20 16:45:06,166 INFO  [TestClient-0] hbase.PerformanceEvaluation: SequentialWriteTest 75th     = 0.0
2017-11-20 16:45:06,166 INFO  [TestClient-0] hbase.PerformanceEvaluation: SequentialWriteTest 95th     = 0.0
2017-11-20 16:45:06,166 INFO  [TestClient-0] hbase.PerformanceEvaluation: SequentialWriteTest 99th     = 0.0
2017-11-20 16:45:06,167 INFO  [TestClient-0] hbase.PerformanceEvaluation: SequentialWriteTest 99.9th   = 0.0
2017-11-20 16:45:06,167 INFO  [TestClient-0] hbase.PerformanceEvaluation: SequentialWriteTest 99.99th  = 0.0
2017-11-20 16:45:06,167 INFO  [TestClient-0] hbase.PerformanceEvaluation: SequentialWriteTest 99.999th = 0.0
2017-11-20 16:45:06,167 INFO  [TestClient-0] hbase.PerformanceEvaluation: SequentialWriteTest Max      = 0.0
2017-11-20 16:45:06,335 INFO  [TestClient-0] client.ConnectionManager$HConnectionImplementation: Closing zookeeper sessionid=0x25fb3641309242a
2017-11-20 16:45:06,342 INFO  [TestClient-0] zookeeper.ZooKeeper: Session: 0x25fb3641309242a closed
2017-11-20 16:45:06,342 INFO  [TestClient-0-EventThread] zookeeper.ClientCnxn: EventThread shut down
2017-11-20 16:45:06,445 INFO  [TestClient-0] hbase.PerformanceEvaluation: Finished class org.apache.hadoop.hbase.PerformanceEvaluation$SequentialWriteTest in 525574ms at offset 0 for 10000000 rows (4.1 MB/s)
2017-11-20 16:45:06,445 INFO  [TestClient-0] hbase.PerformanceEvaluation: Finished TestClient-0 in 525574ms over 10000000 rows
2017-11-20 16:45:34,082 INFO  [TestClient-1] hbase.PerformanceEvaluation: save total time: 553135ms
2017-11-20 16:45:34,082 INFO  [TestClient-1] hbase.PerformanceEvaluation: SequentialWriteTest latency log (microseconds), on 10000000 measures
2017-11-20 16:45:34,101 INFO  [TestClient-1] hbase.PerformanceEvaluation: SequentialWriteTest Min      = 3.0
2017-11-20 16:45:34,101 INFO  [TestClient-1] hbase.PerformanceEvaluation: SequentialWriteTest Avg      = 54.2702524
2017-11-20 16:45:34,101 INFO  [TestClient-1] hbase.PerformanceEvaluation: SequentialWriteTest StdDev   = 1594.4939290755417
2017-11-20 16:45:34,101 INFO  [TestClient-1] hbase.PerformanceEvaluation: SequentialWriteTest 50th     = 4.0
2017-11-20 16:45:34,101 INFO  [TestClient-1] hbase.PerformanceEvaluation: SequentialWriteTest 75th     = 4.0
2017-11-20 16:45:34,101 INFO  [TestClient-1] hbase.PerformanceEvaluation: SequentialWriteTest 95th     = 6.0
2017-11-20 16:45:34,102 INFO  [TestClient-1] hbase.PerformanceEvaluation: SequentialWriteTest 99th     = 9.0
2017-11-20 16:45:34,102 INFO  [TestClient-1] hbase.PerformanceEvaluation: SequentialWriteTest 99.9th   = 18867.898000001092
2017-11-20 16:45:34,102 INFO  [TestClient-1] hbase.PerformanceEvaluation: SequentialWriteTest 99.99th  = 81734.68889998179
2017-11-20 16:45:34,102 INFO  [TestClient-1] hbase.PerformanceEvaluation: SequentialWriteTest 99.999th = 136740.60747014615
2017-11-20 16:45:34,102 INFO  [TestClient-1] hbase.PerformanceEvaluation: SequentialWriteTest Max      = 685969.0
2017-11-20 16:45:34,102 INFO  [TestClient-1] hbase.PerformanceEvaluation: SequentialWriteTest valueSize after 0 measures
2017-11-20 16:45:34,102 INFO  [TestClient-1] hbase.PerformanceEvaluation: SequentialWriteTest Min      = 0.0
2017-11-20 16:45:34,102 INFO  [TestClient-1] hbase.PerformanceEvaluation: SequentialWriteTest Avg      = 0.0
2017-11-20 16:45:34,102 INFO  [TestClient-1] hbase.PerformanceEvaluation: SequentialWriteTest StdDev   = 0.0
2017-11-20 16:45:34,102 INFO  [TestClient-1] hbase.PerformanceEvaluation: SequentialWriteTest 50th     = 0.0
2017-11-20 16:45:34,102 INFO  [TestClient-1] hbase.PerformanceEvaluation: SequentialWriteTest 75th     = 0.0
2017-11-20 16:45:34,102 INFO  [TestClient-1] hbase.PerformanceEvaluation: SequentialWriteTest 95th     = 0.0
2017-11-20 16:45:34,102 INFO  [TestClient-1] hbase.PerformanceEvaluation: SequentialWriteTest 99th     = 0.0
2017-11-20 16:45:34,102 INFO  [TestClient-1] hbase.PerformanceEvaluation: SequentialWriteTest 99.9th   = 0.0
2017-11-20 16:45:34,102 INFO  [TestClient-1] hbase.PerformanceEvaluation: SequentialWriteTest 99.99th  = 0.0
2017-11-20 16:45:34,102 INFO  [TestClient-1] hbase.PerformanceEvaluation: SequentialWriteTest 99.999th = 0.0
2017-11-20 16:45:34,102 INFO  [TestClient-1] hbase.PerformanceEvaluation: SequentialWriteTest Max      = 0.0
2017-11-20 16:45:34,192 INFO  [TestClient-1] client.ConnectionManager$HConnectionImplementation: Closing zookeeper sessionid=0x15fb964db511299
2017-11-20 16:45:34,206 INFO  [TestClient-1] zookeeper.ZooKeeper: Session: 0x15fb964db511299 closed
2017-11-20 16:45:34,206 INFO  [TestClient-1-EventThread] zookeeper.ClientCnxn: EventThread shut down
2017-11-20 16:45:34,307 INFO  [TestClient-1] hbase.PerformanceEvaluation: Finished class org.apache.hadoop.hbase.PerformanceEvaluation$SequentialWriteTest in 553359ms at offset 10000000 for 10000000 rows (3.89 MB/s)
2017-11-20 16:45:34,307 INFO  [TestClient-1] hbase.PerformanceEvaluation: Finished TestClient-1 in 553359ms over 10000000 rows
2017-11-20 16:45:34,308 INFO  [main] hbase.PerformanceEvaluation: [SequentialWriteTest] Summary of timings (ms): [525574, 553359]
2017-11-20 16:45:34,309 INFO  [main] hbase.PerformanceEvaluation: [SequentialWriteTest]    Min: 525574ms    Max: 553359ms    Avg: 539466ms
```
#### 开启两个client，并设置--writeToWAL=false：
```
2017-11-20 16:56:43,944 INFO  [TestClient-0] hbase.PerformanceEvaluation: save total time: 409123ms
2017-11-20 16:56:43,944 INFO  [TestClient-0] hbase.PerformanceEvaluation: SequentialWriteTest latency log (microseconds), on 10000000 measures
2017-11-20 16:56:43,962 INFO  [TestClient-0] hbase.PerformanceEvaluation: SequentialWriteTest Min      = 3.0
2017-11-20 16:56:43,963 INFO  [TestClient-0] hbase.PerformanceEvaluation: SequentialWriteTest Avg      = 39.8398979
2017-11-20 16:56:43,963 INFO  [TestClient-0] hbase.PerformanceEvaluation: SequentialWriteTest StdDev   = 1130.340946500589
2017-11-20 16:56:43,963 INFO  [TestClient-0] hbase.PerformanceEvaluation: SequentialWriteTest 50th     = 4.0
2017-11-20 16:56:43,963 INFO  [TestClient-0] hbase.PerformanceEvaluation: SequentialWriteTest 75th     = 5.0
2017-11-20 16:56:43,963 INFO  [TestClient-0] hbase.PerformanceEvaluation: SequentialWriteTest 95th     = 6.0
2017-11-20 16:56:43,963 INFO  [TestClient-0] hbase.PerformanceEvaluation: SequentialWriteTest 99th     = 10.0
2017-11-20 16:56:43,963 INFO  [TestClient-0] hbase.PerformanceEvaluation: SequentialWriteTest 99.9th   = 12296.98600000015
2017-11-20 16:56:43,963 INFO  [TestClient-0] hbase.PerformanceEvaluation: SequentialWriteTest 99.99th  = 52012.58079999685
2017-11-20 16:56:43,963 INFO  [TestClient-0] hbase.PerformanceEvaluation: SequentialWriteTest 99.999th = 115895.528550321
2017-11-20 16:56:43,963 INFO  [TestClient-0] hbase.PerformanceEvaluation: SequentialWriteTest Max      = 438113.0
2017-11-20 16:56:43,963 INFO  [TestClient-0] hbase.PerformanceEvaluation: SequentialWriteTest valueSize after 0 measures
2017-11-20 16:56:43,963 INFO  [TestClient-0] hbase.PerformanceEvaluation: SequentialWriteTest Min      = 0.0
2017-11-20 16:56:43,963 INFO  [TestClient-0] hbase.PerformanceEvaluation: SequentialWriteTest Avg      = 0.0
2017-11-20 16:56:43,963 INFO  [TestClient-0] hbase.PerformanceEvaluation: SequentialWriteTest StdDev   = 0.0
2017-11-20 16:56:43,963 INFO  [TestClient-0] hbase.PerformanceEvaluation: SequentialWriteTest 50th     = 0.0
2017-11-20 16:56:43,963 INFO  [TestClient-0] hbase.PerformanceEvaluation: SequentialWriteTest 75th     = 0.0
2017-11-20 16:56:43,963 INFO  [TestClient-0] hbase.PerformanceEvaluation: SequentialWriteTest 95th     = 0.0
2017-11-20 16:56:43,964 INFO  [TestClient-0] hbase.PerformanceEvaluation: SequentialWriteTest 99th     = 0.0
2017-11-20 16:56:43,964 INFO  [TestClient-0] hbase.PerformanceEvaluation: SequentialWriteTest 99.9th   = 0.0
2017-11-20 16:56:43,964 INFO  [TestClient-0] hbase.PerformanceEvaluation: SequentialWriteTest 99.99th  = 0.0
2017-11-20 16:56:43,964 INFO  [TestClient-0] hbase.PerformanceEvaluation: SequentialWriteTest 99.999th = 0.0
2017-11-20 16:56:43,964 INFO  [TestClient-0] hbase.PerformanceEvaluation: SequentialWriteTest Max      = 0.0
2017-11-20 16:56:44,113 INFO  [TestClient-0] client.ConnectionManager$HConnectionImplementation: Closing zookeeper sessionid=0x25fb3641309242c
2017-11-20 16:56:44,124 INFO  [TestClient-0] zookeeper.ZooKeeper: Session: 0x25fb3641309242c closed
2017-11-20 16:56:44,124 INFO  [TestClient-0-EventThread] zookeeper.ClientCnxn: EventThread shut down
2017-11-20 16:56:44,226 INFO  [TestClient-0] hbase.PerformanceEvaluation: Finished class org.apache.hadoop.hbase.PerformanceEvaluation$SequentialWriteTest in 409404ms at offset 0 for 10000000 rows (5.26 MB/s)
2017-11-20 16:56:44,227 INFO  [TestClient-0] hbase.PerformanceEvaluation: Finished TestClient-0 in 409404ms over 10000000 rows
2017-11-20 16:56:53,938 INFO  [TestClient-1] hbase.PerformanceEvaluation: save total time: 418508ms
2017-11-20 16:56:53,938 INFO  [TestClient-1] hbase.PerformanceEvaluation: SequentialWriteTest latency log (microseconds), on 10000000 measures
2017-11-20 16:56:53,974 INFO  [TestClient-1] hbase.PerformanceEvaluation: SequentialWriteTest Min      = 3.0
2017-11-20 16:56:53,975 INFO  [TestClient-1] hbase.PerformanceEvaluation: SequentialWriteTest Avg      = 40.8389075
2017-11-20 16:56:53,975 INFO  [TestClient-1] hbase.PerformanceEvaluation: SequentialWriteTest StdDev   = 1201.1793532252063
2017-11-20 16:56:53,975 INFO  [TestClient-1] hbase.PerformanceEvaluation: SequentialWriteTest 50th     = 4.0
2017-11-20 16:56:53,975 INFO  [TestClient-1] hbase.PerformanceEvaluation: SequentialWriteTest 75th     = 5.0
2017-11-20 16:56:53,975 INFO  [TestClient-1] hbase.PerformanceEvaluation: SequentialWriteTest 95th     = 6.0
2017-11-20 16:56:53,975 INFO  [TestClient-1] hbase.PerformanceEvaluation: SequentialWriteTest 99th     = 9.0
2017-11-20 16:56:53,975 INFO  [TestClient-1] hbase.PerformanceEvaluation: SequentialWriteTest 99.9th   = 12024.976000000257
2017-11-20 16:56:53,975 INFO  [TestClient-1] hbase.PerformanceEvaluation: SequentialWriteTest 99.99th  = 47703.199599999934
2017-11-20 16:56:53,975 INFO  [TestClient-1] hbase.PerformanceEvaluation: SequentialWriteTest 99.999th = 126867.98867023061
2017-11-20 16:56:53,975 INFO  [TestClient-1] hbase.PerformanceEvaluation: SequentialWriteTest Max      = 384722.0
2017-11-20 16:56:53,975 INFO  [TestClient-1] hbase.PerformanceEvaluation: SequentialWriteTest valueSize after 0 measures
2017-11-20 16:56:53,975 INFO  [TestClient-1] hbase.PerformanceEvaluation: SequentialWriteTest Min      = 0.0
2017-11-20 16:56:53,975 INFO  [TestClient-1] hbase.PerformanceEvaluation: SequentialWriteTest Avg      = 0.0
2017-11-20 16:56:53,975 INFO  [TestClient-1] hbase.PerformanceEvaluation: SequentialWriteTest StdDev   = 0.0
2017-11-20 16:56:53,975 INFO  [TestClient-1] hbase.PerformanceEvaluation: SequentialWriteTest 50th     = 0.0
2017-11-20 16:56:53,975 INFO  [TestClient-1] hbase.PerformanceEvaluation: SequentialWriteTest 75th     = 0.0
2017-11-20 16:56:53,975 INFO  [TestClient-1] hbase.PerformanceEvaluation: SequentialWriteTest 95th     = 0.0
2017-11-20 16:56:53,975 INFO  [TestClient-1] hbase.PerformanceEvaluation: SequentialWriteTest 99th     = 0.0
2017-11-20 16:56:53,976 INFO  [TestClient-1] hbase.PerformanceEvaluation: SequentialWriteTest 99.9th   = 0.0
2017-11-20 16:56:53,976 INFO  [TestClient-1] hbase.PerformanceEvaluation: SequentialWriteTest 99.99th  = 0.0
2017-11-20 16:56:53,976 INFO  [TestClient-1] hbase.PerformanceEvaluation: SequentialWriteTest 99.999th = 0.0
2017-11-20 16:56:53,976 INFO  [TestClient-1] hbase.PerformanceEvaluation: SequentialWriteTest Max      = 0.0
2017-11-20 16:56:54,052 INFO  [TestClient-1] client.ConnectionManager$HConnectionImplementation: Closing zookeeper sessionid=0x15fb964db51129b
2017-11-20 16:56:54,062 INFO  [TestClient-1] zookeeper.ZooKeeper: Session: 0x15fb964db51129b closed
2017-11-20 16:56:54,063 INFO  [TestClient-1-EventThread] zookeeper.ClientCnxn: EventThread shut down
2017-11-20 16:56:54,164 INFO  [TestClient-1] hbase.PerformanceEvaluation: Finished class org.apache.hadoop.hbase.PerformanceEvaluation$SequentialWriteTest in 418733ms at offset 10000000 for 10000000 rows (5.15 MB/s)
2017-11-20 16:56:54,164 INFO  [TestClient-1] hbase.PerformanceEvaluation: Finished TestClient-1 in 418733ms over 10000000 rows
2017-11-20 16:56:54,164 INFO  [main] hbase.PerformanceEvaluation: [SequentialWriteTest] Summary of timings (ms): [409404, 418733]
2017-11-20 16:56:54,165 INFO  [main] hbase.PerformanceEvaluation: [SequentialWriteTest]    Min: 409404ms    Max: 418733ms    Avg: 414068ms
```
### 第二轮测试
使用hbase自带测试工具，定制化修改rowkey的生成规则：vin码+纳秒时间戳；
模拟业务上报，使用1000个VIN码循环写入，8个column，value设置为20bytes;

> ./hbase org.apache.hadoop.hbase.PerformanceEvaluation --nomapred --rows=10000000 --table=test1 --valueSize=20 --compress=LZO  --flushCommits=true --autoFlush=true --columns=8 sequentialWrite 1

|保存条数|花费时间ms|平均写入速度|99.99%写入耗时ms|DataNode个数|Region划分策略|memstore配置|writeToWAL|CPU|内存|磁盘I/O|其他配置 |备注|
| :----: | :----: | :----: |:----: | :----: |:----: |:----: |:----: |:----: |:----: |:----: |:----: |:----: |
|10000000|  396528   |  25k/s     |   43.113       |            3|             1g|         512MB|      true | 40%  |     |        |          |     |
|10000000|  293303   |  35k/s     |   32.213       |            3|             1g|         512MB|      false| 40%  |     |        |          |     |
|10000000*2|  374142   |  26k/s*2   |   46.904       |            3|             1g|         512MB|      true | 55%  |     |        | 2 clients|     |
|10000000*4|  371640   |  26k/s*4   |   46.583       |            3|             1g|         512MB|      false | 80%  |     |        | 4 clients|     |
|10000000*4|         |            |                |            3|             1g|         512MB|      true  | 90%  |     |        | 4 clients|  该方式cpu利用率太高   |

脚本参数调整：
```
--flushCommits=false
--autoFlush=false
```
> ./hbase org.apache.hadoop.hbase.PerformanceEvaluation --nomapred --rows=10000000 --table=test1 --valueSize=20 --compress=LZO  --flushCommits=false --columns=8 sequentialWrite 1

|保存条数  |花费时间ms|平均写入速度|99.99%写入耗时ms|DataNode个数|Region划分策略|memstore配置|writeToWAL|CPU |内存|磁盘I/O|其他配置 |备注|
| :----: | :----: | :----: |:----: | :----: |:----: |:----: |:----: |:----: |:----: |:----: |:----: |:----: |
|10000000*4|    --     |            |                |            3|             1g|        512MB|      true  |  90% |      |        | 4 clients| cpu利用率太高    |
|10000000*2|  489570   |  20k/s*2   |   53.359       |            3|             1g|        512MB|      true  |  60% |      |        | 2 clients|     |
|10000000*3|  539062   |  18k/s*3   |   64.907       |            3|             1g|        512MB|      true  |  75% |      |        | 3 clients| 用另外一台虚拟机测试，计划调整到slave3上跑脚本|
|10000000*3|  644892   |  15k/s*3   |   74.981       |            3|             1g|        512MB|      true  |  75% |      |        | 3 clients| 每个节点上跑一个脚本，共3个    |
|10000000*2|  520377   |  19k/s*2   |   62.160       |            3|             1g|        512MB|      true  |  55% |      |        | 2 clients|     |

* 保存一段时间后，性能下降----原因未知

#### 清空数据重新测试：
<pre>
./hbase org.apache.hadoop.hbase.PerformanceEvaluation --nomapred --rows=10000000 --table=test1 --valueSize=20 --compress=LZO  --flushCommits=false --columns=8 sequentialWrite 1
</pre>
|保存条数  |花费时间ms|平均写入速度|99.99%写入耗时ms|DataNode个数|Region划分策略|memstore配置|writeToWAL|CPU |内存|磁盘I/O|其他配置 |备注|
| :----: | :----: | :----: |:----: | :----: |:----: |:----: |:----: |:----: |:----: |:----: |:----: |:----: |
|10000000  |  391789   |   25k/s    |   40.749       |            3|             1g|        512MB|      true  |  40% |      |        | 1 clients|     |
|10000000*2|  497258   |   20k/s*2  |   53.440       |            3|             1g|        512MB|      true  |  50% |      |        | 2 clients|   |
|10000000*3|  565870   |   17k/s*3  |   60.888       |            3|             1g|        512MB|      true  |  70% |      |        | 3 clients|     |
|10000000*3|  543435   |   18k/s*3  |   63.467       |            3|            10g|        512MB|      true  |  75% |      |        | 3 clients|   |
|10000000*3|  453985   |   22k/s*3  |   51.078       |            3|            10g|        512MB|      true  |  75% |      |        | 3 clients|hbase.hstore.blockingStoreFiles改为100|

#### 单个data-node
<pre>
./hbase org.apache.hadoop.hbase.PerformanceEvaluation --nomapred --rows=10000000 --table=test1 --valueSize=20 --compress=LZO  --flushCommits=false --columns=8 sequentialWrite 1
</pre>
|保存条数  |花费时间ms|平均写入速度|99.99%写入耗时ms|DataNode个数|Region划分策略|memstore配置|writeToWAL|CPU |内存|磁盘I/O|其他配置 |备注|
| :----: | :----: | :----: |:----: | :----: |:----: |:----: |:----: |:----: |:----: |:----: |:----: |:----: |
|10000000  |   1024198 |  9k/s     |  149.936        |            1|            10g|        512MB|      true  |  40% |      |        | 1 clients|     |
|10000000*2|   1382774 |  7.2k/s*2 |  196.610        |            1|            10g|        512MB|      true  |  45% |      |        | 2 clients|     |
|10000000*3|   1727288 |  5.7k/s*3 |  254.034        |            1|            10g|        512MB|      true  |  75% |      |        | 3 clients|     |
|10000000*3|  1609463  |  6.2k/s*3 |   238.156       |            1|           100g|        512MB|      true  |  75% |      |        | 3 clients|hbase.hstore.blockingStoreFiles改为100|

* 注意测试region爆发split时的性能表现---测试结果很平稳，性能未出现明显变化
