# 一 文件结构

```文件结构
NWatch/
├── APP/
│   ├── main.c			主循环
│   ├── appconfig.c		配置编译选项
│   ├── buttons.c		配置按键功能
│   ├── time.c			更新时间
│   ├── display.c		配置与更新显示
│   │   ├── animation.c		计算偏移
│   │   ├── draw.c			写入buff
│   │   ├── resources.c		图形资源
│   │   └── ui.c			调用资源写入
│   ├── normal.c		封面
│   ├── watchface.c		封面2
│   └── menu.c			菜单
│       ├── m_main.c		主页面
│       ├── torch.c			灯光
│       ├── alarms.c		闹钟页面
│       │   └── alarm.c
│       ├── tunemaker.c		播放歌曲
│       │   ├── tune.c			据歌曲数组、调声音驱动
│       │   └── tunes.c			歌曲数组资源
│       ├── games.c
│       │   ├── game1.c
│       │   ├── game2.c
│       │   └── game3.c
│       ├── settings.c
│       │   ├── timedate.c
│       │   ├── sleep.c
│       │   ├── sound.c
│       │   ├── m_display.c
│       │   ├── diag.c
│       │   └── reset.c
│       └── stopwatch.c
├── APP Driver/
│   ├── driver_color_led.c
│   ├── driver_dht11.c
│   ├── driver_ir_receiver.c
│   ├── driver_irq.c
│   ├── driver_lcd.c
│   ├── driver_passive_buzzer.c
│   ├── driver_spiflash_w25q64.c
│   ├── driver_systick.c
│   ├── driver_timer.c
│   └── driver_uart.c
└── HAL Driver/
```

# 二 裸机运行原理

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

## 6. 程序运行原理

### 6.1 Button

```
typedef struct {
  millis_t pressedTime;		// 按键时间，使用systick心跳
  byte counter;				// 防反跳/抖动计数器，原代码为EXTI_Trigger_Falling出发故存在抖动多次下降
  bool processed;			// 是否被处理标志
  bool funcDone;			// 是否被执行完成
  button_f onPress;			// 下一要执行的函数	typedef bool (*button_f)(void);	typedef定义函数指针
  const tune_t* tune;		// 音调指针，指向音调数据
} s_button;
```

```
//初始化硬件设备
buttons_startup()；
	KEY_Init();
	IRReceiver_Init();3
//设置按键音调
buttons[BTN_1].tune = tuneBtn1;
buttons[BTN_2].tune = tuneBtn2;
buttons[BTN_3].tune = tuneBtn3;
//设置执行函数
buttons_setFuncs(NULL, menu_select, NULL);
    buttons[BTN_1].onPress = btn1;
    buttons[BTN_2].onPress = btn2;
    buttons[BTN_3].onPress = btn3;

//更新
(millis_t)(now - lastUpdate) >= 12	//满足更新周期
processButtons()；
	IRReceiver_Read(&dev, &data)；	//获取IR BUF值，确认按键
	LOOPR(BTN_COUNT, i) = for(byte var=count;var--;)
		processButton(&buttons[i], isPressed[i]);	//执行有效按键
            button->counter <<= 1;	//counter左移一位
            if (isPressed)
                button->counter |= 1;//counter低位=1
                if (bitCount(button->counter) >= BTN_IS_PRESSED)	//开源代码，防止反跳计数 >= 4消抖
                	/* 核心 */
                    if (!button->processed)				//若按键没被处理
                        button->pressedTime = millis();		//记录时间
                        button->processed = true;			//标为按键处理完成
                    if (!button->funcDone && button->onPress != NULL && button->onPress())	//若函数未被执行，执行函数
                        button->funcDone = true;			//标记为函数执行完成
                        tune_play(button->tune, VOL_UI, PRIO_UI);//触发播放音乐       
            else
            	if (bitCount(button->counter) <= BTN_NOT_PRESSED)	//防止反跳，反跳计数<=4上次press无效
                      button->processed = false;	//消除按键被处理标记
                      button->funcDone = false;		//消除函数被执行标记
```

