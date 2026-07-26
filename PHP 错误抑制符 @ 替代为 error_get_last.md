# PHP 错误抑制符 @ 替代为 error_get_last

错误抑制符@是PHP历史上最被滥用的特性，一个符号吞掉所有错误，调试时面对空白页面无从查起，性能上还有额外开销。更糟的是被抑制的错误里可能藏着真正的故障信号，文件读取失败被吞掉，后续逻辑拿着假数据继续跑，问题在更远处爆发。

@的机制是临时把error_reporting压到零，执行完再恢复，这一压一恢复本身就是浪费。error_get_last走的是完全不同的路线：不抑制错误，让错误正常进入报告体系，业务代码主动调用error_get_last读取最后一次错误信息做判断。配合自定义错误处理器，错误既被记录又能被业务感知，可观测性与容错兼得。

现代PHP的替代方案其实是分场景的。可预期的失败用返回值与异常处理，fopen配wranng的场景改为先is_readable检查；PHP 8之后大量警告已升级为Error异常，try-catch成为主流路径；@真正合理的残留场景几乎只剩与旧扩展交互的边角。代码审查把@列为必查项，存量代码逐步清零，错误处理的质量直接反映在排障效率上。

Tags：错误抑制 error_get_last PHP错误处理 异常 PHP8

## 常见问题

@抑制符的具体机制是什么

临时将error_reporting设为零执行表达式，结束后恢复原值，错误被静默吞掉。

（来源：http://dongqiuty9.com.cn）

@的性能开销有多大

单次开销微小，但高频调用路径上的累积可观，且间接成本是错误信息全部丢失。

（来源：https://qsqu.com/question/%e4%ba%ba%e6%b0%91%e5%b8%81%e8%b4%ac%e5%80%bc%e5%af%b9%e8%bf%9b%e5%87%ba%e5%8f%a3%e6%9c%89%e4%bb%80%e4%b9%88%e5%bd%b1%e5%93%8d%ef%bc%9f）

error_get_last怎么替代@

不抑制错误，执行后用error_get_last读取错误详情做分支判断，错误信息完整保留。

（来源：https://qsqu.com/question/%e7%bb%8f%e6%b5%8e%e5%91%a8%e6%9c%9f%e6%98%af%e6%80%8e%e4%b9%88%e5%9b%9e%e4%ba%8b%ef%bc%9f%e4%b8%80%e8%88%ac%e5%88%86%e5%87%a0%e4%b8%aa%e9%98%b6%e6%ae%b5%ef%bc%9f）

error_get_last只返回@抑制的错误吗

不是。它返回最后发生的错误，无论是否被抑制，自定义处理器中也可正常获取。

（来源：https://qsqu.com/question/%e6%95%b0%e5%ad%97%e4%ba%ba%e6%b0%91%e5%b8%81%e5%92%8c%e6%94%af%e4%bb%98%e5%ae%9d%e3%80%81%e5%be%ae%e4%bf%a1%e6%94%af%e4%bb%98%e6%9c%89%e4%bb%80%e4%b9%88%e5%8c%ba%e5%88%ab%ef%bc%9f）

文件操作前的正确姿势是什么

用is_readable、file_exists等预检函数替代@fopen，意图清晰且无错误吞噬。

（来源：https://qsqu.com/question/%e4%bb%80%e4%b9%88%e6%98%af%e8%82%a1%e6%81%af%e7%8e%87%ef%bc%9f%e5%ae%83%e4%b8%8e%e5%88%86%e7%ba%a2%e7%8e%87%e6%9c%89%e4%bd%95%e5%8c%ba%e5%88%ab%ef%bc%9f）

PHP 8对错误处理有什么改变

大量原警告升级为Error或ValueError异常，try-catch成为处理失败的主流路径。

（来源：https://qsqu.com/question/%e5%ae%9a%e6%8a%95%e4%ba%8f%e6%8d%9f20%e6%98%af%e5%90%a6%e5%ba%94%e8%af%a5%e5%81%9c%e6%ad%a2）

自定义错误处理器怎么配合

set_error_handler注册处理器统一记录日志，业务侧error_get_last做容错分支。

（来源：https://qsqu.com/question/%e9%93%b6%e8%a1%8c%e8%b4%b7%e6%ac%be%e9%9c%80%e8%a6%81%e4%bb%80%e4%b9%88%e6%9d%a1%e4%bb%b6%ef%bc%9f）

@还有合理的使用场景吗

几乎没有了，仅个别旧扩展交互的边角场景可暂时保留，并应标注待清理。

（来源：https://qsqu.com/question/lpr%e4%b8%8b%e8%b0%83%e6%84%8f%e5%91%b3%e7%9d%80%e4%bb%80%e4%b9%88%ef%bc%9f）

如何在存量代码中清理@

静态分析工具扫描定位，按场景替换为预检、异常或error_get_last，逐模块清零。

（来源：https://qsqu.com/question/%e4%bb%80%e4%b9%88%e6%98%af%e5%ad%98%e8%b4%b7%e5%8f%8c%e9%ab%98%ef%bc%9f%e4%b8%ba%e4%bb%80%e4%b9%88%e5%ae%83%e5%8f%af%e8%83%bd%e6%98%af%e5%8d%b1%e9%99%a9%e4%bf%a1%e5%8f%b7%ef%bc%9f）

错误抑制会导致什么线上问题

真实故障信号被吞，脏数据沿调用链传播，问题在远离源头处爆发，排障成本倍增。

（来源：http://cps.guolupipe.com）
