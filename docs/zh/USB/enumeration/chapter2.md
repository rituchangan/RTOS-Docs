# 第2章 USB 枚举完整流程（重点）

本章按**真实时间顺序**，从设备插入讲到枚举完成。每个阶段都说明：双方行为、电气/协议交互、标准请求、设备状态与地址、相关包结构。

> 下文默认场景：**USB 2.0 Full-Speed 设备直连 Host Root Hub 端口**。HS 差异在相应步骤补充；LS 主要差在上拉脚位与包时序。

---

## 阶段总时间线（鸟瞰）

```
t0  插入，VBUS 上电，设备拉起 D+（FS）
t1  Host 检测到 Connect
t2  去抖 / 等待电源稳定（常见 ≥100 ms）
t3  Port Reset（SE0 ≥10 ms）
t4  设备 → Default，地址 0
t5  GET_DESCRIPTOR(Device)，常只要前 8 字节
t6  （常见）再次 Port Reset
t7  SET_ADDRESS(addr)
t8  设备 → Address，后续用新地址
t9  GET_DESCRIPTOR(Device) 完整 18 字节
t10 GET_DESCRIPTOR(Configuration) 先读 9 字节拿 wTotalLength
t11 再读完整配置集合（Configuration+Interface+Endpoint+…）
t12 可选：GET_DESCRIPTOR(String)
t13 SET_CONFIGURATION(1)
t14 设备 → Configured，类驱动开始使用非零端点
```

---

## 2.1 设备连接检测（Attach Detection）

### 当前阶段发生了什么？

插头插入后，设备从 VBUS 获得 5 V（经 LDO 等到 3.3 V 给 PHY/逻辑）。FS 设备将 **D+** 经约 **1.5 kΩ** 上拉到 3.3 V，使差分线呈现 **J 状态**（对 FS 而言 D+ 高、D− 低），从而离开空闲未连接时的 **SE0**（D+、D− 都近 0）。

### Host 和 Device 分别做什么？

| 角色 | 行为 |
|------|------|
| **Device** | 使能上拉（有的芯片需固件写寄存器才上拉；有的上电自动上拉）。此时尚无地址、未 Reset，**不应**主动发数据包。 |
| **Host / Hub** | 端口状态机采样 D+/D−：发现 Connect Status 变化，置位连接检测中断/状态位。 |

### 总线上的电气交互

- 未连接：下行口通常看到 SE0
- FS 连接：D+ 被上拉 → Host 判为 Full-Speed 连接
- LS 连接：D− 被上拉 → Host 判为 Low-Speed 连接

### 标准请求？

**无。** 此时还不能发令牌包进行枚举请求。

### 设备状态与地址

- 状态：Attached →（获得 VBUS 后）**Powered**
- 地址：尚未有效参与协议，逻辑上仍未进入 Default

### 实现注意（非规范强制的主机策略）

许多 Host 栈在检测到连接后会：

1. 等待一段时间做**机械去抖**与电容充电（USB 2.0 对主机侧常见实践是约 **100 ms** 量级，具体以实现为准）
2. 再发起 Port Reset

若设备上拉过早而内部时钟/UDC 未就绪，可能出现“检测到了但 Reset 后无响应”——见第7章。

---

## 2.2 Bus Reset（总线复位）

### 当前阶段发生了什么？

Host 驱动 Hub 端口进入 Reset：将 D+、D− 驱动为 **SE0** 并保持。

按 USB 2.0：

- Reset 驱动 SE0 的时间应 **不小于 10 ms**（规范对复位信号持续时间有下限要求）
- 设备在 Reset 结束后进入 **Default** 状态，地址变为 **0**，端点 FIFO/切换位等回到规定的默认条件

### Host 和 Device 分别做什么？

| 角色 | 行为 |
|------|------|
| **Host** | 对端口写 Reset；延时 ≥10 ms；停止 Reset，释放总线；可选再读端口速度状态 |
| **Device** | 检测 SE0 足够长时间后识别为 Bus Reset；复位 USB 协议引擎；地址寄存器清 0；进入 Default |

### HS 附加：Chirp（仅高速）

若设备是 HS Capable：

