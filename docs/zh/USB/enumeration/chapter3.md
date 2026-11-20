# 第3章 深入分析 USB Control Transfer

枚举阶段几乎所有 Host↔Device 交互都是 **Control Transfer**。读不懂控制传输，就无法读懂抓包，也无法写对 Device 固件。

---

## 3.1 Control Transfer 的三个阶段

一次完整控制传输由最多三个阶段组成：

```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│ Setup Stage │ →  │ Data Stage  │ →  │ Status Stage│
│  (必须)      │    │  (可选)      │    │  (必须)      │
└─────────────┘    └─────────────┘    └─────────────┘
```

| 阶段 | 作用 | 方向由谁决定 |
|------|------|----------------|
| **Setup Stage** | 传递 8 字节 Setup Packet，定义请求 | 固定：Host→Device（SETUP + DATA0） |
| **Data Stage** | 按 `wLength` 与方向传输载荷 | 由 `bmRequestType.Dir` 决定 |
| **Status Stage** | 用**零长度数据包**确认整次传输成败 | 与 Data 方向**相反**；若无 Data，则为 IN |

### 有无 Data Stage？

- `wLength == 0`：**无 Data Stage**（如 `SET_ADDRESS`、`SET_CONFIGURATION`）
- `wLength > 0`：有 Data Stage；实际字节数 ≤ `wLength`，设备可用短包提前结束

### Status Stage 方向规则（务必记牢）

| 情形 | Status 阶段 Token | 谁发零长度 DATA |
|------|-------------------|-----------------|
| 无 Data Stage | **IN** | Device → Host |
| 有 Data Stage 且方向为 IN（设备→Host） | **OUT** | Host → Device |
| 有 Data Stage 且方向为 OUT（Host→设备） | **IN** | Device → Host |

记忆口诀：**Status 的数据方向与 Data Stage 相反；没有 Data 时，Status 用 IN。**

---

## 3.2 数据包结构速览（读抓包前先认包）

USB 2.0 全速率/低速总线上，一次事务由若干 **Packet** 组成。与枚举最相关的三类：

### Token 包（Host → Device）

```
SYNC | PID | ADDR(7bit) | ENDP(4bit) | CRC5 | EOP
```

| 字段 | 说明 |
|------|------|
| **PID** | SETUP / OUT / IN / SOF 等（含校验半字节） |
| **ADDR** | 设备地址 0~127；枚举早期为 0 |
| **ENDP** | 端点号 0~15；控制传输用 0 |
| **CRC5** | 覆盖 ADDR+ENDP |

### Data 包

```
SYNC | PID(DATA0/DATA1/DATA2/MDATA) | Data(0~MPS 字节) | CRC16 | EOP
```

控制传输的 Setup 数据阶段固定 **DATA0 + 恰好 8 字节**；其后 Data/Status 按 Toggle 使用 DATA0/DATA1。

### Handshake 包

```
SYNC | PID(ACK/NAK/STALL/NYET) | EOP
```

无地址字段——握手总是针对**刚刚那次** Token/Data 事务。

> **标准 vs 分析仪显示**：规范定义的是上述字段；Wireshark/分析仪往往还解码出“方向、设备地址、端点、Setup 字段展开”。电气层的 SYNC/EOP/位填充通常被工具折叠，排障时先看 PID 与 ADDR/ENDP。

---

## 3.3 Setup Stage 的总线过程

```
Host  ── SETUP token (ADDR, EP=0) ──▶  Device
Host  ── DATA0 + 8 字节 Setup      ──▶  Device
Host  ◀──────── ACK ────────────────  Device
```

要点：

1. Setup 数据**永远用 DATA0** PID（规范固定）
2. 设备必须接受 Setup（即使曾 STALL，Setup 也会清除控制管道上的 Halt——USB 2.0 规定新的 Setup 可重置控制管道的 Stall 条件）
3. 设备用 **ACK** 确认已收到 Setup；若未就绪，行为依赖控制器实现，但协议期望尽快接受 Setup
4. Setup 事务的 Data 包长度必须是 8；长度异常时设备应忽略该 Setup（具体容错属实现，但 Host 不会故意发错长）

