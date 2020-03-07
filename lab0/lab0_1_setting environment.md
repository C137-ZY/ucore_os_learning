如果是window操作系统，首先需要安装VirtualBox 虚拟机软件。  
下载一个已经安装好各种所需编辑/开发/调试/运行软件的Linux实验环境的VirtualBox虚拟硬盘文件(mooc-os-2015.vdi.xz，包含一个虚拟磁盘镜像文件和两个配置描述文件，下载此文件的网址址见https://github.com/chyyuu/ucore_lab下的README中的描述)。  
解压mooc-os-2015.vdi.xz到C盘的vms目录下即： C:\vms\mooc-os-2015.vdi 在VirtualBox中创建新虚拟机（设置64位Linux系统，指定配置刚解压的这个虚拟硬盘mooc-os-2015.vdi），就可以启动并运行已经配置好相关工具的Linux实验环境了。  
实验内容位于ucore_lab目录下。（可以通过如下命令获得整个实验的代码和文档： $ git clone https://github.com/chyyuu/ucore_lab.git）。  
