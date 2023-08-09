# 实盘完整攻略

本文将完整的介绍一下，如何搭建一个可以直接用于生产环境的完整的流程。
本文将以配置股票仿真盘为例，使用`DualThrust`策略，通过XTP行情通道落地数据，并通过XTP仿真交易通道来进行仿真交易。

### 实盘运行步骤：
---
1. 根据策略配置文件
2. 终端运行`datakit`组件`runDT.py`，一般在开盘前启动，如9:20，`datakit`负责在实盘中录制实时行情数据，存储在指定的数据目录中，同时通知策略进行接收
3. 另一个终端运行实盘组件`run.py`

### 文件配置
---
```
common  # 基础文件
    stk_comms.json  # 品种列表（交易所、品种类型、品种信息）
    stocks.json  # 股票/ETF列表
    holidays.json  # 节假日列表
    stk_sessions.json  # 交易时间模板
    fees_stk.json  # 佣金配置文件
datakit_stk  # 数据组件
    runDT.py  # 执行runDT.py启动数据组件
    dtcfg.yaml  # 行情通道、广播端口、数据落地，原demo中分离出了mdparsers.yaml, statemonitor.yaml
    logcfgdt.yaml
cta_stk  # 实盘组件
    run.py
    Strategies
        策略.py
    config.yaml  # 原demo中分离出了filters.yaml, executers.yaml, tdparsers.yaml, tdtraders.yaml, actpolicy.yaml
    logcfg.yaml
```

### 准备数据组件
---
* 复制股票数据组件demo<https://github.com/wondertrader/wtpy/tree/master/demos/datakit_stk>

* 打开配置文件`dtcfg.yaml`，配置XTP仿真行情通道
    ```yaml
    parsers:
    -   active: true
        id: parser
        module: ParserXTP
        host: 120.27.164.138
        port: '6002'
        protocol: 1
        buffsize: 128
        clientid: 1    
        hbinterval: 15    
        user: 你的XTP仿真账号
        pass: 你的XTP仿真密码
        code: SSE.000001,SSE.600009,SSE.600036,SSE.600276,SZSE.000001

    ```

* 打开配置文件`dtcfg.yaml`，配置数据落地目录
    ```yaml
    writer:
        module: WtDtStorage #数据存储模块
        async: true         #同步落地还是异步落地，期货推荐同步，股票推荐异步
        groupsize: 20       #日志分组大小，主要用于控制日志输出，当订阅合约较多时，推荐1000以上，当订阅的合约数较少时，推荐100以内
        path: ../STK_Data   #数据存储的路径
        savelog: false      #是否保存tick到csv
    ```

* 然后配置广播端口
    ```yaml
    broadcaster:                    # UDP广播器配置项
        active: true
        bport: 3997                 # UDP查询端口，主要是用于查询最新的快照
        broadcast:                  # 广播配置
        -   host: 255.255.255.255   # 广播地址，255.255.255.255会向整个局域网广播，但是受限于路由器
            port: 9001              # 广播端口，接收端口要和广播端口一致
            type: 2                 # 数据类型，固定为2
    ```

* 完成上述工作以后，就可以执行`runDT.py`启动数据组件了

### 准备环境
---
* 复制股票实盘demo<https://github.com/wondertrader/wtpy/tree/master/demos/cta_stk>

* 打开配置文件`config.yaml`，修改环境配置
    ```yaml
    env:
        name: cta                       #引擎名称：cta/hft/sel
        fees: ../common/fees_stk.json   #佣金配置文件
        filters: filters.yaml           #过滤器配置文件，这个主要是用于盘中不停机干预的
        product:
            session: TRADING            #驱动交易时间模板，TRADING是一个覆盖国内全部交易品种的最大的交易时间模板，从夜盘21点到凌晨1点，再到第二天15:15，详见sessions.json
    ```

* 修改数据读取路径，确认和datakit设置的存储路径一致
    ```yaml
    data:
        store:
            module: WtDtStorage
            path: ../STK_Data/
    ```

