### 需求

串口接收程序采用中断，每次接收一个字节，将字节存入全局数组中，在串口接收中以连续两个自己0x27,0x03开头才开始接收数据，到连续两个字节0x30,0x72结束，使用vscanf()解析字符串的数据并将其存入共享变量a中。vtask()函数是freertos的任务，它将读取这个变量a，并将其更新到LCD中。

### 示例代码

```c
// 定义全局变量和缓冲区
char buffer[100];

// 定义队列句柄和共享变量
QueueHandle_t queue;

// 串口接收中断程序
void USART_ISR(void)
{
    static int index = 0;
  	char received_byte = USART_GetByte(USART1);

  	// 如果接收到连续两个字节0x27和0x03，则开始接收数据
  	if (index == 0 && received_byte == 0x27) {
    	buffer[index] = received_byte;
    	index++;
  	}
  	else if (index == 1 && received_byte == 0x03) {
    	buffer[index] = received_byte;
    	index++;
  	}
  	else if (index > 1) {
    	buffer[index] = received_byte;
        index++;
    	// 如果接收到连续两个字节0x30和0x72，则将数据存入队列
    	if (buffer[index-2] == 0x30 && buffer[index-1] == 0x72) {
      		buffer[index-2] = '\0'; // 将结尾标记替换为字符串结束符
      		int value;
      		sscanf(buffer, "%d", &value); // 使用sscanf()解析字符串
      		xQueueSendToBack(queue, &value, portMAX_DELAY); // 将数据存入队列
      		index = 0; // 重置缓冲区索引
    	}
  	}
}

// 任务vtask()
void vtask(void *pvParameters)
{
  int value;
  while(1) {
    // 读取队列中的数据并将其更新到LCD中
    if (xQueueReceive(queue, &value, portMAX_DELAY) == pdTRUE) {
      update_lcd(value);
    }
  }
}

// 主函数
int main()
{
  // 创建队列
  queue = xQueueCreate(10, sizeof(int));
  // 启动串口接收中断
  UART_ITConfig(USART1, UART_IT_RXNE, ENABLE);
  // 创建任务vtask()
  xTaskCreate(vtask, "vtask", 1000, NULL, 1, NULL);
  // 启动FreeRTOS调度器
  vTaskStartScheduler();
  return 0;
}

```

...

