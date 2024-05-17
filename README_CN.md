## RT-Thread Wi-Fi Host Driver (WHD)

[English](./README.md)

### 概述
WHD是一个独立的嵌入式Wi-Fi主机驱动程序，它提供了一组与英飞凌WLAN芯片交互的api。WHD是一个独立的固件产品，可以很容易地移植到任何嵌入式软件环境，包括流行的物联网框架，如Mbed OS和Amazon FreeRTOS。因此，WHD包含了RTOS和TCP/IP网络抽象层的钩子。

有关Wi-Fi主机驱动程序的详细信息可在[Wi-Fi Host Driver readme](./wifi-host-driver/README.md)文件中找到。<br>
[release notes](./wifi-host-driver/RELEASE.md)详细说明了当前的版本。您还可以找到有关以前版本的信息。

该存储库已将WHD适应于RT-Thread系统，目前仅支持SDIO总线协议，并使用RT-Thread的mmcsd进行SDIO总线操作。<br>
欢迎大家`PR`支持更多总线接口和芯片。

### 使用

- 将该仓库克隆到RT-Thread项目中的`packages`或`libraries`目录。
- 因为`wifi-host-driver`是一个子模块，所以需要使用`-recursive`选项进行克隆。
- 在RT-Thread项目的`libraries`或`packages`文件夹中，将`WHD`的`Kconfig`文件包含在其Kconfig文件中。
- 例如，在`libraries`目录中包含`WHD`:
```Kconfig
menu "External Libraries"
    source "$RTT_DIR/../libraries/rt-thread_wifi-host-driver/Kconfig"
endmenu
```
**注意:**<br>
SDIO驱动需要支持数据流传输，在RT-Thread的bsp中，大多数芯片都未适配数据流传输的功能。<br>
`Cortex-M4`内核需要软件来计算`CRC16`并在数据后面发送它，参考 [字节流传输解决方案](http://t.csdnimg.cn/pL1KD)。<br>
对于`Cortex-M7`内核，只需要修改`drv_sdio.c`文件的一处地方即可，示例如下: <br>
```c
/* 该示例是STM32H750的SDIO驱动程序 */
SCB_CleanInvalidateDCache();

reg_cmd |= SDMMC_CMD_CMDTRANS;
hw_sdio->mask &= ~(SDMMC_MASK_CMDRENDIE | SDMMC_MASK_CMDSENTIE);
hw_sdio->dtimer = HW_SDIO_DATATIMEOUT;
hw_sdio->dlen = data->blks * data->blksize;
hw_sdio->dctrl = (get_order(data->blksize)<<4) |
                    (data->flags & DATA_DIR_READ ? SDMMC_DCTRL_DTDIR : 0) | \
                    /* 增加数据流标志的检测 */
                    ((data->flags & DATA_STREAM) ? SDMMC_DCTRL_DTMODE_0 : 0);
hw_sdio->idmabase0r = (rt_uint32_t)sdio->cache_buf;
hw_sdio->idmatrlr = SDMMC_IDMA_IDMAEN;
```

### Menuconfig
```
--- Using Wifi-Host-Driver(WHD)
      Select Chips (CYWL6208(cyw43438))  --->           # 选择相应的模块/芯片
[*]   Use resources in external storage(FAL)  --->      # 使用FAL组件加载资源
[ ]   Default enable powersave mode                     # 默认启用低功耗模式
(8)   The priority level value of WHD thread            # 配置WHD线程的优先级
(5120) The stack size for WHD thread                    # 配置WHD线程的堆栈大小
(49)  Set the WiFi_REG ON pin                           # 设置模块的WL_REG_ON引脚
(37)  Set the HOST_WAKE_IRQ pin                         # 设置模块的HOST_WAKE_IRQ引脚
      Select HOST_WAKE_IRQ event type (falling)  --->   # 选择“唤醒主机”的边沿
(2)   Set the interrput priority for HOST_WAKE_IRQ pin  # 设置外部中断优先级
[ ]   Using thread initialization                       # 创建一个线程来初始化驱动
(500) Set the waiting time for mmcsd card scanning      # 设置mmcsd设备驱动扫卡的等待时间
```

### 资源下载
可以通过`ymodem`协议下载资源文件。驱动会使用FAL组件来加载资源文件。<br>
资源下载功能依赖于`ymodem`组件，请确保打开`RT_USING_RYM`和`WHD_RESOURCES_IN_EXTERNAL_STORAGE`宏定义。<br>
- 在终端上执行`whd_res_download`命令开始下载资源。
- 该命令需要输入资源文件的分区名。
- 下载资源文件的实例(使用默认分区名，输入自己的分区名):
```shell
# 例如我的分区配置
/* partition table */
/*      magic_word          partition name      flash name          offset          size            reserved        */
#define FAL_PART_TABLE                                                                                              \
{                                                                                                                   \
    { FAL_PART_MAGIC_WORD,  "whd_firmware",     "onchip_flash",     0,              448 * 1024,         0 },        \
    { FAL_PART_MAGIC_WORD,  "whd_clm",          "onchip_flash",     448 * 1024,     32 * 1024,          0 },        \
    { FAL_PART_MAGIC_WORD,  "easyflash",        "onchip_flash",     480 * 1024,     32 * 1024,          0 },        \
    { FAL_PART_MAGIC_WORD,  "filesystem",       "onchip_flash",     512 * 1024,     512 * 1024,         0 },        \
}

# 下载固件文件
whd_res_download whd_firmware

# 下载clm文件
whd_res_download whd_clm
```
- `ymodem`可以使用`xshell`工具，在完成命令输入后，等待`xshell`启动文件传输。
```
msh >whd_res_download whd_firmware
Please select the whd_firmware file and use Ymodem to send.
```
- 此时，在`xshell`中右键单击鼠标，选择`文件传输`到`使用ymodem发送`。
- 在`whd`的`resources(wifi-host-driver/WiFi_Host_Driver/resources)`目录下，选择对应芯片的资源文件。
- 传输完成后，msh将输出如下日志：
```
Download whd_firmware to flash success. file size: 419799
```
- 下载完固件和clm资源文件后，复位重启即可正常加载资源文件。

### 芯片支持

| **CHIP**  |**SDIO**|**SPI**|**M2M**|
|-----------|--------|-------|-------|
| CYW4343W  |   *    |   x   |   x   |
| CYW43438  |   o    |   x   |   x   |
| CYW4373   |   *    |   x   |   x   |
| CYW43012  |   o    |   x   |   x   |
| CYW43439  |   *    |   x   |   x   |
| CYW43022  |   *    |   x   |   x   |

'x' 表示不支持<br>
'o' 表示已测试和支持<br>
'*' 理论上支持，但未经过测试

### 更多信息
* [Wi-Fi Host Driver API Reference Manual and Porting Guide](https://infineon.github.io/wifi-host-driver/html/index.html)
* [Wi-Fi Host Driver Release Notes](./wifi-host-driver/RELEASE.md)
* [Infineon Technologies](http://www.infineon.com)