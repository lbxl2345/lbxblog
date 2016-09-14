### SGX
SGX:Software Guard Extensions  
在运行时，程序可以分为几个部分：  
1. Untrusted Run-Time System:在SGX enclave外部执行的部分，负责加载和管理一个enclave，并且Ecall enclave，接受enclave中的Ocall。  
2. Trusted Run-Time System:在SGX enclave内部执行的部分，接受Ecall，进行Ocall，并对enclave自身进行管理。
3. Edge routines：指的是函数边的情况。  
4. 第三方库：为SGX定制的库。

---
ECall：Enclave call，调用enclave当中的函数。  
OCall：Out call，从enclave内部到外部的调用。  
在SGX中，enclave是用来减少信任基的，在运行时不可信域决定了可信域函数的调用顺序，也决定了起内部的上下文；并且ECall和OCall的返回值和参数也是不可信的。

### Enclave文件格式
除了代码段、数据段之外，enclave文件还包含**metadata**,一个untrusted loader需要使用这个metadata来决定这个enclave如何被装载。