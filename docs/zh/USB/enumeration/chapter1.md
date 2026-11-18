# 第1章 USB 枚举的基础认知

## 1.1 什么是 USB 枚举？为什么必须枚举？

**USB 枚举（USB Enumeration）** 是 Host 在检测到设备接入后，通过 **Default Control Pipe（端点 0）** 与设备交换标准请求和描述符，完成以下目标的过程：

1. 确认设备存在并可通信（Reset 后能响应地址 0 上的控制传输）
2. 获知控制端点最大包长 `bMaxPacketSize0`
3. 为设备分配唯一总线地址（1~127）
4. 读取设备能力信息（Device / Configuration / Interface / Endpoint 等描述符）
5. 选择并激活一个配置（`SET_CONFIGURATION`），使设备进入 **Configured** 状态

枚举成功后，非零端点才可按配置描述符声明的方式进行 Bulk / Interrupt / Isochronous 传输。

### 为什么“必须”枚举？

USB 设计目标是：**一根总线上挂多种未知设备，Host 事先不知道设备类型、端点数量、包长、供电与协议类**。因此：

| 若跳过枚举 | 后果 |
|------------|------|
| 无地址分配 | 多设备无法区分，全部只能用地址 0（且地址 0 仅用于枚举期） |
| 无描述符 | Host 不知道接口类/子类/协议，无法加载正确驱动 |
| 无 SET_CONFIGURATION | 按 USB 2.0，设备未配置前，除端点 0 外的功能端点不可用 |
| 无 bMaxPacketSize0 | 控制传输分包错误，后续请求极易失败 |

一句话：**枚举 = Host 发现并“认领”设备的标准化握手，是后续一切功能通信的前提。**

> **标准 vs 实现**：USB 2.0 规定了设备状态机、标准请求与描述符格式；**Host 侧具体先读 8 字节还是一次读 18 字节、是否二次 Bus Reset、是否先读语言 ID 再读字符串**，属于主机栈策略，不同 OS/协议栈可能略有差异，但都必须满足规范对状态与请求语义的要求。

---

## 1.2 Host、Device、Hub、Controller 的职责

```
┌──────────────┐     ┌──────────────────┐     ┌──────────────┐
│  USB Host    │────▶│  Root Hub / Hub  │────▶│ USB Device   │
│ (PC / SoC)   │     │  (端口与拓扑)     │     │ (U 盘/HID…)  │
└──────┬───────┘     └────────┬─────────┘     └──────┬───────┘
       │                      │                      │
   Host Controller        Hub Controller         Device Controller
   (EHCI/OHCI/UHCI/…)     (可选外置 Hub)          (UDC)
       │                      │                      │
      PHY                    PHY                    PHY
```

### USB Host

- 唯一总线主人：发起所有事务（Transaction）
- 负责端口电源、Reset、Suspend/Resume、帧定时（SOF）
- 运行枚举状态机：发标准请求、解析描述符、加载类驱动
- 分配地址、选择配置、管理带宽与管道

### USB Device

- 被动响应 Host 发起的事务
- 通过上拉电阻宣告连接与速度（见后文）
- 在端点 0 上处理标准/类/厂商请求
- 按当前 **Device State** 决定哪些请求合法

### USB Hub

- 扩展端口，形成树形拓扑
- 向 Host 报告下游端口状态（Connect / Enable / Suspend / Overcurrent…）
- 转发或隔离 Reset、流量；含 Transaction Translator（TT）以支持 HS Hub 下挂 LS/FS 设备
- Root Hub 通常集成在 Host Controller 内，对外呈现为“根端口”

### USB Controller（易混，请分开记）

| 名称 | 位置 | 职责 |
|------|------|------|
| **Host Controller（HC）** | Host 侧硅片 | 按调度执行 USB 事务，产生 SOF，管理端口 |
| **HCD（Host Controller Driver）** | Host 软件 | 操作 HC 寄存器/DMA 描述符环，向上提供 URB/传输接口 |
| **UDC（USB Device Controller）** | Device 侧硅片 | 管理端点 FIFO、SETUP 包接收、握手 PID |
| **Device Stack / Gadget** | Device 软件 | 解析 Setup Packet，填充描述符，实现类协议 |
| **PHY** | 两侧都有 | 串行化、NRZI/位填充、差分驱动与收发、速度检测相关模拟电路 |

