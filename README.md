# zuart

Simple hardware-independent UART library

简易的硬件无关UART库, 支持多个串口, 用接收字符中断和空闲中断实现接收, 发送则可以选用阻塞方式或者中断方式.

在stm32上使用串口时当然是首选DMA方式. 但有时需要用到的串口很多, DMA通道不够用; 有时DMA通道被更重要的SPI等外设占用, 这时怎么办? 以及, 有时需要把串口程序从这个MCU移植到那个MCU, 改写起来非常麻烦. 有个硬件无关的串口库就方便多了.

## 配置流程

初始化用到的UART或USART, 并实现ReadByte和WriteByte两个函数, 其中参数source是用到的串口序号, 从0开始递增. 类似下面这样:

```
void USART_WriteByte(unsigned char c, int source)
{
    USART_TypeDef* usartx;
    if(source == 0)
        usartx = USART1;
    else if(source == 1)
        usartx = USART6;
    else
        return;

    while(USART_GetFlagStatus(usartx, USART_FLAG_TXE) == RESET);
    USART_SendData(usartx, c);
}

unsigned char USART_ReadByte(int source)
{
    if(source == 0)
        return USART1->DR;
    else if(source == 1)
        return USART6->DR;
    else
        return 0;
}
```

启用IDLE中断和RXNE中断. 再实现你的parser函数, 在里面对接收到的消息做进一步处理:

```
static void parser(const void* msg, int size, int source)
{
    ...
}          
```

把以上三个函数指针传递给zuart, 并在你的USART/UART初始化之前先初始化zuart:
```
    zu_cfg_t cfg;
    cfg.parser_f = parser;
    cfg.readbyte_f = USART_ReadByte;
    cfg.writebyte_f = USART_WriteByte;
    zu_init(&cfg);
```

最后, 在IDLE中断和RXNE中断服务程序里调用zu_rxne_irqhandler和zu_idle_irqhandler, 后面的参数是相应的串口序号, 和ReadByte/WriteByte函数的最后一个参数相对应.

## 使用方法

在主循环里调用zu_poll函数即可实现接收. 发送时调用zu_writedata函数.

如果需要中断方式发送, 在zuart.h里把ZUART_TX_MODE_INTERRUPT后面的0改为1即可. (未充分测试!)

## 支持的MCU

在支持空闲中断的STM32若干型号, STM8S, STM8L上均可使用. 在不支持空闲中断的AVR或51平台则需要用定时器手工实现空闲中断机制.

已测试过: STM32F051C8T6, STM32F401RCT6, STM32F103VET6, STM32F070CBT6, STM32F030C8T6, STM8L151G6U6, STM8S003F3U6.

## 注意事项

当MCU主频较低, 同时串口波特率较高(例如主频8M, 波特率500k), 此时工作会不正常, 这是因为RXNE中断过于频繁, MCU处理不过来了. 提高主频或者降低波特率即可解决.
