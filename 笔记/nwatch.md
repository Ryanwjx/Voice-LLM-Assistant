# 0 裸机运行原理

## 1. 启动

```
复位后，BOOT01的值被锁存，进而选择启动模式
```

![image-20241201111729453](E:\0_wjx_file\embedded_ai\study_data\0 note\project\nwatch\image-20241201111729453.png)

```
1. 主闪存存储器启动
主闪存存储器(0x0800 0000)被映射到启动空间(0x0000 0000)
此区域即内置FLASH，一般使用JTAG或者SWD模式下载烧录

2.从系统存储器启动
系统存储器(联网0x1FFF B000，其它0x1FFF F000)被映射到启动空间(0x0000 0000)
此启动方式少，因为此区域存储出场的BootLoader程序；但BootLoader中提供串口下载程序，帮助下载到FLASH

3.从内置SRAM启动
只能在0x2000 0000开始的地址区访问SRAM。
一般用于调试
```

```
在启动延迟之后， CPU从地址0x0000 0000获取堆栈顶的地址，并从启动存储器的0x0000 0004（Reset_Handler，0x0是堆起始地址）指示的地址开始执行代码。
```

## 2. 代码的链接

```
linux链接使用.ld文件，keil使用默认的.sct文件

LR_IROM1 0x08000000 0x00010000  {    ; load region size_region
  ER_IROM1 0x08000000 0x00010000  {  ; load address = execution address
   *.o (RESET, +First)		//RESET区即中断向量表，要求+First放在开头
   *(InRoot$$Sections)		//
   .ANY (+RO)				//RO区，Const和Code
   .ANY (+XO)
  }
  RW_IRAM1 0x20000000 0x00005000  {  ; RW data
   .ANY (+RW +ZI)			//RW区，全局初始化、静态初始化
   							//ZI区，全局未初始化、静态未初始化
  }
}
```

## 3. 汇编代码的运行逻辑

![image-20241201124306577](E:\0_wjx_file\embedded_ai\study_data\0 note\project\nwatch\image-20241201124306577.png)

```
ARM架构内核，使用对应的汇编指令集（支持ARM、Thumb两种）
```

```
STM32F103的ARM Cortex-M3内核使用Armv7-M架构与对应指令集
伪指令（通知编译器的指令）：
```

| **伪指令助记符** | **格式**                  | **功能说明**                                                 |
| ---------------- | ------------------------- | ------------------------------------------------------------ |
| AREA             | AREA 段名 属性1，属性2…   | 定义一个代码段或者数据段                                     |
| SPACE            | 空间名   SPACE   空间大小 | 分配内存空间                                                 |
| ENTRY            | ENTRY                     | 汇编程序的入口点                                             |
| END              | END                       | 汇编程序的程序的结尾                                         |
| EQU              | 常量名 EQU 常量数值       | 定义一个常量，相当于c语言中的 define                         |
| EXPORT           | EXPORT 标号               | 声明一个全局的标号，可以在其他文件使用声明的标号             |
| IMPORT           | IMPORT 标号               | 通知编译器要使用一个在其他文件中定义的标号，相当于c语言中的 extern |
| DCD              | DCD 表达式                | 用于分配一片连续的字存储单元并用伪指令中指定的表达式初始化，表达式可以是程序标号或数字表达式 |
| PROC             | PROC                      | 表示一个汇编子程序的开始。相当于C语言中函数开始。            |
| ENDP             | ENDP                      | 表示一个汇编子程序的结束。相当于C语言中函数结束。            |

```
指令：
。。。
```

```
//以startup为例
Stack_Size		EQU     0x400
                AREA    STACK, NOINIT, READWRITE, ALIGN=3	//STACK段，放入ZI
Stack_Mem       SPACE   Stack_Size
__initial_sp

Heap_Size      EQU     0x200
                AREA    HEAP, NOINIT, READWRITE, ALIGN=3	//HEAP段，放入ZI
__heap_base
Heap_Mem        SPACE   Heap_Size
__heap_limit
                PRESERVE8
                THUMB

                AREA    RESET, DATA, READONLY				//RESET段，放入RW，被链接器指定放在0x08000000开头
                EXPORT  __Vectors
                EXPORT  __Vectors_End
                EXPORT  __Vectors_Size
__Vectors       DCD     __initial_sp               ; Top of Stack
                DCD     Reset_Handler              ; Reset Handler
                DCD     NMI_Handler                ; NMI Handler
                DCD     HardFault_Handler          ; Hard Fault Handler
                ...

				AREA    |.text|, CODE, READONLY				//.text段，放入RO
Reset_Handler    PROC
                 EXPORT  Reset_Handler             [WEAK]
     IMPORT  __main
     IMPORT  SystemInit
                 LDR     R0, =SystemInit	//执行自行定义的时钟初始化
                 BLX     R0
                 LDR     R0, =__main		//执行__main进行重定向，调用__rt_entry初始化堆栈，调用main
                 BX      R0
                 ENDP
                 ...
```

<img src="E:\0_wjx_file\embedded_ai\study_data\0 note\project\nwatch\a3d51b6d89f10261528ccc517339f362.png" alt="img" style="zoom:67%;" />

## 4. 总结启动流程

```
startup.S以及各文件将代码分为RO、RW、ZI等代码段
startup.S中RESET分区即RW代码段被放在整个可执行文件开头0x0
```

## 5. 基础初始化

