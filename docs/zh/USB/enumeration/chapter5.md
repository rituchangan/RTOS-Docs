# 第5章 从软件驱动角度分析枚举

本章区分 **USB 标准规定的行为** 与 **具体芯片/协议栈的实现选择**，给出 Host 与 Device 两端可落地的状态机伪代码。

---

## 5.1 分层架构：PHY / Controller / Driver / Stack

```
┌─────────────────────────────────────────────┐
│ 应用 / 类驱动（HID、MSC、CDC…）              │  ← 类协议
├─────────────────────────────────────────────┤
│ USB 协议栈（枚举状态机、URB、描述符解析）     │  ← 标准请求语义
├─────────────────────────────────────────────┤
│ HCD 或 UDC Driver（端点、DMA、中断）         │  ← 芯片相关
├─────────────────────────────────────────────┤
│ Host Controller / Device Controller 硬件     │  ← 事务、PID、FIFO
├─────────────────────────────────────────────┤
│ USB PHY                                      │  ← 电气、串行化
└─────────────────────────────────────────────┘
```

| 层级 | Host 侧名称 | Device 侧名称 | 主要职责 |
|------|-------------|---------------|----------|
| PHY | USB PHY | USB PHY | 差分收发、时钟恢复、HS Chirp 模拟部分等 |
| Controller | EHCI/OHCI/UHCI/DWCHCI… | UDC / Device Controller | 执行/响应事务，管理端点硬件 |
| Controller Driver | HCD | UDC Driver | 初始化控制器、提交传输、处理中断 |
| Protocol Stack | USB Core + Hub + 枚举 | Device/Gadget Stack | 标准请求、地址/配置状态、描述符 |
| Class | 主机类驱动 | 设备类实现 | 功能协议 |

**标准规定**：请求格式、设备状态、描述符布局、传输语义。  
**实现决定**：寄存器怎么写、是否用 DMA、中断还是轮询、Host 是否二次 Reset、描述符存在 Flash 还是 RAM。

---

## 5.2 USB Host 如何实现枚举

### 5.2.1 检测与复位（多在 Hub 驱动）

1. HCD 报告端口 Change（Connect）
2. Hub 驱动去抖、上电（若有）
3. 对端口发 **Reset**，等待完成
4. 读取端口速度（LS/FS/HS）
5. 为设备分配临时结构，地址仍为 0，创建默认控制管道（MPS 先按保守值，如 8 或 64）

### 5.2.2 发送标准请求

Host 栈构造 Setup 包，封装为控制 URB/传输，交给 HCD：

- HCD 把 SETUP/DATA/STATUS 阶段编成控制器可执行的描述符/队列
- 中断或完成回调返回实际长度与状态（ACK 完成 / STALL / 超时）

### 5.2.3 解析描述符

- 校验 `bLength`、`bDescriptorType`
- 切片配置集合
- 按 VID/PID 或 Class 匹配驱动，调用 `probe`

---

## 5.3 USB Device 固件如何响应枚举

典型中断驱动模型：

1. UDC 收到 SETUP → 中断，软件读 8 字节 Setup FIFO  
2. 解析 `bRequest` / `wValue`…  
3. 若需 IN 数据：准备缓冲，写 EP0 TX FIFO，使能发送  
4. 若需 OUT 数据：在 OUT 中断中收数据  
5. Status 阶段常由硬件协助完成；`SET_ADDRESS` 在 Status 完成后再写地址寄存器  
6. `SET_CONFIGURATION` 时配置非零端点寄存器（类型、MPS、使能）

---

## 5.4 Host 端枚举状态机（C 伪代码）