1. Reset 的 SE0 期间，设备驱动 **Chirp K**
2. Host 若支持 HS，检测到后回以一串 **K-J-K-J… Chirp**
3. 成功则两端切到 480 Mbps；否则设备以 FS 工作（若支持）

FS/LS 设备无此升速过程。

### 标准请求？

仍无控制请求。Reset 是**端口电气/链路层**行为，不是 Setup 包。

### 设备状态与地址

| 项目 | Reset 前 | Reset 后 |
|------|----------|----------|
| 状态 | Powered（或其它非 Default） | **Default** |
| 地址 | 无意义/旧地址 | **0** |
| 配置 | 若曾配置则丢失 | 未配置 |

### 相关“包”？

Reset 本身不是 PID 包。Reset 结束后，FS 总线上开始出现 Host 的 **SOF**（每 1 ms）以及后续 Token 包。

---

## 2.3 获取初始 Device Descriptor（前 8 字节）

### 为什么先读 8 字节？

Device Descriptor 共 **18 字节**。其中偏移 7 的 **`bMaxPacketSize0`** 决定控制端点最大包长。在地址 0、尚未确知包长时，Host 常用一次 **`wLength = 8`** 的 `GET_DESCRIPTOR`，只取前 8 字节，从而安全获知包长，再配置后续控制传输。

> **标准规定的是描述符布局与请求语义；“先读 8 再读 18”是广泛采用的主机策略，不是唯一合法做法。** 若 Host 已确知包长（例如固定 64 的 HS），也可能直接读 18。

### Host 发送的请求

标准请求 **GET_DESCRIPTOR**：

| 字段 | 值 | 含义 |
|------|-----|------|
| bmRequestType | `0x80` | Device-to-host，Standard，Device |
| bRequest | `0x06` | GET_DESCRIPTOR |
| wValue | `0x0100` | Descriptor Type=DEVICE(1)，Index=0 |
| wIndex | `0x0000` | 语言/接口等，设备描述符为 0 |
| wLength | `0x0008` | 只要 8 字节 |

Setup 包 8 字节十六进制：

```text
80 06 00 01 00 00 08 00
```

（小端：`wValue=0x0100` 在线上为 `00 01`，`wLength=0x0008` 为 `08 00`）

### 总线上的控制传输交互（摘要）

```
Setup Stage:
  Host: SETUP token (ADDR=0, EP=0)
  Host: DATA0 + 8 字节 Setup
  Device: ACK

Data Stage: (IN，设备→Host)
  Host: IN token
  Device: DATA1 + 最多 8 字节描述符前缀
  Host: ACK

Status Stage: (OUT，无数据)
  Host: OUT token
  Host: DATA1 + 0 长度
  Device: ACK
```

（DATA0/DATA1 切换与三阶段细节见第3章。）

### Device 返回的典型前 8 字节示例

```text
12 01 00 02 00 00 00 40
```

| 偏移 | 值 | 字段 | 含义 |
|------|-----|------|------|
| 0 | `12` | bLength | 18 |
| 1 | `01` | bDescriptorType | DEVICE |
| 2-3 | `00 02` | bcdUSB | 0x0200 → USB 2.0 |
| 4 | `00` | bDeviceClass | 在接口中定义 |
| 5 | `00` | bDeviceSubClass | |
| 6 | `00` | bDeviceProtocol | |
| 7 | `40` | bMaxPacketSize0 | **64** |

### 设备状态与地址

- 仍为 **Default / 地址 0**
- 仅完成一次控制读，未改地址、未配置

---

## 2.4 分配 USB 地址（SET_ADDRESS）

### 常见主机额外步骤：二次 Reset

Windows 等主机在读完前 8 字节后，**常常再次 Port Reset**，然后才 `SET_ADDRESS`。目的包括：让设备回到干净的 Default，并带着已知的 `bMaxPacketSize0` 重新开始。

> 二次 Reset **不是** USB 2.0 强制的枚举步骤，但是非常常见的 Host 行为。设备固件必须能容忍多次 Reset。

### Host 发送的请求

