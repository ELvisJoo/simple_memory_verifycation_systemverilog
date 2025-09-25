V1 is simple version without UVM （stable）  
V2 is comply version with UVM

## 一、框架图如下所示

txgen ----------------------→ input_mon ------------→ scb  

-------------→ dut ---------→ output_mon ------------- ↑


## 二、说明

该V1案例在Questa下跑通，V2案例的cover group部分待完善。  
txgen生成transaction把激励分别发送送给input_mon和dut，  
然后在scb中做数据比对和coverage采集。  

## 三、参考


[Welcome To SystemVerilog Central](https://www.asic-world.com/systemverilog/index.html)

## memory.sv ##
```verilog
module memory(
    address,
    data_in,
    data_out,
    read_write,
    chip_en
    );
     
      input wire [7:0] address, data_in;
      output reg [7:0] data_out;
      input wire read_write, chip_en;
     
      reg [7:0] mem [0:255];
     
      always @ (address or data_in or read_write or chip_en)
      if (read_write == 1 && chip_en == 1) begin
        mem[address] = data_in;
      end
     
      always @ (read_write or chip_en or address)
      if (read_write == 0 && chip_en) 
        data_out = mem[address];
      else
        data_out = 0;
     
endmodule
```
## mem_ports.sv ##
```systemverilog
`ifndef MEM_PORTS_SV
`define MEM_PORTS_SV
interface mem_ports(
    input  wire  clock,
    output logic [7:0] address,
    output logic chip_en,
    output logic read_write,
    output logic [7:0] data_in,
    input logic [7:0] data_out
   );
endinterface
`endif

```
## mem_base_object.sv ##
```systemverilog
`ifndef mem_base_object_sv
`define mem_base_object_sv
class mem_base_object;
    rand bit [7:0] addr;
    rand bit [7:0] data;
    // Read = 0, Write = 1
    rand logic rd_wr;

    // 约束：地址和数据范围可以根据需要调整
    constraint addr_c { addr < 256; }
    constraint data_c { data inside {[0:255]}; }
    constraint rd_wr_c {rd_wr inside {[0:1]};}
    function new( );
      begin
        this.addr = $urandom();
        this.data = $urandom();
        this.rd_wr = $urandom();// this.rd_wr = $urandom()%2;
      end
    endfunction
  endclass
  `endif 
```
## mem_txgen.sv ##
```systemverilog
class mem_txgen;
    mem_base_object  mem_object;
    mem_driver  mem_driver;
    
    integer num_cmds;
   
  function new(virtual mem_ports ports);
    begin
      num_cmds = 5;
      mem_driver = new(ports);
    end
  endfunction
   
   
  task gen_cmds();
    begin
      integer i = 0;
      for (i=0; i < num_cmds; i ++ ) begin
        mem_object = new();
        mem_object.addr = $random();
        mem_object.data = $random();
        // 直接随机化，不手动修改随机结果（保持随机性）
        if (mem_object.randomize()) begin
          $display("随机生成的对象: addr=0x%0h, data=0x%0h, rd_wr=%0b", mem_object.addr, mem_object.data, mem_object.rd_wr);
          mem_object.rd_wr = 1;
          mem_driver.drive_mem(mem_object);
          mem_object.rd_wr = 0;
          mem_driver.drive_mem(mem_object);
          //mem_driver.drive_mem(mem_object);
        end else begin
          $error("随机化失败!");
        end
      end
    end

    // 等待所有事务完成
    repeat (2) @(posedge mem_driver.ports.clock);
    $display("[%0t] 所有事务完成，测试结束", $time);
    $stop;
    $finish;
  endtask
   
  endclass
```
## mem_driver.sv ##
```systemverilog
`include "mem_base_object.sv"
class mem_driver;
  virtual mem_ports ports;
   
  function new(virtual mem_ports ports);
    begin
      this.ports = ports;
      ports.address    = 0;
      ports.chip_en    = 0;
      ports.read_write = 0;
      ports.data_in    = 0;
    end
  endfunction
   
  task drive_mem (mem_base_object object);
    begin
      @ (posedge ports.clock);
      ports.address    = object.addr;
      ports.chip_en    = 1;
      ports.read_write = object.rd_wr;
      ports.data_in    = (object.rd_wr) ? object.data : 0;
      if (object.rd_wr) begin
        $display("Driver : Memory write access-> Address : %x Data : %x\n", 
          object.addr,object.data);
      end else begin
        $display("Driver : Memory read  access-> Address : %x\n", 
          object.addr);
      end
      @ (posedge ports.clock);
      ports.address    = 0;
      ports.chip_en    = 0;
      ports.read_write = 0;
      ports.data_in    = 0;
   end
  endtask
   
  endclass
```
## mem_ip_monitor.sv ##
```systemverilog
class mem_ip_monitor;
    mem_base_object mem_object;
    mem_scoreboard  sb;
    virtual mem_ports ports;
   
  function new (mem_scoreboard sb,virtual mem_ports ports);
    begin  
      this.sb    = sb;
      this.ports = ports;
    end
  endfunction
   
   
  task input_monitor();
    begin
      while (1) begin
        @ (posedge ports.clock);
        if ((ports.chip_en == 1) && (ports.read_write == 1)) begin
          mem_object = new();//创建新的object
          $display("input_monitor : Memory wr access-> Address : %x Data : %x", 
          ports.address,ports.data_in);
          mem_object.addr = ports.address;
          mem_object.data = ports.data_in;
          sb.post_input(mem_object);
        end
      end
    end
  endtask
   
  endclass
```
## mem_op_monitor.sv ##
```systemverilog
class mem_op_monitor;
    mem_base_object mem_object;
    mem_scoreboard sb;
    virtual mem_ports ports;
   
  function new (mem_scoreboard sb,virtual mem_ports ports);
    begin
      this.sb    = sb;
      this.ports = ports;
    end
  endfunction
    
   
  task output_monitor();
    begin
      while (1) begin
        @ (negedge ports.clock);
        if ((ports.chip_en == 1) && (ports.read_write == 0)) begin
          mem_object = new();//创建新的object
          $display("Output_monitor : Memory rd access-> Address : %x Data : %x", 
          ports.address,ports.data_out);
          mem_object.addr = ports.address;
          mem_object.data = ports.data_out;
          sb.post_output(mem_object);
        end
      end
    end
  endtask
   
  endclass
```
## mem_scoreboard.sv ##
```systemverilog
`include "mem_ports.sv"
class mem_scoreboard;
  // Create a keyed list to store the written data
  // Key to the list is address of write access
  mem_base_object mem_object [*];// 存储写入的数据，以地址为键
  virtual mem_ports ports;// 保存接口引用
  mem_base_object current_object;// 保存当前要监控的对象
  

    //  Covergroup: cg
  covergroup cg with function sample(mem_base_object current_object);
    option.per_instance = 1;
    addr: coverpoint current_object.addr[7:0] {
      bins low    = {[0:31]};    // 低地址范围
      bins mid    = {[32:63]};   // 中地址范围
      bins high   = {[64:127]};  // 高地址范围
    }
    data: coverpoint current_object.data[7:0] {
      bins d1    = {[0:31]};    // 低数据范围
      bins d2    = {[32:63]};   // 中数据范围
      bins d3   = {[64:127]};  // 高数据范围
    }
    rd_wr: coverpoint current_object.rd_wr{
      bins read    = {0};    // 读
      bins write   = {1};    // 写
    }
    //  Cross: cx_cross_identifier
    cross addr,data,rd_wr;
  endgroup: cg

  // 构造函数，接收接口引用
  function new(virtual mem_ports p);
    ports = p;
    // 初始化覆盖率组，传入接口
    cg = new();
  endfunction

  // post_input method is used for storing write data
  // at write address
  task post_input (mem_base_object  input_object);
    begin
      mem_object[input_object.addr] = input_object;
    end
  endtask

  // post_output method is used by the output monitor to 
  // compare the output of memory with expected data
  task post_output (mem_base_object  output_object);
    // 将输出对象保存到当前对象，供覆盖率收集使用
    current_object = output_object;
    cg.sample(output_object);
      // Check if address exists in scoreboard  
    if (mem_object[output_object.addr] != null) begin 
        mem_base_object  in_mem = mem_object[output_object.addr];
        $display("scoreboard : Found Address %x in list",output_object.addr);
        
        if (output_object.data != in_mem.data)  begin
          $display("Scoreboard : Error : Exp data and Got data don't match");
          $display("             Expected -> %x",in_mem.data);
          $display("             Got      -> %x",output_object.data);
        end else begin
          $display("Scoreboard : Exp data and Got data match");
        end
    end 
  endtask
endclass

```
## test.sv ##
```systemverilog

`include "mem_ports.sv"
 
program memory_top(mem_ports ports);
`include "mem_base_object.sv"
`include "mem_driver.sv"
`include "mem_txgen.sv"
`include "mem_scoreboard.sv"
`include "mem_ip_monitor.sv"
`include "mem_op_monitor.sv"

  mem_base_object mem_object;
  mem_txgen txgen;
  mem_scoreboard sb;
  mem_ip_monitor ipm;
  mem_op_monitor opm;
 
initial begin
  mem_object = new();
  sb    = new(ports);
  ipm   = new (sb, ports);
  opm   = new (sb, ports);
  txgen = new(ports);
  fork
    ipm.input_monitor();
    opm.output_monitor();
  join_none
  txgen.gen_cmds();
  repeat (20) @ (posedge ports.clock);
end
 
endprogram
```
## testbench(top.sv) ##
```systemverilog
`timescale 1ns/1ns

`include "memory.sv"
`include "mem_ports.sv"

module memory_tb();
 
wire [7:0] address, data_in;
wire [7:0] data_out;
wire  read_write, chip_en;
reg clk;
 
// Connect the interface
mem_ports ports(
 .clock       (clk),
 .address     (address),
 .chip_en     (chip_en),
 .read_write  (read_write),
 .data_in     (data_in),
 .data_out    (data_out)
);

memory U_memory(
.address             (address),
.data_in             (data_in),
.data_out            (data_out),
.read_write          (read_write),
.chip_en             (chip_en)
);

// Connect the program
memory_top top (ports);
 
initial begin
  clk = 1;
end	
always #1 clk = ~clk;

// 在仿真开始时创建WLF波形文件并添加信号
// initial begin
//   // 创建波形文件
//   $fsdbdumpfile("waveform.fsdb");
  
//   // 添加所有层次的信号（*表示所有信号）
//   $fsdbumpvars(0, top);  // top_tb是你的testbench顶层模块名
//   // 可选：只添加特定模块的信号
//   // $wlfdumpvars(1, top_tb.dut);  // 只添加DUT模块的信号（1表示深度为1）
//   // $wlfdumpvars(2, top_tb.driver);  // 添加driver模块及其下一级子模块的信号
// end

endmodule
```
