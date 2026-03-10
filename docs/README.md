## MODBUS-LIB 使用指南（STM32 FreeRTOS）

本库为 STM32 平台上的 Modbus 主/从一体化实现，支持 Modbus RTU（USART 中断/可选 DMA）、USB CDC（可选）、Modbus TCP（可选），基于 FreeRTOS 任务、队列、通知与定时器，方便在多任务环境中稳定运行。

### 目录结构
- `Modbus.h/.c`: 核心实现与对外 API
- `ModbusConfig.h`: 按工程定制的配置文件（本项目已提供）
- `UARTCallback.c`: HAL UART 中断回调的统一接入（务必编入工程）

### 主要特性
- **协议支持**: FC1/2/3/4/5/6/15/16
- **角色**: 同时支持 `MB_MASTER` 与 `MB_SLAVE`
- **多硬件后端**: `USART_HW`、`USART_HW_DMA`、`USB_CDC_HW`、`TCP_HW`
- **FreeRTOS 友好**: 使用任务/队列/通知/定时器；提供同步式查询 `ModbusQueryV2`
- **多实例**: 通过 `modbusHandler_t` 支持并发实例（上限由 `MAX_M_HANDLERS` 配置）

---

### 快速开始

1) 配置 `ModbusConfig.h`
- 根据需要取消注释：
  - `// #define ENABLE_USB_CDC 1`
  - `// #define ENABLE_TCP 1`
  - `// #define ENABLE_MODBUS_USART_DMA 1`
- 调整参数：
  - `T35`：RTU 帧间隔检测定时（tick）
  - `MAX_BUFFER`：通信缓冲区字节数
  - `TIMEOUT_MODBUS`：主站查询超时（tick）
  - `MAX_M_HANDLERS`、`MAX_TELEGRAMS`：并发实例与主站队列长度
  - TCP 相关：`NUMBERTCPCONN`、`TCPAGINGCYCLES`

2) CubeMX/HAL 外设与工程准备
- 使能目标 UART（RTU），如使用 RS485：
  - 建议使用硬件半双工模式（HAL 的 HalfDuplex API）
  - 配置 DE/RE 方向控制引脚，接入到 `modbusHandler_t.EN_Port/EN_Pin`
- 将 `UARTCallback.c` 加入编译；如工程已有自定义 UART 回调，请在回调中调用库内逻辑（或合并逻辑），避免覆盖丢失。
- 若启用 DMA 模式，需要 HAL 支持 `HAL_UARTEx_ReceiveToIdle_DMA` 且实现 `HAL_UARTEx_RxEventCallback` 与 `HAL_UART_ErrorCallback`（本库已提供）。
- 如启用 TCP，需要先初始化 LwIP，并保证在调度器启动后进行 TCP 监听/连接（详见下文）。

3) 典型初始化流程
- 填充 `modbusHandler_t` 关键字段：
  - `uModbusType`: `MB_MASTER` 或 `MB_SLAVE`
  - `port`: 指向 `UART_HandleTypeDef`（如 `&huart1`）
  - `u8id`: 主站必须为 0；从站为 1..247
  - `EN_Port/EN_Pin`: RS485 方向控制；非 RS485 可设为 `NULL/0`
  - `u16regs/u16regsize`: 从站的数据寄存器映射与长度（主站在查询时传入缓冲指针）
  - `u16timeOut`: 一般设为 `TIMEOUT_MODBUS`
  - `xTypeHW`: `USART_HW` / `USART_HW_DMA` / `USB_CDC_HW` / `TCP_HW`
  - 仅 TCP 额外：从站可设 `uTcpPort`（缺省 502）

---

### 从站最小示例（USART RTU）

