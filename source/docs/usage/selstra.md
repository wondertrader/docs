# SEL策略编写及回测

## 多引擎设计
WonderTrader采用多引擎设计，针对不同类型的策略提供独立的引擎，目前支持的引擎包括：
    
- **CTA引擎**，也叫**同步策略引擎**，一般适用于标的较少，计算逻辑较快的策略，事件+时间驱动。典型的应用场景包括单标的择时、中频以下的套利等。Demo中提供的DualThrust策略，单次重算平均耗时，Python实现版本约70多微秒，C++实现版本约4.5微秒。
![](../images/cta.jpg)
- **SEL引擎**，也叫**异步策略引擎**，一般适用于标的较多，计算逻辑耗时较长的策略，时间驱动。典型应用场景包括多因子选股策略、截面多空策略等。
- **HFT引擎**，也叫**高频策略引擎**，主要针对高频或者低延时策略，事件驱动，**系统延迟在1微秒之内**
- **UFT引擎**，也叫**极速策略引擎**，主要针对超高频或者超低延时策略，事件驱动，**系统延迟在200纳秒之内**
![](../images/hft.jpg)


## SEL策略概述

要基于WonderTrader实现SEL策略，有两种方式：
- 基于wtcpp，直接使用纯C++开发策略。这种方式适合有较高C++编程能力的用户
- 基于wtpy，使用python开发策略。这种方式适合对C++不大熟悉的用户

无论哪种方式，底层都是纯C++实现，执行效率有保障。
SEL引擎和CTA引擎的相同点：
- SEL引擎和CTA引擎都采用**M+1+N的执行架构**，信号和执行完全剥离，支持同一个组合同时向多个交易通道执行

SEL引擎和CTA引擎的不同点：
- SEL引擎同时支持交易时间和非交易时间驱动on_calculate，而CTA引擎只支持交易时间驱动on_calculate
- SEL引擎采用异步事件驱动的机制，而CTA引擎采用同步事件驱动的机制。如在某个时间节点T0，CTA引擎会把数据阻塞，直到所有策略的回调执行完成，再处理下一笔数据；而SEL引擎则不会阻塞数据，但是策略获取的数据，都是以T0时间为截止时间的。

通过上面可以对比可以发现：CTA引擎和SEL引擎在一定范围内是可以互相替换的，但是一旦组合的计算量变大，阻塞数据会导致很大的延迟从而影响理论撮合价格，那么SEL引擎就相对更合适一些。所以SEL引擎适用的策略场景大致有：
- 实时alpha选股
- 截面策略
- 其他计算量很大的策略

## SEL策略接口

和CTA策略一样，SEL策略也有一个基础结构：
```python
class BaseSelStrategy:
    '''
    选股策略基础类，所有的多因子策略都从该类派生
    包含了策略的基本开发框架
    '''
    def __init__(self, name:str):
        self.__name__ = name
        
    
    def name(self) -> str:
        return self.__name__


    def on_init(self, context:SelContext):
        '''
        策略初始化，启动的时候调用
        用于加载自定义数据

        @context    策略运行上下文
        '''
        return
    
    def on_session_begin(self, context:SelContext, curTDate:int):
        '''
        交易日开始事件

        @curTDate   交易日，格式为20210220
        '''
        return

    def on_session_end(self, context:SelContext, curTDate:int):
        '''
        交易日结束事件

        @curTDate   交易日，格式为20210220
        '''
        return
    
    def on_calculate(self, context:SelContext):
        '''
        K线闭合时调用，一般作为策略的核心计算模块
        @context    策略运行上下文
        '''
        return

    def on_calculate_done(self, context:SelContext):
        '''
        K线闭合时调用，一般作为策略的核心计算模块
        @context    策略运行上下文
        '''
        return

    def on_backtest_end(self, context:CtaContext):
        '''
        回测结束时回调，只在回测框架下会触发

        @context    策略上下文
        '''
        return

    def on_tick(self, context:SelContext, stdCode:str, newTick:dict):
        '''
        tick数据进来时调用
        生产环境中，每笔行情进来就直接调用
        回测环境中，是模拟的逐笔数据
        @context    策略运行上下文
        @stdCode    合约代码
        @newTick    最新逐笔
        '''
        return

    def on_bar(self, context:SelContext, stdCode:str, period:str, newBar:dict):
        '''
        K线闭合时回调
        @context    策略上下文
        @stdCode    合约代码
        @period     K线周期
        @newBar     最新闭合的K线
        '''
        return
```

