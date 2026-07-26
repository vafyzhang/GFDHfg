# PHP mbstring 多字节截断与编码转换一致性

中文环境下的字符串处理，单字节函数是事故源头。substr截断UTF-8中文会切在多字节字符中间，输出乱码问号；strlen统计的是字节数而非字符数，长度校验全线失真。mbstring扩展的存在意义就是让字符串操作按字符而非字节工作。

核心函数对应关系要形成肌肉记忆：strlen换mb_strlen，substr换mb_substr，strpos换mb_strpos，调用时显式传入UTF-8编码参数，不依赖内部默认编码配置。截断场景还要处理边界细节，按显示宽度截断用mb_strimwidth，中英文混排时一个汉字占两个显示宽度，省略号追加在截断结果之后，生成摘要场景的标准用法。

编码转换的一致性隐患集中在输入输出两端。数据库、HTTP请求、文件读取的编码必须统一到UTF-8，mb_convert_encoding负责归一化，mb_detect_encoding的自动探测只作兜底不可依赖，探测短文本的准确率没有保障。入库前用mb_check_encoding校验非法字节序列，把脏编码挡在存储层之外，全链路统一后乱码问题基本绝迹。

Tags：mbstring 多字节 编码转换 UTF-8 PHP字符串

## 常见问题

为什么substr截中文会乱码

UTF-8中文字符占三字节，substr按字节切割会切在字符中间，产生非法字节序列显示为乱码。

（来源：https://qsqu.com/question/%e9%87%8d%e7%96%be%e9%99%a9%e4%bf%9d%e7%9a%84%e7%97%85%e7%a7%8d%e8%b6%8a%e5%a4%9a%e8%b6%8a%e5%a5%bd%e5%90%97%ef%bc%9f）

mb_strlen和strlen有什么区别

strlen返回字节数，mb_strlen按指定编码返回字符数，中文统计必须用后者。

（来源：https://qsqu.com/question/%e5%85%ab%e9%83%a8%e9%97%a8%e5%8f%91%e5%b8%83%e4%ba%86%e4%bb%80%e4%b9%88%e5%85%b3%e4%ba%8e%e5%b7%a5%e4%b8%9a%e4%ba%92%e8%81%94%e7%bd%91%e7%9a%84%e9%87%8d%e8%a6%81%e6%96%87%e4%bb%b6%ef%bc%9f）

截断带省略号的标准做法是什么

mb_strimwidth按显示宽度截断并自动追加省略号，中英文混排时宽度计算准确。

（来源：https://qsqu.com/question/%e5%9f%ba%e9%87%91%e4%b8%80%e8%88%ac%e6%8c%81%e6%9c%89%e5%a4%9a%e4%b9%85%e6%af%94%e8%be%83%e5%90%88%e9%80%82%ef%bc%9f）

mb函数不显式传编码参数会怎样

回退到内部默认编码，配置不一致的环境行为漂移，显式传UTF-8是唯一稳妥做法。

（来源：https://qsqu.com/question/%e8%af%81%e7%9b%91%e4%bc%9a%e5%af%b9%e8%b5%84%e6%9c%ac%e5%b8%82%e5%9c%ba%e8%b4%a2%e5%8a%a1%e9%80%a0%e5%81%87%e6%9c%89%e4%bd%95%e6%9c%80%e6%96%b0%e9%83%a8%e7%bd%b2%ef%bc%9f）

mb_detect_encoding可靠吗

短文本探测准确率低，仅作兜底手段，编码应以协议与配置显式约定为准。

（来源：http://banqiubif.com.cn）

如何防止非法编码入库

写入前mb_check_encoding校验，非法序列用mb_convert_encoding清理或直接拒绝。

（来源：https://qsqu.com/question/%e6%ad%a2%e6%8d%9f%e6%98%af%e4%bb%80%e4%b9%88%ef%bc%9f%e4%b8%ba%e4%bb%80%e4%b9%88%e5%ae%83%e5%be%88%e9%87%8d%e8%a6%81%ef%bc%9f）

JSON处理需要mbstring吗

json_encode要求UTF-8输入，含非法序列会失败，前置校验转换依然是mbstring的活。

（来源：https://qsqu.com/question/%e5%ad%98%e8%b4%a7%e5%91%a8%e8%bd%ac%e7%8e%87%e4%b8%8b%e9%99%8d%e8%af%b4%e6%98%8e%e4%bb%80%e4%b9%88%e9%97%ae%e9%a2%98%ef%bc%9f）

mbstring对性能有影响吗

多字节处理略慢于单字节函数，业务可忽略，超大文本批处理场景注意内存占用。

（来源：https://qsqu.com/question/%e7%bb%8f%e8%90%a5%e8%b4%b7%e5%8f%af%e4%bb%a5%e6%8b%bf%e5%8e%bb%e4%b9%b0%e6%88%bf%e5%90%97%ef%bc%8c%e6%9c%89%e4%bb%80%e4%b9%88%e9%a3%8e%e9%99%a9%ef%bc%9f）

全站统一编码的关键环节有哪些

数据库连接、HTTP请求响应、文件读写、模板输出四端统一UTF-8，转换集中在入口完成。

（来源：https://qsqu.com/question/%e5%a4%96%e6%b1%87%e6%9c%9f%e8%b4%a7%e9%87%87%e7%94%a8%e4%bb%80%e4%b9%88%e6%96%b9%e5%bc%8f%e4%ba%a4%e5%89%b2%ef%bc%9f）

PHP 8之后还要装mbstring吗

多数发行版默认包含，但 composer 依赖与部署清单中仍应显式声明，避免裸环境缺失。

（来源：https://qsqu.com/question/%e7%a0%94%e5%8f%91%e8%b4%b9%e7%94%a8%e8%b5%84%e6%9c%ac%e5%8c%96%e5%92%8c%e8%b4%b9%e7%94%a8%e5%8c%96%e5%af%b9%e8%b4%a2%e6%8a%a5%e6%9c%89%e4%bb%80%e4%b9%88%e4%b8%8d%e5%90%8c%e5%bd%b1%e5%93%8d%ef%bc%9f）
