# 第8章 知识总结与学习检验

---

## 8.1 USB 枚举完整流程图

```text
                    ┌──────────────┐
                    │ 设备物理插入  │
                    └──────┬───────┘
                           ▼
                    ┌──────────────┐
                    │ VBUS 上电     │
                    │ 使能 D+/D-上拉│
                    └──────┬───────┘
                           ▼
                    ┌──────────────┐
                    │ Host 检测连接 │
                    │ 去抖/等待     │
                    └──────┬───────┘
                           ▼
                    ┌──────────────┐
                    │  Bus Reset   │──── HS 设备可 Chirp 升速
                    └──────┬───────┘
                           ▼
              ┌────────────────────────┐
              │ Default 状态, Address=0 │
              └────────────┬───────────┘
                           ▼
              ┌────────────────────────┐
              │ GET_DESCRIPTOR(Device) │
              │ 常先 wLength=8 得 MPS0 │
              └────────────┬───────────┘
                           ▼
              ┌────────────────────────┐
              │ (可选) 再次 Bus Reset  │
              └────────────┬───────────┘
                           ▼
              ┌────────────────────────┐
              │ SET_ADDRESS(n)         │
              │ Status 仍在 Addr0 完成 │
              └────────────┬───────────┘
                           ▼
              ┌────────────────────────┐
              │ Address 状态, Addr=n   │
              └────────────┬───────────┘
                           ▼
              ┌────────────────────────┐
              │ 完整 Device Descriptor │
              └────────────┬───────────┘
                           ▼
              ┌────────────────────────┐
              │ Configuration 集合     │
              │ (先 9 再 wTotalLength) │
              └────────────┬───────────┘
                           ▼
              ┌────────────────────────┐
              │ 可选 String / 其它 Desc│
              └────────────┬───────────┘
                           ▼
              ┌────────────────────────┐
              │ SET_CONFIGURATION(v)   │
              └────────────┬───────────┘
                           ▼
              ┌────────────────────────┐
              │ Configured 状态        │
              │ 类驱动 + 非零端点通信   │
              └────────────────────────┘
```

---

## 8.2 USB 标准请求速查表（枚举常用）

| bRequest | 码 | bmRequestType 常见 | wValue | wIndex | wLength | Data |
|----------|----|--------------------|--------|--------|---------|------|
| GET_STATUS | 0 | `0x80`/`0x81`/`0x82` | 0 | 0/接口/端点 | 2 | IN 2B |
| CLEAR_FEATURE | 1 | `0x00`/`0x02` 等 | Feature | 0/端点 | 0 | 无 |
| SET_FEATURE | 3 | 同上方向 OUT | Feature | 视情况 | 0 | 无 |
| SET_ADDRESS | 5 | `0x00` | 新地址 | 0 | 0 | 无 |
| GET_DESCRIPTOR | 6 | `0x80`（或接口 Recipient） | Type≪8 \| Index | 0 或 LangID | N | IN |
| GET_CONFIGURATION | 8 | `0x80` | 0 | 0 | 1 | IN 1B |
| SET_CONFIGURATION | 9 | `0x00` | ConfigValue | 0 | 0 | 无 |
| GET_INTERFACE | 10 | `0x81` | 0 | 接口号 | 1 | IN 1B |
| SET_INTERFACE | 11 | `0x01` | AltSetting | 接口号 | 0 | 无 |

**Setup 八字节布局**：`bmRequestType | bRequest | wValue(LE) | wIndex(LE) | wLength(LE)`

---

## 8.3 USB Descriptor 层级关系图

```text
Device Descriptor (类型 0x01, 18 字节)
│
├─ idVendor / idProduct / bMaxPacketSize0 / bNumConfigurations ...
│
├─ Configuration Descriptor #1 (类型 0x02)  ◀─ bConfigurationValue
│   │   wTotalLength = 本配置所有描述符字节总和
│   │
│   ├─ Interface Descriptor #0 (类型 0x04)
│   │    ├─ [HID Descriptor 0x21 ...]
│   │    └─ Endpoint Descriptor(s) (类型 0x05)
│   │
│   └─ Interface Descriptor #1
│        └─ Endpoint Descriptor(s)
│
├─ Configuration Descriptor #2 (可选)
│
└─ String Descriptors (类型 0x03) ← 由 iManufacturer 等索引引用
         ├─ Index0: Language ID 列表
         └─ IndexN: UTF-16LE 字符串
```

---

## 8.4 Host 与 Device 枚举职责对照表

| 阶段 | Host 职责 | Device 职责 |
|------|-----------|-------------|
| Attach | 检测端口 Connect，去抖 | 上电，使能正确上拉 |
| Reset | 驱动 SE0≥10 ms；HS 时参与 Chirp | 识别 Reset，回 Default，地址 0 |
| 读 Desc 8B | 发 GET_DESCRIPTOR，记录 MPS0 | 返回 Device Desc 前缀 |
| SET_ADDRESS | 分配并发送新地址 | Status 后启用新地址，进 Address |
| 读完整描述符 | 解析 Device/Config 集合 | 按类型返回描述符数据 |
| SET_CONFIGURATION | 选择配置值 | 使能端点，进 Configured |
| 枚举后 | 绑定类驱动，发起功能传输 | 响应类请求与非零端点传输 |
| 任意时刻再 Reset | 可重新枚举 | 必须可重入回到 Default |

