# 参考资源

Chipwhisperer官方文档：https://chipwhisperer.readthedocs.io/en/latest/

Chipwhisperer官方demo: https://github.com/newaetech/chipwhisperer-jupyter

Pico示波器Python脚本：https://github.com/picotech/picosdk-python-wrappers

# 使用Chipwhisperer-Lite进行数据采集

## 如何连接

<img src="https://205e55-1258657022.cos.ap-nanjing.myqcloud.com/chipwhisper/20221114155607.jpg" style="zoom: 25%;" />

粉色的波形采集线要连接到Chipwhisperer-Lite的MEASURE端而不是GLITCH端

## c与asm程序

程序放在`D:\ChipWhisperer5_64\cw\home\portable\chipwhisperer\tracesCapture\xxx_firmware`文件夹下。应有`Makefile` `.h` `.c` `.S`(注意是**大写**)文件。

`main.c`文件:

```c
#include "hal.h"
#include <stdint.h>
#include <string.h>
#include <stdlib.h>
#include <stdio.h>
#include "simpleserial.h"

uint8_t input_array[10];
uint8_t result_array[10];
uint8_t process(uint8_t* data, uint8_t dlen)
{

	memcpy(input_array, data, 10);		//将输入数据的前10个字节拷贝到input_array    
	trigger_high();		//触发设置为高电平
    //设置触发的代码可以写成汇编形式:
    // 	    asm volatile(
	//   "PUSH {r0-r12}"
    //	 "PUSH {lr}}"
	//   "BL  trigger_high"	
    //   "POP {lr}"
	// 	 "POP {r0-r12}"
	// );
    uint8_t result = your_function();
	trigger_low();		//触发设置为低电平
	simpleserial_put('r', 1, &result);	//输出长度为1个字节的结果
	// simpleserial_put('r', 10, result_array);	//将result_array数组中的前10个字节，用r命令，传回电脑
	return 0;
}

uint8_t test(uint8_t* data, uint8_t dlen)
{	
	xxx;
    return 0
}
int main(void)
{
	platform_init();
	init_uart();
	trigger_setup();	
	simpleserial_init();
    
	//当接收到g命令时，会执行process方法
	simpleserial_addcmd('g', 16, process);	//注册g命令，传送16个字节，注意这里的g和16必须与jupyter中的脚本对应一致
    
    //可以注册多个方法
	//simpleserial_addcmd('t', 1, test);
	while(1)
		simpleserial_get();
  	return 0;
}

```

`Makefile`文件：

```makefile
TARGET = simple-test

# List C source files here.
# Header files (.h) are automatically pulled in.
# 添加.c文件，不需要添加.h文件
SRC += main.c tables.c

# -----------------------------------------------------------------------------
ifeq ($(CRYPTO_OPTIONS),)
CRYPTO_OPTIONS = NONE
endif


OPT = 0	# 关闭优化
# 添加.S文件
ASRC += ../../../tracesCapture/blinky_firmware/asm.S
#Add simpleserial project to build
include ../../hardware/victims/firmware/simpleserial/Makefile.simpleserial

FIRMWAREPATH = ../../hardware/victims/firmware/.
include $(FIRMWAREPATH)/Makefile.inc
```



## Jupyter脚本

Jupyter脚本放在`D:\ChipWhisperer5_64\cw\home\portable\chipwhisperer\tutorials`路径下

设置SimpleSerial版本为`1.1`
设置设备为`STM32F3`

```python
SCOPETYPE = 'OPENADC'
PLATFORM = 'CW308_STM32F3' 如果用的是带随机数的板子，则这里是CW308_STM32F4
SS_VER = 'SS_VER_1_1'
```

编译：

```shell
%%bash -s "$PLATFORM" "$SS_VER"
cd 存放代码的文件夹
make PLATFORM=$1 CRYPTO_TARGET=NONE SS_VER=$2
```

编译后生成hex文件