* 打开配置文件`tdtraders.yaml`，修改XTP交易通道配置
    ```yaml
    traders:
    -   active: true
        client: 1
        host: 120.27.164.69
        id: simnow
        module: TraderXTP
        user: 你的XTP仿真账号,
        pass: 你的XTP仿真密码,
        acckey: 你的XTP仿真key,
        port: 6001
        quick: true
        riskmon:
            active: true
            policy:
                default:
                    cancel_stat_timespan: 10
                    cancel_times_boundary: 20
                    cancel_total_limits: 470
                    order_stat_timespan: 10
                    order_times_boundary: 20
    ```

* 打开配置文件`tdparsers.yaml`，修改行情通道配置
    ```yaml
    parsers:
    -   active: true
        id: parser1
        module: ParserUDP
        host: 127.0.0.1
        bport: 9001       # 广播端口
        sport: 3997       # 查询端口
        filter: ''
    ```

* 至此，`config.yaml`里面需要用户修改的修改项基本就修改完成了，其他的配置项保持即可。

### 修改策略
---
`DualThrust`策略，在编写的时候，就考虑到了对股票的支持，所以加了一个参数`isForStk`。对于一般期货策略逻辑，要改写成支持股票，主要需要注意以下几点：
* 获取历史数据的时候，一定要将股票代码加`Q`
    ```python
    #读取最近50条1分钟线(dataframe对象)
    theCode = code
    if self.__is_stk__:
        theCode = theCode + "Q" #历史数据一定要用复权数据
    df_bars = context.stra_get_bars(theCode, self.__period__, self.__bar_cnt__, isMain = True)
    ```

* 对于国内市场来说，股票买入必须按照1手100股来挂单
    ```python
    trdUnit = 1
    if self.__is_stk__:
        trdUnit = 100   #交易股票的时候需要设定交易单位为100
    ```

* 最后发出信号的时候，品种代码一定不能带`Q`

* 策略逻辑要注意股票和期货的区别
    ```python
    #确定上轨和下轨
    upper_bound = openpx + k1* max(hh-lc,hc-ll)
    lower_bound = openpx - k2* max(hh-lc,hc-ll)

    #读取当前仓位
    curPos = context.stra_get_position(code)/trdUnit

    if curPos == 0:
        if highpx >= upper_bound:
            context.stra_enter_long(code, 1*trdUnit, 'enterlong')
            context.stra_log_text("向上突破%.2f>=%.2f，多仓进场" % (highpx, upper_bound))
            #修改并保存
            self.xxx = 1
            context.user_save_data('xxx', self.xxx)
            return

        if lowpx <= lower_bound and not self.__is_stk__: #不是股票才能做空
            context.stra_enter_short(code, 1*trdUnit, 'entershort')
            context.stra_log_text("向下突破%.2f<=%.2f，空仓进场" % (lowpx, lower_bound))
            return
    elif curPos > 0:
        if lowpx <= lower_bound:
            context.stra_exit_long(code, 1*trdUnit, 'exitlong')
            context.stra_log_text("向下突破%.2f<=%.2f，多仓出场" % (lowpx, lower_bound))
            return
    else:
        if highpx >= upper_bound and not self.__is_stk__: #不是股票才能做空
            context.stra_exit_short(code, 1*trdUnit, 'exitshort')
            context.stra_log_text("向上突破%.2f>=%.2f，空仓出场" % (highpx, upper_bound))
            return
    ```

* 策略修改完成以后，还可以放到回测环境下，做一次回测。回测的教程可以参考[回测自己的策略](mystrategy.md)这一章。


### 启动实盘
---
* 检查run.py是否正确配置
    ```python
    from wtpy import WtEngine,EngineType
    from Strategies.DualThrust import StraDualThrust

    if __name__ == "__main__":
        #创建一个运行环境，并加入策略
        engine = WtEngine(EngineType.ET_CTA)
        engine.init('../common/', "config.yaml", commfile="stk_comms.json", contractfile="stocks.json")
        
        straInfo = StraDualThrust(name='pydt_SH600000', code="SSE.600000", barCnt=50, period="d1", days=30, k1=0.1, k2=0.1, isForStk=True)
        engine.add_cta_strategy(straInfo)
        
        engine.run()

        kw = input('press any key to exit\n')
    ```

* 开盘前启动数据组件，比如9:20

* 执行run.py