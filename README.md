# C1M1 教学级 ROS 2 驱动

![ROS2 Humble](https://img.shields.io/badge/ROS2-Humble-blue)
![Platform](https://img.shields.io/badge/Platform-Ubuntu%2022.04-orange)
![Status](https://img.shields.io/badge/Status-Stable-green)
![LiDAR](https://img.shields.io/badge/LiDAR-RPLIDAR%20C1M1-blueviolet)

> ⚠️ **本仓库是基于原版高性能驱动的二次开发版，专为思岚 C1M1 与新手教学定制。**
> 原版驱动的默认配置是给 S 系列用的（串口波特率 **1000000**），C1M1 用它**一个数据点都收不到**。
> 在 C1M1 上，**启动只认「🚀 唯一正确的启动方式」那一节**里的命令。

---

## 📚 文档导航

| 你的处境 | 该看哪份 |
|---|---|
| 第一次装 ROS 2、第一次编译、串口没权限 | **[零基础安装与配置指南](README_INSTALL.md)** |
| 想加功能、想改参数、不知道动哪个文件 | **[新手参数调优与代码导航](README_NEWBIE.md)** |
| 想了解真机拔插实测、踩过的坑、和上游的差异 | **[详细的真实拔插实测与填坑记录](README_CN.md)** |
| 想搞懂某个参数改了 RViz 会变成什么样 | [参数手册 doc/PARAM_GUIDE_C1M1.md](doc/PARAM_GUIDE_C1M1.md) |
| 想读核心函数为什么这么写 | [代码精读 doc/CODE_NOTES_CN.md](doc/CODE_NOTES_CN.md) |

推荐顺序：**安装 → 新手导航 → 中文说明**。这三份读完，你就能独立改这个驱动了。

---

## ⚠️ 硬件限制清单（调参之前先看）

下面这些是 **C1M1 的物理 / 固件限制，不是配置没生效**，别在它们身上耗时间：

| 限制 | 实测表现 | 怎么验证 |
|---|---|---|
| **不支持动态修改 RPM** | 设 360 / 480 / 600 / 720 / 900，`/scan` 始终约 10 Hz；`param set` 仍返回成功 | `ros2 topic hz /scan` |
| RPM 取值范围 0~1200 | 超出（如 9999）被参数回调拒绝：`RPM must be between 0 and 1200` | `ros2 param set /rplidar_node rpm 9999` |
| 串口波特率固定 460800 | 用 1000000（上游默认）一个点都收不到，SDK 报数据可能损坏 | `ros2 param get /rplidar_node serial_baudrate` |
| 在线开 `publish_point_cloud` 不生效 | 只翻标志位、打印日志，`/cloud` 不会出现，需 deactivate → activate | `ros2 topic list \| grep cloud` |

> ⚠️ **C1M1（FW 1.2）固件物理限制：不支持动态修改 RPM，转速恒定约 10 Hz。**
> 请通过 `ros2 topic hz /scan` 验证，具体见 [参数调优指南](README_NEWBIE.md)。

完整实测数据（5 组 rpm 对照表）见 [README_CN.md 第 5 节](README_CN.md)。

---

## ⚙️ 进阶操作：修改雷达参数

> ⚠️ **硬件限制警告（C1 专属）**：思岚 C1M1（FW 1.2）物理硬件**不支持动态修改电机转速**。
> 即便通过参数设置了，实际转速也始终锁定在 10Hz 左右。请勿在此浪费时间，
> 具体请查阅 [参数调优指南](README_NEWBIE.md)。

```bash
# 修改转速（C1 走 RPM 闭环，频率 ≈ rpm÷60）。用 examples/scan_probe.py 测，别用不带 --window 的 ros2 topic hz
ros2 param set /rplidar_node rpm 480

# 尝试切换扫描模式（部分固件可能支持）
ros2 param set /rplidar_node scan_mode "Standard"
```

命令都会返回 `Set parameter successful`，日志也会打印 `Updating RPM to 480`——**但转速不会变**。
这是硬件行为，不是你配错了。本仓库真机实测数据如下（[README_CN.md 第 5 节](README_CN.md)）：

| 设置的 rpm | `/scan` 实测频率 |
|---|---|
| 360 | 9.997 Hz |
| 480 | 10.004 Hz |
| 600 | 10.009 Hz |
| 720 | 10.010 Hz |
| 900 | 10.007 Hz |

顺带两个已知边界：

- 超出 `0~1200` 的值（例如 `9999`）会被参数回调直接拒绝：`RPM must be between 0 and 1200`
- 在线改 `rpm` 不会联动 `computed_ray_count`，开了 `interpolated_rays` 时画面可能出现空洞

---

## 🚀 唯一正确的启动方式（C1M1）

```bash
# 1) 编译（在工作空间根目录执行）
cd ~/c1m1_ros
colcon build --packages-select rplidar_c1m1_teaching_driver
source install/setup.bash

# 2) 启动：自动加载 param/c1m1.yaml（460800 波特率），并拉起 RViz
ros2 launch rplidar_c1m1_teaching_driver c1m1.launch.py
```

`c1m1.launch.py` 会替你完成两件事：把参数档切到 `param/c1m1.yaml`（`serial_baudrate: 460800`、`frame_id: laser`），
以及自动跑完 lifecycle 的 configure → activate。

### 常用变体

```bash
ros2 launch rplidar_c1m1_teaching_driver c1m1.launch.py rviz:=false          # 没有显示器 / SSH 远程调试
ros2 launch rplidar_c1m1_teaching_driver c1m1.launch.py rviz:=false serial_port:=/dev/ttyUSB1
ros2 launch rplidar_c1m1_teaching_driver c1m1.launch.py dummy_mode:=true     # 手上没雷达，用假数据练手
```

## ✅ 怎么确认真的跑起来了

```bash
ros2 node list                    # 应看到 /rplidar_node
ros2 lifecycle get /rplidar_node  # 应显示 active
ros2 topic hz /scan               # C1M1 实测约 10 Hz
```

RViz 里应看到一圈激光点，Fixed Frame 用 `laser`（`c1m1.launch.py` 默认帮你打开这份 RViz 配置）。

---

## 🚫 不要用这些命令（它们来自上游，对 C1M1 无效）

```bash
ros2 launch rplidar_c1m1_teaching_driver rplidar.launch.py             # ✗ 加载 param/rplidar.yaml，波特率 1000000
ros2 launch rplidar_c1m1_teaching_driver rplidar_with_args.launch.py   # ✗ 同上
ros2 launch rplidar_c1m1_teaching_driver composition.launch.py         # ✗ 上游组件化示例，未配 C1M1 参数
ros2 run rplidar_c1m1_teaching_driver rplidar_node                     # ✗ 不带参数档，全用上游默认值
```

**典型症状**：命令不报错、节点也起来了，但 `ros2 topic hz /scan` 一直没输出——
因为 C1M1 是以 460800 会话的，听不懂按 1000000 波特率发出的指令。

上游原始英文 README 已存档到 [doc/UPSTREAM_README_EN.md](doc/UPSTREAM_README_EN.md)，仅供对照，其中的命令同样不适用于 C1M1。

---

## 📄 许可与来源

本仓库基于 [frozenreboot/rplidar_driver](https://github.com/frozenreboot/rplidar_driver) v1.4.2 修改，
沿用上游同样的许可证（见 [LICENSE](LICENSE)）。
上游作者与贡献者名单见 [README_CN.md](README_CN.md) 开头；GitHub Contributors 面板里为何会出现一个 AI 条目：见 [README_CN.md 第 9.4 节](README_CN.md#94-关于-contributors-列表里的-claude)。