```c
#include "Modbus.h"

// 预留从站保持寄存器区（示例：128 个 16 位寄存器）
static uint16_t g_slaveHoldingRegs[128];

static modbusHandler_t g_modbusSlave = {
    .uModbusType = MB_SLAVE,
    .port = &huart1,
    .u8id = 1,                 // 从站 ID ∈ [1..247]
    .EN_Port = RS485_DE_GPIO_Port,
    .EN_Pin = RS485_DE_Pin,
    .u16regs = g_slaveHoldingRegs,
    .u16regsize = 128,
    .u16timeOut = TIMEOUT_MODBUS,
    .xTypeHW = USART_HW
};

void Modbus_Slave_Init(void)
{
    // 确保 UART 已初始化，且处于就绪态
    ModbusInit(&g_modbusSlave);   // 创建任务、定时器、信号量
    ModbusStart(&g_modbusSlave);  // 启动串口接收与内部状态
}
```

说明：
- 库内部在收到完整帧后，会在从站任务中解析并读写 `u16regs`；
- 你可以在其他任务里更新 `g_slaveHoldingRegs` 对应的数据源（尽量避免与库同时读写同一元素的竞争，见最佳实践）。

---

### 主站最小示例（同步查询，USART RTU）

```c
#include "Modbus.h"

static uint16_t g_masterBuf[16];

static modbusHandler_t g_modbusMaster = {
    .uModbusType = MB_MASTER,
    .port = &huart1,
    .u8id = 0,                 // 主站必须为 0
    .EN_Port = RS485_DE_GPIO_Port,
    .EN_Pin = RS485_DE_Pin,
    .u16timeOut = TIMEOUT_MODBUS,
    .xTypeHW = USART_HW
};

void Modbus_Master_Init(void)
{
    ModbusInit(&g_modbusMaster);
    ModbusStart(&g_modbusMaster);
}

// 在某个任务中发起一次读取保持寄存器（FC=3）的同步查询
void Modbus_Master_ReadOnce(void)
{
    modbus_t telegram = {
        .u8id = 1,                    // 目标从站 ID
        .u8fct = MB_FC_READ_REGISTERS,
        .u16RegAdd = 0x0000,          // 起始地址
        .u16CoilsNo = 8,              // 寄存器数量
        .u16reg = g_masterBuf         // 主站结果缓冲区
    };

    uint32_t ret = ModbusQueryV2(&g_modbusMaster, telegram);
    if (ret == OP_OK_QUERY) {
        // g_masterBuf 已填充返回数据
    } else {
        // ret 为错误码：如 ERR_TIME_OUT/ERR_BAD_CRC/ERR_EXCEPTION 等
    }
}
```

说明：
- `ModbusQueryV2` 会等待响应或超时，并返回结果；
- 若使用 `ModbusQuery`，需要在调用任务侧通过 `ulTaskNotifyTake` 等自行等待通知。

---

### 主站异步查询示例（非阻塞）

场景：主站在任务中发起查询后不阻塞，继续执行其他工作；稍后用任务通知读取结果码与数据。

关键点：
- 使用 `ModbusQuery(&handler, telegram)` 入队，不阻塞；
- 当前任务稍后通过 `xTaskNotifyWait()` 读取由库填充的通知值：
  - `OP_OK_QUERY` 表示成功，主站缓冲 `telegram.u16reg` 已被填充；
  - 否则为错误码（如 `ERR_TIME_OUT/ERR_BAD_CRC/ERR_EXCEPTION`）；
- 单个任务同一时间仅维护一个“在途查询”，使用 `pending` 标志避免重复入队。