检测板子的连接性：（Setup_Generic.ipynb在chipwhisperer-jupyter[仓库](https://github.com/newaetech/chipwhisperer-jupyter)中下载）

```
%run "./Setup_Scripts/Setup_Generic.ipynb"
```

写入固件：

```python
cw.program_target(scope, prog, "xxx.hex".format(PLATFORM))
```

可选，对示波器进行配置：

```python
scope.adc.samples = 5000	# 采样点的数量
scope.adc.presamples = 300	# 预采样的数量
scope.adc.offset = 0	
# scope.adc.decimate = 4  # 可以调节下采样比例
```

```
%matplotlib notebook
import matplotlib.pylab as plt
```



```python
import numpy as np
import random
from tqdm import tnrange
from tqdm.auto import tqdm
from numpy import savetxt
import galois
import gc

################下方这里是参数部分，可以修改#######
traces_per_part = 500  # 分块保存曲线，这里指定每一块文件里有多少条曲线
part = 200  # 分成多少块文件
part_start_index = 0  # 序号从多少号开始。如果是初始采集，就设置为0
filename_of_traces = "aes_capture"	# 曲线文件的文件名前缀
##############################################

global_counter = 0  # 全局的counter，用来判断这条曲线是奇数还是偶数
total_traces = part * traces_per_part  # 总共要采集的曲线数目
pbar = tqdm(total=total_traces)


# 此函数用于采集并保存一块文件
def mainProcess(num_of_traces):
    traces_array = []
    text_in_array = []
    text_out_array = []

    for i in tnrange(0, num_of_traces):

        reset_target(scope)  # 对板子复位
        global quanjucounter
        global pbar
		
        # 因为要对曲线做t-test，所以奇数条和偶数条是不同的输入数据:
        if quanjucounter % 2 == 0:  # # 如果是偶数曲线
            list_to_send = [1, 2, 3]  # 表示输入的数据
            msg = bytearray(list_to_send)
            scope.arm()
            target.simpleserial_write('g', msg)  # 这里的'g'命令要和c文件中的对应一致
            time.sleep(0.05)  # ❗这里必须要sleep,如果采集出错，尝试对睡眠时间进行调整

        
        else:	# 如果是奇数曲线
            list_to_send = [4, 5, 6]
            msg = bytearray(list_to_send)
            scope.arm()
            target.simpleserial_write('g', msg)
            time.sleep(0.05)	# ❗这里必须要sleep,如果采集出错，尝试对睡眠时间进行调整
        ret = scope.capture()
        if ret:
            print("Target timed out!")
        trace = scope.get_last_trace()  # 获得曲线
        recv_msg = target.simpleserial_read('r', 1)  # 这里的r命令要和c文件中对应一致

        traces_array.append(list(trace))
        text_in_array.append(list_to_send)
        text_out_array.append(list(recv_msg))
        assert(验证输出是否正确)
        quanjucounter += 1
        pbar.update(1)  # 更新进度条

    return np.array(traces_array), np.array(text_in_array), np.array(text_out_array)
for p in range(part):  # 遍历所有的part
    traces_arr, text_in_arr, text_out_arr = mainProcess(traces_per_part)
    np.save(filename_of_traces + "tracesPart{0}.npy".format(p + part_start_index), traces_arr)
    np.save(filename_of_traces + "textinPart{0}.npy".format(p + part_start_index), text_in_arr)
    np.save(filename_of_traces + "textoutPart{0}.npy".format(p + part_start_index), text_out_arr)
    counter += 1
    del traces_array  # 回收内存
    del text_in_arr
    del text_out_arr
    gc.collect()

pbar.close()
```



# 使用Pico示波器的BLOCK模式进行数据采集

## 如何连接

<img src="https://205e55-1258657022.cos.ap-nanjing.myqcloud.com/chipwhisper/f6ddba7001b958cbcfdc9acb2ee0b31.jpg" style="zoom: 33%;" />

1. 示波器的镊子连接红色板子的`GND`针脚
2. 示波器的抓钩连接红色板子的`GPIO4/TRIG`针脚
3. 注意连接时，线缆中裸露的金属部位不要接触到其他的金属
4. 示波器的抓钩上的黄色开关，拨到`X1`位置处

## c与asm程序

同上

## Jupyter脚本

采集的流程：

![](https://205e55-1258657022.cos.ap-nanjing.myqcloud.com/%E5%85%AB%E8%82%A1%E6%96%87/clipboard_20230611_021035.png)

参考了[这里](https://github.com/picotech/picosdk-python-wrappers/blob/master/ps5000aExamples/ps5000aBlockExample.py) 和[PicoScope 5000 Series (A API) Programmer's Guide](https://www.picotech.com/download/manuals/picoscope-5000-series-a-api-programmers-guide.pdf)

```python
SCOPETYPE = 'OPENADC'
PLATFORM = 'CW308_STM32F3'
SS_VER = 'SS_VER_1_1'
```



```shell
%%bash -s "$PLATFORM" "$SS_VER"
cd ../xx
make PLATFORM=$1 CRYPTO_TARGET=NONE SS_VER=$2
```



```python
%run "./Setup_Scripts/Setup_Generic.ipynb"
```



```
cw.program_target(scope, prog, "../xxx/simple-test-{}.hex".format(PLATFORM))
```



在`PicoScope`[官网](https://www.picotech.com/downloads)下载PythonSDK，并在`Python`中安装：

```
pip install picosdk
```

### **打开示波器**

注意导入picosdk.ps5000a而不是picosdk.ps5000

```python
import ctypes
import numpy as np
import matplotlib.pyplot as plt
from tqdm import tnrange
from tqdm.auto import tqdm
from picosdk.functions import adc2mV, assert_pico_ok, mV2adc
from picosdk.ps5000a import ps5000a as ps

chandle = ctypes.c_int16()
status = {}

# 设置示波器的BIT，默认为12BIT
resolution =ps.PS5000A_DEVICE_RESOLUTION["PS5000A_DR_12BIT"]
status["openunit"] = ps.ps5000aOpenUnit(ctypes.byref(chandle), None, resolution)
try:
    assert_pico_ok(status["openunit"])
except: # PicoNotOkError:
    powerStatus = status["openunit"]
    if powerStatus == 286:
        status["changePowerSource"] = ps.ps5000aChangePowerSource(chandle, powerStatus)
    elif powerStatus == 282:
        status["changePowerSource"] = ps.ps5000aChangePowerSource(chandle, powerStatus)
    else:
        raise
    assert_pico_ok(status["changePowerSource"])
```

### **设置示波器的采集参数**

（示波器打开后，下面的块可以多次运行调整参数，无需重启示波器）

```python
# 注意这里要改成直流电！
channel = ps.PS5000A_CHANNEL["PS5000A_CHANNEL_A"]
coupling_type = ps.PS5000A_COUPLING["PS5000A_DC"]
# 更改A通道（采集触发信号的通道）的范围
chARange = ps.PS5000A_RANGE["PS5000A_20V"]
# analogue offset = 0 V
status["setChA"] = ps.ps5000aSetChannel(chandle, channel, 1, coupling_type, chARange, 0)
assert_pico_ok(status["setChA"])

channel = ps.PS5000A_CHANNEL["PS5000A_CHANNEL_B"]
# enabled = 1

# 注意这里要改成交流电！
coupling_type2 = ps.PS5000A_COUPLING["PS5000A_AC"]
# chBRange = ps.PS5000A_RANGE["PS5000A_2V"]
# 更改B通道（采集功耗数据的通道）的范围
chBRange = ps.PS5000A_RANGE["PS5000A_200MV"]
# analogue offset = 0 V
status["setChB"] = ps.ps5000aSetChannel(chandle, channel, 1, coupling_type2, chBRange, 0)
assert_pico_ok(status["setChB"])

maxADC = ctypes.c_int16()
status["maximumValue"] = ps.ps5000aMaximumValue(chandle, ctypes.byref(maxADC))
assert_pico_ok(status["maximumValue"])

source = ps.PS5000A_CHANNEL["PS5000A_CHANNEL_A"]
threshold = int(mV2adc(1000,chARange, maxADC))

status["trigger"] = ps.ps5000aSetSimpleTrigger(chandle, 1, source, threshold, 2, 0, 1000)
assert_pico_ok(status["trigger"])

#####################采集参数，需修改###################
# 在触发之前采集的数量
preTriggerSamples = 0
# 在触发之后采集的数量
postTriggerSamples = 10000
# timebase = 8  , 20.8 MHZ
# 通过timebase来控制采样率
# 采样率的计算可以查手册
timebase = 8
#####################采集参数，需修改####################

maxSamples = preTriggerSamples + postTriggerSamples
timeIntervalns = ctypes.c_float()
returnedMaxSamples = ctypes.c_int32()
status["getTimebase2"] = ps.ps5000aGetTimebase2(chandle, timebase, maxSamples, ctypes.byref(timeIntervalns), ctypes.byref(returnedMaxSamples), 0)
assert_pico_ok(status["getTimebase2"])
```

导入画图包：

```python
%matplotlib notebook
import matplotlib.pylab as plt
```

### 进行采集

```python
################采集参数，需修改##############################
traces_per_part = 500  # 每一块文件多少条曲线
part = 200  # 分成多少块文件
part_start_index = 0  # 序号从多少号开始。如果是初始采集，就设置为0
filename_of_traces = "filename"	# 曲线文件的文件名前缀
################采集参数，需修改##############################
```



```python
# mainProcess用来采集一个batch(part)的数据
def mainProcess(num_of_traces):
	# 初始化 用于存放数据的数组
    traces_array = np.empty(shape=(traces_per_part,maxSamples),dtype=np.float64)
    text_in_array = np.empty(shape=(traces_per_part,输入数据的长度),dtype=np.int16)
    text_out_array = np.empty(shape=(traces_per_part,输出数据的长度),dtype=np.int16)
    
    for i in tnrange(num_of_traces):
        global global_counter
        global pbar
        #重置设备
        reset_target(scope)
        
        # 开始采集
        status["runBlock"] = ps.ps5000aRunBlock(chandle, preTriggerSamples, postTriggerSamples, timebase, None, 0, None, None)
        assert_pico_ok(status["runBlock"])
        
       
        if global_counter % 2 == 0:  # 如果是偶数曲线
            list_to_send = [1, 2, 3]  # 表示输入的数据
            msg = bytearray(list_to_send)
            target.simpleserial_write('g', msg)  # 这里的'g'命令要和c文件中的对应一致
            time.sleep(0.01)	#❗❗❗❗❗❗❗❗❗❗❗❗❗虽然下面用while ready.value == check.value来等待采集，但是这里最好还是time.sleep一下，每次发送数据后，都最好sleep一下，否则会出错
        else:	# 如果是奇数曲线
            list_to_send = [4, 5, 6]
            msg = bytearray(list_to_send)
            target.simpleserial_write('g', msg)
            time.sleep(0.01)	#❗❗❗❗❗❗❗❗❗❗❗❗❗虽然下面while ready.value == check.value来等待采集，但是这里最好还是time.sleep一下，每次发送数据后，都最好sleep一下，否则会出错
            
        # 采集的时候奇数偶数曲线输入是不同的，一个为固定组，另一个为随机组。固定组的输入是固定值（如果输入要拆分成多个share的话，注意每个share是随机值）
        # 随机组的输入是随机值（如果输入要拆分成成多个share的话，同样，每个share是随即的）
		
        ready = ctypes.c_int16(0)
        check = ctypes.c_int16(0)
        # 等待触发，接收数据
        while ready.value == check.value:
        	status["isReady"] = ps.ps5000aIsReady(chandle, ctypes.byref(ready))
        

        # 接收完毕数据，准备数据容器
        bufferAMax = (ctypes.c_int16 * maxSamples)()
        bufferAMin = (ctypes.c_int16 * maxSamples)() 
        bufferBMax = (ctypes.c_int16 * maxSamples)()
        bufferBMin = (ctypes.c_int16 * maxSamples)() 
        source = ps.PS5000A_CHANNEL["PS5000A_CHANNEL_A"]
        status["setDataBuffersA"] = ps.ps5000aSetDataBuffers(chandle, source, ctypes.byref(bufferAMax), ctypes.byref(bufferAMin), maxSamples, 0, 0)
        assert_pico_ok(status["setDataBuffersA"])
        source = ps.PS5000A_CHANNEL["PS5000A_CHANNEL_B"]
        status["setDataBuffersB"] = ps.ps5000aSetDataBuffers(chandle, source, ctypes.byref(bufferBMax), ctypes.byref(bufferBMin), maxSamples, 0, 0)
        assert_pico_ok(status["setDataBuffersB"])
        overflow = ctypes.c_int16()
        cmaxSamples = ctypes.c_int32(maxSamples)
        
        # 数据从示波器端返回到数据容器
        status["getValues"] = ps.ps5000aGetValues(chandle, 0, ctypes.byref(cmaxSamples), 0, 0, 0, ctypes.byref(overflow))
        assert_pico_ok(status["getValues"])

        # convert ADC counts data to mV
        adc2mVChAMax =  adc2mV(bufferAMax, chARange, maxADC)
        adc2mVChBMax =  adc2mV(bufferBMax, chBRange, maxADC)
        # Create time data
        times = np.linspace(0, (cmaxSamples.value - 1) * timeIntervalns.value, cmaxSamples.value)
        recv_msg = target.simpleserial_read('r', 1)	#接收回显
        
        # ❗❗❗❗❗❗❗❗❗❗❗❗❗
        # 正式采集之前，先采集一条曲线的触发信号，并画图观察触发的上升沿和下降沿是否全部捕捉到，若看不到下降沿，则尝试增大postTriggerSamples
        #plt.plot(adc2mVChAMax[:])	# 画图观察触发信号是否正常
        #plt.plot(adc2mVChBMax[:])	# 画图观察数据信号
        if adc2mVChBMax[0] == 32512 and adc2mVChBMax[-1] == 32512:
            raise Exception('示波器采集错误，请重新采集')
        adc2mVChBMax = np.array(adc2mVChBMax)
        traces_array[i] = adc2mVChBMax
        text_in_array[i] = 输入的数据
        text_out_array[i] = 接收到的数据
        assert(验证输出是否正确)
        global_counter += 1
        pbar.update(1)	#更新进度条
    return traces_array, text_in_array, text_out_array
```



```python
global_counter = 0  # 全局的counter，用来判断这条曲线是奇数还是偶数
total_traces = part * traces_per_part  # 总共要采集的曲线数目
pbar = tqdm(total=total_traces)	# 设置进度条
for p in range(part):  # 遍历所有的part
    traces_arr, text_in_arr, text_out_arr = mainProcess(traces_per_part)
    np.save(filename_of_traces + "_tracesPart{0}.npy".format(p + part_start_index), traces_arr)
    np.save(filename_of_traces + "_textinPart{0}.npy".format(p + part_start_index), text_in_arr)
    np.save(filename_of_traces + "_textoutPart{0}.npy".format(p + part_start_index), text_out_arr)
pbar.close()
```

### **关闭示波器**

```python
# Stop the scope
# handle = chandle
status["stop"] = ps.ps5000aStop(chandle)
assert_pico_ok(status["stop"])

# Close unit Disconnect the scope
# handle = chandle
status["close"]=ps.ps5000aCloseUnit(chandle)
assert_pico_ok(status["close"])

# display status returns
print(status)
```

**示波器出现故障**时，先**关闭示波器**，释放资源，然后运行**打开示波器**的代码块，重新开始

# 使用Pico示波器的Rapid模式加快数据采集

前面的采集是BLOCK模式，示波器还有一个Rapid模式。可以一次性采集多条，可参考[这里](https://github.com/picotech/picosdk-python-wrappers/blob/master/ps5000aExamples/ps5000aRapidBlockExample.py)、 本目录下的另一个文件 和 [PicoScope 5000 Series (A API) Programmer's Guide](https://www.picotech.com/download/manuals/picoscope-5000-series-a-api-programmers-guide.pdf)



# 使用STM32F4来获得随机数

STM32F4带有随机数发生器，而STM32F3没有，并且STM32F4的内存和FLASH比STM32F3的要大。如果要使用STM32F4，前面的代码中的STM32F3全部要改成STM32F4.

main.c:

```c
#include "hal.h"
#include <stdint.h>
#include <string.h>
#include <stdlib.h>
#include <stdio.h>
#include "simpleserial.h"
void test_func();
uint32_t r_gen[1] = {0};						//存放生成的1个随机数
uint8_t GEN_RAND(uint8_t* data, uint8_t dlen){
    //int r_gen = get_rand();					// 生成随机数,c版本
	test_func();								// 生成随机数，汇编版本
    simpleserial_put('r', 4, (uint8_t*)r_gen);	//输出此随机数，大小为4个字节
    return 0;
}
int main(void)
{
	platform_init();
	init_uart();
	trigger_setup();	
	simpleserial_init();

	simpleserial_addcmd('a', 1, GEN_RAND);
	while(1)
		simpleserial_get();
  return 0;
}
```

asm.S:

```assembly
	.syntax unified
	.cpu cortex-m4
    .global test_func
test_func:
    push {r0-r11}
   
    push {lr}   //保存返回地址
    bl get_rand	//生成随机数，根据ARM调用约定AAPCS，第一个返回值保存在R0
    pop {lr}	//还原返回地址
    LDR r11, =r_gen	//自己定义的数组r_gen
    STR r0,[r11]//随机数存入数组r_gen

    pop {r0-r11}
    bx lr
```

# 一些坑：

0. 如果采集不到信号、触发，或采集数据质量差，首先检查接线是否无误。

1. 示波器开机第一次采集的时候，有几率会采集错误，开头的几条曲线都是`[ 32512  32512  32512 ...  32512  32512  32512]`这样的错误数值。此时无需重启示波器，只需重新采集即可。

2. 发送的数据长度不要超过`simpleserial`的上限，上限见官网文档.文档说SimpleSerial V1.*的上限是64字节，SimpleSerial V2的上限是192字节，但是我测试了下，SimpleSerial V1.1可以发送接收255字节

3. `simpleserial`协议有多个版本，见官网文档

4. 如果示波器出现故障，则重启的方法是，重新插拔线缆，`restart kernel`,重新运行代码

5. `.S` 文件不要和.c文件同名，否则会被makefile删掉

6. `.S`这个后缀名一定要大写

7. 如果采集看不到触发的上升沿或者下降沿，可能是示波器采集范围太短，另一种可能是代码执行过程中出错。对于后者，找到代码出错点的方法是，在代码中的不同位置放置`trigger_high()`。如果仍然出现玄学错误，考虑把蓝色板子卸下重新插拔，或是参考[这个](https://github.com/newaetech/chipwhisperer-jupyter/blob/master/demos/Debugging%20the%20CW308_STM32F3%20Using%20ChipWhisperer.ipynb)上板子调bug.

   ```
   arm-none-eabi-gdb simpleserial-aes-CW308_STM32F3.elf -ex "target extended-remote localhost:3333" -ex "monitor reset halt" -ex "load"
   (gdb) s
   (gdb) b main
   (gdb) r
   ```

8. 汇编中的.ltorg不能乱用，见https://www.jianshu.com/p/4c8194045201 ， LTORG不是随便放置的，随便放置有可能被当成汇编指令来执行，需要放置在无条件跳转的汇编指令之后或者子程序还回汇编指令之后。

9. 如果数组是uint32_t testvec[2] = {0x01020304,0x05060708}; 经过simpleserial_put('r', 8, (uint8_t* )testvec); 输出的顺序是04 03 02 01 08 07 06 05

10. 如果要切换simpleserial必须 修改jupyter中的SS_VER变量，更改c中的函数签名，重新编译，并重新运行一次Setup_Generic.ipynb，重新写入固件.SS_VER_2_1我用的时候碰到一些奇怪问题，暂时先用SS_VER_1_1了

11. 不要用malloc，之前用malloc出了奇怪的错误

12. 配置栈空间和堆空间的配置文件在ChipWhisperer5_64\cw\home\portable\chipwhisperer\hardware\victims\firmware\hal\stm32f3(stm32f4) 下面的LinkerScript.ld

13. 如果出现玄学错误，尝试在target.simpleserial_write()后面，time.sleep()多睡一段时间。

14. 自动触发：status["trigger"] = ps.ps5000aSetSimpleTrigger(chandle, 1, source, threshold, 2, 0, 1000) 中最后一个参数，表示如果1s内没有触发，则这里会自动触发。如果你的算法跑的时间特别长，则需要延长自动触发时间！

15. STM32F4的RNG有玄学问题，ttest结果很差。不要使用！不要使用！不要使用。即使是用作伪随机数发生器的种子，ttest也有问题。

16. 如果需要的随机数很多，最好的方法是PC传一个数作为种子，后续通过rand()来获得随机数。