```
总结：
每个button结构体有tune音调数据指针、函数指针，每次循环间隔达到后获取输入并执行，完成后触发音乐播放
其中counter可对下降沿触发消抖（波动计数次数达到4以上才执行函数）
其中counter还可辅助清除标志（计数随时间减少小于4清除执行标志）
其中processed和funcDone可防止执行完后再次执行
```

### 6.2 display

```
typedef struct{
	bool active;					//动画是否启用
	byte offsetY;					//动画上下偏移量
	void (*animOnComplete)(void);	//动画完成回调函数
	bool goingOffScreen;			//动画进出类型
}anim_s;
```

```
//初始化OLED硬件
OLED_Init();
//设置显示配置
display_set(watchface_normal);
	func = faceFunc;	//static display_f func; 
	
//执行显示配置，启动动画
display_load();
	func();	 //如watchface_normal
		display_setDrawFunc(draw);					//设置画面更新函数
		buttons_setFuncs(up, menu_select, down);	//设置按键执行函数
		animation_start(NULL, ANIM_MOVE_ON);		//触发动画
            animationStatus.active = true;						//动画启动标志
            animationStatus.offsetY = goingOffScreen ? 0 : 192;	//true则选0，开始递增到FRAME_HEIGHT=64，效果为满屏显示从上向下挤出
                                                                 //否则192，开始递增到255（192+65=1，255+64=64），效果为空屏从上向下挤出来
            animationStatus.animOnComplete = animOnComplete;	//执行完回调函数
            animationStatus.goingOffScreen = goingOffScreen;	//动画进出类型

//更新画面
display_update();
    if( (millis8_t)(now - lastDraw) < fpsMs ) return;	//满足更新周期
    animation_update();
        if(animationStatus.active)						//更新动画偏移
            if(animationStatus.goingOffScreen)	//退出计算
            else								//进入计算
            animationStatus.offsetY = offsetY;	//修改偏移
            if(!animationStatus.active && animationStatus.animOnComplete != NULL)	//完成偏移计算
                animationStatus.animOnComplete();		//调用动画回调函数
                animationStatus.animOnComplete = NULL;
    busy = drawFunc();	//如normal的draw
    	drawDate();
            strcpy(day, days[timeDate.date.day]);	//获取day的字符
            strcpy(month, months[timeDate.date.month]);//获取month的字符
            sprintf_P(buff, PSTR(DATE_FORMAT), day, timeDate.date.date, month, timeDate.date.year);	//sprintf组织str
            draw_string(buff,false,12,0);			
    	ticker();
    		if(timeDate.time.secs != secs)	//秒发生变化
                yPos = 0;											//动画y偏移
                moving = true;										//个位秒需要时间动画
                moving2[0] = div10(timeDate.time.hour) != div10(hour2);//十位时前后不等，需要时间动画
                moving2[1] = mod10(timeDate.time.hour) != mod10(hour2);//个位时前后不等，需要时间动画
                moving2[2] = div10(timeDate.time.mins) != div10(mins);//十位分前后不等，需要时间动画
                moving2[3] = mod10(timeDate.time.mins) != mod10(mins);//个位分前后不等，需要时间动画
                moving2[4] = div10(timeDate.time.secs) != div10(secs);//十位秒前后不等，需要时间动画
                hour2 = timeDate.time.hour;		//更新H值
                mins = timeDate.time.mins;		//更新M值
                secs = timeDate.time.secs;		//更新S值
            if(moving)									//时间动画偏移计算
                yPos = xx		//时、分为大字，统一偏移				
                yPos_secs = xx	//秒为小字，统一偏移
                if(yPos_secs > FONT_SMALL2_HEIGHT + TICKER_GAP && yPos > MIDFONT_HEIGHT + TICKER_GAP)//判断动画结束
                    yPos = 0;
                    yPos_secs = 0;
                    moving = false;
                    memset(moving2, false, sizeof(moving2));
            data.x = 104;								//设置显示参数
            data.y = 28;
            data.bitmap = (const byte*)&small2Font;
            data.w = FONT_SMALL2_WIDTH;
            data.h = FONT_SMALL2_HEIGHT;
            data.offsetY = yPos_secs;
            data.val = div10(timeDate.time.secs);
            data.maxVal = 5;
            data.moving = moving2[4];
            drawTickerNum(&data);
            	draw_bitmap(x, y, bitmap, data->w, data->h, NOINVERT, yPos - data->h - TICKER_GAP);	//当前需要显示的值
                draw_bitmap(x, y, &data->bitmap[prev * arraySize], data->w, data->h, NOINVERT, yPos);//上一次要退出的值
    	drawBattery();
    		draw_bitmap(0, FRAME_HEIGHT - 8, battIcon, 16, 8, NOINVERT, 0);	
    draw_end();
        oled_flush();		//刷新页面

//特别说明draw_bitmap的实现
draw_bitmap
	yy += animation_offsetY();	//获取了动画的整体偏移
```