```
HAL_Init();
	HAL_NVIC_SetPriorityGrouping(NVIC_PRIORITYGROUP_4);	//中断初始化
	HAL_InitTick(TICK_INT_PRIORITY);					//SYSTICK初始化
	HAL_MspInit();									//基础外设初始化
		__HAL_RCC_AFIO_CLK_ENABLE();	//复用io时钟
  		__HAL_RCC_PWR_CLK_ENABLE();		//电源控制器时钟
SystemClock_Config();											//时钟调整到对应频率，设置各个总线频率

//各外设初始化与使用
```

# 三 FreeRTOS任务创建

```
裸机程序就是将各个动作分解为流：动画流、声音流，以及时间、按键本身就是依照时间变化的流
需要关注如何分解为流

多任务程序就是将各个流当作一个完整体：动画任务、声音任务
需要关注如何互斥访问资源、阻塞等待
```

## 任务结构

```
分析可知任务交互关系:
button: display tune
display：button time
tune：button
time：display
```

## 任务互斥与阻塞

```
tune
    tune阻塞等待设置/根据任务状态动态创建播放任务，任务请求vs任务修改，队列实现
    音频数据互斥访问，（存在必然先后逻辑，先删除或终止任务，才创建新任务）
time
	systick阻塞等待变化，定时器中断修改vs日期数据修改，任务通知
	时间数据互斥访问，时间数据获取vs设置，互斥量实现
button
	控制数据阻塞等待，底层中断vs数据获取，队列集实现
	button函数互斥访问，函数设置vs执行互斥，互斥量
display
	display函数互斥访问，函数设置vs执行互斥，互斥量实现
```

## button和display互斥与阻塞的细节

```
button和display要有先后，button按下更新完所有数据后触发display切换

```





# 三 FreeRTOS修改过程（编写思路，错误修改，经验总结）

## 1. 经验总结（例子只是例子，需要提取抽象）

### 1. 1工程级

=》起步阶段，需要做好笔记记录，整理好“设计思路” + “问题解决思路”（需要当日完成，过后很难回忆）

=》起步阶段，做好版本管理，记录当前版本的进步

=》找到问题并解决问题的能力，不能慌不择路

​	例子1：考虑菜单如何引入操作系统

​		暂时未完成：一般程序员思路是什么？gpt学习

=》在探索具体内容时，每个小部分写总结，便于生成宏观思想

​	例子1：game1、torch在退出时，都存在按键和显示任务的先后问题，退出进行了设置与清理，可能在draw中会导致设置失效等。stopwatch中退出，需要等动画结束才设置按键调用函数，所以动画执行过程中重复清理，这与game1关动画前情况一致。

​		需要考虑各个具体模块的问题，提出统一解决方法。或基于各个问题的解决方法，总结一个新的方案。

===》mdk总是找不到一些文件，并不是没包含头文件路径/不存在，而是路径太长了导致错误

### 1.2 系统级

=》操作系统vs裸机

​	裸机多任务，外部修改状态，任务交替循环，在一条时间线考虑

​	操作系统多任务，应用层面相互并行，在多条时间线考虑。系统层面，任务就绪、休眠、阻塞等状态切换，在一条时间线考虑。

​		另外程序运行以任务为考虑单位，而非功能，尽管有些代码写在其他功能内，但是是被其他任务调用的。

​		要考虑任务调度时，各任务间变量的安全性。

=》如果引入系统，底层的操作难以做到完全隔离

​	例子：在按键控制中，需要引入队列使得顶层可阻塞读取，底层持续写入

​		故而在底层引入了队列的初始化、中断写入队列等

=》目前思路，基于现有框架，每个更新任务都创建一个任务，另外需要考虑任务的创建与删除

​	比如只需运行显示，按键，时间，声音等必须任务

​	其余小任务等待唤醒与删除

=》纠结使用---互斥同步工具：队列、信号量（互斥量）、事件组、任务通知等

​	各自适应的场合：任务通知更快，内存占用少

​	例子：定时器启动与停止，使用任务通知

=》如何使用---互斥同步工具：提升系统安全性

1. 使用目的：区分变量会不会被不同任务调用，会的话增加互斥

2. 在哪儿使用：

   任务实现功能的一般方法：功能特有变量、其他功能（作为辅助如时间）的变量，完成某个具体功能（如画面显示+时间）（比如按键任务贯穿menu、games、game1的具体函数指针设置）

​	可不在任务层面实现互斥，直接在各个功能里实现互斥，当多个任务调用该功能时，自然可互斥调用

3. 如何使用：

   当使用互斥同步工具时，需要将多个任务在一条时间线上考虑。当不用时，需要将各任务并行考虑。

   例子1：torch功能中，Button调用exit函数，Display调用draw函数，是并行的

   ​	但是发现exit函数设置屏幕，和draw设置屏幕有冲突，要求exit在draw执行完成后才运行，并要求exit后draw不再执行，是串行

=》系统任务的灵活创建与删除

1. 自杀与他杀---只能杀一次，删除一个已经被删除的 / NULL的，出现hard fault

   详细看2.2.5音频播放任务自杀 VS 被音频设置任务他杀,，2.3 TuneTask和DisplayTask---内存泄漏
   
   通过一个任务句柄(指针)pxTuneSegmentTaskHandler == NULL作为任务删除的依据 / 一个任务的代号 / 互斥访问的变量，可以将自杀、他杀的互斥，转换为更清晰的他杀他杀互斥。
   
   具体的，他杀任务先判断pxTuneSegmentTaskHandler == NULL，再TaskDelete(pxTuneSegmentTaskHandler)，后设pxTuneSegmentTaskHandler = NULL。自杀任务不必判断，直接设pxTuneSegmentTaskHandler = NULL，再TaskDelete(NULL)

=》防止内存泄漏，优化内存使用

