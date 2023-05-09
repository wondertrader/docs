# WtDHFactory

### `WtDHFactory`是什么
无论是策略研究还是实盘交易，都离不开数据。使用**WonderTrader**做量化的第一阶段，需要先把历史数据准备好。

但是我们在处理历史数据的时候，一开始要面临的问题就是数据源的选择。市面上的数据源有免费的，也有收费的。当然收费的数据源质量相对要高一些，但是对于初学者来说，免费的数据源也是足够的。

**WonderTrader**为了方便用户使用，在`wtpy`中提供了一个数据辅助模块`WtDHFactory`，可以帮助用户从不同的数据源中获取数据，然后统一导出到文件或者数据库中，方便后续使用。

目前`WtDHFactory`支持的数据源有：
- `tushare`
- `baostock`
- `rqdata`

提供的如下接口：
```py
class BaseDataHelper:

    def dmpCodeListToFile(self, filename:str, hasIndex:bool=True, hasStock:bool=True):
        '''
        将代码列表导出到文件
        @filename   要输出的文件名，json格式
        @hasIndex   是否包含指数
        @hasStock   是否包含股票
        '''
        pass

    def dmpAdjFactorsToFile(self, codes:list, filename:str):
        '''
        将除权因子导出到文件
        @codes  股票列表，格式如["SSE.600000","SZSE.000001"]
        @filename   要输出的文件名，json格式
        '''
        pass

    def dmpBarsToFile(self, folder:str, codes:list, start_date:datetime=None, end_date:datetime=None, period:str="day"):
        '''
        将K线导出到指定的目录下的csv文件，文件名格式如SSE.600000_d.csv
        @folder 要输出的文件夹
        @codes  股票列表，格式如["SSE.600000","SZSE.000001"]
        @start_date 开始日期，datetime类型，传None则自动设置为1990-01-01
        @end_date   结束日期，datetime类型，传None则自动设置为当前日期
        @period K线周期，支持day、min1、min5
        '''
        pass

    def dmpBars(self, codes:list, cb, start_date:datetime=None, end_date:datetime=None, period:str="day"):
        '''
        将K线导出到指定的目录下的csv文件，文件名格式如SSE.600000_d.csv
        @cb     回调函数，格式如cb(exchg:str, code:str, firstBar:POINTER(WTSBarStruct), count:int, period:str)
        @codes  股票列表，格式如["SSE.600000","SZSE.000001"]
        @start_date 开始日期，datetime类型，传None则自动设置为1990-01-01
        @end_date   结束日期，datetime类型，传None则自动设置为当前日期
        @period K线周期，支持day、min1、min5
        '''
        pass
```

### `WtDHFactory`怎么用
其实在[基本用法/历史数据处理](../usage/histdata.md)中，已经介绍了如何使用`WtDHFactory`来准备历史数据了。

`WtDHFactory`采用工厂模式，使用的时候只需要传入对应的数据源名称，就可以获取到对应的数据源对象，然后调用对应的接口即可。具体用法可以参考以下代码：

```py
from wtpy.apps.datahelper import DHFactory as DHF

hlper = DHF.createHelper("baostock")
hlper.auth()

# tushare
# hlper = DHF.createHelper("tushare")
# hlper.auth(**{"token":"xxxxxxxxxxx", "use_pro":True})

# rqdata
# hlper = DHF.createHelper("rqdata")
# hlper.auth(**{"username":"00000000", "password":"0000000"})

# 落地股票列表
# hlper.dmpCodeListToFile("stocks.json")

# 下载K线数据
# hlper.dmpBarsToFile(folder='./', codes=["CFFEX.IF.HOT","CFFEX.IC.HOT"], period='min1')
# hlper.dmpBarsToFile(folder='./', codes=["CFFEX.IF.HOT","CFFEX.IC.HOT"], period='min5')
hlper.dmpBarsToFile(folder='./', codes=["SZSE.399005","SZSE.399006","SZSE.399303"], period='day')

# 下载复权因子
# hlper.dmpAdjFactorsToFile(codes=["SSE.600000",'SZSE.000001'], filename="./adjfactors.json")

# 将数据直接落地成dsb
def on_bars_block(exchg:str, stdCode:str, firstBar:POINTER(WTSBarStruct), count:int, period:str):
    from wtpy.wrapper import WtDataHelper
    dtHelper = WtDataHelper()
    if stdCode[-4:] == '.HOT':
        stdCode = stdCode[:-4] + "_HOT"
    else:
        ay = stdCode.split(".")
        if exchg == 'CZCE':
            stdCode = ay[1] + ay[2][1:]
        else:
            stdCode = ay[1] + ay[2]

    filename = f"../storage/his/{period}/{exchg}/"
    if not os.path.exists(filename):
        os.makedirs(filename)
    filename += f"{stdCode}.dsb"
    if period == "day":
        period = "d"
    elif period == "min1":
        period = "m1"
    else:
        period = "m5"
    dtHelper.store_bars(filename, firstBar, count, period)
    pass

hlper.dmpBars(codes=["CFFEX.IF.2103"], cb=on_bars_block, start_date=20201201, end_date=20210316, period="min5")
```
更多详细用法，可以参考`demo`: `wtpy/demos/test_datafactory`
