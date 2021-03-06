# RDD 编程指南
[TOC]
## 概览

文档的版本：2.4.0
相关编程语言：Python

在高层看，每个Spark应用程序都由一个驱动程序（driver）组成，该驱动程序是用户使用的主要功能，它可以在集群上执行各种并行操作。
Spark提供的主要抽象是弹性分布式数据集(RDD)，它是跨集群节点分区的元素集合，这些元素可以并行操作。
RDD是通过在Hadoop文件系统(或任何其他Hadoop支持的文件系统)中创建一个文件，或者在驱动程序中创建一个Scala集合，并对其进行转换来创建的。
用户还可以要求Spark将RDD持久化到内存中，以便跨并行操作高效地重用它。最后，RDDs可以自动从节点故障中恢复。

At a high level, every Spark application consists of a driver program that runs the user’s main function and executes various parallel operations on a cluster. The main abstraction Spark provides is a resilient distributed dataset (RDD), which is a collection of elements partitioned across the nodes of the cluster that can be operated on in parallel. RDDs are created by starting with a file in the Hadoop file system (or any other Hadoop-supported file system), or an existing Scala collection in the driver program, and transforming it. Users may also ask Spark to persist an RDD in memory, allowing it to be reused efficiently across parallel operations. Finally, RDDs automatically recover from node failures.

Spark中的第二个抽象是可以在并行操作中使用的共享变量。
默认情况下，当Spark作为不同节点上的一组任务并行运行一个函数时，它将函数中使用的每个变量的副本发送给其它任务。
有时候，需要在任务之间或任务和驱动程序之间共享变量。Spark支持两种类型的共享变量：广播变量（广播变量可用于在所有节点的内存中缓存值）和累加器（累加器是仅“添加”到的变量，如计数器和数学运算）。

A second abstraction in Spark is shared variables that can be used in parallel operations. By default, when Spark runs a function in parallel as a set of tasks on different nodes, it ships a copy of each variable used in the function to each task. Sometimes, a variable needs to be shared across tasks, or between tasks and the driver program. Spark supports two types of shared variables: broadcast variables, which can be used to cache a value in memory on all nodes, and accumulators, which are variables that are only “added” to, such as counters and sums.

本指南以每种Spark支持的语言展示了这些特性。
最容易实践的是，启动Spark的交互式shell（要么是Scala shell bin/spark-shell，要么是Python shell bin/pyspark。

This guide shows each of these features in each of Spark’s supported languages. It is easiest to follow along with if you launch Spark’s interactive shell – either bin/spark-shell for the Scala shell or bin/pyspark for the Python one.

## 连接到Spark

Spark 2.4.0适用于Python 2.7+或Python 3.4+。
它可以使用标准的CPython解释器，因此也可以使用像NumPy这样的C库。
它也适用于pypy2.3+。

Spark 2.4.0 works with Python 2.7+ or Python 3.4+. It can use the standard CPython interpreter, so C libraries like NumPy can be used. It also works with PyPy 2.3+.

在Spark 2.2.0中删除了对Python 2.6的支持。

Python 2.6 support was removed in Spark 2.2.0.

Python中的Spark应用程序可以使用Spark的bin/spark-submit脚本运行，脚本会在运行是包含Spark，也可以在setup.py包含它：

Spark applications in Python can either be run with the bin/spark-submit script which includes Spark at runtime, or by including it in your setup.py as:

```python
install_requires=[
    'pyspark=={site.SPARK_VERSION}'
]
```

要在不安装PySpark的情况下，运行Python的Spark应用程序，请使用位于Spark目录中的bin/spark-submit脚本。
这个脚本将加载Spark的Java/Scala库，并可以向集群提交应用程序。
你还可以使用bin/pyspark启动交互式Python shell。

To run Spark applications in Python without pip installing PySpark, use the bin/spark-submit script located in the Spark directory. This script will load Spark’s Java/Scala libraries and allow you to submit applications to a cluster. You can also use bin/pyspark to launch an interactive Python shell.

如果您希望访问HDFS数据，您需要使用构建好的PySpark连接到您的HDFS版本。
对于常见的HDFS版本，在Spark主页上也可以找到预先构建的包。

If you wish to access HDFS data, you need to use a build of PySpark linking to your version of HDFS. Prebuilt packages are also available on the Spark homepage for common HDFS versions.

最后，您需要将一些Spark类导入到您的程序中。添加以下行:

Finally, you need to import some Spark classes into your program. Add the following line:

```python
from pyspark import SparkContext, SparkConf
```

PySpark在驱动程序和工作程序中都需要相同的Python小版本。
它从PATH中获取默认python版本，PYSPARK_PYTHON可以指定想要使用的Python版本，例如:

PySpark requires the same minor version of Python in both driver and workers. It uses the default python version in PATH, you can specify which version of Python you want to use by PYSPARK_PYTHON, for example:

```shell
$ PYSPARK_PYTHON=python3.4 bin/pyspark
$ PYSPARK_PYTHON=/opt/pypy-2.5/bin/pypy bin/spark-submit examples/src/main/python/pi.py
```

## 初始化Spark

Spark程序必须做的第一件事，是要创建SparkContext对象，它告诉Spark如何访问集群。
要创建SparkContext，首先需要构建一个SparkConf对象，它包含有关应用程序的信息。

The first thing a Spark program must do is to create a SparkContext object, which tells Spark how to access a cluster. To create a SparkContext you first need to build a SparkConf object that contains information about your application.

```python
conf = SparkConf().setAppName(appName).setMaster(master)
sc = SparkContext(conf=conf)
```

appName参数是应用程序在集群UI上显示的名称，master是一个Spark、Mesos或YARN集群URL，或在本地模式下运行的特殊字符串`"local"`。
实际上，在集群上运行时，您不希望在程序中硬编码master参数，而是用spark-submit启动应用程序时在那里设置它。
然而，对于本地测试和单元测试，您可以在进程内传递`"local"`来运行Spark。

The appName parameter is a name for your application to show on the cluster UI. master is a Spark, Mesos or YARN cluster URL, or a special “local” string to run in local mode. In practice, when running on a cluster, you will not want to hardcode master in the program, but rather launch the application with spark-submit and receive it there. However, for local testing and unit tests, you can pass “local” to run Spark in-process.

## 在Shell中使用

