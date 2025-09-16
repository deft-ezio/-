# [操作系统实验一：Linux内核代码分析 ](https://www.cnblogs.com/hellovenus/p/os_linux_core_study.html)

URL: https://www.cnblogs.com/hellovenus/p/os_linux_core_study.html

# 一、实验内容

1.安装Linux操作系统（Fedora 7发行版，内核版本为2.6.21.7）；

2.阅读Linux源码以了解Linux内核代码的组织方式、进程管理内部数据结构和进程调度过程、系统调用内部数据结构以及执行过程；

3.熟悉Linux下的编辑、编译和调试工具的使用；

4.实现Linux内核的编译、安装和调试。

# 二、实验目的

1.通过实验，基本具备对操作系统内核的分析与扩展能力；

2.掌握内核调试基本技术；

3.为接下来的实验以及基于Linux的内核级系统开发奠定基础。

# 三、设计思路和流程图

## 1.准备阶段：

为了实验方便，采用虚拟机以及较早的Linux系统版本Fedora7。所需工具包括：VMware-player虚拟机、Fedora7系统镜像、linux 2.6.21内核等。参考资料是《linux 操作系统实验教程》（电子工业出版社）；

## 2.软件安装：

安装VMware-player，打开解压后的Fedora虚拟机，进行实验；

## 3.解压内核：

解压文件linux-2.6.21.tar.gz，然后可以根据需要阅读和分析其中的源码，接下来编译和安装内核；

```shell
$ cd Desktop
$ tar zxvf linux-2.6.21.tar.gz
$ cd linux-2.6.21
```

## 4.生成内核配置文件：

将当前正在运行的内核对应的配置文件作为模板来生成.config文件，即将/boot目录下的已有的config文件复制到linux-2.6.21目录下；

```shell
$ make mrproper
$ cp '/boot/config-2.6.21-1.3194.fc7' ./config
```

其中，make mrproper用来保证内核树干净，执行完上面的命令之后，在根目录下将会看到config文件。

然后更新config文件：

```shell
$ make oldconfig
```

遇到新配置项的选择时，都选N。

可以定义自己的内核版本号，比如在内核代码根目录下Makefile文件，修改“EXTRAVERSION =-kong”。

## 5.编译安装内核：

```shell
$ make all
$ make modules_install
$ make install
```

## 6.最后阶段：

修改引导程序grub的配置文件/boot/grub/menu.lst，注释掉hiddenmenu，然后使用reboot命令重启系统。

```shell
$ reboot
```

 

# 四、主要数据结构及其说明

无

 

# 五、源程序并附上注释

无

 

# 六、程序运行结果及分析

## 1.生成内核配置文件：

![img](https://images0.cnblogs.com/blog/628412/201409/112145430127337.png)

 

## 2.编译和安装内核：

![img](https://images0.cnblogs.com/blog/628412/201409/112148151376895.png)

 

## 3.reboot：

![img](https://images0.cnblogs.com/blog/628412/201409/112155210591808.png)

 

## 4.查看内核

重启之后，使用命令$ uname -r查看内核版本，结果如下：

 

```shell
$ uname -r
```

 

![img](https://images0.cnblogs.com/blog/628412/201409/121525421529308.png)

 

# 七、实验体会

实验之前，我看了很多关于linux的知识，对命令行与命令、进程、文件都有所了解，然后才进行实验。虽然第一个实验不难，但毕竟是我第一次接触linux内核，有了初步的认识，迈出第一步。我觉得，C语言和linux内核操作，就像Java与安卓开发一样，前面是基础，后面是一块神秘而庞大的宝藏，等待我去探索。