# 扩展执行器ExtExecuter


### ExtExecuter是什么
---
**WonderTrader**全部底层组件都采用`C++`开发，即使使用`wtpy`，其实也是作为一个入口，调用底层组件进行回测交易。所以如果想要对**WonderTrader**做二次开发，对于使用者的`C++`水平是有较高的要求的。一方面需要熟练运用`C++`实现业务逻辑，另外一方面还需要有足够的经验解决`C++`中遇到的各种编程和设计的问题。

可是在实践中，我们还是会遇到很多场景下，需要自己实现交易接口。但是如果一定要从`C++`底层开发新的交易接口模块，可行性也不是特别高：
- 首先，很难要求每个使用者都有非常高的`C++`开发水平
- 其次，有一些柜台系统运行模式偏互联网风格，这种柜台系统一般会提供多种不同类型的接口，如`http`、`websocket`，数据包的封装，也采用互联网行业较流行的格式，如`json`等，而这类接口的形式，如果直接采用`C++`实现，开发效率相对较低，而且如果接口调整的时候，`C++`底层修改不够方便
- 再次，在股票交易交易中，有很多交易终端以**文件扫单**的方式提供下单接口，而这种方式只需要用户在使用的时候按照指定的规则提供一个下单文件就可以了

`ExtExecuter`的目标，就是允许用户在`wtpy`中实现一个`python`版本的信号执行模块，`C++`底层会通过接口直接向`ExtExecuter`传递指定标的的目标仓位，`ExtExecuter`就可以根据自身适配的交易接口，灵活的实现底层信号的传递。

`ExtExecuter`的优势在于：
- `Python`实现，相较于`C++`更容易上手
- 相比`C++`，更容易维护
- `ExtExecuter`和`C++`底层实现的执行器互相不冲突，可以同时使用


### `ExtExecuter`怎么用
---
要使用`ExtExecuter`，只需要在交易引擎中注册一个自己扩展的执行器模块，交易引擎就会自动调用该执行器，代码如下：
```py
from wtpy import  BaseExtExecuter

from wtpy import WtEngine,EngineType
from Strategies.DualThrust import StraDualThrust

class MyExecuter(BaseExtExecuter):
    def __init__(self, id: str, scale: float):
        super().__init__(id, scale)

    def init(self):
        print("inited")

    def set_position(self, stdCode: str, targetPos: float):
        print("position confirmed: %s -> %f " % (stdCode, targetPos))

if __name__ == "__main__":
    #创建一个运行环境，并加入策略
    engine = WtEngine(EngineType.ET_CTA)
    engine.init('../common/', "config.yaml")
    
    straInfo = StraDualThrust(name='pydt_au', code="SHFE.au.HOT", barCnt=50, period="m5", days=30, k1=0.2, k2=0.2, isForStk=False)
    engine.add_cta_strategy(straInfo)
    
    myExecuter = MyExecuter('exec', 1)
    engine.commitConfig()
    engine.add_exetended_executer(myExecuter)

    engine.run()

    kw = input('press any key to exit\n')
```
更多详情可以参考`demo`: `wtpy/demos/test_extmodules`