在PySpark shell中，已经为您创建了一个特殊的定义好了的SparkContext在变量`sc`中。
您可以使用`--master`参数设置context连接的主机，您还可以将Python .zip、.egg或.py文件通过向`--py-files`传递以逗号分隔的列表添加到运行时路径。
您还可以通过为`--Packages`参数提供以逗号分隔的Maven列表，将依赖项（例如 Spark包）添加到shell会话中。
任何额外的存储库，可能需要依赖的包（例如 Sonatype），可以传递到`--repository`参数中。
Spark包中列出的任何Python依赖项，必要时，使用pip安装（requirements.txt清单列出的包）。
例如，要在恰好四个核上运行bin/pyspark，请使用:

In the PySpark shell, a special interpreter-aware SparkContext is already created for you, in the variable called sc. Making your own SparkContext will not work. You can set which master the context connects to using the --master argument, and you can add Python .zip, .egg or .py files to the runtime path by passing a comma-separated list to --py-files. You can also add dependencies (e.g. Spark Packages) to your shell session by supplying a comma-separated list of Maven coordinates to the --packages argument. Any additional repositories where dependencies might exist (e.g. Sonatype) can be passed to the --repositories argument. Any Python dependencies a Spark package has (listed in the requirements.txt of that package) must be manually installed using pip when necessary. For example, to run bin/pyspark on exactly four cores, use:

```shell
$ ./bin/pyspark --master local[4]
```

或者，添加code.py到搜索路径（以便以后能够`import code`），用：

Or, to also add code.py to the search path (in order to later be able to import code), use:

```shell
$ ./bin/pyspark --master local[4] --py-files code.py
```

查看完整的选项列表，运行`pyspark --help`。在后台，pyspark调用更通用的spark-submit脚本。

For a complete list of options, run pyspark --help. Behind the scenes, pyspark invokes the more general spark-submit script.

也可以在IPython中启动PySpark shell，增强的Python解释器。
PySpark可以与IPython 1.0.0及更高版本一起工作。
要使用IPython，在运行bin/pyspark时，将PYSPARK_DRIVER_PYTHON变量设置为IPython：

It is also possible to launch the PySpark shell in IPython, the enhanced Python interpreter. PySpark works with IPython 1.0.0 and later. To use IPython, set the PYSPARK_DRIVER_PYTHON variable to ipython when running bin/pyspark:

```shell
$ PYSPARK_DRIVER_PYTHON=ipython ./bin/pyspark
```

使用Jupyter notebook（以前称为IPython notebook），

To use the Jupyter notebook (previously known as the IPython notebook),

```shell
$ PYSPARK_DRIVER_PYTHON=jupyter PYSPARK_DRIVER_PYTHON_OPTS=notebook ./bin/pyspark
```

您可以通过设置PYSPARK_DRIVER_PYTHON_OPTS自定义ipython或jupyter命令。

You can customize the ipython or jupyter commands by setting PYSPARK_DRIVER_PYTHON_OPTS.

在Jupyter Notebook服务启动之后，您可以从“Files”选项卡创建一个新“python2”的notebook。
在notebook中，在开始从Jupyter notebook中尝试Spark之前，可以将命令`%pylab`作为notebook的一部分内嵌的输入。

After the Jupyter Notebook server is launched, you can create a new “Python 2” notebook from the “Files” tab. Inside the notebook, you can input the command %pylab inline as part of your notebook before you start to try Spark from the Jupyter notebook.

## 弹性分布式数据集（RDDs）

Spark就是围绕弹性分布式数据集（RDD）的概念展开，它是可容错的元素集合，可执行并行操作。
有两种方法可以创建rdd：并行化驱动程序中的已经有的集合，或引用外部存储系统中的数据集，例如共享文件系统、HDFS、HBase或任何提供Hadoop InputFormat的数据源。

Spark revolves around the concept of a resilient distributed dataset (RDD), which is a fault-tolerant collection of elements that can be operated on in parallel. There are two ways to create RDDs: parallelizing an existing collection in your driver program, or referencing a dataset in an external storage system, such as a shared filesystem, HDFS, HBase, or any data source offering a Hadoop InputFormat.

## 并行集合

并行集合是在驱动程序中，通过现有的可迭代对象或集合上调用SparkContext的parallelize方法来创建的。
集合的元素被复制出来，以形成可并行操作的分布式数据集。
例如，下面是如何创建一个并行的集合来保存数字1到5：

Parallelized collections are created by calling SparkContext’s parallelize method on an existing iterable or collection in your driver program. The elements of the collection are copied to form a distributed dataset that can be operated on in parallel. For example, here is how to create a parallelized collection holding the numbers 1 to 5:

```python
data = [1, 2, 3, 4, 5]
distData = sc.parallelize(data)
```

一旦创建，分布式数据集（distData）就可以进行并行操作。例如，我们可以调用`distData.reduce(a, b: a + b)`的元素相加。稍后我们将描述对分布式数据集的操作。

Once created, the distributed dataset (distData) can be operated on in parallel. For example, we can call distData.reduce(lambda a, b: a + b) to add up the elements of the list. We describe operations on distributed datasets later on.

并行集合的一个重要参数是分区（partitions）的值，用来指定数据集切割成多少份。
Spark将为集群的每个分区运行一个任务。
通常，集群中的每个CPU需要2-4个分区。
通常，Spark试图根据集群自动设置分区的数量。
但是，您也可以通过将它作为第二个参数传递给并行化来手动设置它（例如`sc.parallelize(data,10)`）。
注意:代码中的一些地方使用slices（分区的同义词）来维护向后兼容性。

One important parameter for parallel collections is the number of partitions to cut the dataset into. Spark will run one task for each partition of the cluster. Typically you want 2-4 partitions for each CPU in your cluster. Normally, Spark tries to set the number of partitions automatically based on your cluster. However, you can also set it manually by passing it as a second parameter to parallelize (e.g. sc.parallelize(data, 10)). Note: some places in the code use the term slices (a synonym for partitions) to maintain backward compatibility.

## 外部数据源

PySpark可以从Hadoop支持的任何存储源创建分布式数据集，包括本地文件系统，HDFS，Cassandra，HBase，Amazon S3等。Spark支持文本文件，有序文件，或其他Hadoop输入格式。

PySpark can create distributed datasets from any storage source supported by Hadoop, including your local file system, HDFS, Cassandra, HBase, Amazon S3, etc. Spark supports text files, SequenceFiles, and any other Hadoop InputFormat.

