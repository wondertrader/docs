# 扩展行情解析器ExtParser

### `ExtParser`是什么
---
**WonderTrader**全部底层组件都采用`C++`开发，即使使用`wtpy`，其实也是作为一个入口，调用底层组件进行回测交易。所以如果想要对**WonderTrader**做二次开发，对于使用者的`C++`水平是有较高的要求的。一方面需要熟练运用`C++`实现业务逻辑，另外一方面还需要有足够的经验解决`C++`中遇到的各种编程和设计的问题。

可是在实践中，我们还是会遇到很多场景下，需要自己实现行情的接入。但是如果一定要从`C++`底层开发新的行情接入模块，可行性也不是特别高：
- 一方面，很难要求每个使用者都有非常高的`C++`开发水平
- 另一方面，有一些柜台系统运行模式偏互联网风格，这种柜台系统一般会提供多种不同类型的接口，如`http`、`websocket`，数据包的封装，也采用互联网行业较流行的格式，如`json`等，而这类接口的形式，如果直接采用`C++`实现，开发效率相对较低，而且如果接口调整的时候，`C++`底层修改不够方便

`ExtParser`的目标，就是允许用户在`wtpy`中实现一个`python`版本的行情接入模块，通过在`python`中直接分配`C++`兼容的数据结构内存块，传递给底层直接使用，避免二次转码，降低开销，提升解析效率。

`ExtParser`的优势在于：
- `Python`实现，相较于`C++`更容易上手
- 语言更灵活，可以很容易解析`json`、`xml`等各种数据格式
- 更容易对接`websocket`接口
- 相比`C++`，更容易维护

`ExtParser`也有不大适应的场景：
- 行情数据量大，并发高
- 低延时场景无法使用
主要还是因为`Python`运行效率相对较低，而且还有`GIL`的问题，所以高并发、大数据量、低延时的场景下，都应该慎用`ExtParser`。


### `ExtParser`怎么用
---
要使用`ExtParser`，只需要在`Python`中实现一个`BaseExtParser`，并传递给`WtDtEngine`，就可以使用了，代码如下：
```py
from wtpy import BaseExtParser
from wtpy import WTSTickStruct
from ctypes import byref
import threading
import time

from wtpy import WtDtEngine

class MyParser(BaseExtParser):
    def __init__(self, id: str):
        super().__init__(id)
        self.__worker__ = None

    def init(self, engine:WtEngine):
        '''
        初始化
        '''
        super().init(engine)

    def random_sim(self):
        while True:
            curTick = WTSTickStruct()
            curTick.code = bytes("IF2106", encoding="UTF8")
            curTick.exchg = bytes("CFFEX", encoding="UTF8")

            self.__engine__.push_quote_from_extended_parser(self.__id__, byref(curTick), True)
            time.sleep(1)


    def connect(self):
        '''
        开始连接
        '''
        print("connect")
        if self.__worker__ is None:
            self.__worker__ = threading.Thread(target=self.random_sim, daemon=True)
            self.__worker__.start()
        return

    def disconnect(self):
        '''
        断开连接
        '''
        print("disconnect")
        return

    def release(self):
        '''
        释放，一般是进程退出时调用
        '''
        print("release")
        return

    def subscribe(self, fullCode:str):
        '''
        订阅实时行情\n
        @fullCode   合约代码，格式如CFFEX.IF2106
        '''
        # print("subscribe: " + fullCode)
        return

    def unsubscribe(self, fullCode:str):
        '''
        退订实时行情\n
        @fullCode   合约代码，格式如CFFEX.IF2106
        '''
        # print("unsubscribe: " + fullCode)
        return


if __name__ == "__main__":
    #创建一个运行环境，并加入策略
    myParser = MyParser("test")
    engine = WtDtEngine()
    engine.initialize("dtcfg.yaml", "logcfgdt.yaml")
    engine.add_exetended_parser(myParser)
    engine.run()
    kw = input('press any key to exit\n')
```

更多详情可以参考以下`demo`: 
- `wtpy/demos/datakit_fut/testExtParser.py`
- `wtpy/demos/datakit_allday`
- `wtpy/demos/test_extmodules`