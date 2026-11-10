# Implementation Notes of PySpark

參加 [pycontw2026 sprint](https://hackmd.io/@pycontw/r1At0RiPfx)，主走 type annotation；當時硬是殺出兩個pr，後來 pycontw 前有再多發兩個小pr。
不過再之後發現有點不太懂這邊的風向是什麼，所以就先停住。(等看懂了再來免得出差錯 XD)

## List
### Coverage
- [SPARK-60285][PYTHON][TESTS] Add tests for ReadLimitRegistry
    - [notes](https://github.com/soaring0616/my_opensource_journey/blob/main/notes/pyspark/[SPARK-60285][PYTHON][TESTS][zh]%20Add%20tests%20for%20ReadLimitRegistry.md)
    - https://github.com/apache/spark/pull/59550
    - https://issues.apache.org/jira/browse/SPARK-60285


### Type Hint
- [SPARK-60417][ML][PYTHON] Remove type: ignore[no-untyped-def] in pyspark.ml.torch.distributor
    - notes
    - https://github.com/apache/spark/pull/59667
    - https://issues.apache.org/jira/projects/SPARK/issues/SPARK-60417
- [SPARK-60174][PYTHON][TEST] Remove type: ignore[union-attr] in pyspark.sql.worker.utils
    - [notes](https://github.com/soaring0616/my_opensource_journey/blob/main/notes/pyspark/[Apache%20Spark%20%E6%96%B0%E6%89%8B%E5%AF%A6%E9%A9%97%E6%97%A5%E8%AA%8C][SPARK-60174][zh]%20%E7%A7%BB%E9%99%A4%20pyspark.sql.worker.utils%20%E7%9A%84%20type%3A%20ignore[union-attr].md)
    - https://github.com/apache/spark/pull/59376
    - https://issues.apache.org/jira/projects/SPARK/issues/SPARK-60174
- [SPARK-60145][PYTHON] Remove type: ignore[var-annotated] in pyspark.sql
    - [notes]()
    - https://github.com/apache/spark/pull/59351
    - https://issues.apache.org/jira/browse/SPARK-60145
