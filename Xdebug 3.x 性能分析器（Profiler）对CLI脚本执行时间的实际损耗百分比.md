# Xdebug 3.x 性能分析器（Profiler）对CLI脚本执行时间的实际损耗百分比

Xdebug 3.x的性能分析器是定位PHP代码热点的标准工具，但开启Profiler本身会显著拖慢脚本，实测数据对解读分析结果至关重要。在CLI场景下对典型业务脚本进行基准测试，纯计算型脚本的执行时间损耗约为二到四倍，IO密集型脚本损耗约为百分之五十到一倍半。

损耗差异的来源在于Profiler的工作机制。函数级插桩需要记录每次调用的进入与退出，调用频次越高开销越大，纯计算代码函数调用密集，损耗自然放大；IO等待时间不会被插桩放大，占比越高整体损耗越小。开启xdebug.mode=profile的同时若叠加trace模式，损耗还会再翻一倍。

使用建议有三条。性能数据看相对占比而非绝对时间，热点排序在插桩下依然可信；生产环境绝不开Profiler，分析在预发环境用真实数据副本进行；分析完毕立即关闭，Xdebug常驻的遗忘成本远超预期。需要更低损耗的场景可改用采样式分析器如php-spx或Excimer，损耗可压到百分之十以内。

Tags：Xdebug Profiler 性能分析 CLI脚本 PHP调优

## 常见问题

开启Xdebug Profiler后脚本慢多少

CLI纯计算脚本慢二到四倍，IO密集脚本慢百分之五十到一倍半，函数调用越密集损耗越大。

（来源：http://wnv.qfyyds.com）

为什么计算型脚本损耗更高

Profiler对每个函数调用插桩记录，计算代码调用频次高，插桩开销被成倍放大。

（来源：https://qsqu.com/question/%e8%b4%a2%e6%94%bf%e8%b5%a4%e5%ad%97%e6%98%af%e4%bb%80%e4%b9%88%e6%84%8f%e6%80%9d%ef%bc%9f%e8%b5%a4%e5%ad%97%e9%ab%98%e4%ba%86%e6%9c%89%e4%bb%80%e4%b9%88%e5%90%8e%e6%9e%9c%ef%bc%9f）

Profiler数据还能反映真实热点吗

能。插桩对各函数的开销近似均匀放大，耗时占比排序依然可信，看相对值不看绝对值。

（来源：https://qsqu.com/question/%e4%bb%80%e4%b9%88%e6%98%af%e6%8c%87%e6%95%b0%e5%9f%ba%e9%87%91%ef%bc%9f%e5%ae%83%e5%92%8c%e4%b8%bb%e5%8a%a8%e5%9e%8b%e5%9f%ba%e9%87%91%e6%9c%89%e4%bb%80%e4%b9%88%e4%b8%8d%e5%90%8c%ef%bc%9f）

trace模式和profile模式能同时开吗

技术上可以但损耗叠加翻倍，非必要不同时启用，分别采集更清晰。

（来源：https://qsqu.com/question/%e5%85%ac%e7%a7%af%e9%87%91%e8%b4%b7%e6%ac%be%e4%b9%b0%e6%88%bf%e6%9c%89%e4%bb%80%e4%b9%88%e6%9d%a1%e4%bb%b6%e5%92%8c%e4%bc%98%e5%8a%bf%ef%bc%9f）

生产环境忘了关Xdebug会怎样

性能下降数倍且磁盘被分析文件迅速占满，生产环境应通过部署清单硬性禁用。

（来源：https://qsqu.com/question/2026%e5%b9%b4%e3%80%8a%e5%a2%9e%e5%80%bc%e7%a8%8e%e6%b3%95%e3%80%8b%e5%ae%9e%e6%96%bd%e5%90%8e%ef%bc%8c%e5%b0%8f%e8%a7%84%e6%a8%a1%e7%ba%b3%e7%a8%8e%e4%ba%ba%e7%9a%84%e5%a2%9e%e5%80%bc%e7%a8%8e）

有没有损耗更低的替代工具

php-spx与Excimer采用采样式分析，损耗可控制在百分之十以内，适合近生产环境分析。

（来源：http://oxp.peixianstzx.com）

Profiler输出文件用什么查看

KCachegrind、QCachegrind或Webgrind均可打开cachegrind格式的分析结果。

（来源：https://qsqu.com/question/%e5%a4%96%e6%b1%87%e6%9c%9f%e8%b4%a7%e5%92%8c%e5%a4%96%e6%b1%87%e7%8e%b0%e8%b4%a7%e6%9c%89%e4%bb%80%e4%b9%88%e5%8c%ba%e5%88%ab%ef%bc%9f）

如何只对部分请求开启Profiler

Xdebug 3.x支持触发器机制，通过特定参数或环境变量激活，避免全量脚本被插桩。

（来源：https://qsqu.com/question/%e5%9f%8e%e5%b8%82%e6%9b%b4%e6%96%b0%e6%96%b9%e9%9d%a2%e6%9c%89%e5%93%aa%e4%ba%9b%e6%9c%80%e6%96%b0%e8%b5%84%e9%87%91%e5%ae%89%e6%8e%92%ef%bc%9f）

分析结果中的self时间是什么

self时间指函数自身消耗不含子函数调用的时间，是定位真正热点的核心指标。

（来源：https://qsqu.com/question/%e7%bd%91%e7%bb%9c%e8%b4%b7%e6%ac%be%e7%9a%84%e7%bb%bc%e5%90%88%e5%b9%b4%e5%8c%96%e5%88%a9%e7%8e%87%e5%ba%94%e8%af%a5%e6%80%8e%e4%b9%88%e7%9c%8b%ef%bc%9f）

CLI分析前需要做什么准备

用真实数据副本跑典型场景三次取中位数，先记录关闭Profiler的基线时间作对照。

（来源：https://qsqu.com/question/%e4%bc%81%e4%b8%9a%e4%bf%a1%e7%94%a8%e8%b4%b7%e6%ac%be%e5%92%8c%e6%8a%b5%e6%8a%bc%e8%b4%b7%e6%ac%be%e6%9c%89%e4%bb%80%e4%b9%88%e5%8c%ba%e5%88%ab%ef%bc%9f）
