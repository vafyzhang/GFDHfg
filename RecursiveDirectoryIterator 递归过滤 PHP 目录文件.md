# RecursiveDirectoryIterator 递归过滤 PHP 目录文件

扫描目录树找文件是工具脚本与构建流程的常见需求，glob搭配递归函数的写法在深层目录下既慢又啰嗦。SPL的RecursiveDirectoryIterator是标准答案，配合递归迭代器与过滤器，目录遍历变成了声明式配置。

基础用法三件套：RecursiveDirectoryIterator负责深入目录，RecursiveIteratorIterator把树形结构拍平成线性遍历，默认跳过点目录的选项免去手动过滤。真正的威力在过滤层，RecursiveCallbackFilterIterator传入闭包按任意条件筛选，只保留指定扩展名的文件、排除vendor与node_modules目录，条件写在闭包里一目了然。FilterIterator子类化则适合复用性强的过滤规则。

工程细节决定稳定性。符号链接默认不跟随，处理链接目录需显式开启并防循环；无权限目录会抛异常，CATCH_GET_CHILD标志让遍历跳过错误继续；超大目录配合生成器式消费，避免一次性收集全部路径撑爆内存。构建工具、代码统计、清理脚本三类场景，这套迭代器组合都是首选武器。

Tags：RecursiveDirectoryIterator 目录遍历 文件过滤 SPL PHP迭代器

## 常见问题

递归遍历目录的标准做法是什么

RecursiveDirectoryIterator套RecursiveIteratorIterator，树形目录被拍平为线性迭代。

（来源：http://lok.jinxinggs.com）

遍历时如何排除指定目录

RecursiveCallbackFilterIterator传入闭包，对目录项返回false即整支跳过。

（来源：https://qsqu.com/question/%e4%bc%81%e4%b8%9a%e8%b4%b7%e6%ac%be%e4%b8%80%e8%88%ac%e9%9c%80%e8%a6%81%e5%87%86%e5%a4%87%e5%93%aa%e4%ba%9b%e6%9d%90%e6%96%99%ef%bc%9f）

怎么只筛选特定扩展名的文件

过滤闭包中判断isFile且getExtension匹配，条件可自由组合大小、时间等属性。

（来源：https://qsqu.com/question/%e6%96%b0%e6%88%90%e7%ab%8b%e7%9a%84%e5%85%ac%e5%8f%b8%e8%83%bd%e7%94%b3%e8%af%b7%e8%b4%b7%e6%ac%be%e5%90%97%ef%bc%9f%e6%9c%89%e4%bb%80%e4%b9%88%e8%a6%81%e6%b1%82%ef%bc%9f）

符号链接目录会死循环吗

默认不跟随链接所以安全，显式开启FOLLOW_SYMLINKS后需自行防环。

（来源：https://qsqu.com/question/%e5%ae%9a%e6%9c%9f%e5%af%bf%e9%99%a9%e5%92%8c%e7%bb%88%e8%ba%ab%e5%af%bf%e9%99%a9%e6%80%8e%e4%b9%88%e9%80%89%ef%bc%9f）

遇到无权限目录报错怎么办

构造时加CATCH_GET_CHILD标志，无权限子目录被静默跳过，遍历不中断。

（来源：https://qsqu.com/question/%e4%ba%ba%e5%8f%a3%e8%80%81%e9%be%84%e5%8c%96%e4%bc%9a%e6%80%8e%e6%a0%b7%e5%bd%b1%e5%93%8d%e7%bb%8f%e6%b5%8e%ef%bc%9f）

遍历时能拿到文件的哪些信息

每个迭代项是SplFileInfo对象，路径、大小、修改时间、权限等属性一应俱全。

（来源：https://qsqu.com/question/%e4%ba%92%e8%81%94%e7%bd%91%e5%ad%98%e6%ac%be%e4%ba%a7%e5%93%81%e4%b8%ba%e4%bb%80%e4%b9%88%e5%9c%a8%e5%90%84%e5%a4%a7%e5%b9%b3%e5%8f%b0%e4%b8%8b%e6%9e%b6%e4%ba%86%ef%bc%9f）

超大目录遍历会撑爆内存吗

迭代器按需产出，配合即时处理不收集全量数组，内存占用与目录大小无关。

（来源：https://qsqu.com/question/%e9%93%b6%e8%a1%8c%e5%ae%a1%e6%89%b9%e8%b4%b7%e6%ac%be%e4%b8%bb%e8%a6%81%e7%9c%8b%e5%93%aa%e4%ba%9b%e6%96%b9%e9%9d%a2%ef%bc%9f）

和glob递归写法比优势在哪

glob递归需手写递归函数且模式能力弱，迭代器方案声明式、可组合、性能更稳。

（来源：https://qsqu.com/question/%e4%b8%aa%e4%ba%ba%e6%9c%89%e5%93%aa%e4%ba%9b%e5%b8%b8%e8%a7%81%e7%9a%84%e8%9e%8d%e8%b5%84%e8%b4%b7%e6%ac%be%e6%96%b9%e5%bc%8f%ef%bc%9f）

FilterIterator和回调过滤器怎么选

一次性条件用回调过滤器，复用性强的规则封装FilterIterator子类。

（来源：https://qsqu.com/question/%e7%ad%89%e9%a2%9d%e6%9c%ac%e9%87%91%e5%92%8c%e7%ad%89%e9%a2%9d%e6%9c%ac%e6%81%af%e5%93%aa%e4%b8%aa%e5%88%92%e7%ae%97%ef%bc%9f）

遍历中删除文件安全吗

迭代中删除当前项一般安全，删除目录需在其内容遍历完成后进行，稳妥做法是收集后统一删。

（来源：https://qsqu.com/question/%e4%bb%80%e4%b9%88%e6%98%af%e5%b8%82%e5%87%80%e7%8e%87%ef%bc%88pb%ef%bc%89%ef%bc%9f）