| 字段 | 值 | 含义 |
|------|-----|------|
| bmRequestType | `0x00` | Host-to-device，Standard，Device |
| bRequest | `0x05` | SET_ADDRESS |
| wValue | `0x00nn` | 新地址 nn（1~127），例如 `0x0005` |
| wIndex | `0x0000` | |
| wLength | `0x0000` | **无 Data Stage** |

示例（地址 = 5）：

```text
00 05 05 00 00 00 00 00
```

### 关键时序细节（极易错）

USB 2.0 规定：

1. `SET_ADDRESS` **没有 Data Stage**，只有 Setup + Status
2. 设备必须在 **Status Stage 成功完成之后** 才启用新地址
3. Status Stage 期间，设备仍应在 **地址 0** 上响应
4. 之后 Host 用**新地址**访问该设备；地址 0 释放给下一个未枚举设备

### 总线上交互（无 Data Stage）

```
Setup Stage:  ADDR=0, EP=0, SETUP + DATA0(8) → ACK
Status Stage: ADDR=0, EP=0, IN + 0-length DATA1 → ACK
（之后设备启用地址 5）
```

### 设备状态与地址

| 时刻 | 状态 | 地址 |
|------|------|------|
| Setup 收到后、Status 前 | 仍为 Default | 仍按 0 响应 |
| Status 成功完成后 | **Address** | **新地址** |

Device 固件典型写法：先把“待设置地址”存到变量，在 Status 完成中断里再写入 UDC 地址寄存器。

---

## 2.5 获取完整 Device Descriptor

### 发生了什么？

Host 对**新地址**再次 `GET_DESCRIPTOR(DEVICE)`，`wLength = 18`（或更大，设备只返回实际 18 字节）。

Setup 示例（地址已变，但 Setup 数据相同类型）：

```text
80 06 00 01 00 00 12 00
```

### Host / Device

- Host：用新 ADDR 发控制传输；解析 18 字节，得到 VID/PID、设备类、配置数量、字符串索引等
- Device：从描述符数组返回 18 字节；按 `bMaxPacketSize0` 分包（若 18 > MPS 则多个 DATA 事务）

### 完整 18 字节示例

```text
12 01 00 02 00 00 00 40
5E 04 01 C0 00 01 01 02
03 01
```

| 字段 | 示例值 | 含义 |
|------|--------|------|
| idVendor | `0x045E` | Vendor ID |
| idProduct | `0xC001` | Product ID |
| bcdDevice | `0x0100` | 设备版本 1.00 |
| iManufacturer | 1 | 厂商字符串索引 |
| iProduct | 2 | 产品字符串索引 |
| iSerialNumber | 3 | 序列号字符串索引 |
| bNumConfigurations | 1 | 配置个数 |

### 状态

仍为 **Address**；地址不变。

---

## 2.6 获取 Configuration Descriptor 及其关联描述符

### 两步读取策略（常见）

1. 先 `GET_DESCRIPTOR(CONFIGURATION)`，`wLength = 9`，只取配置描述符头，读取 **`wTotalLength`**
2. 再按 `wTotalLength` 发起第二次读，取回 **整包配置集合**

配置集合通常按顺序紧密排列：

```
Configuration Descriptor
  Interface Descriptor 0
    Endpoint Descriptor 1-IN
    Endpoint Descriptor 1-OUT
    （可选 HID Descriptor 等类描述符）
  Interface Descriptor 1
    ...
```

### 第一次请求示例

```text
80 06 00 02 00 00 09 00
```

- `wValue = 0x0200`：Type=CONFIGURATION(2)，Index=0（第一个配置）

返回 9 字节中，偏移 2-3 为 `wTotalLength`（小端）。若为 `0x0029`（41），则第二次 `wLength = 0x0029`。

### 第二次请求示例

```text
80 06 00 02 00 00 29 00
```

Device 返回 41 字节连续描述符流。Host 按 `bLength` / `bDescriptorType` 逐个切片解析。

### Host 从中学到什么？

- 需要多少供电（`bmAttributes`、`bMaxPower`）
- 有几个接口、每个接口的 class/subclass/protocol（决定加载 HID/MSC/CDC…）
- 每个端点的地址、传输类型、MaxPacketSize、间隔

### 状态

仍为 **Address**。读取配置描述符**不会**自动进入 Configured；必须 `SET_CONFIGURATION`。

