# 经典日内CTA策略

#### 1.要实现的CTA策略简介

​	经典的CTA策略，有DualThrust策略、R-breaker策略、ATR策略、菲阿里四价策略、空中花园策略等。前两个策略已经有其他wtpy相关的文章进行过介绍，本文实现后面三个策略。

##### 1.1ATR策略

​	ATR（Average True Range）又称平均真实波动范围，主要是用来衡量市场波动的程度，是显示市场变化率的指标。

 ATR 平均真实波幅（ATR）的计算方法：

​	1.当前交易日的最高价与最低价间的波幅； 

​	2.前一交易日收盘价与当个交易日最高价间的波幅；

​	3.前一交易日收盘价与当个交易日最低价间的波幅。

 今日振幅、今日最高与昨收差价和今日最低与昨收差价中的最大值为真实波幅，在有了 真实波幅后，就可以利用一段时间的平均值计算 ATR 了。至于用多久计算，不同的使 用者习惯不同，10 天、20 天乃至 65 天都有。求真实波幅的 N 日移动平均，N 一般取 14 天，计算式如下所示： 

```
TR=Max(Max((High-Low),ABS(REF(Close,1)-High)),ABS(REF(Close,1)-Low)); ATR=TR 的 N 日移动平均。
```

  依据 ATR 的特性，可以设计通道突破策略，策略主要特点是：

1) 日内交易策略，收盘平仓；
2) 日内 ATR 突破基于当根 K 线开盘价与过去 N 个周期的 ATR；
3) 上轨＝当根 K 线开盘价 N 周期 ATR*M，下轨＝当根 K 线开盘价－N 周期 ATRxM；
4)  当价格突破上轨，买入开仓； 
5)  当价格跌穿下轨，卖出开仓。
6)  ATR 除了进场和出场策略外，还有止损止盈策略。由于 ATR 计算的是在某一个时间段内价格的真实波动范围，因此可以把该范围作为是计算止损和止盈的标准。

##### 1.2菲阿里四价策略

​	菲阿里四价策略是一种比较简单的趋势型日内交易策略。昨天高点、昨天低点、昨日收盘价、今天开盘价，可并称为菲阿里四价。

​	菲阿里四价是日内突破策略，所以每日收盘之前都需要进行平仓。

​	该策略的上下轨以及用法如下所示: 

​	上轨＝昨日高点；

​	下轨＝昨日低点； 

​	昨日高点和昨日低点可以视为近期的一个波动范围，该范围的存在一定程度是一种压力 线，只有足够的价格上涨或者下跌才会突破前期的高点或者低点。因此突破位置是一个比较好的入场信号，如果突破该波动范围，则证明动能较大，后续走势强度维持较强的概率比较高，因此该策略采用以下开仓方式: 

​	当价格突破上轨，买入开仓； 

​	当价格跌穿下轨，卖出开仓。

 策略在开仓之后可能面临假突破的问题，因为该价位存在很大的阻力，可能是暂时性的突破，随机回落，因此具体策略使用之中可以设置一些过滤条件来剔除假突破的情况。 这样使得策略的胜率变大。开仓之后的止损止盈根据具体环境具体确定。

##### 1.3空中花园策略

​	空中花园比较看重开盘突破。开盘时的高开或者低开均说明有大的利好或者利空使得开盘大幅远离昨天的收盘价。开盘突破，是最快的一种入场方式。当然出错的概率也最高。因此为了提高策略的胜率，空中花园策略加了额外的条件，也就是开盘要大幅高开或者低开，形成一个空窗，因此顾 名思义称为空中花园，然后再根据是否突破上下轨来进行开仓判断。这样一来，策略的胜率将大大提高，不过由于对高开或者低开的幅度要求过高，一般是超过1%，因此使得策略的交易次数可能相对其它策略而言要偏低一些。开盘第一根 K 线是收阳还是收阴，是判断日内趋势可能运动方向的标准。在当天开盘高开或低开时更有效。

​	空中花园策略主要特点：

​	日内交易策略，当日收盘平仓；

​	空中花园在当天高开或低开时使用，即当开盘价>=昨天收盘价*1.01 或开盘价<=昨天收盘价x0.99 时； 

