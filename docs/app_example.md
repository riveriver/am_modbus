# Example
```
#include "read_encoder_task.h"

#define RET_D(...) printf(__VA_ARGS__)
#define RET_I(...) printf(__VA_ARGS__)
#define RET_E(...) printf(__VA_ARGS__)

extern UART_HandleTypeDef huart7; 

static modbusHandler_t encoder_hub_client;
osThreadId_t encoder_hub_task_handle;
const osThreadAttr_t encoder_hub_task_attributes = {
  .name = "encoder_hub_task",
  .stack_size = 256 * 4,
  .priority = (osPriority_t) osPriorityNormal,
};

#define RESP_DATA_BUFF_SIZE 8
static uint16_t resp_data_buff[RESP_DATA_BUFF_SIZE] = {0};
#define REQUEST_SLAVE_NUM 3
static uint32_t encoder_data[9] = {0};

void init_encoder_hub(void)
{
    memset(resp_data_buff, 0, sizeof(resp_data_buff));
    memset(encoder_data, 0, sizeof(encoder_data));
    
    encoder_hub_client.uModbusType = MB_MASTER;
    encoder_hub_client.u8id = 0;  // Master ID = 0
    encoder_hub_client.port = &huart7;
    encoder_hub_client.EN_Port = NULL; 
    encoder_hub_client.EN_Pin = 0;
    encoder_hub_client.u16regs = resp_data_buff;
    encoder_hub_client.u16regsize = RESP_DATA_BUFF_SIZE;
    encoder_hub_client.u16timeOut = 500;
    encoder_hub_client.xTypeHW = USART_HW;
    
    ModbusInit(&encoder_hub_client);
    ModbusStart(&encoder_hub_client);

    RET_I("init encoder hub master\r\n");
}

static bool read_encoder_non_block(uint8_t slave_addr)
{
    modbus_t telegram;
    
    telegram.u8id = slave_addr;
    telegram.u8fct = MB_FC_READ_REGISTERS; 
    telegram.u16RegAdd = 0x0000;
    telegram.u16CoilsNo = TOTAL_HOLDING_REGS; 
    telegram.u16reg = resp_data_buff;
    // 发送异步查询
    ModbusQuery(&encoder_hub_client, telegram);
    
    return true;
}

// 基于同步阻塞读取的帮助函数：读取指定从站的全部寄存器
uint32_t read_encoder_block(uint8_t slave_addr)
{
    modbus_t telegram;
    telegram.u8id = slave_addr;
    telegram.u8fct = MB_FC_READ_REGISTERS;
    telegram.u16RegAdd = 0x0000;
    telegram.u16CoilsNo = TOTAL_HOLDING_REGS;
    telegram.u16reg = resp_data_buff;
    // 发送并同步等待返回
    return ModbusQueryV2(&encoder_hub_client, telegram);
}

void encoder_hub_thread_non_block(void *argument)
{    

    bool pending = false;
    uint32_t notifyValue = 0;
    TickType_t xLastWakeTime = xTaskGetTickCount();
    const TickType_t xFrequency = pdMS_TO_TICKS(1000);
    static uint8_t current_addr = 0x00;

    for(;;)
    {
        // 发送异步查询请求
        if (!pending)
        {
        	current_addr = (current_addr + 1)%REQUEST_SLAVE_NUM; // 0 1 2
            if (read_encoder_non_block(current_addr + 1))
            {
                pending = true;
            }
        }
        
        // 处理查询响应
        if (xTaskNotifyWait(0, 0xFFFFFFFF, &notifyValue, 0) == pdTRUE)
        {
            pending = false;
            if (notifyValue != OP_OK_QUERY){
                RET_E("Failed to read sensors from slave: %lu\r\n", notifyValue);
            }else{
                for (int i = 0; i < 3; i++) {
                    encoder_data[current_addr * 3 + i] = (resp_data_buff[i] << 16) | resp_data_buff[i + 1];
                }
                RET_D("E(%d,%lu,%lu,%lu)\r\n", current_addr, encoder_data[current_addr * 3], 
                    encoder_data[current_addr * 3 + 1], encoder_data[current_addr * 3 + 2]);
            }
        }

        vTaskDelayUntil(&xLastWakeTime, xFrequency);
    }
}

void encoder_hub_thread_block(void *argument)
{
    const TickType_t xFrequency = pdMS_TO_TICKS(200);    // 4Hz = 250ms
    TickType_t xLastWakeTime = xTaskGetTickCount();

    for(;;)
    {
        // 发送阻塞读取请求
        static uint8_t current_addr = 0x00;
        current_addr = (current_addr+1)%3; // 0 1 2
        uint32_t err = read_encoder_block(current_addr + 1);

        // 处理读取结果
        if (err != OP_OK_QUERY){
            RET_E("Failed to read sensors from slave: %lu\r\n", err);
        }else{
            for (int i = 0; i < 3; i++) {
                encoder_data[current_addr * 3 + i] = (resp_data_buff[i] << 16) | resp_data_buff[i + 1];
            }
            RET_D("E(%d,%lu,%lu,%lu)\r\n", current_addr, encoder_data[current_addr * 3], 
                encoder_data[current_addr * 3 + 1], encoder_data[current_addr * 3 + 2]);
        }

        vTaskDelayUntil(&xLastWakeTime, xFrequency);
    }
}

void encoder_hub_init()
{
    init_encoder_hub();
    encoder_hub_task_handle = osThreadNew(encoder_hub_thread_block, NULL, &encoder_hub_task_attributes);
    RET_I("create encoder hub task\r\n");
}

```