```
总结：
页面切换动画需要触发，每个循环进行一次偏移量计算，当前循环的draw_bitmap使用该偏移进行绘制，直至动画结束
动画管理与画面分离，画面可按照正常逻辑绘制
```

### 6.3 tune

```
static byte idx;			// Position in tune
static const tune_t* tune;	// The tune
static vol_t vol;			// Volume
static tonePrio_t prio;		// Priority
```

```
//初始化蜂鸣器
PassiveBuzzer_Init();

//触发播放，按键为例
tune_play(button->tune, VOL_UI, PRIO_UI);
    if(_prio < prio) return;	//当前优先级与请求优先级比较
	prio	= _prio;			//输入音频优先级
	tune	= _tune;			//输入音频音调
	vol		= _vol;				//输入音频音量
	idx		= 0;				//输入音频数据序号
	next();			//触发一次播放

//音调更新
tune_update();
	if(buzzLen && (millis8_t)(millis() - startTime) >= buzzLen)	//达到循环周期，且达到播放时间
		stop();			//停止上次播放
            buzzLen = 0;
            prio = PRIO_MIN;
		next();			//触发下一次播放

//单独说明next
next();
    uint data = tune_read(tune[idx++]);	//获取音频数据
    buzzLen = data;						//解析播放时间
    tone_t tone = (tone_t)(data>>8);	//解析音调
    //根据音调进行播放
    if(tone == TONE_REPEAT)			
        idx = 0;							//重复开头，只有数据序号需要修改
        next();
    else if(tone == TONE_STOP)
        tune_stop(PRIO_MIN);				//停止播放，相关影响数据重置
            PassiveBuzzer_Control(0);
            stop();
                buzzLen = 0;
                prio = PRIO_MIN;
    else if(tone == TONE_PAUSE)
        PassiveBuzzer_Control(0);			//空音调
    else
        PassiveBuzzer_Set_Freq_Duty(ocr, 50);//默认按照音调、音量设置蜂鸣器频率，占空比一定=0.5
```

```
总结：
音乐播放需要触发设置声音数据，并循环中不断播放下一声音数据
```

### 6.4 time

```
定时器中断自动累加计数，修改全局时间变量
```

### 6.5 menu

```
//全局：记录当前菜单信息
typedef struct{
	byte selected;		//menu选中项的索引
	byte scroll;
	byte optionCount;	//menu中可选项数目
	bool isOpen;		//打开标志，menu是否被打开
	const char* title;	//menu的名称
	menu_type_t menuType;//menu的类型
	
	menuFuncs_t func;	//menu的控制函数
	
	menu_f prevMenu;	//上一菜单的启动函数
}menu_s;

//局部：记录当前页的上一菜单
typedef struct{
	byte lastSelected;	//上一菜单选中的选项索引
	menu_f last;		//上一次菜单的启动函数
}prev_menu_s;

typedef struct
{
	byte data;		//数据信息
	operation_t op;	//操作类型，包括DRAWICON、DRAWNAME_ICON、DRAWNAME_STR、ACTION运行
	byte id;		//目标索引
}operation_s;
```

