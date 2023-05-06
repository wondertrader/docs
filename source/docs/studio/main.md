# 延迟优化日记

### 测试环境
---
* 测试环境 Intel i9-10980 XE，未超频，未关闭超线程
* 测试项目 WtLatencyUFT

### 第一阶段 UFT拆分，完成后系统延迟在1us
---
* 弃用内部标准代码SHFR.rb.2205，全部改成SHFE.rb2205，减少转换的开销
* 去掉对主力次主力的支持，减少tick数据进来以后，对主力合约判断的开销
* 下单接口去掉usertag，不在TraderAdapter做任何数据落地，减少策略下单接口的开销
* 将WTSContractInfo和WTSCommodityInfo以及WTSSessionInfo在初始化的之后做直接关联，在使用的过程中不用再去查找，减少查找的开销
* 将接口相关的数据类型，新增一个WTSContractInfo指针成员变量，每次构造的时候设置该指针，减少查找的开销


### 第二阶段 时间函数，完成后系统延迟在500ns
---
* 延迟测试工具中，没次模拟tick都TimeUtils获取最新的时间，但是时间函数开销很大，而且win下调用TimeUtils::getDateTime比linux下快，导致linux测试的速度在2us左右
* 因为实盘情况下，ontick触发时间，行情接口已经把处理好的数据给parser了，所以parser只需要做数据转换就可以了，不用自己去读取数据，测试工具去掉获取时间的逻辑


### 第三阶段 字符串优化，完成以后系统延迟在250ns
---
* 开始性能探查器进行分析，发现StrUtils::split()、sprintf开销很大
* 关键路径上调用StrUtil::split的地方改成自己查找，不用split
* sprintf改成预分配的char[]，调用fmt::format_to进行格式化处理提高效率
* 将WTSDataDef中所有的std::string成员全部改成char[]，并提供直接访问内存的接口，减少赋值过程中的二次构造


### 第四阶段 内存分配优化，完成以后系统延迟在185ns
----
* 继续分析发现，瓶颈集中在new/delete，调研以后决定用boost的object_pool
* 新增一个模板类WTSPoolObject，内部用boost::object_pool做一个对象池，管理对象的分配，派生类将类型作为模板参数传入
* 将频繁分配的对象都从WTSPoolObject进行派生，这样频繁创建和释放的对象都放在池子里进行管理，减少了new/delete的开销


### 第五阶段 hash优化，完成以后系统延迟在175ns
---
* 继续分析发现，瓶颈变成了WTSBaseDataMgr的getContract，这下面使用robin_hashmap作为容器，已经是性能最高的容器了，再优化就需要从key着手了
* 于是将key改成int64[]，然后自己实现hash函数，处理key中的整数（整数计算会快），提升hashmap查找的效率，减少了约10ns的开销