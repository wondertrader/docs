# WtDataHelper

### `WtDatahelper`是什么
---
作为一个内核使用`C++`开发的量化交易框架，**WonderTrader**是天然追求更高的性能的。作为一个量化交易框架，如果高效地读写数据就成了追求高性能的必须要解决的问题。

**WonderTrader**采用自定义的数据结构对文件进行存储，为了提高访问性能，实时数据直接采用不原始数据结构连续内存排列的方式存储数据，而历史数据则采用`zstd`进行压缩存放，在读取的时候做一次解压就可以全部加载到内存中。采用这样的机制，**WonderTrader**在数据读写的性能上可以达到非常高的效率。

我们在使用的过程中，新的数据可以通过`datakit`来落地，历史数据我们也可以通过[WtDHFacor](./wtdhfactory.md)来拉取。但是**WonderTrader**毕竟不可能覆盖全部的数据源，另外在做数据维护和管理的时候，也会面临着检查指定数据文件的场景。

`WtDataHelper`就是为了给更多场景下的数据读写提供灵活的访问接口的。`WtDataHelper`主要提供的就是数据文件的读写能力。具体如下：

```py
class WtDataHelper:

    def dump_bars(self, binFolder:str, csvFolder:str, strFilter:str=""):
        '''
        将目录下的.dsb格式的历史K线数据导出为.csv格式
        @binFolder  .dsb文件存储目录
        @csvFolder  .csv文件的输出目录
        @strFilter  代码过滤器(暂未启用)
        '''
        pass

    def dump_ticks(self, binFolder: str, csvFolder: str, strFilter: str=""):
        '''
        将目录下的.dsb格式的历史Tik数据导出为.csv格式
        @binFolder  .dsb文件存储目录
        @csvFolder  .csv文件的输出目录
        @strFilter  代码过滤器(暂未启用)
        '''
        pass

    def trans_csv_bars(self, csvFolder: str, binFolder: str, period: str):
        '''
        将目录下的.csv格式的历史K线数据转成.dsb格式
        @csvFolder  .csv文件的输出目录
        @binFolder  .dsb文件存储目录
        @period     K线周期，m1-1分钟线，m5-5分钟线，d-日线
        '''
        pass

    def read_dsb_ticks(self, tickFile: str) -> WtTickRecords:
        '''
        读取.dsb格式的tick数据
        @tickFile   .dsb的tick数据文件
        @return     WtTickRecords
        '''
        pass


    def read_dsb_bars(self, barFile: str, isDay:bool = False) -> WtBarRecords:
        '''
        读取.dsb格式的K线数据
        @tickFile   .dsb的K线数据文件
        @return     WtBarRecords
        '''
        pass

    def read_dmb_ticks(self, tickFile: str) -> WtTickRecords:
        '''
        读取.dmb格式的tick数据
        @tickFile   .dmb的tick数据文件
        @return     WTSTickStruct的list
        '''
        pass

    def read_dmb_bars(self, barFile: str) -> WtBarRecords:
        '''
        读取.dmb格式的K线数据
        @tickFile   .dmb的K线数据文件
        @return     WTSBarStruct的list
        '''
        pass

    def store_bars(self, barFile:str, firstBar:POINTER(WTSBarStruct), count:int, period:str) -> bool:
        '''
        将K线转储到dsb文件中
        @barFile    要存储的文件路径
        @firstBar   第一条bar的指针
        @count      一共要写入的数据条数
        @period     周期，m1/m5/d
        '''
        pass

    def store_ticks(self, tickFile:str, firstTick:POINTER(WTSTickStruct), count:int) -> bool:
        '''
        将Tick数据转储到dsb文件中
        @tickFile   要存储的文件路径
        @firstTick  第一条tick的指针
        @count      一共要写入的数据条数
        '''
        pass

    def resample_bars(self, barFile:str, period:str, times:int, fromTime:int, endTime:int, sessInfo:SessionInfo) -> WtBarRecords:
        '''
        重采样K线
        @barFile    dsb格式的K线数据文件
        @period     基础K线周期，m1/m5/d
        @times      重采样倍数，如利用m1生成m3数据时，times为3
        @fromTime   开始时间，日线数据格式yyyymmdd，分钟线数据为格式为yyyymmddHHMMSS
        @endTime    结束时间，日线数据格式yyyymmdd，分钟线数据为格式为yyyymmddHHMMSS
        @sessInfo   交易时间模板
        '''
        pass
```

### WtDataHelper怎么用
---
`WtDataHelper`的使用相对简单，就是调用接口读写文件即可，下面是代码示例：

```py
from wtpy.wrapper import WtDataHelper
from wtpy.WtCoreDefs import WTSBarStruct, WTSTickStruct
from ctypes import POINTER
from wtpy.SessionMgr import SessionMgr
import pandas as pd

dtHelper = WtDataHelper()

def test_store_bars():
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

    dtHelper.store_bars(barFile="./CFFEX.IF.HOT_m5.bin", firstBar=buffer, count=len(df), period="m5")
    
def test_store_ticks():

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
            setattr(curTick, f"bid_price_{x}", float(df[i]["申买价"+tags[x]]))
            setattr(curTick, f"bid_qty_{x}", float(df[i]["申买量"+tags[x]]))
            setattr(curTick, f"ask_price_{x}", float(df[i]["申卖价"+tags[x]]))
            setattr(curTick, f"ask_qty_{x}", float(df[i]["申卖量"+tags[x]]))

    dtHelper.store_ticks(tickFile="./SHFE.rb.HOT_ticks.dsb", firstTick=buffer, count=len(df))

def test_resample():
    # 测试重采样
    sessMgr = SessionMgr()
    sessMgr.load("sessions.json")
    sInfo = sessMgr.getSession("SD0930")
    ret = dtHelper.resample_bars("IC2009.dsb",'m1',5,202001010931,202009181500,sInfo)
    print(ret)
```

更多详情可以参考`demo`: `wtpy/demos/test_datahelper`