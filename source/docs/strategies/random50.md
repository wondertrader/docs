## 随机选股策略

#### 1.随机50交易法介绍

&emsp;&emsp;沪深300指数的成分股每年调整2次（每半年调整一次），一般是1月份和7月份进行调整，每次调整比例不超过10%。每日随机选取50只沪深300的成分股股票作为股票池等权重买入，在第二日重新随机选取50只沪深300成分股股票作为新的股票池。卖出不在新股票池内的股票，买入新股票池中的股票，若新股票池当中的股票已经买入则继续持有，保证每日固定持有50只股票。重复该过程。

#### 2.策略实现

- 参数说明

```python
self.__capital__ = capital # 起始资金
self.__period__ = period # 交易k线的时间级，如5分钟，1天
self.__bar_cnt__ = barCnt # 拉取的bar的次数
self.__codes__ = codes # 所有沪深300成分股股票代码
self.__codes2__ = [] #每日50股票池
```
&emsp;&emsp;若新股票池的股票已经买入，则继续持有，并对股票池进行更新。

- 策略框架介绍

```python

class BaseSTKStrategy:
    '''
    STK策略基础类，所有的策略都从该类派生\n
    包含了策略的基本开发框架
    '''
    def __init__(self, name):
        self.__name__ = name

    def on_init(self, context:STKContext):
        '''
        策略初始化，启动的时候调用\n
        用于加载自定义数据\n
        @context    策略运行上下文
        '''
        return
        
    def on_calculate(self, context:Context):
        '''
        策略主调函数，所有的计算逻辑都在这里完成
        '''
        return
```

**策略实现具体思路：**

第一步，由于沪深300成分股会发生变化，每日都要重新读取沪深300成分股的名单

第二步，生成50个不重复的随机数，确定当日的股票池，使用初始资金等权重买入

第三步，第二个交易日重新生成随机的新股票池，对股票池的持仓进行更新。保证每日持有50支股票。

- 最终完整策略代码如下

```python
from wtpy import BaseCtaStrategy
from wtpy import CtaContext
import random
import pandas as pd

class StraHushenStk(BaseCtaStrategy):
    def __init__(self, name:str, codes:list, capital:float, barCnt:int, period:str):
        BaseCtaStrategy.__init__(self, name)

		self.__capital__ = capital # 起始资金
		self.__period__ = period # 交易k线的时间级，如5分钟，1天
		self.__bar_cnt__ = barCnt # 拉取的bar的次数
		self.__codes__ = codes # 所有沪深300成分股股票代码
		self.__codes2__ = [] # 每日50股票池

    def on_init(self, context:CtaContext):
        codes = self.__codes__    # 所有沪深300成分股股票代码
        codes2 = self.__codes2__ # 每日50股票池

        for i in range(0,len(codes)):
            if i == 0:
                context.stra_get_bars(codes[i], self.__period__, self.__bar_cnt__, isMain=True) # 设置第一支股票为主要品种
            else:
                context.stra_get_bars(codes[i], self.__period__, self.__bar_cnt__, isMain=False)

        context.stra_log_text("Hushen inited")

    
    def on_calculate(self, context:CtaContext):
        codes = self.__codes__    # 所有沪深300成分股股票代码
        codes2 = self.__codes2__  # 每日50股票池
        capital = self.__capital__ # 初始资金
        date = context.stra_get_date() # 获取当前日期

        #读取当日对应的沪深300成分股代码
        stocks = pd.read_csv('E:/沪深300历年成分股/{}.csv'.format(date), index_col=0)['order_book_id'].values 

        nums = [] # 股票池的数量
        while len(nums) < 50: # 每日固定生成50个不重复的随机数
            num = random.randint(0, len(stocks)-1)
            if num in nums: # 若随机数重复，跳过
                pass
            elif num not in nums: # 确保保存的随机数不重复
                stock = stocks[num]
                if stock[-1] == 'E': # 将深市股票代码转化为wtpy识别的股票代码
                    code = "SZSE.STK.{}".format(stock[:6])
                    curPrice = context.stra_get_price(code)
                    if curPrice > 0: # 确保股票当日价格数据正常
                        codes2.append(code) # 将股票放入股票池
                        nums.append(num) # 将随机数放入nums
                elif stock[-1] == 'G': # 将沪市股票代码转化为wtpy识别的股票代码
                    code = "SSE.STK.{}".format(stock[:6])
                    curPrice = context.stra_get_price(code)
                    if curPrice > 0: # 确保股票当日价格数据正常
                        codes2.append(code) # 将股票放入股票池
                        nums.append(num) # 将随机数放入nums

        for code in list(context.stra_get_all_position().keys()): # 遍历当前的持仓
            curPos = context.stra_get_position(code) # 获取每只股票的持仓头寸
            if code not in codes2 and curPos > 0: # 若当前持有的股票不在新生成的股票池内
                context.stra_exit_long(code, curPos, 'exitlong') # 平仓

        for code in codes2: # 遍历股票池
            curPos = context.stra_get_position(code) # 获取当前仓位
            curPrice = context.stra_get_price(code) # 获取当前价格
            if curPos == 0: # 若尚未持仓，买入该股票
                unit = int((200000/curPrice)/100) # 确定买入的数量
                context.stra_enter_long(code, unit, 'enterlong') # 买入unit手code
            elif curPos > 0: # 若已经持有该股票，则继续保留
                pass
        self.__codes2__ = [] # 当天交易结束后，将股票池清空

```

&emsp;&emsp;写好了策略主体，将策略主体代码命名为Hushen300.py放入wtpy/demos/cta_stk_bt/strategies目录下，然后我们还需要runBT.py来让策略跑起来，runBT.py的格式和代码如下：

```python
from wtpy import WtBtEngine,EngineType
from Strategies.Hushen import StraHushenStk
from wtpy.apps import WtBtAnalyst
import os
path = "E:/wtpy/deploy/storage/hushen300/csv"
datanames = os.listdir(path)
files = [string.replace('.csv','') for string in datanames]
files = [string.replace('_d','') for string in files]

if __name__ == "__main__":
    #创建一个运行环境，并加入策略
    engine = WtBtEngine(EngineType.ET_CTA)
    engine.init(folder='../common/', cfgfile="configbt.yaml", commfile="stk_comms.json", contractfile="stocks.json")
    engine.configBacktest(201101010930,202210241500)
    engine.configBTStorage(mode="csv", path="../storage/hushen300")
    engine.commitBTConfig()
    straInfo = StraHushenStk(name='Hushen', codes=files, barCnt=10, period="d", capital=10000000)
    engine.set_cta_strategy(straInfo)

    engine.run_backtest()

    #绩效分析
    analyst = WtBtAnalyst()
    analyst.add_strategy("Hushen", folder="./outputs_bt", init_capital=10000000, rf=0.02, annual_trading_days=240)
    analyst.run()

    kw = input('press any key to exit\n')
    engine.release_backtest()
```

&emsp;&emsp;将runBT.py放入wtpy/demos/cta_stk_bt运行即可，其中的参数可以自行修改替换。这次是使用的沪深300成分股数据来进行回测，回测需要的日线数据可以通过下面的链接获取，然后放入wtpy/demos/storage/hushen300/csv文件夹中。

https://pan.baidu.com/s/12WJd3tMRBdOgsDXiiBA_sg?pwd=wtpy 


#### 3.回测结果

&emsp;&emsp;打开运行之后生成的xlsx文件，可以看到策略的绩效，如下图所示：

- 回测绩效

![](https://raw.githubusercontent.com/leoyinhaiqing/images/main/img/1666854625408.png)