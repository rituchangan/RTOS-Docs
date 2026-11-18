# 第4章 深入分析 USB Descriptor

描述符是设备的“身份证与能力清单”。Host 完全依靠描述符决定驱动匹配与管道创建。

---

## 4.1 描述符层级关系

```
Device Descriptor                          （设备唯一，18 字节）
 └── Configuration Descriptor × N          （用 bConfigurationValue 选择）
      ├── Interface Descriptor × M
      │    ├── （可选）Class Descriptor，如 HID Descriptor
      │    │     └── （经类请求再取）Report Descriptor
      │    └── Endpoint Descriptor × K
      └── （同配置下更多 Interface …）
String Descriptor × …                      （通过索引引用，可选）
```

要点：

- **Device** 描述“这是谁、有几个配置”
- **Configuration** 描述“一种工作模式（功耗+接口集合）”
- **Interface** 描述“一个功能类（HID/MSC/…）”
- **Endpoint** 描述“该功能用哪些管道”
- **String** 不参与结构树，只被其它描述符里的 `iXxx` 索引引用

Host 识别流程：

1. Device Descriptor → VID/PID/类/配置数/`bMaxPacketSize0`
2. 每个 Configuration 的全集 → 接口与端点
3. `bInterfaceClass/SubClass/Protocol` → 加载类驱动
4. Endpoint 字段 → 创建 Pipe、限制 MaxPacket、调度周期

---

## 4.2 Device Descriptor（18 字节）

### 字段表

| 偏移 | 域 | 大小 | 说明 |
|------|----|------|------|
| 0 | bLength | 1 | 恒为 `0x12` |
| 1 | bDescriptorType | 1 | `0x01` DEVICE |
| 2 | bcdUSB | 2 | USB 版本 BCD，如 `0x0200` |
| 4 | bDeviceClass | 1 | 设备类；`0x00` 表示类在接口中定义 |
| 5 | bDeviceSubClass | 1 | |
| 6 | bDeviceProtocol | 1 | |
| 7 | bMaxPacketSize0 | 1 | EP0 最大包长 |
| 8 | idVendor | 2 | VID |
| 10 | idProduct | 2 | PID |
| 12 | bcdDevice | 2 | 设备发行版本 BCD |
| 14 | iManufacturer | 1 | 厂商字符串索引，0=无 |
| 15 | iProduct | 1 | 产品字符串索引 |
| 16 | iSerialNumber | 1 | 序列号字符串索引 |
| 17 | bNumConfigurations | 1 | 配置个数 |

### 真实十六进制示例

```text
12 01 00 02 00 00 00 40 5E 04 01 C0 00 01 01 02 03 01
```

### 逐字节拆解

| 字节 | 值 | 含义 |
|------|-----|------|
| 0 | `12` | 长度 18 |
| 1 | `01` | Device Descriptor |
| 2-3 | `00 02` | bcdUSB = 0x0200（USB 2.0） |
| 4 | `00` | 类在 Interface 定义 |
| 5 | `00` | |
| 6 | `00` | |
| 7 | `40` | EP0 MPS = 64 |
| 8-9 | `5E 04` | idVendor = 0x045E |
| 10-11 | `01 C0` | idProduct = 0xC001 |
| 12-13 | `00 01` | bcdDevice = 1.00 |
| 14 | `01` | iManufacturer = 1 |
| 15 | `02` | iProduct = 2 |
| 16 | `03` | iSerialNumber = 3 |
| 17 | `01` | 1 个配置 |

### 速度与 `bMaxPacketSize0` 约束（再强调）

- LS：只能 8  
- FS：8/16/32/64  
- HS：必须 64  

---

## 4.3 Configuration Descriptor（9 字节头 + 后续集合）

### 头字段

