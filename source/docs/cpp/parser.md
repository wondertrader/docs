# 对接新的行情接口

### 行情接口简介
顾名思义，行情接口就是用于接入实时行情数据的接口。无论是什么量化框架，行情接入都是一个必不可少的步骤。为了适配不同的行情源，几乎所有的量化框架都会做一级抽象，然后根据不同的行情源，实现不同的行情接口。

**WonderTrader**作为以`C++`为核心的量化平台，为了最大化`C++`语言的速度优势，目前所有的行情源，都采用`C++`开发的行情接口模块来实现对接。

目前**WonderTrader**已经对接的行情源如下：

* 期货
    * CTP
    * CTPMini
    * 飞马Femas
    * 艾克朗科（仅组播行情）
    * 易达
* 期权
    * CTPOpt
* 股票
    * 中泰XTP
    * 华鑫奇点
    * 华锐ATP
    * 宽睿OES

虽然**WonderTrader**已经覆盖了市面上常见的行情源，但是总是有一些不常见的行情源会需要重新对接。这个时候最好的方式就是自己实现一个行情解析模块。

### 行情解析模块结构
**WonderTrader**抽象了一个行情解析器接口类`IParserApi`，定义如下：

```cpp
/*
 *	行情解析模块接口
 */
class IParserApi
{
public:
	/*
	 *	初始化解析模块
	 *	@config	模块配置
	 *	返回值	是否初始化成功
	 */
	virtual bool init(WTSVariant* config) { return false; }

	/*
	 *	释放解析模块
	 *	用于退出时
	 */
	virtual void release(){}

	/*
	 *	开始连接服务器
	 *	@返回值	连接命令是否发送成功
	 */
	virtual bool connect() { return false; }

	/*
	 *	断开连接
	 *	@返回值	命令是否发送成功
	 */
	virtual bool disconnect() { return false; }

	/*
	 *	是否已连接
	 *	@返回值	是否已连接
	 */
	virtual bool isConnected() { return false; }

	/*
	 *	订阅合约列表
	 */
	virtual void subscribe(const CodeSet& setCodes){}

	/*
	 *	退订合约列表
	 */
	virtual void unsubscribe(const CodeSet& setCodes){}

	/*
	 *	注册回调接口
	 */
	virtual void registerSpi(IParserSpi* spi) {}
};
```

为了解耦行情解析模块与调用模块，还定义了一个回调接口类`IParserSpi`，定义如下：

```cpp
/*
 *	行情解析模块回调接口
 */
class IParserSpi
{
public:
	/*
	 *	处理模块事件
	 *	@e	事件类型,如连接、断开、登录、登出
	 *	@ec	错误码,0为没有错误
	 */
	virtual void handleEvent(WTSParserEvent e, int32_t ec){}

	/*
	 *	处理合约列表
	 *	@aySymbols	合约列表,基础元素为WTSContractInfo,WTSArray的用法请参考定义
	 */
	virtual void handleSymbolList(const WTSArray* aySymbols)		= 0;

	/*
	 *	处理实时行情
	 *	@quote		实时行情
	 *	@procFlag	处理标记，0-切片行情，无需处理(ParserUDP)；1-完整快照，需要切片(国内各路通道)；2-极简快照，需要缓存累加（主要针对日线、tick，m1和m5都是自动累加的，虚拟货币行情）
	 */
	virtual void handleQuote(WTSTickData *quote, uint32_t procFlag)	= 0;

	/*
	 *	处理解析模块的日志
	 *	@ll			日志级别
	 *	@message	日志内容
	 */
	virtual void handleParserLog(WTSLogLevel ll, const char* message)	= 0;
};
```
行情解析模块就通过`IParserSpi`和调用者完成交互，具体可以参考`WtCore/ParserAdapter`的实现，代码片段如下：

```cpp
class ParserAdapter : public IParserSpi,
					private boost::noncopyable
{
    //这里是ParserAdapter的实现
}
```

### 行情解析模块调用流程
前面介绍了行情解析模块涉及到的几个接口的定义，在运行的时候行情解析模块的调用流程如下：

* 行情解析模块`ParserXXX.dll`加载
* 调用`createParser`接口创建一个`IParserApi`对象
* 向`IParserApi`注册`IParserSpi`对象指针
* 调用`IParserApi`->`init`传入行情解析模块配置的参数
* 调用`IParserApi`->`connect`开始连接行情源的服务端，并完成账户验证等流程
* 连接成功以后，触发`IParserSpi`->`handleEvent`告知调用者行情接口初始化完成
* 调用者收到初始化完成的事件通知以后，调用`IParserSpi`->`subscribe`订阅所需要的合约列表
* 订阅成功以后，开始进入持续的行情数据接收流程，收到`tick`数据以后，调用`IParserSpi`->`handleQuote`通知调用者
* 调用者收到`tick`行情以后，根据自身的业务逻辑进行处理