```
menu_select();
	if(!menuData.isOpen) mMainOpen();	//第一次打开menu时
		beginAnimation(mOpen);					 //触发表盘退出动画
		mopen();								//调用m_main页面的启动函数mOpen
            display_setDrawFunc(menu_draw);				     			  //设置display函数，为统一menu操作函数
            buttons_setFuncs(menu_up, menu_select, menu_down);			   //设置button函数，为统一menu操作函数
            setMenuInfo(OPTION_COUNT, MENU_TYPE_ICON, PSTR(STR_MAINMENU));	//设置menudata中基础信息
            	clear();				//清除menu.Data.func
                menuData.scroll = 0;
                menuData.selected = 0;
                menuData.optionCount = optionCount + 1;
                menuData.menuType = menuType;
                menuData.title = title;
            setMenuFuncs(MENUFUNC_NEXT, mSelect, MENUFUNC_PREV, itemLoader);//设置menudata中的func，为m_main页的相关函数
            	menuData.func.btn1 = btn1Func;
                menuData.func.btn2 = btn2Func;
                menuData.func.btn3 = btn3Func;
                menuData.func.loader = loader;
            setPrevMenuOpen(&prevMenuData, mOpen);						  //设置menudata中prevMenu，根据当前页
            beginAnimation2(NULL);										 //触发启动出现动画		
    menuData.func.btn2();			//menu内选中选项时
```

1. display menu

```
menu_draw()
	if(menuData.menuType == MENU_TYPE_STR)
		menu_drawStr();
	else
		menu_drawIcon();
		
menu_drawIcon();
    drawTitle();
    draw_bitmap(46, 14, selectbar_top, 36, 8, NOINVERT, 0);
    draw_bitmap(46, 42, selectbar_bottom, 36, 8, NOINVERT, 0);
    static int animX = 64;					//第一个图标中心x动画位置	
    int x = 64 - (48 * menuData.selected);	//第一个图标中心x目标位置
    if(x > animX)
        speed = ((x - animX) / 4) + 1;			//计算移动量
        if(speed > 16) speed = 16;
        animX += speed;							//修改移动量
        if(x <= animX) animX = x;				 //动画完成位置
    else if(x < animX)
        speed = ((animX - x) / 4) + 1;
        if(speed > 16) speed = 16;
        animX -= speed;
        if(x >= animX) animX = x;
    x = animX - 16;						   //第一个图标左上角x动画位置
    LOOP(menuData.optionCount, i)			//依次计算偏移，在屏幕内的画出
        if(x < FRAME_WIDTH && x > -32)		//图标左上位置满足屏幕
            loader(OPERATION_DRAWICON, i, x);		//设置operation，调用m_main页面的itemloader函数
                setMenuOption(num, buff, icon, actionFunc);
                    switch(operation.op)
                    case OPERATION_DRAWICON:
                        byte y = 28; // comment this out for magic
                        draw_bitmap(operation.data, y + 4 - 16, icon != NULL ? icon : menu_default, 32, 32, NOINVERT, 0);
                    case OPERATION_DRAWNAME_ICON:
                    case OPERATION_DRAWNAME_STR:
                    case OPERATION_ACTION:		beginAnimation(actionFunc)
        x += 48;							//下一个图标左上位置
```

2. button menu

```

```

```
总结：
menu抽象了菜单所需要的结构与工具，多个菜单可依据menu构造，选中一个作为当前菜单
menu有统一的画面display函数可绘制str与icon，当前菜单需要设置全局menuData的selected、menuType、optionCount作为空间布局提示，并设置menuData的func loader作为画面内容提示
```

# 三 FreeRTOS改进思路

