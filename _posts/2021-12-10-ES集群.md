## 1.相关概念解释

（1）Near Realtime（NRT）：近实时，两个意思，从写入数据到数据可以被搜索到有一个小延迟（大概 1 秒）；基于 es 执行搜索和分析可以达到秒级

（2）Cluster：集群，包含多个节点，每个节点属于哪个集群是通过一个配置（集群名称，默认是 elasticsearch）来决定的，对于中小型应用来说，刚开始一个集群就一个节点很正常

（3）Node：节点 (简单理解为集群中的一个服务器)，集群中的一个节点，节点也有一个名称（默认是随机分配的），节点名称很重要（在执行运维管理操作的时候），默认节点会去加入一个名称为“elasticsearch” 的集群，如果直接启动一堆节点，那么它们会自动组成一个 elasticsearch 集群，当然一个节点也可以组成一个 elasticsearch 集群

（4）Index：索引 (简单理解就是一个数据库)，包含一堆有相似结构的文档数据，比如可以有一个客户索引，商品分类索引，订单索引，索引有一个名称。一个 index 包含很多 document，一个 index 就代表了一类类似的或者相同的 document。比如说建立一个 product index，商品索引，里面可能就存放了所有的商品数据，所有的商品 document。

（5）Type：类型 (简单理解就是一张表)，每个索引里都可以有一个或多个 type，type 是 index 中的一个逻辑数据分类，一个 type 下的 document，都有相同的 field，比如博客系统，有一个索引，可以定义用户数据 type，博客数据 type，评论数据 type。

（6）Document&field：文档 (就是一行数据)，es 中的最小数据单元，一个 document 可以是一条客户数据，一条商品分类数据，一条订单数据，通常用 JSON 数据结构表示，每个 index 下的 type 中，都可以去存储多个 document。一个 document 里面有多个 field，每个 field 就是一个数据字段。

（7）shard：单台机器无法存储大量数据，es 可以将一个索引中的数据切分为多个 shard，分布在多台服务器上存储。有了 shard 就可以横向扩展，存储更多数据，让搜索和分析等操作分布到多台服务器上去执行，提升吞吐量和性能。每个 shard 都是一个 lucene index。

（8）replica：任何一个服务器随时可能故障或宕机，此时 shard 可能就会丢失，因此可以为每个 shard 创建多个 replica 副本。replica 可以在 shard 故障时提供备用服务，保证数据不丢失，多个 replica 还可以提升搜索操作的吞吐量和性能。primary shard（建立索引时一次设置，不能修改，默认 5 个），replica shard（随时修改数量，默认 1 个），默认每个索引 10 个 shard，5 个 primary shard，5 个 replica shard，最小的高可用配置，是 2 台服务器。

## 2 ElasticSearch 分布式架构原理

## 2.1shad 与 replica 机制

（1）一个 index 包含多个 shard, 也就是一个 index 存在多个服务器上

（2）每个 shard 都是一个最小工作单元，承载部分数据，比如有三台服务器, 现在有三条数据, 这三条数据在三台服务器上各方一条.

（3）增减节点时，shard 会自动在 nodes 中负载均衡

（4）primary shard 和 replica shard，每个 document 肯定只存在于某一个 primary shard 以及其对应的 replica shard 中，不可能存在于多个 primary shard

（5）replica shard 是 primary shard 的副本，负责容错，以及承担读请求负载

（6）primary shard 的数量在创建索引的时候就固定了，replica shard 的数量可以随时修改

（7）primary shard 的默认数量是 5，replica 默认是 1，默认有 10 个 shard，5 个 primary shard，5 个 replica shard

（8）primary shard 不能和自己的 replica shard 放在同一个节点上（否则节点宕机，primary shard 和副本都丢失，起不到容错的作用），但是可以和其他 primary shard 的 replica shard 放在同一个节点上


2.2 分布式架构图
----------

