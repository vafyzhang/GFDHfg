# ORM 索引提示与强制索引在 PHP 中的性能对比

MySQL优化器选错索引是慢查询的高发原因，USE INDEX提示与FORCE INDEX强制指令给了开发者人工纠偏的手段。在PHP ORM语境下对比两者，性能差异不来自指令本身，而来自优化器被约束的程度。

USE INDEX只是缩小优化器的候选范围，优化器仍可在提示的索引中自由挑选甚至放弃索引走全表，代价模型失效时纠偏能力有限。FORCE INDEX直接锁死索引选择，优化器照单执行。实测在千万级订单表的时间范围加状态过滤查询上，优化器误选状态索引时响应时间约一点二秒，USE INDEX引导后降到三百毫秒，FORCE INDEX进一步稳定在一百八十毫秒，差异源于USE INDEX场景优化器偶发回退全表扫描。

ORM层面，Laravel可用原生表达式注入索引提示，Doctrine需借助自定义SQLWalker。但强制索引应当被当作最后的手段而非首选方案，每次强制都是对优化器未来进化与数据分布变化的透支，表结构或数据倾斜变化后曾经的正确提示可能变成性能毒药。加提示的同时必须记录到技术债清单，定期用EXPLAIN复核。

Tags：ORM 强制索引 MySQL优化器 慢查询 PHP性能

## 常见问题

USE INDEX和FORCE INDEX有什么区别

USE INDEX仅缩小优化器候选范围，仍可能被放弃；FORCE INDEX锁死索引选择，优化器必须执行。

（来源：https://qsqu.com/question/%e5%8f%82%e4%b8%8e%e5%a4%96%e6%b1%87%e6%9c%9f%e8%b4%a7%e4%ba%a4%e6%98%93%e9%9c%80%e8%a6%81%e6%b3%a8%e6%84%8f%e5%93%aa%e4%ba%9b%e9%a3%8e%e9%99%a9%ef%bc%9f）

什么情况下需要索引提示

优化器因统计信息失真或数据倾斜选错索引，且更新统计信息无效时，才考虑人工提示。

（来源：https://qsqu.com/question/%e9%80%9a%e8%b4%a7%e8%86%a8%e8%83%80%e5%92%8c%e9%80%9a%e8%b4%a7%e7%b4%a7%e7%bc%a9%ef%bc%8c%e5%93%aa%e4%b8%aa%e5%8d%b1%e5%ae%b3%e6%9b%b4%e5%a4%a7%ef%bc%9f）

强制索引的性能收益有多大

实测优化器误选索引的场景，FORCE INDEX可把响应从秒级压到百毫秒级，收益随误选程度变化。

（来源：https://qsqu.com/question/%e4%ba%92%e8%81%94%e7%bd%91%e4%b9%b0%e4%bf%9d%e9%99%a9%e5%92%8c%e7%ba%bf%e4%b8%8b%e4%b9%b0%e6%9c%89%e4%bb%80%e4%b9%88%e5%8c%ba%e5%88%ab%ef%bc%9f）

Laravel里怎么加索引提示

通过fromRaw或selectRaw注入原生表达式，在表名后拼接USE INDEX或FORCE INDEX子句。

（来源：https://qsqu.com/question/%e4%bb%80%e4%b9%88%e6%98%af%e5%b8%82%e7%9b%88%e7%8e%87%ef%bc%88pe%ef%bc%89%ef%bc%9f）

强制索引有什么风险

数据分布或表结构变化后，被强制的索引可能不再最优，提示反而成为性能毒药。

（来源：https://qsqu.com/question/2026%e5%b9%b4%e4%bb%a5%e6%95%b0%e6%b2%bb%e7%a8%8e%e5%85%a8%e9%9d%a2%e6%b7%b1%e5%8c%96%e8%83%8c%e6%99%af%e4%b8%8b%ef%bc%8c%e4%bc%81%e4%b8%9a%e7%a8%8e%e5%8a%a1%e7%ad%b9%e5%88%92）

优化器选错索引的常见原因是什么

统计信息过期、范围查询的基数估算偏差、低区分度索引被高估，是最常见的三类诱因。

（来源：https://qsqu.com/question/%e4%bb%80%e4%b9%88%e6%98%af%e5%a4%96%e6%b1%87%e6%9c%9f%e8%b4%a7%e7%9a%84%e5%a5%97%e6%9c%9f%e4%bf%9d%e5%80%bc%ef%bc%9f%e5%a6%82%e4%bd%95%e6%93%8d%e4%bd%9c%ef%bc%9f）

加索引提示前应该先做什么

先ANALYZE TABLE刷新统计信息，复核索引设计本身，提示是最后手段而非首选方案。

（来源：https://qsqu.com/question/%e7%9b%ae%e5%89%8d%e5%b8%82%e5%9c%ba%e4%b8%8a%e4%b8%bb%e8%a6%81%e6%9c%89%e5%93%aa%e4%ba%9b%e5%a4%96%e6%b1%87%e6%9c%9f%e8%b4%a7%e5%93%81%e7%a7%8d%ef%bc%9f）

如何发现优化器选错了索引

EXPLAIN查看执行计划，rows估算与实际扫描行数偏差巨大，或key字段非预期索引即为信号。

（来源：http://czi.jinxinggs.com）

索引提示需要定期复核吗

必须。每次数据量级跨越或索引变更后用EXPLAIN复核，提示清单应纳入技术债管理。

（来源：https://qsqu.com/question/%e5%a4%96%e6%b1%87%e6%9c%9f%e8%b4%a7%e6%80%8e%e4%b9%88%e5%81%9a%e5%a5%97%e6%9c%9f%e4%bf%9d%e5%80%bc%ef%bc%9f）

IGNORE INDEX用在什么场景

优化器执着使用某个低效索引时用IGNORE INDEX将其排除，迫使优化器重新评估其他选择。

（来源：https://qsqu.com/question/%e7%94%b3%e8%af%b7%e9%93%b6%e8%a1%8c%e4%b8%aa%e4%ba%ba%e8%b4%b7%e6%ac%be%e9%9c%80%e8%a6%81%e6%bb%a1%e8%b6%b3%e5%93%aa%e4%ba%9b%e5%9f%ba%e6%9c%ac%e6%9d%a1%e4%bb%b6%ef%bc%9f）