```
分析可知交互关系:
button: display tune
display：button time
tune：button
time：display

menu作为全局数据
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

#### 2.1.4 button队列集控制---rotary + ir

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

#### 2.1.6 Button互斥与死锁---任务内进行互斥

1.  死锁

Mutex被Func占用；Mutex被SetFunc申请

<img src="E:\0_wjx_file\embedded_ai\study_data\0 note\nwatch\249245B3DE2B98588C85F3DFBB5583C4.png" alt="249245B3DE2B98588C85F3DFBB5583C4" style="zoom: 25%;" />

“T”被D占用；“U”被D申请；"U"被C占用；“T”被C申请

<img src="E:\0_wjx_file\embedded_ai\study_data\0 note\nwatch\image-20240920105805308.png" alt="image-20240920105805308" style="zoom:50%;" />

2. 死锁原因

任务申请的资源被另外的任务上锁，而另外的任务申请的资源也被当前任务上锁

2. 死锁解决---不在任务内进行互斥

按键func的使用 vs setFuncs设置按键func。

大部分情况是button任务执行func，并直接调用setFuncs重新设置func。

小部分情况是button任务执行func，设置display动画以及动画结束函数，display任务内调用任务结束函数，其中用setFuncs重新设置func。如：animation_start(display_load, ANIM_MOVE_OFF);    其中display_load内调用了display_set设置函数，其中有setFuncs

即实现button任务 vs display任务内，func的设置与运行互斥，而且要避免button任务内进行互斥。

实现两套buttons_setFuncs函数供外部使用，对于确定是被button任务调用的无需加入互斥，对于确定是display等其他任务调用的，加入互斥。

但是上述方法过于繁杂，可以直接在互斥之前，获取当前任务名字进行判断，再进行互斥。

### 2.2 音频功能

#### 2.2.1 原始功能说明（底层逻辑服务于底层循环，但是底层循环更新不合理）

1. 初始化
2. tune_play
   1. 判断输入声音优先级是否>本地全局优先级(当前)
   2. 设置优先级，资源，声音，索引等
   3. 调用一次next

3. next中
   1. 根据声音序列提取音调
   2. 根据*音调判断是否重复（因为需要和取音调放一起）*
      1. 是则重设音调索引
      2. 否则转到底层逻辑
4. **底层逻辑**开始
   1. 判断输入声音优先级是否>本地全局优先级(当前)
   2. 判断是否到了*结束音调*
      1. 是则，直接返回(时间==0，无法调用下次next)
      2. 否则，设置参数，*修改音调*
         1. 设置歌曲优先级，设置next为下次循环调用函数，***设置长度(长度>0才可调用next)***
         2. 判断*是否暂停*
         3. 设置*其他音调*
5. **主循环中调用底层**判断***是否到时间***，到了则调用下次next

#### 2.2.2 原始功能修改（将音乐逻辑放到应用层）

1. 初始化
2. tune_play
   1. 判断输入声音优先级是否>本地全局优先级(当前)
   2. 设置优先级，资源，声音，索引等
   3. 调用一次next
3. next中（**调用底层**）
   1. 根据声音序列提取音调，***记录开始时间，长度***
   2. *判断是否重复*，是则重设音调索引
   3. 判断是否到了*结束音调*
   4. 判断是否*到了暂停*
   5. 判断是否设置*其他音调*
4. ***主循环中更新判断是否到时间***，到了则调用下次next



#### 2.2.3 FreeRTOS编写思路

思路一：一般一个音调要100/50ms，设置10ms的周期休眠判断并调用next√

思路二：将tune_play作为一个设置内容，next作为任务，结构更好看

​		tune_play中优先级判断+设置资源声音优先级等

​		next中周期根据资源，音量，索引，进行判断与动作

思路二+1：将tune_play作为一个设置内容，next作为任务

​			next大循环	阻塞等待音乐启动 优先级队列

​						    启动小循环	延迟等待下次更新

​										 完成后退出播放

#### 2.2.4 编写过程与错误修改

1. 进行oled+ir+buzzer程序移植到rtos程序后，发现蜂鸣器声音连续但很小。

​		首先调试发现程序运行到了按键调用程序，执行到音调播放程序，成功运行PassiveBuzzer_Set_Freq_Duty设置频率与周期，但是没有效果

​		怀疑rtos的任务调度导致播放不正常（tim受影响），故去除默认任务、设置音乐播放任务最高优先级/只进行音乐播放任务等，使蜂鸣器不受系统影响，但无效

​		对比尝试调用原程序运行，与移植后程序的设置频率和周期一致，并调用了PassiveBuzzer_Set_Freq_Duty设置频率与周期，有效果

​		怀疑底层硬件导致，检查硬件mx设置，时钟源使用内部时钟 ！= 旧程序的disable，修改完成后音量增加但音调仍然不对

​		尝试直接调用底层设置音调程序PassiveBuzzer_Set_Freq_Duty发现确实音调不对

​		尝试使用提供PassiveBuzzer_Test程序发现音调一致

​		回头检查PassiveBuzzer_Set_Freq_Duty是否存在错误，果然原程序为4000000而新程序为1000000

​		总结例子1：要先调试找问题，发现问题关键程序执行了到音调播放程序但无效果，思考是否操作系统影响，思考底层频率和周期是否正确。

​					要认识底层Hard TIM 运行是否会受到rtos影响，硬件定时器配置并启动后io输出不受任务调度影响，但产生的中断可能被任务屏蔽，未屏蔽则优先级最高也不受影响。（该案例可以排除检查系统）

​					要保证移植的前后，底层配置一致

2. 配置freertos在idle任务中用钩子任务进行cpu占用输出

​		首先mx设置空闲任务的钩子函数可用宏

​		编译出现xxx.axf: Undefined symbol vTaskGetRunTimeStats (referred from freertos.o).

​		然后mx设置运行时间+任务状态可用宏

​		发现打印无输出，检查打印无问题

​		然后对比原程序，需要提供两个函数 weak void configureTimerForRunTimeStats(void) 和 weak unsigned long getRunTimeCounterValue(void)

​		设置了getRunTimeCounterValue(void)获取tim4的精确计时后，可以记录cpu时间

​		总结例子2：编译器链接时，无法找到vTaskGetRunTimeStats 函数的链接编译结果，即该函数未被编译，一般为没写 或者 #define宏注释掉了

3. 通过任务创建与删除进行音乐播放

``` 
void tune_play(const tune_t* _tune, vol_t _vol, tonePrio_t _prio)
{
	...	
	xQueueSend(g_xQueueTune,&mrequest,0);
}
void tune_refresh(void)
{
	...
	while(1)
	{
		//获取队列
		xQueueReceive( g_xQueueTune,&mrequest,portMAX_DELAY );
		//高于当前优先级抢占播放
		if ( mrequest.priority >=  prio)
		{
			//结束当前任务
			if (pxTuneSegmentTaskHandler != NULL)
			{
				vTaskDelete(pxTuneSegmentTaskHandler);
				pxTuneSegmentTaskHandler = NULL;
				stop();
			}
			//开启新任务
			...
			xTaskCreate(pxTuneSegmentTask, "pxTuneSegmentTask", 128, NULL, osPriorityNormal, &pxTuneSegmentTaskHandler);
		}
	}
}

