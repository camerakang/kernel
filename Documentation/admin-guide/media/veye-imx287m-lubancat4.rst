.. SPDX-License-Identifier: GPL-2.0

LubanCat-4 双 MV-MIPI-IMX287M 摄像头
==================================

适用范围
--------

本配置针对 LubanCat-4 / LubanCat-4 V1（RK3588S），CAM0 和 CAM1，
以及本 SDK 的 Linux 6.1。不能直接用于 LubanCat-4IO 或 LubanCat-5，
它们的 I2C 总线、GPIO 和部分 CSI 链路不同。

该模组通过 VEYE 固件提供单色图像和触发控制，应使用 ``veye,mvcam``
驱动，不能仅声明一个普通 Sony IMX287 传感器。
驱动来源为 VEYE ``rk35xx_veye_bsp`` 的提交
``e4516077e1fe5d3d7b7844019ed18ab22978540e``，
目录 ``linux/drivers/kernel_v6.1``。
原驱动声明 ``MODULE_LICENSE("GPL v2")``。

本地适配将控制参数和链路频率存储改为每颗摄像头独立保存，
修正可选 GPIO 的错误处理及 endpoint 的查找与 lane 能力检查。

硬件连接
--------

依据 LubanCat4_EBF410116V1R0_SCH_20240221.pdf：

* CAM0：I2C1，DCPHY0，GPIO1_A3 为 CAM0_PWDN / 模组 ENABLE。
* CAM1：I2C5，DCPHY1，GPIO4_A5 为 CAM1_PWDN / 模组 ENABLE。
* 两颗摄像头默认 I2C 地址均为 ``0x3b``；它们在不同总线上，无需改地址。
* 本配置使用两条 CSI-2 data lane，和模组的两 lane 输出对应。
* 模组自带时钟，不使用主板传感器时钟或自动对焦驱动。
  Raspberry Pi 转接路径通常不连接模组 reset，因此没有声明 reset-gpios。

ENABLE 为高有效。必须确认实际转接板将主板 PWDN 接到模组 ENABLE；
若转接板具有反相电路，需调整 GPIO 极性。
模组需要 5 V 供电、3.3 V I2C 与触发电平。
不要把模组供电接在仅提供 3.3 V 的树莓派接口供电引脚上。
原版 ADP-MV1 的 J1 为 15 pin，其第 15 针为 NC，不提供相机电源。
每块 ADP-MV1 的 J3-1 接 5 V，J3-2 接 GND；使用 J1 的 I2C 时，
每块板各自短接 J3 的 11--13（SDA）和 12--14（SCL）。
LubanCat4 的 J9 物理第 2、4 针为 5 V，第 6 针为 GND。

外同步需要额外的物理连接：同一个信号源连接两颗摄像头的 TrigIN，
并将信号源与两颗摄像头共地。原版 ADP-MV1 的输入是 J3-3。
本分支启用 PWM14_M2，将 LubanCat4 J9 物理第 12 针作为共用触发输出，
连接到两块 ADP-MV1 的 J3-3。触发信号为 0--3.3 V 脉冲，不能用固定
3.3 V 电源代替，不能直接输入 5 V、12 V 或 24 V 信号。
PWM 不自动启动；使用下面的工具在两路采集开始后手动启用。

独立采集链路为：

* CAM0 -> csi2_dcphy0 -> mipi0_csi2 -> rkcif_mipi_lvds -> DDR。
* CAM1 -> csi2_dcphy1 -> mipi1_csi2 -> rkcif_mipi_lvds1 -> DDR。

两路 CIF 到 ISP 的 sditf 关闭，避免将单色图像当成 Bayer 图像处理。
不要在相同接口上同时加载 IMX415、OV5647 等其他摄像头插件。

编译与部署
----------

RK3588 内核配置已启用 ``CONFIG_VIDEO_VEYEMVCAM=y``。
根据使用的根文件系统选择原有的 RK3588 Linux 6.1 SDK 配置。
例如 Debian 12 lite：

.. code-block:: bash

   ./build.sh LubanCat_rk3588_debian_linux6.1_lite_defconfig
   ./build.sh kerneldeb

本配置没有更改 U-Boot，也不要求重新刷写引导程序。
按 SDK 原有流程安装新编译的内核包、对应的基础 DTB 和 DTBO，
安装前保留当前可启动的内核及设备树。不要只安装插件而保留旧驱动或旧 DTB。
部署后确认 ``uname -r`` 对应新内核，且使用 LubanCat-4 的实际 DTB。

在板卡 ``/boot/uEnv/uEnv.txt`` 中启用：

.. code-block:: text

   enable_uboot_overlays=1
   dtoverlay=/dtb/overlay/rk3588s-lubancat-4-cam0-cam1-imx287m-overlay.dtbo

