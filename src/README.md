# SPDRW

根据开源项目 https://github.com/1a2m3/SPD-Reader-Writer 做的配套硬件；利用Arduino对I2C总线进行读写。


内存支持：

 - DDR4-UDIMM
 - DDR4-SODIMM 
 - DDR5-UDIMM
 - DDR5-SODIMM

功能支持：

 - SPD读取并支持保存为bin文件。
 - SPD写入，支持写入bin、spd、thp文件。
 - 设置写保护，防止数据篡改。
 - 解除写保护，写入自定义配置，进行超频。

产品展示：

![DDR4-SODIMM-RW](/img/ddr4-sodimm.png)
![DDR5-SODIMM-RW](/img/ddr5-sodimm.png)
