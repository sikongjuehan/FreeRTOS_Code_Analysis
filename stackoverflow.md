栈溢出是一个很常见的软件问题。本文就来聊聊 FreeRTOS 中的栈溢出问题，以及其所相关的栈溢出检测机制。

FreeRTOS 在创建任务的时候都会为其开辟栈空间。任务创建函数中就有为任务指定栈空间大小的参数：usStackDepth。

```c
BaseType_t xTaskCreate(TaskFunction_t pxTaskCode,							//任务函数
                          const char *const pcName,							//任务名
                          const configSTACK_DEPTH_TYPE usStackDepth,		//栈大小
                          void * const pvParameters,						//任务参数
                          UBaseType_t uxPriority,							//优先级
                          TaskHandle_t * const pxCreateTask);				//任务句柄
```

正常情况下，函数的调用路径过长，超出了栈空间大小，就会导致栈溢出问题。这里强调的是正常情况下的栈溢出问题，当然也有不正常的情况，如栈结构被破坏了，或者函数调用出问题了等，也会导致栈溢出问题。

先说正常情况，FreeRTOS 提供了两种检测机制，都在 stack_macro.h 文件中。

第一种方法通过比较该任务的栈顶指针有没有超过该任务栈的限制来判断是否栈溢出了。因为在 TCB 中都记录了栈顶指针和栈空间等信息，这个是很容易实现的。如下图，假设栈的增长方向是从高地址到低地址，那么只要比较 pxTopOfStack （栈顶指针）和 pxStack（栈空间的顶部地址） 即可。

![](https://github.com/sikongjuehan/FreeRTOS_Code_Analysis/blob/main/res/stack_overflow/first.png)


第二种方法通过比较栈顶最后16个字节有没有被破坏来判断是否发生了栈溢出。如果使用该方法，就需要预先在栈顶空间填入特殊的字符用于栈溢出的检测。实际上，在初始化新任务的时候就将整个栈空间初始化为 0xa5U。

![](https://github.com/sikongjuehan/FreeRTOS_Code_Analysis/blob/main/res/stack_overflow/second.png)


不管是第一种方法还是第二种方法，都需要进行判断，那么什么时候判断合适呢？？答案是在任务切换的时候进行判断，即在 vTaskSwitchContext(void) 函数中。这个时候，当前任务执行完了，下一个任务还为开始执行，就可以对当前任务进行栈溢出检测了。



正常情况下的栈溢出问题，通过扩大栈空间就能解决了。但在不正常的情况下，比如栈结构被破坏了或者函数调用出问题了，扩大栈空间是没用的。这个时候最好的办法就是把当前任务栈空间给 dump 出来，具体做法就是在检测出栈溢出的时候，将当前任务的栈空间内容打印出来，查看函数调用链是否正常，需要根据具体内容进行分析，这里就不多做介绍了。



以上就是 FreeRTOS 中提供的栈溢出检测方法，完毕。