```c
typedef enum {
    HE_WAIT_ATTACH,
    HE_RESET,
    HE_GET_DEV_DESC_8,
    HE_RESET_AGAIN,      /* 可选，许多 Host 会做 */
    HE_SET_ADDRESS,
    HE_GET_DEV_DESC_FULL,
    HE_GET_CFG_DESC_HDR,
    HE_GET_CFG_DESC_FULL,
    HE_GET_STRINGS,      /* 可选 */
    HE_SET_CONFIGURATION,
    HE_BIND_CLASS,
    HE_FAILED
} host_enum_state_t;

int host_enumerate(port_t *port)
{
    u8  setup[8];
    u8  buf[256];
    u8  addr;
    u16 total;
    int rc;

    /* 1. 等待连接并去抖（去抖时间为实现策略） */
    wait_port_connect(port);
    delay_ms(100);

    /* 2. Bus Reset */
    port_reset(port, /*se0_ms*/10);
    port->dev_addr = 0;
    port->ep0_mps  = 8;   /* 先保守；读到描述符后再更新 */

    /* 3. GET_DESCRIPTOR(Device) 前 8 字节 */
    fill_setup(setup, 0x80, 0x06, 0x0100, 0x0000, 8);
    rc = ctrl_transfer(port, setup, buf, 8);
    if (rc < 8) return -1;
    port->ep0_mps = buf[7];
    if (!valid_ep0_mps(port->speed, port->ep0_mps)) return -1;

    /* 4. 可选二次 Reset */
    port_reset(port, 10);
    port->dev_addr = 0;

    /* 5. SET_ADDRESS */
    addr = alloc_address();
    fill_setup(setup, 0x00, 0x05, addr, 0x0000, 0);
    rc = ctrl_transfer(port, setup, NULL, 0); /* Status 后设备改地址 */
    if (rc < 0) return -1;
    port->dev_addr = addr;

    /* 6. 完整 Device Descriptor */
    fill_setup(setup, 0x80, 0x06, 0x0100, 0x0000, 18);
    rc = ctrl_transfer(port, setup, buf, 18);
    if (rc < 18) return -1;
    parse_device_descriptor(&port->dd, buf);

    /* 7. Configuration：先 9 再 wTotalLength */
    fill_setup(setup, 0x80, 0x06, 0x0200, 0x0000, 9);
    rc = ctrl_transfer(port, setup, buf, 9);
    if (rc < 9) return -1;
    total = buf[2] | (buf[3] << 8);
    if (total > sizeof(buf)) return -1;

    fill_setup(setup, 0x80, 0x06, 0x0200, 0x0000, total);
    rc = ctrl_transfer(port, setup, buf, total);
    if (rc < total) return -1;
    parse_config_set(&port->cfg, buf, total);

    /* 8. SET_CONFIGURATION */
    fill_setup(setup, 0x00, 0x09, port->cfg.value, 0x0000, 0);
    rc = ctrl_transfer(port, setup, NULL, 0);
    if (rc < 0) return -1;

    /* 9. 绑定类驱动 */
    return bind_class_drivers(port);
}
```

说明：

- `ctrl_transfer()` 内部完成 Setup/Data/Status 三阶段，并处理 NAK 重试与 STALL 错误  
- `alloc_address()`、是否 `HE_RESET_AGAIN`、字符串是否读取，均属 **Host 策略**

---

## 5.5 Device 端响应伪代码

