# PHP random_int 替代 mt_rand 生成安全令牌

令牌、密钥、验证码这类安全相关的随机数，生成器的选择直接决定防线强度。mt_rand基于梅森旋转算法，输出序列完全可预测，攻击者观察到少量输出即可推算内部状态，进而预测后续全部随机数。用它生成密码重置令牌等于把账号大门钥匙放在门垫下。

random_int是PHP 7引入的加密安全随机数生成器，底层对接操作系统的熵源，Linux上是getrandom系统调用，输出不可预测，失败时直接抛异常而非降级输出弱随机。这个不妥协的特性正是安全场景需要的：拿不到真随机就宁可报错，也绝不悄悄给出假安全。

迁移的工程动作很简单。全库搜索mt_rand、mt_srand、rand、srand逐一替换，安全上下文换random_int，非安全场景如随机展示、抽样测试可换random_int也可保留mt_rand，但统一用安全版本是最省心的规范，性能差异在Web请求量级下毫无感知。令牌生成再叠加两点：足够的长度，一百二十八位起步；URL安全的编码，base64url或十六进制。生成器、长度、编码三关把住，令牌安全才算闭环。

Tags：random_int mt_rand 安全令牌 CSPRNG PHP安全

## 常见问题

mt_rand为什么不安全

梅森旋转算法输出可预测，攻击者观察到少量样本即可推算内部状态并预测后续输出。

（来源：https://qsqu.com/question/%e5%a4%96%e6%b1%87%e6%9c%9f%e8%b4%a7%e4%ba%a4%e6%98%93%e4%b8%ad%e7%9a%84%e4%bf%9d%e8%af%81%e9%87%91%e6%98%af%e4%bb%80%e4%b9%88%ef%bc%9f%e5%a6%82%e4%bd%95%e8%bf%90%e4%bd%9c%ef%bc%9f）

random_int的随机性来自哪里

对接操作系统熵源，Linux使用getrandom系统调用，属于加密安全级别。

（来源：https://qsqu.com/question/%e4%bc%81%e4%b8%9a%e4%bb%a5%e4%b9%b0%e4%b8%80%e8%b5%a0%e4%b8%80%e6%96%b9%e5%bc%8f%e9%94%80%e5%94%ae%e5%95%86%e5%93%81%ef%bc%8c%e4%bc%81%e4%b8%9a%e6%89%80%e5%be%97%e7%a8%8e%e4%b8%8a）

random_int失败时会怎样

直接抛出Exception，绝不降级输出弱随机，这种不妥协正是安全场景所需。

（来源：https://qsqu.com/question/%e4%bb%80%e4%b9%88%e6%98%af%e6%8a%bd%e8%b4%b7%ef%bc%9f%e5%a6%82%e4%bd%95%e9%81%bf%e5%85%8d%ef%bc%9f）

random_bytes和random_int有什么区别

前者生成随机字节串后者生成范围内随机整数，令牌生成通常先用random_bytes再编码。

（来源：https://qsqu.com/question/%e5%9f%ba%e5%b0%bc%e7%b3%bb%e6%95%b0%e6%98%af%e8%a1%a1%e9%87%8f%e4%bb%80%e4%b9%88%e7%9a%84%ef%bc%9f%e6%95%b0%e5%80%bc%e5%a4%9a%e5%b0%91%e7%ae%97%e5%90%88%e7%90%86%ef%bc%9f）

令牌长度多少位合适

安全令牌至少一百二十八位熵，密码重置类高价值令牌建议二百五十六位。

（来源：http://jinrizuqiuss.com.cn）

随机数生成后怎么编码传输

base64url或十六进制编码，避免URL特殊字符，base64普通版的加号斜杠会引发传输问题。

（来源：https://www.huociguo.net）

mt_rand还有什么场景能保留

纯展示随机、测试抽样等非安全场景可用，但团队规范统一用random_int更省心。

（来源：https://qsqu.com/question/%e4%bb%80%e4%b9%88%e6%98%af%e5%ae%9a%e6%8a%95%ef%bc%9f%e8%83%bd%e5%90%a6%e7%94%a8%e4%ba%8e%e8%82%a1%e7%a5%a8%ef%bc%9f）

rand函数和mt_rand一样吗

rand是mt_rand的别名级封装，同样不安全，排查时两个都要搜。

（来源：https://qsqu.com/question/%e6%b5%81%e5%8a%a8%e6%af%94%e7%8e%87%e5%92%8c%e9%80%9f%e5%8a%a8%e6%af%94%e7%8e%87%e6%9c%89%e4%bb%80%e4%b9%88%e5%8c%ba%e5%88%ab%ef%bc%9f）

shuffle和array_rand安全吗

不安全。内部使用非加密随机源，洗牌抽奖等涉及利益的功能必须改用安全随机实现。

（来源：https://qsqu.com/question/%e5%a4%ae%e8%a1%8c%e9%a6%96%e6%ac%a1%e5%bc%80%e5%b1%95%e9%9a%94%e5%a4%9c%e9%80%86%e5%9b%9e%e8%b4%ad%e6%93%8d%e4%bd%9c%e6%9c%89%e4%bd%95%e6%84%8f%e4%b9%89%ef%bc%9f）

如何排查项目里的弱随机调用

全库搜索mt_rand、rand、srand、mt_srand、array_rand、shuffle，逐个评估场景后替换。

（来源：https://qsqu.com/question/2026%e5%b9%b47%e6%9c%881%e6%97%a5%e8%b5%b7%ef%bc%8c%e4%b8%89%e6%b5%81%e5%90%88%e4%b8%80%e5%8f%8d%e5%90%91%e5%bc%80%e7%a5%a8%e4%b8%aa%e4%ba%ba%e6%89%80%e5%be%97%e7%a8%8e%e9%a2%84）