1. 使用freetros的内置堆计算方式heap4，其中提供pvPortMalloc，vPortFree，相比c库的malloc可减少碎片化内存

	blocks = pvPortMalloc(BLOCK_COUNT*sizeof(bool));
	memset(blocks, 0, BLOCK_COUNT*sizeof(bool));//设置为0

2. 尽量减少malloc的使用，如果使用一定要确保每一个malloc都被free

=》系统函数学习.

eTaskState eTaskGetState( TaskHandle_t xTask )需要传入已经初始化的TaskHandle_t  

vTaskDelete( TaskHandle_t xTask )若TaskHandle_t为无效/已经删除的，会进入Hard Fault

configASSERT( xTaskToNotify );如果xTaskToNotify 为无效，则关中断且死循环

TaskHandle_t 就是一个指向TCB结构体的指针，TCB结构体被放入队列中，TaskHandle_t=NULL并不影响。

vPortFree不可重复对同一个内存进行free

## 2. FreeRTOS编写思路与错误修改（例子就是要掌握的本领）

### 2.1 Button和Display功能=>菜单切换 

#### 2.1.1 原始功能思路

1. 初始化

   ``` 初始化代码
   buttons_init();	//初始化按键硬件，设置静态按键组的声音
   
   LCD_Init();		//初始化硬件
   LCD_Clear();
   draw_init();	//设置draw内 framebuffer全局变量
   ```

2. 设置页面

   ``` 页面初始化函数
   display_set(watchface_normal);		//设置页面初始化函数指针 
   display_load();						//执行初始化函数
   									//设置按键调用函数， display更新时调用draw函数
   ```

3. 循环检测输入与更新页面

   ``` 更新与检测
   buttons_update();	//读取按键buf并执行对应的函数
   display_update();	//Button和Display功能=>菜单切换 
   ```

#### 2.1.2 FreeRTOS编写思路

1. 维护一个队列，中断写入队列，外部阻塞读取队列
   1. 了解红外的输出接口，一次有效数据=data+dev
   2. ir底层创建本地队列
   3. ir底层写入本地队列
   4. 提供接口顶层调用
   5. button应用层阻塞调用
   6. 阻塞调用后处理需要修改
      1. 原来为判断是否上次运行完成，删去简化

#### 2.1.3 编写过程与错误修改

1. 按键与红外引入了队列，可以顺利写入，但是无法读取并执行按键操作

​		调试发现读取阻塞成功，但按键处理里存在逻辑判断防止反复执行(原裸机程序为循环执行，该判断必须，现为系统阻塞，非必须)，滤除了按键操作

​		总结例子1：要先调试找问题，发现关键是获取了红外但没有执行按键程序，思考原程序为裸机循环导致程序的编写逻辑，思考系统使用阻塞可无需该逻辑，故删除逻辑判断。

#### 2.1.4 =>button队列集控制---rotary + ir

由于红外遥控不灵敏，加入转动编码器读取。本来打算读取左+右+按下，但是stm32f103只支持5-10、11-15的中断，按下键无法实现中断写队列，故只得读取右+按下

队列集要配置 #define configUSE_QUEUE_SETS 1

队列集读取是读取一个队列handler，还要再根据handler读取数据

#### 2.1.5 对抗/防止button按键函数重复执行---Game1退出内存重复free

按键退出功能中  A。实现了本地功能；B。设置了display任务的动画变量，只有display任务的动画结束以后才会重新设置按键调用函数。当执行btn_exit完成后由于动画未结束按键未重设，可能立马再次执行btn_exit，导致重复执行的错误。

在menu逻辑下，button和display的任务关系为：button设置display的函数，调用动画以及动画结束后调用的函数。display动画结束后可能设置button的函数。

1. 思路一：消除重复进入的影响---重复进入导致vportfree空内存

如在exit内free本地内存，而内存地址未被改为null，会在vportfree内删两次导致错误。

就算vportfree后将指针设为null，重复运行不会因为vportfree内删两次导致错误。但是若设置了display任务的动画变量，会导致退出动画中display调用被free过的内存，而出现hard fault。

让按键free block  在  game1播放调用block 之后，而且按键释放必须等待game1的播放结束。通过顺序安排实现互斥，可以在动画结束后的函数中进行block的释放。(需要改原作者程序结构)

2. 思路二：防止重复进入。

按键退出功能中  A。实现了本地功能；B。不设置display动画变量，直接设置新的按键调用函数、画面设置函数。

上述通过顺序避免了冲突，对于并行运行？？？如何考虑 动画 和 按键的互斥

#### =>2.1.6 Button互斥与死锁---任务内进行互斥

1.  死锁

Mutex被Func占用；Mutex被SetFunc申请

<img src="E:\0_wjx_file\embedded_ai\study_data\0 note\project\nwatch\249245B3DE2B98588C85F3DFBB5583C4.png" alt="249245B3DE2B98588C85F3DFBB5583C4" style="zoom: 25%;" />

“T”被D占用；“U”被D申请；"U"被C占用；“T”被C申请

<img src="E:\0_wjx_file\embedded_ai\study_data\0 note\project\nwatch\image-20240920105805308.png" alt="image-20240920105805308" style="zoom:50%;" />

2. 死锁原因

任务申请的资源被另外的任务上锁，而另外的任务申请的资源也被当前任务上锁

2. 死锁解决---不在任务内进行互斥

按键func的使用 vs setFuncs设置按键func。

大部分情况是button任务执行func，并直接调用setFuncs重新设置func。

小部分情况是button任务执行func，设置display动画以及动画结束函数，display任务内调用任务结束函数，其中用setFuncs重新设置func。如：animation_start(display_load, ANIM_MOVE_OFF);    其中display_load内调用了display_set设置函数，其中有setFuncs