文本文件RDDs可以使用SparkContext的textFile方法，该方法接受文件的URI（要么是本地路径，要么是hdfs://、s3a://等URI）和将其读取为行集合。下面是一个调用示例：

Text file RDDs can be created using SparkContext’s textFile method. This method takes an URI for the file (either a local path on the machine, or a hdfs://, s3a://, etc URI) and reads it as a collection of lines. Here is an example invocation:

```python
>>> distFile = sc.textFile("data.txt")
```

一旦创建，就可以通过数据集操作对distFile进行操作。例如，我们可以按如下方式使用map和reduce操作将所有行的大小相加：distFile.map(lambda s: len(s)).reduce(lambda a, b: a + b)。

Once created, distFile can be acted on by dataset operations. For example, we can add up the sizes of all the lines using the map and reduce operations as follows: distFile.map(lambda s: len(s)).reduce(lambda a, b: a + b).

关于使用Spark读取文件的注意事项：

Some notes on reading files with Spark:

如果使用在本地文件系统上路径，文件也必须在worker节点上的相同路径上可访问。要么将文件复制到所有workers，要么使用网络挂载的共享文件系统。

If using a path on the local filesystem, the file must also be accessible at the same path on worker nodes. Either copy the file to all workers or use a network-mounted shared file system.

Spark所有基于文件的输入方法，包括文本文件，支持目录，压缩文件，以及通配符。
例如，您可以使用textFile("/my/directory")、textFile("/my/directory/*.txt")和textFile("/my/directory/*.gz")。

All of Spark’s file-based input methods, including textFile, support running on directories, compressed files, and wildcards as well. For example, you can use `textFile("/my/directory")`, `textFile("/my/directory/*.txt")`, and `textFile("/my/directory/*.gz")`.

textFile方法同样还有可选的第二个参数来控制文件的分区数量。
默认情况下，Spark为文件的每个块创建一个分区（块在HDFS中默认为128MB），但是您也可以通过传递更大的值来请求更多的分区。
注意，分区不能少于块。

The textFile method also takes an optional second argument for controlling the number of partitions of the file. By default, Spark creates one partition for each block of the file (blocks being 128MB by default in HDFS), but you can also ask for a higher number of partitions by passing a larger value. Note that you cannot have fewer partitions than blocks.

除了文本文件，Spark的Python API还支持其他几种数据格式:

Apart from text files, Spark’s Python API also supports several other data formats:

SparkContext.wholeTextFiles允许您读取包含多个小文本文件的目录，并将它们作为(filename, content)对返回。
这与textFile形成对比，后者将在每个文件中每行返回一条记录。

SparkContext.wholeTextFiles lets you read a directory containing multiple small text files, and returns each of them as (filename, content) pairs. This is in contrast with textFile, which would return one record per line in each file.

RDD.saveAsPickleFile和SparkContext.pickleFile支持以包含特制的Python对象的简单格式保存RDD。
批处理用于pickle序列化，默认批大小为10。

RDD.saveAsPickleFile and SparkContext.pickleFile support saving an RDD in a simple format consisting of pickled Python objects. Batching is used on pickle serialization, with default batch size 10.

## SequenceFile和Hadoop输入/输出格式

注意，这个特性目前被标记为实验性的，并且是为高级用户准备的。
将来它可能会被基于Spark SQL的读/写支持所替代，在这种情况下，Spark SQL是首选的方法。

Note this feature is currently marked Experimental and is intended for advanced users. It may be replaced in future with read/write support based on Spark SQL, in which case Spark SQL is the preferred approach.

### Writable Support

PySpark SequenceFile support loads an RDD of key-value pairs within Java, converts Writables to base Java types, and pickles the resulting Java objects using Pyrolite. When saving an RDD of key-value pairs to SequenceFile, PySpark does the reverse. It unpickles Python objects into Java objects and then converts them to Writables. The following Writables are automatically converted:


|Writable Type | Python Type |
|-|-|
|Text|unicode str|
|IntWritable|int|
|FloatWritable|float|
|DoubleWritable|float|
|BooleanWritable|bool|
|BytesWritable|byte array|
|NullWritable|None|
|MapWritable|dict|

Arrays are not handled out-of-the-box. Users need to specify custom ArrayWritable subtypes when reading or writing. When writing, users also need to specify custom converters that convert arrays to custom ArrayWritable subtypes. When reading, the default converter will convert custom ArrayWritable subtypes to Java Object[], which then get pickled to Python tuples. To get Python array.array for arrays of primitive types, users need to specify custom converters.

### Saving and Loading SequenceFiles

Similarly to text files, SequenceFiles can be saved and loaded by specifying the path. The key and value classes can be specified, but for standard Writables this is not required.

```python
>>> rdd = sc.parallelize(range(1, 4)).map(lambda x: (x, "a" * x))
>>> rdd.saveAsSequenceFile("path/to/file")
>>> sorted(sc.sequenceFile("path/to/file").collect())
[(1, u'a'), (2, u'aa'), (3, u'aaa')]
```

### Saving and Loading Other Hadoop Input/Output Formats

PySpark can also read any Hadoop InputFormat or write any Hadoop OutputFormat, for both ‘new’ and ‘old’ Hadoop MapReduce APIs. If required, a Hadoop configuration can be passed in as a Python dict. Here is an example using the Elasticsearch ESInputFormat:

```shell
$ ./bin/pyspark --jars /path/to/elasticsearch-hadoop.jar
>>> conf = {"es.resource" : "index/type"}  # assume Elasticsearch is running on localhost defaults
>>> rdd = sc.newAPIHadoopRDD("org.elasticsearch.hadoop.mr.EsInputFormat",
                             "org.apache.hadoop.io.NullWritable",
                             "org.elasticsearch.hadoop.mr.LinkedMapWritable",
                             conf=conf)
>>> rdd.first()  # the result is a MapWritable that is converted to a Python dict
(u'Elasticsearch ID',
 {u'field1': True,
  u'field2': u'Some Text',
  u'field3': 12345})
```

Note that, if the InputFormat simply depends on a Hadoop configuration and/or input path, and the key and value classes can easily be converted according to the above table, then this approach should work well for such cases.

If you have custom serialized binary data (such as loading data from Cassandra / HBase), then you will first need to transform that data on the Scala/Java side to something which can be handled by Pyrolite’s pickler. A Converter trait is provided for this. Simply extend this trait and implement your transformation code in the convert method. Remember to ensure that this class, along with any dependencies required to access your InputFormat, are packaged into your Spark job jar and included on the PySpark classpath.

See the Python examples and the Converter examples for examples of using Cassandra / HBase InputFormat and OutputFormat with custom converters.

## RDD 操作

RDDs支持两种类型的操作：
转换（transformations），从现有数据集创建新数据集的方法；
动作（actions），在数据集中运行计算之后，返回一个值给驱动程序。
例如，map是一个转换，它通过将一个函数传递每个数据集元素使用，并返回一个表示结果的新的RDD。
另一方面，reduce是一个动作，它使用某个函数聚合RDD的所有元素，并将最终结果返回给驱动程序（尽管还有一个返回分布式数据集的并行reduceByKey）。

RDDs support two types of operations: transformations, which create a new dataset from an existing one, and actions, which return a value to the driver program after running a computation on the dataset. For example, map is a transformation that passes each dataset element through a function and returns a new RDD representing the results. On the other hand, reduce is an action that aggregates all the elements of the RDD using some function and returns the final result to the driver program (although there is also a parallel reduceByKey that returns a distributed dataset).

Spark中的所有转换都是惰性的，因为它们不会立即计算结果。
相反，他们只记得应用于某些基本数据集（例如一个文件）的转换。
只有当一个操作需要将结果返回到驱动程序时，才会计算转换。
这种设计使Spark运行更加高效。
例如，我们可以知道通过map创建的数据集，将在reduce中使用，只将reduce的结果返回给驱动程序，而不是更大的映射（mapped）数据集。

All transformations in Spark are lazy, in that they do not compute their results right away. Instead, they just remember the transformations applied to some base dataset (e.g. a file). The transformations are only computed when an action requires a result to be returned to the driver program. This design enables Spark to run more efficiently. For example, we can realize that a dataset created through map will be used in a reduce and return only the result of the reduce to the driver, rather than the larger mapped dataset.

默认情况下，每次在每个已转换的RDD上运行操作时，都会重新计算它。
但是，您也可以使用持久化（或缓存）方法将RDD持久化到内存中，在这种情况下，Spark将在集群中保留元素，以便在下一次查询时更快地访问它。
还支持在磁盘上持久化rdd，或跨多个节点复制rdd。

By default, each transformed RDD may be recomputed each time you run an action on it. However, you may also persist an RDD in memory using the persist (or cache) method, in which case Spark will keep the elements around on the cluster for much faster access the next time you query it. There is also support for persisting RDDs on disk, or replicated across multiple nodes.

为了说明RDD的基本功能，请考虑下面的简单程序:

To illustrate RDD basics, consider the simple program below:

```python
lines = sc.textFile("data.txt")
lineLengths = lines.map(lambda s: len(s))
totalLength = lineLengths.reduce(lambda a, b: a + b)
```

第一行从外部文件定义了一个基本的RDD。
此数据集没有加载到内存中或其它地方：lines只是指向文件的指针。
第二行将lineLengths定义为映射转换的结果。
同样，由于惰性运算，linelength不会立即计算。
最后，我们运行reduce，这是一个操作。
此时，Spark将计算分解为在不同机器上运行的任务，每台机器都运行它的那部分map和一个局部计算，只返回它给驱动程序的结果。

The first line defines a base RDD from an external file. This dataset is not loaded in memory or otherwise acted on: lines is merely a pointer to the file. The second line defines lineLengths as the result of a map transformation. Again, lineLengths is not immediately computed, due to laziness. Finally, we run reduce, which is an action. At this point Spark breaks the computation into tasks to run on separate machines, and each machine runs both its part of the map and a local reduction, returning only its answer to the driver program.

如果以后我们还想再使用lineLengths，我们可以添加：

If we also wanted to use lineLengths again later, we could add:

```
lineLengths.persist()
```

在reduce之前，这将会在第一次计算后将lineLengths保存在内存中。

before the reduce, which would cause lineLengths to be saved in memory after the first time it is computed.

## 传递函数给Spark

Spark的API在很大程度上依赖于在驱动程序中传递函数以在集群上运行。
有三种推荐的方法：

Spark’s API relies heavily on passing functions in the driver program to run on the cluster. There are three recommended ways to do this:

Lambda表达式，可以用于写成表达式的简单函数。（Lambdas不支持多语句函数或不返回值的语句。）
函数内部本地defs，以获得更长的代码能力。
模块中的顶级函数。
例如，要传递一个长度超过lambda所支持的函数，请考虑以下代码:

Lambda expressions, for simple functions that can be written as an expression. (Lambdas do not support multi-statement functions or statements that do not return a value.)
Local defs inside the function calling into Spark, for longer code.
Top-level functions in a module.
For example, to pass a longer function than can be supported using a lambda, consider the code below:

```python
"""MyScript.py"""
if __name__ == "__main__":
    def myFunc(s):
        words = s.split(" ")
        return len(words)

    sc = SparkContext(...)
    sc.textFile("file.txt").map(myFunc)
```

注意，虽然也可以在类实例中传递方法的引用（与单例对象相反），这需要发送包含该类的对象和方法。
例如，考虑：

Note that while it is also possible to pass a reference to a method in a class instance (as opposed to a singleton object), this requires sending the object that contains that class along with the method. For example, consider:

```python
class MyClass(object):
    def func(self, s):
        return s
    def doStuff(self, rdd):
        return rdd.map(self.func)
```

在这里，如果我们创建一个新的MyClass并调用doStuff，其中的map引用那个MyClass实例的func方法，所以需要将整个对象发送到集群。

Here, if we create a new MyClass and call doStuff on it, the map inside there references the func method of that MyClass instance, so the whole object needs to be sent to the cluster.

同理，访问外部对象的字段会引用整个对象：

In a similar way, accessing fields of the outer object will reference the whole object:

```python
class MyClass(object):
    def __init__(self):
        self.field = "Hello"
    def doStuff(self, rdd):
        return rdd.map(lambda s: self.field + s)
```

为了避免这个问题，最简单的方法是将字段复制到本地变量中，而不是从外部访问它：

To avoid this issue, the simplest way is to copy field into a local variable instead of accessing it externally:

```python
def doStuff(self, rdd):
    field = self.field
    return rdd.map(lambda s: field + s)
```

## 理解闭包

Spark的难点之一是理解跨集群执行代码时变量和方法的作用域和生命周期。
在变量作用域之外修改变量的RDD操作经常引起混乱。
在下面的示例中，我们将使用foreach()来增加counter的代码，但其他操作也可能出现类似的问题。

One of the harder things about Spark is understanding the scope and life cycle of variables and methods when executing code across a cluster. RDD operations that modify variables outside of their scope can be a frequent source of confusion. In the example below we’ll look at code that uses foreach() to increment a counter, but similar issues can occur for other operations as well.

### Example

考虑下面简单的RDD元素sum，根据是否在同一个JVM中执行，其行为可能有所不同。
一个常见的用法是在本地模式下运行Spark（--master = local[n]），而不是将Spark应用程序部署到集群（例如，通过spark-submit到YARN）：

Consider the naive RDD element sum below, which may behave differently depending on whether execution is happening within the same JVM. A common example of this is when running Spark in local mode (--master = local[n]) versus deploying a Spark application to a cluster (e.g. via spark-submit to YARN):

```python
counter = 0
rdd = sc.parallelize(data)

# Wrong: Don't do this!!
def increment_counter(x):
    global counter
    counter += x
rdd.foreach(increment_counter)

print("Counter value: ", counter)
```

## 本地 VS 集群

上述代码的行为是未定义的，将会无法正常工作。
为了执行作业，Spark将RDD操作的处理分解为任务，每个任务由执行者（executor）执行。
在执行之前，Spark计算任务的闭包。
闭包是那些必须对执行程序可见的变量和方法，以便在RDD上执行其计算（在本例中是foreach()）。
这个闭包被序列化并发送到每个执行者。

The behavior of the above code is undefined, and may not work as intended. To execute jobs, Spark breaks up the processing of RDD operations into tasks, each of which is executed by an executor. Prior to execution, Spark computes the task’s closure. The closure is those variables and methods which must be visible for the executor to perform its computations on the RDD (in this case foreach()). This closure is serialized and sent to each executor.

发送到每个执行者的闭包中的变量现在是副本形式，因此，当在foreach函数中引用counter时，它不再是驱动节点上的counter。

The variables within the closure sent to each executor are now copies and thus, when counter is referenced within the foreach function, it’s no longer the counter on the driver node. There is still a counter in the memory of the driver node but this is no longer visible to the executors! The executors only see the copy from the serialized closure. Thus, the final value of counter will still be zero since all operations on counter were referencing the value within the serialized closure.

在本地模式下，某些情况，foreach函数实际将在与驱动程序相同的JVM中执行，并引用相同的原始counter，并可能实际更新它。

In local mode, in some circumstances, the foreach function will actually execute within the same JVM as the driver and will reference the same original counter, and may actually update it.

要确保在这类场景中定义良好的行为，应该使用累加器（Accumulator）。
Spark中的累加器专门用于在集群中的工作节点之间分割执行时安全地更新变量。
本指南的累加器部分会更详细地讨论了它。

To ensure well-defined behavior in these sorts of scenarios one should use an Accumulator. Accumulators in Spark are used specifically to provide a mechanism for safely updating a variable when execution is split up across worker nodes in a cluster. The Accumulators section of this guide discusses these in more detail.

一般来说，闭包（结构如循环或局部定义的方法），不应该用于改变某些全局状态。Spark不定义或保证对闭包外部引用的对象的修改行为。
一些执行此操作的代码可能在本地模式下能工作，但这是偶然的，而且这些代码在分布式模式下的行为不会像预期的那样。
如果需要一些全局聚合，则使用累加器。

In general, closures - constructs like loops or locally defined methods, should not be used to mutate some global state. Spark does not define or guarantee the behavior of mutations to objects referenced from outside of closures. Some code that does this may work in local mode, but that’s just by accident and such code will not behave as expected in distributed mode. Use an Accumulator instead if some global aggregation is needed.

## 打印一个RDD

另一个常见的习惯用法是尝试使用RDD.foreach(println)或RDD.map(println)打印RDD的元素。
在一台机器上，这将生成预期的输出并打印所有RDD元素。
但是，在集群模式下，执行者调用的stdout是现在正在写入执行者的stdout，而不是驱动程序上的stdout，所以驱动程序上的stdout不会显示这些的！
要打印驱动程序上的所有元素，可以使用collect()方法首先将RDD带到驱动节点，例如：RDD.collect().foreach(println)。
但是，这可能会导致驱动程序耗尽内存，因为collect()会将所有RDD提取到一台机器上；
如果只需要打印RDD的几个元素，更安全的方法是使用rdd.take(100).foreach(println)。

Another common idiom is attempting to print out the elements of an RDD using rdd.foreach(println) or rdd.map(println). On a single machine, this will generate the expected output and print all the RDD’s elements. However, in cluster mode, the output to stdout being called by the executors is now writing to the executor’s stdout instead, not the one on the driver, so stdout on the driver won’t show these! To print all elements on the driver, one can use the collect() method to first bring the RDD to the driver node thus: rdd.collect().foreach(println). This can cause the driver to run out of memory, though, because collect() fetches the entire RDD to a single machine; if you only need to print a few elements of the RDD, a safer approach is to use the take(): rdd.take(100).foreach(println).

## 使用键-值对

虽然大多数Spark操作都是在包含任何类型对象的RDDs上进行的，但是有一些特殊操作只在键-值对的RDDs上可用。
最常见的是分布式的“shuffle”操作，例如通过键对元素进行分组或聚合。

While most Spark operations work on RDDs containing any type of objects, a few special operations are only available on RDDs of key-value pairs. The most common ones are distributed “shuffle” operations, such as grouping or aggregating the elements by a key.

在Python中，这些操作在包含内置Python元组(1,2)的rdd上工作。
只需创建这样的元组，然后调用所需的操作。

In Python, these operations work on RDDs containing built-in Python tuples such as (1, 2). Simply create such tuples and then call your desired operation.

例如，下面的代码使用键-值对上的reduceByKey操作来计算每行文本在文件中出现的次数:

For example, the following code uses the reduceByKey operation on key-value pairs to count how many times each line of text occurs in a file:

```python
lines = sc.textFile("data.txt")
pairs = lines.map(lambda s: (s, 1))
counts = pairs.reduceByKey(lambda a, b: a + b)
```

例如，我们还可以使用counts.sortByKey()按字母顺序进行排序，最后使用count.collect()将它们作为对象列表带回驱动程序。

We could also use counts.sortByKey(), for example, to sort the pairs alphabetically, and finally counts.collect() to bring them back to the driver program as a list of objects.

## Transformations

下表列出了Spark支持的一些常见转换。有关详细信息，请参阅RDD API和RDD函数部分。

The following table lists some of the common transformations supported by Spark. Refer to the RDD API doc (Scala, Java, Python, R) and pair RDD functions doc (Scala, Java) for details.

|Transformation|Meaning|
|-|-|
|map(func)|通过传递函数func到每个元素，返回一个新的分布式数据集。
filter(func)|通过选择func的返回为true的源元素，返回一个新的数据集。
flatMap(func)|与map类似，但是每个输入项都可以映射到0或多个输出项（因此func应该返回Seq，而不是单个项）。
mapPartitions(func)|类似map，但是在RDD的每个分区（块）上单独运行，所以当在类型为T的RDD上运行时，func必须是`Iterator<T> => Iterator<U>`。
mapPartitionsWithIndex(func)|类似于mapPartitions，但也为func提供了一个表示分区索引的整数值，所以在类型为T的RDD上运行时，func必须是`(Int, Iterator<T>) => Iterator<U>`。
sample(withReplacement, fraction, seed)|Sample a fraction fraction of the data, with or without replacement, using a given random number generator seed.
union(otherDataset)|	Return a new dataset that contains the union of the elements in the source dataset and the argument.
intersection(otherDataset)|	Return a new RDD that contains the intersection of elements in the source dataset and the argument.
distinct([numPartitions]))|	Return a new dataset that contains the distinct elements of the source dataset.
groupByKey([numPartitions])|当调用是对(K, V)对的数据集时，返回(K, Iterable)对的数据集。注意:如果您分组是为了对每个键执行聚合（如总和或平均值），那么使用reduceByKey或aggregateByKey将获得更好的性能。注意:默认情况下，输出中的并行程度取决于父RDD的分区数量。您可以传递一个可选的numPartitions参数来设置不同数量的任务。
reduceByKey(func, [numPartitions])|当调用是对(K、V)的数据集，返回一个数据集(K、V)，每个值都是使用给定的reduce函数func聚合的，必须`(V,V) => V`型。像groupByKey，reduce任务的数量通过一个可选的第二个参数配置。
aggregateByKey(zeroValue)(seqOp, combOp, [numPartitions])|	When called on a dataset of (K, V) pairs, returns a dataset of (K, U) pairs where the values for each key are aggregated using the given combine functions and a neutral "zero" value. Allows an aggregated value type that is different than the input value type, while avoiding unnecessary allocations. Like in groupByKey, the number of reduce tasks is configurable through an optional second argument.
sortByKey([ascending], [numPartitions])|	When called on a dataset of (K, V) pairs where K implements Ordered, returns a dataset of (K, V) pairs sorted by keys in ascending or descending order, as specified in the boolean ascending argument.
join(otherDataset, [numPartitions])|	When called on datasets of type (K, V) and (K, W), returns a dataset of (K, (V, W)) pairs with all pairs of elements for each key. Outer joins are supported through leftOuterJoin, rightOuterJoin, and fullOuterJoin.
cogroup(otherDataset, [numPartitions])|	When called on datasets of type (K, V) and (K, W), returns a dataset of (K, (Iterable<V>, Iterable<W>)) tuples. This operation is also called groupWith.
cartesian(otherDataset)|	When called on datasets of types T and U, returns a dataset of (T, U) pairs (all pairs of elements).
pipe(command, [envVars])|	Pipe each partition of the RDD through a shell command, e.g. a Perl or bash script. RDD elements are written to the process's stdin and lines output to its stdout are returned as an RDD of strings.
coalesce(numPartitions)|	Decrease the number of partitions in the RDD to numPartitions. Useful for running operations more efficiently after filtering down a large dataset.
repartition(numPartitions)|	Reshuffle the data in the RDD randomly to create either more or fewer partitions and balance it across them. This always shuffles all data over the network.
repartitionAndSortWithinPartitions(partitioner)|	Repartition the RDD according to the given partitioner and, within each resulting partition, sort records by their keys. This is more efficient than calling repartition and then sorting within each partition because it can push the sorting down into the shuffle machinery.