| 偏移 | 域 | 大小 | 说明 |
|------|----|------|------|
| 0 | bLength | 1 | `0x09` |
| 1 | bDescriptorType | 1 | `0x02` |
| 2 | wTotalLength | 2 | **本配置全部描述符总字节数** |
| 4 | bNumInterfaces | 1 | 接口数 |
| 5 | bConfigurationValue | 1 | 供 SET_CONFIGURATION 使用的值 |
| 6 | iConfiguration | 1 | 字符串索引 |
| 7 | bmAttributes | 1 | bit7 恒 1；bit6 Self-powered；bit5 Remote Wakeup |
| 8 | bMaxPower | 1 | 单位 **2 mA**（USB 2.0）；如 `0x32`=100 mA |

### 示例头

```text
09 02 29 00 01 01 00 80 32
```

| 字段 | 值 | 含义 |
|------|-----|------|
| wTotalLength | `0x0029` | 其后总长 41 字节 |
| bNumInterfaces | 1 | |
| bConfigurationValue | 1 | SET_CONFIGURATION(1) |
| bmAttributes | `0x80` | 总线供电，无远程唤醒 |
| bMaxPower | `0x32` | 100 mA |

> **标准注意**：USB 2.0 的 `bMaxPower` 单位是 2 mA。USB 3.x 改为 8 mA——本文只按 2.0。

---

## 4.4 Interface Descriptor（9 字节）

| 偏移 | 域 | 说明 |
|------|----|------|
| 0 | bLength | `0x09` |
| 1 | bDescriptorType | `0x04` |
| 2 | bInterfaceNumber | 接口号，从 0 起 |
| 3 | bAlternateSetting | 备用设置号，默认 0 |
| 4 | bNumEndpoints | **不含** EP0 的端点数 |
| 5 | bInterfaceClass | 类代码，如 `0x03`=HID，`0x08`=Mass Storage |
| 6 | bInterfaceSubClass | |
| 7 | bInterfaceProtocol | |
| 8 | iInterface | 字符串索引 |

### 示例（HID 鼠标类接口）

```text
09 04 00 00 01 03 01 02 00
```

| 字段 | 值 | 含义 |
|------|-----|------|
| bInterfaceNumber | 0 | |
| bAlternateSetting | 0 | |
| bNumEndpoints | 1 | 一个中断 IN |
| Class/Sub/Prot | 03/01/02 | HID / Boot / Mouse |

Host 据此加载 HID 驱动，而不是 MSC 驱动。

---

## 4.5 Endpoint Descriptor（7 字节）

| 偏移 | 域 | 说明 |
|------|----|------|
| 0 | bLength | `0x07` |
| 1 | bDescriptorType | `0x05` |
| 2 | bEndpointAddress | bit7 方向，bit3..0 端点号 |
| 3 | bmAttributes | bit1..0：0=Control，1=Iso，2=Bulk，3=Interrupt |
| 4 | wMaxPacketSize | 最大包长（HS 还有额外位域用于高带宽，FS/LS 主要看低 11 位） |
| 6 | bInterval | 轮询间隔；含义随速度与传输类型而变 |

### 示例（FS Interrupt IN，EP1，MPS=8，间隔 10 ms）

```text
07 05 81 03 08 00 0A
```

| 字节 | 含义 |
|------|------|
| `81` | EP1 IN |
| `03` | Interrupt |
| `08 00` | wMaxPacketSize = 8 |
| `0A` | bInterval = 10（FS 中断：单位 ms） |

---

## 4.6 完整配置集合示例（把 4.3~4.5 串起来）

假设 HID 设备：Configuration + Interface + HID Descriptor + Endpoint，`wTotalLength = 0x0022`（34 字节）时的一种典型布局：

```text
09 02 22 00 01 01 00 A0 32   ; Configuration (9)
09 04 00 00 01 03 01 02 00   ; Interface   (9)
09 21 11 01 00 01 22 2E 00   ; HID         (9)  ← 类描述符
07 05 81 03 08 00 0A         ; Endpoint    (7)
```

Host 解析算法（简化）：