即实现button任务 vs display任务内，func的设置与运行互斥，而且要避免button任务内进行互斥。

实现两套buttons_setFuncs函数供外部使用，对于确定是被button任务调用的无需加入互斥，对于确定是display等其他任务调用的，加入互斥。

但是上述方法过于繁杂，可以直接在互斥之前，获取当前任务名字进行判断，再进行互斥。

### =>2.3 TuneTask和DisplayTask---内存泄漏

#### 2.3.1 多次调用声音播放后，声音无法播放

当多次存在按键调用声音播放后，声音无法播放的情况，g_xQueueTune队列满，可能是xSemaphoreTake阻塞了。通过加入打印调试信息，验证

```
//结束当前任务
printf("tune_refresh mutex get\r\n");
xSemaphoreTake(xMutexTuneCounting,portMAX_DELAY);//有时候会被检测到未删除任务，但是到vTaskDelete又出错,防止中间任务切换
if(pxTuneSegmentTaskHandler!=NULL)	//保证第一次执行不错
{
        printf("pxTuneSegmentTask Delete Passively\r\n");
        tune_stop();
        vTaskDelete(*pxTuneSegmentTaskHandler);	//已经运行完，删除了就不删第二次	
        pxTuneSegmentTaskHandler = NULL;
        stop();
}
xSemaphoreGive( xMutexTuneCounting );
printf("tune_refresh mutex release\r\n");
----------
printf("pxTuneSegmentTask mutex get\r\n");
xSemaphoreTake(xMutexTuneCounting,portMAX_DELAY);
pxTuneSegmentTaskHandler = NULL; //设置为被杀掉，其他任务无需再杀
xSemaphoreGive( xMutexTuneCounting );
printf("pxTuneSegmentTask mutex release\r\n");
printf("pxTuneSegmentTask Delete\r\n");
vTaskDelete(NULL);
```

输出结果如下便再无声音

```
pxTuneSegmentTask mutex get
pxTuneSegmentTask mutex release
pxTuneSegmentTask Delete
---子任务结束 自杀
tune_refresh mutex get
tune_refresh mutex release
---tune任务 创建
tune_refresh mutex get
tune_refresh Delete Passively
---tune任务 杀死上一个子任务，没有释放
？？？再次进入 某个子任务
pxTuneSegmentTask mutex get
```

增加调试信息

```
//结束当前任务
if (xSemaphoreTake(xMutexTuneCounting, portMAX_DELAY) == pdTRUE)
{	
    if(pxTuneSegmentTaskHandler!=NULL)	//保证第一次执行不错
    {
            tune_stop();
            vTaskDelete(*pxTuneSegmentTaskHandler);	//已经运行完，删除了就不删第二次	
            pxTuneSegmentTaskHandler = NULL;
            stop();
    }

    xSemaphoreGive( xMutexTuneCounting );
    //开启新任务
    prio = mrequest.priority;
    tune = mrequest.tune;
    vol = mrequest.vol;
    pxTuneSegmentTaskHandler = malloc(sizeof(TaskHandle_t));

    if (pxTuneSegmentTaskHandler == NULL)
    {
            printf("Memory allocation failed\r\n");
            printHeapStatus();
            continue;
    }

    if (xTaskCreate(pxTuneSegmentTask, "pxTuneSegmentTask", 128, NULL, osPriorityNormal, pxTuneSegmentTaskHandler) != pdPASS)
    {
            printf("Task creation failed\n");
            free(pxTuneSegmentTaskHandler);
            pxTuneSegmentTaskHandler = NULL;
    }
}
else
{
        printf("Failed to take mutex in tune_refresh\n");
}
```

结果为

```
Memory allocation failed
Free heap size: 4792
Minimum ever free heap size: 4168
```

可见pxTuneSegmentTaskHandler指针获取新内存失败，出现后续未知错误

#### 2.3.2 反复进退出game1，blocks模块的值不正确，

![image-20240826132951362](E:\0_wjx_file\embedded_ai\study_data\0 note\project\nwatch\image-20240826132951362.png)

随后当小球碰到方块，要修改值时，出现了display内的hard fault，增加调试信息

```
blocks = calloc(BLOCK_COUNT, 1);
if (blocks == NULL)
{
        printf("blocks Memory allocation failed\r\n");
        printHeapStatus();
        return;
}
```

结果为

```
blocks Memory allocation failed
Free heap size: 4792
Minimum ever free heap size: 3544
```

#### 2.3.3 TuneTask内存泄漏分析

可见总的栈空间还有，考虑减少碎片化，并提高内存分配的效率。

在tune中改用pvPortMalloc，vPortFree，可以使用得更久，但还是会导致内存使用殆尽

```
pxTuneSegmentTaskHandler = pvPortMalloc(sizeof(TaskHandle_t));		
if (pxTuneSegmentTaskHandler == NULL)
{
        printf("Memory allocation failed\r\n");
        printHeapStatus();
        continue;
}
if (xTaskCreate(pxTuneSegmentTask, "pxTuneSegmentTask", 128, NULL, osPriorityNormal, pxTuneSegmentTaskHandler) != pdPASS)
{
        printf("Task creation failed\r\n");
        vPortFree(pxTuneSegmentTaskHandler);
        pxTuneSegmentTaskHandler = NULL;
        printHeapStatus();
}
```

输出调试信息为

```
Task creation failed
Free heap size: 552
Minimum ever free heap size: 0
```

可见存在内存泄漏，空闲任务并未释放完成，等待片刻后释放完成，可再次播放音乐！发现当不断创建音乐播放任务时，栈空间在不断减小，不创建时仅恢复一部分。