---

## 3.4 Data Stage 的总线过程

Data Stage 由一个或多个 **IN 或 OUT 事务**组成，每次最多 `bMaxPacketSize0` 字节。

### 控制读（Device → Host），例如 GET_DESCRIPTOR

```
Host ── IN ──▶ Device
Host ◀─ DATA1/DATA0 + payload ─ Device
Host ── ACK ─▶ Device
（必要时重复 IN 事务，直到短包或达到 wLength）
```

### 控制写（Host → Device），例如 SET_DESCRIPTOR（少用）或类请求写

```
Host ── OUT ──▶ Device
Host ── DATA1/DATA0 + payload ──▶ Device
Host ◀─ ACK/NAK/STALL ── Device
```

**数据切换（Data Toggle）**：控制传输在 Setup 的 DATA0 之后，Data Stage 第一个数据包从 **DATA1** 开始，之后 DATA0/DATA1 交替。

### 短包与 ZLP（Zero-Length Packet）——枚举里极常见却易错

设 `MPS = bMaxPacketSize0`，`N = min(实际可提供长度, wLength)`：

| 情况 | Data Stage 如何结束 |
|------|---------------------|
| 最后一包长度 **&lt; MPS**（短包） | Host 认为传输结束，进入 Status |
| 最后一包长度 **恰好 = MPS**，且已送满 N | 若 N 是 MPS 的整数倍，设备常需再发一个 **ZLP（0 字节 DATA）** 作为结束标志；是否必须取决于 Host 是否还在继续 IN，以及栈对“精确满包”的处理 |
| 已送字节数达到 `wLength` | Host 侧以 `wLength` 为上限结束 Data；即使没有短包也可能直接进 Status（**Host 行为**） |

实务建议（Device 固件）：

1. 发送长度始终取 `min(avail, wLength)`  
2. 按 MPS 拆包；若 `min(avail,wLength)` 恰好整除 MPS，且 Host 可能继续 IN，准备好 **ZLP**（许多 UDC 提供“发零长度包”寄存器位）  
3. 枚举阶段最常见：`wLength=18`、`MPS=8` → `8+8+2`，最后一包是短包，**不需要** ZLP；`wLength=8`、`MPS=8` → 一包满包且达到 wLength，Host 通常直接结束  

> **标准语义**：短包表示“数据结束”；Host 用 `wLength` 限制上限。ZLP 的具体触发条件在不同 Host 栈上略有差异，属实现细节——固件应对“满包后再来一次 IN”的情况能回 ZLP 或 NAK 直至 ZLP，避免 Host 超时。

---

## 3.5 Status Stage 的总线过程

以控制读为例（GET_DESCRIPTOR）：

```
Host ── OUT token ──▶ Device
Host ── DATA1 + 0 字节 ──▶ Device
Host ◀─ ACK ── Device      // 成功
```

以 SET_ADDRESS 为例（无 Data）：

```
Host ── IN token ──▶ Device
Host ◀─ DATA1 + 0 字节 ── Device
Host ── ACK ──▶ Device
```

若设备在 Status 返回 **STALL**，表示请求失败（不支持/错误参数等）。Host 通常会重试或放弃枚举。

---

## 3.6 Setup Packet 的 8 字节结构

```
偏移:  0        1        2    3    4    5    6    7
     ┌────────┬────────┬────────┬────────┬────────┐
     │bmReqTyp│bRequest│ wValue │ wIndex │wLength │
     │  1B    │  1B    │  2B LE │  2B LE │ 2B LE  │
     └────────┴────────┴────────┴────────┴────────┘
```

所有多字节字段均为 **小端（Little-Endian）**。

---

## 3.7 逐字段说明

### 3.7.1 bmRequestType（1 字节）位域

```
 7 6 5 4 3 2 1 0
 ┌─┬───┬─────────┐
 │D│Type│ Recipient│
 └─┴───┴─────────┘
```

| 位 | 字段 | 取值 |
|----|------|------|
| 7 | **Direction** | 0=Host→Device（OUT），1=Device→Host（IN） |
| 6..5 | **Type** | 0=Standard，1=Class，2=Vendor，3=Reserved |
| 4..0 | **Recipient** | 0=Device，1=Interface，2=Endpoint，3=Other |

