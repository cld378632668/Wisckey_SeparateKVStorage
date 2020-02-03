**WiscKey启发于FAST会议发表了论文 WiscKey: Separating Keys from Valuesin SSD-conscious Storage.是基于LSM（Log-Structured Merge-Tree）结构并且采用kv分离存储思路实现的存储引擎，WiscKey是基于leveddb 1.20版本进行修改的，WiscKey并不是按照论文里的设计来实现的，只是借鉴了kv分离的思想**

# 特点
 * 相比leveldb，WiscKey的写入速度更快，因为kv分离后减少了写放大，合并的效率大大提高，特别是对大kv场景这种优势更加明显.个人觉得WiscKey适合于对写入速度要求很高并且value较大的场景（leveldb虽然写入速度也很快，但一旦合并速度跟不上写入速度的时候就会自动降低写入速度）
  
 * 相比leveldb，WiscKey的读速度可能稍慢一些，因为kv分离带来的坏处就是需要多一次从磁盘读取value的操作，但因为kv分离使sst文件能容纳的kv记录更多了，大大减少了sst文件个数，便于tablecache以及blockcache容纳更多的kv记录，同时降低了lsm树的深度，因此读速度并没有想像中那么差.
  * 相比leveldb，WiscKey不支持快照机制，因为垃圾回收机制会重新将有效的数据put到db中，这就导致了原来在lsm树深层的记录重新回到了lsm浅层的位置，我们知道leveldb读的顺序就是由浅往深。
  * leveldb虽然官方文档上写Keys and values are arbitrary byte arrays.但其实细看代码发现Keys and values的长度均需小于4G（详见PutLengthPrefixedSlice函数）,WiscKey与leveldb一样，Keys and values的长度均需小于4G
  * 支持自动垃圾回收（在线清理）和手动垃圾回收（离线清理，api为DBImpl::CleanVlog()）

