以下是在CW305 FPGA上的示例

之前的采集逻辑是，和板子交互一次，示波器返回一次采集数据。

Rapid模式采集的逻辑是，首先开启示波器，和板子交互N次，示波器**一次性**返回N次的采集数据。这样采集的速度更快。



```python
import ctypes
import numpy as np
import matplotlib.pyplot as plt
import serial
from tqdm import tqdm, trange
from picosdk.ps5000a import ps5000a as ps
from picosdk.functions import adc2mV, assert_pico_ok, mV2adc

# Create chandle and status ready for use
status = {}
chandle = ctypes.c_int16()

# Opens the device/s
status["openunit"] = ps.ps5000aOpenUnit(ctypes.byref(chandle), None, 1)

try:
    assert_pico_ok(status["openunit"])
except:  # PicoNotOkError:

    powerStatus = status["openunit"]

    if powerStatus == 286:
        status["changePowerSource"] = ps.ps5000aChangePowerSource(chandle, powerStatus)
    elif powerStatus == 282:
        status["changePowerSource"] = ps.ps5000aChangePowerSource(chandle, powerStatus)
    else:
        raise

    assert_pico_ok(status["changePowerSource"])

# Displays the serial number and handle
print(chandle.value)
# Set up channel A
# handle = chandle
channel = ps.PS5000A_CHANNEL["PS5000A_CHANNEL_A"]
# enabled = 1
coupling_type = ps.PS5000A_COUPLING["PS5000A_DC"]
chARange = ps.PS5000A_RANGE["PS5000A_2V"]
# analogue offset = 0 V
status["setChA"] = ps.ps5000aSetChannel(chandle, channel, 1, coupling_type, chARange, 0)
assert_pico_ok(status["setChA"])

# Set up chnnel B
# handle = chandle
channel = ps.PS5000A_CHANNEL["PS5000A_CHANNEL_B"]
# enabled = 1
# coupling_type = ps.PS5000A_COUPLING["PS5000A_DC"]
chBRange = ps.PS5000A_RANGE["PS5000A_50MV"]
# analogue offset = 0 V
status["setChB"] = ps.ps5000aSetChannel(chandle, channel, 1, coupling_type, chBRange, 0)
assert_pico_ok(status["setChB"])

# Finds the max ADC count
# Handle = chandle
# Value = ctype.byref(maxADC)
maxADC = ctypes.c_int16()
status["maximumValue"] = ps.ps5000aMaximumValue(chandle, ctypes.byref(maxADC))

# Set up single trigger
# handle = chandle
# enabled = 1
source = ps.PS5000A_CHANNEL["PS5000A_CHANNEL_A"]
threshold = int(mV2adc(1000, chARange, maxADC))
# direction = PS5000A_RISING = 2
# delay = 0 s
# auto Trigger = 1000 ms
status["trigger"] = ps.ps5000aSetSimpleTrigger(chandle, 1, source, threshold, 2, 0, 1000)
assert_pico_ok(status["trigger"])

# Setting the number of sample to be collected
preTriggerSamples = 150
postTriggerSamples = 150
maxsamples = preTriggerSamples + postTriggerSamples

# Gets timebase innfomation
# Warning: When using this example it may not be possible to access all Timebases as all channels are enabled by default when opening the scope.
# To access these Timebases, set any unused analogue channels to off.
# Handle = chandle
timebase = 2
# Nosample = maxsamples
# TimeIntervalNanoseconds = ctypes.byref(timeIntervalns)
# MaxSamples = ctypes.byref(returnedMaxSamples)
# Segement index = 0
timeIntervalns = ctypes.c_float()
returnedMaxSamples = ctypes.c_int32()
status["GetTimebase"] = ps.ps5000aGetTimebase2(chandle, timebase, maxsamples, ctypes.byref(timeIntervalns),
                                               ctypes.byref(returnedMaxSamples), 0)
assert_pico_ok(status["GetTimebase"])
portx = "COM7"
bps = 115200
# 超时设置,None：永远等待操作，0为立即返回请求结果，其他值为等待超时时间(单位为秒）
timex = 10
# 打开串口，并得到串口对象
ser = serial.Serial(portx, bps, timeout=timex)
# 清空缓冲区
ser.flushInput()  
# traces_all是总共采集的条数
# traces_each_file是每个文件采集的条数
traces_all = 500
traces_each_file = 100
# 用来判断奇偶
counter = 0
# traces_all / traces_each_file是最终产生的文件数
for p in range(0, traces_all // traces_each_file):
    
    # 用于保存采集的曲线
    arr = np.empty((traces_each_file, preTriggerSamples + postTriggerSamples), float)
    # 用于保存触发曲线
    arr_tr = np.empty((traces_each_file, preTriggerSamples + postTriggerSamples), float)
    # 用于保存输入数据
    text_in = np.empty((traces_each_file, preTriggerSamples + postTriggerSamples), float)
    # 用于保存输出数据
    text_out = np.empty((traces_each_file, preTriggerSamples + postTriggerSamples), float)

    for j in trange(0, traces_each_file):
        
        # 当j = 0 的时候，为j=0,1,2...9的曲线采集做准备，设置Segments数量，RunBlock，并设置缓冲区Buffer
        # 当j = 10 的时候，为j=10,11,12...19的曲线采集做准备，设置Segments数量，RunBlock，并设置缓冲区Buffer
        if j % 10 == 0:
            # Creates a overlow location for data
            overflow = ctypes.c_int16()
            # Creates converted types maxsamples
            cmaxSamples = ctypes.c_int32(maxsamples)
            status["MemorySegments"] = ps.ps5000aMemorySegments(chandle, 10, ctypes.byref(cmaxSamples))
            assert_pico_ok(status["MemorySegments"])
            # sets number of captures
            status["SetNoOfCaptures"] = ps.ps5000aSetNoOfCaptures(chandle, 10)
            assert_pico_ok(status["SetNoOfCaptures"])


            status["runblock"] = ps.ps5000aRunBlock(chandle, preTriggerSamples, postTriggerSamples, timebase, None, 0,
                                                    None, None)
            assert_pico_ok(status["runblock"])

            # Create buffers ready for assigning pointers for data collection
            bufferAMax = (ctypes.c_int16 * maxsamples)()
            bufferAMin = (ctypes.c_int16 * maxsamples)()  # used for downsampling which isn't in the scope of this example
            bufferBMax = (ctypes.c_int16 * maxsamples)()
            bufferBMin = (ctypes.c_int16 * maxsamples)()  # used for downsampling which isn't in the scope of this example
            # Setting the data buffer location for data collection from channel A
            # Handle = Chandle
            source = ps.PS5000A_CHANNEL["PS5000A_CHANNEL_A"]
            sourceB = ps.PS5000A_CHANNEL["PS5000A_CHANNEL_B"]
            status["SetDataBuffersA"] = ps.ps5000aSetDataBuffers(chandle, source, ctypes.byref(bufferAMax),
                                                                 ctypes.byref(bufferAMin), maxsamples, 0, 0)
            assert_pico_ok(status["SetDataBuffersA"])
            status["setDataBuffersB"] = ps.ps5000aSetDataBuffers(chandle, sourceB, ctypes.byref(bufferBMax),
                                                                 ctypes.byref(bufferBMin), maxsamples, 0, 0)
            assert_pico_ok(status["setDataBuffersB"])


            # Create buffers ready for assigning pointers for data collection
            bufferAMax1 = (ctypes.c_int16 * maxsamples)()
            bufferAMin1 = (ctypes.c_int16 * maxsamples)()  # used for downsampling which isn't in the scope of this example
            bufferBMax1 = (ctypes.c_int16 * maxsamples)()
            bufferBMin1 = (ctypes.c_int16 * maxsamples)()  # used for downsampling which isn't in the scope of this example
            status["SetDataBuffersA"] = ps.ps5000aSetDataBuffers(chandle, source, ctypes.byref(bufferAMax1),
                                                                 ctypes.byref(bufferAMin1), maxsamples, 1, 0)
            assert_pico_ok(status["SetDataBuffersA"])
            status["setDataBuffersB"] = ps.ps5000aSetDataBuffers(chandle, sourceB, ctypes.byref(bufferBMax1),
                                                                 ctypes.byref(bufferBMin1), maxsamples, 1, 0)
            assert_pico_ok(status["setDataBuffersB"])
            # Create buffers ready for assigning pointers for data collection
            bufferAMax2 = (ctypes.c_int16 * maxsamples)()
            bufferAMin2 = (ctypes.c_int16 * maxsamples)()  # used for downsampling which isn't in the scope of this example
            bufferBMax2 = (ctypes.c_int16 * maxsamples)()
            bufferBMin2 = (ctypes.c_int16 * maxsamples)()  # used for downsampling which isn't in the scope of this example

            status["SetDataBuffersA"] = ps.ps5000aSetDataBuffers(chandle, source, ctypes.byref(bufferAMax2),
                                                                 ctypes.byref(bufferAMin2), maxsamples, 2, 0)
            assert_pico_ok(status["SetDataBuffersA"])
            status["setDataBuffersB"] = ps.ps5000aSetDataBuffers(chandle, sourceB, ctypes.byref(bufferBMax2),
                                                                 ctypes.byref(bufferBMin2), maxsamples, 2, 0)
            assert_pico_ok(status["setDataBuffersB"])

            # Create buffers ready for assigning pointers for data collection
            bufferAMax3 = (ctypes.c_int16 * maxsamples)()
            bufferAMin3 = (ctypes.c_int16 * maxsamples)()  # used for downsampling which isn't in the scope of this example
            bufferBMax3 = (ctypes.c_int16 * maxsamples)()
            bufferBMin3 = (ctypes.c_int16 * maxsamples)()  # used for downsampling which isn't in the scope of this example

            status["SetDataBuffersA"] = ps.ps5000aSetDataBuffers(chandle, source, ctypes.byref(bufferAMax3),
                                                                 ctypes.byref(bufferAMin3), maxsamples, 3, 0)
            assert_pico_ok(status["SetDataBuffersA"])
            status["setDataBuffersB"] = ps.ps5000aSetDataBuffers(chandle, sourceB, ctypes.byref(bufferBMax3),
                                                                 ctypes.byref(bufferBMin3), maxsamples, 3, 0)
            assert_pico_ok(status["setDataBuffersB"])

            # Create buffers ready for assigning pointers for data collection
            bufferAMax4 = (ctypes.c_int16 * maxsamples)()
            bufferAMin4 = (ctypes.c_int16 * maxsamples)()  # used for downsampling which isn't in the scope of this example
            bufferBMax4 = (ctypes.c_int16 * maxsamples)()
            bufferBMin4 = (ctypes.c_int16 * maxsamples)()  # used for downsampling which isn't in the scope of this example

            status["SetDataBuffersA"] = ps.ps5000aSetDataBuffers(chandle, source, ctypes.byref(bufferAMax4),
                                                                 ctypes.byref(bufferAMin4), maxsamples, 4, 0)
            assert_pico_ok(status["SetDataBuffersA"])
            status["setDataBuffersB"] = ps.ps5000aSetDataBuffers(chandle, sourceB, ctypes.byref(bufferBMax4),
                                                                 ctypes.byref(bufferBMin4), maxsamples, 4, 0)
            assert_pico_ok(status["setDataBuffersB"])

            # Create buffers ready for assigning pointers for data collection
            bufferAMax5 = (ctypes.c_int16 * maxsamples)()
            bufferAMin5 = (ctypes.c_int16 * maxsamples)()  # used for downsampling which isn't in the scope of this example
            bufferBMax5 = (ctypes.c_int16 * maxsamples)()
            bufferBMin5 = (ctypes.c_int16 * maxsamples)()  # used for downsampling which isn't in the scope of this example

            status["SetDataBuffersA"] = ps.ps5000aSetDataBuffers(chandle, source, ctypes.byref(bufferAMax5),
                                                                 ctypes.byref(bufferAMin5), maxsamples, 5, 0)
            assert_pico_ok(status["SetDataBuffersA"])
            status["setDataBuffersB"] = ps.ps5000aSetDataBuffers(chandle, sourceB, ctypes.byref(bufferBMax5),
                                                                 ctypes.byref(bufferBMin5), maxsamples, 5, 0)
            assert_pico_ok(status["setDataBuffersB"])

            # Create buffers ready for assigning pointers for data collection
            bufferAMax6 = (ctypes.c_int16 * maxsamples)()
            bufferAMin6 = (ctypes.c_int16 * maxsamples)()  # used for downsampling which isn't in the scope of this example
            bufferBMax6 = (ctypes.c_int16 * maxsamples)()
            bufferBMin6 = (ctypes.c_int16 * maxsamples)()  # used for downsampling which isn't in the scope of this example

            status["SetDataBuffersA"] = ps.ps5000aSetDataBuffers(chandle, source, ctypes.byref(bufferAMax6),
                                                                 ctypes.byref(bufferAMin6), maxsamples, 6, 0)
            assert_pico_ok(status["SetDataBuffersA"])
            status["setDataBuffersB"] = ps.ps5000aSetDataBuffers(chandle, sourceB, ctypes.byref(bufferBMax6),
                                                                 ctypes.byref(bufferBMin6), maxsamples, 6, 0)
            assert_pico_ok(status["setDataBuffersB"])

            # Create buffers ready for assigning pointers for data collection
            bufferAMax7 = (ctypes.c_int16 * maxsamples)()
            bufferAMin7 = (
                        ctypes.c_int16 * maxsamples)()  # used for downsampling which isn't in the scope of this example
            bufferBMax7 = (ctypes.c_int16 * maxsamples)()
            bufferBMin7 = (ctypes.c_int16 * maxsamples)()  # used for downsampling which isn't in the scope of this example

            status["SetDataBuffersA"] = ps.ps5000aSetDataBuffers(chandle, source, ctypes.byref(bufferAMax7),
                                                                 ctypes.byref(bufferAMin7), maxsamples, 7, 0)
            assert_pico_ok(status["SetDataBuffersA"])
            status["setDataBuffersB"] = ps.ps5000aSetDataBuffers(chandle, sourceB, ctypes.byref(bufferBMax7),
                                                                 ctypes.byref(bufferBMin7), maxsamples, 7, 0)
            assert_pico_ok(status["setDataBuffersB"])

            # Create buffers ready for assigning pointers for data collection
            bufferAMax8 = (ctypes.c_int16 * maxsamples)()
            bufferAMin8 = (ctypes.c_int16 * maxsamples)()  # used for downsampling which isn't in the scope of this example
            bufferBMax8 = (ctypes.c_int16 * maxsamples)()
            bufferBMin8 = (ctypes.c_int16 * maxsamples)()  # used for downsampling which isn't in the scope of this example

            status["SetDataBuffersA"] = ps.ps5000aSetDataBuffers(chandle, source, ctypes.byref(bufferAMax8),
                                                                 ctypes.byref(bufferAMin8), maxsamples, 8, 0)
            assert_pico_ok(status["SetDataBuffersA"])
            status["setDataBuffersB"] = ps.ps5000aSetDataBuffers(chandle, sourceB, ctypes.byref(bufferBMax8),
                                                                 ctypes.byref(bufferBMin8), maxsamples, 8, 0)
            assert_pico_ok(status["setDataBuffersB"])

            # Create buffers ready for assigning pointers for data collection
            bufferAMax9 = (ctypes.c_int16 * maxsamples)()
            bufferAMin9 = (ctypes.c_int16 * maxsamples)()  # used for downsampling which isn't in the scope of this example
            bufferBMax9 = (ctypes.c_int16 * maxsamples)()
            bufferBMin9 = (ctypes.c_int16 * maxsamples)()  # used for downsampling which isn't in the scope of this example

            status["SetDataBuffersA"] = ps.ps5000aSetDataBuffers(chandle, source, ctypes.byref(bufferAMax9),
                                                                 ctypes.byref(bufferAMin9), maxsamples, 9, 0)
            assert_pico_ok(status["SetDataBuffersA"])
            status["setDataBuffersB"] = ps.ps5000aSetDataBuffers(chandle, sourceB, ctypes.byref(bufferBMax9),
                                                                 ctypes.byref(bufferBMin9), maxsamples, 9, 0)
            assert_pico_ok(status["setDataBuffersB"])

        # 如果是偶数曲线
        if counter % 2 == 0:
            for i in range(0, 11):
                seed[i] = np.random.randint(0, 256)
                ser.write(bytes([seed[i]]))

        # 如果是奇数曲线
        if counter % 2 == 1:
            for i in range(0, 11):
                seed[i] = np.random.randint(0, 256)
                ser.write(bytes([seed[i]]))

        # 计数器加1
        counter += 1

        # 当j = 9 的时候，ps5000aIsReady，ps5000aGetValuesBulk，从示波器中一次性获取j=0,1,2...9的曲线
        # 当j = 19 的时候，ps5000aIsReady，ps5000aGetValuesBulk，从示波器中一次性获取j=10,11,12...19的曲线
        if j % 10 == 9:
            # Creates a overlow location for data
            overflow = (ctypes.c_int16 * 10)()
            # Creates converted types maxsamples
            cmaxSamples = ctypes.c_int32(maxsamples)

            # Checks data collection to finish the capture
            ready = ctypes.c_int16(0)
            check = ctypes.c_int16(0)
            # 在这里等待多次触发完毕
            while ready.value == check.value:
                status["isReady"] = ps.ps5000aIsReady(chandle, ctypes.byref(ready))


            status["GetValuesBulk"] = ps.ps5000aGetValuesBulk(chandle, ctypes.byref(cmaxSamples), 0, 9, 0, 0,
                                                              ctypes.byref(overflow))
            assert_pico_ok(status["GetValuesBulk"])


            Times = (ctypes.c_int64 * 10)()
            TimeUnits = ctypes.c_char()
            status["GetValuesTriggerTimeOffsetBulk"] = ps.ps5000aGetValuesTriggerTimeOffsetBulk64(chandle,
                                                                                                  ctypes.byref(Times),
                                                                                                  ctypes.byref(
                                                                                                      TimeUnits), 0, 9)
            assert_pico_ok(status["GetValuesTriggerTimeOffsetBulk"])

            # Get and print TriggerInfo for memory segments
            # Create array of ps.PS5000A_TRIGGER_INFO for each memory segment
            Ten_TriggerInfo = (ps.PS5000A_TRIGGER_INFO * 10)()

            status["GetTriggerInfoBulk"] = ps.ps5000aGetTriggerInfoBulk(chandle, ctypes.byref(Ten_TriggerInfo), 0, 9)
            assert_pico_ok(status["GetTriggerInfoBulk"])


            # Converts ADC from channel A to mV
            # 得到A通道的值
            adc2mVChAMax = adc2mV(bufferAMax, chARange, maxADC)
            adc2mVChAMax1 = adc2mV(bufferAMax1, chARange, maxADC)
            adc2mVChAMax2 = adc2mV(bufferAMax2, chARange, maxADC)
            adc2mVChAMax3 = adc2mV(bufferAMax3, chARange, maxADC)
            adc2mVChAMax4 = adc2mV(bufferAMax4, chARange, maxADC)
            adc2mVChAMax5 = adc2mV(bufferAMax5, chARange, maxADC)
            adc2mVChAMax6 = adc2mV(bufferAMax6, chARange, maxADC)
            adc2mVChAMax7 = adc2mV(bufferAMax7, chARange, maxADC)
            adc2mVChAMax8 = adc2mV(bufferAMax8, chARange, maxADC)
            adc2mVChAMax9 = adc2mV(bufferAMax9, chARange, maxADC)


            # Converts ADC from channel B to mV
            # 得到B通道的值
            adc2mVChBMax = adc2mV(bufferBMax, chBRange, maxADC)
            adc2mVChBMax1 = adc2mV(bufferBMax1, chBRange, maxADC)
            adc2mVChBMax2 = adc2mV(bufferBMax2, chBRange, maxADC)
            adc2mVChBMax3 = adc2mV(bufferBMax3, chBRange, maxADC)
            adc2mVChBMax4 = adc2mV(bufferBMax4, chBRange, maxADC)
            adc2mVChBMax5 = adc2mV(bufferBMax5, chBRange, maxADC)
            adc2mVChBMax6 = adc2mV(bufferBMax6, chBRange, maxADC)
            adc2mVChBMax7 = adc2mV(bufferBMax7, chBRange, maxADC)
            adc2mVChBMax8 = adc2mV(bufferBMax8, chBRange, maxADC)
            adc2mVChBMax9 = adc2mV(bufferBMax9, chBRange, maxADC)


            # 保存曲线
            arr[j - 9] = np.array(adc2mVChBMax)
            arr[j - 8] = np.array(adc2mVChBMax1)
            arr[j - 7] = np.array(adc2mVChBMax2)
            arr[j - 6] = np.array(adc2mVChBMax3)
            arr[j - 5] = np.array(adc2mVChBMax4)
            arr[j - 4] = np.array(adc2mVChBMax5)
            arr[j - 3] = np.array(adc2mVChBMax6)
            arr[j - 2] = np.array(adc2mVChBMax7)
            arr[j - 1] = np.array(adc2mVChBMax8)
            arr[j - 0] = np.array(adc2mVChBMax9)


        getBytes = b''
        count = ser.inWaiting()  # 获取串口缓冲区数据
        # print('count = ',count)
        # print("\n")

        while count != 2:
            count = ser.inWaiting()
        # if count !=0 :
        # getBytes = ser.read(ser.in_waiting) # 读出串口数据
        # ser.flushInput()
        getBytes = ser.read(count)  # 读出串口数据
        ser.flushInput()
        ser.flushOutput()



    # 保存曲线
    np.save(r'E:/result/trace_test/arrPart{0}.npy'.format(p), arr)
    # 保存输入数据
    np.save(r'E:/result/trace_test/textInPart{0}.npy'.format(p), text_in)
    # 保存输出数据
    np.save(r'E:/result/trace_test/textOutPart{0}.npy'.format(p), text_out)



# 关闭示波器
# Stops the scope
# Handle = chandle
status["stop"] = ps.ps5000aStop(chandle)
assert_pico_ok(status["stop"])

# Closes the unit
# Handle = chandle
status["close"] = ps.ps5000aCloseUnit(chandle)
assert_pico_ok(status["close"])

# Displays the staus returns
print(status)



```

