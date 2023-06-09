# 控制台使用手册

### 系统登录和主界面
在浏览器输入控制台的访问地址`http://部署地址:8099/console`，进入登录界面，输入用户名和密码，点击登录按钮即可进入主界面。

控制台默认账号密码为`superman`/`Helloworld!`
![](./images/console/1_login.png)

主界面默认显示监控中心
![](./images/console/2_monitor_main.png)

整个导航栏位于上方，分为三个部分：
* Logo
* 导航切卡
* 用户区域，包含了*主题切换*、*修改密码*、*注销登录*三个按钮，鼠标移动到头像上，还可以看到用户的登录信息

![](./images/console/3_header.png)

点击*主题切换*按钮，会自动在*黑色主题*和*白色主题*之间进行切换，默认为*黑色主题*
![](./images/console/25_theme.png)

### 组合控制
监控中心整体上分为两个区域：
* 左侧为*组合列表*区
* 右侧为*组合详情*区，组合详情区域，也分为5个模块：
    - 策略管理
    - 组合管理
    - 通道管理
    - 配置管理
    - 滚动日志

在*组合列表*区，展示了组合的基本信息，以及最后一个交易日的*盈亏情况*，点击*组合名称*，右侧的*组合详情*会自动切换。点击“*刷新*”按钮，可以刷新组合的最新当日*盈亏数据*
![](./images/console/4_groups.png)

点击组合列表上方的“+”按钮，可以添加新的组合
![](./images/console/05_addgroup.png)

每一个组合，一共展示了4个按钮：
* 左侧按钮会根据组合的运行状态，自动在“*启动*”和“*停止*”之间进行切换

* *右侧第一个*按钮为“*删除*”按钮，用于删除组合

* 点击*右侧第二个*按钮，可以查看和修改组合的基本信息
![](./images/console/06_modgroup.png)

* 点击*右侧第三个*按钮，可以配置组合的调度信息
![](./images/console/07_moncfg.png)

### 组合详情-策略管理
策略管理模块，包含了策略的持仓、交易、信号、回合以及资金数据
* 策略持仓明细
![](./images/console/08_stra_pos.png)

* 策略成交明细
![](./images/console/09_stra_trd.png)

* 策略信号明细
![](./images/console/10_stra_sig.png)

* 策略回合明细
![](./images/console/11_stra_rnd.png)

* 全部策略当日资金
![](./images/console/12_stra_allfnd.png)

* 单策略每日资金及净值曲线
![](./images/console/12_stra_fnd.png)

### 组合详情-组合管理
组合管理模块，包含了组合的持仓、交易、风控、回合以及资金数据
* 组合持仓明细，上半部分为组合的*持仓明细表*，下半部分是*不同品种的累计收益*的柱状图
![](./images/console/14_grp_pos.png)

* 组合风控，是盘中紧急干预的入口。主要通过对组合中的`filter.json`文件进行管理，从而实现在不干预策略的前提下，达到人工介入的目的。
    ![](./images/console/15_grp_risk.png)

    组合风控，主要通过三类过滤器来实现：

    1）*策略过滤器*

    策略过滤器，即当策略被配置为过滤时，则该策略所有的信号将不再执行。如X策略，正常有3手多A合约，当X策略过滤器配置生效时，3手多A合约，的目标仓位将被忽略，从而基本组合上就会减少3手多的A合约，执行器发现目标仓位少了3手多的A合约，就会发出卖出3手A合约的交易指令。

    操作指南：修改策略过滤器时，只需要根据需要将开关拨到通过或者过滤的位置，然后点击右下角的提交按钮，即可生效。

    2）*代码过滤器*

    代码过滤器，即针对交易合约代码进行过滤。当B品种被过滤时，则整个基本组合在B品种上的目标仓位将会强制设置为0。无论在过滤器生效之前，B品种有多少仓位，执行器都会发出清理掉全部B品种的头寸的交易指令。

    操作指南：因为交易品种是策略自己确定的，所以从管理上来说，就没有办法预设，因此代码过滤器的设置，需要根据自己的需求添加要过滤的代码。代码的规则为“交易所.品种”，如CFFEX.IF。

    添加了代码过滤器以后，如果需要删除，点击右侧的删除按钮即可。如果不想删除，也可以通过将开关拨到通过位置上来重新开放该品种。配置完成以后，点击右下角的提交按钮，即可生效。

    3）*通道过滤器*

    通道过滤器，是针对执行通道（交易通道）进行紧急干预的机制。当通道过滤器生效时，所有的目标仓位都不会再像特定的通道发出同步指令了。假如P组合，拥有3手多的A合约，4手空的B合约，这时通道Y也有3手多的A合约和4手空的B合约。当针对Y的通道过滤器生效以后，如果P组合的A合约调整到1手多，B合约调整到2手空，这个时候Y通道就还是保持在3手多的A和4手空的B的仓位。
    
    通道过滤器的意义在于：如果需要进行强人工干预的时候，比如因为组合P表现不好，F产品需要紧急强平，这个时候就需要先断开组合P和F产品通道的联系，然后手动对F产品的交易通道进行操作。

* 组合成交明细
![](./images/console/15_grp_trd.png)

* 组合回合明细
![](./images/console/17_grp_rnd.png)

* 组合每日资金及净值曲线
![](./images/console/18_grp_fnd.png)

### 组合详情-通道管理
通道管理，即查看交易通道的各种数据，简单点理解就是各个交易账号的实时数据。包括持仓明细、成交明细、订单明细以及资金明细四个子类。

* 通道持仓明细
![](./images/console/19_chnl_pos.png)

* 通道成交明细
![](./images/console/20_chnl_trd.png)

* 通道订单明细
![](./images/console/22_chnl_ord.png)

* 通道资金数据
![](./images/console/21_chnl_fnd.png)

### 组合详情-配置管理
配置管理，即对组合的配置文件进行管理。该管理界面以左侧一个可以管理的文件树，右侧一个编辑器的形式展示出来。
![](./images/console/22_cfg_editor.png)

### 调度管理
调度管理，即对程序的调度进行管理。调度管理主要包括两个部分，一个是程序的调度配置，一个是调度日志。
调度任务的配置，需要配置的内容包括：
* 任务ID，`全局唯一ID`，用于标识任务
* 启动参数，如果是交易组合，则一般是交易的入口脚本`run.py`
* 执行程序，即执行的程序。点击最右侧的“关联”按钮，可以直接读取`WtMonSvr`运行的`python`环境的路径。如果是其他程序，可以直接填写程序的路径
* 工作目录，即程序的`运行目录`，对于交易组合来说，一般为组合的根目录
* 消息地址，这个为组合特有的，如果配置了消息地址，服务端会自动接收消息队列的消息，并通过`websocket`接口推送给控制台
* 监控设置，包含了`自动调度`和`进程守护`：
    - 进程守护，即保证进程的运行，监控服务会在设置的时间间隔检查进程是否存在，如果不存在，则会自动启动进程，一般用于`7*24小时`的服务
    - 计划任务，可以针对每一周的7天，配置6个时间点，执行启动、停止和重启的操作

![](./images/console/23_schedule.png)

### 用户管理
用户管理，即控制台的用户管理，目前预设了三个角色：`管理员`、`风控员`。
* 管理员具有全部权限
* 风控员则只能管理组合，没有用户管理的权限
另外，对于风控员和管理员，还可以单独配置`组合权限`，如果不配置组合权限则默认全部查看
![](./images/console/24_admin.png)