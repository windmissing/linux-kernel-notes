block_write() ----- 块设备文件数据的读操作

1.把参数中文件指针pos位置映射成数据块号和块中偏移量

2.将pos所在位置的数据读入到缓冲区的一个缓冲块bread()

3.计算要写的长度

4.从用户数据缓冲区将数据复制到当前缓冲块的位移位开始处（从第2次开始，偏移量都是0）

5.如果还有数据，go to step 2

图：见笔记


用户读写操作过程，以读为例：

![](/assets/7.gif)

再以块设备读函数为例：

block_read() ---> bread() ---> ll_rw_block()