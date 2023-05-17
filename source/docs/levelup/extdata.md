# 对接外部数据存储


### 为什么要对接外部数据存储
---
为了让**WonderTrader**更精简更高效，从架构设计之初，**WonderTrader**就否决了对第三方组件的依赖。所以**WonderTrader**的数据存储全部采用自研的数据结构，按照文件存储。这样的设计方案，除了不依赖第三方组件之外，数据读写的速度也是非常高的，而且可以节省数据格式转换的开销，加载到内存以后直接就是内部的数据结构。

虽然**WonderTrader**自身数据存储机制设计已经非常优秀了，对于很多用户来说都绝对能够满足需求。但是始终存在一类用户，他们可能是成熟的量化团队，一开始使用的其他量化平台，而数据的存储也是针对其他的量化平台搭建的。他们在迁移到**WonderTrader**的时候，要面对的第一件事情就是数据的转换。这种转换，除了历史数据的转换之外，还必须要考虑到新数据的及时同步，从实现的角度来说是非常繁琐的，而且还需要反复的测试才能稳定下来。

有没有一种方法，可以让**WonderTrader**直接从原有的数据存储引起加载数据，并且在`datakit`做收盘作业的时候能够再将新的数据存到原有的数据引擎呢？

答案是有！为了解决这种平台迁移的首要问题，**WonderTrader**专门提供了两个组件：
- 扩展数据加载器`ExtDataLoader`
- 扩展数据存储器`ExtDataDumper`

### `ExtDataLoader`简介
---
`ExtDataLoader`组件，是由回测框架和实盘框架调用的，允许用户自己实现从外部数据存储加载数据。如果用户向底层核心注册了自定义的`ExtDataLoader`，那么系统会在加载数据的时候，主动调用`ExtDataLoader`的数据加载接口。`ExtDataLoader`支持的接口如下：
```py
class BaseExtDataLoader:

    def __init__(self):
        pass

    def load_final_his_bars(self, stdCode:str, period:str, feeder) -> bool:
        '''
        加载最终历史K线（回测、实盘）
        该接口一般用于加载外部处理好的复权数据、主力合约数据

        @stdCode    合约代码，格式如CFFEX.IF.2106
        @period     周期，m1/m5/d1
        @feeder     回调函数，feed_raw_bars(bars:POINTER(WTSBarStruct), count:int)
        '''
        return False

    def load_raw_his_bars(self, stdCode:str, period:str, feeder) -> bool:
        '''
        加载未加工的历史K线（回测、实盘）
        该接口一般用于加载原始的K线数据，如未复权数据和分月合约数据

        @stdCode    合约代码，格式如CFFEX.IF.2106
        @period     周期，m1/m5/d1
        @feeder     回调函数，feed_raw_bars(bars:POINTER(WTSBarStruct), count:int)
        '''
        return False

    def load_his_ticks(self, stdCode:str, uDate:int, feeder) -> bool:
        '''
        加载历史K线（只在回测有效，实盘只提供当日落地的）
        @stdCode    合约代码，格式如CFFEX.IF.2106
        @uDate      日期，格式如yyyymmdd
        @feeder     回调函数，feed_raw_bars(bars:POINTER(WTSTickStruct), count:int)
        '''
        return False

    def load_adj_factors(self, stdCode:str = "", feeder = None) -> bool:
        '''
        加载的权因子
        @stdCode    合约代码，格式如CFFEX.IF.2106，如果stdCode为空，则是加载全部除权数据，如果stdCode不为空，则按需加载
         @feeder     回调函数，feed_adj_factors(stdCode:str, dates:list, factors:list)
        '''
        return False
```

