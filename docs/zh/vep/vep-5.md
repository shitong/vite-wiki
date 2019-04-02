# VEP 5: 移除作为时间戳的快照块哈希显式引用

## 背景

在Vite白皮书1.0版中，章节6.2.2（资源量化）提到：
由于快照链相当于一个全局时钟，我们可以利用它准确的量化一个账户的资源使用情况。在每个交易中，必须引用一个快照块的哈希，将快照块的高度作为该交易的时间戳。因此可以根据两个交易时间戳的差，来判断这两个交易之间是否间隔足够长的时间。

![figure](../../../assets/images/explicit_timestamp.png)<div align="center">图 1</div>

如上图所示，每个交易都包含一个快照块哈希的显式引用，作为交易创建时间的证明。这个时间戳的主要用途是计算一个账户的资源消耗是否超出配额。

除了这个时间戳之外，系统中还存在另一个"隐式"的时间戳，也就是一个快照块中所记录的每个账户的最新交易哈希，代表的是一个交易的"确认时间"。

由于这两种时间戳的存在，DAG账本中的交易和快照链之间存在双向依赖关系，如下图所示：

![figure](../../../assets/images/timestamps.png)<div align="center">图 2</div>

红色的箭头表示交易对快照链的引用，也就是显式时间戳（交易的创建时间）；蓝色的箭头表示快照块对账本中交易的引用，也就是隐式时间戳（交易的确认时间）。

## 快照链分叉对时间戳的影响

由于快照链存在分叉的可能，实际上是一个树状结构，快照块的高度越大，被回滚的概率越高。当快照块被回滚时，这两种时间戳都会受到影响。
其中，显式时间戳受到的影响更大，如下图所示：

![figure](../../../assets/images/explicit_forked.png)<div align="center">图 3</div>

快照链末尾右侧的快照块被回滚，为另一个分叉的快照块所取代。所有引用该快照块的交易都将作废，包括图中A和B账户的最后一个交易。
这是由于这些交易在签名时使用了被回滚的快照块哈希，在更新这个字段后，必须由持有私钥的用户重新签名并发布新的交易。
重新签名交易是一个非常昂贵的操作，在实际应用中应该尽量避免。
一个比较保守的策略是尽量引用比较旧的快照块，也就是快照链的“树干”的部分。这种做法又失去了显式时间戳作为交易创建时间的现实意义。
因此，显式时间戳的存在非常鸡肋。

与此相对，隐式时间戳在快照块回滚时就没有同样的问题，如下图所示：

![figure](../../../assets/images/implicit_forked.png)<div align="center">图 4</div>

一个快照块被回滚，其中的隐式时间戳与这个快照块一起被废弃，A账户的第二个交易变成未确认状态。另一个分叉中新的快照块仍然会包含一组隐式时间戳，只是与原分叉中的记录未必相同。
由于隐式时间戳是反向引用，DAG账本中的交易没有发生改变，因此，时间戳的更新只与SBP有关，用户无需重新签名交易，分叉处理逻辑相对简单。

## 采用隐式时间戳计算资源消耗

那么，使用隐式时间戳，是否仍然可以准确量化每个账户的资源使用并计算配额消耗呢？答案是肯定的。

如图2所示，如果按照原来的方式，使用显式时间戳计算，在快照N-2到快照N的2个时间单位内，账户A创建了4个交易，账户B创建了2个交易；
而采用隐式时间戳计算，在快照N-1到快照N的1个时间单位内，账户A被确认了2个交易，账户B被确认了1个交易。
这样，按照每个账户在固定时间间隔内被确认的交易所消耗的资源总量，仍然可以准确量化该账户近期的平均资源消耗，从而判断出该账户新增的交易是否超出其配额。

当然，由于交易的创建和确认是异步的，交易在网络上传输，以及账户的状态计算都会消耗一定时间，用交易确认时间来判断配额可能会存在一定误差。
我们可以通过提高统计的时间间隔数，也就是统计更长时间的平均资源消耗，来减小这种误差的影响。
在Vite系统中，只要在同一时间段内，针对不同账户的资源消耗统计是公平的，就足以保证配额模型的正常工作。

## 结论

基于以上讨论，本提案建议将Vite账本中所有显式时间戳移除，只保留隐式时间戳。

这样，虽然链上数据中不再保存交易的创建时间，会损失一些依赖该时间戳的语义，但这种影响非常有限。尤其是，资源消耗计算和配额模型仍然可以使用"确认时间"这个隐式时间戳来实现。

而这个优化却可以带来如下收益：

- 去除DAG账本和快照链结构之间的双向依赖，使DAG账本和快照链可以异步构建；
- 简化了快照块的回滚处理逻辑，有助于提高系统的吞吐量；
- 时间戳失效后，无需重新签名交易，简化了客户端处理逻辑；
- 减少了DAG账本的存储空间。