​	上轨＝第一根 K 线的最高价；

​	下轨＝第一根 K 线的最低价； 

​	当价格突破上轨，买入开仓；

​	当价格跌穿下轨，卖出开仓。 

​	实际上是一种当天大幅高开（>1%），搏高开低走；反之，大幅低开（<1%）,博低开高走。

#### 2.策略实现

##### 	2.1 ATR策略的实现

​	根据上文中对于ATR策略的基本信息描述，我们设置当前bar之前的n根bar的收盘价平均为中轨，中轨加上K1倍的ATR为上轨，中轨减去K2倍的ATR为下轨，价格突破上轨，开仓做多，当持有多头仓位且价格回落到中轨时，平多；价格跌穿下轨时，开仓做空，当持有做空仓位时且价格回升到中轨时，平空。另外，收盘前平仓。

​	因为有日内交易策略收盘前平仓的要求，所以先介绍一下wtpy上实现收盘前平仓的方式：

```python
# 首先需要设定判断清仓时间区间的cleartimes，有夜盘和白盘需要清仓，格式如[[1455,1515],[2255,2300]]，因为回测使用股指期货，没有夜盘，所以只需使用14：55~15：15的区间
cleartimes = [[1455,1515]]  # 传入时间区间
self.__cleartimes__ = cleartimes  # 写在def __init__()中
# 以下写在 def on_calculate()中
curPos = context.stra_get_position(code)  # 获取当前持仓和价格
curTime = context.stra_get_time()

bCleared = False
for tmPair in self.__cleartimes__:
    if curTime >= tmPair[0] and curTime <= tmPair[1]:  # 当时间大于等于14：55时，对五分钟线数据来说，当日倒数第二根k线已经闭合，应在当日最后一根k线进行尾盘清仓
        if curPos != 0: # 如果持仓不为0，则要检查尾盘清仓
            context.stra_set_positions(code, 0, "clear") # 清仓直接设置仓位为0
            context.stra_log_text("尾盘清仓")
            bCleared = True
            break

            if bCleared:
                return
```

策略还需要计算均线和ATR，从而计算得出上轨和下轨：

```python
		# 参数说明：k1,k2是传入的上轨、下轨的系数，barN_atr是计算ATR时使用的bar根数，barN_ma是计算均线时使用的bar根数
    	TR_SUM = 0
        # 计算之前N个bar的ATR
        for i in range(barN_atr):
            TR_SUM += max(highs[-1-i]-lows[-1-i], highs[-1-i]-closes[-2-i], closes[-2-i]-lows[-1-i])
        ATR = TR_SUM / barN_atr
        # 计算中轨，为n根bar收盘价的均线
        ma = np.average(closes[-barN_ma:-1])  # closes是通过context.stra_get_bars接口拉取df_bars之后用closes = df_bars.closes 得到的收盘价序列
        up_price = ma + k1 * ATR
        down_price = ma - k2 * ATR
```

策略的交易逻辑部分代码如下：

```python
        if curPos == 0:
        	# 当价格突破上轨时做多，手数由当前总资金乘以参数money_pct然后除以一手所需保证金算出
            if curPrice > up_price:
                context.stra_enter_long(code, math.floor(cur_money * money_pct/trdUnit_price), 'enterlong')
                context.stra_log_text('当前价格:%.2f > 由atr计算出的上轨：%.2f，做多%s手' % (curPrice,
                                     up_price, math.floor(cur_money * money_pct/trdUnit_price)))
                return
            # 当价格跌破下轨时做空，手数同上
            if curPrice < down_price:
                context.stra_enter_short(code, math.floor(cur_money * money_pct/trdUnit_price), 'entershort')
                context.stra_log_text('当前价格:%.2f < 由atr计算出的下轨:%.2f,做空%s手'%(curPrice,
                                        down_price, math.floor(cur_money * money_pct/trdUnit_price)))
                return
        elif curPos != 0:
            # 当持有多仓且价格跌破均线时，平多
            if curPos > 0 and curPrice < ma:
                context.stra_set_position(code, 0, 'clear')
                context.stra_log_text('当前价格:%.2f < 由均线计算出的中轨:%.2f,平多' % (curPrice, ma,))
                return
            # 当持有空仓且价格突破均线时，平空
            elif curPos < 0 and curPrice > ma:
                context.stra_set_position(code, 0, 'clear')
                context.stra_log_text('当前价格:%.2f > 由均线计算出的中轨:%.2f,平空' % (curPrice, ma,))
                return
```

