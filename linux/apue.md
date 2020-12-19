# Chapter 1 UNIX基础知识
## 1.5 输入与输出
文件描述符通常是一个小的非负整数，kernel uses it to identify files accessed by a process.\
open、read、write、lseek、close提供了不带缓冲的I/O，均使用文件描述符。标准输入输出对应的文件描述符为0和1。\
标准I/O函数为不带缓冲的I/O函数提供了带缓冲的接口，无需自己选取缓冲区大小作为函数参数。