前面的流程介绍中，主要围绕`tick`数据展开，而像股票的`Level2`行情源，还涉及到*逐笔成交*、*逐笔委托*以及*委托队列*等高频数据。

不同的调用者，实现了`IParserSpi`的目的是不同的。比如`WtDtCore`作为数据组件的核心模块，派生了`IParserSpi`，收到`handleQuote`回调以后，就会触发`IDataWriter`进行数据的重采样以及落地文件等操作。而`WtCore`作为实盘交易的核心模块，收到`handleQuote`以后主要是向交易引擎的各个策略分发行情数据，驱动策略计算。

### 行情解析模块开发调试流程
前面已经将行情解析模块相关的内容都介绍了，最后我们来看一下，开发的具体流程：

* 首先根据目标策略复制一个工程，如`ParserCTP`，并将文件夹和工程名改成自己需要的工程名`ParserXXX`
* 然后，修改`Parser`类的类名为自己需要的类名，如将`ParserCTP`改成`ParserXXX`
* 第三，修改`ParserXXX`的代码，如引用目标行情源的头文件以及静态库等等，再修改行情解析部分的接口
* 第四，修改`sln`解决方案，将新的行情解析模块添加进去(`linux`下，需要修改`CMakeLists.txt`)
* 最后，编译该行情解析模块，生成最终的模块文件`ParserXXX.dll`(`linux`为`libParserXXX.so`)

经过以上步骤，生成的模块就可以进行调试了。可以使用`QuoteFactory`进行调用调试。模块调试稳定以后，就可以编译`release`的版本，放到各种环境下使用新的行情解析模块了。

有一些代码上的技巧，可以做一个分享：
* `Windows`下引用静态库`xxxx.lib`以后，在行情解析模块加载的时候，会到工作目录下去寻找对应的`xxxx.dll`，如果动态库不在工作目录下，则会导致行情解析模块加载失败。这个时候，可以采用两种方式来规避这种问题：
    * 在vs工程中配置延迟加载某个dll的编译指令`/DELAYLOAD:"xxxx.dll"`，然后在代码中显式调用LoadLibrary直接加载指定位置的xxxx.dll
    * 将行情源SDK提供的动态库全部复制到工作目录下

还有一种方式，就是不在编译的时候使用`xxxx.lib`文件，而采用隐式加载的方式。这种方式相对比较复杂，**WonderTrader**中很多`Parser`和`Trader`都采用这种方式来实现的，这个方式的好处就是不需要`xxxx.lib`文件，只需要头文件即可编译成功。缺点就是有一些门槛，不熟悉的人运用会比较难。

### ParserCTP简单讲解
首先是模块的接口部分，主要是`createParser`和`deleteParser`，代码如下：
```cpp
extern "C"
{
	EXPORT_FLAG IParserApi* createParser()
	{
		ParserCTP* parser = new ParserCTP();
		return parser;
	}

	EXPORT_FLAG void deleteParser(IParserApi* &parser)
	{
		if (NULL != parser)
		{
			delete parser;
			parser = NULL;
		}
	}
};
```

