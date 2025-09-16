# [操作系统实验三：Linux进程管理及其扩展 ](https://www.cnblogs.com/hellovenus/p/3967597.html)

Link: https://www.cnblogs.com/hellovenus/p/3967597.html

# 一、实验内容

1.阅读并分析Linux内核源代码，了解进程控制块、进程队列等数据结构；
2.实现一个系统调用，使得可以根据指定的参数隐藏进程，使用户无法使用ps或top观察到进程状态。具体要求如下：
 (1)实现系统调用int hide(pid_t pid, int on)，在进程pid有效的前提下，如果on置1，进程被隐藏，用户无法通过ps或top观察到进程状态；如果on置0且此前为隐藏状态，则恢复正常状态。
 (2)考虑权限问题，只有根用户才能隐藏进程。
 (3)设计一个新的系统调用int hide_user_processes(uid_t uid, char *binname)，参数uid为用户ID号，当binname参数为NULL时，隐藏该用户的所有进程；否则，隐藏二进制映像名为binname的用户进程。该系统调用应与hide系统调用共存。
 (4)在/proc目录下创建一个文件/proc/hidden，该文件可读可写，对应一个全局变量hidden_flag，当hidden_flag为0时，所有进程都无法隐藏，即便此前进程被hide系统调用要求隐藏。只有当hidden_flag为1时，此前通过hide调用要求被屏蔽的进程才隐藏起来。
 (5)在/proc目录下创建一个文件/proc/hidden_process，该文件的内容包含所有被隐藏进程的pid，各pid之间用空格分开。

# 二、实验目的

1.加深理解进程控制块、进程队列等概念

2.了解进程管理的具体实施方法

# 三、新增系统调用hide

## １、设计思路和流程图

**(1)设置标识**

通过设置一个标记位来控制进程是否隐藏。

在include/linux/sched.h中修改结构体 task_struct ，添加一个成员 cloak ，0表示显示，1表示隐藏。