**对照记忆**：

- Host/Device 是**角色**
- Hub 是**拓扑节点**
- Controller/PHY 是**硬件实现分层**
- HCD/协议栈是**软件分层**

---

## 1.3 枚举涉及的核心概念

### Endpoint（端点）

设备内的数据源/汇，用 **Endpoint Address** 标识：

- 低 4 bit：端点号（0~15）
- bit7：方向（IN=1 设备→Host，OUT=0 Host→设备）

端点 0 双向（IN/OUT 共享控制逻辑），专用于控制传输。其它端点在配置描述符中声明类型与方向。

### Pipe（管道）

Host 视角：`（设备地址 + 端点地址 + 传输类型）` 构成的逻辑通道。

- **Default Control Pipe**：地址分配前后都存在的控制管道（枚举期地址为 0，之后为分配地址）
- 管道有各自的最大包长、传输类型、轮询间隔等属性（来自描述符）

### Control Transfer（控制传输）

唯一带 **Setup Stage** 的传输类型，用于标准/类/厂商请求。枚举几乎全部依赖控制传输。结构见第3章。

### Transaction / Transfer

| 术语 | 含义 |
|------|------|
| **Transaction（事务）** | 一次 Token + Data（可选）+ Handshake 的总线交互 |
| **Transfer（传输）** | 由一个或多个事务组成的完整数据移动（如一次 GET_DESCRIPTOR） |

### Descriptor（描述符）

设备用固定二进制布局“自我介绍”的数据结构。Host 通过 `GET_DESCRIPTOR` 读取。层级见第4章。

### Address（地址）

- **0**：默认地址，仅用于未分配地址或正在处理 `SET_ADDRESS` 的设备
- **1~127**：Host 分配的设备地址
- 同一时刻，总线上（同一 Host 控制器管理域内）地址唯一

### Device State（设备状态，USB 2.0）

```
Attached → Powered → Default → Address → Configured
                         ↑         │
                         └─────────┘  (可再 Reset 回到 Default)
任何已上电状态还可进入 Suspend（挂起）
```

| 状态 | 要点 |
|------|------|
| **Attached** | 已物理连接，未必已上电 |
| **Powered** | 已获得电源（VBUS），尚未 Reset |
| **Default** | 完成 Reset，地址仍为 0，可响应控制传输 |
| **Address** | 已成功 `SET_ADDRESS`，仍未配置 |
| **Configured** | 已 `SET_CONFIGURATION`（非 0），功能端点可用 |
| **Suspended** | 总线空闲超过 3 ms 进入；唤醒后回到进入前状态 |

### PID、Packet、Frame

- **PID**：包标识（SETUP/OUT/IN/SOF/DATA0/DATA1/ACK/NAK/STALL/PRE…）
- **Packet**：Token / Data / Handshake / Special
- **Frame（FS）/ Microframe（HS）**：Host 以 1 ms / 125 μs 为节拍发 SOF

### 速度相关术语

| 速度 | 信号速率 | 典型标识 |
|------|----------|----------|
| Low-Speed (LS) | 1.5 Mbps | D− 上拉 |
| Full-Speed (FS) | 12 Mbps | D+ 上拉 |
| High-Speed (HS) | 480 Mbps | 先以 FS 上拉连接，再 Chirp 协商升速 |

---

## 1.4 从物理连接到枚举成功的阶段总览

下面是**概念阶段**（细节与时序在第2章展开）：

```
① 物理插入 / VBUS 上电
        ↓
② Attach 检测（D+/D− 上拉导致线路离开 SE0）
        ↓
③ 去抖与上电稳定等待（Host 侧常见 ≥100 ms 策略）
        ↓
④ Bus Reset（≥10 ms SE0；HS 另有 Chirp）
        ↓
⑤ 设备进入 Default（地址 0）
        ↓
⑥ GET_DESCRIPTOR(Device) —— 常先取前 8 字节获知 bMaxPacketSize0
        ↓
⑦ （许多 Host）再次 Reset，然后 SET_ADDRESS
        ↓
⑧ 设备进入 Address 状态
        ↓
⑨ 读取完整 Device Descriptor / Configuration 集合 / 可选 String
        ↓
⑩ SET_CONFIGURATION(n)
        ↓
⑪ 设备进入 Configured，类驱动接管，开始功能通信
```