# 写流程
与leveldb不同的是，写入memtable以及sst文件的记录其实是<key,address>,address指向这条kv操作的日志记录在vlog文件中的偏移以及长度。
![image](https://github.com/joker-qi/WiscKey/raw/master/images/put.png)
插入或者删除：先append到vLog末尾，再将Key和日志记录的地址插入LSM

# 读流程
比leveldb多了一步，因为从memtable或者sst文件读到的v其实是指向vlog文件某个位置的address，需要从address所指的位置读出真正的value。

简而言之，就是LSM中获得地址，vLog中读取

# 如何保证一致性
leveldb的实现里，当immemtbale刷入到sst文件并且成功在manifest文件中记录下这次变化后（即VersionSet::LogAndApply成功），log文件就可以被删除掉了。否则，在恢复的时候，数据库会重新回放该log文件。

由前面可知WiscKey的vlog文件时不能删除的，因为各个value的内容还存放在这里，那么故障恢复时该怎么做，不可能从头到尾回放vlog，WiscKey的做法是在imm生成sst文件前，会在VersionEdit中加入headInfo：pos（headInfo信息是WiscKey增加的信息类型，leveldb的edit是没有的），pos的含义就是在vlog中的偏移，代表在pos以前的kv记录已经成功写入到了sst文件，恢复的时候VersionSet::Recover会回放edit记录，便可以得到headInfo信息，从而获得pos，然后从pos位置处回放vlog文件即可。（可能解释的有点不清，需要对LogAndApply以及Recover函数十分了解，并且明白version_edit和manifest文件的关系,详见(http://blog.csdn.net/weixin_36145588/article/details/78029415
 (http://blog.csdn.net/weixin_36145588/article/details/77978433)

# 被用户删除或者过期版本的Value的空间回收
Compaction过程需要被删除的数据由于只是删除了Key，Value还保留在分开的vlog文件中，这就需要异步的回收。LSM本身的Compaction其实也是垃圾回收。

先看一下WiscKey论文的回收思路（详见360康神的博客(http://catkang.github.io/2017/04/30/lsm-upon-ssd.html)
- 离线回收：扫描整个LSM对Value Log进行mark and sweep，但这相当于给系统带来了负载带来了陡峭的波峰
- 在线回收：其中head的位置是新的Block插入的位置，tail是Value回收操作的开始位置，垃圾回收过程被触发后，顺序从Tail开始读取Block，将有效的Block插入到Head。删除空间并后移Tail。
![image](https://github.com/joker-qi/WiscKey/raw/master/images/garbage.png)

这种在线回收的缺点：
- 什么时候触发？根据删除修改请求的多少？如何知道删除修改请求了多少次？（如果数据库关闭了重新打开呢，之前的删除修改次数怎么获取）
- 什么时候终止？WiscKey论文里的设计是只有一个vlog文件，利用fallocate系统调用实现tail之前磁盘空间的归还，但好像没有提什么时候终止？
- 不够灵活，如果垃圾（过期版本的kv日志记录）主要集中在vlog文件的某一块位置呢，这样从头开始对vlog文件进行垃圾回收显然效率太低
- head和tail需要经常写入lsm中，增加了lsm树的压力，论文中每回收一条vlog文件里的kv记录时，都需要将tail信息put到lsm树中，对效率影响很大，不仅增加了lsm树中kv记录条数，而且会导致不同sst文件范围冲突的概率加大（因为大家可能都含tail记录），同理head也是。


本项目WiscKey中设计的在线垃圾回收机制
- 什么时候触发？根据lsm Compaction（指的是major Compaction）的情况，因为每次Compaction后就可以知道lsm已经清除了多少条过期版本的kv记录，只不过这些过期版本的kv日志记录还在vlog文件中，当lsm已经清除的过期版本的kv记录数达到我们自己设定的某个阀值（Options::clean_threshold）时，便触发在线回收。为了防止崩溃或者db关闭，需要定期将lsm已经清除的过期版本的kv记录数（简称为vloginfo信息，即该vlog文件中已经失效的kv记录总数）持久化，这里采用和持久化headInfo一样的方法，将vloginfo信息定期(新增失效kv记录数超过Options::log_dropCount_threshold时)持久化到edit中，然后VersionSet::LogAndApply（vloginfo的持久化很方便的，因为leveldb本来的每次major Compaction完成后都会调用LogAndApply）。
- 依然没有解决不够灵活的问题，为了解决该问题采取多vlog文件，一个vlog文件到达一定大小后，生成下一个vlog文件。
- 因为采用了多vlog文件，这就自然而然解决了如何终止垃圾回收的问题，把整个vlog文件扫描完后便可终止。
- 什么样的vlog文件该进行垃圾回收呢？举个例子，假如一个vlog文件能够容纳1000条kv日志记录，其中500条kv日志记录已经在lsm中清除过了，意味着该vlog文件中至少有一半的日志记录已经过期失效。我们可以在option中（Options::clean_threshold）设置vlog文件可以容忍的过期失效记录总数的上限。超过该上限便进行该vlog文件的垃圾回收。

# vlog垃圾回收的过程

从头到尾扫描vlog文件的每一条kv日志记录（一次读很大一块vlog文件的内容）
- 如果是条put记录，则GetPtr(key)获取addr，如果addr所指的位置就是当前日志记录的位置（省去读具体value值，也就减少了一次read磁盘），说明该kv日志记录有效，重新将该kv记录put到db中，否则直接丢弃该记录（可以维护一个有效的writebatch,用来存放vlog文件回放出来的多条有效kv记录，当writebatch够大时（writebatch的size超过Options::clean_write_buffer_size）在一口气write到db中）。vlog文件扫描完后便可删除该vlog文件。
- 如果是条delete记录，直接丢弃。

# 引入多vlog文件后WiscKey的改变
![image](https://github.com/joker-qi/WiscKey/raw/master/images/put1.png)
- address里多出来了file_numb用来表示value所在的vlog文件的编号
- 在imm生成sst文件前，在VersionEdit中加入headInfo：<file_numb,pos>,pos的含义就是在vlog中的偏移，代表在pos以前的kv记录已经成功写入到了sst文件,file_numb代表的是从哪个vlog文件进行恢复
- 为了防止崩溃或者db关闭，需要定期将各个vlog文件中已经被lsm清除的过期版本的kv记录数持久化到edit中。
- 清理vlog文件进行到一半时数据库进行关闭操作怎么办，把tail：<file_numb,pos>加入到edit中，然后LogAndApply，下一次打开db时接着从file_numb对应的vlog文件的pos位置进行垃圾清理。

# 性能
 
 ## 测试环境1
 创建一个100万条kv记录的数据库，其中每条记录的key为16个字节，value为100个字节，我们没有开启snappy压缩，所以写入磁盘的文件大小就是数据的原始大小，其余参数都是option的默认值。
 
    LevelDB:    version 1.20
    Date:       Mon Feb  5 12:00:07 2018
    CPU:        24 * Intel(R) Xeon(R) CPU E5-2620 0 @ 2.00GHz
    CPUCache:   15360 KB
    Keys:       16 bytes each
    Values:     100 bytes each (50 bytes after compression)
    Entries:    1000000
    Raw Size:   110.6 MB (estimated)
    File Size:  62.9 MB (estimated)
    WARNING: Snappy compression is not enabled

 ### 写性能

 leveldb
 
    fillseq      :       5.549 micros/op;   19.9 MB/s     
    fillrandom   :      11.597 micros/op;    9.5 MB/s     
    overwrite    :      14.144 micros/op;    7.8 MB/s
 
 WiscKey
 
    fillseq      :       5.576 micros/op;   19.8 MB/s     
    fillrandom   :       6.123 micros/op;   18.1 MB/s     
    overwrite    :       6.525 micros/op;   17.0 MB/s

 分析：
- 可以看出fillseq（顺序写，即各sst文件之间没有范围冲突）二者差不多，因为是顺序写，所以对应leveldb，各个sst文件间没有范围冲突，因此可以快速合并。理论上WiscKey可能会稍微快一点点，因为WiscKey写入memtable中的value值很小，WiscKey生成的sst文件个数更少。这种差距应该会在put(k,v)中v比较大的时候会体现出来。 
- fillrandom（随机写，即各sst文件范围冲突概率很大）以及overwrite（覆盖写，即各sst文件范围冲突概率很大），会经常触发合并，因为WiscKey是kv分离的，所以合并速度快于leveldb，自然fillrandom以及overwrite更快。
 
 ### 读性能
  leveldb
 
    readrandom   :       6.219 micros/op; (1000000 of 1000000 found)   
    readrandom   :       5.026 micros/op; (1000000 of 1000000 found)     
    readseq      :       0.531 micros/op;  208.4 MB/s
    compact      : 2189861.000 micros/op;
    readrandom   :       3.718 micros/op; (1000000 of 1000000 found)
    readseq      :       0.477 micros/op;  231.7 MB/s
    fill100K     :    3313.683 micros/op;   28.8 MB/s (1000 ops)
 WiscKey
 
    readrandom   :       7.014 micros/op; (1000000 of 1000000 found)     
    readrandom   :       6.288 micros/op; (1000000 of 1000000 found)
    readseq      :       2.498 micros/op;   44.3 MB/s
    compact      :  489370.000 micros/op;
    readrandom   :       5.148 micros/op; (1000000 of 1000000 found)
    readseq      :       2.462 micros/op;   44.9 MB/s
    fill100K     :     225.675 micros/op;  422.7 MB/s (1000 ops)
 分析：
- 先解释一下第一个和第二个readrandom，就是随机选key进行Get操作，因为读的时候可能伴随这合并操作的进行，合并会减少垃圾kv数据，所以第二个readrandom的读速度更快，所以compact完成后的readrandom速度更快，因为此时垃圾kv记录都被合并完成了。
- 顺序读readseq会远远快于readrandom，这是因为leveldb设计的cache的作用。而且顺序读的kv记录很可能是在磁盘上连续的，操作系统的页缓存也会起一定的作用，因此leveldb的readseq极大的减少了读磁盘操作，所以很快。
- 可以看出，当db负载很重时，不停地在触发合并，WiscKey的随机读速度并不会比leveldb慢很多，顺序读会慢很多，因为WiscKey的读至少会需要读一次vlog文件，这就意味着昂贵的读磁盘。
- 另外，可以看出WiscKey的compact操作更快
- fill100k：每条写入的kv记录的v为100k大小的大v，测试db对写大kv记录的性能，这当然是WiscKey的强项，因为WiscKey是kv分离的，合并时不用将100k的大v重新读入到内存进行归并排序，因此合并速度快，自然写速度就上去了。
 
 ## 测试环境2
 创建一个1000万条kv记录的数据库，其余环境都是跟测试环境1一样，只不过这次是千万级规模了。
 
  ### 写性能

 leveldb
 
    fillseq      :       5.943 micros/op;   18.6 MB/s     
    fillrandom   :      26.803 micros/op;    4.1 MB/s     
    overwrite    :      33.790 micros/op;    3.3 MB/s
 
 WiscKey
 
    fillseq      :       5.524 micros/op;   20.0 MB/s     
    fillrandom   :       8.564 micros/op;   12.9 MB/s     
    overwrite    :      10.969 micros/op;   10.1 MB/s 
 WiscKey的抗压能力能强，依旧能保证在大规模数据写的情况下保持高速。之所以随机写覆盖写速度差这么多，就是因为WiscKey的合并速度更快，不会造成level0的文件堆积情况出现。

 ### 读性能
  leveldb
 
    readrandom   :       7.739 micros/op; (10000000 of 10000000 found)     
    readrandom   :       4.947 micros/op; (10000000 of 10000000 found)     
    readseq      :       0.311 micros/op;  355.8 MB/s
    compact      :20013959.000 micros/op;
    readrandom   :       4.112 micros/op; (10000000 of 10000000 found)
    readseq      :       0.296 micros/op;  374.4 MB/s
    fill100K     :   11263.494 micros/op;    8.5 MB/s (10000 ops)
    
 WiscKey
 
    readrandom   :       7.507 micros/op; (10000000 of 10000000 found)     
    readrandom   :       7.003 micros/op; (10000000 of 10000000 found)
    readseq      :       2.954 micros/op;   41.7 MB/s
    compact      :13784032.000 micros/op;
    readrandom   :       5.363 micros/op; (10000000 of 10000000 found)
    readseq      :       2.800 micros/op;   42.4 MB/s
    fill100K     :     167.766 micros/op;  568.5 MB/s (10000 ops)
    
可以看出二者的读速度差距越来越小，原因有：
1. leveldb的读慢一方面是因为sst文件个数太多，导致lsm树层数很深，不利于读。相比 WiscKey 的lsm树就会浅很多。
2. leveldb的合并速度更慢，因此level0层会堆积很多文件，不仅影响写速度，还影响读速度，因为level0文件间所管的kv范围很可能互相冲突。 WiscKey的合并速度就快的多，因此level0层文件数可能会少点。
3. levledb的合并是在后台线程进行的，合并会涉及读写磁盘，因此当读操作时很可能后台也在合并，跟读抢磁盘。当然 WiscKey的合并也是在后台线程进行，但因为效率很高，需要读写磁盘的数据相比leveldb少很多。
4.  WiscKey的一个sst文件能容纳的kv记录条数更多，因为 WiscKey这里的v就是一个address，同样大小的sst情况下，kv大小越小，当然能容纳的kv记录数更多了，因此 WiscKey比levledb的sst文件更少，这除了1中提到的lsm树的深度更浅的优势外，还更有利于tablecache，因为tablecache需要缓存的sst文件数变少了，减少了tablecache的淘汰触发。
5.  上面的测试全是用的默认option选项，默认是不使用blockcache的，如果开启了blockcache，WiscKey的性能上升的会更明显，因为相同的block大小WiscKey能容纳更多的kv记录数。

## 测试环境3
 与测试环境2一样，创建一个1000万条kv记录的数据库，k为16字节，v为100字节，只是修改option的值，因为想测试在线垃圾回收时的性能，默认option选项会把clean_threshold设置的很大，这就导致不会触发在线垃圾回收。1000万条kv记录大约1106.3 MB，将option的max_vlog_size设置为300M，意味着当vlog文件大小超过300M时便会生成新的vlog文件，一个300M的vlog文件至少能容纳2\*1024\*1024条kv记录条数，因此将clean_threshold设为1\*1024\*1024，即当vlog文件的垃圾kv记录条数达到1*1024 * 1024便会触发垃圾回收。
 
  ### 写性能

 WiscKey
 
    fillseq      :       5.630 micros/op;   19.6 MB/s     
    fillrandom   :       9.233 micros/op;   12.0 MB/s     
    overwrite    :      10.717 micros/op;   10.3 MB/s
 
 ### 读性能

    
 WiscKey
 
    readrandom   :       7.845 micros/op; (10000000 of 10000000 found)     
    readrandom   :       7.190 micros/op; (10000000 of 10000000 found)
    readseq      :       2.741 micros/op;   40.4 MB/s
    compact      :  12571723.000 micros/op;
    readrandom   :       5.427 micros/op; (10000000 of 10000000 found)
    readseq      :       2.797 micros/op;   39.5 MB/s
    fill100K     :     208.036 micros/op;  458.5 MB/s (10000 ops)
    

让我很吃惊的是在线回收后性能并没有啥变化，虽然垃圾回收是大块的读跟大块的写，但感觉多少会有点影响把，可能是哪里有bug把，后续继续检查。