![img](https://images0.cnblogs.com/blog/628412/201409/120210468242867.png)

 

**(2)初始化**

进程刚创建时是处于显示的状态，所以需要将标记cloak初始化为0。

fork系统调用的实现位于kernel/fork.c中，具体实现的主要函数为do_fork，do_fork中调用copy_process函数创建子进程，在该函数中初始化cloak。

![img](https://images0.cnblogs.com/blog/628412/201409/120210555433104.png)

 

**(3)添加hide系统调用**

具体流程参考实验二，具体解释见代码部分。

![img](https://images0.cnblogs.com/blog/628412/201409/120211071684853.png)

 

**(4)过滤隐藏的进程**

修改fs/proc/base.c中的proc_pid_readdir函数以及proc_pid_lookup函数，过滤掉被隐藏的进程。

![img](https://images0.cnblogs.com/blog/628412/201409/120211200436886.png)

![img](https://images0.cnblogs.com/blog/628412/201409/120211426844268.png)

 

## 2、主要数据结构及说明

使用了结构体task_struct，通过对结构体里的cloak等变量进行赋值，改变进程的状态。

 

## 3、源程序并附上注释

**(1)hide.c**

使用内核函数find_task_by_pid，通过pid获取进程task_struct。在隐藏后调用函数proc_flush_task来清空VFS层的缓冲，解除已有的dentry项。

其中包括对用户权限的判断，必须是root用户，即uid=0才能进行相关操作。

```c
#include<linux/linkage.h>
#include<linux/types.h>
#include<linux/sched.h>
#include<linux/pid.h>
#include<linux/proc_fs.h>
 
asmlinkage int sys_hide(pid_t pid, int on)
{
    struct task_struct *p=NULL;
    if(pid>0 && (current->uid)==0)//only the root user can hide the process
    {
        p=find_task_by_pid(pid);
        p->cloak=on;//set the state of the process
        if(on==1)
        {
            printk("Process %d is hidden by root.\n",pid);
        }
        if(on==0)
        {
            printk("Process %d is displayed by root.\n",pid);
        }
        proc_flush_task(p);
    }
    else
        printk("Sorry, you are not root user.Permission denied.\n");
     
    return 0;
}
```

 

**(2)测试程序**

比如隐藏进程号为1的进程，代码如下

```c
#include<stdio.h>
#include<sys/syscall.h>
#include<unistd.h>
int main()
{
    int syscallNum=321;
    pid_t pid=1;
    int on=1;
    syscall(syscallNum,pid,on);
    return 0;
}
```

　　

## 4、程序运行结果及分析

**(1)初始状态**

目前进程都是处于显示的状态，我们的目的是隐藏1号进程

![img](https://images0.cnblogs.com/blog/628412/201409/120231171993281.png)

 

**(2)在非root用户下执行测试程序**

进程未被隐藏，用 dmesg 命令查看输出

![img](https://images0.cnblogs.com/blog/628412/201409/120231432157033.png)

 

**(3)切换到root用户**

再次执行程序，结果如下，1号进程被隐藏了

![img](https://images0.cnblogs.com/blog/628412/201409/120232588713520.png)

 

**(4)更改参数on=0**

被隐藏的进程将再次出现

![img](https://images0.cnblogs.com/blog/628412/201409/120259333249004.png)

 

 

# 四、新增系统调用hide_user_processes

## 1、设计思路和流程图

基于实验二以及新增系统调用hidden这两个实验，这个实验很容易完成。

遍历系统中所有的进程，隐藏满足要求的进程。使用函数for_each_process遍历所有进程，每个进程的task_struct中有成员变量uid和comm，uid为该进程的用户id，comm为进程名，根据要求隐藏对应进程。具体解释见代码部分。

![img](https://images0.cnblogs.com/blog/628412/201409/120245143245901.png)

 

## 2、主要数据结构及说明

使用了结构体task_struct，通过其中的cloak等变量，控制进程的状态。 

 

## 3、源程序并附上注释

**(1)hide_user_processes.c**

包括三个参数，前两个参数是根据实验要求设置的，第三个参数recover用于恢复被隐藏的进程。如果recover=0那么隐藏相应的进程，如果不为0，则恢复所有进程为显示状态。

```c
#include<linux/linkage.h>
#include<linux/types.h>
#include<linux/sched.h>
#include<linux/pid.h>
#include<linux/proc_fs.h>
#include<linux/string.h>
 
asmlinkage int sys_hide_user_processes(uid_t uid,char *binname,int recover){
    struct task_struct *p=NULL;
    if(recover==0)
    {
        if(current->uid==0)//1.Paragram recover=0;2.root => you can hide the process
        {
            if(binname==NULL)//when null, hide all the processes of the corresponding user
            {
                for_each_process(p)
                {
                    if((p->uid)==uid)
                    {
                        p->cloak=1;
                        proc_flush_task(p);
                    }
                }
                printk("All of the processes of uid %d are hidden.\n",uid);
            }
            else//otherwise, hide the process of the corresponding name
            {
                for_each_process(p)
                {
                    char *s=p->comm;
                    if(strcmp(s,binname)==0 && p->uid==uid)
                    {
                        p->cloak=1;
                        printk("Process %s of uid %d is hidden.\n",binname,uid);
                        proc_flush_task(p);
                    }
                }
            }
        }
        else
            printk("Sorry, you are not root user. Permission denied.\n");
    }
    else if(recover != 0 && (current->uid)==0)//display all of the processes, including which are hidden
    {
        for_each_process(p)
        {
            p->cloak=0;
        }
    }
     
    return 0;
}
```

　　

**(2)测试程序（可以修改参数，从而实现不同的要求）**

```c
#include<stdio.h>
#include<sys/syscall.h>
#include<unistd.h>
int main()
{
    int syscallNum=322;
    uid_t uid=500;
    char *binname=NULL;//Author:Kong
    int recover=0;
    syscall(syscallNum,uid,binname,recover);
    return 0;
}
```

　　

## 4、程序运行结果及分析

**(1)与上个实验一样，如果非root用户，则没有隐藏进程的权限**

初始状态如下

![img](https://images0.cnblogs.com/blog/628412/201409/120254098565459.png)

 

**(2)隐藏uid为0，进程名为init的进程**

修改参数uid_t uid=0; char *binname="init";

结果如下，对应的进程被隐藏了

![img](https://images0.cnblogs.com/blog/628412/201409/120255502622185.png)

 

**(3)隐藏uid=500（对应用户名为seu）的所有进程**

修改参数uid_t uid=500; char *binname=NULL;

结果如下，seu用户的所有进程被隐藏了

![img](https://images0.cnblogs.com/blog/628412/201409/120302260595533.png)

 

**(4)更改参数recover=1，所有进程将恢复为显示状态**

结果如下

![img](https://images0.cnblogs.com/blog/628412/201409/120303597623906.png)

 

 

# 五、在/proc目录下创建一个文件/proc/hidden

## 1、设计思路和流程图

**(1)全局变量hidden_flag**

因为这个实验又涉及到隐藏进程的问题，所以需要再设置一个标识作为判断，hidden文件的读写对它进行操作即可。

在/fs/proc目录下新建一个头文件，里面包含一个全局变量，其它文件中需要用到这个全局变量的时候，需使用include包含这个头文件。

![img](https://images0.cnblogs.com/blog/628412/201409/121013226527731.png)

```
1 extern int hidden_flag;
```

 

**(2)hidden文件的创建及读写**

 proc文件系统在初始化函数proc_root_init中会调用proc_misc_init函数，此函数用于创建/proc根目录下的文件，那么将创建hidden文件的代码插入到此函数中就可以在proc初始化时得到执行。在/fs/proc/proc_misc.c中添加回调函数，在/fs/proc/proc_misc.c中proc_misc_init函数的最后添加创建hidden文件的代码，并指定其回调函数，具体说明见代码部分。

![img](https://images0.cnblogs.com/blog/628412/201409/121016521528661.png)

 

![img](https://images0.cnblogs.com/blog/628412/201409/121017175749528.png)

 

![img](https://images0.cnblogs.com/blog/628412/201409/121017243718852.png)

 

**(3)根据hidden_flag显示/隐藏进程**

结合上面根据cloak判断进程，这个实验与之类似，只需在fs/proc/base.c文件中，修改proc_pid_readdir函数以及proc_pid_lookup函数，在cloak判断之前，增加hidden_flag对进程的约束。

![img](https://images0.cnblogs.com/blog/628412/201409/121026040903109.png)

 

![img](https://images0.cnblogs.com/blog/628412/201409/121026113245132.png)

 

**(4)重新编译安装内核，重启之后进行测试。**

 

## 2、主要数据结构及说明

使用到了结构体、进程控制块等数据结构。 

 

## 3、源程序并附上注释

主要是hidden文件的创建及读写。

在/fs/proc/proc_misc.c中添加回调函数，首先初始化全局变量hidden_flag为1；读操作中sprintf将hidden_flag的值传给page，然后显示出来；写操作中使用copy_from_user先将用户输入的值传给temp，然后从temp中得到值传给hidden_flag。

```c
int hidden_flag=1;
 
static int proc_read_hidden(char *page,char **start,off_t off,int count,int *eof,void *data)
{
    int len=0;
    len=sprintf(page,"%d",hidden_flag);
    return len;
}
 
static int proc_write_hidden(struct file *file,const char *buffer,unsigned long count,void *data)
{
    int len=0;
    char temp[BUF_LEN];
    if(count > BUF_LEN)
            len = BUF_LEN;
        else
            len = count;
     
    copy_from_user(temp, buffer, len);//convert user's input to temp
    temp[len]='\0';
    hidden_flag=temp[0]-'0';//set the value of hidden_flag
    return len;
}
```

　　

下面是创建hidden文件的代码，使用的是create_proc_entry函数，然后指定其回调函数。

```c
struct proc_dir_entry *ptr=create_proc_entry("hidden",0644,NULL);
ptr->read_proc=proc_read_hidden;
ptr->write_proc=proc_write_hidden;
```

　　

## 4、程序运行结果及分析

**(1)首先默认设置hidden_flag=1，使用hide_user_processes隐藏uid=500，即seu用户的所有进程。**

![img](https://images0.cnblogs.com/blog/628412/201409/121040357939532.png)

 

**(2)将hidden_flag的值改为0，这时再查看进程，所有被隐藏的进程又出现了。**

![img](https://images0.cnblogs.com/blog/628412/201409/121044065599274.png)

 

![img](https://images0.cnblogs.com/blog/628412/201409/121044147931827.png)

 

**(3)将hidden_flag改回为1，查看进程，seu用户的进程又处于隐藏状态了。**

![img](https://images0.cnblogs.com/blog/628412/201409/121046267155357.png)

 

![img](https://images0.cnblogs.com/blog/628412/201409/121046355901079.png)

 

 

# 六、在/proc目录下创建一个文件/proc/hidden_process

## 1、设计思路和流程图

 该文件用于存储所有被隐藏进程的pid，因为这个文件暂时不涉及用户写入，所以只需设置其读回调函数。具体说明见代码部分。

 ![img](https://images0.cnblogs.com/blog/628412/201409/121054249966272.png)

 

![img](https://images0.cnblogs.com/blog/628412/201409/121055374964790.png)

 

![img](https://images0.cnblogs.com/blog/628412/201409/121055477622053.png)

 

然后重新编译安装内核，重启后测试。

 

## 2、主要数据结构及说明

使用了结构体、进程控制块、进程队列等数据结构。 

 

## 3、源程序并附上注释

通过for_each_process函数，将被隐藏的进程的pid传给buf，最后将buf传给page显示在终端上。

```
static` `int` `proc_read_hidden_processes(``char` `*page,``char` `**start,off_t off,``int` `count,``int` `*eof,``void` `*data)``{  ``  ``static` `char` `buf[1024*8]=``""``;``    ``char` `tmp[128]; ``    ``struct` `task_struct *p;``  ``if``(off>0)``    ``return` `0;``  ``sprintf``(buf,``"%s"``,``""``);``    ``for_each_process(p)``    ``{``    ``if``(p->cloak==1)``    ``{``        ``sprintf``(tmp,``"%d "``,p->pid); ``//a important step``        ``strcat``(buf,tmp); ``    ``}``    ``} ``  ``sprintf``(page,``"%s"``,buf);``//convert buf to page to display on the terminal``    ``return` `strlen``(buf);``//Author:Kong``}
```

　　

##  4、程序运行结果及分析

**(1)首先隐藏uid=500，即seu用户的所有进程**

查看hidden_process文件里的内容，发现所有被隐藏进程的pid都存在这里。

![img](https://images0.cnblogs.com/blog/628412/201409/121102386376561.png)

 

**(2)恢复所有进程为显示状态**

这时没有被隐藏的进程，hidden_process文件里的内容也为空。

![img](https://images0.cnblogs.com/blog/628412/201409/121105144968446.png)

 

**(3)经过多次不同的测试，实现了效果，没有bug发生**

 

# 七、实验体会

这次实验的前两个，增加系统函数的使用很容易就完成了。我主要在后面的文件系统部分遇到了一些问题，使用指针的时候总是遇到很大的问题，要么是重启之后不能进入内核，要么是测试的时候虚拟机卡住一动不动，然后必须进入原来的内核当中修改新内核，耗费了很多的时间。其实该做好备份的，这样也许可以节省很多时间。

最令我不解的是，使用char *a的形式总是出问题，改为char a[]的形式问题便消失了，具体原因还有待我继续寻找。

这次实验之后，我对linux的进程控制、命令、系统调用、文件系统等方面都有了更深的认识，使用起来比以前熟练多了，但问题依然有很多，需要我继续想方设法解决。以前看的资料大多是笼统地讲解linux，虽然对整体了解有帮助，但是内部实现我涉猎不广，这次实验帮助我对内部实现有了更多的认识。