相信你经过基础阶段的学习，已经对虚拟机技术有了一定的认识，这个阶段我们将加入更多的实操内容，学习如何使用 QEMU 进行设备建模，并且掌握常见的 QEMU 虚拟化实践方法。

本阶段，我们将会讨论：

- QEMU 初始化流程：硬件模型的实例化流程、客户机程序加载流程

- QEMU 加速器介绍：以软件二进制翻译加速器 TCG 为主，以硬件虚拟化 KVM 技术为辅

- QEMU 体系结构模拟：以 RISC-V 为例，讲解 TCG 如何模拟指令

- QEMU 硬件建模：Machine 建模、CPU 建模、外设建模、中断建模、Rust 建模等等

- QEMU 调试：介绍常用调试手段，比如 gdbstub，log，trace-event

- QEMU 测试：介绍 QEMU 测试框架 Qtest，以及其他测试框架（float-test、tcg-test）

- QEMU 虚拟化实践：QEMU 常见参数配置、如何创建虚拟机等等。


