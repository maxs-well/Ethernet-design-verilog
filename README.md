千兆以太网芯片RTL8211Verilog程序
========

本程序由多个verilog文件组成，包括了hdl文件夹里所有的相关文件以及一些必要的IP核。\
文件功能介绍\
`crc.v`:主要在udp发送时使用，计算udp的包的CRC数值。\
`ethernet.v`:对udp_send.v和udp_receive.v文件的封装。\
`ethernet_top.v`:整个工程的示例顶层文件，只是展示如何调用文件和其他相关的ip核文件。\
`udp_receive.v`:负责udp的接收处理过程。\
`udp_send.v`:负责udp的发送处理过程。

文件讲解
=====
udp_receive.v\
端口：
------
`clk          `:接收模块的主时钟。\
`rst_n        `:接收模块的全局复位信号。\
`rx_data_len  `:接收数据的udp数据包长度，是udp报头的参数。\
`rx_total_len `:接收数据的ip数据包长度，是ip数据报头的参数。\
`update       `:当接收到新的udp数据包时，该值会拉高一周期。\
`ip_header    `:接收的整个IP数据包头，仅支持160位的IP数据报头。\
`udp_header   `:接收的整个UDP数据报头，仅支持64位的UDP数据报头。\
`mac          `: 接收的整个MAC数据包，包括目的MAC，源MAC和IP包类型标识。\
`data_o       `:接收的有效数据。\
`src_mac      `:数据包的源MAC地址。\
`src_addr     `:数据包的源IP地址。\
`src_port     `:数据包的源端口地址。\
`DF           `:IP数据包的参数，为0代表数据包可分片，为1代表不可分片。\
`MF           `:IP数据包的参数，为1代表还有分片，为0代表这是最后一片。\
`e_rxdv       `:RTL8211的控制IO，当为高时代表接收数据有效。\
`rxd          `:RTL8211的接收数据IO。

常数定义：
-------
`CAN_RECEIVE_BROADCAST`:是否支持接收广播数据包，也就是MAC地址为FFFFFFFFFFFF的数据包。\
`DST_ADDR`:设置目的地址的IP地址，也就是RTL8211的网卡地址。\
`DST_PORT`:设置目的地址的端口地址，也就是RTL8211的端口。\
`DST_MAC`:设置目的地址的MAC地址，也就是RTL8211的MAC地址。

状态机状态跳转设置：
--------
`IDLE`：空闲状态，只要接收到前导码，就进入R_PRE状态。\
`R_PRE`:接收前导码，接收到7个前导码和一个开始位数据后，进入R_MAC状态。\
`R_MAC`:接收MAC数据状态，接收完毕后进入R_HEADER状态。\
`R_HEADER`:接收IP数据报头和UDP数据报头，接收完毕进入R_DATA状态。\
`R_DATA`:接收UDP有效数据，接收完毕后进入R_FINISH。\
`R_FINISH`:返回IDLE状态。

udp_send.v\
端口：
------
`clk`,`rst_n`信号如上；\
`data_i`:需要发送出去的数据，在tx_dv为高时将需要发送的数据通过该端口传输到UDP数据包。\
`tx_data_len`:发送的数据长度，UDP数据包的参数。\
`crc`:整个以太网数据包的CRC校验码，留作数据校验使用。\
`crcen`:crc模块的使能信号，为高crc校验模块才可以计算。\
`crcrst`:crc模块的复位信号，为高crc检验模块复位。\
`start`:UDP发送模块的开始信号，为高驱动模块开始组帧发送。\
`busy`:UDP发送模块的忙碌信号，当模块正在组帧发送的过程中时该信号拉高。\
`tx_dv`:UDP发送模块的指示信号，告知当前UDP模块已经可以将数据发送出去，请将需要发送的数据发过来。\
`dst_mac`:目的地址的MAC地址。\
`dst_addr`:目的地址的IP地址。\
`dst_port`:目的地址的端口地址。\
`DF`,`MF`如上所述。\
`tx_en`:RTL8211的发送使能信号，为高发送数据有效。\
`txer`:RTL8211的发送错误信号。\
`txd`:RTL8211的发送数据IO。

常数定义：
------
`IP_HEADER_LEN`:ip数据包的长度，默认就好，不需要改。\
`TTL`:IP数据包的参数，生存时间。\
`SRC_ADDR`:源地址的IP地址。\
`SRC_PORT`:源地址的端口。\
`SRC_MAC`:源地址的MAC地址。

状态机状态跳转设置：
--------
`IDLE`:空闲状态，start为高进入MAKE_IP状态。\
`MAKE_IP`:生成IP数据包，进入MAKE_SUM状态。\
`MAKE_SUM`:计算IP数据包的首部校验和，计算完毕进入SEND_PRE状态。\
`SEND_PRE`:发送7个前导码和1个开始码。然后进入SEND_MAC状态。\
`SEND_MAC`:发送目的MAC，源MAC和IP数据包类型，然后进入SEND_HEADER状态。\
`SEND_HEADER`:发送IP数据报头和UDP数据报头，然后进入SEND_DATA状态。\
`SEND_DATA`:发送有效数据，发送完毕进入SEND_CRC状态。\
`SEND_CRC`:发送CRC数据，发送完毕进入IDLE_CODE状态。\
`IDLE_CODE`:发送12个空闲码，以太网数据包的要求。发送完毕回到IDLE状态


为了满足125Mhz的时序要求，这两个核心部分的代码尽力做了一些修改，比如udp_send中，发送如MAC，数据报头时，都采用了状态机根据计数器的值发送对应位的数据，
之前曾经试过直接发高八位，然后通过不断地移位将没有发送过得数据移到高八位，但这个方法无法满足125Mhz的时序要求，所以改成了现在这个方式。又比如计算首部检验和，
将计算的过程分成了几步来计算，也有效的减少了时序违例的总数。\
但是即使如此，仍有部分信号不满足时序要求，时序报告会产生负1ns左右的时序路径。

未完待续，等着填坑。。。

