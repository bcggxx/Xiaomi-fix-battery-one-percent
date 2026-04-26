# English

## Background

Some Xiaomi devices with PM8150 (qcom gen4) power management IC, mainly Snapdragon 865/870 devices including Redmi K30Pro/POCO F2 Pro, K30S Ultra, K40, Xiaomi 10. (Xiaomi 10 Pro is not included because it uses BQ27Z561 PMIC), there is a chance the battery level may stuck at 1%, even charging(actually it can be charged, but still displaying 1%) or reboot doesn't recover it. 

If you run into this situation, the only way to escape is trying to "reset" the PMIC, one way is just to let the battery drain, so the motherboard and PMIC would be completely powered off. Another way is by pressing the Volume Down & Power button for 60s+, power cycle into FASTBOOT around 10+ times, then the PMIC could be reset.

## What caused this

The qualcomm's PMIC driver `qpnp-fg-gen4.c`, it has a strategy called "rapid_soc_dec", when the battery is under-voltage(usually caused by an aging battery or/and low temperature with high current and low battery level), it will immediately report the battery level (aka. `soc` in the code) to 0% for asking the upper-level software to power off the system. (Actually that is bad, this is why some other phones sometimes suddenly power off when the battery is low or in a low-temperature environment). **There is no chance of exiting this state because the original design logic is powering off the device! This is the root reason for this "bug"!!**

However, Xiaomi Implemented a "smooth strategy" in the driver, that can let the battery level change "smoothly" without suddenly jumping to a very high or low value. So when it has entered "rapid_soc_dec" state, it will slowly and smoothly move to 1% (not reporting 0% because there are some other conditions that need to be met), then always stuck there, not powering off, and no chance of exiting this "rapid_soc_dec" state. Charging the battery is also not able to exit this state.

Even, rebooting/power cycling also doesn't work, because for entering "rapid_soc_dec" state, the parameters have been written to the PMIC register, so it reports 0% `soc` from the hardware, not the kernel/software thing. Even there is a "fg_gen4_shutdown" callback that seems to try to restore the value, but I don't know why it doesn't work on many devices, at least not my device. Some people say it can be solved by rebooting/power cycling the device, which means this callback working on these devices.

## The fix
Quite simple, just add an exit strategy of "rapid_soc_dec" state. Checking the battery voltage when it > 3700mV, then exit this state. 

Just check this commit, it is the code of the fix: [链接](https://github.com/liyafe1997/Xiaomi-fix-battery-one-percent/commit/83cb4c684d0a483e8c2c39f6ae80be428b855d25)

Why did I choose the value 3700mV? I don't know exactly, It just seems for a battery that is at a relatively low charge (Like 20+%), still easy to go back to >3700mV when in a low load(current). If you connect to a charger surely it should go back to >3700.

This fix just makes sure 1. The battery/kernel accidentally entered that state, in a "enough charge level" like 20+ present. 2. At least it can escape this state when you charge the battery. If your battery voltage is really low and can not go back to 3700, it is better to just stay in this "rapid_soc_dec" state, reports 1% to you for asking you to charge the battery as soon as possible :)

# About this repository
This repository is forked from `lmi`(Redmi K30 Pro / POCO F2 Pro)'s [official kernel](https://github.com/MiCode/Xiaomi_Kernel_OpenSource/tree/lmi-q-oss), 和 I just committed this [commit](https://github.com/liyafe1997/Xiaomi-fix-battery-one-percent/commit/83cb4c684d0a483e8c2c39f6ae80be428b855d25) for showing how does the fix works

If you are building a third-party kernel for these Xiaomi devices(not only `lmi`) you can refer to this commit to fix this problem. Highly recommend you do that for your kernel to avoid this annoying 1% battery bug for the end-users.

# 简体中文

## 背景

一些使用PM8150（qcom gen4）电源管理IC的小米设备，主要是骁龙865/870设备，包括Redmi K30Pro/POCO F2 Pro、K30S Ultra、K40、小米10。（小米10 Pro不包括在内，因为它使用BQ27Z561 PMIC），电池电量可能会卡在1%，即使充电（实际上可以充电，但仍然显示1%）或重启也无法恢复。

如果您遇到这种情况，唯一的解决方法是尝试"重置"PMIC，一种方法是让电池完全耗尽，这样主板和PMIC将完全断电。另一种方法是按住音量减和电源按钮60秒以上，进入FASTBOOT模式循环10多次，然后PMIC可以被重置。

## 原因

高通的PMIC驱动程序`qpnp-fg-gen4.c`，它有一个名为"rapid_soc_dec"的策略，当电池欠压（通常由老化电池和/或低温高电流和低电量引起）时，它会立即将电池电量（即代码中的`soc`）报告为0%，以要求上层软件关闭系统。（实际上这是不好的，这就是为什么其他一些手机在电池电量低或低温环境下有时会突然关机）。**没有机会退出此状态，因为原始设计逻辑是关闭设备！这是此"错误"的根本原因！！**

然而，小米在驱动程序中实现了一个"平滑策略"，可以让电池电量"平滑"变化，而不会突然跳到非常高的或低的值。因此，当它进入"rapid_soc_dec"状态时，它会缓慢而平滑地移动到1%（不报告0%，因为需要满足其他一些条件），然后一直卡在那里，不关机，也没有机会退出这个"rapid_soc_dec"状态。给电池充电也无法退出此状态。

甚至，重启/电源循环也不起作用，因为进入"rapid_soc_dec"状态时，参数已经写入PMIC寄存器，所以它从硬件报告0% `soc`，而不是内核/软件的问题。即使有一个"fg_gen4_shutdown"回调似乎试图恢复值，但我不知道为什么它在许多设备上不起作用，至少在我的设备上不起作用。有些人说可以通过重启/电源循环设备来解决，这意味着此回调在这些设备上起作用。

## 修复方法
相当简单，只需添加"rapid_soc_dec"状态的退出策略。当电池电压> 3700mV时进行检查，然后退出此状态。

只需查看此提交，它是修复的代码: [链接](https://github.com/liyafe1997/Xiaomi-fix-battery-one-percent/commit/83cb4c684d0a483e8c2c39f6ae80be428b855d25)

为什么我选择3700mV这个值？我不太确定，只是对于一个电量相对较低（如20+%）的电池，在低负载（电流）下仍然很容易回到>3700mV。如果您连接充电器，它肯定应该回到>3700。

此修复只是确保1. 电池/内核意外进入该状态时，处于"足够电量水平"（如20+%）。2. 至少在您给电池充电时可以退出此状态。如果您的电池电压确实很低且无法回到3700，最好保持在"rapid_soc_dec"状态，向您报告1%以要求您尽快给电池充电 :)

# 关于此仓库
此仓库是从`lmi`（Redmi K30 Pro / POCO F2 Pro）的[官方内核](https://github.com/MiCode/Xiaomi_Kernel_OpenSource/tree/lmi-q-oss)fork的，我只是提交了这个[提交](https://github.com/liyafe1997/Xiaomi-fix-battery-one-percent/commit/83cb4c684d0a483e8c2c39f6ae80be428b855d25)以展示修复的工作原理。

如果您正在为这些小米设备（不仅限于`lmi`）构建第三方内核，您可以参考此提交来修复此问题。强烈建议您为您的内核这样做，以避免用户遇到这个烦人的1%电池错误。
