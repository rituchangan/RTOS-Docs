# 第6章 USB 抓包实战分析

本章构造一个**典型 USB Full-Speed HID 设备**从插入到 `SET_CONFIGURATION` 完成的枚举抓包记录。包序按教学简化，字段符合 USB 2.0 控制传输规则。

> 真实 OS 可能插入额外的 `GET_STATUS`、二次 Reset、String 读取、Hub 请求等；以你实际分析仪记录为准，用本章对照字段含义。

---

## 6.1 案例设定

| 项目 | 值 |
|------|-----|
| 速度 | Full-Speed |
| EP0 MPS | 64（`0x40`） |
| 分配地址 | 3 |
| 配置值 | 1 |
| 设备类 | 接口级 HID |

时间轴符号：

- `H→D` Host 到 Device  
- `D→H` Device 到 Host  

---

## 6.2 按时间顺序的通信记录

### A. 连接与复位（无 PID 数据阶段）

```text
T+0 ms     电气: D+ 上拉，Host 检测 FS Connect
T+100 ms   Host 去抖结束
T+100 ms   Host Port Reset: SE0 ≈ 10~50 ms
T+140 ms   设备进入 Default，地址=0
T+141 ms   总线开始出现 SOF（每 1 ms）
```

### B. GET_DESCRIPTOR(Device)，wLength=8，地址 0

**Setup Packet 十六进制：**

```text
80 06 00 01 00 00 08 00
```

**事务级记录：**

```text
1  H→D  SETUP  ADDR=0 EP=0
2  H→D  DATA0  [80 06 00 01 00 00 08 00]   CRC16
3  D→H  ACK

4  H→D  IN     ADDR=0 EP=0
5  D→H  DATA1  [12 01 00 02 00 00 00 40]   // 前 8 字节
6  H→D  ACK

7  H→D  OUT    ADDR=0 EP=0
8  H→D  DATA1  []  (0 字节，Status)
9  D→H  ACK
```

**字段解析：**

| 字段 | 原始 | 解析 |
|------|------|------|
| bmRequestType | `80` | IN, Standard, Device |
| bRequest | `06` | GET_DESCRIPTOR |
| wValue | `00 01` | Type=Device(1), Index=0 |
| wIndex | `00 00` | 0 |
| wLength | `08 00` | 8 |

返回数据：`bMaxPacketSize0 = 0x40`。

### C. （常见）再次 Bus Reset

```text
Host Port Reset SE0
设备再次进入 Default，地址=0
```

### D. SET_ADDRESS(3)

**Setup：**

```text
00 05 03 00 00 00 00 00
```

```text
1  H→D  SETUP  ADDR=0 EP=0
2  H→D  DATA0  [00 05 03 00 00 00 00 00]
3  D→H  ACK

4  H→D  IN     ADDR=0 EP=0          // Status，无 Data Stage
5  D→H  DATA1  []
6  H→D  ACK

// 此后设备启用地址 3
```

| 字段 | 值 | 含义 |
|------|-----|------|
| bmRequestType | `00` | OUT, Standard, Device |
| bRequest | `05` | SET_ADDRESS |
| wValue | `03 00` | 地址 3 |
| wLength | `00 00` | 无 Data |

### E. GET_DESCRIPTOR(Device)，18 字节，地址 3

**Setup：**

```text
80 06 00 01 00 00 12 00
```

```text
SETUP+DATA0+ACK   (ADDR=3)
IN + DATA1 + 18 字节 + ACK
OUT + DATA1 0B + ACK     // Status
```

**返回数据示例：**

```text
12 01 00 02 00 00 00 40 5E 04 01 C0 00 01 01 02 03 01
```

若 MPS=8，则 18 字节会拆成 `8 + 8 + 2` 三次 IN；本例 MPS=64，一次 DATA 即可。

### F. GET_DESCRIPTOR(Configuration)，先 9 后全长

**第一次 Setup：**

```text
80 06 00 02 00 00 09 00
```

返回 9 字节示例：

```text
09 02 22 00 01 01 00 A0 32
```

→ `wTotalLength = 0x0022`。

**第二次 Setup：**

```text
80 06 00 02 00 00 22 00
```

返回 34 字节配置集合（见第4章示例布局）。

### G. SET_CONFIGURATION(1)

**Setup：**

```text
00 09 01 00 00 00 00 00
```

```text
SETUP+DATA0+ACK   (ADDR=3)
IN + DATA1 0B + ACK       // Status
```

之后设备 **Configured**，可见类请求与 Interrupt IN 轮询。

---

## 6.3 与第2章流程的对应

| 抓包段落 | 流程阶段 | 状态/地址 |
|----------|----------|-----------|
| A | Attach + Reset | → Default / 0 |
| B | 初始 Device Desc 8B | Default / 0 |
| C | 二次 Reset | Default / 0 |
| D | SET_ADDRESS | → Address / 3 |
| E | 完整 Device Desc | Address / 3 |
| F | Configuration 集合 | Address / 3 |
| G | SET_CONFIGURATION | → Configured / 3 |

---

## 6.4 在 Wireshark / 协议分析仪中关注什么

### Wireshark（usbmon / USBPcap）

适合看**主机栈视角**的 URB，不一定显示每次 ACK/NAK 电气细节。

重点列：

- `bmRequestType` / `bRequest` / `wValue` / `wIndex` / `wLength`
- `GET DESCRIPTOR` 响应中的描述符树展开
- URB 状态：`Success` / `Stalled` / `Timeout`
- 设备地址变化是否出现在 `SET ADDRESS` 之后

显示过滤器示例：

```text
usb.bmRequestType_tree || usb.setup.bRequest
usb.bDescriptorType
usb.urb_status
```

### 硬件 Protocol Analyzer（Ellisys / Total Phase / 乐得瑞等）

适合看**总线级** PID、Chirp、Reset、NAK 密度。

重点：

1. **Reset 时长**是否 ≥10 ms  
2. Setup 是否为 **DATA0**，随后 Data 是否从 **DATA1** 起  
3. `SET_ADDRESS` 的 Status 是否仍在 **ADDR=0**  
4. 之后 Token 的 ADDR 是否改为新地址  
5. 出现 **STALL** 的阶段（Data 还是 Status）  
6. 连续 **NAK** 是否过久（设备响应慢或 FIFO 未装载）  
7. 配置描述符 `wTotalLength` 与实际 Data 总量是否一致  

### 建议对照检查表

| 检查项 | 正常期望 |
|--------|----------|
| 第一个 SETUP | ADDR=0，`80 06 …` |
| 第一个 Data 返回 | 至少含 `bMaxPacketSize0` |
| SET_ADDRESS | `00 05 …`，Status 在 ADDR=0 |
| 下一 SETUP | ADDR=新地址 |
| SET_CONFIGURATION | `00 09 …`，随后可有非零 EP 流量 |

---

## 6.5 把一次失败抓包读薄的方法

1. 找到**最后一次成功的标准请求**  
2. 看下一次是 STALL、超时还是无响应  
3. 若 STALL：解析该次 Setup 八字节，对照设备是否实现该请求  
4. 若超时：查 Reset 后时钟/上拉/地址是否改早  
5. 若描述符阶段失败：把 Data 载荷按 `bLength` 手工切片，查 `wTotalLength`  

下一章按故障症状排查：[第7章 常见枚举失败](./chapter7.md)