## Actions

下表列出了Spark支持的一些常见操作。有关详细信息，请参阅RDD API和RDD函数部分。

The following table lists some of the common actions supported by Spark. Refer to the RDD API doc (Scala, Java, Python, R)and pair RDD functions doc (Scala, Java) for details.

|Action|Meaning|
|-|-|
reduce(func)|使用函数func聚合数据集的元素（接受两个参数并返回一个）。这个函数应该是可交换的和相联的，这样才能正确地并行计算。
collect()|在驱动程序中以数组的形式返回数据集的所有元素。这通常在过滤器或其他操作返回足够小的数据子集之后非常有用。
count()|返回数据集中元素的数量。
first()|返回数据集的第一个元素（类似于take(1)）。
take(n)|返回包含数据集的前n个元素的数组。
takeSample(withReplacement, num, [seed])|返回一个数组，该数组带有数据集的num个元素的随机样本，可以替换也可以不替换，可以预先指定随机数生成器种子。
takeOrdered(n, [ordering])|	Return the first n elements of the RDD using either their natural order or a custom comparator.
saveAsTextFile(path)|将数据集的元素写入文本文件（或一组文本文件）给定的本地目录、HDFS或者其他hadoop支持的文件系统。Spark将对每个元素调用toString，将其转换为文件中的一行文本。
saveAsSequenceFile(path)| （Java and Scala）在本地文件系统、HDFS或任何Hadoop支持的文件系统中的给定路径中，将数据集的元素写成Hadoop SequenceFile。这在实现Hadoop可写接口的键-值对的RDDs上是可用的。在Scala中，它也可以用于隐式转换为可写的类型（Spark包括对Int、Double、String等基本类型的转换）。
saveAsObjectFile(path)| (Java and Scala)	Write the elements of the dataset in a simple format using Java serialization, which can then be loaded using SparkContext.objectFile().
countByKey()|仅在类型为(K, V)的RDDs上可用。返回一个hashmap (K, Int)对和每个键的计数。
foreach(func)|在数据集的每个元素上运行func函数。这通常是针对其它作用，如更新累加器或与外部存储系统交互。注意:修改在foreach()之外的累加器以外的变量可能会导致未定义的行为。有关更多细节，请参见理解闭包。