核心部分还是在如何处理行情数据的逻辑上，代码如下：
```cpp
void ParserCTP::OnRtnDepthMarketData( CThostFtdcDepthMarketDataField *pDepthMarketData )
{	
	if(m_pBaseDataMgr == NULL)
	{
		return;
	}

    WTSContractInfo* contract = m_pBaseDataMgr->getContract(pDepthMarketData->InstrumentID, pDepthMarketData->ExchangeID);
    if (contract == NULL)
        return;

    uint32_t actDate, actTime, actHour;

    if(m_bLocaltime)
    {
        TimeUtils::getDateTime(actDate, actTime);
        actHour = actTime / 10000000;
    }
    else
    {
        actDate = strtoul(pDepthMarketData->ActionDay, NULL, 10);
        actTime = strToTime(pDepthMarketData->UpdateTime) * 1000 + pDepthMarketData->UpdateMillisec;
        actHour = actTime / 10000000;

        if (actDate == m_uTradingDate && actHour >= 20) {
            //这样的时间是有问题,因为夜盘时发生日期不可能等于交易日
            //这就需要手动设置一下
            uint32_t curDate, curTime;
            TimeUtils::getDateTime(curDate, curTime);
            uint32_t curHour = curTime / 10000000;

            //早上启动以后,会收到昨晚12点以前收盘的行情,这个时候可能会有发生日期=交易日的情况出现
            //这笔数据直接丢掉
            if (curHour >= 3 && curHour < 9)
                return;

            actDate = curDate;

            if (actHour == 23 && curHour == 0) {
                //行情时间慢于系统时间
                actDate = TimeUtils::getNextDate(curDate, -1);
            } else if (actHour == 0 && curHour == 23) {
                //系统时间慢于行情时间
                actDate = TimeUtils::getNextDate(curDate, 1);
            }
        }
    }

	WTSCommodityInfo* pCommInfo = contract->getCommInfo();

	WTSTickData* tick = WTSTickData::create(pDepthMarketData->InstrumentID);
	tick->setContractInfo(contract);

	WTSTickStruct& quote = tick->getTickStruct();
	strcpy(quote.exchg, pCommInfo->getExchg());
	
	quote.action_date = actDate;
	quote.action_time = actTime;
	
	quote.price = checkValid(pDepthMarketData->LastPrice);
	quote.open = checkValid(pDepthMarketData->OpenPrice);
	quote.high = checkValid(pDepthMarketData->HighestPrice);
	quote.low = checkValid(pDepthMarketData->LowestPrice);
	quote.total_volume = pDepthMarketData->Volume;
	quote.trading_date = m_uTradingDate;
	if(pDepthMarketData->SettlementPrice != DBL_MAX)
		quote.settle_price = checkValid(pDepthMarketData->SettlementPrice);
	if(strcmp(quote.exchg, "CZCE") == 0)
	{
		quote.total_turnover = pDepthMarketData->Turnover*pCommInfo->getVolScale();
	}
	else
	{
		if(pDepthMarketData->Turnover != DBL_MAX)
			quote.total_turnover = pDepthMarketData->Turnover;
	}

	quote.open_interest = pDepthMarketData->OpenInterest;

	quote.upper_limit = checkValid(pDepthMarketData->UpperLimitPrice);
	quote.lower_limit = checkValid(pDepthMarketData->LowerLimitPrice);

	quote.pre_close = checkValid(pDepthMarketData->PreClosePrice);
	quote.pre_settle = checkValid(pDepthMarketData->PreSettlementPrice);
	quote.pre_interest = pDepthMarketData->PreOpenInterest;

	//委卖价格
	quote.ask_prices[0] = checkValid(pDepthMarketData->AskPrice1);
	quote.ask_prices[1] = checkValid(pDepthMarketData->AskPrice2);
	quote.ask_prices[2] = checkValid(pDepthMarketData->AskPrice3);
	quote.ask_prices[3] = checkValid(pDepthMarketData->AskPrice4);
	quote.ask_prices[4] = checkValid(pDepthMarketData->AskPrice5);

	//委买价格
	quote.bid_prices[0] = checkValid(pDepthMarketData->BidPrice1);
	quote.bid_prices[1] = checkValid(pDepthMarketData->BidPrice2);
	quote.bid_prices[2] = checkValid(pDepthMarketData->BidPrice3);
	quote.bid_prices[3] = checkValid(pDepthMarketData->BidPrice4);
	quote.bid_prices[4] = checkValid(pDepthMarketData->BidPrice5);

	//委卖量
	quote.ask_qty[0] = pDepthMarketData->AskVolume1;
	quote.ask_qty[1] = pDepthMarketData->AskVolume2;
	quote.ask_qty[2] = pDepthMarketData->AskVolume3;
	quote.ask_qty[3] = pDepthMarketData->AskVolume4;
	quote.ask_qty[4] = pDepthMarketData->AskVolume5;

	//委买量
	quote.bid_qty[0] = pDepthMarketData->BidVolume1;
	quote.bid_qty[1] = pDepthMarketData->BidVolume2;
	quote.bid_qty[2] = pDepthMarketData->BidVolume3;
	quote.bid_qty[3] = pDepthMarketData->BidVolume4;
	quote.bid_qty[4] = pDepthMarketData->BidVolume5;

	if(m_sink)
		m_sink->handleQuote(tick, 1);

	tick->release();
}
```

总的来说，自定义行情解析模块还是比较简单的，因为一旦连接建立成功，都是接收主推数据，单向推送给调用者，交互相对来说没有那么多。

### 行情解析模块的延伸
前面简单介绍了`C++`实现一个新的行情解析模块的大致流程，实际上在`wtpy`中还提供了`Python`版本的扩展行情解析器`ExtParser`，感兴趣的朋友也可以参考文档[ExtParser](../levelup/extpaser.md)。