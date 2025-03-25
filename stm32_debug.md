## STM32 SRAM调试方法

参考：

https://blog.csdn.net/goodrenze/article/details/123938272

https://doc.embedfire.com/mcu/stm32/f103badao/std/zh/latest/book/SRAM.html

1. 更改链接脚本.ld文件

   将FLASH重定向为SRAM的一部分。例如，原本的LD这部分内容为：

   ```ld
   MEMORY
   {
   RAM (xrw)      : ORIGIN = 0x20000000, LENGTH = 128K
   CCMRAM (xrw)      : ORIGIN = 0x10000000, LENGTH = 64K
   FLASH (rx)      : ORIGIN = 0x8000000, LENGTH = 512K
   }
   ```

   更改后为：

   ```ld
   MEMORY
   {
   RAM (xrw)      : ORIGIN = 0x20010000, LENGTH = 64K
   CCMRAM (xrw)      : ORIGIN = 0x10000000, LENGTH = 64K
   FLASH (rx)      : ORIGIN = 0x20000000, LENGTH = 64K
   }
   ```

   FLASH定义为RAM的前半部分，由于占用了RAM的一半空间，所以RAM大小缩减一半，地址也增加到64\*1024之后。

2. 更改中断向量表

   在system_stm32f4xx.c中更改中断向量表定义。启用用户中断向量表（开启USER_VECT_TAB_ADDRESS宏定义），同时启用SRAM中断向量表（启用VECT_TAB_SRAM宏定义），然后中断向量就会放在物理地址SRAM的开头。

3. 更改gdb启动前指令

   在launch.json中配置启动前指令：

   ```json
   "preLaunchCommands": [
                   "monitor reset halt",                        // OpenOCD 命令: 复位并停机
                   "monitor mww 0xE000ED08 0x20000000",         // OpenOCD 命令: 设置 VTOR (中断向量表)
                   "load",                                      // GDB 命令: 加载 ELF 到 SRAM
                   "set $sp = *(uint32_t*)0x20000000",          // GDB 命令: 设置 SP
                   "set $pc = *(uint32_t*)0x20000004"           // GDB 命令: 设置 PC (跳转到 Reset_Handler)
               ]
   ```

   其中`mww 0xE000ED08 0x20000000`是在设置寄存器`SCB->VTOR`（参见CMSIS/Include/core_cm4.h），这个地址是cortex-m4写死的。设置sp和pc是因为从sram reset时sp和pc可能是乱指的（参考野火文章）。