park RDD API还公开了一些操作的异步版本，比如foreachAsync和foreach，它将立即向调用方返回一个futuresponse，而不是在操作完成前阻塞。这可以用于管理或等待操作的异步执行。

The Spark RDD API also exposes asynchronous versions of some actions, like foreachAsync for foreach, which immediately return a FutureAction to the caller instead of blocking on completion of the action. This can be used to manage or wait for the asynchronous execution of the action.

## Shuffle（洗牌） operations
Certain operations within Spark trigger an event known as the shuffle. The shuffle is Spark’s mechanism for re-distributing data so that it’s grouped differently across partitions. This typically involves copying data across executors and machines, making the shuffle a complex and costly operation.

### Background
To understand what happens during the shuffle we can consider the example of the reduceByKey operation. The reduceByKey operation generates a new RDD where all values for a single key are combined into a tuple - the key and the result of executing a reduce function against all values associated with that key. The challenge is that not all values for a single key necessarily reside on the same partition, or even the same machine, but they must be co-located to compute the result.

In Spark, data is generally not distributed across partitions to be in the necessary place for a specific operation. During computations, a single task will operate on a single partition - thus, to organize all the data for a single reduceByKey reduce task to execute, Spark needs to perform an all-to-all operation. It must read from all partitions to find all the values for all keys, and then bring together values across partitions to compute the final result for each key - this is called the shuffle.