void pxTuneSegmentTask(void * params)
{
	while(1)
	{
		data = tune_read(tune[idx++]);
		buzzLen = data;
		tone = (tone_t)(data>>8);
		switch(tone)
		{
			case TONE_REPEAT:
						idx = 0;
						continue;break;
			case TONE_STOP:
						PassiveBuzzer_Control(0);
						vTaskDelete(NULL);
------------------------return;--如不写，会调用vTaskDelay休眠其他任务---
						break;
			case TONE_PAUSE:
						PassiveBuzzer_Control(0);break;
			default:
						PassiveBuzzer_Control(0);
						...
						PassiveBuzzer_Set_Freq_Duty(ocr, 50);
						break;
		}
----vTaskDelay(buzzLen);-------------------------------------
	}
}
```

​	音乐播放任务要创建并保持，只运行一个音乐播放子任务，若新tune优先级高于当前则要删除上一个子任务，再创建一个新子任务。一个任务被删除后再使用该任务的TCB句柄重新创建任务（后被证明是错误的，TCB删除后并未消失，再removelist内，需要重新窗前，idle任务会清除尸体）。

​	首先调试发现成功接收到按键，并通过音乐播放阻塞读取，但并未执行到创建的新任务，发现进入创建任务后卡死。很疑惑，使用chatgpt检查程序，发现创建任务使用TCBhandle时，需要传入指针&pxTuneSegmentTaskHandler，缺少取址符号

​	进入pxTuneSegmentTask任务后，任务完成一次播放后tune_refresh，pxTuneSegmentTask都无法进入。再次调试发现pxTuneSegmentTask任务执行到了vTaskDelete，程序并未结束并继续往后执行到vTaskDelay。对比一般的任务创建删除工程，发现任务删除一般在程序默认，故加上return，发现可以正常播放，但出现声音紊乱

``` 
void tune_play(const tune_t* _tune, vol_t _vol, tonePrio_t _prio)
{
	...	
	xQueueSend(g_xQueueTune,&mrequest,0);
}