如果要使用`ExtDataLoader`，可以参考以下代码：
```py
from wtpy.ExtModuleDefs import BaseExtDataLoader
from ctypes import POINTER
from wtpy.WtCoreDefs import WTSBarStruct, WTSTickStruct

import pandas as pd

import random

from wtpy import WtEngine,WtBtEngine,EngineType
from wtpy.apps import WtBtAnalyst
from Strategies.DualThrust import StraDualThrust

class MyDataLoader(BaseExtDataLoader):
    
    def load_final_his_bars(self, stdCode:str, period:str, feeder) -> bool:
        '''
        加载历史K线（回测、实盘）
        @stdCode    合约代码，格式如CFFEX.IF.2106
        @period     周期，m1/m5/d1
        @feeder     回调函数，feed_raw_bars(bars:POINTER(WTSBarStruct), count:int, factor:double)
        '''
        print("loading %s bars of %s from extended loader" % (period, stdCode))

        df = pd.read_csv('../storage/csv/CFFEX.IF.HOT_m5.csv')
        df = df.rename(columns={
            '<Date>':'date',
            ' <Time>':'time',
            ' <Open>':'open',
            ' <High>':'high',
            ' <Low>':'low',
            ' <Close>':'close',
            ' <Volume>':'vol',
            })
        df['date'] = df['date'].astype('datetime64').dt.strftime('%Y%m%d').astype('int64')
        df['time'] = (df['date']-19900000)*10000 + df['time'].str.replace(':', '').str[:-2].astype('int')

        BUFFER = WTSBarStruct*len(df)
        buffer = BUFFER()

        def assign(procession, buffer):
            tuple(map(lambda x: setattr(buffer[x[0]], procession.name, x[1]), enumerate(procession)))


        df.apply(assign, buffer=buffer)
        print(df)
        print(buffer[0].to_dict)
        print(buffer[-1].to_dict)

        feeder(buffer, len(df))
        return True

    def load_his_ticks(self, stdCode:str, uDate:int, feeder) -> bool:
        '''
        加载历史K线（只在回测有效，实盘只提供当日落地的）
        @stdCode    合约代码，格式如CFFEX.IF.2106
        @uDate      日期，格式如yyyymmdd
        @feeder     回调函数，feed_raw_ticks(ticks:POINTER(WTSTickStruct), count:int)
        '''
        print("loading ticks on %d of %s from extended loader" % (uDate, stdCode))

        df = pd.read_csv('../storage/csv/rb主力连续_20201030.csv')
        BUFFER = WTSTickStruct*len(df)
        buffer = BUFFER()

        tags = ["一","二","三","四","五"]

        for i in range(len(df)):
            curTick = buffer[i]

            curTick.exchg = b"SHFE"
            curTick.code = b"SHFE.rb.HOT"

            curTick.price = float(df[i]["最新价"])
            curTick.open = float(df[i]["今开盘"])
            curTick.high = float(df[i]["最高价"])
            curTick.low = float(df[i]["最低价"])
            curTick.settle = float(df[i]["本次结算价"])
            
            curTick.total_volume = float(df[i]["数量"])
            curTick.total_turnover = float(df[i]["成交额"])
            curTick.open_interest = float(df[i]["持仓量"])

            curTick.trading_date = int(df[i]["交易日"])
            curTick.action_date = int(df[i]["业务日期"])
            curTick.action_time = int(df[i]["最后修改时间"].replace(":",""))*1000 + int(df[i]["最后修改毫秒"])

            curTick.pre_close = float(df[i]["昨收盘"])
            curTick.pre_settle = float(df[i]["上次结算价"])
            curTick.pre_interest = float(df[i]["昨持仓量"])

            for x in range(5):
                curTick.bid_prices[x] = float(df[i]["申买价"+tags[x]])
                curTick.bid_qty[x] = float(df[i]["申买量"+tags[x]])
                curTick.ask_prices[x] = float(df[i]["申卖价"+tags[x]])
                curTick.ask_qty[x] = float(df[i]["申卖量"+tags[x]])

        feeder(buffer, len(df))

def test_in_bt():
    engine = WtBtEngine(EngineType.ET_CTA)

    # 初始化之前，向回测框架注册加载器
    engine.set_extended_data_loader(loader=MyDataLoader(), bAutoTrans=False)

    engine.init('../common/', "configbt.yaml")

    engine.configBacktest(201909100930,201912011500)
    engine.configBTStorage(mode="csv", path="../storage/")
    engine.commitBTConfig()

    straInfo = StraDualThrust(name='pydt_IF', code="CFFEX.IF.HOT", barCnt=50, period="m5", days=30, k1=0.1, k2=0.1, isForStk=False)
    engine.set_cta_strategy(straInfo)

    engine.run_backtest()

    analyst = WtBtAnalyst()
    analyst.add_strategy("pydt_IF", folder="./outputs_bt/pydt_IF/", init_capital=500000, rf=0.02, annual_trading_days=240)
    analyst.run()

    kw = input('press any key to exit\n')
    engine.release_backtest()

def test_in_rt():
    engine = WtEngine(EngineType.ET_CTA)

    # 初始化之前，向实盘框架注册加载器
    engine.set_extended_data_loader(MyDataLoader())

    engine.init('../common/', "config.yaml")
    
    straInfo = StraDualThrust(name='pydt_au', code="SHFE.au.HOT", barCnt=50, period="m5", days=30, k1=0.2, k2=0.2, isForStk=False)
    engine.add_cta_strategy(straInfo)

    engine.run()

    kw = input('press any key to exit\n')

test_in_bt()
```
更多详细信息可以参考`demo`: `wtpy/demos/test_dataexts/testDtLoader.py`