**状态与地址变化摘要**：

| 阶段 | Device State | 设备地址 |
|------|--------------|----------|
| 刚 Reset 完成 | Default | 0 |
| SET_ADDRESS 状态阶段成功后 | Address | Host 分配值 |
| SET_CONFIGURATION(非0) 成功后 | Configured | 保持 |
| 再次 Bus Reset | Default | 回到 0 |

---

## 1.5 LS / FS / HS 在枚举上的区别

枚举的**逻辑步骤相似**（Reset → 地址 0 控制传输 → SET_ADDRESS → 读描述符 → SET_CONFIGURATION），差异主要在**电气、速度协商与包时序**。

### 1.5.1 连接检测（上拉位置）

| 速度 | 上拉 | Host 如何初判 |
|------|------|----------------|
| Low-Speed | **D−** 经 1.5 kΩ 上拉到 3.3 V | 看到 D− 高 |
| Full-Speed | **D+** 上拉 | 看到 D+ 高 |
| High-Speed | 起初按 **FS 方式**接 D+ 上拉 | 先被当成 FS，Reset 后 Chirp 再升到 HS |

### 1.5.2 Bus Reset 之后

- **LS/FS**：Reset 结束即保持 LS 或 FS，进入 Default，可开始控制传输。
- **HS 设备**：
  1. Reset 期间设备发 **Chirp K**
  2. Host（若支持 HS）回 **Chirp K-J-K-J…**
  3. 协商成功则双方切换到 HS；失败则设备保持 FS 工作（若设备支持 FS 回落）

> Chirp 属于高速检测协议，细节以 USB 2.0 规范 Chapter 7 为准。枚举请求本身仍走控制传输，只是后续包在 HS 信令下发送。

### 1.5.3 控制端点最大包长

| 速度 | `bMaxPacketSize0` 合法值 |
|------|---------------------------|
| LS | 8 |
| FS | 8 / 16 / 32 / 64 |
| HS | **必须为 64** |

这正是 Host 常在分配地址前先读 Device Descriptor **前 8 字节**的原因：第 8 字节（偏移 7）就是 `bMaxPacketSize0`，用于正确设置后续控制传输的最大包长。

### 1.5.4 其它实务差异

| 项目 | LS | FS | HS |
|------|----|----|----|
| 位时间 | 较慢，超时与间隙不同 | 中等 | 最快，微帧调度 |
| Hub | 经 TT 时 Host 看到的是分割事务 | 直接或经 TT | 原生 HS |
| SOF | 1 ms 帧内可能用 Keep-Alive（LS） | SOF | 微帧 SOF |
| 抓包 | 分析仪需选对速度 | 最常见教学案例 | 需支持 HS 的分析仪 |

**学习建议**：先用 **Full-Speed** 设备把枚举抓包跑通（第6章案例即 FS），再对比 LS 上拉与 HS Chirp，避免一开始被三种速度的电气细节淹没。

---

## 1.6 本章易混点对照

| 概念 A | 概念 B | 区别 |
|--------|--------|------|
| Endpoint | Pipe | 端点是设备侧实体；管道是 Host 侧到该端点的逻辑连接 |
| Transaction | Transfer | 事务是一次总线握手；传输是一次完整请求（可含多事务） |
| Address 0 | Default 状态 | 地址 0 是地址值；Default 是状态。Default 时地址必为 0，但地址为 0 主要出现在 Default（及 SET_ADDRESS 完成前） |
| Reset | SET_CONFIGURATION(0) | Reset 强制回 Default/地址0；配置 0 是取消配置，回到 Address 状态（不改变地址） |
| Device Descriptor | Configuration Descriptor | 设备级一份；配置级可有多份，选其中一个激活 |

## 1.7 小结

- 枚举是 Host 用控制传输“认识并激活”设备的标准流程。
- 记住六个设备状态，尤其是 **Default → Address → Configured**。
- 端点 0 + 标准请求 + 描述符，是枚举的三条主线。
- LS/FS/HS 逻辑相近，差别在上拉、Chirp 与包长约束。

下一章按真实时间顺序，把从插线到配置完成的每一步拆开：[第2章 USB 枚举完整流程](./chapter2.md)
