# Hbase基准测试

## Hbase测试工具

## hbase自带测试工具类说明

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

## 该工具类存在的问题
   经查源码，该工具中randomWrite（随机写入）方式实际为rowkey的写入条数随机取，实际上会比期望的插入条数少很多，且生成的rowkey为顺序递增，不能均衡写入各个region，存在热点问题，不能满足我们的业务模拟；
   sequentialWrite（顺序写入）方式rowkey的生成方式为顺序递增，不能均衡写入各个region，同样存在热点问题，不能满足我们的业务模拟；
## 工具类定制化修改
   对该工具类中sequentialWrite方式的rowkey生成方式做了修改，修改rowkey的生成规则为vin码+纳秒时间戳，为了消除热点问题，只选择1000个vin码，循环生成rowkey（模拟1000个车辆上报），为了不影响存储时间的统计，将整个rowkey预先放入内存中，在for循环内部直接读取rowkey，提高效率。
   该工具基于hbase 1.1.2版本工具类修改，不同的版本之间可能存在差异，需要针对不同的版本做定制化修改；
## 工具类的部署：
   将该工具包hbase-server-1.1.2-tests.jar,放到hbase的lib目录中，替换原有的hbase-server-1.1.2-tests.jar包，建议在单独的一个服务器上部署该hbase包；
## 工具类的使用：
使用方式与原命令保持一致，注意修改的是sequentialWrite方式，命令如下：
<pre>
./hbase org.apache.hadoop.hbase.PerformanceEvaluation --nomapred --rows=10000000 --table=test1 --valueSize=20 --compress=LZO  --flushCommits=false --columns=8 sequentialWrite 1
</pre>
## 命令解读：
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