```c
u8 *p = buf;
u8 *end = buf + wTotalLength;
while (p + 1 < end) {
    u8 len  = p[0];
    u8 type = p[1];
    if (len < 2 || p + len > end) break; /* 异常 */
    switch (type) {
    case 0x02: /* already know config */ break;
    case 0x04: /* parse interface */ break;
    case 0x05: /* parse endpoint  */ break;
    case 0x21: /* HID desc        */ break;
    default:   /* skip unknown    */ break;
    }
    p += len;
}
```

---

## 4.7 String Descriptor

### 格式

| 偏移 | 域 | 说明 |
|------|----|------|
| 0 | bLength | 全长（含自身 2 字节） |
| 1 | bDescriptorType | `0x03` |
| 2.. | bString | UTF-16LE 字符 |

### Index 0：语言 ID 列表

```text
04 03 09 04
```

- 长度 4  
- 语言 `0x0409`（English US）

### Index N：具体字符串（如 “Demo”）

```text
0A 03 44 00 65 00 6D 00 6F 00
```

| 内容 | 含义 |
|------|------|
| `0A` | 10 字节 |
| `03` | STRING |
| `44 00` | 'D' |
| `65 00` | 'e' |
| `6D 00` | 'm' |
| `6F 00` | 'o' |

请求时 `wIndex` 必须为语言 ID（对 index≠0）。

---

## 4.8 HID Descriptor（类描述符示例）

当 Interface Class = 0x03 时，配置集合中常紧跟 **HID Descriptor**（类型 `0x21`）：

| 偏移 | 域 | 说明 |
|------|----|------|
| 0 | bLength | 常 `0x09` |
| 1 | bDescriptorType | `0x21` |
| 2 | bcdHID | HID 版本，如 `0x0111` |
| 4 | bCountryCode | 0=不限定 |
| 5 | bNumDescriptors | 下级描述符个数 |
| 6 | bDescriptorType | 常 `0x22` = Report |
| 7 | wDescriptorLength | Report Descriptor 长度 |

### 示例

```text
09 21 11 01 00 01 22 2E 00
```

含义：HID 1.11，1 个下级描述符，Report 类型，长度 `0x002E`（46 字节）。

**注意**：Report Descriptor **不在**配置集合里一并返回，而需 Host 再发：

- `bmRequestType = 0x81`（IN, Standard, **Interface**）或某些栈用 Class 请求  
- 实际常见：`GET_DESCRIPTOR`，Recipient=Interface，`wValue=0x2200`，`wIndex=接口号`

（精确 bmRequestType 以 HID 规范与主机栈为准；设备固件需按自己声明的方式响应。）

---

## 4.9 Host 如何从描述符识别设备能力

| 问题 | 看哪里 |
|------|--------|
| 是什么设备（厂商/产品）？ | `idVendor` / `idProduct` |
| 按类还是按 VID/PID 匹配驱动？ | `bDeviceClass` 或接口 Class |
| 有几个可切换模式？ | `bNumConfigurations` |
| 耗电多少？ | `bmAttributes` + `bMaxPower` |
| 几个功能？ | 各 `Interface` |
| 能不能当键盘/鼠标？ | HID Class + Protocol |
| 有哪些通信管道？ | 各 `Endpoint` 的地址/类型/MPS/间隔 |
| EP0 怎么分包？ | `bMaxPacketSize0` |
| 人可读名称？ | String 索引 |

---

## 4.10 描述符设计与固件常见错误

1. **`wTotalLength` 与真实返回长度不一致** → Host 解析越界或截断，枚举失败  
2. **`bNumEndpoints` 与后续 Endpoint 描述符个数不符**  
3. **FS 设备却声明 HS 专用组合** / MPS 非法  
4. **配置集合中遗漏类描述符**（HID 缺 0x21）  
5. **字符串 UTF-16LE 编码错误或 bLength 算错**  
6. **`bConfigurationValue` 为 0**（合法但易混淆；SET_CONFIGURATION(0) 是取消配置）  

下一章从 Host/Device 软件与控制器分层看实现：[第5章 软件驱动角度](./chapter5.md)