根据以上内容，ATR策略的大致内容已经完成得差不多了，完整代码放到文档最后，感兴趣的读者可以自己实现一下。

##### 2.2菲阿里四价策略的实现

​	根据上述对菲阿里策略的介绍，设定交易逻辑如下：

​	1.当前价高于昨日最高价且高于开盘价，做多；

​	2.当前价低于昨日最低价且低于开盘价，做空；

​	3.做多时价格低于今日开盘价，平多；

4. 做空时价格低于今日开盘价，平空。

​	作为日内策略，菲阿里四价策略同样也需要尾盘清仓，尾盘清仓的代码与上文中ATR策略中的完全相同。为了防止在交易边界策略频繁开仓平仓损失过大，设定每天只能开仓一次，下面说明一下在wtpy上如何实现每日开仓次数限制。

```python
# 在def __init__() 中加入self.today_entry
# 在def on_calculate()之前加入 def on_session_begin() (on_session_begin将会在每次新交易日开始时调用一次)
    def on_session_begin(self, context:CtaContext, curTDate:int):
        self.new_day = 1 # 用于判断当前是不是新交易日
        self.today_entry = 0 # 用于记录当天的进场次数
    # 写在 def on_calculate()中的部分
    if self.new_day == 1: # 当调用on_session_begin来得到self.new_day=1时，每日第一根k线已经闭合，所以可以用df_bar.xxx[-1]获取上一根k线的数据也就是当日第一根k线的数据
        self.cur_open = df_bars.opens[-1] # 记录新交易日的开盘价
        self.new_day = 0 # 记录完成后重置
    # 进场时的交易逻辑部分
    if curPos == 0 and self.today_entry == 0:
        #  满足进场条件后进场且记录今日入场
        if curPrice > last_day_high and curPrice > self.cur_open:
            context.stra_enter_long(code,math.floor(cur_money*money_pct/trdUnit_price),'enterlong')
            context.stra_log_text('当前价格：%.2f > 昨日最高价：%.2f，做多%s手' % (curPrice, last_day_high, \
                                                                    math.floor(cur_money*money_pct/trdUnit_price)))
            self.today_entry = 1
            return
        if curPrice < last_day_low and curPrice < self.cur_open:
            context.stra_enter_short(code,math.floor(cur_money*money_pct/trdUnit_price),'entershort')
            context.stra_log_text('当前价格:%.2f < 昨日最低价：%.2f，做空%s手' % (curPrice, last_day_low, \
                                                                    math.floor(cur_money*money_pct/trdUnit_price)))
            self.today_entry = 1
            return
        elif curPos != 0:
            if curPos < 0 and curPrice > self.cur_open:
                context.stra_set_position(code, 0, 'clear')
                context.stra_log_text('当前价格高于今日开盘价，止损，平空')
                return
            if curPos > 0 and curPrice < self.cur_open:
                context.stra_set_position(code, 0, 'clear')
                context.stra_log_text('当前价格低于今日开盘价，止损，平多')
                return
```

菲阿里策略总体交易逻辑还是比较简单，完整代码同样放在文章最后，感兴趣的读者可以自己实现一下。

##### 2.3空中花园策略的实现

根据上文中的介绍，制定空中花园策略的交易逻辑：

1.尾盘清仓；

2.开盘价>昨日收盘价*1.01且开盘价>当天第一根k线的最高价，做多；

3.开盘价<昨日收盘价*0.99且开盘价<当天第一根k线的最低价，做空；

4.每日限开仓一次。

核心代码：

