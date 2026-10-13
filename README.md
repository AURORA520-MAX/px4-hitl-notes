# PX4 HITL（硬件在环）仿真环境搭建记录

在 Ubuntu 22.04 + QGroundControl 下，为 **Pixhawk 6C mini** 飞控搭建 **PX4 HITL（Hardware-In-the-Loop，硬件在环）** 仿真环境，通过 jMAVSim 与实机飞控联调，实现遥控器控制下的仿真飞行测试。

> HITL 与 SITL 的区别：SITL 是纯软件仿真，飞控逻辑也跑在电脑上；HITL 则是**真实飞控硬件**运行固件，由电脑上的仿真器提供传感器数据、并接收飞控输出的执行器指令，因此可以验证真实固件的模块状态与参数配置。

---

## 1. 复现前提

**硬件**：一台组装完成、可通过遥控器正常连接控制的无人机

**环境参数**

| 项目 | 版本 |
| --- | --- |
| 固件版本 | PX4 1.17 |
| 飞控型号 | Pixhawk 6C mini |
| 地面站版本 | QGroundControl 5.0.8 |
| 电脑系统 | Ubuntu 22.04 |

---

## 2. 固件 HITL 模块检测

HITL 仿真必需的核心模块是 `pwm_out_sim`，**它不会默认集成在所有 PX4 固件中**，需提前检测，缺失则无法使用 HITL 仿真。

1. 打开 QGroundControl（QGC）
2. 依次打开：**分析工具（Analyze Tools） → MAVLink 控制台**
3. 在控制台输入检测指令：

```bash
pwm_out_sim status
```

4. 结果判断：

- 若返回 `nsh: pwm_out_sim: command not found`，说明固件未安装该模块，需重新编译固件并添加 `pwm_out_sim`
- 若正常返回模块状态，说明固件支持 HITL 仿真，可继续后续配置

---

## 3. jMAVSim 仿真环境部署

需提前在本地克隆完成 `PX4-Autopilot` 源码仓库，再执行以下操作。

**1）安装依赖**

```bash
sudo apt update
sudo apt install ant
```

**2）下载 jMAVSim 源码**

进入 PX4 源码目录，拉取 jMAVSim 子模块：

```bash
cd ~/PX4-Autopilot
git submodule update --init --recursive Tools/simulation/jmavsim/jMAVSim
```

---

## 4. QGroundControl 核心配置

### 4.1 设备连接与机型选择

1. 使用 USB 数据线，将 Pixhawk 6C mini 飞控与电脑连接
2. 在 QGC 中选择适配的 **HITL 机型**，点击应用并重启，等待飞控重启生效
3. 开启**硬件在环仿真模式**

### 4.2 关闭自动连接，防止端口抢占

关闭 QGC 自动连接 Pixhawk 的功能，避免地面站抢占飞控 USB 端口，导致 jMAVSim 无法连接飞控、仿真启动失败。

### 4.3 手动配置飞控连接参数

手动添加 Pixhawk 飞控连接，严格匹配 **连接类型、串口端口、波特率** 参数。参数错误会导致地面站连接失败、仿真异常。

> ⚠️ 注意：仅配置手动连接参数，**切勿开启自动连接**，否则会挤占仿真端口。

---

## 5. PX4 固件关键参数配置

在 QGC 参数编辑器中修改以下两个核心参数，保障 HITL 仿真正常运行：

| 参数 | 值 | 说明 |
| --- | --- | --- |
| `COM_RC_IN_MODE` | `3` | RC or MAVLink keep first，优先遥控器 / MAVLink 控制 |
| `NAV_RCL_ACT` | `0` | Disabled，关闭仿真模式下的丢控失效保护 |

修改完成后保存参数，固件配置全部完成。

---

## 6. HITL 仿真测试流程

严格按照以下顺序操作，避免端口冲突、连接失败问题。

1. 确认飞控已通过 USB 连接电脑，**完全关闭 QGC 地面站**
2. 终端启动 jMAVSim HITL 仿真程序：

```bash
cd ~/PX4-Autopilot
./Tools/simulation/jmavsim/jmavsim_run.sh -q -s -d /dev/ttyACM0 -b 921600 -r 250
```

3. 仿真程序启动成功后，**再打开 QGC 地面站**
4. **仿真连通性验证**：查看 QGC 界面 MAVLink 数据状态，出现对应连通标识即代表仿真与飞控通信成功
5. 仿真就绪，可通过**遥控器控制仿真无人机完成飞行测试**

---

## 7. 踩坑记录

- `pwm_out_sim` 缺失 → 固件需自行编译并加入该模块，否则 HITL 无法启动
- QGC 的自动连接会抢占飞控 USB 端口 → 必须关闭，改用手动连接
- 启动顺序不能颠倒：先启动 jMAVSim，再打开 QGC；顺序反了会连接失败
- 丢控失效保护会拦截仿真中的解锁/动作 → 需将 `NAV_RCL_ACT` 置 0

---

## 测试效果

<!-- 建议在这里补 1~2 张 QGC MAVLink 连通截图，以及一段飞行测试录屏（大文件建议放 GitHub Release 或压缩后提交） -->

---

## 参考资料

- [PX4 官方文档：HITL vs SITL](https://docs.px4.io/main/en/simulation/hitl#hitl-vs-sitl)
- [CSDN：PX4 HITL 配置教程](https://blog.csdn.net/weixin_42037083/article/details/137450308)
- [知乎：PX4 硬件在环仿真](https://zhuanlan.zhihu.com/p/1965511743506716001)
- [B 站：超维小课堂（四）PX4 软件硬件在环仿真](https://www.bilibili.com/video/BV1W14y117CF/)
