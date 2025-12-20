##  一.工作模式/校准模式的切换
上电时SET接高电平运行在校准模式，否则运行在工作模式

- 校准模式：校准模式下，首先采集第一种颜色信息，请在上电后5s内将传感器摆放置合适位置。上电5s后开始采集第一种颜色数据。当蓝色指示灯由常量切换至熄灭状态，则第一种颜色数据采集完毕。需在10s内将传感器摆放至合适位置开始采集第二种颜色数据。当颜色信息采集结束并且数据写入完成，INS指示灯快速闪烁。此时可以断开连接，校准完成。

- 工作模式：工作模式下，INS蓝色指示灯1s闪烁一次。5路灰度传感器数据刷新率约1kHZ，闭环控制建议读取时间间隔>=1ms。内置滤波算法，即输出滤波数据。

---

## 二.IIC/串口模式的切换
默认为串口模式，波特率115200
IIC模式，时钟速率400k
在校准模式中，在第一种颜色采集前的5s准备时间中，如果将SCL引脚连接至GND，则传感器被设置为IIC模式，否则为串口模式
另外，支持并行数据读取，可以直接读取OUT1-OUT5端口电平以获取颜色识别状况

---

## 三.注意事项
灰度传感器在高度或光线强度变化较大时通过重新校准能提高识别准确率

---

## 四.程序设计参考
### 灰度传感器通信参数设置：
- IIC初始化参数
```c 
  void IIC_INIT()
  {
    /* I2C初始化 */
    I2cHandle.Instance             = I2C;     /* I2C */
    I2cHandle.Init.ClockSpeed      = 400000;  /* I2C通讯速度 */
    I2cHandle.Init.DutyCycle       = 0x4000;  /* I2C占空比DUTYCYCLE_16_9 */
    I2cHandle.Init.OwnAddress1     = 0xA0;    /* I2C地址 */
    I2cHandle.Init.GeneralCallMode = 0;       /* 禁止广播呼叫 */
    I2cHandle.Init.NoStretchMode   = 0;       /* 允许时钟延长 */
    if (HAL_I2C_Init(&I2cHandle) != HAL_OK)
    {
      Error_Handler();
    }
  }
```
- 串口初始化参数
```c
  void BSP_USART_Config(void)
  {
    GPIO_InitTypeDef GPIO_InitStruct;
    USART_CLK_ENABLE();
    DebugUartHandle.Instance = USART2;
    DebugUartHandle.Init.BaudRate = 115200;               //波特率115200
    DebugUartHandle.Init.WordLength = UART_WORDLENGTH_8B; //数据长度8位
    DebugUartHandle.Init.StopBits = UART_STOPBITS_1;      //停止位设置为1位
    DebugUartHandle.Init.Parity = UART_PARITY_NONE;       //无校验
    DebugUartHandle.Init.HwFlowCtl = UART_HWCONTROL_NONE; //无硬件流控制
    DebugUartHandle.Init.Mode = UART_MODE_TX;             //仅发送模式
    HAL_UART_Init(&DebugUartHandle);
  }
```

### 接收端程序设计参考：
  
- 串口接收例程 (测试环境：STMf103c8t6)

  连接说明：灰度传感器TX连接接收端串口RX，连接两者GND