```
--------------不断创建播放任务，持续下降-------------
Free heap size: 1640
Minimum ever free heap size: 1640
Free heap size: 1624
Minimum ever free heap size: 1624
Free heap size: 1608
Minimum ever free heap size: 1608
Free heap size: 1592
Minimum ever free heap size: 1592
Free heap size: 1576
Minimum ever free heap size: 1576
Free heap size: 1560
Minimum ever free heap size: 1560
Free heap size: 1544
Minimum ever free heap size: 1544
Free heap size: 1528
Minimum ever free heap size: 1528
Free heap size: 1512
Minimum ever free heap size: 1512
---------------------恢复一部分后不再恢复------------
Free heap size: 2136
Minimum ever free heap size: 1512
```

```
Free heap size: 4392
Minimum ever free heap size: 3504
Free heap size: 3752
Minimum ever free heap size: 3504
Free heap size: 3736
Minimum ever free heap size: 3488
Free heap size: 3720
Minimum ever free heap size: 3456
Free heap size: 4344
Minimum ever free heap size: 3456
```

继续观察发现，从4392到4344，实际减少16*3，而其余的会恢复。可知是创建的任务句柄未被释放，而任务本身已经被空闲任务释放了。继续检查发现，未被释放的内存是在pxTuneSegmentTask内，当任务自行结束时pxTuneSegmentTaskHandler = NULL;导致内存泄漏。

由于任务句柄只是一个TCB的指针，用于外部使用TCB，当他杀时需要提供该指针，自杀时可使用pxCurrentTCB修改如下,故可在任务自杀前释放

```
xSemaphoreTake(xMutexTuneCounting,portMAX_DELAY);

vPortFree(pxTuneSegmentTaskHandler);	//释放当前句柄，但实际上TCB还在，vTaskDelete还是可以找到当前的tcb
pxTuneSegmentTaskHandler = NULL; //设置为被杀掉，其他任务无需再杀

xSemaphoreGive( xMutexTuneCounting );
vTaskDelete(NULL);
```

发现仍然存在泄漏，检查发现，pxTuneSegmentTask被TuneTask杀时，TuneTask并未释放指针。TuneTask每次创建任务时，都重新malloc了一个新指针。TuneTask修改如下

```
if(pxTuneSegmentTaskHandler!=NULL)	//如果播放任务还在运行
{
        tune_stop();
        vTaskDelete(*pxTuneSegmentTaskHandler);	//已经运行完，删除了就不删第二次	
        vPortFree(pxTuneSegmentTaskHandler);	//释放指针
        pxTuneSegmentTaskHandler = NULL;
}
```

另外，使用TaskHandle_t已经是TCB指针，使用TaskHandle_t * pxTuneSegmentTaskHandler进行操作显得很臃肿。直接使用TaskHandle_t作为任务是否结束的依据，为NULL则为结束，否则为运行。

```
void tune_refresh(void)
{
	while(1)
	{
		//获取队列
		xQueueReceive( g_xQueueTune,&mrequest,portMAX_DELAY );
		//高于当前优先级抢占播放
		if ( mrequest.priority >=  prio)
		{
			//结束当前任务
			if (xSemaphoreTake(xMutexTuneCounting, portMAX_DELAY) == pdTRUE)
			{	
				if(pxTuneSegmentTaskHandler!=NULL)	//如果播放任务还在运行
				{
						tune_stop();
						vTaskDelete(pxTuneSegmentTaskHandler);	//已经运行完，删除了就不删第二次	
						pxTuneSegmentTaskHandler = NULL;
				}
				
				xSemaphoreGive( xMutexTuneCounting );
				//开启新任务
				prio = mrequest.priority;
				tune = mrequest.tune;
				vol = mrequest.vol;
				if (xTaskCreate(pxTuneSegmentTask, "pxTuneSegmentTask", 128, NULL, osPriorityNormal, &pxTuneSegmentTaskHandler) != pdPASS)
				{
						printf("Task creation failed\r\n");
						pxTuneSegmentTaskHandler = NULL;
						printHeapStatus();
				}
			}
			else
			{
					printf("Failed to take mutex in tune_refresh\n");
			}
		}
	}
}

static void pxTuneSegmentTask(void * params)
{
	while(1)
	{
		
		switch(tone)
		{
			case TONE_STOP:
						tune_stop();
						xSemaphoreTake(xMutexTuneCounting,portMAX_DELAY);
						
						pxTuneSegmentTaskHandler = NULL; //设置为被杀掉，其他任务无需再杀
						
						xSemaphoreGive( xMutexTuneCounting );
						vTaskDelete(NULL);
						break;
		}
		vTaskDelay(buzzLen);
	}
}
```

#### 2.3.4 game1动态内存获取修改

将calloc、free改为pvPortMalloc、vPortFree

```
void game1_start()
{
	...
	blocks = pvPortMalloc(BLOCK_COUNT*sizeof(bool));
	memset(blocks, 0, BLOCK_COUNT*sizeof(bool));
	if (blocks == NULL)
	{
			printf("blocks Memory allocation failed\r\n");
			printHeapStatus();
			return;
	}
	...
}

static bool btnExit()
{
	vPortFree(blocks);
	...
	return true;
}
```

但是按键重复进入btnExit时，会重复vPortFree，导致出错。应当使button任务在运行当前btnFunc的时候阻塞，见2.1的2.1.5 防止button按键函数重复执行

### =>2.4.定时计时功能---hardfault

#### 2.4.1 原始功能

