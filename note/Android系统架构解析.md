想要了解Android平台架构，那么就避免不了一张图片，作为安卓开发都必须见过的一张图片，如下

![Android架构图](https://github.com/MickeyQiong/ANote/blob/main/picture/android-stack_2x.png)

Android是一种基于Linux的开放源代码软件栈，为各类设备和机型而创建。下面我们来解析一下上面图片中的组件。

## Linux Kernel（Linux 内核）

Android平台的基础是Linux内核。比如ART虚拟机最终调用底层Linux内核来执行功能。Linux内核的安全机制为Android提供相应的保障，也允许设备制造商为内核开发硬件驱动程序。

## HAL（硬件抽象层）

硬件抽象层提供标准界面，向更高级别的Java API 框架显示设备硬件功能。HAL 包含多个库模块，其中每个模块都为特定类型的硬件组件实现一个界面，例如相机或蓝牙模块。当框架 API 要求访问设备硬件时，Android 系统将为该硬件组件加载库模块。

## Android Runtime

持续更新···