```c
  #include "main.h"
  UART_HandleTypeDef huart1;
  DMA_HandleTypeDef hdma_usart1_rx;
  uint8_t ure[6]={0};
  uint8_t end = 0;

  void SystemClock_Config(void);
  static void MX_GPIO_Init(void);
  static void MX_DMA_Init(void);
  static void MX_USART1_UART_Init(void);

  int main(void)
  {
    HAL_Init();
    SystemClock_Config();
    MX_GPIO_Init();
    MX_DMA_Init();
    MX_USART1_UART_Init();
    while(end!=0xb5)//确保0xb5在数组末尾
    {
      HAL_UART_Receive(&huart1, &end, 1, 1000);
    }
    
    while (1)
    {
    HAL_UART_Receive_DMA(&huart1, ure, 6);//数据读取，建议实际使用时在定时器中定时读取
    }
  }
  void SystemClock_Config(void)
  {
    RCC_OscInitTypeDef RCC_OscInitStruct = {0};
    RCC_ClkInitTypeDef RCC_ClkInitStruct = {0};
    RCC_OscInitStruct.OscillatorType = RCC_OSCILLATORTYPE_HSE;
    RCC_OscInitStruct.HSEState = RCC_HSE_ON;
    RCC_OscInitStruct.HSEPredivValue = RCC_HSE_PREDIV_DIV1;
    RCC_OscInitStruct.HSIState = RCC_HSI_ON;
    RCC_OscInitStruct.PLL.PLLState = RCC_PLL_ON;
    RCC_OscInitStruct.PLL.PLLSource = RCC_PLLSOURCE_HSE;
    RCC_OscInitStruct.PLL.PLLMUL = RCC_PLL_MUL9;
    if (HAL_RCC_OscConfig(&RCC_OscInitStruct) != HAL_OK)
    {
      Error_Handler();
    }
    RCC_ClkInitStruct.ClockType = RCC_CLOCKTYPE_HCLK|RCC_CLOCKTYPE_SYSCLK
                                |RCC_CLOCKTYPE_PCLK1|RCC_CLOCKTYPE_PCLK2;
    RCC_ClkInitStruct.SYSCLKSource = RCC_SYSCLKSOURCE_PLLCLK;
    RCC_ClkInitStruct.AHBCLKDivider = RCC_SYSCLK_DIV1;
    RCC_ClkInitStruct.APB1CLKDivider = RCC_HCLK_DIV2;
    RCC_ClkInitStruct.APB2CLKDivider = RCC_HCLK_DIV1;

    if (HAL_RCC_ClockConfig(&RCC_ClkInitStruct, FLASH_LATENCY_2) != HAL_OK)
    {
      Error_Handler();
    }
  }
  /*串口初始化*/
  static void MX_USART1_UART_Init(void)
  {
    huart1.Instance = USART1;
    huart1.Init.BaudRate = 115200;
    huart1.Init.WordLength = UART_WORDLENGTH_8B;
    huart1.Init.StopBits = UART_STOPBITS_1;
    huart1.Init.Parity = UART_PARITY_NONE;
    huart1.Init.Mode = UART_MODE_RX;
    huart1.Init.HwFlowCtl = UART_HWCONTROL_NONE;
    huart1.Init.OverSampling = UART_OVERSAMPLING_16;
    if (HAL_UART_Init(&huart1) != HAL_OK)
    {
      Error_Handler();
    }

  }

  /**
    * Enable DMA controller clock
    */
  static void MX_DMA_Init(void)
  {

    /* DMA controller clock enable */
    __HAL_RCC_DMA1_CLK_ENABLE();

    /* DMA interrupt init */
    /* DMA1_Channel5_IRQn interrupt configuration */
    HAL_NVIC_SetPriority(DMA1_Channel5_IRQn, 0, 0);
    HAL_NVIC_EnableIRQ(DMA1_Channel5_IRQn);

  }

  static void MX_GPIO_Init(void)
  {
    __HAL_RCC_GPIOC_CLK_ENABLE();
    __HAL_RCC_GPIOD_CLK_ENABLE();
    __HAL_RCC_GPIOA_CLK_ENABLE();
  }

  void Error_Handler(void)
  {
    /* USER CODE BEGIN Error_Handler_Debug */
    /* User can add his own implementation to report the HAL error return state */
    __disable_irq();
    while (1)
    {
    }
    /* USER CODE END Error_Handler_Debug */
  }
```

- IIC接收例程 (测试环境Air001)

  连接说明：灰度传感器与接收端的SCL、SDA、GND对应连接，两者同时上电
