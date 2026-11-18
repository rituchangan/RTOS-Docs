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

## 3.2 Setup Stage 的总线过程

```
Host  ── SETUP token (ADDR, EP=0) ──▶  Device
Host  ── DATA0 + 8 字节 Setup      ──▶  Device
Host  ◀──────── ACK ────────────────  Device
```

要点：

1. Setup 数据**永远用 DATA0** PID（规范固定）
2. 设备必须接受 Setup（即使曾 STALL，Setup 也会清除某些停顿条件——具体见端点 Halt 与控制管道规则）
3. 设备用 **ACK** 确认已收到 Setup；若未就绪，行为依赖控制器实现，但协议期望尽快接受 Setup

---

## 3.3 Data Stage 的总线过程

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

---

## 3.4 Status Stage 的总线过程

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

## 3.5 Setup Packet 的 8 字节结构

```
偏移:  0        1        2    3    4    5    6    7
     ┌────────┬────────┬────────┬────────┬────────┐
     │bmReqTyp│bRequest│ wValue │ wIndex │wLength │
     │  1B    │  1B    │  2B LE │  2B LE │ 2B LE  │
     └────────┴────────┴────────┴────────┴────────┘
```

所有多字节字段均为 **小端（Little-Endian）**。

---

## 3.6 逐字段说明

### 3.6.1 bmRequestType（1 字节）位域

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

### 3.6.2 bRequest（1 字节）

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

### 3.6.3 wValue（2 字节）

依请求而变：

| 请求 | wValue 含义 |
|------|-------------|
| SET_ADDRESS | 新地址（0~127） |
| SET_CONFIGURATION | `bConfigurationValue` |
| GET_DESCRIPTOR | 高字节=Descriptor Type，低字节=Index |
| SET/CLEAR_FEATURE | Feature Selector |
| GET_STATUS | 常为 0 |

Descriptor Type 常用值：`1=DEVICE`，`2=CONFIGURATION`，`3=STRING`，`4=INTERFACE`，`5=ENDPOINT`，`0x21=HID`（类定义）等。

### 3.6.4 wIndex（2 字节）

| 场景 | wIndex |
|------|--------|
| 设备级请求 | 通常 0 |
| 接口级 | Interface Number |
| 端点级 | Endpoint Address（含方向位） |
| 字符串描述符 | Language ID（如 `0x0409` en-US） |

### 3.6.5 wLength（2 字节）

Host 期望在 Data Stage 传输的字节数。

- 控制读：设备返回的数据 ≤ wLength；不足时以**短包**（长度 &lt; MPS）结束
- 控制写：Host 将发送的数据字节数
- 为 0：无 Data Stage

---

## 3.7 标准请求完整交互分析

### 3.7.1 GET_DESCRIPTOR（控制读）

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

### 3.7.2 SET_ADDRESS（无 Data）

```text
00 05 05 00 00 00 00 00
│  │  └─ 地址 5
│  └─ bRequest = 5
└─ bmRequestType = 0x00
```

**阶段**：Setup → Status(IN 零长度)  
**关键**：Status 成功前仍用地址 0；成功后启用地址 5。

### 3.7.3 SET_CONFIGURATION（无 Data）

```text
00 09 01 00 00 00 00 00
      └─ bConfigurationValue = 1
```

**阶段**：Setup → Status(IN 零长度)  
**成功后**：Configured；使能配置内端点。  
**wValue=0**：取消配置，回到 Address 状态。

### 3.7.4 GET_STATUS

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

### 3.7.5 CLEAR_FEATURE

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

## 3.8 PID 在控制传输中的作用

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

## 3.9 控制传输与枚举的对应关系

| 枚举步骤 | 控制传输类型 | Data Stage |
|----------|--------------|------------|
| GET_DESCRIPTOR | 控制读 | 有（IN） |
| SET_ADDRESS | 无数据控制 | 无 |
| SET_CONFIGURATION | 无数据控制 | 无 |
| GET_STATUS | 控制读 | 有（2 字节） |
| CLEAR_FEATURE | 无数据控制 | 无 |

---

## 3.10 实现层面注意（标准 vs 芯片）

| 项目 | USB 标准 | 典型实现 |
|------|----------|----------|
| Setup 8 字节内容 | 固定布局 | UDC 写入 Setup FIFO/寄存器，触发 Setup 中断 |
| ACK/NAK 时机 | 协议语义 | 常由 UDC 硬件自动 ACK Setup；应用负责尽快填 IN 缓冲 |
| STALL | 语义 | 软件置 Stall 位，硬件发 STALL PID |
| Data Toggle | 必须正确 | 多数 UDC 自动维护 EP0 toggle；厂商文档需确认 |
| 地址生效点 | Status 后 | 必须在 Status 完成回调里写地址寄存器 |

下一章把描述符字节级拆开：[第4章 USB Descriptor](./chapter4.md)
