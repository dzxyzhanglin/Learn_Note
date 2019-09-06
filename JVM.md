# JVM



## JVM内存



### 查看当前所有进程指令

`jps -l`



### 导出某个进程的内存映像文件

`jmap -help` 查看`jmap`所有指令

`jmap -dump:format=b,file=heap.hprof 进程ID`



### 查看内存映像文件

* 在<https://www.eclipse.org/mat/downloads.php>下载对应的MAT  （Memory Analyzer Tool）
* 在MAT中打开映像文件







## jstack

### jstack <进程ID>  > <文件名.txt>

 