### 可选：String Descriptor

若 `iManufacturer` 等非 0：

1. 先 `GET_DESCRIPTOR(STRING, index=0)` 获取语言 ID 列表
2. 再对具体 index + `wIndex=语言ID` 读取 UNICODE 字符串

Setup 读语言 ID 示例：

```text
80 06 00 03 00 00 FF 00
```

（`wValue=0x0300`，Type=STRING，Index=0）

---

## 2.7 选择并配置设备（SET_CONFIGURATION）

### Host 发送

| 字段 | 值 |
|------|-----|
| bmRequestType | `0x00` |
| bRequest | `0x09`（SET_CONFIGURATION） |
| wValue | 配置值 = Configuration 描述符中的 **`bConfigurationValue`**（常为 1） |
| wIndex | 0 |
| wLength | 0（无 Data Stage） |

示例：

```text
00 09 01 00 00 00 00 00
```

### Device 行为

- 根据 `bConfigurationValue` 激活对应配置
- 使能该配置下各接口的端点（非零 EP）
- 进入 **Configured** 状态

注意：`wValue = 0` 表示取消配置，设备回到 **Address** 状态（地址保持）。

### 总线交互

与 `SET_ADDRESS` 类似：Setup + Status（IN 零长度），无 Data Stage。

### 状态与地址

| 项目 | 变化 |
|------|------|
| 状态 | Address → **Configured** |
| 地址 | 不变 |
| 端点 0 | 继续可用 |
| 其它端点 | 按描述符可用 |

---

## 2.8 枚举完成后的设备通信

### 发生了什么？

Host 根据接口类信息绑定驱动，例如：

- HID：读取 HID Report Descriptor（仍可能经端点 0 类请求），再周期 Interrupt IN 取报
- MSC：Bulk ONLY 协议，Bulk IN/OUT 传 CBW/数据/CSW
- CDC ACM：控制接口 + Bulk 数据接口

此后总线上除控制传输外，还有：

- 周期性 **SOF**
- Interrupt/Bulk/Iso 事务
- 可选 `GET_STATUS` / `CLEAR_FEATURE(ENDPOINT_HALT)` 做错误恢复

### 枚举“完成”的判据（实务）

对设备固件而言，收到合法的 `SET_CONFIGURATION(非0)` 且 Status 成功，即可认为枚举完成。对 Host 而言，还需类驱动初始化成功，用户态才能看到设备节点。

---

## 2.9 各阶段对照速查

| 阶段 | 标准请求 | Data | 状态 | 地址 |
|------|----------|------|------|------|
| Attach | — | — | Powered | — |
| Reset | — | — | Default | 0 |
| 读 Desc 8B | GET_DESCRIPTOR | IN 8B | Default | 0 |
| （可选）再 Reset | — | — | Default | 0 |
| SET_ADDRESS | SET_ADDRESS | 无 | → Address | 0→N（Status 后） |
| 读 Desc 18B | GET_DESCRIPTOR | IN 18B | Address | N |
| 读 Config | GET_DESCRIPTOR | IN | Address | N |
| SET_CONFIG | SET_CONFIGURATION | 无 | → Configured | N |
| 功能通信 | 类/厂商请求 + 非控制传输 | 视协议 | Configured | N |

---

## 2.10 必须刻意记住的协议细节

1. **只有 Host 发 Token**；设备从不主动“打电话”，只能应答。
2. **地址 0 窗口很短**：用于 Default 态枚举；`SET_ADDRESS` 的 Status 仍在地址 0 完成。
3. **读描述符 ≠ 配置设备**；没 `SET_CONFIGURATION` 就没有功能端点。
4. **`bMaxPacketSize0` 决定控制传输怎么拆包**；固件返回长度与 Host `wLength`、MPS 之间的关系要正确（短包结束）。
5. **Reset 可随时发生**；设备必须从任意状态正确回到 Default/地址 0。
6. **HS Chirp 失败时可能落在 FS**；描述符中 `bcdUSB`/`bMaxPacketSize0` 仍须与真实工作速度匹配。

下一章把枚举全部依赖的控制传输拆到 PID 与 Setup 字段级：[第3章 Control Transfer](./chapter3.md)
