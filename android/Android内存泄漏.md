# Android内存泄漏

## 判断是否有内存泄漏方法
### 粗略判断
* 界面工具:Android Monitor-->System Information-->MemoryUsage查看Objects里面是否有没有被回收的activities或者Views。
* 命令行: adb shell dumpsys meminfo