```c
#include "Modbus.h"

static uint16_t g_masterBuf[16];
static modbusHandler_t g_modbusMaster = {
    .uModbusType = MB_MASTER,
    .port = &huart1,
    .u8id = 0,
    .EN_Port = RS485_DE_GPIO_Port,
    .EN_Pin = RS485_DE_Pin,
    .u16timeOut = TIMEOUT_MODBUS,
    .xTypeHW = USART_HW
};

void Task_ModbusMaster_Async(void *argument)
{
    ModbusInit(&g_modbusMaster);
    ModbusStart(&g_modbusMaster);

    // 预构建一个读寄存器查询模板（可根据需要修改/轮询不同从站与地址）
    modbus_t telegram = {
        .u8id = 1,
        .u8fct = MB_FC_READ_REGISTERS,
        .u16RegAdd = 0x0000,
        .u16CoilsNo = 8,
        .u16reg = g_masterBuf
    };

    bool pending = false;           // 标记是否已有在途查询

    for(;;)
    {
        // 1) 若无在途查询，则发起一次查询（不阻塞）
        if (!pending)
        {
            ModbusQuery(&g_modbusMaster, telegram);
            pending = true;
        }

        // 2) 做其他工作（计算/IO/调度等）
        // ...

        // 3) 非阻塞检查是否有结果返回（0 超时表示仅检查一次）
        uint32_t notifyValue = 0;
        if (xTaskNotifyWait(0, 0xFFFFFFFF, &notifyValue, 0) == pdTRUE)
        {
            pending = false;    // 当前在途查询完成（无论成功还是失败）

            if (notifyValue == OP_OK_QUERY)
            {
                // 成功：g_masterBuf 已被库填充，可直接使用
                // TODO: 处理数据
            }
            else
            {
                // 失败：notifyValue 为错误码（如 ERR_TIME_OUT / ERR_BAD_CRC / ERR_EXCEPTION）
                // TODO: 根据需要重试/告警/退避
            }
        }

        // 4) 合理让出 CPU，避免忙轮询（按系统实时性调节周期）
        vTaskDelay(pdMS_TO_TICKS(10));
    }
}
```

变体：若希望“等待但设定上限时间”，将 `xTaskNotifyWait` 的最后一个参数设为超时（如 `pdMS_TO_TICKS(200)`），并在返回 `pdFALSE` 时保留 `pending=true`，下轮继续等待或决定取消/重发。

注意：
- 不要在同一任务中并行发起多个“在途查询”，否则后续通知会相互覆盖（库使用 `eSetValueWithOverwrite` 设置通知值）。若需要多路并发，可：
  - 为不同请求分配不同的任务；
  - 或在单任务内做状态机，严格串行化请求；
  - 或提高 `MAX_TELEGRAMS` 仅用于排队，但仍需按通知完成一个再发下一个。

---

### USB CDC 与 Modbus TCP

- USB CDC：
  - 取消 `ModbusConfig.h` 中 `ENABLE_USB_CDC` 注释；
  - 初始化后调用 `ModbusStartCDC(&handler)`；
  - 仍需提供 `u16regs/u16regsize`（从站）。

- Modbus TCP：
  - 取消 `ENABLE_TCP` 注释，确保 LwIP 初始化完成；
  - 从站（服务器）通过 `StartTaskModbusSlave` 内部 `TCPinitserver` 在调度器运行后进行 `netconn_listen`；
  - 主站（客户端）查询时根据 `modbus_t` 的 `xIpAddress/u16Port/u8clientID` 连接目标；
  - 注意：`TCPinitserver` 的注释提示需在调度器启动后调用相关网络 API，建议在任务上下文中初始化。

---

### API 速查
- 初始化与启动：`ModbusInit`, `ModbusStart`, `ModbusStartCDC`
- 主站查询：`ModbusQuery`, `ModbusQueryV2`, `ModbusQueryInject`
- 工具：`calcCRC`, `setTimeOut/getTimeOut/getTimeOutState`
- 回调（库内已实现）：`HAL_UART_TxCpltCallback`, `HAL_UART_RxCpltCallback`

---

### 最佳实践（Production Tips）
- **ID 约束**：主站 `u8id` 必须为 0；从站 `u8id` ∈ [1..247]。
- **RS485 方向控制**：
  - 提供 `EN_Port/EN_Pin`，库会在发送前使能 TX，发送完成后切回 RX，并调用 `HAL_HalfDuplex_EnableTransmitter/Receiver`，避免回波干扰。
  - 硬件层推荐使用半双工模式，减少引脚占用与冲突。