```python
        if self.new_day == 1: # 获取当日开盘价以及第一根K线的最高价、最低价
            self.cur_open = df_bars.opens[-1]
            self.cur_high = df_bars.highs[-1]
            self.cur_low = df_bars.lows[-1]
            self.new_day = 0
        if curPos == 0 and self.today_entry == 0:
            if curPrice > self.cur_high and self.cur_open > last_day_close*1.01:
                context.stra_enter_long(code, math.floor(cur_money*money_pct/trdUnit_price),'enterlong')
                context.stra_log_text('当前价格：%.2f > 昨日收盘价：%.2f*1.01，做多%s手' % (curPrice, last_day_close, \
                                                                    math.floor(cur_money*money_pct/trdUnit_price)))
                self.today_entry = 1
                return
            if curPrice < self.cur_low and curPrice < last_day_close * 0.99:
                context.stra_enter_short(code, math.floor(cur_money*money_pct/trdUnit_price),'entershort')
                context.stra_log_text('当前价格:%.2f < 昨日收盘价：%.2f * 0.99，做空%s手' % (curPrice, self.cur_low, \
                                                                        math.floor(cur_money*money_pct/trdUnit_price)))
                self.today_entry = 1
                return
```

笔者根据互联网上的资料将高开、低开的比例比较死板地设置成了1.01与0.99，如果读者对这个策略比较感兴趣，也可以自己添加2个参数来控制高开、低开的比例来进行回测。

#### 3.策略回测

##### 3.1ATR策略的回测

选择2014年9月10日到2019年9月01日的股指期货进行回测，k1,k2都设定为0.5,使用的资金比率设定为20%，均线和ATR计算使用的K线条数都为10根，回测结果如图：

