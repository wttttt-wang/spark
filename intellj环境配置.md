# 写在前面：

准备开始学spark，于是准备在IDE配一个spark的开发环境。

# 相关配置：

## 关于系统

* mac os10.12
* intellj IDEA

## 关于我

* scala&函数式编程零基础
* 会hadoop， java， maven

 

# 失败的经验1

* 脑子一热，用sbt替换了maven。但事实是

　　1. 国内的sbt自动下载慢哭（用maven配国内镜像简直快到飞起，感谢阿里爸爸

　　2. sbt的依赖配置总是报各种bug，要根据stackoverflow去补很多依赖（事实证明用maven只要一条依赖

* 而由于sbt需要的很多依赖之间兼容性并不好，每次修改都是一次漫长的等待。最后最崩溃的是还会产生冲突

 

# 成功的经验

* 心累之下还是回归了maven
* 有一个一定要注意的是scala2.12版本及以上跟spark各个版本都不兼容（至少目前是这样）
* 我先装的scala是2.12.2， 为此重新下了个2.10.3，并将2.10.3作为intellj的默认scala，sdk
* 记得配maven的ali镜像，爽到飞（修改maven的conf目录下的setting.xml，修改完记得intellj的maven设置也用这个配置
* 事实证明maven中只需要在pom.xml中添加如下一个依赖，

[![复制代码](http://common.cnblogs.com/images/copycode.gif)](http://www.cnblogs.com/wttttt/articles/javascript:void(0);)

```xml
    <dependencies>
        <!-- https://mvnrepository.com/artifact/org.apache.spark/spark-core_2.10 -->
        <dependency>
            <groupId>org.apache.spark</groupId>
            <artifactId>spark-core_2.10</artifactId>
            <version>2.1.0</version>
        </dependency>
    </dependencies>
```

# 顺便给个scala的spark wordcount

```scala
object Test {
  def main(args: Array[String]) {
    // println("Hello World")
    val conf = new SparkConf()
      .setAppName("Test")
      .setMaster("local")

    val sc = new SparkContext(conf)

    val text = sc.textFile("input/")

    val counts = text.flatMap(line => line.split("\t"))
      .map(word => (word, 1))
      .reduceByKey(_+_)

    counts.foreach(println)
  }
}
```

最后祝学spark愉快！~







