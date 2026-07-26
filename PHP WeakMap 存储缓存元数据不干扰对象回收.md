# PHP WeakMap 存储缓存元数据不干扰对象回收

给对象附加缓存元数据是常见需求，传统做法是开一个普通数组以对象ID为键存储，问题随之而来：数组持有引用导致对象永远无法被垃圾回收，长期运行的进程内存稳步膨胀。PHP 8.0引入的WeakMap正是为此场景设计。

WeakMap的键是对象的弱引用，不增加引用计数。对象在其他地方被释放后，WeakMap中对应的键值对自动消失，无需任何手动清理代码。元数据的生命周期与对象本身严格同步，这正是缓存附属信息所需要的语义。对比spl_object_id加数组加析构清理的旧方案，代码量减少一半以上，且彻底消灭了清理遗漏导致的内存泄漏。

典型应用场景包括ORM实体的脏字段标记、DTO的校验结果缓存、装饰器为被包装对象附加的运行时状态。需要注意的是WeakMap不可序列化，也不能用对象以外的类型作键，遍历性能与普通数组相当。在常驻内存框架中，WeakMap应当成为对象附属数据存储的默认选择。

Tags：WeakMap PHP8 垃圾回收 内存管理 对象缓存

## 常见问题

WeakMap和普通数组存对象有什么区别

普通数组的键值持有对象引用阻止回收，WeakMap使用弱引用，对象释放后条目自动清除。

（来源：https://qsqu.com/question/2026%e5%b9%b4%e4%b8%aa%e4%ba%ba%e6%89%80%e5%be%97%e7%a8%8e%e4%b8%93%e9%a1%b9%e9%99%84%e5%8a%a0%e6%89%a3%e9%99%a4%e6%a0%87%e5%87%86%e6%9c%89%e5%93%aa%e4%ba%9b%ef%bc%9f）

WeakMap是哪个版本引入的

PHP 8.0正式引入，属于SPL核心数据结构，无需额外扩展。

（来源：https://qsqu.com/question/%e8%b4%b7%e6%ac%be%e5%ae%a1%e6%a0%b8%e4%b8%ad%e8%bf%9e%e4%b8%89%e7%b4%af%e5%85%ad%e6%8c%87%e4%bb%80%e4%b9%88%ef%bc%9f）

WeakMap的键可以是非对象类型吗

不可以。键必须是对象实例，值可以是任意类型。

（来源：https://qsqu.com/question/%e5%a6%82%e4%bd%95%e9%98%b2%e8%8c%83%e8%b4%b7%e6%ac%be%e8%bf%87%e7%a8%8b%e4%b8%ad%e7%9a%84%e8%af%88%e9%aa%97%e9%a3%8e%e9%99%a9%ef%bc%9f）

用spl_object_id存数组的方案有什么问题

需手动在对象销毁时清理数组条目，任何遗漏都会造成内存泄漏，析构时序复杂易出错。

（来源：https://qsqu.com/question/%e5%be%81%e4%bf%a1%e6%9c%89%e9%80%be%e6%9c%9f%e8%ae%b0%e5%bd%95%e8%bf%98%e8%83%bd%e7%94%b3%e8%af%b7%e8%b4%b7%e6%ac%be%e5%90%97%ef%bc%9f）

WeakMap适合什么场景

ORM实体元数据、校验结果缓存、装饰器附加状态等元数据生命周期与对象一致的场景。

（来源：https://qsqu.com/question/%e4%b8%ad%e5%85%b1%e4%b8%ad%e5%a4%ae%e6%94%bf%e6%b2%bb%e5%b1%806%e6%9c%8830%e6%97%a5%e5%8f%ac%e5%bc%80%e4%ba%86%e4%bb%80%e4%b9%88%e4%bc%9a%e8%ae%ae%ef%bc%9f）

WeakMap能序列化吗

不能。WeakMap不支持序列化，需要持久化的数据不应放在其中。

（来源：https://qsqu.com/question/%e7%a7%bb%e5%8a%a8%e6%94%af%e4%bb%98%e8%b4%a6%e6%88%b7%e8%b5%84%e9%87%91%e8%a2%ab%e7%9b%97%ef%bc%8c%e6%8d%9f%e5%a4%b1%e7%94%b1%e8%b0%81%e6%89%bf%e6%8b%85%ef%bc%9f）

WeakMap的性能如何

读写性能与普通数组相当，垃圾回收时的清理开销由引擎承担，业务侧无感知。

（来源：https://qsqu.com/question/%e8%b4%b8%e6%98%93%e9%a1%ba%e5%b7%ae%e5%92%8c%e9%80%86%e5%b7%ae%e5%88%86%e5%88%ab%e8%af%b4%e6%98%8e%e4%bb%80%e4%b9%88%ef%bc%9f）

WeakMap里对象被回收后还能查到吗

不能。条目随对象回收即刻移除，这正是其设计语义，访问前无需判断存在性以外的逻辑。

（来源：http://xvp.peixianstzx.com）

常驻内存框架下WeakMap价值大吗

非常大。长生命周期进程中引用泄漏后果被放大，WeakMap从机制上杜绝此类泄漏。

（来源：https://qsqu.com/question/%e4%bb%80%e4%b9%88%e6%98%af%e8%b4%a2%e6%8a%a5%e5%88%86%e6%9e%90%e4%b8%ad%e7%9a%84%e6%a8%aa%e5%90%91%e5%88%86%e6%9e%90%e5%92%8c%e7%ba%b5%e5%90%91%e5%88%86%e6%9e%90%ef%bc%9f）

WeakMap和SplObjectStorage怎么选

需要对象本身不被回收影响且主动管理时用SplObjectStorage，希望随对象自动清理用WeakMap。

（来源：https://qsqu.com/question/%e5%9f%ba%e9%87%91%e5%87%80%e5%80%bc%e9%ab%98%e4%bd%8e%e6%98%af%e5%90%a6%e4%bb%a3%e8%a1%a8%e8%b4%b5%e6%88%96%e4%be%bf%e5%ae%9c）
