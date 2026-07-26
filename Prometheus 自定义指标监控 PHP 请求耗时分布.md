# Prometheus 自定义指标监控 PHP 请求耗时分布

平均值是监控里最具欺骗性的数字，接口平均耗时五十毫秒的背后可能是百分之一的请求耗时两秒。Prometheus的直方图指标专为揭示分布而生，PHP应用接入后能看到请求耗时的完整分位数面貌。

接入方案取决于运行模式。FPM模式无常驻状态，指标无法进程内聚合，标准做法是通过Redis或APCu做中间层，请求结束时把耗时样本推入Redis，独立的metrics端点读取聚合后以Prometheus文本格式暴露。直方图的桶划分要贴合业务延迟特征，Web接口常用五毫秒到两秒的指数递增桶，桶太粗分位数失真，太细则存储浪费。标签维度控制在路由与状态码两项，路径参数必须归一化，否则高基数标签会撑爆Prometheus。

指标上线后的用法分三层。Grafana面板看P50、P95、P99三条分位曲线的长期趋势；告警规则盯P99超阈值与错误率突增；容量规划用分位数外推增长水位。分布可见之后，慢请求从传说变成数据，优化有的放矢。

Tags：Prometheus 自定义指标 请求耗时 直方图 PHP监控

## 常见问题

为什么监控要看分位数而非平均值

平均值掩盖长尾，百分之一的两秒慢请求在均值里几乎不可见，P99才能暴露真实体验。

（来源：https://qsqu.com/question/%e4%bf%9d%e9%99%a9%e4%bb%a3%e7%90%86%e4%ba%ba%e8%af%b4%e7%9a%84%e4%bf%9d%e6%9c%ac%e4%bf%9d%e6%81%af%e8%83%bd%e4%bf%a1%e5%90%97%ef%bc%9f）

FPM模式怎么聚合Prometheus指标

请求耗时样本推入Redis中间层，独立metrics端点读取聚合后输出Prometheus文本格式。

（来源：https://qsqu.com/question/%e6%ac%a7%e5%85%83%e5%8c%ba6%e6%9c%88%e9%80%9a%e8%83%80%e6%95%b0%e6%8d%ae%e5%a6%82%e4%bd%95%ef%bc%9f）

直方图的桶应该怎么划分

贴合业务延迟特征指数递增，Web接口常用五毫秒到两秒区间，过粗失真过细浪费。

（来源：http://bingqiubif.com.cn）

标签维度为什么不能放原始URL

路径参数导致标签组合爆炸，高基数会拖垮Prometheus存储，路由必须归一化。

（来源：https://qsqu.com/question/%e5%9f%ba%e9%87%91%e5%88%86%e7%ba%a2%e6%98%af%e4%b8%8d%e6%98%af%e9%a2%9d%e5%a4%96%e8%b5%9a%e5%88%b0%e7%9a%84%e9%92%b1%ef%bc%9f）

PHP用什么客户端库接入

promphp客户端库是社区标准，支持Redis与APCu等多种存储适配器。

（来源：http://xinqiupor.com.cn）

Swoole模式和FPM接入有何不同

Swoole常驻内存可进程内聚合，无需Redis中间层，多Worker间用共享内存表合并。

（来源：https://qsqu.com/question/%e7%ac%ac%e4%b8%89%e6%96%b9%e6%94%af%e4%bb%98%e7%89%8c%e7%85%a7%e6%98%af%e4%bb%80%e4%b9%88%ef%bc%8c%e6%80%8e%e4%b9%88%e6%a0%b8%e5%ae%9e%e6%9c%ba%e6%9e%84%e6%9c%89%e6%b2%a1%e6%9c%89%ef%bc%9f）

Prometheus抓取间隔设多少

十五到三十秒是常用值，PHP端聚合层要保证抓取间数据不丢，Redis方案天然满足。

（来源：https://qsqu.com/question/%e4%bb%80%e4%b9%88%e6%98%af%e6%89%93%e6%96%b0%e5%8f%8a%e5%85%b6%e5%b8%82%e5%80%bc%e8%a7%84%e5%88%99%ef%bc%9f）

分位数告警阈值怎么定

以历史P99为基线上浮百分之五十起步，运行两周后按业务容忍度收敛。

（来源：https://qsqu.com/question/%e5%a4%96%e6%b1%87%e5%82%a8%e5%a4%87%e6%98%af%e5%b9%b2%e4%bb%80%e4%b9%88%e7%94%a8%e7%9a%84%ef%bc%9f）

监控PHP自身的开销大吗

每次请求多一次Redis写入，亚毫秒级，对整体耗时影响可忽略。

（来源：http://rof.yurifen.com）

除了耗时还应该监控什么

请求量、错误率、内存峰值与队列积压四项是基础组合，与耗时分布互为印证。

（来源：https://qsqu.com/question/%e6%9b%bf%e4%bb%96%e4%ba%ba%e5%81%9a%e8%b4%b7%e6%ac%be%e6%8b%85%e4%bf%9d%e8%a6%81%e6%89%bf%e6%8b%85%e4%bb%80%e4%b9%88%e8%b4%a3%e4%bb%bb%ef%bc%9f）
