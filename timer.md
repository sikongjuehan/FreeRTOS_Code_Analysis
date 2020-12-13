在[《FreeRTOS 之任务调度》](https://link.zhihu.com/?target=http%3A//mp.weixin.qq.com/s%3F__biz%3DMzA3NDMwNjc5NA%3D%3D%26mid%3D2247483656%26idx%3D1%26sn%3Db9eef180ccbaa5929563a99ebc556ade%26chksm%3D9f00848da8770d9bba9dfe95d3a51e52125797bdf695c3b79433e8550d903235229ea8da9327%26scene%3D21%23wechat_redirect)一文中提到，硬件定时器是和硬件设计相关的，不同的芯片有不同的配置方法，通过中断方式触发执行，精确度高。相对于硬件定时器，FreeRTOS 中还提供了软件定时器。本文就来聊聊软件定时器是如何实现的，以及它的精度如何呢？？

首先看一个例子，如下的代码就是使用软件定时器，周期性的打印 “This is a softtimer” 信息。

```c
TimerHandle_t xSoftTimer = NULL;

//回调函数
static void vPrintTimer(TimerHandle_t xTimer)
{
	xTimer = xTimer;

	printf("This is a softtimer.\n");
}

//创建定时器
xSoftTimer = xTimerCreate("Timer", pdMS_TO_TICKS(INT_TEST_TIMER_PERIOD), pdTURE, NULL, vPrintTimer);
//开启定时器
xTimerStart(xSoftTimer, 0);
```



xTimerCreate() 函数用于创建一个定时器。其中， “时长” 是以 tick 为单位的，uxAutoReload 参数表示当定时器到期时，是否自动装载，等待下一次触发。

```c
lib/FreeRTOS/timers.c:

TimerHandle_t xTimerCreate(	const char * const pcTimerName,                  //名字
								const TickType_t xTimerPeriodInTicks,        //时长
								const UBaseType_t uxAutoReload,              //是否自动加载
								void * const pvTimerID,                      //编号
								TimerCallbackFunction_t pxCallbackFunction ) //回调函数
```

从函数的定义中可以看出，其参数就是一个定时器所包括的资源：

```c
lib/FreeRTOS/timers.c:

typedef struct tmrTimerControl {
    const char *pcTimerName;					//名字
    ListItem_t xTimerListItem;					//链表项，用于后面被加入到链表中进行管理
    TickType_t xTimerPeriodInTicks;				//时长
    UBaseType_t uxAutoReload;					//是否自动加载
    void *pvTimerID;							//编号
    TimerCallbackFunction_t pxCallbackFunction;	//回调函数
} xTIMER;
```

当创建一个定时器实际就是初始化一个 xTIMER 结构体的实例。首先为定时器分配内存，然后将 xTimerCreate()  函数的参数填入该定时器中。此时这些数据还是静静的躺在结构体中，并没有和系统进行交互。那么什么时候这些定时器和系统发生交集呢？？继续看。在创建一个定时器的时候，还会调用一个函数：

```text
prvCheckForValidListAndQueue(void);
```

这个函数的作用就是为定时器的管理提供一些基础设施。

1. 创建并初始化两个链表: xActiveTimerList1, xActiveTimerList2，用于管理 timer，判断是否超时。
2. 创建一个队列，其大小为 configTIMER_QUEUE_LENGTH。队列用于对 timer 本身的操作，如对一个 timer 是进行开启，停止，销毁等操作。



这里可能会好奇为什么要创建两个 list？要想回答这个问题，需要先看看，系统是如何判断定时器到期的？

![](https://github.com/sikongjuehan/FreeRTOS_Code_Analysis/blob/main/res/timer/ExpireTime.png)

当系统调用 xTimerStart() 开启一个 timer 的时候，会记录超时时间（调用该函数时的时间 + 定时时长）到 timer 中，并将  timer 插入到 list 上，升序排列，这样系统只需要比较 list 中第一个 timer  的超时时间和当前时间值。如果当前时间大于超时时间，说明该定时器到期，进而就会去处理定时器，也就是先从 list 中将该 timer  移除，然后执行该定时器的回调函数。如果小于，说明还没到期，等待下次比较。

时间是以 tick 数进行记录的，随着系统不断运行，记录该 tick 的变量是个整型数，不管是 32bit，还是 64bit 总有溢出的时候。在溢出的时候，如果 list 上还有 timer  没有被处理，而新的 timer 又被创建，这个时候如果将新 timer 插入到同一个 list 上，由于 list 上是按升序排列的，新加入的  timer 被插入到 list 的最前面，这就会导致溢出前的 timer 得不到执行。所以为了解决这个问题，引入第二个  list，将溢出后新创建的 timer 加入到第二个 list 中，而第一个 list 中的 timer 稍后再处理。这样就不影响了。

以上就解答了为什么需要两个 list 。



上面说，系统会不断的去比较当前时间和超时时间，那谁来做这个比较的事呢？？

在 FreeRTOS 中创建了一个 task 来处理。当系统开启调度（ xStartSchedule() ）的时候会创建一个 Timer Task。

```c
lib/FreeRTOS/timers.c:

BaseType_t xTimerCreateTimerTask(void)
{
    ...
    xTaskCreate (prvTimerTask, 												//任务函数
                 configTIMER_SERVICE_TASK_NAME, 							//任务名
                 configTIMER_TASK_STACK_DEPTH,								//栈大小
                 NULL,														//参数
                 ((UBaseType_t)configTIMER_TASK_PRIORITY)|portPRILEGE_BIT, 	//优先级
                 &xTimerTaskHandle);										//句柄
    ...
}

static void prvTimerTask(void *pvParameter)
{
    ...
    for ( ;; ) {
        //获取 list 上的第一个 timer
        xNextExpireTime = prvGetNextExpireTime(&xListWasEmpty);	
        //判断 timer 是否超时
        prvProcessTimerOrBlockTask(xNextExpireTime, xListWasEmpty);
        //处理 timer 指令
        prvProcessReceivedCommand();
    }
}
```

该 task 会不停的去处理链表上的 timer，但它不会一直占这 CPU。在 prvProcessTimerOrBlockTask()  函数中，判断 timer 是否超时了，如果没有超时，就让该 task 睡眠，让出 CPU 的执行权。等超时时间到了才会去处理链表上的  timer，首先将 timer 从 list 上移除，然后执行该 timer 的回调函数。如果该 timer 是自加载周期性的，那么  prvProcessReceivedCommand() 函数会将该 timer 重新加入到 list  中。当然了，prvProcessReceivedCommand() 函数除了能处理重新加载操作，还有停止，删除等操作。

可以看到，timer 的处理是在 task 中完成的，这就有个问题，如果该 task 优先级比较低，就很容易被抢占，导致 timer 并不能准时的被处理，也就是 timer 的精度是不能得到保证的。

有一点需要注意，在开启调度的时候会使用 list 和 queue 这些基础设施，而这些基础设施的初始化是在如下函数中完成的。

```c
prvCheckForValidListAndQueue(void);
```

这个函数在开启 timer 的时候会被调用。但定时器的创建不一定就在开始调度之前，这就需要在开始调度的时候也要调用该函数。基础设施只会被初始化一次，所以在该函数一开始会判断，是否已经被初始化，如果已经初始化了就直接跳过。



以上就是 FreeRTOS 中软件定时器的实现了。总结起来就是一点：

FreeRTOS 软件定时器的时长是以 tick 为单位的，而且基于 task 实现，可能会被抢占，所以精确度并不能得到保障。