```c
    #include "main.h"

  /* Private define ------------------------------------------------------------*/
  #define DARA_LENGTH       6                /* 数据长度 */
  #define I2C_ADDRESS        0xA0             /* 本机地址0xA0 */
  #define I2C_SPEEDCLOCK   600000             /* 通讯速度100K */
  #define I2C_DUTYCYCLE    I2C_DUTYCYCLE_2    /* 占空比 */

  /* Private variables ---------------------------------------------------------*/
  I2C_HandleTypeDef I2cHandle;
  uint8_t aTxBuffer[6] = {1, 2, 3, 4, 5,6};
  uint8_t aRxBuffer[6] = {0, 0, 0, 0, 0,0};

  /* Private user code ---------------------------------------------------------*/
  /* Private macro -------------------------------------------------------------*/
  /* Private function prototypes -----------------------------------------------*/
  static void APP_SystemClockConfig(void);
  static void APP_CheckEndOfTransfer(void);
  static uint8_t APP_Buffercmp8(uint8_t* pBuffer1, uint8_t* pBuffer2, uint8_t BufferLength);
  static void APP_LedBlinking(void);
  volatile uint32_t tims = 0;
  /**
    * @brief  应用程序入口函数.
    * @retval int
    */
  int main(void)
  {
    /* 复位所有外设，初始化flash接口和systick */
    HAL_Init();
    
    /* 初始化LED */
    BSP_LED_Init(LED_RED);

    
    /* 配置时钟 */
    APP_SystemClockConfig();
    
    /* I2C初始化 */
    I2cHandle.Instance             = I2C;                                   /* I2C */
    I2cHandle.Init.ClockSpeed      = I2C_SPEEDCLOCK;                        /* I2C通讯速度 */
    I2cHandle.Init.DutyCycle       = I2C_DUTYCYCLE;                         /* I2C占空比 */
    I2cHandle.Init.OwnAddress1     = I2C_ADDRESS;                           /* I2C地址 */
    I2cHandle.Init.GeneralCallMode = I2C_GENERALCALL_DISABLE;               /* 禁止广播呼叫 */
    I2cHandle.Init.NoStretchMode   = I2C_NOSTRETCH_DISABLE;                 /* 允许时钟延长 */
    if (HAL_I2C_Init(&I2cHandle) != HAL_OK)
    {
      Error_Handler();
    }
    
      HAL_GPIO_WritePin(GPIOB, GPIO_PIN_1, GPIO_PIN_SET);
    HAL_GPIO_WritePin(GPIOB, GPIO_PIN_0, GPIO_PIN_SET);
    HAL_GPIO_WritePin(GPIOB, GPIO_PIN_3, GPIO_PIN_SET);
    
    while(1)
    {
      /*I2C从机DMA方式接收*/
      while (HAL_I2C_Slave_Receive_DMA(&I2cHandle, (uint8_t *)aRxBuffer, DARA_LENGTH) != HAL_OK)
      {
      Error_Handler();
      }
      /*判断当前I2C状态*/
      while (HAL_I2C_GetState(&I2cHandle) != HAL_I2C_STATE_READY)
      {
      }
      tims++;
      if(tims>=500) 
      {
        tims=0;
        HAL_GPIO_TogglePin(GPIOB, GPIO_PIN_1);
        HAL_GPIO_TogglePin(GPIOB, GPIO_PIN_0);
        HAL_GPIO_TogglePin(GPIOB, GPIO_PIN_3);
      }
      
    }
  }

  /**
    * @brief  系统时钟配置函数
    * @param  无
    * @retval 无
    */
  static void APP_SystemClockConfig(void)
  {
  RCC_OscInitTypeDef RCC_OscInitStruct = {0};
    RCC_ClkInitTypeDef RCC_ClkInitStruct = {0};

    RCC_OscInitStruct.OscillatorType = RCC_OSCILLATORTYPE_HSE | RCC_OSCILLATORTYPE_HSI \
                                    | RCC_OSCILLATORTYPE_LSI | RCC_OSCILLATORTYPE_LSE;  /* 配置HSE、HSI、LSI、LSE */
    RCC_OscInitStruct.HSIState = RCC_HSI_ON;                                             /* 开启HSI */
    RCC_OscInitStruct.HSIDiv = RCC_HSI_DIV1;                                             /* HSI不分频 */
    RCC_OscInitStruct.HSICalibrationValue = RCC_HSICALIBRATION_24MHz;                     /* HSI校准频率24MHz */
    RCC_OscInitStruct.HSEState = RCC_HSE_OFF;                                            /* 关闭HSE */
    /* RCC_OscInitStruct.HSEFreq = RCC_HSE_16_32MHz; */                                  /* HSE频率范围16~32MHz */
    RCC_OscInitStruct.LSIState = RCC_LSI_OFF;                                            /* 关闭LSI */
    RCC_OscInitStruct.LSEState = RCC_LSE_OFF;                                            /* 关闭LSE */
    /* RCC_OscInitStruct.LSEDriver = RCC_LSEDRIVE_MEDIUM; */                             /* 默认LSE驱动能力 */
    RCC_OscInitStruct.PLL.PLLState = RCC_PLL_OFF;                                        /* 关闭PLL */
    /* RCC_OscInitStruct.PLL.PLLSource = RCC_PLLSOURCE_NONE; */                          /* PLL无时钟源 */
    /* 配置振荡器 */
    if (HAL_RCC_OscConfig(&RCC_OscInitStruct) != HAL_OK)
    {
      Error_Handler();
    }

    RCC_ClkInitStruct.ClockType = RCC_CLOCKTYPE_HCLK | RCC_CLOCKTYPE_SYSCLK | RCC_CLOCKTYPE_PCLK1;/* 配置SYSCLK、HCLK、PCLK */
    RCC_ClkInitStruct.SYSCLKSource = RCC_SYSCLKSOURCE_HSI;                                        /* 配置系统时钟为HSI */
    RCC_ClkInitStruct.AHBCLKDivider = RCC_SYSCLK_DIV1;                                            /* AHB时钟不分频 */
    RCC_ClkInitStruct.APB1CLKDivider = RCC_HCLK_DIV1;  
    /* 配置时钟源 */
    if (HAL_RCC_ClockConfig(&RCC_ClkInitStruct, FLASH_LATENCY_0) != HAL_OK)
    {
      Error_Handler();
    }
  }

  /**
    * @brief  校验数据函数
    * @param  无
    * @retval 无
    */
  static void APP_CheckEndOfTransfer(void)
  {
      HAL_GPIO_WritePin(GPIOB, GPIO_PIN_1, GPIO_PIN_SET);
    HAL_GPIO_WritePin(GPIOB, GPIO_PIN_0, GPIO_PIN_SET);
    HAL_GPIO_WritePin(GPIOB, GPIO_PIN_3, GPIO_PIN_SET);
  }


  /**
    * @brief  LED灯闪烁
    * @param  无
    * @retval 无
    */
  static void APP_LedBlinking(void)
  {
    while (1)
    {
      BSP_LED_Toggle(LED_RED);; 
      HAL_Delay(500);
    }
  }

  /**
    * @brief  错误执行函数
    * @param  无
    * @retval 无
    */
  void Error_Handler(void)
  {
    while (1)
    {
    }
  }
```
- 并行数据接收例程 (测试环境：STMf103c8t6)
  
  连接说明：配置单片机5个GPIO引脚为input模式，对应连接到灰度传感器OUT1-OUT5端口