本分支的 ``uEnvLubanCat4.txt`` 与 ``uEnvLubanCat4-V1.txt`` 模板已启用双摄插件。
已有系统的 uEnv 文件需手动核对，模板修改不会自动覆盖板卡上的文件。
双摄插件应放在 I2C 禁用插件之后，以重新启用 CAM1 使用的 I2C5。
如果仅调试单路，可改用 ``cam0-imx287m`` 或 ``cam1-imx287m`` 插件，
不要与双摄插件重复加载。其他 CAM0/CAM1 摄像头插件应关闭。

首次验证
--------

重启后先检查驱动和链路，再设置外触发：

.. code-block:: bash

   dmesg | grep -Ei 'mvcam|imx287|csi|rkcif'
   cat /sys/bus/i2c/devices/1-003b/veye_mvcam/camera_model
   cat /sys/bus/i2c/devices/5-003b/veye_mvcam/camera_model
   media-ctl -p -d /dev/media0

两个型号都应显示 ``MV-MIPI-IMX287M``。视频节点和 media 节点的编号不是固定的。
SDK 中的工具位于 ``device/rockchip/common/data/configure-double-imx287m.py``，
将其复制到板卡后使用。它会找到两路摄像头的真实设备节点，
并检查型号和独立 CIF 采集节点；不自动启动采集或产生触发信号。

板卡安装运行依赖：

.. code-block:: bash

   sudo apt install python3 v4l-utils i2c-tools
   sudo python3 configure-double-imx287m.py --mode free --fps 30 --exposure-us 1000

工具设置两路为 720x544、Mono8（``Y8_1X8`` / ``GREY``），手动曝光。
先使用它输出的两个采集命令，在两个终端确认连续采集都能取得图像。
工具的 ``--dry-run`` 只检测设备并打印计划写入的命令。

外同步验证
----------

停止两路采集，关闭外部触发源，然后配置两路：

.. code-block:: bash

   sudo python3 configure-double-imx287m.py --mode hardware --fps 30 \
       --exposure-us 1000 --edge rising

工具设置 ``trigger_mode=1``、``trigger_src=1``、每个有效边沿只采一帧，
关闭触发延迟、曝光延迟和触发滤波，并校验两路实际曝光时间相同。
相机曝光时间可能被量化，因此以读取到的实际值为准。
不要在采集进行中运行该工具；格式和控制设置会失败或影响采集。

先启动两路采集，再启用共用触发信号。无触发时，采集等待新帧是正常行为。
``--fps`` 设置模组的内部时序参数；实际外触发采集帧率由触发源决定。
初次测试从 30 Hz 开始，脉冲间隔需满足实际曝光及读出时间要求。
工具使用普通边沿触发；脉宽控制曝光的 pulse trigger 模式不在此配置范围内。

板载 PWM 触发
------------

``uEnvLubanCat4.txt`` 与 ``uEnvLubanCat4-V1.txt`` 已启用
``rk3588-lubancat-pwm14-m2-overlay.dtbo``。工具按 PWM14 控制器地址
``febf0020`` 查找 sysfs 节点，不依赖 ``pwmchipN`` 的编号。
将 ``device/rockchip/common/data/imx287m-trigger-pwm.py`` 安装为
``/usr/local/bin/imx287m-trigger-pwm``，相机配置工具安装为
``/usr/local/bin/configure-double-imx287m``，然后依次执行：

.. code-block:: bash

   sudo imx287m-trigger-pwm stop
   sudo configure-double-imx287m --mode hardware --fps 30 --exposure-us 1000 --edge rising
   # 按配置工具打印的真实视频节点，在两个终端分别启动采集。
   sudo imx287m-trigger-pwm start --fps 30 --pulse-us 100
   # 结束采集或重新配置前关闭脉冲。
   sudo imx287m-trigger-pwm stop

这里 100 us 是触发脉冲宽度，1000 us 是相机曝光时间，两者独立。
工具默认 30 Hz，初次调试允许 1--100 Hz；更高帧率需结合相机时序和
采集链路实测。配置工具的 ``--fps`` 应与目标触发频率相匹配。

软件依次调用两次 soft_trgone 不能实现严格的双摄曝光同步。
同一个硬件触发沿可以用于曝光同步；两路 V4L2 buffer 时间戳是接收时刻，
并不直接代表曝光开始时刻。验证同步应使用两路 Strobe/TriggerWait 信号和
示波器，检查波形与丢触发计数，并结合有明显运动的同一场景验证图像。
高帧率、RAW10/RAW12、转接板信号质量和实际同步误差都需要实机测量。

参考资料
--------

* https://wiki.veye.cc/index.php/MV-MIPI-IMX287M_Index
* https://www.veye.cc/en/product/mv-mipi-imx287m/
* https://github.com/veyeimaging/rk35xx_veye_bsp
* https://github.com/veyeimaging/rk35xx_LubanCat
* https://wiki.veye.cc/index.php/MV_Camera_Application_Development_Guide_on_Rochchip/zh

本次验证包含驱动编译、基础 DTB/插件编译、双摄及独立插件的离线应用检查。
板卡尚未连接进行实机测试；不能据此认定已经出图或已满足同步精度要求。