void tune_refresh(void)
{
	...
	while(1)
	{
		//获取队列
		xQueueReceive( g_xQueueTune,&mrequest,portMAX_DELAY );
		//高于当前优先级抢占播放
		if ( mrequest.priority >=  prio)
		{
			//结束当前任务
			if(pxTuneSegmentTaskHandler!=NULL)	//保证第一次执行不错
			{
				state = eTaskGetState(*pxTuneSegmentTaskHandler); //如果为pxTuneSegmentTaskHandler = null则，其取出内容的state错误
				if (state != eDeleted && state != eInvalid )
				{
					vTaskDelete(*pxTuneSegmentTaskHandler);	//已经运行完，删除了就不删第二次
					stop();
				}
			}
			//开启新任务
			...
			pxTuneSegmentTaskHandler = malloc(sizeof(TaskHandle_t));
			xTaskCreate(pxTuneSegmentTask, "pxTuneSegmentTask", 128, NULL, osPriorityNormal, pxTuneSegmentTaskHandler);
		}
	}
}

static void pxTuneSegmentTask(void * params)
{
	...
	while(1)
	{
		data = tune_read(tune[idx++]);
		buzzLen = data;
		tone = (tone_t)(data>>8);
		
		switch(tone)
		{
			case TONE_REPEAT:
						idx = 0;
						continue;break;
			case TONE_STOP:
						stop();	-----------------------------暂停+idx清零
						vTaskDelete(NULL);
						break;
			case TONE_PAUSE:
						PassiveBuzzer_Control(0);break;
			default:
						PassiveBuzzer_Control(0);
						...
						PassiveBuzzer_Set_Freq_Duty(ocr, 50);
						break;
		}
		vTaskDelay(buzzLen);
	}
}
```

​	一个任务被删除（即任务的控制块TCB被放入了FreeRTOS的删除列表中，通常称为“remove list”），你不能再使用该任务的TCB句柄重新创建任务。可用FreeRTOS提供的函数`eTaskGetState`，可以用来获取特定任务的状态。该函数返回一个`eTaskState`枚举值，表示任务的状态。使用指针(而不是一个TCB结构体对象)指向一个播放子任务进行操作。

​	主任务判断上一任务状态不是 eDeleted或者eInvalid ，使用了if  (state != eDeleted || state != eInvalid )。与口头语言理解错误，我们说不是 eDeleted或者eInvalid ，也就是都不，要用if  (state != eDeleted && state != eInvalid )

​	声音紊乱，尝试增大优先级，并使用HAD_delay，紊乱存在，还可能卡死。考虑全局参数音频地址是否传递正确，发现全局位置索引idx未清零导致访问错误地址tune[idx]。

​	总结：逻辑判断需要精进。对音乐播放子任务的规定不明确，不够清楚它要完成什么任务/任务怎么实现，所以导致idx未清零问题

#### 2.2.5 音频播放任务自杀 VS 被音频设置任务他杀

调试发现，打开定时器以后，pxTuneSegmentTask多次处于blocked的时候被主任务删除，一次处于ready的时候被删除并死机

可能state = eTaskGetState(*pxTuneSegmentTaskHandler)返回未ready后，任务调度器转到对应任务运行，而且在外部删除了该任务，所以再删除就错误了

改进，通过增加任务TCB使用的互斥量。！！！！！！很关键，状态量检测必须互斥，但是思考一下，好像仅仅是互斥也很模糊，如果是禁止调度会更安全。

``` 
//更新音乐任务
xSemaphoreTake(xMutexTuneCounting,portMAX_DELAY);//有时候会被检测到未删除任务，但是到vTaskDelete又出错,防止中间任务切换

if(pxTuneSegmentTaskHandler!=NULL)	//保证第一次执行不错
{
    state = eTaskGetState(*pxTuneSegmentTaskHandler); //如果为pxTuneSegmentTaskHandler = null则，其取出内容的state错误
    if (state != eDeleted && state != eInvalid )
    {
        printf("state:%d\r\n",state);
        vTaskDelete(*pxTuneSegmentTaskHandler);	//已经运行完，删除了就不删第二次	
        stop();
    }
}

