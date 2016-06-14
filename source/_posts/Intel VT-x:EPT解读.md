### 1 VT-x
VT-x是intel针对硬件辅助虚拟化的技术，它解决x86指令集不能被虚拟化的问题，并且简化了VMM软件，减少了软件虚拟化的需求。Virtual Machine Extensions定义了一系列新的操作，称为VMX操作，来提供处理器级别的支持。同时它提供了一个新的特权等级VMX root给VMM，从而避免了ring deprivileging方法(让操作系统运行于ring 1，VMM使用ring 0)带来的虚拟化漏洞。VMX操作可以分为两类：  
root:VMM执行的VMX root 操作  
non-root:Guest执行的VMX non-root操作  
对于这两种模式之间的转换，VMX提供了确切的说法：  
VM Entry:转换到VMX non-root操作  
VM Exit:从VMX non-root操作转换到VMX root操作  
实际上，这个操作过程也就是VMM和虚拟机之间的转换过程。VMCS是一个用来管理VMX non-root操作和VMX转换的数据结构。它由VMM配置，指定guest OS状态，并在VM exits发生时进行控制。  

