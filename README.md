# FreeRTOS_Code_Analysis
本系列记录的是对 FreeRTOS 源码的解析。目前 FreeRTOS 官网发布的最新版本是：10.4.1，可以从如下地址下载：
```https://github.com/FreeRTOS/FreeRTOS-Kernel```
由于 FreeRTOS 已经被 Amazon 收购，所以在 Amazon 也维护了一个 FreeRTOS 的源码版本：
```https://github.com/aws/amazon-freertos```
虽然是两套源码，但是 FreeRTOS 内核源码基本是一致的，所以本系列就以 FreeRTOS 官网发布的 10.4.1 版本为例进行分析。

主要介绍：调度，内存管理，同步机制，同时也会介绍基于 ARMv8，RISC-V 架构的 FreeRTOS 移植相关内容，最后介绍一些调试方法和工具。
