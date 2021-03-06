# Skynet 日志文件服务
修改自 [Veinin/skynet-logger](https://github.com/Veinin/skynet-logger)，扩展实现按日期建文件夹，单个服务可享有独立日志文件  

需要**修改 skynet 配置**文件：
```
daemon = "./skynet.pid" -- 后台时才启用日志

my_logger = true
logger     = "logger" -- 日志服务名
logservice = "snlua"
log_debug = true      -- 记录 debug 类型
```
注释 `daemon` 将调用 `print`  
注释 `log_debug` 将不记录调试打印  
注释全部 `my_logger` 以下部分会转到调用 `skynet.error`  

**目录结构**如下：  
```
├── 20190822
│   ├── serviceType1
│   │   └── service1.log
│   ├── serviceType2
│   │   ├── service2.log
│   │   ├── service3.log
│   │   └── service4.log
│   ├── serviceType3
│   │   └── service5.log
│   ├── service6.log
│   └── skynet.log
├── 20190823
└── 20190824
```
目录会在凌晨零点新建，单服务日志可设置不转移目录。如果需要推延或提早新建转移时间点，可修改 *logger.lua* 中的 `ZERO_MOVE_TIME`；如果要改变文件默认打开模式，可修改 `DEFAULT_FILE_MODE`，当前为 `"w"`   

API
日志服务默认在每个进程主服务启动之前就会加载，日志 API 分为5个等级，操作 API 针对单服务独立文件  
使用前需先引用 `local log = require "log"`  
**日志 API**  
`log.debug(...)` 基本调试信息，设置了 `log_debug = true` 时记录  
`log.info(...)` 应用程序运行过程中关键信息  
`log.warning(...)` 警告信息，表明会出现潜在错误的情形（会追加打印最后的文件调用位置  
`log.error(...)` 错误信息，虽然发生错误事件，但仍然不影响系统的继续运行（会追加打印最后的文件调用位置  
`log.fatal(...)` 严重的错误事件将会导致应用程序的退出（会追加打印最后的文件调用位置  
**操作 API**  
`log.separate(path, file, no_change_dir, mode)` 以 `path` 为独立目录， `file` 为文件名，独立一份日志文件 */path/file.log* 记录当前服务，设置 `no_change_dir` 在凌晨不转移目录文件，`mode` 文件打开模式，通常为 `"w"` 或 `"a"` ，不填则为默认的 `DEFAULT_FILE_MODE`  
`log.forward(path, file, no_change_dir, mode)` 当前服务日志转接到 */path/file.log* 文件中，不传值时转到默认的主文件 *skynet.log*，其他参数同上  
`log.close()` 关闭当前服务独立的日志文件，即 `io.close`  
注意  
本服务使用 `skynet.time()` 做日志时间，即 skynet 程序时间而非系统时间  

示例：  
```lua
local log = require "log"

log.info("info something")
log.debug("debug something")
log.error("get error", "i am error")
skynet.error("called skynet.error")
log.info("i will separate")
log.separate("separate", "separateOne")
log.info("separate done")

--通常只需要用到以上部分
log.info("i will forward")
log.forward()
log.info("forward done")
log.info("i will separate again")
log.separate("", "separateTwo")
log.info("separate again done")
log.info("i will forward to", "separate/separateOne.log")
log.forward("separate", "separateOne")
log.info("forward to", "separate/separateOne.log", "done")

log.close()
skynet.exit()
```
*./log/20190824/skynet.log* 结果如下：  
```
[:00000008][12:04:44][ info ] info something
[:00000008][12:04:44][debug ] debug something
[:00000008][12:04:44][ error] get error i am error   <main.lua:54>
[:00000008][12:04:44][SKYNET] called skynet.error
[:00000008][12:04:44][ info ] i will separate
[:00000008][12:04:44][ info ] forward done
[:00000008][12:04:44][ info ] i will separate again
[:00000008][12:04:44][SKYNET] KILL self
```
*./log/20190824/separate/separateOne.log* 结果如下：  
```
[:00000008][12:04:44][ info ] separate done
[:00000008][12:04:44][ info ] i will forward
[:00000008][12:04:44][ info ] forward to separate/separateOne.log done
```
*./log/20190824/separateTwo.log* 结果如下：  
```
[:00000008][12:04:44][ info ] separate again done
[:00000008][12:04:44][ info ] i will forward to separate/separateOne.log
```