Although the set of elements in each partition of newly shuffled data will be deterministic, and so is the ordering of partitions themselves, the ordering of these elements is not. If one desires predictably ordered data following shuffle then it’s possible to use:

mapPartitions to sort each partition using, for example, .sorted
repartitionAndSortWithinPartitions to efficiently sort partitions while simultaneously repartitioning
sortBy to make a globally ordered RDD
Operations which can cause a shuffle include repartition operations like repartition and coalesce, ‘ByKey operations (except for counting) like groupByKey and reduceByKey, and join operations like cogroup and join.

### Performance Impact（性能影响）
The Shuffle is an expensive operation since it involves disk I/O, data serialization, and network I/O. To organize data for the shuffle, Spark generates sets of tasks - map tasks to organize the data, and a set of reduce tasks to aggregate it. This nomenclature comes from MapReduce and does not directly relate to Spark’s map and reduce operations.

Internally, results from individual map tasks are kept in memory until they can’t fit. Then, these are sorted based on the target partition and written to a single file. On the reduce side, tasks read the relevant sorted blocks.

Certain shuffle operations can consume significant amounts of heap memory since they employ in-memory data structures to organize records before or after transferring them. Specifically, reduceByKey and aggregateByKey create these structures on the map side, and 'ByKey operations generate these on the reduce side. When data does not fit in memory Spark will spill these tables to disk, incurring the additional overhead of disk I/O and increased garbage collection.