```c
static u8  pending_addr;
static u8  current_cfg;
static enum { ST_DEFAULT, ST_ADDRESS, ST_CONFIGURED } dev_state;

void udc_reset_isr(void)
{
    pending_addr = 0;
    current_cfg  = 0;
    dev_state    = ST_DEFAULT;
    udc_set_address(0);
    udc_ep0_enable(desc_device.bMaxPacketSize0);
    /* 禁用非零端点 */
}

void udc_setup_isr(void)
{
    u8 s[8];
    udc_read_setup(s);

    u8  bm  = s[0], req = s[1];
    u16 wValue = s[2] | (s[3] << 8);
    u16 wIndex = s[4] | (s[5] << 8);
    u16 wLen   = s[6] | (s[7] << 8);

    if ((bm & 0x60) != 0x00) { /* 非 Standard：交给类/厂商 */
        if (class_setup_handler(s) != 0)
            udc_ep0_stall();
        return;
    }

    switch (req) {
    case 0x06: /* GET_DESCRIPTOR */
        handle_get_descriptor(wValue, wIndex, wLen);
        break;

    case 0x05: /* SET_ADDRESS */
        pending_addr = (u8)(wValue & 0x7F);
        udc_ep0_status_in();          /* 先完成 Status */
        /* 真正改地址放到 Status 完成回调 */
        break;

    case 0x09: /* SET_CONFIGURATION */
        if (wValue == 0) {
            current_cfg = 0;
            disable_non_zero_eps();
            dev_state = (udc_get_address() == 0) ? ST_DEFAULT : ST_ADDRESS;
            udc_ep0_status_in();
        } else if (wValue == CONFIG_VALUE) {
            current_cfg = (u8)wValue;
            enable_config_endpoints();
            dev_state = ST_CONFIGURED;
            udc_ep0_status_in();
        } else {
            udc_ep0_stall();
        }
        break;

    case 0x00: /* GET_STATUS */
        handle_get_status(bm, wIndex, wLen);
        break;

    case 0x01: /* CLEAR_FEATURE */
        handle_clear_feature(bm, wValue, wIndex);
        break;

    default:
        udc_ep0_stall();
        break;
    }
}

void udc_ep0_status_complete_isr(void)
{
    if (pending_addr) {
        udc_set_address(pending_addr);
        dev_state = ST_ADDRESS;
        pending_addr = 0;
    }
}

void handle_get_descriptor(u16 wValue, u16 wIndex, u16 wLen)
{
    u8  type  = wValue >> 8;
    u8  index = wValue & 0xFF;
    const u8 *data = NULL;
    u16 avail = 0;

    switch (type) {
    case 0x01: /* DEVICE */
        data = (const u8 *)&desc_device;
        avail = 18;
        break;
    case 0x02: /* CONFIGURATION */
        data = desc_config_set;
        avail = CONFIG_TOTAL_LEN; /* 等于 wTotalLength */
        break;
    case 0x03: /* STRING */
        data = find_string(index, wIndex, &avail);
        break;
    default:
        udc_ep0_stall();
        return;
    }

    if (!data) { udc_ep0_stall(); return; }
    if (avail > wLen) avail = wLen;
    udc_ep0_write_in(data, avail); /* 底层按 MPS 拆包，短包结束 */
}
```

### 与规范对齐的检查清单

- [ ] Reset 后地址为 0、状态 Default  
- [ ] `SET_ADDRESS` 在 Status **之后**生效  
- [ ] `GET_DESCRIPTOR` 返回长度 = `min(可用, wLength)`，并用短包结束  
- [ ] 未知请求 STALL  
- [ ] `SET_CONFIGURATION` 后才使能非零 EP  

---

## 5.6 中断、DMA、Endpoint FIFO 注意点

### 中断

- **Reset / Suspend / Resume / Setup / EP 完成** 是常见中断源  
- Setup 中断里避免长时间阻塞；准备数据可放到主循环/任务，但对枚举请求应尽快响应，减少 Host 超时  
- 注意中断优先级：USB 中断过低可能导致频繁 NAK，枚举变慢甚至超时失败  

### Endpoint FIFO

- EP0 常有独立小 FIFO；注意 **写超 MPS**、**未读空又写入**  
- 控制写（OUT）要在 Status 前收齐 Host 数据  
- 某些 UDC 在 Setup 到达时自动清空 EP0 缓冲——属硬件行为，读手册  

### DMA

- 枚举阶段数据短，很多 MCU 用 PIO 填 EP0 即可  
- 若 EP0 也走 DMA：注意对齐、缓存一致性（有 D-Cache 的 SoC 必须 clean/invalidate）  
- 非零端点 Bulk 用 DMA 时，不要在枚举未完成时误使能  

### 时钟与 PHY

- Device 必须在 Reset 结束前稳定工作时钟；HS 还需 ULPI/UTMI 时钟正确  
- 上拉使能时机：过早 → Host 检测到但设备未就绪；过晚 → 检测超时  

### 标准 vs 实现再强调

| 行为 | 归属 |
|------|------|
| Setup 8 字节布局、状态机 | **标准** |
| 二次 Reset、先读 8 字节 | **Host 实现策略** |
| Setup FIFO 深度、是否自动 ACK | **芯片 UDC** |
| 描述符存放位置、RTOS 任务划分 | **固件架构** |

下一章用完整 FS 抓包把流程钉在十六进制上：[第6章 抓包实战](./chapter6.md)
