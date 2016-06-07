#### df命令:获取磁盘使用情况
![df](https://raw.githubusercontent.com/lbxl2345/blogbackup/master/source/pics/%E7%A3%81%E7%9B%98IO/df.png)
在这里可以看到虚拟机中的磁盘是/dev/sda1。
#### tune2fs:获取磁盘block大小
![tune2fs](https://raw.githubusercontent.com/lbxl2345/blogbackup/master/source/pics/%E7%A3%81%E7%9B%98IO/tune2fs.png)
利用tune2fs命令，可以看到/dev/sda1磁盘中，block的大小是4096。  
#### debugfs:获取物理块

	debugfs /dev/sda1
	debugfs:stat test.c