### ExtDataDumper简介
---
`ExtDataDumper`，是由`datakit`调用的，允许用户自己转储到其他存储引擎。如果用户向`datakit`注册了一个自定义的数据转储器，`datakit`会在收盘作业的时候将底层数据块传递出来，由`ExtDataDumper`将历史行情数据转储。如果`datakit`配置的是`allday`模式，那么转储的接口会在每条基础K线闭合的时候调用。`ExtDataDumper`支持的接口如下：
```py
class BaseExtDataDumper:

    def __init__(self, id:str):
        self.__id__ = id

    def id(self):
        return self.__id__

    def dump_his_bars(self, stdCode:str, period:str, bars, count:int) -> bool:
        '''
        加载历史K线（回测、实盘）
        @stdCode    合约代码，格式如CFFEX.IF.2106
        @period     周期，m1/m5/d1
        @bars       回调函数，WTSBarStruct的指针
        @count      数据条数
        '''
        return True

    def dump_his_ticks(self, stdCode:str, uDate:int, ticks, count:int) -> bool:
        '''
        加载历史K线（只在回测有效，实盘只提供当日落地的）
        @stdCode    合约代码，格式如CFFEX.IF.2106
        @uDate      日期，格式如yyyymmdd
        @ticks      回调函数，WTSTickStruct的指针
        @count      数据条数
        '''
        return True
```

如果要使用`ExtDataDumper`，可以参考以下代码：
```py
from wtpy import WtDtEngine

from wtpy.ExtModuleDefs import BaseExtDataDumper

class MyDataDumper(BaseExtDataDumper):
    def __init__(self, id:str):
        BaseExtDataDumper.__init__(self, id)

    def dump_his_bars(self, stdCode:str, period:str, bars, count:int) -> bool:
        '''
        加载历史K线（回测、实盘）
        @stdCode    合约代码，格式如CFFEX.IF.2106
        @period     周期，m1/m5/d1
        @bars       回调函数，WTSBarStruct的指针
        @count      数据条数
        '''
        print("dumping %s bars of %s via extended dumper" % (period, stdCode))
        return True

    def dump_his_ticks(self, stdCode:str, uDate:int, ticks, count:int) -> bool:
        '''
        加载历史K线（只在回测有效，实盘只提供当日落地的）
        @stdCode    合约代码，格式如CFFEX.IF.2106
        @uDate      日期，格式如yyyymmdd
        @ticks      回调函数，WTSTickStruct的指针
        @count      数据条数
        '''
        print("dumping ticks on %d of %s via extended dumper" % (uDate, stdCode))
        return True

def test_ext_dumper():
    #创建一个运行环境，并加入策略
    engine = WtDtEngine()
    engine.add_extended_data_dumper(MyDataDumper("dumper"))
    engine.initialize("dtcfg.yaml", "logcfgdt.yaml")
    
    engine.run()

    kw = input('press any key to exit\n')
```
更多详细信息可以参考`demo`: `wtpy/demos/test_dataexts/testDtDumper.py`