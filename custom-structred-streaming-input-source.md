### spark structred streaming

https://spark.apache.org/docs/latest/structured-streaming-programming-guide.html

### 输入源介绍

目前官方支持的输入源还很少，[参考](https://spark.apache.org/docs/latest/structured-streaming-programming-guide.html#input-sources)，不支持的地方需要自己实现。目前官方已经升级到了 [V2](https://issues.apache.org/jira/browse/SPARK-15689) 版本，并且很快不再向后兼容。所以这里只记录 V2 版的相关实现

### 需要实现的接口
- [DataSourceV2](https://people.apache.org/~pwendell/spark-nightly/spark-master-docs/spark-2.3.0-SNAPSHOT-2017_12_08_20_01-acf7ef3-docs/api/java/org/apache/spark/sql/sources/v2/DataSourceV2.html)
- [MicroBatchReadSupport](https://spark.apache.org/docs/2.3.0/api/java/org/apache/spark/sql/sources/v2/MicroBatchReadSupport.html)
- [DataSourceRegister](https://spark.apache.org/docs/2.3.0/api/java/org/apache/spark/sql/sources/DataSourceRegister.html)
- [MicroBatchReader](https://spark.apache.org/docs/2.3.0/api/java/org/apache/spark/sql/sources/v2/reader/streaming/MicroBatchReader.html)
- [DataReaderFactory](https://spark.apache.org/docs/2.3.0/api/java/org/apache/spark/sql/sources/v2/reader/DataReaderFactory.html)
- [DataReader](https://spark.apache.org/docs/2.3.0/api/java/org/apache/spark/sql/sources/v2/reader/DataReader.html)

#### DataSourceV2
是个空接口， spark 用于标记数据源，直接实现它的子接口即可

#### MicroBatchReadSupport
继承 DataSourceV2 ，是一个支持类的借口， 作为自定义输入源的入口，用于创建一个 MicroBatchReader。

- createMicroBatchReader(java.util.Optional<StructType> schema, String checkpointLocation, DataSourceOptions options)，会被调用两次，一次为 readStream() 时，此时会传入自定义的 schema ，并且 readStream() 得到的 DataFrame 会读取这里的 schema 作为 Row 。一次为 writeStream() 时，真正开始按批处理，此时因为 DataFrame schema 已经固定，所以不会传递 schema。checkpointLocation 为检查点位置，用于输入源中断再开启时恢复 offset 使用。 options 为自定义选项，可以将一些参数传入源中使用，主要类型为 Map<Stirng, String> 且 key 不区分大小写(会直接把 key 都转成小写)。

#### DataSourceRegister
注册自定义输入源，需要在 META-INF/services/org.apache.spark.sql.sources.DataSourceRegister 文件中添加实现 DataSourceRegister 类的详细路径，[参考官方源](https://github.com/apache/spark/blob/master/sql/core/src/main/resources/META-INF/services/org.apache.spark.sql.sources.DataSourceRegister)。这个接口较为简单通常与　MicroBatchReadSupport　一起实现

- String shortName()，直接返回自定义输入源的简单名字，这样就可在 readStream()　之后调用　format()　传入这个简单名字，来让 spark　找到自定义输入源

#### MicroBatchReader
主要用于分批，每次 trigger 都会调用其中的方法，通过实现一些 offset 规则来将数据分批。

- StructType readSchema() ，调用的结果会直接到 readStream() 的 DataFrame　里，　用于数据源可以对自定义的　schema　进行修改或者重定义。

- List<DataReaderFactory<Row>> createDataReaderFactories() 创建一个　DataReaderFactory　的列表，之所以是列表是因为每个 Factory 都会被送到一个　RDD　分区中，可以在这个方法中实现自定义的分区规则

- void setOffsetRange(Optional<Offset> start, Optional<Offset> end)

- Offset getStartOffset()

- Offset getEndOffset()

- Offset deserializeOffset(String json)　用于反序列化 offset，offset 是需要实现序列化的，这里需要实现对应的反序列化

- void commit(Offset end)

- void stop()， structred streaming 中断或者抛出异常时，输入源退出调用，可以关闭一些资源

分批的规则是，spark 会自己维护一个　availableOffset，　每次　trigger 会固定调　setOffsetRange　一次，把ａvailableOffset 作为　start　参数， 它为上次结束的值(如果是第一次，没有　checkpointLocation　做恢复会是０，如果有，会是上次　checkpointLocation　的位置)，然后调用　getEndOffset()　得到一个 offset 看是否与　availableOffset　是否相同，如果相同，认为这次 trigger 没有数据不能形成一批，等待下次 trigger。如果不同，认为有数据，然后调用　commit() 提交上一批的数据(注意这里的机制导致新的数据没有到来形成一批，会导致上一批数据一直无法提交)，提交之后，再调用一次　setOffsetRange　， start 还是　ａvailableOffset　， end 为刚刚掉　getEndOffset() 得到的　offset　，然后调用　createDataReaderFactories　读取源数据，然后将　ａvailableOffset　设置为　getEndOffset() 得到的　offset。等待下一次 triiger　循环。

#### DataReaderFactory

- DataReader<T> createDataReader()

#### DataReader 

- boolean next()
- Row get()
- void close()

读取的规则是，调用 next() 得到是 true 就调用　get 获取一行数据，得到　false 就调用 close()　完成读取。注意这里是运行在 spark executor 上的，通常会把数据传递到这里，进行读取。