- **T3.5 帧间隔**：
  - `T35` 直接影响 RTU 帧结束判断；低波特率可适当增大，避免包截断；高波特率可保持较小以提升吞吐。
- **超时管理**：
  - 主站超时通过 `xTimerTimeout` 控制，合理设置 `TIMEOUT_MODBUS`，并对 `ERR_TIME_OUT` 做降级处理（重试或标记失效）。
- **缓冲区大小**：
  - `MAX_BUFFER` 默认 128，若一次写多寄存器/线圈或 TCP 负载更大，可适当增大；同时评估堆栈/堆内存。
- **DMA 模式（高吞吐）**：
  - 高波特率或密集轮询建议启用 `ENABLE_MODBUS_USART_DMA`；确保 HAL 版本支持 `ReceiveToIdle`，并启用对应回调（本库已实现）。
- **任务优先级**：
  - Modbus 任务默认 `osPriorityNormal`；如系统较繁忙，适当提高优先级，避免接收丢帧。
- **并发与线程安全**：
  - 库内部通过信号量保护 Modbus 缓冲与状态；
  - 应用侧若频繁更新从站寄存器映射，建议在更新关键区域关闭中断或使用轻量临界区，避免与库同时访问同一元素。
- **回调整合**：
  - 若工程已有 UART 回调，请将本库回调逻辑并入，确保 RX/TX 事件能通知到 Modbus 任务。
- **TCP 连接管理**：
  - 服务器端有连接老化与清理机制（`TCPAGINGCYCLES`）；客户端按 `u8clientID` 复用连接槽，注意上限 `NUMBERTCPCONN`。

---

### 常见问题排查（FAQ）
- **无响应/收不到数据**：
  - 检查 `u8id` 设置、线缆与波特率；
  - RS485 DE/RE 引脚是否正确接入并配置为推挽输出；
  - `UARTCallback.c` 是否已编译，且回调未被覆盖；
  - 从站是否设置了有效的 `u16regs/u16regsize`。
- **CRC 错误**：
  - 确保发送与接收的大小端顺序正确；
  - 排查 RS485 回波或线缆干扰；
  - 降低波特率或提升 `T35` 观察稳定性。
- **超时 ERR_TIME_OUT**：
  - 检查对端是否响应、线路质量、接收中断是否触发；
  - 适当增大 `TIMEOUT_MODBUS`；
  - TCP 模式确认对端 IP/端口、LwIP 状态正确。
- **缓冲溢出 ERR_BUFF_OVERFLOW**：
  - 增大 `MAX_BUFFER` 或降低单次负载；
  - 使用 DMA 模式以减少中断抢占与拷贝开销。

---

### 本库优势
- **轻量且清晰**：单一 `modbusHandler_t` 管理状态，发送/接收路径直观，易调试。
- **多后端统一抽象**：串口中断、串口 DMA、USB CDC、TCP 统一在同一套 API 下切换。
- **FreeRTOS 深度融合**：基于队列与任务通知的查询模型，既支持异步也提供同步 `ModbusQueryV2`。
- **稳健的错误处理**：统一错误/操作码（如 `ERR_BAD_CRC/ERR_TIME_OUT/OP_OK_QUERY`），便于上层逻辑分支。
- **可移植性**：HAL 回调集中在 `UARTCallback.c`，迁移到新 MCU 只需最小改动（USART TC 寄存器位已做系列区分）。
- **扩展友好**：`ModbusConfig.h` 暴露关键参数，易于在不同项目中裁剪与调优。

---

### 进阶建议
- 结合硬件看门狗，在主站长时间超时/无响应时复位或降级；
- 将从站寄存器映射与真实业务数据解耦，使用缓冲快照周期性同步，降低竞争；
- 在主站侧实现查询节流与失败重试策略，避免总线拥塞。

---

如需进一步集成示例（含 CubeMX 配置、RS485 原理图建议、TCP 与 LWIP 参数），可在 `Docs/` 目录补充工程说明或联系维护者扩展示例。


