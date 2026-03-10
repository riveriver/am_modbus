#ifndef AM_MODBUS_CONFIG_H_
#define AM_MODBUS_CONFIG_H_

#include "main.h"

#define LOG_D(...)    LOG_DEBUG("MODBUS", __VA_ARGS__)
#define LOG_I(...)    LOG_INFO("MODBUS", __VA_ARGS__)
#define LOG_E(...)    LOG_ERR("MODBUS", __VA_ARGS__)

#define ENABLE_MODBUS_USART_DMA 1

#define T35  5              // Timer T35 period (in ticks) for end frame detection.
#define MAX_BUFFER  256	    // Maximum size for the communication buffer in bytes.
#define TIMEOUT_MODBUS 1000 // Timeout for master query (in ticks)
#define MAX_M_HANDLERS 4   //Maximum number of modbus handlers that can work concurrently
#define MAX_TELEGRAMS 6     //Max number of Telegrams in master queue

#endif /* AM_MODBUS_CONFIG_H_ */