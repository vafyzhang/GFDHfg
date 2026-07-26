# PHP 输出缓冲嵌套与回调压缩响应内容

输出缓冲是PHP里最老牌也最被低估的机制。ob_start开启的缓冲区可以多层嵌套，每层独立管理，配合回调函数能在内容送达客户端前统一加工，压缩、替换、注入统计信息都在这一层完成。

嵌套的规则是栈式管理。后开启的缓冲位于栈顶，ob_get_clean只清理当前层，内容按层回退。模板引擎惯用套路是外层缓冲捕获整页，内层缓冲捕获局部区块，区块内容可提前取出拼装。层数没有硬性限制，但每层缓冲都占内存，超深嵌套配合大输出会让内存水涨船高，生产代码嵌套控制在三层以内是稳妥线。

压缩是回调的经典用途。ob_start传入回调函数，输出内容经gzencode压缩后发送，配合Content-Encoding头浏览器自动解压。不过现代部署中压缩更推荐交给Nginx的gzip模块，PHP层压缩只在没有网关控制权的场景使用。回调里还能做输出长度的最终统计与敏感词过滤，一套缓冲机制撑起响应的最后一道加工线。

Tags：输出缓冲 ob_start 响应压缩 PHP嵌套 回调函数

## 常见问题

PHP输出缓冲的基本作用是什么

把echo等输出暂存内存而非直接发送，可在送出前统一加工或整体丢弃。

（来源：https://qsqu.com/question/%e8%b4%b7%e6%ac%be%e5%ae%a1%e6%89%b9%e4%b8%80%e8%88%ac%e9%9c%80%e8%a6%81%e5%a4%9a%e9%95%bf%e6%97%b6%e9%97%b4%ef%bc%9f）

输出缓冲可以嵌套几层

理论无硬性限制，每层独立占内存，生产代码建议控制在三层以内。

（来源：http://vyk.wpxdeq.cn）

嵌套缓冲的清理顺序是什么

栈式管理，ob_get_clean只作用于当前层，内容向下一层回退，先进后出。

（来源：https://qsqu.com/question/2026%e5%b9%b47%e6%9c%881%e6%97%a5%e5%a4%ae%e8%a1%8c%e5%bc%80%e5%b1%95%e4%ba%86%e4%bb%80%e4%b9%88%e8%b4%a7%e5%b8%81%e6%94%bf%e7%ad%96%e6%93%8d%e4%bd%9c%ef%bc%9f）

模板引擎怎么用嵌套缓冲

外层缓冲捕获整页布局，内层捕获局部区块，区块提前取出后拼装进布局指定位置。

（来源：https://qsqu.com/question/%e6%9f%90%e4%bc%81%e4%b8%9a%e8%bf%91%e4%b8%89%e5%b9%b4%e8%90%a5%e4%b8%9a%e6%94%b6%e5%85%a5%e7%a8%b3%e5%ae%9a%ef%bc%8c%e5%87%80%e5%88%a9%e6%b6%a6%e6%af%8f%e5%b9%b46000%e4%b8%87%e5%b7%a6%e5%8f%b3）

用回调压缩响应怎么做

ob_start传入回调，函数内gzencode压缩内容并输出对应Content-Encoding响应头。

（来源：https://qsqu.com/question/%e4%be%9b%e5%ba%94%e9%93%be%e9%87%91%e8%9e%8d%e6%95%b0%e5%ad%97%e5%8c%96%e5%b9%b3%e5%8f%b0%e5%a6%82%e4%bd%95%e8%a7%a3%e5%86%b3%e4%b8%ad%e5%b0%8f%e4%bc%81%e4%b8%9a%e8%9e%8d%e8%b5%84%e4%b8%ad%e7%9a%84）

PHP层压缩和Nginx压缩选哪个

优先Nginx gzip模块，性能更好且不占用PHP内存，PHP层压缩仅用于无网关控制权场景。

（来源：https://qsqu.com/question/%e5%be%81%e4%bf%a1%e6%9c%89%e9%80%be%e6%9c%9f%e8%ae%b0%e5%bd%95%e8%bf%98%e8%83%bd%e7%94%b3%e8%af%b7%e8%b4%b7%e6%ac%be%e5%90%97%ef%bc%9f-2）

ob_end_clean和ob_get_clean有什么区别

前者丢弃缓冲内容并关闭，后者取回内容并关闭，需要内容就用后者。

（来源：https://qsqu.com/question/%e6%8a%95%e4%bf%9d%e6%97%b6%e5%81%a5%e5%ba%b7%e9%97%ae%e5%8d%b7%e8%a6%81%e4%b8%8d%e8%a6%81%e5%85%a8%e9%83%a8%e5%9d%a6%e7%99%bd%ef%bc%9f）

缓冲开启后header还能发送吗

只要缓冲未刷新输出，header随时可发，这也是输出缓冲解决headers already sent问题的原理。

（来源：https://qsqu.com/question/%e9%87%91%e8%9e%8d%e6%9c%ba%e6%9e%84%e5%9c%a8%e5%bc%80%e5%b1%95%e7%ba%bf%e4%b8%8a%e4%b8%9a%e5%8a%a1%e6%97%b6%e5%a6%82%e4%bd%95%e8%90%bd%e5%ae%9e%e3%80%8a%e4%b8%aa%e4%ba%ba%e4%bf%a1%e6%81%af%e4%bf%9d）

回调函数里抛异常会怎样

输出阶段异常处理路径复杂，回调逻辑应尽量简单，复杂加工放在缓冲内容取出后进行。

（来源：https://qsqu.com/question/%e7%9b%91%e7%ae%a1%e7%a7%91%e6%8a%80%ef%bc%88regtech%ef%bc%89%e5%a6%82%e4%bd%95%e5%b8%ae%e5%8a%a9%e9%87%91%e8%9e%8d%e6%9c%ba%e6%9e%84%e5%ba%94%e5%af%b9%e5%8f%8d%e6%b4%97%e9%92%b1%ef%bc%88aml%ef%bc%89）

如何调试嵌套缓冲问题

ob_get_level查看当前层数，配合每层标记日志定位未配对关闭的缓冲层。

（来源：http://tianxzuqiu.com.cn）