![](https://img-blog.csdnimg.cn/20190522205818635.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3NpbmF0XzE2NjU4MjYz,size_16,color_FFFFFF,t_70)


2.3 容错机制
--------

在集群中会有一个 master 负责当 leader 进行协调, 比如上图的 Node2 为 master, 那么当它挂了的时候会重现选举一个新的 master, 比如新选举的是 Node3, 这个时候 replica 2 这时候会变成 primary.

当 Node2 恢复了的时候, 这个时候 node2 的 prinary 会变成 replica


2.4ES 写入数据的过程
-------------

### 2.4.1 简单流程:

1.  客户端选择一个 node 发送请求过去，这个 node 就是 coordinating node (协调节点)
2.  coordinating node，对 document 进行路由，将请求转发给对应的 node
3.  实际上的 node 上的 primary shard 处理请求，然后将数据同步到 replica node
4.  coordinating node，如果发现 primary node 和所有的 replica node 都搞定之后，就会返回请求到客户端

      这个路由简单的说就是取模算法, 比如说现在有 3 太服务器, 这个时候传过来的 id 是 5, 那么 5%3=2, 就放在第 2 太服务器

### 2.4.2 写入数据底层原理:

  ![](https://img-blog.csdnimg.cn/20190522155754105.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3NpbmF0XzE2NjU4MjYz,size_16,color_FFFFFF,t_70)

1.  数据先写入到 buffer 里面，在 buffer 里面的数据时搜索不到的，同时将数据写入到 translog 日志文件之中
2.  如果 buffer 快满了，或是一段时间之后 (定时)，就会将 buffer 数据 refresh 到一个新的 OS cache 之中，然后每隔 1 秒，就会将 OS cache 的数据写入到 segment file 之中，但是如果每一秒钟没有新的数据到 buffer 之中，就会创建一个新的空的 segment file，只要 buffer 中的数据被 refresh 到 OS cache 之中，就代表这个数据可以被搜索到了。当然可以通过 restful api 和 Java api，手动的执行一次 refresh 操作，就是手动的将 buffer 中的数据刷入到 OS cache 之中，让数据立马搜索到，只要数据被输入到 OS cache 之中，buffer 的内容就会被清空了。同时进行的是，数据到 shard 之后，就会将数据写入到 translog 之中，每隔 5 秒将 translog 之中的数据持久化到磁盘之中
3.  重复以上的操作，每次一条数据写入 buffer，同时会写入一条日志到 translog 日志文件之中去，这个 translog 文件会不断的变大，当达到一定的程度之后，就会触发 commit 操作。
4.  将一个 commit point 写入到磁盘文件，里面标识着这个 commit point 对应的所有 segment file
5.  强行将 OS cache 之中的数据都 fsync 到磁盘文件中去。  
     解释：translog 的作用：在执行 commit 之前，所有的而数据都是停留在 buffer 或 OS cache 之中，无论 buffer 或 OS cache 都是内存，一旦这台机器死了，内存的数据就会丢失，所以需要将数据对应的操作写入一个专门的日志问价之中，一旦机器出现宕机，再次重启的时候，es 会主动的读取 translog 之中的日志文件的数据，恢复到内存 buffer 和 OS cache 之中。
6.  将现有的 translog 文件进行清空，然后在重新启动一个 translog，此时 commit 就算是成功了，默认的是每隔 30 分钟进行一次 commit，但是如果 translog 的文件过大，也会触发 commit，整个 commit 过程就叫做一个 flush 操作，我们也可以通过 ES API, 手动执行 flush 操作，手动将 OS cache 的数据 fsync 到磁盘上面去，记录一个 commit point，清空 translog 文件  
     补充：其实 translog 的数据也是先写入到 OS cache 之中的，默认每隔 5 秒之中将数据刷新到硬盘中去，也就是说，可能有 5 秒的数据仅仅停留在 buffer 或者 translog 文件的 OS cache 中，如果此时机器挂了，会丢失 5 秒的数据，但是这样的性能比较好，我们也可以将每次的操作都必须是直接 fsync 到磁盘，但是性能会比较差。
7.  如果时删除操作，commit 的时候会产生一个. del 文件，里面讲某个 doc 标记为 delete 状态，那么搜索的时候，会根据. del 文件的状态，就知道那个文件被删除了。
8.  如果时更新操作，就是讲原来的 doc 标识为 delete 状态，然后重新写入一条数据即可。
9.  buffer 每次更新一次，就会产生一个 segment file 文件，所以在默认情况之下，就会产生很多的 segment file 文件，将会定期执行 merge 操作
10.  每次 merge 的时候，就会将多个 segment file 文件进行合并为一个，同时将标记为 delete 的文件进行删除，然后将新的 segment file 文件写入到磁盘，这里会写一个 commit point，标识所有的新的 segment file，然后打开新的 segment file 供搜索使用。


2.5ES 查询过程
----------

### 2.5.1 倒排序算法

查询有个算法叫倒排序: 简单的说就是: 通过分词把词语出现的 id 进行记录下来, 再查询的时候先去查到哪些 id 包含这个数据, 然后再根据 id 把数据查出来. 要是不理解的可以先去查下什么是倒排序

### 2.5.2 查询过程

1.  客户端发送一个请求给 coordinate node
2.  协调节点将搜索的请求转发给所有的 shard 对应的 primary shard 或 replica shard
3.  query phase：每一个 shard 将自己搜索的结果（其实也就是一些唯一标识），返回给协调节点，有协调节点进行数据的合并，排序，分页等操作，产出最后的结果
4.  fetch phase ，接着由协调节点，根据唯一标识去各个节点进行拉去数据，最总返回给客户端

### 2.5.3 查询原理

查询过程大体上分为查询和取回这两个阶段，广播查询请求到所有相关分片，并将它们的响应整合成全局排序后的结果集合，这个结果集合会返回给客户端。

1.  查询阶段

    1.  当一个节点接收到一个搜索请求，这这个节点就会变成协调节点，第一步就是将广播请求到搜索的每一个节点的分片拷贝，查询请求可以被某一个主分片或某一个副分片处理，协调节点将在之后的请求中轮训所有的分片拷贝来分摊负载。
    2.  每一个分片将会在本地构建一个优先级队列，如果客户端要求返回结果排序中从 from 名开始的数量为 size 的结果集，每一个节点都会产生一个 from+size 大小的结果集，因此优先级队列的大小也就是 from+size，分片仅仅是返回一个轻量级的结果给协调节点，包括结果级中的每一个文档的 ID 和进行排序所需要的信息。
    3.  协调节点将会将所有的结果进行汇总，并进行全局排序，最总得到排序结果。
2.  取值阶段

    1.  查询过程得到的排序结果，标记处哪些文档是符合要求的，此时仍然需要获取这些文档返回给客户端
    2.  协调节点会确定实际需要的返回的文档，并向含有该文档的分片发送 get 请求，分片获取的文档返回给协调节点，协调节点将结果返回给客户端


2.6 更新过程
--------

### 2.6.1document 的全量替换

1.  这个就是用新的数据全部覆盖以前的数据
2.  重新创建一个 document 并把原来的标记为 delete
3.  partial update, 就是制定需要更新的字段.


         全量是把数据找出来, 然后再 java 代码中进行修改, 再放回去.

          partial 是直接提交需要修改的字段然后直接修改, 在一个 shard 中进行, 内部也是全量替换.

### 2.6.2 强制创建

就是不管原来的数据, 直接强制创建一个新的

### 2.7 删除过程

当要进行删除 document 的时候, 只是把它标记为 delete, 当数据到达一定的时候再进行删除, 有点像 JVM 中标记清除法


3.Es 并发解决方案
===========

为什么会出现并发问题, 简单的说就是多个线程去操作同一个数据.

假如现在下单系统下 2 单, 都要去减库存 (100 件), 第一个线程进去减 1 件 100-1=99, 这时候还没更像到 ES 中, 第二线程进去了, 也要减一个库存 100-1=99. 现在系统卖出去 2 个, 可是库存却还有 99 个, 应该是 98 个


3.1 解决方案 - 悲观锁
--------------


3.2 解决方案 - 乐观锁
--------------

温馨提示, 乐观锁会出现 ABA 情况

下面是 2 种解决方案, 在网上找的图片:

![](https://img-blog.csdnimg.cn/20190522171506262.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3NpbmF0XzE2NjU4MjYz,size_16,color_FFFFFF,t_70)

