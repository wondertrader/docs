# WtMonSvr监控服务

### WtMonSvr的作用
`WtMonSvr`的核心作用就是作为控制台的服务端，提供`http`接口和`websocket`的推送服务，整合数据管理、任务调度、事件转发、回测管理几个组件，完成整个控制台各种任务的执行。

`WtMonSvr`通过前台的`http`接口以及`websocket`接口和`web`端进行交互，然后通过调用后台的功能组件完成整个监控服务的工作。

### 监控服务组件
从服务端的构成来说，监控服务包含以下组件：
- 监控服务`WtMonSvr`，整个监控服务的核心入口，提供`http`接口服务
- 数据管理器`DataMgr`，用户管理用户和读取交易组合的相关数据
- 调度管理器`WatchDog`，自动调度任务，并监控组合的运行情况
- 推送服务`PushSvr`，实时推送组合的日志、交易回报等信息到`web`端
- 回测管理器`WtBtMon`，提供在线回测服务的管理模块
- 事件接收器`EventReceiver`，通过`WtMsgQue`组件接收各个交易组合的推送消息，并通过`PushSvr`转发到`web`端

### 如何使用监控服务
因为监控服务的接口都是预先设计的，所以绝大部分情况下，用户都无需做任何修改，启动服务可以参考以下代码：
```py
from wtpy.monitor import WtMonSvr

# 如果要配置在线回测，则必须要配置WtDtServo
# from wtpy import WtDtServo
# dtServo = WtDtServo()
# dtServo.setBasefiles(commfile="../common/commodities.json", 
#                 contractfile="../common/contracts.json", 
#                 holidayfile="../common/holidays.json", 
#                 sessionfile="../common/sessions.json", 
#                 hotfile="../common/hots.json")
# dtServo.setStorage("../storage/")
# dtServo.commitConfig()

# 创建监控服务，deploy_dir是策略组合部署的根目录，默认不需要传，会自动定位到wtpy内置的html资源目录
svr = WtMonSvr(deploy_dir="./deploy")

# 将回测管理模块提交给WtMonSvr
# from wtpy.monitor import WtBtMon
# btMon = WtBtMon(deploy_folder="./bt_deploy", logger=svr.logger) # 创建回测管理器
# svr.set_bt_mon(btMon) # 设置回测管理器
# svr.set_dt_servo(dtServo) # 设置dtservo

# 启动服务
svr.run(port=8099, bSync=False)
input("press enter key to exit\n")

# PC版控制台入口地址： http://127.0.0.1:8099/console
# 移动版控制台入口地址： http://127.0.0.1:8099/mobile
```