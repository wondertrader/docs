# WtExecMon独立执行管理器

### WtExecMon是什么

在设计的时候，**WonderTrader**已经尽可能地考虑了各种场景下的不同需求，自上而下的架构设计，可以让**WonderTrader**很轻松的在不同的场景下适配起来。无论**WonderTrader**适应的场景多么全面，总是有**WonderTrader**不能覆盖的交易场景。
那么在**WonderTrader**中，是否可以从不同的场景中，设计一种共性的基础功能模块呢？答案是可以的，那就是**交易执行**。无论策略的实现方式是什么样的，最终都要通过交易接口下单。
`WtExecMon`独立执行管理器就是从**WonderTrader**中将执行模块单独剥离出来，封装成一个独立的模块。用户可以绕开`CTA`引擎、`SEL`引擎等，直接将最终的组合目标仓位丢给独立执行管理器。
`WtExecMon`会调用*多账户执行器*，并且针对不同标的创建*不同的执行单元*，不同标的的执行算法也通过*线程池*同步并发，最大限度的*提升执行效率*。

**M+1+N执行架构**
![](../images/m1n.png)

### WtExecMon怎么用
`WtExecMon`的使用比较简单，只需要启动以后，然后直接设置标的的目标仓位即可。调用代码如下：
```py
from wtpy import WtExecApi
import time

def test_exec_mon():
    api = WtExecApi()
    api.initialize(logCfg = "logcfgexec.yaml")
    api.config(cfgfile = 'cfgexec.yaml')
    api.run()

    time.sleep(10)
    api.set_position("CFFEX.IF.HOT", 1)

test_exec_mon()
```
更多信息可以参考demo: `wtpy/demos/test_execmon`。