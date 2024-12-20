# Kotlin listOf performance benchmark

## How to run

```
mvn clean verify && java -jar target/benchmarks.jar
```

## Results

```
# JMH version: 1.37
# VM version: JDK 17.0.10, OpenJDK 64-Bit Server VM, 17.0.10+7-LTS
# VM invoker: /Users/Nikita.Bobko/.sdkman/candidates/java/17.0.10-amzn/bin/java
# VM options: <none>
# Blackhole mode: compiler (auto-detected, use -Djmh.blackhole.autoDetect=false to disable)
# Warmup: 4 iterations, 2 s each
# Measurement: 5 iterations, 2 s each
# Timeout: 10 min per iteration
# Threads: 1 thread, will synchronize iterations
# Benchmark mode: Throughput, ops/time

Benchmark                                           Mode  Cnt       Score       Error   Units
MyBenchmark._005_listOf                            thrpt    5  336185,131 ± 25448,803  ops/ms
MyBenchmark._005_listOf_manual                     thrpt    5  135880,149 ±  4350,914  ops/ms
MyBenchmark._005_mapOf                             thrpt    5   25126,468 ±   216,641  ops/ms
MyBenchmark._005_mapOf_manual                      thrpt    5   51427,209 ±   559,959  ops/ms
MyBenchmark._005_mapOf_separateArrays_avoidBoxing  thrpt    5   38359,693 ±   507,297  ops/ms
MyBenchmark._005_setOf                             thrpt    5   40616,448 ±   697,894  ops/ms
MyBenchmark._005_setOf_manual                      thrpt    5   49257,618 ±   778,119  ops/ms
MyBenchmark._005_stdlib_listOf_java                thrpt    5  357239,466 ±  9888,777  ops/ms
MyBenchmark._005_stdlib_listOf_kotlin              thrpt    5  340452,967 ±  5513,226  ops/ms
MyBenchmark._011_listOf                            thrpt    5  282807,020 ± 24923,083  ops/ms
MyBenchmark._011_listOf_java                       thrpt    5   92784,219 ±  1085,780  ops/ms
MyBenchmark._011_listOf_kotlin                     thrpt    5  282471,829 ±  7058,089  ops/ms
```

## Interpretation

https://github.com/Kotlin/KEEP/blob/master/proposals/collection-literals.md
