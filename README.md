# C1M1 教学级 ROS 2 驱动

> ⚠️ **本仓库是基于原版高性能驱动的二次开发版，专为思岚 C1M1 与新手教学定制。**
> 原版驱动的默认配置是给 S 系列用的（串口波特率 **1000000**），C1M1 用它**一个数据点都收不到**。
> 在 C1M1 上，只认下面那一条启动命令。

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

## 🚀 唯一正确的启动方式（C1M1）

```bash
# 1) 编译（在工作空间根目录执行）
cd ~/c1m1_ros
colcon build --packages-select rplidar_driver
source install/setup.bash

# 2) 启动：自动加载 param/c1m1.yaml（460800 波特率），并拉起 RViz
ros2 launch rplidar_driver c1m1.launch.py
```

`c1m1.launch.py` 会替你完成两件事：把参数档切到 `param/c1m1.yaml`（`serial_baudrate: 460800`、`frame_id: laser`），
以及自动跑完 lifecycle 的 configure → activate。

### 常用变体

```bash
ros2 launch rplidar_driver c1m1.launch.py rviz:=false          # 没有显示器 / SSH 远程调试
ros2 launch rplidar_driver c1m1.launch.py rviz:=false serial_port:=/dev/ttyUSB1
ros2 launch rplidar_driver c1m1.launch.py dummy_mode:=true     # 手上没雷达，用假数据练手
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
ros2 launch rplidar_driver rplidar.launch.py             # ✗ 加载 param/rplidar.yaml，波特率 1000000
ros2 launch rplidar_driver rplidar_with_args.launch.py   # ✗ 同上
ros2 launch rplidar_driver composition.launch.py         # ✗ 上游组件化示例，未配 C1M1 参数
ros2 run rplidar_driver rplidar_node                     # ✗ 不带参数档，全用上游默认值
```

**典型症状**：命令不报错、节点也起来了，但 `ros2 topic hz /scan` 一直没输出——
因为 C1M1 是以 460800 会话的，听不懂按 1000000 波特率发出的指令。

上游原始英文 README 已存档到 [doc/UPSTREAM_README_EN.md](doc/UPSTREAM_README_EN.md)，仅供对照，其中的命令同样不适用于 C1M1。

---

## 📄 许可与来源

本仓库基于 [frozenreboot/rplidar_driver](https://github.com/frozenreboot/rplidar_driver) v1.4.2 修改，
沿用上游同样的许可证（见 [LICENSE](LICENSE)）。
上游作者与贡献者名单、以及 GitHub Contributors 面板里为何会出现一个 AI 条目：见 [README_CN.md](README_CN.md) 开头。