Shuffle also generates a large number of intermediate files on disk. As of Spark 1.3, these files are preserved until the corresponding RDDs are no longer used and are garbage collected. This is done so the shuffle files don’t need to be re-created if the lineage is re-computed. Garbage collection may happen only after a long period of time, if the application retains references to these RDDs or if GC does not kick in frequently. This means that long-running Spark jobs may consume a large amount of disk space. The temporary storage directory is specified by the spark.local.dir configuration parameter when configuring the Spark context.

Shuffle behavior can be tuned by adjusting a variety of configuration parameters. See the ‘Shuffle Behavior’ section within the Spark Configuration Guide.

## RDD Persistence（持久化）
One of the most important capabilities in Spark is persisting (or caching) a dataset in memory across operations. When you persist an RDD, each node stores any partitions of it that it computes in memory and reuses them in other actions on that dataset (or datasets derived from it). This allows future actions to be much faster (often by more than 10x). Caching is a key tool for iterative algorithms and fast interactive use.

You can mark an RDD to be persisted using the persist() or cache() methods on it. The first time it is computed in an action, it will be kept in memory on the nodes. Spark’s cache is fault-tolerant – if any partition of an RDD is lost, it will automatically be recomputed using the transformations that originally created it.

In addition, each persisted RDD can be stored using a different storage level, allowing you, for example, to persist the dataset on disk, persist it in memory but as serialized Java objects (to save space), replicate it across nodes. These levels are set by passing a StorageLevel object (Scala, Java, Python) to persist(). The cache() method is a shorthand for using the default storage level, which is StorageLevel.MEMORY_ONLY (store deserialized objects in memory). The full set of storage levels is:

Storage Level|	Meaning
-|-
MEMORY_ONLY|	Store RDD as deserialized Java objects in the JVM. If the RDD does not fit in memory, some partitions will not be cached and will be recomputed on the fly each time they're needed. This is the default level.
MEMORY_AND_DISK|	Store RDD as deserialized Java objects in the JVM. If the RDD does not fit in memory, store the partitions that don't fit on disk, and read them from there when they're needed.
MEMORY_ONLY_SER|    (Java and Scala)	Store RDD as serialized Java objects (one byte array per partition). This is generally more space-efficient than deserialized objects, especially when using a fast serializer, but more CPU-intensive to read.
MEMORY_AND_DISK_SER|    (Java and Scala)	Similar to MEMORY_ONLY_SER, but spill partitions that don't fit in memory to disk instead of recomputing them on the fly each time they're needed.
DISK_ONLY|	Store the RDD partitions only on disk.
MEMORY_ONLY_2, MEMORY_AND_DISK_2, etc.|	Same as the levels above, but replicate each partition on two cluster nodes.
OFF_HEAP (experimental)|	Similar to MEMORY_ONLY_SER, but store the data in off-heap memory. This requires off-heap memory to be enabled.
Note: In Python, stored objects will always be serialized with the Pickle library, so it does not matter whether you choose a serialized level. The available storage levels in Python include MEMORY_ONLY, MEMORY_ONLY_2, MEMORY_AND_DISK, MEMORY_AND_DISK_2, DISK_ONLY, and DISK_ONLY_2.

Spark also automatically persists some intermediate data in shuffle operations (e.g. reduceByKey), even without users calling persist. This is done to avoid recomputing the entire input if a node fails during the shuffle. We still recommend users call persist on the resulting RDD if they plan to reuse it.

## Which Storage Level to Choose?
Spark’s storage levels are meant to provide different trade-offs between memory usage and CPU efficiency. We recommend going through the following process to select one:

If your RDDs fit comfortably with the default storage level (MEMORY_ONLY), leave them that way. This is the most CPU-efficient option, allowing operations on the RDDs to run as fast as possible.

If not, try using MEMORY_ONLY_SER and selecting a fast serialization library to make the objects much more space-efficient, but still reasonably fast to access. (Java and Scala)

