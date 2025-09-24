V1 is simple version without UVM （stable）
V2 is comply version with UVM

## 一、框架图如下所示

![image-20250925003828733](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20250925003828733.png)

## 二、说明

该V1案例在Questa下跑通，V2案例的cover group待完善。
txgen生成transaction把激励分别发送送给input_mon和dut，然后在scb中做数据比对和coverage采集。

## 三、参考

[Welcome To SystemVerilog Central](https://www.asic-world.com/systemverilog/index.html)