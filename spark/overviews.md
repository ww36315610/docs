
作者：用心阁
链接：https://www.zhihu.com/question/26568496/answer/41608400
来源：知乎
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。

那么Spark解决了Hadoop的哪些问题呢？抽象层次低，需要手工编写代码来完成，使用上难以上手。=>基于RDD的抽象，实数据处理逻辑的代码非常简短。。只提供两个操作，Map和Reduce，表达力欠缺。=>提供很多转换和动作，很多基本操作如Join，GroupBy已经在RDD转换和动作中实现。一个Job只有Map和Reduce两个阶段（Phase），复杂的计算需要大量的Job完成，Job之间的依赖关系是由开发者自己管理的。=>一个Job可以包含RDD的多个转换操作，在调度时可以生成多个阶段（Stage），而且如果多个map操作的RDD的分区不变，是可以放在同一个Task中进行。处理逻辑隐藏在代码细节中，没有整体逻辑=>在Scala中，通过匿名函数和高阶函数，RDD的转换支持流式API，可以提供处理逻辑的整体视图。代码不包含具体操作的实现细节，逻辑更清晰。中间结果也放在HDFS文件系统中=>中间结果放在内存中，内存放不下了会写入本地磁盘，而不是HDFS。ReduceTask需要等待所有MapTask都完成后才可以开始=> 分区相同的转换构成流水线放在一个Task中运行，分区不同的转换需要Shuffle，被划分到不同的Stage中，需要等待前面的Stage完成后才可以开始。时延高，只适用Batch数据处理，对于交互式数据处理，实时数据处理的支持不够=>通过将流拆成小的batch提供Discretized Stream处理流数据。对于迭代式数据处理性能比较差=>通过在内存中缓存数据，提高迭代式计算的性能。