Don’t spill to disk unless the functions that computed your datasets are expensive, or they filter a large amount of the data. Otherwise, recomputing a partition may be as fast as reading it from disk.

Use the replicated storage levels if you want fast fault recovery (e.g. if using Spark to serve requests from a web application). All the storage levels provide full fault tolerance by recomputing lost data, but the replicated ones let you continue running tasks on the RDD without waiting to recompute a lost partition.

## Removing Data
Spark automatically monitors cache usage on each node and drops out old data partitions in a least-recently-used (LRU) fashion. If you would like to manually remove an RDD instead of waiting for it to fall out of the cache, use the RDD.unpersist() method.

## Shared Variables
Normally, when a function passed to a Spark operation (such as map or reduce) is executed on a remote cluster node, it works on separate copies of all the variables used in the function. These variables are copied to each machine, and no updates to the variables on the remote machine are propagated back to the driver program. Supporting general, read-write shared variables across tasks would be inefficient. However, Spark does provide two limited types of shared variables for two common usage patterns: broadcast variables and accumulators.

## Broadcast Variables
Broadcast variables allow the programmer to keep a read-only variable cached on each machine rather than shipping a copy of it with tasks. They can be used, for example, to give every node a copy of a large input dataset in an efficient manner. Spark also attempts to distribute broadcast variables using efficient broadcast algorithms to reduce communication cost.

Spark actions are executed through a set of stages, separated by distributed “shuffle” operations. Spark automatically broadcasts the common data needed by tasks within each stage. The data broadcasted this way is cached in serialized form and deserialized before running each task. This means that explicitly creating broadcast variables is only useful when tasks across multiple stages need the same data or when caching the data in deserialized form is important.

Broadcast variables are created from a variable v by calling SparkContext.broadcast(v). The broadcast variable is a wrapper around v, and its value can be accessed by calling the value method. The code below shows this:

```python
>>> broadcastVar = sc.broadcast([1, 2, 3])
<pyspark.broadcast.Broadcast object at 0x102789f10>

>>> broadcastVar.value
[1, 2, 3]
```

After the broadcast variable is created, it should be used instead of the value v in any functions run on the cluster so that v is not shipped to the nodes more than once. In addition, the object v should not be modified after it is broadcast in order to ensure that all nodes get the same value of the broadcast variable (e.g. if the variable is shipped to a new node later).

## Accumulators
Accumulators are variables that are only “added” to through an associative and commutative operation and can therefore be efficiently supported in parallel. They can be used to implement counters (as in MapReduce) or sums. Spark natively supports accumulators of numeric types, and programmers can add support for new types.

As a user, you can create named or unnamed accumulators. As seen in the image below, a named accumulator (in this instance counter) will display in the web UI for the stage that modifies that accumulator. Spark displays the value for each accumulator modified by a task in the “Tasks” table.

## Accumulators in the Spark UI

Tracking accumulators in the UI can be useful for understanding the progress of running stages (NOTE: this is not yet supported in Python).

An accumulator is created from an initial value v by calling SparkContext.accumulator(v). Tasks running on a cluster can then add to it using the add method or the += operator. However, they cannot read its value. Only the driver program can read the accumulator’s value, using its value method.

The code below shows an accumulator being used to add up the elements of an array:

```python
>>> accum = sc.accumulator(0)
>>> accum
Accumulator<id=0, value=0>

>>> sc.parallelize([1, 2, 3, 4]).foreach(lambda x: accum.add(x))
...
10/09/29 18:41:08 INFO SparkContext: Tasks finished in 0.317106 s

>>> accum.value
10
```

While this code used the built-in support for accumulators of type Int, programmers can also create their own types by subclassing AccumulatorParam. The AccumulatorParam interface has two methods: zero for providing a “zero value” for your data type, and addInPlace for adding two values together. For example, supposing we had a Vector class representing mathematical vectors, we could write:

```python
class VectorAccumulatorParam(AccumulatorParam):
    def zero(self, initialValue):
        return Vector.zeros(initialValue.size)

    def addInPlace(self, v1, v2):
        v1 += v2
        return v1

# Then, create an Accumulator of this type:
vecAccum = sc.accumulator(Vector(...), VectorAccumulatorParam())
```
For accumulator updates performed inside actions only, Spark guarantees that each task’s update to the accumulator will only be applied once, i.e. restarted tasks will not update the value. In transformations, users should be aware of that each task’s update may be applied more than once if tasks or job stages are re-executed.

Accumulators do not change the lazy evaluation model of Spark. If they are being updated within an operation on an RDD, their value is only updated once that RDD is computed as part of an action. Consequently, accumulator updates are not guaranteed to be executed when made within a lazy transformation like map(). The below code fragment demonstrates this property:

```python
accum = sc.accumulator(0)
def g(x):
    accum.add(x)
    return f(x)
data.map(g)
# Here, accum is still 0 because no actions have caused the `map` to be computed.
```

## Deploying to a Cluster
The application submission guide describes how to submit applications to a cluster. In short, once you package your application into a JAR (for Java/Scala) or a set of .py or .zip files (for Python), the bin/spark-submit script lets you submit it to any supported cluster manager.

## Launching Spark jobs from Java / Scala
The org.apache.spark.launcher package provides classes for launching Spark jobs as child processes using a simple Java API.

## Unit Testing
Spark is friendly to unit testing with any popular unit test framework. Simply create a SparkContext in your test with the master URL set to local, run your operations, and then call SparkContext.stop() to tear it down. Make sure you stop the context within a finally block or the test framework’s tearDown method, as Spark does not support two contexts running concurrently in the same program.

## Where to Go from Here
You can see some example Spark programs on the Spark website. In addition, Spark includes several samples in the examples directory (Scala, Java, Python, R). You can run Java and Scala examples by passing the class name to Spark’s bin/run-example script; for instance:
```shell
./bin/run-example SparkPi
For Python examples, use spark-submit instead:

./bin/spark-submit examples/src/main/python/pi.py
For R examples, use spark-submit instead:

./bin/spark-submit examples/src/main/r/dataframe.R
```
For help on optimizing your programs, the configuration and tuning guides provide information on best practices. They are especially important for making sure that your data is stored in memory in an efficient format. For help on deploying, the cluster mode overview describes the components involved in distributed operation and supported cluster managers.

Finally, full API documentation is available in Scala, Java, Python and R.