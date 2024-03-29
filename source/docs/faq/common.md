# 通用常见问题


### on_bar和on_schedule（on_calculate）有什么区别
答：
K线闭合的时候会触发on_bar，on_schedule（on_calculate）只在CTA引擎和SEL引擎下会触发。对于CTA引擎来说，主K线闭合并且其他已订阅的K线都正常闭合，才会触发on_schedule；对于SEL引擎来说，当设定的重算时间到了才会触发on_schedule。on_schedule触发前，底层都会确保应该闭合的K线已经触发了on_bar

### WonderTrader中标的代码规则
答：
WonderTrader内部采用统一的合约代码标准：
* 期货合约为“交易所.品种.月份”，如CFFEX.IF.2306，郑商所也同样需要扩充到4位月份
* 商品期权合约为“交易所.品种月份.方向.行权价”，如CFFEX.IO2007.C.4000，郑商所也同样需要扩种到4位月份
* 股票代码为如SSE.STK.600000或者SZSE.STK.000001
* 股票指数代码SSE.IDX.000001
* ETF代码SSE.ETF.510050
* ETF期权代码SSE.ETFO.10003961

### 如何选择交易引擎
答：
WonderTrader一共有四种交易引擎：
* CTA引擎，主要针对CTA策略的需求来设计，主要适用于少量标的（单策略50个标的以内）时序策略的引擎，采用M+1+N执行架构，信号执行剥离，可以配置多路执行器实现多账户统一交易
* SEL引擎，和CTA引擎类似，也采用M+1+N执行架构，支持多账户统一交易。底层采用异步驱动机制，适合大量计算的场景（策略计算时长超过1分钟）
* HFT引擎，高频交易引擎，支持wtpy开发策略，适用于高频交易场景
* UFT引擎，极速交易引擎，只支持C++开发策略，适用于超高频超低延迟场景

### 如何理解策略组合
答：
在WonderTrader中，无论使用什么引擎，都绕不开策略组合的概念。组合，全称是策略组合，顾名思义，就是若干个策略形成的一组策略。

对于HFT引擎和UFT引擎来说，组合就只是单纯的将策略统一管理起来。

而对于CTA引擎和SEL引擎来说，组合层面会有一层持仓、资金、成交的概念。组合的成交遵循以下流程：
* 首先不同策略生成自己的目标头寸
* 每分钟组合会汇总每个子策略的目标头寸，并将相反的头寸做一个轧平，得到组合的净头寸
* 然后将组合的净头寸，提交给执行器来执行

从形式上来说，一个交易进程中有且只有一个组合。组合在代码上存在的形式，可以等同于engine，也就是说一个engine就是一个组合。

从功能上来说，尤其对于CTA引擎和SEL引擎，组合延伸了持仓、成交、资金等概念，还可以基于组合做一层理论上的风控。

这里要注意区分一下：多个子策略并不是共享了组合的持仓数据，而是组合汇总了子策略的持仓数据。


### 如何使用openctp
答：
openctp主要提供了行情接口和交易接口，要使用openctp，主要是修改parsers和traders的配置。

openctp的环境，可以参考此页面： <http://121.37.80.177:50080/detail.html>

parsers的配置可以参考以下配置：
```yaml
parsers:
-   active: true
    broker: '9999'
    code: ''
    front: tcp://210.14.72.14:4402
    id: parser
    module: ParserCTP
    pass: test
    user: test
    # 如果要使用openctp，启用该配置项即可
    ctpmodule: tts_thostmduserapi_se

```
traders的配置可以参考以下配置：
```yaml
traders:
-   active: true
    id: openctp          # id
    module: TraderCTP   # 模块文件名，win下会自动转成xxx.dll，linux会自动转成libxxx.so
    savedata: true      # 是否保存数据，如果为true，会将接口拉取到的成交、订单、资金和持仓都写到本地文件中
    front: tcp://210.14.72.14:4400  #VIP
    broker: '9000'
    appid: empty
    authcode: empty    
    pass: test
    user: test
    quick: true                       # 是否订阅快速私有流，如果为true，则不会接受上次之前的私有流，这个一定要为true！！！
    ctpmodule: tts_thosttraderapi_se    # ctp模块名，如果需要使用其他仿制CTP模块，使用该配置项直接将仿制的CTP模块传给TraderCTP即可
```

### CTA仿真（实盘）配置流程指引
答：
详细流程可以参考文档[实盘完整攻略](../usage/product)。这里以期货CTA为例，梳理一下流程步骤：
* 1、配置CTPLoader，并拉取最新的合约列表和品种列表，参考demo: [wtpy/demos/ctp_loader](https://gitee.com/wondertrader/wtpy/tree/master/demos/ctp_loader)
* 2、配置HotPicker，并拉取最新的主力换月规则，参考demo: [wtpy/demos/test_hotpicker](https://gitee.com/wondertrader/wtpy/tree/master/demos/test_hotpicker)。（如果不需要使用主力合约如CFFEX.IF.HOT，可以跳过该步骤）
* 3、配置datakit并运行，参考demo: [datakit_fut](https://gitee.com/wondertrader/wtpy/tree/master/demos/datakit_fut)
* 4、配置交易组合，运行策略，参考demo: [wtpy/demos/cta_fut](https://gitee.com/wondertrader/wtpy/tree/master/demos/cta_fut)

注意事项：
* 1、datakit的配置文件dtcfg.yaml中的broadcaster配置的端口必须要和交易组合中的tdparsers.yaml中的端口一致
* 2、交易组合中config.yaml配置的storage的数据目录必须要和datakit中的一致

### 如何理解CTA引擎中的下单接口
答：
CTA引擎的下单接口，汇总起来包含以下几个接口：
```python     
def stra_enter_long(self, stdCode:str, qty:float, usertag:str = "", limitprice:float = 0.0, stopprice:float = 0.0):

def stra_exit_long(self, stdCode:str, qty:float, usertag:str = "", limitprice:float = 0.0, stopprice:float = 0.0):

def stra_enter_short(self, stdCode:str, qty:float, usertag:str = "", limitprice:float = 0.0, stopprice:float = 0.0):

def stra_exit_short(self, stdCode:str, qty:float, usertag:str = "", limitprice:float = 0.0, stopprice:float = 0.0):
```
四个接口分别开多、平多、开空、平空四种操作。接口的实现，接口用法和MultiCharts的buy/sell/sellShort/butToCover相同，可以概括如下：
* 开多N手——当持仓大于等于0时，则直接开多N手；当持仓小于0（持有空头）时，则先平掉空头持仓，再开多N手
* 平多N手——当持仓大于等于N时，则平掉N手多头；当持仓小于N时，则平掉全部多头持仓
* 开空N手——当持仓小于等于0（持有空头）时，则直接开空N手；当持仓大于0时，则先平掉多头持仓，再开空N手
* 平空N手——当空持仓大于等于N时，则平掉N手空头；当空头持仓小于N时，则平掉全部空头持仓

如果设置了limitprice和stopprice，则下单指令变成了条件单。所谓条件单，即需要满足条件才会触发信号。CTA引擎的条件单，每次on_calc调用之前，都会清除掉之前设置的条件单，即条件单的声明周期仅在下一根主K线的bar闭合之前。这样的机制，可以将策略简化到只需要实现on_calc就可以了，这个机制也是参考了MultiCharts来实现的。

限价limitprice和止价stopprice从某种角度来说，更像是交易信号的二次确认。以买入（开多或者平空）为例，当最新价小于等于limitprice，或者当最新价大于等于stopprice，才会触发买入信号。如果一个条件单同时设置了limitprice和stopprice，只有limitprice会生效。