---

## 8.5 最容易混淆的 10 个知识点

1. **读了 Configuration Descriptor ≠ 已配置**  
   必须 `SET_CONFIGURATION` 后才进入 Configured，功能端点才可用。

2. **`SET_ADDRESS` 的 Status 仍在地址 0**  
   新地址在 Status 成功之后才生效；提前改地址是高频 bug。

3. **`wValue`/`wIndex`/`wLength` 都是小端**  
   线上 `12 00` 表示 0x0012，不是 0x1200。

4. **`bMaxPacketSize0` 与速度强相关**  
   HS 必须 64；乱填会导致后续控制传输失败。

5. **Endpoint 的 IN/OUT 是 Host 视角**  
   IN = 设备→Host；OUT = Host→设备。和 MCU“串口 TX/RX”直觉容易反。

6. **Interface 的 `bNumEndpoints` 不含 EP0**  
   只有非零端点计数。

7. **`wTotalLength` 包含配置头自己**  
   是整包集合长度，不是“后续部分长度”。

8. **`bMaxPower` 单位是 2 mA（USB 2.0）**  
   `0x32` = 50×2 = 100 mA；不要按 1 mA 或 USB3 的 8 mA 理解。

9. **NAK ≠ 失败**  
   NAK 表示暂时未就绪，Host 会重试；STALL 才是明确失败/停顿。

10. **二次 Bus Reset 是 Host 策略不是规范强制步骤**  
    设备必须容忍；不要写死“Reset 只会发生一次”的逻辑。

---

## 8.6 学习检验：5 道由浅入深的技术题

### 题 1（基础）

设备 FS 插入后，Host 为什么能区分这是 Full-Speed 而不是 Low-Speed？若要做成 LS 设备，硬件上要改什么？

<details>
<summary>参考答案</summary>

FS 设备上拉 **D+**，LS 上拉 **D−**。Host 通过检测哪一根数据线被拉高来初判速度。做成 LS 需把 1.5 kΩ 上拉改到 D−（并满足 LS 电气与描述符/`bMaxPacketSize0=8` 等约束）。

</details>

### 题 2（协议）

下面 Setup 包在做什么？设备应如何分阶段响应？

```text
00 05 07 00 00 00 00 00
```

<details>
<summary>参考答案</summary>

`SET_ADDRESS(7)`：OUT/Standard/Device，无 Data Stage。  
设备：Setup 收齐并 ACK → 用地址 **0** 完成 Status（IN 零长度）→ **之后**才把地址寄存器改为 7，进入 Address 状态。

</details>

### 题 3（描述符）

配置集合抓包如下（十六进制），请指出 `wTotalLength`、接口类、端点地址与类型，并判断 `wTotalLength` 是否与各 `bLength` 之和一致：

```text
09 02 19 00 01 01 00 80 32
09 04 00 00 01 08 06 50 00
07 05 81 02 40 00 00
07 05 01 02 40 00 00
```

<details>
<summary>参考答案</summary>

- `wTotalLength = 0x0019` = 25  
- 长度和：9+9+7+7 = 32 ≠ 25 → **不一致，异常**（常见固件错误：改了端点却忘了改 wTotalLength；或漏字节）  
- 接口类 `0x08`：Mass Storage；SubClass `0x06`，Protocol `0x50`（常见 BBB）  
- 端点：`0x81` Bulk IN，`0x01` Bulk OUT，MPS=64  

（本题刻意给出错误总长，用于训练校验能力。）

</details>

### 题 4（控制传输）

控制读 `GET_DESCRIPTOR` 成功时，Status Stage 为什么是 **OUT + 零长度**，而 `SET_CONFIGURATION` 的 Status 却是 **IN + 零长度**？

<details>
<summary>参考答案</summary>

规则：Status 数据方向与 Data Stage **相反**；若无 Data Stage，则 Status 使用 **IN**。  
`GET_DESCRIPTOR` 有 IN 数据，故 Status 为 OUT 零长度；`SET_CONFIGURATION` 无 Data，故 Status 为 IN 零长度。

</details>

### 题 5（综合排障）

抓包显示：

1. `GET_DESCRIPTOR` 8 字节成功，`bMaxPacketSize0=64`  
2. `SET_ADDRESS(5)` 的 Setup 在 ADDR0 成功  
3. Status 阶段 Host 发 IN ADDR0，设备无应答，最终超时  
4. Host 放弃枚举  

请给出最可能的固件缺陷，并写出正确处理顺序。

<details>
<summary>参考答案</summary>

最可能：设备在 **Setup 阶段就写入了地址 5**，导致 Status 阶段仍在 ADDR0 上的 IN 无人应答。  

正确顺序：Setup 中只保存 `pending_addr=5` → 完成 Status（ADDR0）→ 在 Status 完成回调里 `udc_set_address(5)` → 进入 Address 状态。

</details>

---

## 8.7 继续学习建议

1. 用 FS 设备实机抓一条完整枚举，按第6章表格逐项标注。  
2. 在 Device 固件里对每次 Setup 打印八字节与处理结果（STALL/长度）。  
3. 阅读 USB 2.0 规范：Chapter 9（Device Framework）与控制传输相关章节；HID 再读 HID 1.11。  
4. 对照你使用的 MCU UDC 手册，把本文“标准行为”映射到具体寄存器。  

返回目录：[USB 枚举教程首页](./README.md)