![](http://wt.f-sailors.cn/daytradecta/1.png)

根据回测内容来看，总体还是有一些收益，不过收益部分主要集中在2016年之前，后面的年份表现就比较一般了。可以用demos中的Optimizer来寻找更加合适的参数。

##### 3.2菲阿里策略的回测

选择2014年9月10日到2019年9月01日的股指期货进行回测,使用的资金比率设定为20%,回测结果：

![](http://wt.f-sailors.cn/daytradecta/2.png)

分析回测结果，总体收益率比较一般，菲阿里策略也是一种趋势突破型的策略，交易逻辑在于，如果价格突破昨日的最高价，则认为还有继续上涨的潜力，此时买入，而下个跌破昨日最低点时，则认为还有继续向下的动力，此时卖出开仓，以获取其中的收益。总体而言，菲阿里策略判断趋势成立的标准实在有些过于简单，目前的市场趋势判断不仅是靠昨日的最高价最低价就可以建立的，因此获得的收益率也是不太理想。

##### 3.3空中花园策略

选择2014年9月10日到2019年9月01日的股指期货进行回测,使用的资金比率设定为20%,回测结果：

![](http://wt.f-sailors.cn/daytradecta/3.png)

空中花园策略也是趋势突破型的策略，策略进场只考虑当日的开盘价与昨日的收盘价之间的关系，博取大幅高开或者大幅低开的行情，本次回测仅仅使用空中花园策略默认定义的高开或低开幅度1%，但是意外地表现出了较好的回测结果。不过2016年至2018年初之间策略表现一般，资金曲线较平，推测是因为这段时间之内股指波动相对较小，因此1%的高开、低开判定条件相比之下过于严格，没有触发开仓条件。感兴趣的读者可以将策略中的高开与低开比例作为参数，调整参数后再进行回测。

##### 4.完整策略代码

##### 4.1ATR策略

```python
from wtpy import BaseCtaStrategy
from wtpy import CtaContext
import numpy as np
import math


class ATRStra(BaseCtaStrategy):
    
    def __init__(self, name:str, code:str, barCnt:int, period:str, barN_atr:int, barN_ma:int,k1:float, \
                 k2:float, margin_rate:float, money_pct:float,capital, cleartimes:list):
        BaseCtaStrategy.__init__(self, name)

        self.__period__ = period  # 策略运行的时间区间
        self.__bar_cnt__ = barCnt   # 拉取的K线条数
        self.__code__ = code
        self.__margin_rate__ = margin_rate  # 保证金比率
        self.__money_pct__ = money_pct  # 每次使用的资金比率
        self.__cleartimes__ = cleartimes  # 尾盘清仓的时间区间
        self.__capital__ = capital  # 起始资金
        self.__k1__ = k1  # 上轨的系数
        self.__k2__ = k2  # 下轨的系数
        self.__barN_atr__ = barN_atr  # 计算atr时的k线根数
        self.__barN_ma__ = barN_ma   # 计算均线时的k线根数

    def on_init(self, context:CtaContext):
        code = self.__code__    #品种代码

        context.stra_get_bars(code, self.__period__, self.__bar_cnt__, isMain = True)
        context.stra_log_text("ATRStra inited")
        pInfo = context.stra_get_comminfo(code)
        self.__volscale__ = pInfo.volscale

    def on_calculate(self, context:CtaContext):
        code = self.__code__    #品种代码

        df_bars = context.stra_get_bars(code, self.__period__, self.__bar_cnt__, isMain = True)
        closes = df_bars.closes
        highs = df_bars.highs
        lows = df_bars.lows
        opens = df_bars.opens

        # 日内策略，尾盘清仓
        curPos = context.stra_get_position(code)
        curPrice = context.stra_get_price(code)
        curTime = context.stra_get_time()
        bCleared = False
        for tmPair in self.__cleartimes__:
            if curTime >= tmPair[0] and curTime <= tmPair[1]:
                if curPos != 0:  # 如果持仓不为0，则要检查尾盘清仓
                    context.stra_set_position(code, 0, "clear")  # 清仓直接设置仓位为0
                    context.stra_log_text("尾盘清仓")
                bCleared = True
                break

        if bCleared:
            return

        # 把策略参数读进来，作为临时变量，方便引用
        margin_rate = self.__margin_rate__
        k1 = self.__k1__
        k2 = self.__k2__
        barN_atr = self.__barN_atr__
        barN_ma = self.__barN_ma__
        money_pct = self.__money_pct__
        volscale = self.__volscale__
        capital = self.__capital__
        trdUnit_price = volscale * margin_rate * curPrice
        cur_money = capital + context.stra_get_fund_data(0)
        TR_SUM = 0
        # 计算之前N个bar的ATR
        for i in range(barN_atr):
            TR_SUM += max(highs[-1-i]-lows[-1-i], highs[-1-i]-closes[-2-i], closes[-2-i]-lows[-1-i])
        ATR = TR_SUM / barN_atr
        # 计算中轨，为n根bar收盘价的均线
        ma = np.average(closes[-barN_ma:-1])
        up_price = ma + k1 * ATR
        down_price = ma - k2 * ATR

        if curPos == 0:
            if curPrice > up_price:
                context.stra_enter_long(code, math.floor(cur_money * money_pct/trdUnit_price), 'enterlong')
                context.stra_log_text('当前价格:%.2f > 由atr计算出的上轨：%.2f，做多%s手' % (curPrice,
                                     up_price, math.floor(cur_money * money_pct/trdUnit_price)))
                return
            if curPrice < down_price:
                context.stra_enter_short(code, math.floor(cur_money * money_pct/trdUnit_price), 'entershort')
                context.stra_log_text('当前价格:%.2f < 由atr计算出的下轨:%.2f,做空%s手'%(curPrice,
                                        down_price, math.floor(cur_money * money_pct/trdUnit_price)))
                return
        elif curPos != 0:
            if curPos > 0 and curPrice < ma:
                context.stra_set_position(code, 0, 'clear')
                context.stra_log_text('当前价格:%.2f < 由均线计算出的中轨:%.2f,平多' % (curPrice, ma,))
                return
            elif curPos < 0 and curPrice > ma:
                context.stra_set_position(code, 0, 'clear')
                context.stra_log_text('当前价格:%.2f > 由均线计算出的中轨:%.2f,平空' % (curPrice, ma,))
                return

```

##### 4.2菲阿里策略

```python
from wtpy import BaseCtaStrategy
from wtpy import CtaContext
import numpy as np
import math

class FairyStra(BaseCtaStrategy):
    
    def __init__(self, name:str, code:str, barCnt:int, period:str, margin_rate:float, money_pct:float,capital, cleartimes:list):
        BaseCtaStrategy.__init__(self, name)

        self.__period__ = period
        self.__bar_cnt__ = barCnt
        self.__code__ = code
        self.__margin_rate__ = margin_rate # 保证金比率
        self.__money_pct__ = money_pct # 每次使用的资金比率
        self.__cleartimes__ = cleartimes # 尾盘清仓的时间区间
        self.__capital__ = capital
        self.today_entry = 0 # 限制每天开仓次数的参数

    def on_init(self, context:CtaContext):
        code = self.__code__    #品种代码

        context.stra_get_bars(code, 'd1', self.__bar_cnt__, isMain=False)
        context.stra_get_bars(code, self.__period__, self.__bar_cnt__, isMain = True)
        context.stra_log_text("FairyStra inited")
        pInfo = context.stra_get_comminfo(code)
        self.__volscale__ = pInfo.volscale


    def on_session_begin(self, context:CtaContext, curTDate:int):
        self.new_day = 1
        self.today_entry = 0

    def on_calculate(self, context:CtaContext):
        code = self.__code__    #品种代码

        trdUnit = 1

        theCode = code
        df_bars = context.stra_get_bars(code,'d1', self.__bar_cnt__, isMain=False)
        closes = df_bars.closes
        highs = df_bars.highs
        lows = df_bars.lows
        opens = df_bars.opens
        # 获取菲阿里四价：昨日开盘价、最高价、最低价、收盘价 (但是本策略没有用到昨日的close和open，互联网上的例子也没有)
        last_day_high = highs[-1]
        last_day_low = lows[-1]
        last_day_close = closes[-1]
        last_day_open = opens[-1]
        df_bars = context.stra_get_bars(theCode, self.__period__, self.__bar_cnt__, isMain = True)
        # 尾盘清仓
        curPos = context.stra_get_position(code)
        curPrice = context.stra_get_price(code)
        curTime = context.stra_get_time()
        bCleared = False
        for tmPair in self.__cleartimes__:
            if curTime >= tmPair[0] and curTime <= tmPair[1]:
                if curPos != 0:  # 如果持仓不为0，则要检查尾盘清仓
                    context.stra_set_position(code, 0, "clear")  # 清仓直接设置仓位为0
                    context.stra_log_text("尾盘清仓")
                bCleared = True
                break

        if bCleared:
            return
        if self.new_day == 1:
            self.cur_open = df_bars.opens[-1]
            self.new_day = 0

        #把策略参数读进来，作为临时变量，方便引用
        margin_rate = self.__margin_rate__
        money_pct = self.__money_pct__
        volscale = self.__volscale__
        capital = self.__capital__
        trdUnit_price = volscale * margin_rate * curPrice
        cur_money = capital + context.stra_get_fund_data(0)
        curPos = context.stra_get_position(code)/trdUnit

        if curPos == 0 and self.today_entry == 0:
            if curPrice > last_day_high and curPrice > self.cur_open:
                context.stra_enter_long(code,math.floor(cur_money*money_pct/trdUnit_price),'enterlong')
                context.stra_log_text('当前价格：%.2f > 昨日最高价：%.2f，做多%s手' % (curPrice, last_day_high, \
                                                                    math.floor(cur_money*money_pct/trdUnit_price)))
                self.today_entry = 1
                return
            if curPrice < last_day_low and curPrice < self.cur_open:
                context.stra_enter_short(code,math.floor(cur_money*money_pct/trdUnit_price),'entershort')
                context.stra_log_text('当前价格:%.2f < 昨日最低价：%.2f，做空%s手' % (curPrice, last_day_low, \
                                                                        math.floor(cur_money*money_pct/trdUnit_price)))
                self.today_entry = 1
                return
        elif curPos != 0:
            if curPos < 0 and curPrice > self.cur_open:
                context.stra_set_position(code, 0, 'clear')
                context.stra_log_text('当前价格高于今日开盘价，止损，平空')
                return
            if curPos > 0 and curPrice < self.cur_open:
                context.stra_set_position(code, 0, 'clear')
                context.stra_log_text('当前价格低于今日开盘价，止损，平多')
                return

```

##### 4.3空中花园策略

```python
import pandas as pd
from wtpy import BaseCtaStrategy
from wtpy import CtaContext
import numpy as np
import math

class SkyGardenStra(BaseCtaStrategy):
    
    def __init__(self, name:str, code:str, barCnt:int,
                 period:str, margin_rate:float, money_pct:float, capital, cleartimes:list):
        BaseCtaStrategy.__init__(self, name)

        self.__period__ = period
        self.__bar_cnt__ = barCnt
        self.__code__ = code
        self.__margin_rate__ = margin_rate # 保证金比率
        self.__money_pct__ = money_pct # 每次使用的资金比率
        self.__capital__ = capital
        self.today_entry = 0 # 限制每天开仓次数的参数
        self.__cleartimes__=cleartimes

    def on_init(self, context:CtaContext):
        code = self.__code__    #品种代码

        context.stra_get_bars(code, 'd1', self.__bar_cnt__, isMain=False)
        context.stra_get_bars(code, self.__period__, self.__bar_cnt__, isMain = True)
        context.stra_log_text("SkyGardenStra inited")
        pInfo = context.stra_get_comminfo(code)
        self.__volscale__ = pInfo.volscale


    def on_session_begin(self, context:CtaContext, curTDate:int):
        self.new_day = 1

        self.today_entry = 0

    def on_calculate(self, context:CtaContext):
        code = self.__code__    #品种代码

        trdUnit = 1

        theCode = code
        df_bars = context.stra_get_bars(code,'d1', self.__bar_cnt__, isMain=False)
        closes = df_bars.closes

        # 获取昨日收盘价
        last_day_close = closes[-1]
        df_bars = context.stra_get_bars(theCode, self.__period__, self.__bar_cnt__, isMain = True)
        # 尾盘清仓
        curPos = context.stra_get_position(code)
        curPrice = context.stra_get_price(code)
        curTime = context.stra_get_time()
        bCleared = False
        for tmPair in self.__cleartimes__:
            if curTime >= tmPair[0] and curTime <= tmPair[1]:
                if curPos != 0:  # 如果持仓不为0，则要检查尾盘清仓
                    context.stra_set_position(code, 0, "clear")  # 清仓直接设置仓位为0
                    context.stra_log_text("尾盘清仓")
                bCleared = True
                break

        if bCleared:
            return
        if self.new_day == 1:
            self.cur_open = df_bars.opens[-1]
            self.cur_high = df_bars.highs[-1]
            self.cur_low = df_bars.lows[-1]
            self.new_day = 0

        #把策略参数读进来，作为临时变量，方便引用
        margin_rate = self.__margin_rate__
        money_pct = self.__money_pct__
        volscale = self.__volscale__
        capital = self.__capital__
        trdUnit_price = volscale * margin_rate * curPrice
        cur_money = capital + context.stra_get_fund_data(0)
        curPos = context.stra_get_position(code)/trdUnit

        if curPos == 0 and self.today_entry == 0:
            if curPrice > self.cur_high and self.cur_open > last_day_close*1.01:
                context.stra_enter_long(code, math.floor(cur_money*money_pct/trdUnit_price),'enterlong')
                context.stra_log_text('当前价格：%.2f > 昨日收盘价：%.2f*1.01，做多%s手' % (curPrice, last_day_close, \
                                                                    math.floor(cur_money*money_pct/trdUnit_price)))
                self.today_entry = 1
                return
            if curPrice < self.cur_low and curPrice < last_day_close * 0.99:
                context.stra_enter_short(code, math.floor(cur_money*money_pct/trdUnit_price),'entershort')
                context.stra_log_text('当前价格:%.2f < 昨日收盘价：%.2f * 0.99，做空%s手' % (curPrice, self.cur_low, \
                                                                        math.floor(cur_money*money_pct/trdUnit_price)))
                self.today_entry = 1
                return
```

将策略代码的.py文件放入wtpy/demos/cta_fut_bt/strategies，然后修改runBT文件，传入相应的参数，即可自己进行策略的回测。至此本文关于几个经典cta日间策略的分享就结束了，后续笔者还会分享一些经典的cta跨日策略的编写过程。
