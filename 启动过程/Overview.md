# Overview  

> 该部分文档将分析zircon内核的启动过程，即从BIOS将控制权移交给bootloader开始，到正常进入用户态shell的过程。

* 正式进入C++代码(lk_main)之前的故事

* 从lk_main到userboot：进入用户态！

* userboot到shell，有多远？