SelContext提供了相应的各种接口，更多信息可以参考API相关文档的介绍。

## 一个简单的SEL策略
wtpy的demo中提供了一个简单的SEL策略，还是基于DualThrust做一个SEL的实现。

```py
from wtpy import BaseSelStrategy
from wtpy import SelContext
import numpy as np


class StraDualThrustSel(BaseSelStrategy):
    def __init__(self, name, codes:list, barCnt:int, period:str, days:int, k1:float, k2:float, isForStk:bool = False):
        BaseSelStrategy.__init__(self, name)

        self.__days__ = days
        self.__k1__ = k1
        self.__k2__ = k2

        self.__period__ = period
        self.__bar_cnt__ = barCnt
        self.__codes__ = codes

        self.__is_stk__ = isForStk
    

    def on_init(self, context:SelContext):
        return

    
    def on_calculate(self, context:SelContext):
        curTime = context.stra_get_time()
        trdUnit = 1
        if self.__is_stk__:
            trdUnit = 100

        for code in self.__codes__:
            sInfo = context.stra_get_sessioninfo(code)
            if not sInfo.isInTradingTime(curTime):
                continue

             #读取最近50条1分钟线(dataframe对象)
            theCode = code
            if self.__is_stk__:
                theCode = theCode + "Q"
            df_bars = context.stra_get_bars(theCode, self.__period__, self.__bar_cnt__)

            #把策略参数读进来，作为临时变量，方便引用
            days = self.__days__
            k1 = self.__k1__
            k2 = self.__k2__

            #平仓价序列、最高价序列、最低价序列
            closes = df_bars.closes
            highs = df_bars.highs
            lows = df_bars.lows

            #读取days天之前到上一个交易日位置的数据
            hh = np.amax(highs[-days:-1])
            hc = np.amax(closes[-days:-1])
            ll = np.amin(lows[-days:-1])
            lc = np.amin(closes[-days:-1])

            #读取今天的开盘价、最高价和最低价
            # lastBar = df_bars.get_last_bar()
            openpx = df_bars.opens[-1]
            highpx = df_bars.highs[-1]
            lowpx = df_bars.lows[-1]

            '''
            !!!!!这里是重点
            1、首先根据最后一条K线的时间，计算当前的日期
            2、根据当前的日期，对日线进行切片,并截取所需条数
            3、最后在最终切片内计算所需数据
            '''

            #确定上轨和下轨
            upper_bound = openpx + k1* max(hh-lc,hc-ll)
            lower_bound = openpx - k2* max(hh-lc,hc-ll)

            #读取当前仓位
            curPos = context.stra_get_position(code)/trdUnit

            if curPos == 0:
                if highpx >= upper_bound:
                    context.stra_set_position(code, 1*trdUnit, 'enterlong')
                    context.stra_log_text("{} 向上突破{}>={}，多仓进场".format(code, highpx, upper_bound))
                    continue

                if lowpx <= lower_bound and not self.__is_stk__:
                    context.stra_set_position(code, -1*trdUnit, 'entershort')
                    context.stra_log_text("{} 向下突破{}<={}，空仓进场".format(code, lowpx, lower_bound))
                    continue
            elif curPos > 0:
                if lowpx <= lower_bound:
                    context.stra_set_position(code, 0, 'exitlong')
                    context.stra_log_text("{} 向下突破{}<={}，多仓出场".format(code, lowpx, lower_bound))
                    #raise Exception("except on purpose")
                    continue
            else:
                if highpx >= upper_bound and not self.__is_stk__:
                    context.stra_set_position(code, 0, 'exitshort')
                    context.stra_log_text("{} 向上突破{}>={}，空仓出场".format(code, highpx, upper_bound))
                    continue


    def on_tick(self, context:SelContext, code:str, newTick:dict):
        return

    def on_bar(self, context:SelContext, code:str, period:str, newBar:dict):
        return
```
更多SEL相关信息可以参考wtpy/demos/sel_fut_bt。
