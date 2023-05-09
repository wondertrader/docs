# WtDtServo

### `WtDtServo`是什么

通常情况下，一个策略从研发到实盘的流程都遵循以下几个步骤：
- 1. **思路研究**，这个阶段一般验证一些思路是否可行，比如一些技术指标的组合，或者一些简单的交易逻辑
- 2. **策略回测**，如果思路被验证是可行的以后，就可以基于这些思路实现策略，除了主要逻辑的实现之外，还要加入止盈止损、资金管理、频率控制等模块，然后进行回测，看看策略的表现
- 3. **仿真跟踪**，如果回测可行，收益指标符合预期，就可以进行仿真跟踪，跟踪一段时间，看看策略在样本外数据的表现
- 4. **实盘上线**，仿真跟踪一段时间以后，基本符合实盘的要求，就可以进行实盘交易了

**WonderTrader**在整个量化交易的环节中，提供了数据组件`datakit`，`datakit`会自动将实时的行情数据落地，并实时重采样成`min1`、`min5`两种基础周期的K线。策略在回测和实盘（仿真）的时候，就可以直接和`datakit`配合使用，通过数据读取模块`WtDataReader`就可以直接读取`datakit`落地的数据。

对于思路研究来说，一般就直接对数据进行向量化处理，而不是通过回测模拟的事件驱动机制来回放数据。`WtDtServo`就是针对这样的需求，向用户提供了随机访问`datakit`落地的数据的接口。概括起来，包含两类接口：
- 根据指定的时间区间获取历史数据
- 根据指定截止时间以及所需的数据量获取历史数据

实际上，除了用户层面的接口之外，`WtDtServo`底层也采用了`WtDataReader`一样的缓存机制，提供了和策略内无差别的访问效率和使用体验。

虽然`WtDtServo`最初的设计思路是针对投研需求，但是`WtDtServo`因为提供的是基础数据的访问能力，所以适用的场景是非常丰富的，`wtpy`内部很多组件都依赖`WtDtServo`提供的数据访问能力：
- 回测查看器`WtBtSnooper`
- **WonderTrader**工作台`WtStudio`
- 在线回测管理器`WtBtMonitor`(该组件以后可能会取消，慎用)

### 如何使用WtDtServo
对于用户来说，要使用`WtDtServo`，可以参考以下代码：
```py
from wtpy import WtDtServo

dtServo = WtDtServo()
# 这是基础数据文件
dtServo.setBasefiles(folder="../common/")
# 这里设置datakit配置的数据落地根目录
dtServo.setStorage(path='../storage/')

# 读取IF主力合约的前复权数据
bars = dtServo.get_bars("CFFEX.IF.HOT-", "m5", fromTime=202205010930, endTime=202205281500).to_df()
bars.to_csv("CFFEX.IF.HOT-.csv")

# 读取IF主力合约的后复权数据
bars = dtServo.get_bars("CFFEX.IF.HOT+", "m5", fromTime=202205010930, endTime=202205281500).to_df()
bars.to_csv("CFFEX.IF.HOT+.csv")

# 读取IF主力合约的原始拼接数据
bars = dtServo.get_bars("CFFEX.IF.HOT", "m5", fromTime=202205010930, endTime=202205281500).to_df()
bars.to_csv("CFFEX.IF.HOT.csv")

# 读取IF主力合约的tick数据
bars = dtServo.get_ticks("CFFEX.IF.HOT", fromTime=202207250930, endTime=202207291500).to_df()
bars.to_csv("CFFEX.IF.HOT_ticks.csv")
```

如果想了解更多详情，可以参考`wtpy/demos/test_dtservo`