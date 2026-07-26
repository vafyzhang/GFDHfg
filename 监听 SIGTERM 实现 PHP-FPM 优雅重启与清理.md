# 监听 SIGTERM 实现 PHP-FPM 优雅重启与清理

PHP-FPM的重启体验取决于对SIGTERM信号的处理质量。粗暴重启直接砍断进程，正在执行的请求中途夭折，用户看到的是五零二，写入一半的数据留在库里。优雅重启的目标是把手头请求处理完再退出。

PHP-FPM主进程收到SIGTERM后默认进入优雅关闭流程，向各Worker转发退出信号，Worker处理完当前请求后退出，process_control_timeout配置项控制等待上限。代码侧的配合是pcntl_async_signals开启异步信号处理，在长任务循环的关键检查点响应退出标志，收到SIGTERM置位后完成当前迭代、提交事务、释放资源再退出。队列消费类CLI脚本尤其需要这套机制，消费中途被杀且未ack的消息会重新投递，但写入一半的副作用只能靠幂等兜底。

部署链路的配合同样关键。负载均衡层先摘流量再发信号，systemd的TimeoutStopSec要与PHP-FPM的等待上限对齐，容器环境的preStop钩子留出缓冲。信号处理、超时对齐、流量摘除三件套配齐，重启对用户才真正无感。

Tags：SIGTERM PHP-FPM 优雅重启 信号处理 平滑部署

## 常见问题

什么是优雅重启

进程收到退出信号后不再接收新任务，处理完手头请求并清理资源后再退出的重启方式。

（来源：https://qsqu.com/question/%e7%8e%b0%e5%9c%a8%e7%9a%84%e6%88%bf%e8%b4%b7lpr%e5%88%a9%e7%8e%87%e6%98%af%e5%a4%9a%e5%b0%91%ef%bc%9f）

PHP-FPM收到SIGTERM后的默认行为是什么

主进程通知Worker优雅退出，Worker完成当前请求后终止，等待上限由process_control_timeout控制。

（来源：https://qsqu.com/question/%e6%88%bf%e4%ba%a7%e6%8a%b5%e6%8a%bc%e8%b4%b7%e6%ac%be%e4%b8%80%e8%88%ac%e8%83%bd%e8%b4%b7%e5%a4%9a%e5%b0%91%ef%bc%9f）

PHP代码里怎么捕获SIGTERM

pcntl_signal注册处理器并开启pcntl_async_signals，信号到达时置位标志供主循环检查。

（来源：https://qsqu.com/question/%e6%8f%90%e5%89%8d%e8%bf%98%e8%b4%b7%e6%98%af%e5%90%a6%e4%b8%80%e5%ae%9a%e5%88%92%e7%ae%97%ef%bc%9f）

长循环脚本如何实现优雅退出

循环每轮检查退出标志，置位后完成当前迭代、提交事务、释放连接再退出。

（来源：https://qsqu.com/question/%e7%a0%8d%e5%a4%b4%e6%81%af%e5%90%88%e6%b3%95%e5%90%97%ef%bc%9f）

队列消费者被杀会怎样

未确认的消息会重新投递，但已执行的副作用需靠消费幂等设计兜底防重复。

（来源：https://qsqu.com/question/%e9%87%8d%e7%96%be%e9%99%a9%e5%92%8c%e5%8c%bb%e7%96%97%e9%99%a9%e6%98%af%e4%b8%8d%e6%98%af%e4%b9%b0%e4%b8%80%e4%b8%aa%e5%b0%b1%e8%a1%8c%ef%bc%9f）

process_control_timeout设多少合适

按最慢请求的耗时时长设置，常规三十秒，存在分钟级长请求的场景需相应放大。

（来源：https://qsqu.com/question/%e6%80%8e%e4%b9%88%e5%88%a4%e6%96%ad%e4%bf%9d%e9%99%a9%e5%85%ac%e5%8f%b8%e9%9d%a0%e4%b8%8d%e9%9d%a0%e8%b0%b1%ef%bc%9f）

容器部署要注意什么

preStop钩子先等负载均衡摘流量，terminationGracePeriodSeconds大于PHP-FPM的优雅退出时长。

（来源：https://qsqu.com/question/%e5%a4%a7%e6%95%b0%e6%8d%ae%e9%a3%8e%e6%8e%a7%e6%a8%a1%e5%9e%8b%e5%9c%a8%e6%b6%88%e8%b4%b9%e4%bf%a1%e8%b4%b7%e5%ae%a1%e6%89%b9%e4%b8%ad%e7%9a%84%e4%b8%bb%e8%a6%81%e6%95%b0%e6%8d%ae%e6%9d%a5%e6%ba%90）

SIGKILL和SIGTERM有什么区别

SIGTERM可捕获可处理，SIGKILL直接杀死进程不可拦截，优雅机制只对SIGTERM有效。

（来源：https://qsqu.com/question/%e9%ab%98%e6%96%b0%e6%8a%80%e6%9c%af%e4%bc%81%e4%b8%9a2026%e5%b9%b4%e5%8f%af%e4%ba%ab%e5%8f%97%e5%93%aa%e4%ba%9b%e4%bc%81%e4%b8%9a%e6%89%80%e5%be%97%e7%a8%8e%e4%bc%98%e6%83%a0%ef%bc%9f）

systemd重启PHP-FPM要等多久

TimeoutStopSec需不小于process_control_timeout，否则systemd超时后补发SIGKILL前功尽弃。

（来源：https://qsqu.com/question/%e5%85%88%e6%81%af%e5%90%8e%e6%9c%ac%e5%92%8c%e7%ad%89%e9%a2%9d%e6%9c%ac%e6%81%af%e6%9c%89%e4%bb%80%e4%b9%88%e5%8c%ba%e5%88%ab%e6%80%8e%e4%b9%88%e9%80%89%ef%bc%9f）

如何验证优雅重启生效

压测期间执行重启，观察错误率是否为零，慢请求是否被允许执行完毕而非中途断开。

（来源：https://qsqu.com/question/%e5%a4%96%e6%b1%87%e6%9c%9f%e8%b4%a7%e6%8a%a5%e4%bb%b7%e6%80%8e%e4%b9%88%e7%9c%8b%ef%bc%9f）