1. 在systick中断中维护另外一个变量systick_t，用于记录全局时间。还维护系统日期变量timeDate.time.secs，systick_t整除1000则++。
1. 底层driver_systick中定义了该全局变量，并提供获取时间的接口SysTickGetMs
1. 定时功能中存储当前时间、上次时间、定时器状态三个变量。系统按下stopbtn记录一个last_time，循环更新根据定时器状态为start才持续更新Timer。

#### 2.4.2 FreeRTOS编写思路

想在systick中维护另外一个全局时间变量，但是#define xPortSysTickHandler SysTick_Handler，xPortSysTickHandler 定义在port.c文件内，故不好修改。直接获取全局时间xTickCount。

外部调用TickType_t xTaskGetTickCountFromISR( void )获取全局时间，并fresh更新，使用delay阻塞。

启动定时更新需要外部信号，使用任务通知。

#### 2.4.3 编写过程与错误修改

将获取systick_t的程序全改为，调用xTaskGetTickCountFromISR( void )，获取全局时间

编写任务，并在任务循环中阻塞

在此基础上精进，使用阻塞启动，按键函数执行发送通知

任务通知可以通知具体的数字，用于控制启动与结束

肯定有一个任务用于循环更新时间，并且只有启动了定时器才会调用该任务

还必须有一个任务监听外部通知，有通知以后启动更新时间的任务

发现像StopWatchTask发送启动信号时，跳到Hard Fault，检查StopWatchTask创建成功并进入等待通知，比较原程序发现直接操作了任务的函数xTaskNotify( StopWatchTask, 1, eSetValueWithOverwrite );问题还是出在地址访问上

修改为操作TCB， xTaskNotify( StopWatchTaskHandler, 1, eSetValueWithOverwrite );解决问题，大致耗费1h

发现xTaskNotify卡在configASSERT( xTaskToNotify );因为xTaskToNotify为空，说明还未初始化，发现xTaskCreateFlag = xTaskCreate(StopWatchTask, "StopWatchTask", 128, NULL, osPriorityNormal, &StopWatchTaskHandler);忘记使用取址符号

