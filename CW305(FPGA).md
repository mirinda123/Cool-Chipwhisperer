# 在CW305(FPGA)上进行曲线采集

BLOCK模式进行曲线采集



```python
import ctypes
import numpy as np
import time as tt
from picosdk.ps5000a import ps5000a as ps
import matplotlib.pyplot as plt
from picosdk.functions import adc2mV, assert_pico_ok, mV2adc
import serial  # 导入模块
from tqdm import tqdm, trange

# Create chandle and status ready for use
chandle = ctypes.c_int16()
status = {}

# Open 5000 series PicoScope
# Resolution set to 12 Bit
resolution = ps.PS5000A_DEVICE_RESOLUTION["PS5000A_DR_12BIT"]
# Returns handle to chandle for use in future API functions
status["openunit"] = ps.ps5000aOpenUnit(ctypes.byref(chandle), None, resolution)

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

# Set up channel A
# handle = chandle
channel = ps.PS5000A_CHANNEL["PS5000A_CHANNEL_A"]
# enabled = 1
coupling_type = ps.PS5000A_COUPLING["PS5000A_DC"]
chARange = ps.PS5000A_RANGE["PS5000A_2V"]
# analogue offset = 0 V
status["setChA"] = ps.ps5000aSetChannel(chandle, channel, 1, coupling_type, chARange, 0)
assert_pico_ok(status["setChA"])

# Set up channel B
# handle = chandle
channel = ps.PS5000A_CHANNEL["PS5000A_CHANNEL_B"]
# enabled = 1
# coupling_type = ps.PS5000A_COUPLING["PS5000A_DC"]
chBRange = ps.PS5000A_RANGE["PS5000A_50MV"]
# analogue offset = 0 V
status["setChB"] = ps.ps5000aSetChannel(chandle, channel, 1, coupling_type, chBRange, 0)
assert_pico_ok(status["setChB"])

# find maximum ADC count value
# handle = chandle
# pointer to value = ctypes.byref(maxADC)
maxADC = ctypes.c_int16()
status["maximumValue"] = ps.ps5000aMaximumValue(chandle, ctypes.byref(maxADC))
assert_pico_ok(status["maximumValue"])

# Set up single trigger
# handle = chandle
# enabled = 1
source = ps.PS5000A_CHANNEL["PS5000A_CHANNEL_A"]
threshold = int(mV2adc(1000, chARange, maxADC))
# direction = PS5000A_RISING = 2
# delay = 0 s
# auto Trigger = 1000 ms
status["trigger"] = ps.ps5000aSetSimpleTrigger(chandle, 1, source, threshold, 2, 0, 10000)
assert_pico_ok(status["trigger"])

# Set number of pre and post trigger samples to be collected
preTriggerSamples = 150
postTriggerSamples = 150
maxSamples = preTriggerSamples + postTriggerSamples

# Get timebase information
# handle = chandle
timebase = 2
# noSamples = maxSamples
# pointer to timeIntervalNanoseconds = ctypes.byref(timeIntervalns)
# pointer to maxSamples = ctypes.byref(returnedMaxSamples)
# segment index = 0
timeIntervalns = ctypes.c_float()
returnedMaxSamples = ctypes.c_int32()
status["getTimebase2"] = ps.ps5000aGetTimebase2(chandle, timebase, maxSamples, ctypes.byref(timeIntervalns),
                                                ctypes.byref(returnedMaxSamples), 0)
assert_pico_ok(status["getTimebase2"])

portx = "COM7"

bps = 115200
# 超时设置,None：永远等待操作，0为立即返回请求结果，其他值为等待超时时间(单位为秒）
timex = 10
# 打开串口，并得到串口对象
ser = serial.Serial(portx, bps, timeout=timex)
ser.flushInput()  # 清空缓冲区

# zongshu是总共采集的条数
# numEachFile是每个文件采集的条数
# zongshu = 200000
# numEachFile = 10000

all_traces = 100000
traces_each_file = 10000

# all_traces / traces_each_file
for part in range(0, all_traces // traces_each_file):
    counter = 0

    # 用来存放采集到的曲线
    traces_container = np.empty((0, preTriggerSamples + postTriggerSamples), float)

    for j in trange(0, traces_each_file):
        status["runBlock"] = ps.ps5000aRunBlock(chandle, preTriggerSamples, postTriggerSamples, timebase, None, 0, None,
                                                None)
        assert_pico_ok(status["runBlock"])

        if counter % 2 == 0:
            ser.write(bytes([1, 2, 3]))
        if counter % 2 == 1:
            ser.write(bytes([4, 5, 6]))

        counter += 1

        # Check for data collection to finish using ps5000aIsReady
        ready = ctypes.c_int16(0)
        check = ctypes.c_int16(0)
        while ready.value == check.value:
            status["isReady"] = ps.ps5000aIsReady(chandle, ctypes.byref(ready))

        # Create buffers ready for assigning pointers for data collection
        bufferAMax = (ctypes.c_int16 * maxSamples)()
        bufferAMin = (ctypes.c_int16 * maxSamples)()  # used for downsampling which isn't in the scope of this example
        bufferBMax = (ctypes.c_int16 * maxSamples)()
        bufferBMin = (ctypes.c_int16 * maxSamples)()  # used for downsampling which isn't in the scope of this example
        source = ps.PS5000A_CHANNEL["PS5000A_CHANNEL_A"]
        status["setDataBuffersA"] = ps.ps5000aSetDataBuffers(chandle, source, ctypes.byref(bufferAMax),
                                                             ctypes.byref(bufferAMin), maxSamples, 0, 0)
        assert_pico_ok(status["setDataBuffersA"])
        source = ps.PS5000A_CHANNEL["PS5000A_CHANNEL_B"]
        status["setDataBuffersB"] = ps.ps5000aSetDataBuffers(chandle, source, ctypes.byref(bufferBMax),
                                                             ctypes.byref(bufferBMin), maxSamples, 0, 0)
        assert_pico_ok(status["setDataBuffersB"])
        overflow = ctypes.c_int16()
        cmaxSamples = ctypes.c_int32(maxSamples)
        status["getValues"] = ps.ps5000aGetValues(chandle, 0, ctypes.byref(cmaxSamples), 0, 0, 0,
                                                  ctypes.byref(overflow))
        assert_pico_ok(status["getValues"])
        adc2mVChAMax = adc2mV(bufferAMax, chARange, maxADC)
        adc2mVChBMax = adc2mV(bufferBMax, chBRange, maxADC)
        # Create time data
        time = np.linspace(0, (cmaxSamples.value - 1) * timeIntervalns.value, cmaxSamples.value)

        traces_container[j] = np.array(adc2mVChBMax)
        count = ser.inWaiting()
        getBytes = ser.read(count)  # 读出串口数据
        ser.flushInput()
        ser.flushOutput()

    np.save(r'E:/result/traces/tracesPart{0}.npy'.format(part), traces_container)
    del traces_container

ser.close()  # 关闭串口
# Stop the scope
# handle = chandle
status["stop"] = ps.ps5000aStop(chandle)
assert_pico_ok(status["stop"])

# Close unit Disconnect the scope
# handle = chandle
status["close"] = ps.ps5000aCloseUnit(chandle)
assert_pico_ok(status["close"])

# display status returns
print(status)

```

