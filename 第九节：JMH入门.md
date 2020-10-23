# 词汇
    QPS:query per second每秒查询率
    ops:operation per second
    
    主流压力测试工具
        Apache JMeter:
        LoadRunner

# JMH Java准测试工具套件

## 什么是JMH

JMH:Java MicroBenchmark Harness

2013年首发  
-由JIT的开发人员开发  
-归于openJDK  

### 官网

 http://openjdk.java.net/projects/code-tools/jmh/ 

## 创建JMH测试

1. 创建Maven项目，添加依赖

   ```java
   <?xml version="1.0" encoding="UTF-8"?>
   <project xmlns="http://maven.apache.org/POM/4.0.0"
            xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
            xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
       <modelVersion>4.0.0</modelVersion>
   
       <properties>
           <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
           <encoding>UTF-8</encoding>
           <java.version>1.8</java.version>
           <maven.compiler.source>1.8</maven.compiler.source>
           <maven.compiler.target>1.8</maven.compiler.target>
       </properties>
   
       <groupId>mashibing.com</groupId>
       <artifactId>HelloJMH2</artifactId>
       <version>1.0-SNAPSHOT</version>
   
   
       <dependencies>
           <!-- https://mvnrepository.com/artifact/org.openjdk.jmh/jmh-core -->
           <dependency>
               <groupId>org.openjdk.jmh</groupId>
               <artifactId>jmh-core</artifactId>
               <version>1.21</version>
           </dependency>
   
           <!-- https://mvnrepository.com/artifact/org.openjdk.jmh/jmh-generator-annprocess -->
           <dependency>
               <groupId>org.openjdk.jmh</groupId>
               <artifactId>jmh-generator-annprocess</artifactId>
               <version>1.21</version>
               <scope>test</scope>
           </dependency>
       </dependencies>
   
   
   </project>
   ```

2. idea安装JMH插件 JMH plugin v1.0.3

3. 由于用到了注解，打开运行程序注解配置

   > compiler -> Annotation Processors -> Enable Annotation Processing

4. 定义需要测试类PS (ParallelStream)

```java
package com.learn;

import java.util.ArrayList;
import java.util.List;
import java.util.Random;

public class PS {

	static List<Integer> nums = new ArrayList<>();
	static {
		Random r = new Random();
		for (int i = 0; i < 10000; i++) nums.add(1000000 + r.nextInt(1000000));
	}

	static void foreach() {
		nums.forEach(v->isPrime(v));
	}

	static void parallel() {
		nums.parallelStream().forEach(PS::isPrime);
	}
	
	static boolean isPrime(int num) {
		for(int i=2; i<=num/2; i++) {
			if(num % i == 0) return false;
		}
		return true;
	}
}
   ```

5. 写单元测试

   > 这个测试类一定要在test package下面
  
```
package com.learn;

import com.learn.PS;
import org.openjdk.jmh.annotations.*;

public class PSTest {
    @Benchmark
//    @Warmup(iterations = 1,time = 3)
//    @Fork(5)
//    @BenchmarkMode(Mode.Throughput)
//    @Measurement(iterations = 1,time = 3)
    public void testForEach() {
        PS.foreach();
//        PS.parallel();
    }
}
```


6. 运行测试类，如果遇到下面的错误：

   ```java
   ERROR: org.openjdk.jmh.runner.RunnerException: ERROR: Exception while trying to acquire the JMH lock (C:\WINDOWS\/jmh.lock): C:\WINDOWS\jmh.lock (拒绝访问。), exiting. Use -Djmh.ignoreLock=true to forcefully continue.
   	at org.openjdk.jmh.runner.Runner.run(Runner.java:216)
   	at org.openjdk.jmh.Main.main(Main.java:71)
   ```

   这个错误是因为JMH运行需要访问系统的TMP目录，解决办法是：

    Run -->Run...-->Edit Configurations-->Environment Variables -> include system environment viables
   打开RunConfiguration -> Environment Variables -> include system environment viables

7. 阅读测试报告

```
# JMH version: 1.21
# VM version: JDK 13.0.2, Java HotSpot(TM) 64-Bit Server VM, 13.0.2+8
# VM invoker: C:\Program Files\Java\jdk-13.0.2\bin\java.exe
# VM options: -Dfile.encoding=UTF-8
# Warmup: 1 iterations, 3 s each
# Measurement: 1 iterations, 3 s each
# Timeout: 10 min per iteration
# Threads: 1 thread, will synchronize iterations
# Benchmark mode: Throughput, ops/time
# Benchmark: com.learn.PSTest.testForEach

# Run progress: 0.00% complete, ETA 00:00:30
# Fork: 1 of 5
# Warmup Iteration   1: 0.906 ops/s
Iteration   1: 0.920 ops/s

# Run progress: 20.00% complete, ETA 00:00:28
# Fork: 2 of 5
# Warmup Iteration   1: 0.893 ops/s
Iteration   1: 0.907 ops/s

# Run progress: 40.00% complete, ETA 00:00:21
# Fork: 3 of 5
# Warmup Iteration   1: 0.916 ops/s
Iteration   1: 0.926 ops/s

# Run progress: 60.00% complete, ETA 00:00:13
# Fork: 4 of 5
# Warmup Iteration   1: 0.894 ops/s
Iteration   1: 0.907 ops/s

# Run progress: 80.00% complete, ETA 00:00:06
# Fork: 5 of 5
# Warmup Iteration   1: 0.913 ops/s
Iteration   1: 0.931 ops/s


Result "com.learn.PSTest.testForEach":
  0.918 ±(99.9%) 0.042 ops/s [Average]
  (min, avg, max) = (0.907, 0.918, 0.931), stdev = 0.011
  CI (99.9%): [0.877, 0.960] (assumes normal distribution)


# Run complete. Total time: 00:00:34

REMEMBER: The numbers below are just data. To gain reusable insights, you need to follow up on
why the numbers are the way they are. Use profilers (see -prof, -lprof), design factorial
experiments, perform baseline and negative tests that provide experimental control, make sure
the benchmarking environment is safe on JVM/OS/HW level, ask for reviews from the domain experts.
Do not assume the numbers tell you what you want them to tell.

Benchmark            Mode  Cnt  Score   Error  Units
PSTest.testForEach  thrpt    5  0.918 ± 0.042  ops/s

Process finished with exit code 0
```



## JMH中的基本概念

    1. Warmup
       预热，由于JVM中对于特定代码会存在优化（本地化），预热对于测试结果很重要
        iterations = 1,time = 3
        迭代一次，预热一次，预热3秒
    
    2. Mesurement
       总共执行多少次测试
        iterations = 1,time = 3    
        执行1次，间隔3秒
    
    3. Timeout
        每次迭代花费多长时间
       
    4. Threads
       线程数，由fork指定
        启动几个线程执行
    
    5. Benchmark mode
       基准测试的模式
        使用最多的是Throughput 吞吐量
    
    6. Benchmark
       测试哪一段代码

## Next

官方样例：
http://hg.openjdk.java.net/code-tools/jmh/file/tip/jmh-samples/src/main/java/org/openjdk/jmh/samples/

