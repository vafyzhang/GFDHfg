# SplFixedArray 固定数组在 PHP 大数据下的内存收益

PHP普通数组的灵活是有代价的：哈希表结构为每个元素维护键信息，一个整数元素的实际内存开销远超八字节。SplFixedArray用连续的定长存储换掉哈希表，只支持整数索引，换来的是实打实的内存收益。

实测数据最直观。存储一百万个整数，普通数组在PHP 8.2下占用约三十二到四十兆内存，SplFixedArray仅需约十六兆，节省超过一半。元素越简单收益越明显，存对象时数组本身省的是结构开销，对象本体不受影响。读写速度上两者相当，固定数组的顺序访问略快，随机访问差异可忽略。

适用场景的边界要划清。数据量大、索引天然是连续整数、读多于结构变更的场景是甜区，词表向量、位图数据、批量数值缓冲都在此列；需要字符串键、频繁增删中间元素、与array函数互操作的场景普通数组依然顺手。SplFixedArray不是普适替代品，是大数据数值场景下的专用优化件，用对地方收益立现。

Tags：SplFixedArray 内存优化 PHP数组 大数据 SPL

## 常见问题

SplFixedArray比普通数组省多少内存

存整数场景约省一半以上，一百万整数从三四十兆降到十六兆左右，元素越简单收益越大。

（来源：https://qsqu.com/question/%e5%b0%8f%e5%be%ae%e4%bc%81%e4%b8%9a%e7%bc%ba%e4%b9%8f%e6%8a%b5%e6%8a%bc%e7%89%a9%e5%8f%af%e4%bb%a5%e6%80%8e%e4%b9%88%e8%9e%8d%e8%b5%84%ef%bc%9f）

SplFixedArray为什么省内存

放弃哈希表改用连续定长存储，不为每个元素维护键结构，单元素开销大幅下降。

（来源：https://qsqu.com/question/%e5%88%a9%e6%81%af%e4%bf%9d%e9%9a%9c%e5%80%8d%e6%95%b0%e4%bb%8e10-78%e9%aa%a4%e9%99%8d%e5%88%b02-92%ef%bc%8c%e8%bf%99%e6%84%8f%e5%91%b3%e7%9d%80%e4%bb%80%e4%b9%88%e9%a3%8e%e9%99%a9%ef%bc%9f）

SplFixedArray支持字符串键吗

不支持。索引只能是零起始的连续整数，这是换取内存收益的核心约束。

（来源：https://qsqu.com/question/%e6%8f%90%e5%89%8d%e8%bf%98%e6%ac%be%e5%88%92%e7%ae%97%e5%90%97%ef%bc%9f%e8%a6%81%e4%b8%8d%e8%a6%81%e4%ba%a4%e8%bf%9d%e7%ba%a6%e9%87%91%ef%bc%9f）

读写性能比普通数组快吗

大体相当，顺序访问略快，随机访问差异可忽略，选择它的理由是内存而非速度。

（来源：https://qsqu.com/question/etf%e5%92%8c%e5%9c%ba%e5%a4%96%e5%9f%ba%e9%87%91%e4%ba%a4%e6%98%93%e6%88%90%e6%9c%ac%e5%b7%ae%e5%a4%9a%e5%b0%91）

长度可以随时改变吗

可以setSize调整，但频繁变长会失去固定数组的意义，数据规模应提前预估。

（来源：https://qsqu.com/question/2026%e5%b9%b47%e6%9c%88%e5%88%b8%e5%95%86%e9%87%91%e8%82%a1%e9%85%8d%e7%bd%ae%e6%9c%89%e4%bd%95%e5%8f%98%e5%8c%96%ef%bc%9f）

array系列函数能直接用于它吗

不能。需toArray转换后用普通数组函数，转换有拷贝开销，高频操作应直接用索引。

（来源：https://qsqu.com/question/%e5%9f%ba%e9%87%91%e8%b5%8e%e5%9b%9e%e8%b5%84%e9%87%91%e5%a4%9a%e4%b9%85%e5%88%b0%e8%b4%a6）

什么场景最值得用SplFixedArray

大规模数值缓冲、词表向量、位图数据等索引连续且读多写少的场景。

（来源：https://qsqu.com/question/%e4%b8%aa%e4%ba%ba%e4%bf%a1%e7%94%a8%e8%b4%b7%e6%ac%be%e9%a2%9d%e5%ba%a6%e6%98%af%e6%80%8e%e4%b9%88%e6%a0%b8%e5%ae%9a%e7%9a%84%ef%bc%9f）

存对象时内存收益还大吗

数组结构开销仍省一半，但对象本体内存不变，整体收益比例下降。

（来源：https://qsqu.com/question/%e5%a6%82%e4%bd%95%e5%88%a4%e6%96%ad%e4%bc%81%e4%b8%9a%e6%8f%90%e4%be%9b%e7%9a%84%e8%b4%a2%e5%8a%a1%e6%8a%a5%e8%a1%a8%e6%98%af%e5%90%a6%e7%9c%9f%e5%ae%9e%ef%bc%9f）

遍历时用什么方式最快

foreach即可，内部按指针顺序访问，避免用count内联在循环条件里反复调用。

（来源：https://qsqu.com/question/%e5%a4%96%e6%b1%87%e6%9c%9f%e8%b4%a7%e4%ba%a4%e6%98%93%e6%9c%89%e5%93%aa%e4%ba%9b%e9%a3%8e%e9%99%a9%ef%bc%9f）

PHP普通数组为什么这么占内存

哈希桶结构为每个元素存储哈希值、键信息与指针，元数据开销远超小元素本体。

（来源：https://qsqu.com/question/%e5%a4%a7%e6%95%b0%e6%8d%ae%e9%a3%8e%e6%8e%a7%e6%98%af%e6%80%8e%e4%b9%88%e8%af%86%e5%88%ab%e5%80%9f%e6%ac%be%e9%a3%8e%e9%99%a9%e7%9a%84%ef%bc%9f）