xSemaphoreGive( xMutexTuneCounting );
------------------------------------------------------------------
//音乐播放任务
xSemaphoreTake(xMutexTuneCounting,portMAX_DELAY);

vTaskDelete(NULL);

xSemaphoreGive( xMutexTuneCounting ); //不会再执行了
```

如果先进入了播放任务的删除，无法运行到释放信号量，更新任务就不会判断错误，所以只能运行一次，更新任务会一直阻塞

只能使用禁止调度，当进入更新任务时，禁止调度到播放任务，可以正确检测播放任务的状态，删除播放任务即可。

```
vTaskSuspendAll(  );

if(pxTuneSegmentTaskHandler!=NULL)	//保证第一次执行不错
{
    state = eTaskGetState(*pxTuneSegmentTaskHandler); //如果为pxTuneSegmentTaskHandler = null则，其取出内容的state错误
    if (state != eDeleted && state != eInvalid )
    {
        printf("state:%d\r\n",state);
        vTaskDelete(*pxTuneSegmentTaskHandler);	//已经运行完，删除了就不删第二次	
        stop();
    }
}

xTaskResumeAll( );
```

仍然不行，会出现state==eready的现象，vTaskSuspendAll只是暂停调度器并不更改各任务状态。

在删除任务后立即获取状态，可能会获取到无效或过期的状态。因此，通常删除任务后不再检查其状态，防止使用无效的任务句柄。如果确实需要确保任务已经终止，建议通过其他机制（如任务句柄置为 `NULL` 或检查其他标志位）来确认。

思考得，主要出错是任务自杀后在他杀，而不必担心任务他杀后再自杀，所以这是一个不那么严格的互斥问题。

一般的互斥问题是，一个变量我访问了，你就不能访问，防止一起访问出现读取和设置错误。

这里使用互斥量，是为了实现设置任务、播放任务，对播放任务的指针，互斥读取与设置

他杀的时候互斥地检查指针及指向的状态并判断是否删除，此时程序不允许修改指针状态/自杀

自杀的时候互斥地设置指针为NULL并再后续自杀删除，此时程序不允许其他任务读取指针并尝试删除

``` 
//结束当前任务
xSemaphoreTake(xMutexTuneCounting,portMAX_DELAY);//有时候会被检测到未删除任务，但是到vTaskDelete又出错,防止中间任务切换

if(pxTuneSegmentTaskHandler!=NULL)	//保证第一次执行不错
{
    state = eTaskGetState(*pxTuneSegmentTaskHandler); //如果为pxTuneSegmentTaskHandler = null则，其取出内容的state错误
    if (state != eDeleted && state != eInvalid )
    {
        printf("state:%d\r\n",state);
        vTaskDelete(*pxTuneSegmentTaskHandler);	//已经运行完，删除了就不删第二次	
        pxTuneSegmentTaskHandler = NULL;
        stop();
    }
}

xSemaphoreGive( xMutexTuneCounting );
-----------------------------------------------------------
xSemaphoreTake(xMutexTuneCounting,portMAX_DELAY);

pxTuneSegmentTaskHandler = NULL; //设置为被杀掉，其他任务无需再杀

xSemaphoreGive( xMutexTuneCounting );
vTaskDelete(NULL);
```

通过一个pxTuneSegmentTaskHandler == NULL作为任务删除的依据 / 一个任务的代号 / 互斥访问的变量，可以将自杀、他杀的互斥，转换为更清晰的他杀他杀互斥。注意，这里的pxTuneSegmentTaskHandler 是指针！！！

### 2.3 TuneTask和DisplayTask---内存泄漏

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

![image-20240826132951362](E:\0_wjx_file\embedded_ai\study_data\0 note\nwatch\image-20240826132951362.png)

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

### 2.4.定时计时功能

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

![image-20240903140427176](E:\0_wjx_file\embedded_ai\study_data\0 note\nwatch\image-20240903140427176.png)

menu与button,display的具体关系---以games为例

![ButtonTask_menu](E:\0_wjx_file\embedded_ai\study_data\0 note\nwatch\ButtonTask_menu.jpg)

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

