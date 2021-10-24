<p align="center">
 <img width="100px" src="https://github.com/NekoSilverFox/NekoSilverfox/blob/master/icons/silverfox.svg" align="center" alt="Corner detection on ZYNQ" />
 <h1 align="center">Corner detection on ZYNQ</h2>
 <p align="center"><b>Based on XC7Z020clg400-2 Soc</b></p>
</p>


<div align=center>
[![License](https://img.shields.io/badge/license-Apache%202.0-brightgreen)](LICENSE)
![ZYNQ](https://img.shields.io/badge/ZYNQ-ZYNQ--7-orange)
![SoC](https://img.shields.io/badge/SoC-XC7Z020clg400--2-orange)

![Vivado](https://img.shields.io/badge/Vivado-v2016.3%20v2018.3-blue)
![SDK](https://img.shields.io/badge/SDK-v2016.3%20v2018.3-blue.svg)
![Kernel](https://img.shields.io/badge/Kernel-Linux%20v4.14-yellow)

<div align=left>

## 实验说明

实现 OV5640 摄像头实时采集图像数据并通过 ZYNQ-7 7020clg400-2 SoC 处理实现角点监测，并通过 HDMI 输出经过处理的灰度画面

## 硬件平台

- Board：Mizar Z7
- SoC：Zynq-7 XC7Z020clg400-2
- Camera：OV5640
- Screen：Xiaomi 34''

## 软件平台

- Vivado v2018.3
- Xilinx SDK v2018.3
- Vivado HLS v2018.3

## 实验流程

1. HLS 开发部分
2. Vivado 设计部分
3. Xilinx SDK 程序设计部分
4. 开发板验证

### 1.  HLS 开发部分

1. 打开 Vivado HLS v2018.3 并选择 `Create New Project`，输入项目名称和路径

   ![image-20211024201308932](README.assets/image-20211024201308932.png)

2. 点击 `Next`，在 `Solution Configuration` 处选择 `xc7z020clg400-2`

   ![image-20211024201502409](README.assets/image-20211024201502409.png)

3. 在 `Source` 处导入测试图片文件及以下代码

   - CornerDetect.h

     ```C++
     #ifndef __CORNER_HEAD_H_
     #define __CORNER_HEAD_H_
     
     #include "hls_video.h"
     #include "ap_axi_sdata.h"
     
     //定义图像的大小
     #define WIDTH 1024
     #define HEIGHT 768
     
     //定义输入输出图像
     #define SRC_IMAGE "screen.bmp"		//输入图像路径
     #define DST_IMAGE "DstImage.bmp"	//输出图像路径
     #define GOLD_IMAGE "GoldImage.bmp"	//参考图像路径
     
     //定义图像数据类型
     
     //定义AXI-Stream数据流
     typedef ap_axiu<24,1,1,1> int_sideChannel;
     typedef hls::stream<ap_axiu<24,1,1,1> > AXI_STREAM ;
     
     //定义图像矩阵，其中图像矩阵中的像素格式为3通道8位无符号数，RGB图像
     typedef hls::Mat<HEIGHT,WIDTH,HLS_8UC3> IMAGE_RGB;
     
     //定义图像矩阵的每个像素格式，3通道8位无符号型
     typedef hls::Scalar<3,unsigned char> PIXEL_RGB;
     
     //定义图像矩阵，其中图像矩阵中的像素格式为1通道8位无符号数，灰度图像
     typedef hls::Mat<HEIGHT,WIDTH,HLS_8UC1> IMAGE_GRAY;
     
     //定义图像矩阵的每个像素格式，1通道8位无符号型
     typedef hls::Scalar<1,unsigned char> PIXEL_GRAY;
     
     
     //top function
     void rgb2gray(IMAGE_RGB & imgIn, IMAGE_RGB & imgOut_3C, IMAGE_GRAY & imgOut_1C);
     void doCorner(AXI_STREAM & inStream,AXI_STREAM & outStream);
     #endif
     ```

   - CornerDetect.cpp

     ```C++
     #include "CornerDetect.h"
     
     void rgb2gray(IMAGE_RGB & imgIn, IMAGE_RGB & imgOut_3C, IMAGE_GRAY & imgOut_1C){
     	PIXEL_RGB 	pixIn;
     	PIXEL_RGB 	pixOut_3C;
     	PIXEL_GRAY	pixOut_1C;
     
     	for(int idxRow = 0; idxRow < HEIGHT; idxRow++){
     		for(int idxCol = 0; idxCol < WIDTH; idxCol++){
     			//输入图像是三通道的RGB图像
     			imgIn >> pixIn;
     			unsigned short R = pixIn.val[0];
     			unsigned short G = pixIn.val[1];
     			unsigned short B = pixIn.val[2];
     
     			//输出三通道图像，各个通道都是灰度值，最终显示出来的图像是灰度图
     			pixOut_3C.val[0] = (unsigned char) ((R*76 + G*150 + B*30) >> 8);
     			pixOut_3C.val[1] = (unsigned char) ((R*76 + G*150 + B*30) >> 8);
     			pixOut_3C.val[2] = (unsigned char) ((R*76 + G*150 + B*30) >> 8);
     
     			//输出单通道图像，只有一个通道，为灰度图像
     			pixOut_1C.val[0] = (unsigned char) ((R*76 + G*150 + B*30) >> 8);
     			imgOut_3C << pixOut_3C;
     			imgOut_1C << pixOut_1C;
     		}
     	}
     }
     
     void doCorner(AXI_STREAM & inStream,AXI_STREAM & outStream){
     #pragma HLS INTERFACE axis  port=outStream
     #pragma HLS INTERFACE axis  port=inStream
     #pragma HLS INTERFACE s_axilite port=return bundle=CTRL_BUS
     
     
     	IMAGE_RGB img_0;
     	IMAGE_RGB img_1;
     	IMAGE_GRAY img_2;
     	IMAGE_RGB img_3;
     	IMAGE_GRAY mask;
     	IMAGE_GRAY dmask;
     
     #pragma HLS dataflow
     #pragma HLS stream depth=20000 variable=img_1_.data_stream
     	hls::AXIvideo2Mat(inStream, img_0);
     	PIXEL_RGB color(255,255,0);
     	rgb2gray(img_0, img_1, img_2);
     	hls::FASTX(img_2, mask, 20, true);			// 快速脚点检测
     	hls::Dilate(mask, dmask);					//对检测出来的点进行膨胀
     	hls::PaintMask(img_1, dmask, img_3, color);		//将膨胀后的点添加到图像
     	hls::Mat2AXIvideo(img_3, outStream);
     }
     ```

4. 在 `Test Bench` 处导入以下测试文件

   ```C++
   #include <stdio.h>
   #include <opencv2/opencv.hpp>
   #include "CornerDetect.h"
   #include "hls_opencv.h"
   using namespace cv ;
   
   
   //图片比对
   int image_compare(const char* output_image, const char* golden_image) {
       if (!(output_image) || !(golden_image)) {
           printf("Failed to open images...exiting.\n");
           return -1;
       } else {
           Mat o = imread(output_image);
           Mat g = imread(golden_image);
           assert(o.rows == g.rows && o.cols == g.cols);	//assert如果正确，则继续运行后面的程序，否则报错
           assert(o.channels() == g.channels() && o.depth() == g.depth());
           printf("rows = %d, cols = %d, channels = %d, depth = %d\n", o.rows, o.cols, o.channels(), o.depth());
           int flag = 0;
           for (int i = 0; i < o.rows && flag == 0; i++) {
               for (int j = 0; j < o.cols && flag == 0; j++) {
                   for (int k = 0; k < o.channels(); k++) {
                       unsigned char p_o = (unsigned char)*(o.data + o.step[0]*i + o.step[1]*j + k);
                       unsigned char p_g = (unsigned char)*(g.data + g.step[0]*i + g.step[1]*j + k);
                       if (p_o != p_g) {
                           printf("First mismatch found at row = %d, col = %d\n", i, j);
                           printf("(channel%2d) output:%5d, golden:%5d\n", k, p_o, p_g);
                           flag = 1;
                           break;
                       }
                   }
               }
           }
           if (flag)
               printf("Test Failed!\n");
           else
               printf("Test Passed!\n");
   
           return flag;
       }
   }
   
   int main(){
   	IplImage * SrcImage;				//输入的图像
   	IplImage * DstImage;				//输出的图像
   	SrcImage = cvLoadImage(SRC_IMAGE,-1);//导入图像
   
   	//创建opencv支持的输出的图像
   	DstImage = cvCreateImage(cvGetSize(SrcImage),SrcImage->depth, SrcImage->nChannels);
   	//AXI-Stream数据流
   	AXI_STREAM inStream;
   	AXI_STREAM outStream;
   
   	IplImage2AXIvideo(SrcImage,inStream);
   
   	//可综合的函数
   	doCorner(inStream, outStream);
   	//保存输出结果
   	AXIvideo2IplImage(outStream,DstImage);
   	cvSaveImage(DST_IMAGE,DstImage);
   
   	//参考图像输出
   	cvShowImage(DST_IMAGE, DstImage);
   	cvWaitKey(0);
   
   
   	//释放图片
   	cvReleaseImage(&SrcImage);
   	cvReleaseImage(&DstImage);
   	return 0;
   }
   ```

5. 在确认代码无误后，依次执行：

   1. Run C Simulatoin（仿真）
   2. Run C Synthesis（综合）
   3. Run C/RTL Cosimulation（联合仿真）
   4. Export RTL（导出 RTL）![image-20211024202730811](README.assets/image-20211024202730811.png)

   至此，我到了我们所需要的图像处理 IP

### 2. Vivado 设计部分

1. 新建 Vivado 工程，Soc 选择 `xc7z020clg400-2`

2. 导入自定义IP核

   ![image-20211024203206551](README.assets/image-20211024203206551.png)

3. 添加 `ZYNQ-7` IP核

   - 使能 UART 端口

   - Bank 1 的电压改为 1.8V

     ![image-20211024203429996](README.assets/image-20211024203429996.png)

   - 引出对应引脚

     配置完成如下图所示：

     ![image-20211024205512280](README.assets/image-20211024205512280.png)

4. 添加 OV5640 摄像头 IP 核

   并将连接外部引脚的端口信号引出，修改信号名称方便辨识及引脚约束。需引出信号如下图所示：

   ![image-20211024210038679](README.assets/image-20211024210038679.png)

   将PS 端的输出时钟作为该 IP 的驱动时钟，连接时钟信号如下图所示：

   ![image-20211024210139183](README.assets/image-20211024210139183.png)

5. 添加 Video in to AXI4-Stream IP 核（视频输出控制器）

   设置时钟模式为独立模式（Independent），如下图所示：

   ![image-20211024210733387](README.assets/image-20211024210733387.png)

   并将 OV5640 的摄像头信号进行以下连接：

   ![image-20211024210720502](README.assets/image-20211024210720502.png)

6. 添加 Clocking Wizard IP核

   设置输入时钟频率为50Mhz，设置输出时钟1 的输出频率为65Mhz，输出时钟2 的频率为325Mhz，设置复位信号为低电平有效。如下图所示：

   ![image-20211024211032965](README.assets/image-20211024211032965.png)

   ![image-20211024211229502](README.assets/image-20211024211229502.png)

7. 添加 Vector Logic 逻辑门电路IP

   设置门电路为1位并取反

   ![image-20211024211443115](README.assets/image-20211024211443115.png)

8. 连接时钟和复位信号如下

   ![image-20211024211906794](README.assets/image-20211024211906794.png)

9. 添加两个 VDMA IP

   > 使用 VDMA IP 核来实现对于 AXI4-Stream 类目标外设的高带宽直接存储器存储或读取来读取 DDR 中的数据。VDMA 读取到数据之后通过 AXI4-Stream to Video Out IP 核将数据流转换成视频协议的数据流

   `Frame Buffers` 选项可以选择 AXI VDMA 要处理的帧缓冲存储位置 的数量。由于本次显示实验只显示一张图片，数据只需要写入一次，因此不需要 设置多个帧缓存区域，这里设置为 1。因为本实验是从 DDR3 中读取数据输出给 LCD，所以只需要勾选 Enable Read Channel 就可以了，无需勾选 Enable Write Channel。

   `Memory Map Data Width` 选项可以为 MM2S 通道选择所需的 AXI4 数据宽度。此处保持默认 64 即可

   `Write/Read Burst Size` 用于指定突发写/读的大小，此处选择 32

   `Stream Data Width` 选 项可以选择 MM2S 通道的 AXI4-Stream 数据宽度。 有效值是 8 的倍数，最大 到 1024。 必须注意的是该值必须小于或等于 Memory Map Data Width。**此处因输出数据格式为 RGB888，设置为 24**

   `Line Buffer Depth` 选项可以选择 MM2S 通道的行缓冲深度（行缓冲区宽度 为 stream data 的大小） ，此处设置 **512**

   ![image-20211024204045417](README.assets/image-20211024204045417.png)

10. 导入在 HLS 中生成的自定义视频处理 IP 核

    并连接视频输出输入信号

    ![image-20211024204957875](README.assets/image-20211024204957875.png)

11. 连接 Video in to AXI4-Stream IP 的视频信号输出端口 `video_out` 和 VDMA 的数据输入端口 `S_AXIS_S2MM`。如下图所示：

    ![image-20211024212202126](README.assets/image-20211024212202126.png)

12. 导入 Timing IP核 （视频时序控制器）

    配置如下

    ![image-20211024212344015](README.assets/image-20211024212344015.png)

    ![image-20211024212430016](README.assets/image-20211024212430016.png)

    连接IP 核的输入时钟信号及复位信号，如下图所示:

    ![image-20211024212530302](README.assets/image-20211024212530302.png)

13. 导入 Video out（视频输出控制器）

    这里我们使用了独立时钟作为输入，所以选择独立时钟

    ![image-20211009231913924](../ZYNQ/README.assets/image-20211009231913924.png)

14. 添加非门逻辑电路，之后如下图所示连接相关信号

    ![image-20211024223635532](README.assets/image-20211024223635532.png)

    ![image-20211024223850627](README.assets/image-20211024223850627.png)

15. 添加 HDMI IP

    将所有输出信号引出并修改信号名称

    ![image-20211024224043644](README.assets/image-20211024224043644.png)

    连接信号

    ![image-20211024224436573](README.assets/image-20211024224436573.png)

16. 添加 Constant IP

    设置输出位宽为1，输出高电平作为HDMI 输出使能信号

    ![image-20211024224529858](README.assets/image-20211024224529858.png)

    导出引脚

    ![image-20211024224623909](README.assets/image-20211024224623909.png)

17. 添加 Processor System Reset IP

    并按以下方式连接

    ![image-20211024225427974](README.assets/image-20211024225427974.png)

18. 点击Run Connection Automation 自动连接信号

19. 最终得到的 Block Deign 如下

    ![image-20211024225517565](README.assets/image-20211024225517565.png)

20. 生成顶层 HDL 文件

21. 进行管脚约束

    ```xdc
    set_property PACKAGE_PIN N18 [get_ports {cmos_pclk}]
    set_property PACKAGE_PIN Y16 [get_ports {cmos_href}]
    set_property PACKAGE_PIN Y19 [get_ports {cmos_vsync}]
    set_property PACKAGE_PIN Y17 [get_ports {cmos_rst_n}]
    set_property PACKAGE_PIN N17 [get_ports {cmos_sda}]
    set_property PACKAGE_PIN P19 [get_ports {cmos_scl}]
    
    set_property PACKAGE_PIN P20 [get_ports {cmos_data[9]}]
    set_property PACKAGE_PIN N20 [get_ports {cmos_data[8]}]
    set_property PACKAGE_PIN T17 [get_ports {cmos_data[7]}]
    set_property PACKAGE_PIN R18 [get_ports {cmos_data[6]}]
    set_property PACKAGE_PIN T20 [get_ports {cmos_data[5]}]
    set_property PACKAGE_PIN V20 [get_ports {cmos_data[4]}]
    set_property PACKAGE_PIN P18 [get_ports {cmos_data[3]}]
    set_property PACKAGE_PIN U20 [get_ports {cmos_data[2]}]
    set_property PACKAGE_PIN Y18 [get_ports {cmos_data[1]}]
    set_property PACKAGE_PIN W20 [get_ports {cmos_data[0]}]
    
    set_property IOSTANDARD LVCMOS33 [get_ports cmos_scl]
    set_property IOSTANDARD LVCMOS33 [get_ports cmos_sda]
    set_property IOSTANDARD LVCMOS33 [get_ports cmos_rst_n]
    set_property IOSTANDARD LVCMOS33 [get_ports cmos_pclk]
    set_property IOSTANDARD LVCMOS33 [get_ports cmos_vsync]
    set_property IOSTANDARD LVCMOS33 [get_ports cmos_href]
    set_property IOSTANDARD LVCMOS33 [get_ports {cmos_data[*]}]
    
    set_property CLOCK_DEDICATED_ROUTE FALSE [get_nets cmos_pclk_IBUF]
    
    set_property PACKAGE_PIN K17 [get_ports hdmi_tx_clk_p]
    set_property PACKAGE_PIN G19 [get_ports {hdmi_tx_chn_r_p}]
    set_property PACKAGE_PIN F19 [get_ports {hdmi_tx_chn_g_p}]
    set_property PACKAGE_PIN D19 [get_ports {hdmi_tx_chn_b_p}]
    set_property IOSTANDARD LVCMOS33 [get_ports {hdmi_oen[0]}]
    set_property PACKAGE_PIN M20 [get_ports {hdmi_oen[0]}]
    set_property IOSTANDARD TMDS_33 [get_ports hdmi_tx_chn_r_p]
    set_property IOSTANDARD TMDS_33 [get_ports hdmi_tx_chn_g_p]
    set_property IOSTANDARD TMDS_33 [get_ports hdmi_tx_chn_b_p]
    set_property IOSTANDARD TMDS_33 [get_ports hdmi_tx_clk_p]
    ```

22. 生成 Bitstream

23. Export Hardware（Include Bitstream）

24. 启动 SDK

### 3. Xilinx SDK 程序设计部分

1. 新建工程

2. 添加以下文件

   ```c
   #include <stdio.h>
   #include "platform.h"
   #include "xil_printf.h"
   #include "vdma_api/vdma_api.h"
   #include "xdocorner.h"
   #include "xparameters.h"
   #include "xil_cache.h"
   
   
   #define DISPLAY_VDMA_DEV_ID 	XPAR_AXI_VDMA_1_DEVICE_ID
   #define HLS_VDMA_DEV_ID 		XPAR_AXI_VDMA_0_DEVICE_ID
   #define CORNER_DEV_ID 			XPAR_DOCORNER_0_DEVICE_ID
   
   
   #define DISP_BASE_ADDR 			0x08000000
   #define HLS_BASE_ADDR			0x03000000
   #define SCREEN_X				1024
   #define SCREEN_Y				768
   
   
   static XAxiVdma					Vdma;
   static XDocorner 				doCorner 	;
   static XDocorner_Config 		*doCorner_Cfg;
   
   
   int initdoCorner(){
   	int status;
   	doCorner_Cfg = XDocorner_LookupConfig(CORNER_DEV_ID);
   	status = XDocorner_CfgInitialize(&doCorner ,doCorner_Cfg);
   	if(status != XST_SUCCESS){
   		printf("initialize failed! \n");
   		return status;
   	}
   	return status;
   }
   
   
   
   //设置显示器背景
   void setBackground(){
   	u32 *memAddr;
   	int idxRow;
   	int idxCol;
   	memAddr = (u32 *) DISP_BASE_ADDR;
   	for(idxRow = 0;idxRow < SCREEN_Y; idxRow++){
   		for(idxCol = 0;idxCol < SCREEN_X; idxCol++){
   			if(idxRow < SCREEN_Y/2){
   				memAddr[idxCol + idxRow*SCREEN_X] = 0;
   			}
   			else{
   				memAddr[idxCol + idxRow*SCREEN_X] = 0xFFFFFF;
   			}
   		}
   	}
   	Xil_DCacheFlush();
   }
   
   
   
   
   
   int main()
   {
   	printf("initialize running! \n");
   	int status;
       //初始化VDMA并进行配置
   	run_vdma_frame_buffer(&Vdma, HLS_VDMA_DEV_ID, SCREEN_X, SCREEN_Y,HLS_BASE_ADDR,0, 0,BOTH);
   	run_vdma_frame_buffer(&Vdma, DISPLAY_VDMA_DEV_ID, SCREEN_X, SCREEN_Y,DISP_BASE_ADDR,0, 0,BOTH);
   	status = initdoCorner();
   	if(status != XST_SUCCESS){
   		printf("initialize failed! \n");
   		return status;
   	}
   	setBackground();
   
   	while(1){
   		XDocorner_Start(&doCorner);
   		while(!XDocorner_IsDone(&doCorner)){
   
   		}
   	}
       return 0;
   }
   
   ```

### 4. 开发板验证

1. 将开发板各个接口如以下方式连接

   - UART 和 JTAG 串口连接电脑

   - OV5640 摄像头连接 GPIO1插槽

   - HDMI 显示器连接在 HDMI TX接口

   - 模式切换到 JTAG

   - 连接电源

     连接如图所示：

     ![image-20211024230736115](README.assets/image-20211024230736115.png)

2. 在 SDK 中写入 Bitstream 后开始运行

3. 可以观察到成功的对检测到了角点，并且进行了标注

   ![image-20211024231410716](README.assets/image-20211024231410716.png)

## 引用

[Xilinx xp1167 Accelerating OpenCV Applications with Zynq-7000 All Programmable SoC using Vivado HLS Video Libraries](https://www.xilinx.com/support/documentation/application_notes/xapp1167.pdf)

[Xilinx UG902 Vivado Design Suite User Guide High-Level Synthesis](https://china.xilinx.com/support/documentation/sw_manuals/xilinx2020_1/c_ug902-vivado-high-level-synthesis.pdf)

[Datasheet OV5640](https://cdn.sparkfun.com/datasheets/Sensors/LightImaging/OV5640_datasheet.pdf)

[MIZAR Z7 Circuit Schematic]()

[Dcam 5M OV5640 Circuit Schematic]()