常见组合：

| 值 | 含义 | 典型请求 |
|----|------|----------|
| `0x80` | IN, Standard, Device | GET_DESCRIPTOR / GET_STATUS(设备) |
| `0x00` | OUT, Standard, Device | SET_ADDRESS / SET_CONFIGURATION |
| `0x02` | OUT, Standard, Endpoint | CLEAR_FEATURE(ENDPOINT_HALT) |
| `0x82` | IN, Standard, Endpoint | GET_STATUS(端点) |
| `0xA1` | IN, Class, Interface | HID GET_REPORT 等 |
| `0x21` | OUT, Class, Interface | HID SET_REPORT 等 |

### 3.7.2 bRequest（1 字节）

标准请求码（Type=Standard 时）：

| bRequest | 码 | 名称 |
|----------|----|------|
| GET_STATUS | 0 | 读状态 |
| CLEAR_FEATURE | 1 | 清特性 |
| SET_FEATURE | 3 | 置特性 |
| SET_ADDRESS | 5 | 设地址 |
| GET_DESCRIPTOR | 6 | 读描述符 |
| SET_DESCRIPTOR | 7 | 写描述符（少用） |
| GET_CONFIGURATION | 8 | 读当前配置值 |
| SET_CONFIGURATION | 9 | 设配置 |
| GET_INTERFACE | 10 | 读备用接口设置 |
| SET_INTERFACE | 11 | 设备用接口 |
| SYNCH_FRAME | 12 | 同步帧（等时） |

### 3.7.3 wValue（2 字节）

依请求而变：

| 请求 | wValue 含义 |
|------|-------------|
| SET_ADDRESS | 新地址（0~127） |
| SET_CONFIGURATION | `bConfigurationValue` |
| GET_DESCRIPTOR | 高字节=Descriptor Type，低字节=Index |
| SET/CLEAR_FEATURE | Feature Selector |
| GET_STATUS | 常为 0 |

Descriptor Type 常用值：`1=DEVICE`，`2=CONFIGURATION`，`3=STRING`，`4=INTERFACE`，`5=ENDPOINT`，`0x21=HID`（类定义）等。

### 3.7.4 wIndex（2 字节）

| 场景 | wIndex |
|------|--------|
| 设备级请求 | 通常 0 |
| 接口级 | Interface Number |
| 端点级 | Endpoint Address（含方向位） |
| 字符串描述符 | Language ID（如 `0x0409` en-US） |

### 3.7.5 wLength（2 字节）

Host 期望在 Data Stage 传输的字节数。

- 控制读：设备返回的数据 ≤ wLength；不足时以**短包**（长度 &lt; MPS）结束
- 控制写：Host 将发送的数据字节数
- 为 0：无 Data Stage

---

## 3.8 标准请求完整交互分析

### 3.8.1 GET_DESCRIPTOR（控制读）

**Setup 示例**：读 18 字节 Device Descriptor

```text
80 06 00 01 00 00 12 00
│  │  │     │     └─ wLength = 18
│  │  │     └─ wIndex = 0
│  │  └─ wValue = 0x0100 (DEVICE, index 0)
│  └─ bRequest = 6 GET_DESCRIPTOR
└─ bmRequestType = 0x80
```

**阶段**：

1. Setup：DATA0 + 上表 8 字节 → ACK  
2. Data：若干 IN 事务，首包 DATA1，返回描述符  
3. Status：OUT + 零长度 DATA1 → 设备 ACK  

**设备可 STALL 的典型原因**：不支持的描述符类型/索引、错误状态。

### 3.8.2 SET_ADDRESS（无 Data）

```text
00 05 05 00 00 00 00 00
│  │  └─ 地址 5
│  └─ bRequest = 5
└─ bmRequestType = 0x00
```

**阶段**：Setup → Status(IN 零长度)  
**关键**：Status 成功前仍用地址 0；成功后启用地址 5。

### 3.8.3 SET_CONFIGURATION（无 Data）

```text
00 09 01 00 00 00 00 00
      └─ bConfigurationValue = 1
```

