---
title: React Native开发
---

# 开发环境的搭建

### 安卓模拟器
- 模拟器的启动 主要使用 emulator 的命令。
- emulator -list-avds 查询有哪些模拟器设备
- emlator -avd Nexus_10_API_30 启动模拟器

### 控制调试中的模拟机和真机
- 主要使用adb进行控制。
- adb devices 查询已经连接上的虚拟机或者真机。
- adb shell input keyevent 82 触发连接设备的菜单键 可以进行reload和debug模式。
- adb -s #代表设备的参数 shell input keyevent 82 连接多个设备时指定某个设备触发菜单按钮。
- adb install ***.apk 给设备安装某个包

### 日志
> 主要使用 react_native_debugger。