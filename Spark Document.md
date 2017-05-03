## Initializing Spark

```scala
// master is a Spark, Mesos or YARN cluster URL
val conf = new SparkConf().setAppName(appName).setMaster(master)
new SparkContext(conf)
```

## Using the shell

* In spark shell, a special interpreter-aware **SparkContext** is created for u, in the variable called **sc**.
* —master  —>   to set which master the context connects to
* —jars   —>   add JARS to the classpath
* —package  —>  add dependencies
* —repositories   —> any additional repositories where dependencies might exist.

## Resilient Distributed Datasets

* A fault-tolerant collection of elements that can be operated on in parallel.

  ### Create: 

  * *parallelizing* an existing collection in ur driver program.

    ```scala
    val data = Array(1, 2, 3, 4, 5)
    val distData = sc.parallelize(data)
    ```

    And one important parameter for parallel collections is **the number of partitions** to cut the dataset into.     —> Spark tries to set this based on ur cluster, and also u can set it manually by ```sc.parallelize(data, 10)```

  * Or referencing a dataset in an external storage system, such as HDFS, HBASE, or any data source offering a Hadoop InputFormat.

    ```
    val distFile = sc.textFile("data.txt")
    ```

  ### RDD Operations

  * Two Operations: 
    1. **transformation**: creates a new dataset from an exsiting one.

       ​				lazy —>  just remember the transformations

    2. **actions**: return a value to the driver program after running a computation on the dataset.

       ​		trigger the computation. 

  * Attention:

    * **By default, each transformed RDD may be recomputed each time u run an action on it**. However, **u may also persist an RDD in memory using the persist(or cache) method**. Also there is support for persisting RDDs on disk, or replicated across multiple nodes.

  ### Basics

  ```scala
  val lines = sc.textFile("data.txt")   // base RDD from an external file. Not loaded in memory, merely a pointer to the file.
  val lineLengths = lines.map(s => s.length) // Not immediately computed due to laziness
  // lineLength.persist() if we want to use it again later
  val totalLength = lineLengths.reduce((a, b) => a + b) // Action, Spark breaks the computation into tasks to run on separte machines, and each machine runs both its part of the map and a local reduction.
  ```

  ### Passing Functions to Spark

  * Anonymous function syntax: used for short pieces of code

  * Static methods in a global singleton object

    ```scala
    // object是包含static方法的singleton
    object MyFunctions{
      def func1(s: String): String = {...}
    }
    myRdd.map(MyFunctions.func1)
    ```

    也可以传一个reference给一个class instance中的方法(与单例object不同)，这需要sending包含...

  ### Understanding closures

  * **Understanding the scope and life cycle of variables and methods** when executing code across a cluster.  —> RDD operations that modify variables outside of their scope can be a frequent source of confusion.

  * Example:

    ```scala
    var counter = 0
    var rdd = sc.parallelize(data)
    // Wrong: Don't do this!!!
    rdd.foreach(x => counter += x)
    // this may behave differently depending on whether execution is happening within the same JVM.
    ```

  * Local vs. cluster modes

    The behavior of the above code is undefined, and may not work as intended.

    To execute jobs, Spark **breaks up the processing of RDD operations into tasks, each of which is executed by an executor**.

    **Prior to execution, Spark computes the task's closure**. The closure is those variables and methods which must be **visible** for the executor to perform its computations on the RDD.

    **Accumulators** in Spark are used to specifically to provide a mechanism for safely updating a variable when execution is split up across worker nodes in a cluster.[More details following]

    In general, closures - constructs like loops or locally defined methods, should not be used to mutate some global state. Use an Accumulator instead if some global aggregation is needed.

  * Printing elements of an RDD: ```rdd.collect().foreach(println)```, this can cause the driver to run out of memory, because collect() fetches the entire RDD to a single machine; use ```rdd.take(100).foreach(println)```to print a few elements of the RDD.

  ### Working with key-value Pairs

  * A few special operations are only available on RDDs of key-value pairs.

  * The most common ones are distributed "shuffle" operations, such as grouping or aggregating the elements by a key.

  * Tuple2** objects: the built-in tuples in the language, created by simply ```writing(a, b)```

  * The k-v pair operations are available in the ```PairRDDFunctions``` class.

    ```scala
    val lines = sc.textFile("data.txt")
    val pairs = lines.map(s => (s, 1))
    val counts = pairs.reduceByKey((a, b) => a + b)
    ```

  ### Transformations

  | Transformation                           | Meaning                                  |
  | ---------------------------------------- | ---------------------------------------- |
  | **map**(*func*)                          | Return a new distributed dataset formed by passing each element of the source through a function *func*. |
  | **filter**(*func*)                       | Return a new dataset formed by selecting those elements of the source on which *func* returns true. |
  | **flatMap**(*func*)                      | Similar to map, but each input item can be mapped to 0 or more output items(so *func* should return a Seq rather than a single item). |
  | **mapPartitons**(*func*)                 | Similar to map, but runs separately on each partition(block) of the RDD, so *func* must be of type Iterator\<T> => Iterator\<U> when running on an RDD of type T. |
  | **mapPartitonsWithIndex**(*func*)        | Similar to mapPartitions, but also probides *func* with an integer value representing the index of the partitions, so *func* must be of type(Int, Iterator\<T>) => Iterator\<U> when running on an RDD. |
  | **sample**(*withReplacement, fraction, seed*) | Sample a fraction *fraction* of the data, with or without replacement, using a given random number generator seed. |
  | **union**(*otherDataset*)                | Return a new dataset that contains the union. |
  | **intersection**(*otherDataset*)         |                                          |
  | **distinct**([*numTasks*])               | Return a new dataset that contains the distinct elements of the source dataset. |

  ...

  ### Actions

  | Action                                   | Meaning                                  |
  | ---------------------------------------- | ---------------------------------------- |
  | **reduce**(*func*)                       | Aggregate the elements of the dataset using a function *func* (which takes two arguments and returns one). The function should be commutative and associative so that it can be computed correctly in parallel. |
  | **collect**()                            | Return all the elements of the dataset as an array at the driver program. This is usually useful after a filter or other operation that returns a sufficiently small subset of the data. |
  | **count**()                              | Return the number of elements in the dataset. |
  | **first**()                              | Return the first element of the dataset (similar to take(1)). |
  | **take**(*n*)                            | Return an array with the first *n* elements of the dataset. |
  | **takeSample**(*withReplacement*, *num*, [*seed*]) | Return an array with a random sample of *num* elements of the dataset, with or without replacement, optionally pre-specifying a random number generator seed. |
  | **takeOrdered**(*n*, *[ordering]*)       | Return the first *n* elements of the RDD using either their natural order or a custom comparator. |
  | **saveAsTextFile**(*path*)               | Write the elements of the dataset as a text file (or set of text files) in a given directory in the local filesystem, HDFS or any other Hadoop-supported file system. Spark will call toString on each element to convert it to a line of text in the file. |
  | **saveAsSequenceFile**(*path*) (Java and Scala) | Write the elements of the dataset as a Hadoop SequenceFile in a given path in the local filesystem, HDFS or any other Hadoop-supported file system. This is available on RDDs of key-value pairs that implement Hadoop's Writable interface. In Scala, it is also available on types that are implicitly convertible to Writable (Spark includes conversions for basic types like Int, Double, String, etc). |
  | **saveAsObjectFile**(*path*) (Java and Scala) | Write the elements of the dataset in a simple format using Java serialization, which can then be loaded using`SparkContext.objectFile()`. |
  | **countByKey**()                         | Only available on RDDs of type (K, V). Returns a hashmap of (K, Int) pairs with the count of each key. |
  | **foreach**(*func*)                      | Run a function *func* on each element of the dataset. This is usually done for side effects such as updating an [Accumulator](http://spark.apache.org/docs/latest/#accumulators) or interacting with external storage systems. **Note**: modifying variables other than Accumulators outside of the `foreach()` may result in undefined behavior. See [Understanding closures ](http://spark.apache.org/docs/latest/#understanding-closures-a-nameclosureslinka)for more details. |

  ### Shuffle operations

  * Certain operations within Spark trigger an event known as the shuffle. 

  * The **shuffle is Spark's mechanism for re-distributing data** so that it's grouped differently across partitions.

  * This typically involves copying data across executors and machines, making the shuffle a complex and costly operation.

    #### Background

    Considering the example of the reduceByKey operation. This generates a new RDD where all values for a single key are combined into a tuple. To do this, Spark needs to perform an all-to-all operation. It must read from all partitions, and then bring together values across partitons to compute the final result for each key - this is called the shuffle.

    #### Performance Impact

    * **Shffule is expensive since it involves disk I/O, data serialization, and network I/O**.
    * Internally, result from individual map taks are kept in memory until they can't fit it. Then, these are sorted based on the target partition and written to a single file. On the reduce side, tasks read the relevant sorted blocks.
    * Certain shuffle operations can consume significant amounts of heap memory since they employ in-memory data structures. 

  ### RDD Persistence

  * One of the most important capabilities in Spark is *persisting*(or caching) a dataset in memort across operations. 

  * Caching is a **key tool for iterative algorithms and fast interactive use**.

  * **Spark's cache is fault-tolerant** - if any partition of an RDD is lost, it will automatically be recomputed using the transformations that originally created it.

  * In addition, each persisted RDD can be stored using a different storage level, allowing u, for example, to persist the dataset on disk, **persist it in memory but as serialized Java objects(to save space)**, replicate it across nodes.

    | Storage Level                          | Meaning                                  |
    | -------------------------------------- | ---------------------------------------- |
    | MEMORY_ONLY                            | Store RDD as deserialized Java objects in the JVM. **If the RDD does not fit in memory, some partitions will not be cached and will be recomputed on the fly each time they're needed.** This is the default level. |
    | MEMORY_AND_DISK                        | Store RDD as deserialized Java objects in the JVM. If the RDD does not fit in memory, store the partitions that don't fit on disk, and read them from there when they're needed. |
    | MEMORY_ONLY_SER (Java and Scala)       | Store RDD as ***serialized* Java objects** (one byte array per partition). This is generally **more space-efficient** than deserialized objects, especially when using a [fast serializer](http://spark.apache.org/docs/latest/tuning.html), but more CPU-intensive to read. |
    | MEMORY_AND_DISK_SER (Java and Scala)   | Similar to MEMORY_ONLY_SER, but spill partitions that don't fit in memory to disk instead of recomputing them on the fly each time they're needed. |
    | DISK_ONLY                              | Store the RDD partitions only on disk.   |
    | MEMORY_ONLY_2, MEMORY_AND_DISK_2, etc. | Same as the levels above, but replicate each partition on two cluster nodes. |
    | OFF_HEAP (experimental)                | Similar to MEMORY_ONLY_SER, but store the data in [off-heap memory](http://spark.apache.org/docs/latest/configuration.html#memory-management). This requires off-heap memory to be enabled. |

    * Note: **Spark automatically persists some intermediate data in shuffle operations**, even without *persist*. This is done to avoid recomputing the entire input if a node fails during the shuffle.

  ### Which Storage Level to Choose?

  * Spark's storage levels are meant to provide different **trade-iffs** between memory usage and CPU efficiency.
    * If ur RDDs fit comfortably with MEMORY_ONLY, leave them that way, this is most CPU-efficient option.
    * If not, try using **MEMORY_ONLY_SER** and selecting a fast serialization library.
    * **Don't spill to disk unless the functions that computed ur datasets are expensive, or they filter a large amount of data.** Otherwise, recomputing a partition may be as fast as readibg it from disk
    * Use the replicated storage levels if u want fast fault recovery. **All the storage levels provide full fault tolerance by recomputing lost data**, but the replicated ones let you continue running tasks on the RDD without waiting to recompute a lost partition.

  ### Removing data

  * Spark automatically monitors cache usage on each node and drops out old data partitions in a **least-recently-used(LRU)** fashion.
  * If u would like to manually remove an RDD instead of waiting for it to fall out of cache, use the ```RDD.unpersist()``` method.

## Shared Varaibles

* Normally. when a function passed to a Spark operation is executed on a remote cluster node, it works on separate copies of all the variables used in the function. These variables are copied to each machine, and no updates to the variables on the remote machine are propagated back to the driver program.
* **Supporting general, read-write shared variables across tasks would be inefficient.**
* Spark does provide two limited types of shared variables for two common usage patterns: 
  * broadcast variables
  * accumulators

### Broadcast Variables

* Allowing the programmer to keep a read-only variable cached on each machine rather tahn shipping a copy of it with tasks.

* Spark attempts to distribute broadcast variables using efficient boardcast algorithms to reduce communication cost.

* **Spark actions are executed through a set of stages, separated by distributed "shuffle" operations.** Spark automatically boardcasts the common data needed by tasks within each stage.

  ```scala
  val broadcastVar = sc.broadcast(Array(1, 2, 3))
  broadcastVar.value
  ```

### Accumulators

* Accumulators are variables that are only "added" to through an associative and commutative operation and can therefore be efficiently supported in parallel.

* They can be used to implement counters(as in MapReduce) or sums.

* Spark natively supports accumulators of numeric types, and programmers can add support for new types.

  ```scala
  // numeric ccumulator, SparkContext.doubleAccumulator()
  val accum = sc.longAccumulator("My Accumulator")  
  sc.parallelize(Array(1, 2, 3, 4)).foreach(x => accum.add(x))
  // foreach is an action
  // INFO SparkCOntext: Tasks finished in 0.317106s
  accm.value
  ```

* Programmers can create their own types by subclassing AccumulatorV2. This abstract class has several methods to override.

  * reset: for resetting the accumulator to zero;
  * add: for adding another value into the accumulator
  * merge: for merging another same-type accumulator into this one.
  * ….

* lazy

  ```scala
  val accum = sc.longAccumulator
  data.map { x => accum.add(x); x }
  // Here, accum is still 0 because no actions have caused the map operation to be computed.
  ```



# FYI

* [Spark Programming Guide](http://spark.apache.org/docs/latest/programming-guide.html)