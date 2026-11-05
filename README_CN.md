# C1M1 教学版 RPLIDAR ROS 2 驱动

基于 [frozenreboot/rplidar_driver](https://github.com/frozenreboot/rplidar_driver) 的教学分支 `teaching/c1m1`。
原驱动的功能一行没改，只加了中文注释、C1M1 配置、一个看门狗示例和这份文档。
英文原版说明见 [README.md](README.md)。

环境：Ubuntu 22.04 + ROS 2 Humble。

---

## 1. 三分钟跑起来

```bash
# 1) 确认雷达插上了（C1 是 CP2102N 串口芯片，一般是 /dev/ttyUSB0）
ls /dev/ttyUSB*
groups | grep dialout     # 没有 dialout 就执行：sudo usermod -aG dialout $USER，然后注销重登

# 2) 编译
cd ~/c1m1_ros
source /opt/ros/humble/setup.bash
colcon build --symlink-install
source install/setup.bash

# 3) 启动驱动 + RViz（一行命令，会自动弹出 RViz 窗口，看到绿色的点就成功了）
ros2 launch rplidar_driver c1m1.launch.py

# 4) 另开终端看频率（应约 10 Hz）
ros2 topic hz /scan
```

| 想要 | 命令 |
|---|---|
| 没有雷达也能练习（模拟数据） | `ros2 launch rplidar_driver c1m1.launch.py dummy_mode:=true` |
| 不开 RViz（机器人上没有显示器、SSH 远程） | `ros2 launch rplidar_driver c1m1.launch.py rviz:=false` |
| 用自己保存的 RViz 配置 | `ros2 launch rplidar_driver c1m1.launch.py rviz_config:=/path/to/my.rviz` |

RViz 配置在 `rviz/c1m1.rviz`：俯视图，Fixed Frame 是 `laser`（不需要 TF），1 m 网格，
红色轴是雷达正前方（+X），绿色轴是左边（+Y），点云是绿色方块。

> ⚠️ **避坑：自己开 rviz2 看不到点？**
> 驱动用 SensorDataQoS（Best Effort）发布 `/scan`。RViz 里 LaserScan 默认是 Reliable，两边不匹配就一个点都收不到，
> 也不报错。在 LaserScan → Topic → Reliability Policy 里改成 **Best Effort**。`rviz/c1m1.rviz` 已经设好了。

---

## 2. 和思岚官方驱动 `sllidar_ros2` 有什么不同

| | 官方 `sllidar_ros2` | 本驱动 |
|---|---|---|
| 程序结构 | 一个 `while` 循环：读一圈 → 发布 | Lifecycle 节点 + 后台读取线程 + 6 状态内部状态机 |
| 拔 USB | 读取失败，节点退出，要手动重启 | 自动进入 RESETTING，插回后自己重连 |
| 改 RPM | 改 launch 参数后重启节点 | `ros2 param set` 立即生效（C1M1 实测转速不变，见第 5 节） |
| 硬件访问 | 节点里直接调用 SDK | 通过 `LidarDriverInterface` 接口，真/假雷达可切换 |
| C1 波特率 | `sllidar_c1_launch.py` 里 460800 | `param/c1m1.yaml` 里 460800（原 `rplidar.yaml` 默认 1000000，不适用 C1） |

---

## 3. 两层状态机（最重要的概念）

本驱动有**两层**状态，新手最容易搞混：

```
外层：ROS 2 Lifecycle（"节点要不要干活"，由 launch 或命令控制）

  unconfigured ──configure──> inactive ──activate──> active
       ^                         │  ^                  │
       └──────cleanup────────────┘  └───deactivate─────┘

内层：DriverState（"雷达硬件现在怎么样"，只在 active 时运行，在 scan_loop 线程里）

  CONNECTING ─> CHECK_HEALTH ─> WARMUP ─> RUNNING
      ^                                     │ 连续读失败 > max_retries
      └────────── RESETTING <───────────────┘
                                   STANDBY（电机停转，串口保持连接）
```

- `c1m1.launch.py` 启动后会自动 configure → activate。
- **拔 USB 时外层一直是 active**，恢复是内层状态机自己在做，不需要任何人去 activate。

手动操作外层：

```bash
ros2 lifecycle get /rplidar_node          # 查看当前状态
ros2 lifecycle set /rplidar_node deactivate   # 让雷达回到待机（电机停，串口不关）
ros2 lifecycle set /rplidar_node activate     # 把雷达从待机叫醒
```

---

## 4. 拔插 USB 会怎样（C1M1 实测）

测试条件：RPLIDAR C1（FW 1.2，HW 18），`/dev/ttyUSB0`，460800，`max_retries: 3`，拔掉 36 秒后插回。
用的是本分支修复第 9.2、9.3 节之后的代码。
时间以拔掉 USB 的时刻为 0 秒；拔/插时刻用 `date +%s` 记录，精度约 1 秒。

| 时刻 | 发生了什么 | 终端里会看到 |
|---|---|---|
| 0 s | 拔掉 USB | `/scan` 停止 |
| 约 5 s | 看门狗发现断流（3 秒超时 + 1 秒检查周期） | 看门狗：`已经 3.6s 没收到 /scan 了！` |
| 约 9.5 s | 连续读取失败超过 3 次，RUNNING → RESETTING | `[FSM] Hardware unresponsive (Over 3 errors). Resetting...` |
| 约 9.5 s | 重建驱动对象，2 秒后进入 CONNECTING | `[FSM] Performing hardware reset (recreating driver)...` |
| 约 11.7 s 起 | 约每 1 秒尝试一次重连，一直持续到插回 | 见下方"重连循环" |
| 36 s | 插回 USB | |
| 约 40.5 s（插回后 4.5 s） | 串口连上，识别到 C1 | `[FSM] Connection established.`、`Type: RPLIDAR C1` |
| 约 41.8 s（插回后 5.8 s） | 电机启动完成 | `[FSM] Motor started. Entering RUNNING state.` |
| 约 44 s（插回后 8.1 s） | 第一帧 `/scan` 发出 | 看门狗：`/scan 恢复了，中断约 42.6s。` |

要点：
- **拔掉后大约 10 秒驱动才确认掉线**：每次读取要等 SDK 超时（约 2 秒），超过 `max_retries`=3 次才进入 RESETTING。
  调小 `max_retries` 能更快确认，但偶尔的串口干扰也可能被误判成掉线。
- **插回后大约 8 秒恢复出数据**，分三段：
  1. 约 4.5 秒：插回后驱动每秒都在尝试，但前几次仍然失败，说明雷达自己上电启动要几秒才能应答。
     修复第 9.2 节前后都是约 4.5 秒，所以这段时间是雷达的，不是驱动的。
  2. 约 1.3 秒：电机预热。
  3. 约 2.2 秒：从进入 RUNNING 到第一圈数据（正常开机也一样）。
- `rpm` 等参数在重建驱动后会保留（日志里 `Target RPM` 仍是拔掉前设置的值）。
- 恢复全程节点一直是 `active`，看门狗没有做任何操作，恢复完全是驱动内部状态机完成的。
- 拔着 USB 期间，`ros2 lifecycle get /rplidar_node` 照常在约 0.9 秒内返回（见第 9.3 节）。

**重连循环（拔着的时候）的日志**

修复第 9.2 节之前，每一轮重连约 4 秒，日志会误导人：

```
[Driver] Failed to get device info (data may be corrupted).
[FSM] Connection failed. Retrying...
[Driver] Device Model ID: 0
[Driver] Detected legacy (A-series) device.          <- 并没有 A 系列雷达
[Hardware Detail] N/A (Not connected or permission denied)
[FSM] Health error: 2. Sending device reset...       <- 雷达根本不在，健康检查当然失败
[FSM] Health error: 2 after reset. Disconnecting...
```

修复后（实测），每秒重试一次，日志只有这两种：

```
[Driver] Failed to get device info (data may be corrupted).
[Driver] Failed to get device info (data may be corrupted).
[FSM] Connection failed. Retrying...
```

为什么 `Failed to get device info` 是两行一组？驱动每秒重试一次，每次都打印这一行；
而 `Connection failed. Retrying...` 用的是 `RCLCPP_WARN_THROTTLE(..., 2000, ...)`，最多每 2 秒打印一次。
所以每两次重试才看到一次 `Retrying...`。这是限频，不是重复打印的 bug。

其他注意：
- 插回去后设备名可能变成 `/dev/ttyUSB1`（比如旧的还没释放），这时驱动会一直连不上。解决办法：
  执行 `scripts/create_udev_rules.sh` 生成固定别名 `/dev/rplidar`，再用
  `ros2 launch rplidar_driver c1m1.launch.py serial_port:=/dev/rplidar`。
  本次实测插回后仍是 `/dev/ttyUSB0`。
- 拔插期间看 `/diagnostics` 可以看到当前状态：`ros2 topic echo /diagnostics`。

---

## 5. 在线修改参数（不用重启节点）

| 参数 | 命令示例 | 取值 | 效果 |
|---|---|---|---|
| `rpm` | `ros2 param set /rplidar_node rpm 600` | 0~1200，0 = 型号默认 | 立即改转速，电机不停。**C1M1 实测无效，见下方** |
| `scan_mode` | `ros2 param set /rplidar_node scan_mode Standard` | 设备支持的模式名 | **会停电机再重启**，失败自动回到默认模式 |
| `auto_standby` | `ros2 param set /rplidar_node auto_standby true` | true/false | 没人订阅 `/scan` 时自动停电机 |
| `publish_point_cloud` | `ros2 param set /rplidar_node publish_point_cloud true` | true/false | 额外发布 `/cloud`（PointCloud2） |
| 其他参数（`serial_port`、`frame_id` 等） | — | — | 运行中改了不生效，需要改 `param/c1m1.yaml` 后重新 launch |

- 驱动没连上雷达时（比如 USB 拔着），所有在线修改都会被拒绝：`Driver not ready`。
- 查看当前值：`ros2 param get /rplidar_node rpm`。

**C1M1 的 rpm 实测：改了也不变**

| 设置的 rpm | `/scan` 实测频率 |
|---|---|
| 600 | 10.009 Hz |
| 480 | 10.004 Hz |
| 720 | 10.010 Hz |
| 360 | 9.997 Hz |
| 900 | 10.007 Hz |

每次 `param set` 都显示 `Set parameter successful`，驱动日志也打印 `Updating RPM to ...`，但雷达始终是 10 Hz（600 RPM）。
超出 0~1200 的值（比如 9999）会被拒绝：`RPM must be between 0 and 1200`。

> ⚠️ **避坑：C1 改 rpm 无效**
> 在 C1M1（FW 1.2）上实测，`rpm` 设成多少，雷达都按 10 Hz 转，而 `param set` 依然显示成功。
> 原因是雷达固件没有执行调速命令，而这个命令不等回应，驱动无从知道它失败了。
> 想确认转速到底变没变，只有一个办法：`ros2 topic hz /scan`，频率约等于 rpm ÷ 60。
> 换了别的型号或升级了固件，记得重新测一次。

细节：SDK 按型号判断 C1 支持转速控制，于是发出 `SL_LIDAR_CMD_HQ_MOTOR_SPEED_CTRL`。这个命令是
"只发不等回应"的，所以 SDK 返回 `SL_RESULT_OK`。就算驱动检查返回值，也发现不了问题。

**教训**：参数设置成功 ≠ 硬件真的变了。验证要看实际效果，不要只看命令返回值。

电机控制服务（不离开 active 状态，只停/启电机）：

```bash
ros2 service call /stop_motor  std_srvs/srv/Trigger
ros2 service call /start_motor std_srvs/srv/Trigger
```

---

## 6. 看门狗示例 `examples/scan_watchdog.py`

订阅 `/scan`，3 秒没数据就警告，然后查询驱动的 Lifecycle 状态再决定怎么做：

| 驱动状态 | 看门狗的动作 |
|---|---|
| active | 默认不插手：说明内层状态机正在重连，只提示检查 USB |
| active + `force_restart:=true` | 强制 deactivate → activate，演示手动重启生命周期 |
| inactive | 发送 activate |
| unconfigured | 发送 configure → activate（第 9.1 节的坑修好后才能成功） |
| 找不到节点 | 报错：驱动进程不在了 |

```bash
ros2 run rplidar_driver scan_watchdog.py
ros2 run rplidar_driver scan_watchdog.py --ros-args -p force_restart:=true
ros2 run rplidar_driver scan_watchdog.py --ros-args -p timeout_sec:=5.0
```

动手实验：
1. 运行驱动和看门狗，执行 `ros2 lifecycle set /rplidar_node deactivate`，约 3 秒后看门狗会把它重新 activate。
2. 拔 USB：看门狗只会提示，不会动驱动；断流期间每 5 秒提醒一次，插回后打印一次 `/scan 恢复了，中断约 X s`。

看门狗里有一个值得注意的细节：`get_state` 是异步调用，回应可能来得很晚（第 9.3 节修复前实测超过 4 秒）。
如果回应到达时 `/scan` 已经恢复，这个回应就是过时的，代码会直接丢弃（见 `on_state()` 开头）。
写异步代码时要记得问：**回应回来的时候，世界是不是已经变了？**
3. 用 `force_restart:=true` 再执行 `ros2 service call /stop_motor std_srvs/srv/Trigger`：
   看门狗会强制 deactivate → activate，对照驱动终端里的 `[Lifecycle]` 日志看每一步做了什么。

---

## 7. 想改代码去哪儿

| 我想… | 去这里 |
|---|---|
| 理解整体流程 | `include/rplidar_node.hpp` 顶部 `DriverState` 的注释 |
| 改启动/暂停时做的事 | `src/rplidar_node.cpp` 的 `on_configure` / `on_activate` / `on_deactivate` |
| 改掉线重连的逻辑（重试次数、等待时间） | `src/rplidar_node.cpp` 的 `scan_loop()`，`CONNECTING` 和 `RESETTING` 两个 case |
| 改发布出去的 `/scan` 内容（角度、过滤、强度） | `src/rplidar_node.cpp` 的 `publish_scan()` |
| 加一个新参数 | `include/rplidar_node.hpp` 里的 `Parameters` 结构体 → `init_parameters()` → 需要在线改的话再加到 `parameters_callback()` |
| 改 C1 的默认配置（串口、frame_id） | `param/c1m1.yaml` |
| 改 RViz 显示（颜色、点大小、视角） | `rviz/c1m1.rviz`；或在 RViz 里调好后 File → Save Config As 另存，再用 `rviz_config:=` 加载 |
| 改 launch 默认值（是否开 RViz、默认串口） | `launch/c1m1.launch.py` 里的 `DeclareLaunchArgument` |
| 改和 SDK 打交道的代码（型号识别、开电机、读数据） | `src/lidar_driver_wrapper.cpp` |
| 改假雷达生成的数据 | `src/lidar_driver_wrapper.cpp` 里的 `DummyLidarDriver` |
| 看串口收数据的底层线程 | `src/sdk/src/sl_async_transceiver.cpp` 的 `_proc_rxThread` / `_proc_decoderThread`（SDK 代码，一般不要改） |

改完记得：`cd ~/c1m1_ros && colcon build --symlink-install && source install/setup.bash`。
Python 示例用了 `--symlink-install`，改完不用重新编译；C++ 改了必须重新编译。

---

## 8. 为什么用多线程（给新手的一段话）

`grab_scan_data()` 读串口是"阻塞"的：调用后要等雷达转完一圈、数据收齐才返回，
USB 被拔时还要等到超时（约 2 秒）。如果在主线程里读，这段时间节点没法处理参数修改、
服务调用和诊断信息，看起来就像卡死了。所以驱动把读取放到单独的 `scan_thread_` 里，
主线程一直保持能响应。

两个线程会同时用到雷达对象 `driver_`（读数据 vs 改转速），所以用 `driver_mutex_` 加锁，
保证同一时间只有一个线程在通过串口和雷达通信；跨线程的标志位用 `std::atomic`。

思岚 SDK 内部还有两个线程：`_rxThread` 从串口收字节放进队列 `_rxQueue`，
`_decoderThread` 从队列取出来解包成测量点。

---

## 9. 填坑记录

下面三个坑都是本教学分支修掉的上游问题（代码位置汇总见第 12.2 节），每一条都写了现象、原因、修法和怎么验证。

### 9.1 cleanup 之后无法再 configure

**现象**：执行 `deactivate → cleanup → configure`，configure 失败，驱动日志：

```
Original error: parameter 'serial_port' has already been declared
Callback returned ERROR during the transition: configure
```

**原因**：ROS 2 规定一个参数在节点里只能 `declare_parameter` 一次。上游的 `init_param()` 每次都直接 declare，
而 `on_configure()` 每次 configure 都会调用它。第一次 configure 没问题，第二次就抛异常。

**修法**（`include/rplidar_node.hpp` 的 `init_param()`）：

```cpp
if (this->has_parameter(name)) {
  variable = this->get_parameter(name).template get_value<T>();   // 已声明：只读
} else {
  variable = this->declare_parameter<T>(name, variable);          // 第一次：声明
}
```

顺带的好处：运行中用 `ros2 param set` 改过的值（比如 rpm），重新 configure 后会保留。

**同一个坑的第二处**：`diagnostic_updater_` 是成员变量，cleanup 不清空它，每次 configure 都会再注册一个
`RPLidar Status`，`/diagnostics` 里出现重复条目。修法是在 `on_configure()` 里先 `removeByName()` 再 `add()`。

**教训**：Lifecycle 节点的回调可能被调用很多次，写 `on_configure()` 时要问自己：
"这里做的事第二次再做一遍，会不会出问题？"

**怎么自己验证**：

```bash
ros2 lifecycle set /rplidar_node deactivate
ros2 lifecycle set /rplidar_node cleanup
ros2 lifecycle set /rplidar_node configure
ros2 lifecycle set /rplidar_node activate
ros2 lifecycle get /rplidar_node      # 应该是 active
```

---

### 9.2 拔着 USB 时，日志说雷达是 "A-series"，还报 "Health error"

**现象**（C1M1 真机，USB 拔着）：每轮重连约 4 秒，日志反复出现：

```
[Driver] Failed to get device info (data may be corrupted).
[FSM] Connection failed. Retrying...
[Driver] Device Model ID: 0
[Driver] Detected legacy (A-series) device.
[FSM] Health error: 2. Sending device reset...
[FSM] Health error: 2 after reset. Disconnecting...
```

**原因**（`src/lidar_driver_wrapper.cpp` 的 `RealLidarDriver::connect()`）：
1. `drv_->connect(channel)` 成功，SDK 内部的"已连接"标志 `_isConnected` 变成 true。
2. 接着 `getDeviceInfo()` 失败，函数 `return false`，但**没有断开**，`_isConnected` 还是 true。
3. 下一轮 `scan_loop()` 里 `isConnected()` 返回 true，状态机以为连着，跳过重连，
   拿全零的设备信息去识别型号（Model ID 0 → A 系列），再去做健康检查，最后给不存在的设备发复位命令。

**修法**：失败分支里补一行。

```cpp
  if (SL_IS_FAIL(res)) {
    ...
    drv_->disconnect();   // 连接没完成，就把"已连接"标志清掉
    return false;
  }
```

**教训**：一个函数做了好几步，中间某一步失败时，要把前面已经做成的步骤撤销掉（"回滚"）。
否则对象会处在"一半连上"的状态，后面的代码会被它骗。

**怎么验证**：拔掉 USB，日志里只应出现 `Failed to get device info` 和 `Connection failed. Retrying...`，
不应再出现 `A-series` 和 `Health error`；插回后应看到 `Type: RPLIDAR C1`。

**实测结果**（C1M1，拔掉 36 秒）：

| | 修复前 | 修复后 |
|---|---|---|
| 拔着时每轮重连 | 约 4 秒 | 约 1 秒 |
| `A-series` / `Health error` / 复位命令 | 每轮都有 | 0 次 |
| 插回后到 `Connection established` | 4.6 秒 | 4.5 秒 |

插回后的时间几乎没变：这段时间主要是雷达自己上电启动（见第 4 节），驱动重试得再快也不会更快。
修复的收益是日志不再误导人，也不再对不存在的设备发复位命令。

### 9.3 拔着 USB 时，`ros2 lifecycle get` 要等好几秒

**现象**：拔掉 USB 后，看门狗向驱动发 `get_state` 请求，实测 4.2 秒才收到回应。
同一时间 lifecycle 命令、服务调用也都会卡住。

**原因**：三件事叠在一起。
1. 节点用 `SingleThreadedExecutor`：诊断定时器、`get_state` 服务、参数回调**全在同一个主线程**里排队执行。
2. `update_diagnostics()` 一开头就 `lock_guard(driver_mutex_)`。
3. 拔掉 USB 后，scan 线程拿着 `driver_mutex_` 在 `grab_scan_data()` 里等超时（约 2 秒一次），
   释放锁的时间极短又重新抢回去（见第 10 节）。

于是主线程卡在诊断回调里等锁，排在后面的 `get_state` 也跟着等。
而且 `update_diagnostics()` 根本不碰 `driver_`，它只读 atomic 变量、`params_` 和 `cached_device_info_`，
这把锁是多余的。更糟的是，scan 线程写 `cached_device_info_` 时本来就**没加锁**，所以这把锁也没保护到它。

**修法**：
- 删掉 `update_diagnostics()` 里的 `lock_guard(driver_mutex_)`。
- 新增一把只保护字符串的小锁 `info_mutex_`：scan 线程写 `cached_device_info_` 时加锁，诊断回调读的时候加锁。
  持有时间只有一次字符串拷贝，主线程不会被拖住。
- `params_.rpm` 只在参数回调里写，参数回调和诊断回调都在主线程里，不会同时执行，不需要加锁。

**教训**：第 8 节说"读串口单独开线程，主线程就不会卡"。但只要主线程去抢慢线程手里的锁，它照样会卡。
**锁要尽量小：只保护真正共享的数据，持有时间越短越好。**

**怎么验证**：先在雷达插着的时候执行一次 `time ros2 lifecycle get /rplidar_node`，记下耗时作为基准
（`ros2` 命令自己启动就要约 1 秒）。拔掉 USB 后马上再执行几次，耗时应该和基准差不多，不应再多出 2~4 秒。

**实测结果**（C1M1，用脚本连续执行 `ros2 lifecycle get` 两分钟，共 136 次）：

| 阶段 | 次数 | 平均耗时 | 最慢一次 |
|---|---|---|---|
| 拔掉之前 | 71 | 0.88 s | 1.21 s |
| 拔着 USB（含驱动等串口超时的前 10 秒） | 41 | 0.87 s | 0.89 s |
| 插回之后 | 24 | 0.88 s | 0.89 s |

拔着 USB 时和平时一样快。修复前同样的请求要等约 4.2 秒。
（拔掉之前有两次 1.21 s，是 `ros2` 命令自己偶尔慢一点，和驱动无关。）

---

## 10. 已知限制

**C1 改 rpm 无效**：见第 5 节的避坑提示。

**拔着 USB 时 `ros2 param set` 仍然要等几秒**
第 9.3 节的修复只让诊断回调不再抢 `driver_mutex_`。参数回调要通过 `driver_` 往串口发命令，必须持有这把锁，
所以拔着 USB 时改参数仍然要等 scan 线程读超时。`get_state`、lifecycle 命令和 `/diagnostics` 不受影响。

**留给你思考：scan 线程为什么"不让锁"**
`scan_loop()` 的 RUNNING 分支里，读取失败后的 `sleep_for(1ms)` 写在加锁的代码块**里面**。
scan 线程释放锁的时间极短，马上又去读串口、再等约 2 秒超时。`std::mutex` 不保证先来先得，
别的线程可能连续几轮都抢不到锁。试试把 sleep 移到锁外面，会有什么变化？会不会引入新问题？

---

## 11. 其他说明

- 本分支代码里有中文注释，上游仓库的 pre-commit 检查禁止非 ASCII 字符，所以这个分支不能直接给上游提 PR。
  如果要把第 9 节的修复贡献给上游，需要单独开一个只含英文的分支。

---

## 12. 本项目与上游原版的差异

对比基准：上游 [frozenreboot/rplidar_driver](https://github.com/frozenreboot/rplidar_driver) 的 `v1.4.2`
（commit `4446f8f`，即本仓库的 `main` 分支）。本节内容来自 `git diff main`。

**一句话**：驱动的核心架构（Lifecycle、scan 线程、状态机、SDK）一点没换；真正改动逻辑的只有 3 处 bug 修复，
共新增 25 行、删除 9 行（不含注释）。其余都是新增的教学文件和中文注释。

### 12.1 完全新增的文件

| 文件 | 作用 |
|---|---|
| `README_CN.md` | 本文档 |
| `param/c1m1.yaml` | C1M1 专用参数：波特率 460800、`/dev/ttyUSB0`、`frame_id: laser`、`publish_tf: false`。原版 `param/rplidar.yaml` 不动 |
| `launch/c1m1.launch.py` | 一行命令启动驱动 + RViz。只在第一次 configure 后自动 activate 一次，不会撤销你手动的 deactivate/cleanup |
| `rviz/c1m1.rviz` | RViz 配置：俯视图、Fixed Frame `laser`、网格、坐标轴、LaserScan（Best Effort） |
| `examples/scan_watchdog.py` | 看门狗示例：订阅 `/scan`，3 秒无数据就按 Lifecycle 状态处理（见第 6 节） |

### 12.2 改了逻辑的地方（重点看这里）

| 文件 | 函数 | 改了什么 | 为什么 |
|---|---|---|---|
| `include/rplidar_node.hpp` | `init_param()` | `declare_parameter` 前先 `has_parameter`，已声明就 `get_parameter` | 第 9.1 节：cleanup 后无法再 configure |
| `src/rplidar_node.cpp` | `on_configure()` | `diagnostic_updater_.add()` 前加 `removeByName("RPLidar Status")` | 第 9.1 节：重复的诊断条目 |
| `src/lidar_driver_wrapper.cpp` | `RealLidarDriver::connect()` | `getDeviceInfo()` 失败分支加 `drv_->disconnect()` | 第 9.2 节：误报 A-series |
| `include/rplidar_node.hpp` | 成员变量 | 新增 `std::mutex info_mutex_` | 第 9.3 节 |
| `src/rplidar_node.cpp` | `scan_loop()`，CONNECTING 分支 | 先写局部变量 `info`，再加 `info_mutex_` 写入 `cached_device_info_` | 第 9.3 节：原来写入时没加锁 |
| `src/rplidar_node.cpp` | `update_diagnostics()` | 删掉 `lock_guard(driver_mutex_)`；读 `cached_device_info_` 时加 `info_mutex_` | 第 9.3 节：主线程被串口超时堵住 |

关键代码一共就这些：

```cpp
// include/rplidar_node.hpp —— init_param()
if (this->has_parameter(name)) {
  variable = this->get_parameter(name).template get_value<T>();
} else {
  variable = this->declare_parameter<T>(name, variable);
}
std::mutex info_mutex_;                                   // 新成员

// src/rplidar_node.cpp —— on_configure()
diagnostic_updater_.removeByName("RPLidar Status");

// src/rplidar_node.cpp —— update_diagnostics()
- std::lock_guard<std::mutex> lock(driver_mutex_);        // 删掉
+ { std::lock_guard<std::mutex> info_lock(info_mutex_);
+   stat.add("Device Info", cached_device_info_); }

// src/lidar_driver_wrapper.cpp —— connect()
drv_->disconnect();                                       // getDeviceInfo 失败时
```

构建和配置文件（不影响驱动行为）：

| 文件 | 改动 |
|---|---|
| `CMakeLists.txt` | `install(DIRECTORY ...)` 里加 `rviz`；新增 `install(PROGRAMS examples/scan_watchdog.py ...)` |
| `package.xml` | 新增 `exec_depend`：`rclpy`（看门狗）、`rviz2`（launch 里的 RViz） |
| `.gitignore` | 忽略 `*.log`（真机测试日志） |

### 12.3 只加了中文注释、没动逻辑的地方

| 文件 | 新增注释行数 | 注释在哪 |
|---|---|---|
| `include/rplidar_node.hpp` | 52 | `DriverState` 状态图和每个状态、类说明（为什么用 Lifecycle）、各 `on_*` 回调、`scan_loop()`、`scan_thread_` / `is_scanning_` / `current_state_` / `driver_mutex_` |
| `src/rplidar_node.cpp` | 49 | `on_configure` / `on_activate` / `on_deactivate` / `on_cleanup`、`scan_loop()` 开头（为什么单开线程）、5 个 `case` 分支、`parameters_callback()` |
| `include/lidar_driver_wrapper.hpp` | 11 | 接口抽象层（"插座和插头"）、Real / Dummy 两个实现 |
| `src/sdk/src/sl_async_transceiver.cpp` | 4 | `_proc_rxThread`（收字节）、`_proc_decoderThread`（解包） |
| `src/sdk/src/sl_async_transceiver.h` | 2 | `_rxQueue`（两个线程共用的队列） |

> `git diff` 里 `DriverState` 枚举的 6 行显示为"删除 + 新增"，实际只是在每行末尾加了注释，枚举值和顺序都没变。
> 上表的注释行数包含 12.2 那几处 bug 修复旁边的 `【填坑】` 注释（这些注释本身也不影响逻辑）。

### 12.4 自己动手看差异

```bash
cd ~/c1m1_ros/rplidar_driver
git diff main --stat                      # 每个文件改了多少行
git diff main -- src/lidar_driver_wrapper.cpp   # 看某个文件的完整改动

# 只看改了逻辑的行（过滤掉注释和空行）。输出约 46 行，其中 12 行是 DriverState 枚举（只加了行尾注释）
git diff main -U0 -- src include | grep -E '^[+-]' | grep -vE '^(\+\+\+|---)' \
  | grep -vE '^[+-]\s*//' | grep -vE '^[+-]\s*$'
```

改动是**读代码的路线图**：先看 12.2 的 6 处，再对照第 9 节理解每处为什么要改；
其余代码按 12.3 的注释从 `include/rplidar_node.hpp` 的 `DriverState` 开始读。