发现StopWatchTask卡在了eTaskGetState(StopWatchCountingTaskHandler); 因为StopWatchCountingTaskHandler没有初始化是不可使用的，加入初始化为NULL，并判断第一次。

	TaskHandle_t StopWatchCountingTaskHandler = NULL;
	while(1)
	{
		xTaskNotifyWait( ULONG_MAX, ULONG_MAX, &pulNotificationValue,portMAX_DELAY );
		if (StopWatchCountingTaskHandler != NULL)
			state = eTaskGetState(StopWatchCountingTaskHandler); 

发现偶尔不能暂停，而且会运行到hard Fault，存在任务间变量的未互斥问题，button任务会修改state，timer，lastMS

增加信号量，可以解决部分hard Fault，但仍然存在不能暂停，而且会运行到hard Fault

	void StopWatchCountingTask(void *params)
	{
		while(1)
		{		
			xSemaphoreTake(xMutexStopWatch,portMAX_DELAY);
	        millis_t now = xTaskGetTickCount();
	        timer += now - lastMS;
	        lastMS = now;
	        if(timer > 359999999) // 99 hours, 59 mins, 59 secs, 999 ms
	            timer = 359999999;
	
	        xSemaphoreGive( xMutexStopWatch );
	
	        vTaskDelay(10); //一方面允许调度程序运行，一方面防止信号量抢占
		}
	}

怀疑是各全局变量并未做到互斥访问，尤其是指针，尤其是按键。

调试发现，当秒表任务进入阻塞，并没有将其恢复----暂未处理

调试发现，打开定时器以后，pxTuneSegmentTask多次处于blocked的时候被主任务删除，一次处于ready的时候被删除并死机

转到2.2音频功能的2.2.5

发现stopwatch任务是被实时创建的，而不是预先就运行的，把StopWatchTask分成几部分，分别放到按键函数内

出现一直未获取互斥量，因为初始化的时候判断互斥量==null才创建，但是上次退出时已经删除(释放)了该互斥量，但是指向了一个未定义地址，所以第二次未创建

出现hard faul，vTaskDelete( StopWatchCountingTaskHandler );因为animation_start(display_load, ANIM_MOVE_OFF);并只是设置了下一页面的初始化函数display_load，并未执行，所以如果在display_load执行之前因为抖动再次运行button就会再次删除

```
eTaskState taskstate = eInvalid;
taskstate = eTaskGetState(StopWatchCountingTaskHandler);
printf("stopwatch:%d",taskstate);

if (taskstate != eInvalid && taskstate != eDeleted)
    vTaskDelete( StopWatchCountingTaskHandler );
```

经过修改为上述代码，问题未解决，打印输出为stopwatch:1stopwatch:1，验证确实运行了两次，但是当任务刚被删除，其状态并未马上修改，所以该方案不行，尝试使用指针规避

```
if(StopWatchCountingTaskHandler_P != NULL)
    vTaskDelete( *StopWatchCountingTaskHandler_P );
StopWatchCountingTaskHandler_P = NULL;

vSemaphoreDelete(xMutexStopWatch);
xMutexStopWatch = NULL;
```

上述代码规避了重复删除任务的错误，但仍然存在重复删除队列！导致一直等待。

### 2.5 时间功能

#### 2.5.1 原始功能实现

1. 在systick中断中维护系统日期变量timeDate.time.secs，systick_t整除1000则++，并设置全局时间更新标志update为true。
1. 顶层time.c中定义了这俩全局变量
1. 定时功能循环中，根据标志位进行一次从秒到分时的变换。

#### 2.5.2 FreeRTOS编写思路

1. 在tim4的中断中维护系统日期变量timeDate.time.secs，systick_t整除1000则++，并发送任务通知给时间任务。
2. 在顶层time.c中定义了变量
3. 当接收到任务通知进行更新

#### 2.5.3 编写过程与错误修改

```
void HAL_TIM_PeriodElapsedCallback(TIM_HandleTypeDef *htim)
{
...
  if (htim->Instance == TIM4) 
	{
		tick = HAL_GetTick();
		if (tick %10 == 0)
		{
			vTaskNotifyGiveFromISR(TimeTaskHandler,NULL);
		}
	}
}
```

当任务还未创建时就在中断中发送通知，卡死在configASSERT( xTaskToNotify ); 因为xTaskToNotify 无效。

### 2.6 Game1

#### 2.6.1 功能实现

在退出games后，加载game1的参数设置（包括内存的malloc），按键和显示设置。

在退出函数里free对应内存。

#### 2.6.2 编写过程与错误修改

1. 程序退出时，出现屏幕显示异常但正常退出，当再次进入后，小球碰到blocks就导致hardfault。

经过查验是blocks已经经历了free过程，但是原显示程序仍然运行。也就是按键任务已经执行退出函数并开启动画，但在动画结束后才重设按键和显示函数，所以原退出函数可被重复调用。

方法是取消退出动画，防止退出函数重复运行。

详细说明如-》2.1.5 对抗/防止button按键函数重复执行---Game1退出内存重复free

2. 当不断玩游戏，慢慢地声音就无法播放了。

经查验是堆内存耗尽导致无法malloc新TaskHandler了，所以无法创建新任务。另外malloc的方法也会导致程序碎片化。

方法是取消TuneTask中的TaskHandler指针的动态，而使用全局变量。取消使用malloc，而是用pvPortMalloc等heap4提供的函数。

具体如-》2.3 TuneTask和DisplayTask---内存泄漏

### 2.7 Torch

#### 2.7.1 功能实现

退出m_main菜单后进入torch功能页面，设置了按键和显示函数

按键设置本地参数值，显示任务获取时间实时比较参数值，调整屏幕的亮和暗。

#### 2.7.2程序编写与错误修改

由于按键任务退出时取消屏幕的颜色反转，但显示任务可能还在原程序运行，又设置一次导致颜色再次反转。

1. 可以考虑关闭动画直接执行操作。按键任务退出队列阻塞后调用exit，设置画面不反转、更换显示函数，但显示任务可能继续执行原draw，还是会反转。

2. 若利用互斥量进行同步，当开始执行退出函数时获取互斥量，当完成修改时释放信号量，但只能保证运行中内容的操作完整不受影响，还是会出现按键任务设置好各种函数，但显示任务运行再执行原任务的可能。

   要保证按键任务进入exit函数设置draw和画面不反转前，draw任务已经运行到了将要退出的状态。

3. 若使用任务通知，display在draw设置了屏幕后发送通知给button，button在exit中等待此次通知并在获取通知后立马关闭调度、重设draw。但上次draw运行结束后发送的任务通知也对exit有效，所以不行。

4. 考虑使用两个任务通知，button调用exit后先通知display，display可以开始准备通知button，并在程序结束前通知button。

5. 使用高级任务通知，进入前可清除通知值。好像逻辑上没问题，但是画面依旧无法正确，可能时iic传输上有问题？

6. 继续使用两个任务通知，可以解决问题

   ```
   static bool btnExit()
   {	
   	xTaskNotify( DisplayTaskHandler, TorchExit, eSetValueWithOverwrite );
   	xTaskNotifyWait( ULONG_MAX,ULONG_MAX,&pulNotificationValue,portMAX_DELAY ); //进入、退出时清除标志
   	if(pulNotificationValue == TorchDrawOver)
   	{
   		vTaskSuspendAll( );
   	
   		display_load();
   		LCD_SetColorTurn(false);
   		
   		xTaskResumeAll( );
   		return true;
   	}
   	return true;
   }
   
   static display_t draw()
   {
     if (strobe)
     {
       millis_t now = millis();
       if (now - lastStrobe >= strobe)
       {
         lastStrobe = now;
         invert = !invert;
         LCD_SetColorTurn(invert);
   			
       if (xTaskNotifyWait( ULONG_MAX,ULONG_MAX,&pulNotificationValue,0 ) == pdPASS)
           xTaskNotify( ButtonTaskHandler, TorchDrawOver, eSetValueWithOverwrite );
       }
       return DISPLAY_BUSY;
     }
       LCD_SetColorTurn(true);
   	
       if (xTaskNotifyWait( ULONG_MAX,ULONG_MAX,&pulNotificationValue,0 ) == pdPASS)
           xTaskNotify( ButtonTaskHandler, TorchDrawOver, eSetValueWithOverwrite );
   	
     return DISPLAY_DONE;
   }
   
   ```

### 2.8 Button和Display的相互影响

#### 2.8.1 button和display都用到了loader---menu的分解

menu和各子页面的关系：作为中间统一抽象，将各子目录的按键、显示函数，提供给button,display任务

![image-20240903140427176](E:\0_wjx_file\embedded_ai\study_data\0 note\project\nwatch\image-20240903140427176.png)

menu与button,display的具体关系---以games为例

![ButtonTask_menu](E:\0_wjx_file\embedded_ai\study_data\0 note\project\nwatch\ButtonTask_menu.jpg)

button任务和display任务都调用了loader函数

1. button通过loader函数切换菜单，其中需要设置operation

operation.id = selected;
operation.op = OPERATION_ACTION;
operation.data = 动画控制信号;

然后调用setMenuOption执行操作。

2. display通过loader函数显示菜单，其中需要设置operation

operation.id = selected
operation.op=OPERATION_DRAWICON或
operation.data=x坐标（icon信息在setMenuOption传入）

然后调用setMenuOption执行操作。

3. 冲突分析

当按键设置完operation后，被display修改，再切换回来时，执行了display，此次按键无效

当display设置完operation后，被按键修改，再次切换回后，执行按键页面切换，此次display无效（看似未影响功能，实际顺序已乱）

需要实现设置完operation后，执行完后续的操作，才可以再设置operation，也就是loader函数完整执行

4. 解决方法

两个任务调用同一个函数，在函数的开头获取，在函数结尾释放，就可以实现函数在两个任务中分开执行

#### 2.8.2 按键操作和显示draw执行的先后问题

1. 问题描述

Torch中，由于按键任务退出时取消屏幕的颜色反转，但显示任务可能还在原程序运行，又设置一次导致颜色再次反转。

2. 初步解决思路

   1. 可以考虑关闭动画直接执行操作。但仍然存在按键任务设置好各种函数，但显示任务运行再执行原任务的可能。

   2. 若利用互斥量进行同步，当开始执行退出函数时获取互斥量，当完成修改时释放信号量，但只能保证运行中内容的操作完整不受影响，还是会出现按键任务设置好各种函数，但显示任务运行再执行原任务的可能。要保证按键任务进入退出函数设置前，draw任务已经运行到了将要退出的状态。

3. 总结与思考

   1. 什么导致了该现象，直到本质

      torch退出后颜色不对

      因为任务并行，按键任务设置了又被显示任务修改，且显示任务长所以可能性大

      需要按键设置了以后，显示任务不能再设置（显示任务中的相关设置不再设置）

   2. 联想game1

      game1使用free后，退出动画结束才重设函数，所以draw被调用，也是类似的先后问题

   3. 思考

      我们需要实现先按键，后调用draw显示的功能

      我们任何时候，根据draw指针循环调用屏幕刷新，draw肯定存在按键结束后仍在运行时的情况。

      可以加入按键的调用，与draw的刷新互斥，按键退出程序和draw程序不会交叉运行。但需要在按键调用中更改draw才能解决torch问题。要在按键调用中不用动画直接设置函数，才能解决game1的问题。

   4. 解决思路

      各任务的退出都存在先后问题，需要先等原draw运行结束，再按键设置新draw。可像Torch加入两个任务通知，也可以用两个全局变量。

      但现在的架构，导致我们必须一个个功能修改exit按键和draw函数。

4. 解决---其他任务也可参考

   1. torch的按键退出与display的播放之间进行3次握手

      ```
      //torch退出按键函数
      extern TaskHandle_t DisplayTaskHandler ;
      xTaskNotify( DisplayTaskHandler, DISPLAY_DRAW_STOP_REFRESH_APPLICATION, eSetValueWithOverwrite );
      xTaskNotifyWait( ULONG_MAX,ULONG_MAX,&pulNotificationValue,portMAX_DELAY ); //进入、退出时清除标志
      if(pulNotificationValue == DISPLAY_DRAW_OVER)
      {
          display_load();
          LCD_SetColorTurn(false);
      
          xTaskNotify( DisplayTaskHandler, DISPLAY_DRAW_START_REFRESH_APPLICATION, eSetValueWithOverwrite );
          return true;
      }
      
      //display刷新函数
      extern TaskHandle_t ButtonTaskHandler ;
      if (xTaskNotifyWait( ULONG_MAX,ULONG_MAX,&pulNotificationValue,0 ) == pdPASS)
      {
          if(pulNotificationValue == DISPLAY_DRAW_STOP_REFRESH_APPLICATION)
          {
              xTaskNotify( ButtonTaskHandler, DISPLAY_DRAW_OVER, eSetValueWithOverwrite );
              xTaskNotifyWait( ULONG_MAX,ULONG_MAX,&pulNotificationValue, portMAX_DELAY );
              if (pulNotificationValue == DISPLAY_DRAW_START_REFRESH_APPLICATION)
              {
                  printf("display draw changed!");
              }
          }
      }
      ```

# 五 版本记录

my_nwatch_freertos_oled_button

my_nwatch_freertos_oled_button_buz

my_nwatch_freertos_oled_buz_butos

my_nwatch_freertos_oled_butos_bizos

my_nwatch_freertos_oledclean_butos_bizos_swos

my_nwatch_freertos_oledclean_butos_bizos_swos_timos

my_nwatch_freertos_oledclean_butos_bizos_swos_timos_debugmsg(biz内存泄漏)

my_nwatch_freertos_oledclean_butos_bizos_swos_timos_debugmsgok ：biz无内存泄漏

my_nwatch_freertos_1 ：将tune、stopwatch的子任务创建改为直接使用句柄，而不用句柄指针导致泄漏

my_nwatch_freertos_2 ：将游戏设置为freertos内存动态获取与释放。设置直接退出无动画，防止释放空内存报错

my_nwatch_freertos_3 ：button内加入互斥量，并通过任务名字解决死锁问题；	将button和display，共同调用的loader函数，加入互斥

my_nwatch_freertos_4 ：torch使用两个任务通知，保证任务执行顺序，但是将display中的互斥量取消了，否则会死锁

my_nwatch_freertos_V2_1：使用状态机作为中控，button仅进行按键获取，不进行相关函数调用；display根据状态进行函数调用。使用两个全局变量保证任务切换时display完成一个周期，lcd不受影响。

my_nwatch_freertos_5 ：将display中的互斥量加回，在display_refresh与torch按键退出间，增加三次信号通讯，保证按键结束后调用新的draw。

------

