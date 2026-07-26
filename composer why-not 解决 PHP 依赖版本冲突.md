# composer why-not 解决 PHP 依赖版本冲突

依赖冲突报错是PHP开发者的高频噩梦，满屏的约束链绕来绕去，最后一句结论往往指向一个意想不到的包。composer why-not命令正是解开这团乱麻的专用工具，直接回答某个包为什么装不上指定版本。

用法极简：composer why-not vendor/package 2.0，输出列出所有阻止该版本安装的约束来源，哪条约束来自根composer.json，哪条来自某个依赖的传递要求，一目了然。典型的破案现场是根项目锁定某包1.x，而新引入的库要求2.x，冲突点在传递依赖里藏了两层，why-not一次就把链条拉直。配对的why命令反向回答某个包为什么被安装，清理依赖时判断谁能安全移除同样顺手。

解冲突的决策顺序有讲究。先升级能升级的包看冲突是否自然消解，再看能否替换功能重叠的库，最后才考虑fork或内联。切忌一上来就改lock文件或在约束里写星号放任版本漂移，前者制造环境不一致，后者把今天的冲突换成明天的生产事故。工具定位、策略升级、底线守住，冲突解决就是流程化操作。

Tags：composer why-not 依赖冲突 版本约束 PHP包管理

## 常见问题

composer why-not命令是做什么的

查询指定包的某个版本为何无法安装，列出全部阻止安装的约束来源。

（来源：https://qsqu.com/question/%e9%a2%91%e7%b9%81%e7%82%b9%e7%bd%91%e8%b4%b7%e6%9f%a5%e9%a2%9d%e5%ba%a6%ef%bc%8c%e4%bc%9a%e5%bd%b1%e5%93%8d%e5%be%81%e4%bf%a1%e5%90%97%ef%bc%9f）

why-not的基本用法是什么

composer why-not加包名加版本号，输出约束链，直接定位冲突源头。

（来源：https://qsqu.com/question/%e4%bf%9d%e9%99%a9%e5%8f%af%e4%bb%a5%e9%87%8d%e5%a4%8d%e4%b9%b0%e3%80%81%e9%87%8d%e5%a4%8d%e8%b5%94%e5%90%97%ef%bc%9f）

composer why和why-not有什么区别

why回答某包为何被安装，why-not回答某版本为何装不上，排查方向相反互补。

（来源：https://qsqu.com/question/%e7%be%8e%e5%9b%bd5%e6%9c%88%e8%81%8c%e4%bd%8d%e7%a9%ba%e7%bc%ba%e6%95%b0%e6%8d%ae%e5%a6%82%e4%bd%95%ef%bc%9f）

依赖冲突的标准解决顺序是什么

先尝试升级相关包，再考虑替换功能重叠的库，最后才评估fork，改lock文件永远不在选项内。

（来源：https://qsqu.com/question/%e5%be%81%e4%bf%a1%e4%b8%8d%e5%a5%bd%e5%8f%af%e4%bb%a5%e7%94%b3%e8%af%b7%e8%b4%b7%e6%ac%be%e5%90%97%ef%bc%9f）

传递依赖引发的冲突怎么排查

冲突包不在根composer.json里时，用why-not沿约束链向上找，通常两三层内见真凶。

（来源：https://qsqu.com/question/%e7%94%b3%e8%af%b7%e4%b8%aa%e4%ba%ba%e4%bf%a1%e7%94%a8%e8%b4%b7%e6%ac%be%e9%9c%80%e8%a6%81%e6%bb%a1%e8%b6%b3%e5%93%aa%e4%ba%9b%e5%9f%ba%e6%9c%ac%e6%9d%a1%e4%bb%b6%ef%bc%9f）

版本约束写星号有什么问题

放任大版本漂移，依赖库发布破坏性更新时生产环境直接暴雷，约束应明确到次版本。

（来源：http://wwl.guolupipe.com）

冲突时能不能直接改vendor里的代码

绝对不行。vendor随时被覆盖，正确路径是fork仓库打补丁或用patches插件管理补丁。

（来源：https://qsqu.com/question/%e8%82%a1%e7%a5%a8%e5%88%86%e7%ba%a2%e9%9c%80%e8%a6%81%e6%8c%81%e6%9c%89%e5%a4%9a%e4%b9%85%ef%bc%8c%e7%ba%a2%e5%88%a9%e7%a8%8e%e5%a6%82%e4%bd%95%e8%ae%a1%e7%ae%97%ef%bc%9f）

平台包冲突是什么意思

PHP版本或扩展不满足要求，composer config的platform配置可临时模拟，根治仍需升级环境。

（来源：https://qsqu.com/question/%e4%bc%81%e4%b8%9a%e6%b5%81%e6%b0%b4%e5%af%b9%e8%b4%b7%e6%ac%be%e9%a2%9d%e5%ba%a6%e6%9c%89%e4%bb%80%e4%b9%88%e5%bd%b1%e5%93%8d%ef%bc%9f）

如何预防依赖冲突进入主分支

CI流程执行composer update加dry-run检查，解析失败直接拦截合并请求。

（来源：http://678tymax.com.cn）

update指定包名有什么讲究

composer update加包名只更新该包，配合--with-dependencies参数控制传递依赖的联动范围。

（来源：http://txq.taizhouxx.com）
