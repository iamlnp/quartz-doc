链接： http://confluence.ln.ad/pages/viewpage.action?pageId=77404822

近日QA测试时，多次遇到跨挂载目录mv文件但文件的ino发生变化问题。此处简单查了下内核和mv命令实现代码，对此进行了探究。
##### 1，跨两个挂载目录，因为其vfsmount不同，所以会导致rename在系统调用层返错-1，errno为EXDEV；

详见截图代码：
![|675](Pasted%20image%2020240425111131.png)
关于vfsmount，vfsmount结构描述的是一个独立文件系统的挂载信息，每个不同挂载点对应一个独立的vfsmount结构，属于同一文件系统的所有目录和文件隶属于同一个vfsmount，该vfsmount结构对应于该文件系统顶层目录，即挂载目录  
比如对于mount -t ext4 /mnt/dir1,挂载点为/mnt/dir1，对于dir1这个目录，其产生新的vfsmount，独立于根文件系统挂载点/所在的vfsmount；
##### 2，mv 命令会执行系统调用renameat，在renameat返错后，其又如何处理呢？

简单浏览mv的代码后，可以发现在报错后执行了copy加删除操作。具体可以看下注释。
![|650](Pasted%20image%2020240425111210.png)

##### 3，strace跟踪记录

这个Bug [[DNP4-8853]](http://jira.ln.ad/browse/DNP4-8853?filter=12855) 里，Zengyan同学也提供strace跟踪记录，可以更详细了解mv执行过程。

  

谢谢！