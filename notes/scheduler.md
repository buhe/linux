## 内核模块方式

> 参考 https://github.com/m4n1c22/Loadable-Scheduler

采用内核模块的方式加载一个调度器。思路是这样的：

- 采用 /proc/ 下的虚拟文件系统作为用户空间和内核空间进行数据交换，通过 libc 获取用户空间的进程的 pid 写入 /proc/ 下的一个文件，内核就可以知道需要调度的用户空间进程的 pid 。
- sched_main 负责调度，通过调用 sched_queue 来改变进程的状态，然后给进程发送信号，内核的调度器接收到信号真正的调度进程

这个调度器确切的说，是“假”的，真实的调度还是内核的调度器完成的，这个调度器只是通过给内核发送信号的形式来调度。这个“假”调度器由3个内核模块组成，分别是：sched_file sched_queue sched_main 。下面一个个介绍：

### sched_file

sched_file 是用来内核空间和用户空间交换数据的。

```c
/**
	Function Name : process_sched_add_module_init
	Function Type : Module INIT
	Description   : Initialization method of the Kernel module. The
			method gets invoked when the kernel module is being
			inserted using the command insmod.
*/
static int __init process_sched_add_module_init(void)
{
	printk(KERN_INFO "Process Add to Scheduler module is being loaded.\n");
	
	/**Proc FS is created with RD&WR permissions with name process_sched_add*/
	proc_sched_add_file_entry = proc_create(PROC_CONFIG_FILE_NAME,0777,NULL,&process_sched_add_module_fops);
	/** Condition to verify if process_sched_add creation was successful*/
	if(proc_sched_add_file_entry == NULL) {
		printk(KERN_ALERT "Error: Could not initialize /proc/%s\n",PROC_CONFIG_FILE_NAME);
		/** File Creation problem.*/
		return -ENOMEM;
	}
	/** Successful execution of initialization method. */
	return 0;
}
```

首先内核模块加载的时候，通过 proc_create 在 /proc 下创建文件交换数据。

```c
static struct proc_ops process_sched_add_module_fops = {
	.proc_read =		process_sched_add_module_read,
	.proc_write =	process_sched_add_module_write,
	.proc_open =		process_sched_add_module_open,
	.proc_release =	process_sched_add_module_release,
};
```

通过指定回调在文件打开、释放、写入、读取的时候调用不同的函数。

再看看测试程序

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

void test_pr(void) {

	FILE *fp = fopen("/proc/process_sched_add","w");
	fprintf(fp, "%d", getpid());
 	fclose(fp);

 	while(1) {
 		printf("My pid: %d\n", getpid());
 		sleep(1);
 	}
}

int main() {
    test_pr();
	return 0;
}
```

向 /proc/process_sched_add 写入测试进程的 pid ，通过 /proc/process_sched_add 内核模块就获取到测试程序的 pid ，因为在写入 /proc/process_sched_add 的时候其实回调了 process_sched_add_module_write 方法。看看 process_sched_add_module_write 的内容

```c
static ssize_t process_sched_add_module_write(struct file *file, const char *buf, size_t count, loff_t *ppos)
{
	int ret;
	long int new_proc_id;	
	
	printk(KERN_INFO "Process Scheduler Add Module write.\n");
	printk(KERN_INFO "Registered Process ID: %s\n", buf);
	
	ret = kstrtol(buf,BASE_10,&new_proc_id);
	if(ret < 0) {
		/** Invalid argument in conversion error.*/
		return -EINVAL;
	}
	
	/**	Add process to the process queue.*/
	ret = add_process_to_queue(new_proc_id);
	/**Check if the add process to queue method was successful or not.*/
	if(ret != eExecSuccess) {
		printk(KERN_ALERT "Process Set ERROR:add_process_to_queue function failed from sched set write method");
		/** Add process to queue error.*/
		return -ENOMEM;
	}

	/** Successful execution of write call back.*/
	return count;
}
```

内容很简单，把 pid 写入队列，提供队列是另一个内核模块，下面就来看看这个模块。

### sched_queue



### sched_main



