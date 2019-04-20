基于TI CC2650代码分析

## 文档工具下载

* 文档下载位置：

http://www.ti.com.cn/tool/cn/launchxl-cc2650

* SDK下载位置：

http://www.ti.com.cn/tool/cn/ble-stack

* 下载CCS

http://www.ti.com.cn/tool/cn/ccstudio-wcs

## 环境搭建

* SDK安装

安装完SDK之后，在SDK的安装位置会有文档。位于docs目录。

* CCS安装

SWRU393_CC2640_BLE_Software_Developer's_Guide.pdf

参考文档2.6.3 Code Composer Studio, 安装CCS.

安装完成之后需要选择交叉工具编译版本。这个是根据SDK来选择的。

查看SDK/release_notes_BLE_Stack_2_2_2.html，选择对应的交叉编译版本。

选择错误，编译之后，烧录到板子上，程序不能正常运行。

针对LaunchPad的板子，应该打开后缀是lp的工程。

另外一个板子，由于封装不同，使用后缀是em的工程。

## 源码分析

[GAP初始化](./GAP初始化.md)

GAP主要设置广播和连接相关参数，例如广播时间间隔等。

[GATT初始化](./GATT初始化.md)

GATT设置连接以后的service，attribute等参数。