```C
  #include "main.h"
  void SystemClock_Config(void);
  static void MX_GPIO_Init(void);
  uint8_t ure[5]={0};

  int main(void)
  {
    HAL_Init();
    SystemClock_Config();
    MX_GPIO_Init();
    while (1)
    {
      ure[0]=HAL_GPIO_ReadPin(GPIOA, GPIO_PIN_0);
      ure[1]=HAL_GPIO_ReadPin(GPIOA, GPIO_PIN_1);
      ure[2]=HAL_GPIO_ReadPin(GPIOA, GPIO_PIN_2);
      ure[3]=HAL_GPIO_ReadPin(GPIOA, GPIO_PIN_3);
      ure[4]=HAL_GPIO_ReadPin(GPIOA, GPIO_PIN_4);
    }
  }

  void SystemClock_Config(void)
  {
    RCC_OscInitTypeDef RCC_OscInitStruct = {0};
    RCC_ClkInitTypeDef RCC_ClkInitStruct = {0};
    RCC_OscInitStruct.OscillatorType = RCC_OSCILLATORTYPE_HSE;
    RCC_OscInitStruct.HSEState = RCC_HSE_ON;
    RCC_OscInitStruct.HSEPredivValue = RCC_HSE_PREDIV_DIV1;
    RCC_OscInitStruct.HSIState = RCC_HSI_ON;
    RCC_OscInitStruct.PLL.PLLState = RCC_PLL_ON;
    RCC_OscInitStruct.PLL.PLLSource = RCC_PLLSOURCE_HSE;
    RCC_OscInitStruct.PLL.PLLMUL = RCC_PLL_MUL9;
    if (HAL_RCC_OscConfig(&RCC_OscInitStruct) != HAL_OK)
    {
      Error_Handler();
    }

    RCC_ClkInitStruct.ClockType = RCC_CLOCKTYPE_HCLK|RCC_CLOCKTYPE_SYSCLK
                                |RCC_CLOCKTYPE_PCLK1|RCC_CLOCKTYPE_PCLK2;
    RCC_ClkInitStruct.SYSCLKSource = RCC_SYSCLKSOURCE_PLLCLK;
    RCC_ClkInitStruct.AHBCLKDivider = RCC_SYSCLK_DIV1;
    RCC_ClkInitStruct.APB1CLKDivider = RCC_HCLK_DIV2;
    RCC_ClkInitStruct.APB2CLKDivider = RCC_HCLK_DIV1;

    if (HAL_RCC_ClockConfig(&RCC_ClkInitStruct, FLASH_LATENCY_2) != HAL_OK)
    {
      Error_Handler();
    }
  }

  static void MX_GPIO_Init(void)
  {
    GPIO_InitTypeDef GPIO_InitStruct = {0};

    /* GPIO Ports Clock Enable */
    __HAL_RCC_GPIOC_CLK_ENABLE();
    __HAL_RCC_GPIOD_CLK_ENABLE();
    __HAL_RCC_GPIOA_CLK_ENABLE();

    /*Configure GPIO pins : PA0 PA1 PA2 PA3 PA4 */
    GPIO_InitStruct.Pin = GPIO_PIN_0|GPIO_PIN_1|GPIO_PIN_2|GPIO_PIN_3
                            |GPIO_PIN_4;
    GPIO_InitStruct.Mode = GPIO_MODE_INPUT;
    GPIO_InitStruct.Pull = GPIO_PULLUP;
    HAL_GPIO_Init(GPIOA, &GPIO_InitStruct);
  }

  void Error_Handler(void)
  {
    /* USER CODE BEGIN Error_Handler_Debug */
    /* User can add his own implementation to report the HAL error return state */
    __disable_irq();
    while (1)
    {
    }
    /* USER CODE END Error_Handler_Debug */
  }
```

  


### 校准模式灯光
- 上电准备2————>常亮
- 第一种颜色3————>快闪
- 采集成功5————>熄灭
- 第二种颜色3————>快闪
- 完成————>熄灭

### 工作模式灯光
上电快闪1S
- 串口————>闪亮一次熄灭
- IIC————>闪亮两次熄灭
