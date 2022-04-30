说开发过程中的几个难点：

1.其实不算难点，就是添加各个路径下的.c文件，很痛苦，很多

2.切换C99和AC6编译后，会有一大堆原始DEMO中的旧版文件，eg:cm4_core.h这种keil 报 unknown register name 'vfpcc' in asm，上网查这个问题都是一个答案：		　　将Target标签下的ARM complier改为版本5即可

	气得我直骂人，本来就是要换高版本编译器，你让我换回去？！xx
		处理方法如下：
		在keil中编译cortex M4内核单片机时，由于使用了AC6编译器，导致报unknown register name 'vfpcc' in asm 错误，新编译器支持C++11无需人工加 --cpp11 选项，所以必须使用新编译器实现
		这个错误是因为core_cm4.h文件版本太低导致，在keil目录下搜索core_cm4.h，将工程引用的头文件替换成此文件即可
		替换文件后，还需要cmsis_version.h，armv4xxxx.h，mpuxxxx.h等几个头文件，建议下个everything软件，快速找一下电脑其他高版本的同名官方.h文件，打开头部就能看到版本号，选择2016版后的，即可通过编译。

参考自：https://www.cnblogs.com/yangzifb/p/14212748.html
…………
还有一些，LVGL都大概玩明白后继续写……

以下部分来自野火Demo
/*********************************************************************************************/
本文档使用 TAB = 4 对齐，使用keil5默认配置打开阅读比较方便。
【*】程序简介

-工程名称：LTDC—液晶显示英文
-实验平台: 野火STM32 F429 开发板 
-MDK版本：5.16
-ST固件库版本：1.5.1

【 ！】功能简介：
驱动5寸液晶屏，显示英文、绘制各种图形，使用液晶双层特效。

学习目的：学习STM32的LTDC驱动液晶屏，了解DMA2D图形加速器。


【 ！】实验操作：
连接好配套的5.0寸液晶屏，下载程序后复位开发板即可，屏幕会绘制文字及图形。

【*】注意事项：
无

【*】显存空间：
驱动液晶屏必须要使用SDRAM，用作显存。
液晶屏有两层，每层有独立显存空间，每层占据 [屏幕像素*像素宽度] 的空间。
驱动好LTDC后并设置好对应的层显示效果，直接往显存写数据就会显示到液晶上。

在bsp_lcd.h头文件的LCD_RGB_888宏可配置RGB888或RBG565模式。                  
显存分配：
RGB888,像素宽度3字节。
第一层：地址 (0xD0000000)至((0xD0000000 + 3*液晶屏像素宽*液晶屏像素高)-1)
第二层：地址 (0xD0000000 + 3*液晶屏像素宽*液晶屏像素高) 至(0xD0000000 + 2*3*液晶屏像素宽*液晶屏像素高) 

显存分配：
RGB565，像素宽度2字节。
第一层：地址 (0xD0000000)至((0xD0000000 + 2*液晶屏像素宽*液晶屏像素高)-1)
第二层：地址 (0xD0000000 + 2*液晶屏像素宽*液晶屏像素高) 至（(0xD0000000 + 2*2*液晶屏像素宽*液晶屏像素高)-1） 

【*】性能
800*480分辨率时，
DMA2D刷纯色矩形：149帧/s 
普通DMA刷数据：90帧/s
普通DMA按行刷SD卡的BMP图片：6帧/s（主要受限SD卡的速度）
直接用指针操作往显存描点：28帧/s


LTDC时钟频率：10MHz （可在LCD_Init函数配置）

【*】 引脚分配

液晶屏：
液晶屏接口与STM32的LTDC接口相连，支持RGB888、565格式，
STM32直接驱动，无需外部液晶屏驱动芯片.

		/*液晶控制信号线*/		
		CLK		<--->PG7
		HSYNC	<--->PI10
		VSYNC	<--->PI9
		DE		<--->PF10
		DISP	<--->PD4
		BL		<--->PD7
		
		/*电容触摸屏信号线*/		
		RSTN	<--->PD13
		INT		<--->PD12
		SDA		<--->PH5
		SCL		<--->PH4

RGB信号线省略,本实验没有驱动触摸屏，详看触摸画板实验。


SDRAM （W9825G6 32M 字节）：
SDRAM芯片的接口与STM32的FMC相连。
		/*控制信号线*/
		CS	<--->PH6
		BA0	<--->PG4
		BA1	<--->PG5
		WE	<--->PC0
		CS	<--->PH6
		RAS	<--->PF11
		CAS	<--->PG15
		CLK	<--->PG8
		CKE	<--->PH7
		UDQM<--->PE1
		LDQM<--->PE0
		
地址和数据信号线省略，本连接的SDRAM基地址为 (0xD0000000)，结束地址为(0xD2000000),大小为32M字节
												
/*****************************************************************************************************/


【*】 时钟

A.晶振：
-外部高速晶振：25MHz
-RTC晶振：32.768KHz

B.各总线运行时钟：
-系统时钟 = SYCCLK = AHB1 = 180MHz
-APB2 = 90MHz 
-APB1 = 45MHz

C.浮点运算单元：
  使用