**阶段**：Setup → Status(IN 零长度)  
**成功后**：Configured；使能配置内端点。  
**wValue=0**：取消配置，回到 Address 状态。

### 3.8.4 GET_STATUS

设备状态（2 字节）：

```text
80 00 00 00 00 00 02 00
```

返回示例：`01 00` → bit0=Self Powered，bit1=Remote Wakeup 等（按规范位定义）。

端点状态：

```text
82 00 00 00 00 81 02 00
               └─ wIndex = EP1 IN (0x81)
```

返回 `01 00` 表示该端点 Halt（Stall）中。

### 3.8.5 CLEAR_FEATURE

清除端点 Halt（恢复批量/中断端点）：

```text
02 01 00 00 00 81 00 00
│  │  │        └─ Endpoint 0x81
│  │  └─ Feature = ENDPOINT_HALT (0)
│  └─ CLEAR_FEATURE (1)
└─ OUT, Standard, Endpoint
```

**阶段**：Setup → Status(IN 零长度)  
固件侧通常：清除 UDC 的 Stall 位，并重置该端点的 Data Toggle 为 DATA0（规范对 Clear Halt 有 toggle 复位要求）。

---

## 3.9 PID 在控制传输中的作用

### Token PID

| PID | 作用 |
|-----|------|
| **SETUP** | 标识 Setup 事务；后跟 8 字节 Setup 数据 |
| **OUT** | Host→Device 数据事务或 Status |
| **IN** | 请求 Device→Host 数据或 Status |
| **SOF** | 帧定时，不属于某次控制传输本身 |

### Data PID

| PID | 作用 |
|-----|------|
| **DATA0** | Setup 阶段固定使用；此后与 DATA1 交替 |
| **DATA1** | Data/Status 阶段常用起始数据 PID |

若 Toggle 失步，接收方会忽略数据或导致重传失败——固件与主机栈都必须正确维护。

### Handshake PID

| PID | 含义 | 控制传输中常见场景 |
|-----|------|-------------------|
| **ACK** | 成功接收 | Setup/Data/Status 正常完成 |
| **NAK** | 暂时未就绪 | 设备数据未准备好、缓冲区满；Host 稍后重试 |
| **STALL** | 请求失败/端点停顿 | 不支持的请求、非法参数、功能错误 |
| **NYET**（HS） | 已收数据但尚不可再收 | HS 特有优化握手 |

### 设备固件如何选择握手？

| 情况 | 通常返回 |
|------|----------|
| Setup 已收，正在准备 IN 数据 | 对提前到的 IN 先 **NAK**，就绪后发 DATA |
| 请求不支持 | 在 Data 或 Status 返回 **STALL** |
| OUT 数据已收且处理完 | **ACK** |
| 控制管道协议错误 | STALL；下一 Setup 可恢复控制管道 |

---

## 3.10 控制传输与枚举的对应关系

| 枚举步骤 | 控制传输类型 | Data Stage |
|----------|--------------|------------|
| GET_DESCRIPTOR | 控制读 | 有（IN） |
| SET_ADDRESS | 无数据控制 | 无 |
| SET_CONFIGURATION | 无数据控制 | 无 |
| GET_STATUS | 控制读 | 有（2 字节） |
| CLEAR_FEATURE | 无数据控制 | 无 |

---

## 3.11 实现层面注意（标准 vs 芯片）

| 项目 | USB 标准 | 典型实现 |
|------|----------|----------|
| Setup 8 字节内容 | 固定布局 | UDC 写入 Setup FIFO/寄存器，触发 Setup 中断 |
| ACK/NAK 时机 | 协议语义 | 常由 UDC 硬件自动 ACK Setup；应用负责尽快填 IN 缓冲 |
| STALL | 语义 | 软件置 Stall 位，硬件发 STALL PID |
| Data Toggle | 必须正确 | 多数 UDC 自动维护 EP0 toggle；厂商文档需确认 |
| 地址生效点 | Status 后 | 必须在 Status 完成回调里写地址寄存器 |

下一章把描述符字节级拆开：[第4章 USB Descriptor](./chapter4.md)
