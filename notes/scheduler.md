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

```c
int init_process_queue(void) {

	printk(KERN_INFO "Initializing the Process Queue...\n");
	/**Generating the head of the queue and initializing an empty process queue.*/
	INIT_LIST_HEAD(&top.list);
	return 0;
}
```

加载这个内核模块，创建一个队列。

当 pid 传进来的时候，调用 add_process_to_queue 把对应的进程放入队列。

```c
int add_process_to_queue(int pid) {
	...
	/**Setting the process id to the process info node new_process*/
	new_process->pid = pid;
	/**Setting the process state to the process info node new_process as waiting.*/
	new_process->state = eWaiting;
	/**Make the task level alteration therefore the process pauses its execution since in wait state.*/
	task_status_change(new_process->pid, new_process-> state);
  ...
}
```

重点看这几行，在加入队列之前，把进程的状态设为 waiting 。

```c
enum task_status_code task_status_change(int pid, enum process_state eState) {
	...
	/**Obtain the task struct associated with provided PID.*/
	current_pr = pid_task(find_vpid(pid), PIDTYPE_PID);
	...
	if(eState == eRunning) {
		/**Trigger a signal to continue the given task associated with the process.*/
		kill_pid(task_pid(current_pr), SIGCONT, 1);
		printk(KERN_INFO "Task status change to Running\n");
	}
	/**Check if the state change was Waiting.*/
	else if(eState == eWaiting) {
		/**Trigger a signal to pause the given task associated with the process.*/
		kill_pid(task_pid(current_pr), SIGSTOP, 1);
		printk(KERN_INFO "Task status change to Waiting\n");
	}
	/**Check if the state change was Blocked.*/
	else if(eState == eBlocked) {

		printk(KERN_INFO "Task status change to Blocked\n");
	}
	/**Check if the state change was Terminated.*/
	else if(eState == eTerminated) {

		printk(KERN_INFO "Task status change to Terminated\n");
	}
	/**Return the task status code as exists.*/
	return eTaskStatusExist;
}
```

如上，当待设置的状态是 waiting ，发送 SIGSTOP 信号，使进程暂停交出 cpu 时间，完成加入队列。

### sched_main

来看 sched_main.c ，这是第三个内核模块，调度主逻辑在这。

DECLARE_DELAYED_WORK 为 context_switch 函数生成一个任务，并绑定在 scheduler_hdlr。

```c
static DECLARE_DELAYED_WORK(scheduler_hdlr, context_switch);
```

当内核模块被加载时，调用 queue_delayed_work 启动 scheduler_hdlr ，开启第一次调度。

```c
static int __init process_scheduler_module_init(void)
{
		...
		q_status = queue_delayed_work(scheduler_wq, &scheduler_hdlr, time_quantum*HZ);
  	...
}
```

调度的逻辑在 context_switch 函数。

```c
static void context_switch(struct work_struct *w){
	
	/** Boolean status of the queue.*/
	bool q_status=false;
	
	printk(KERN_ALERT "Scheduler instance: Context Switch\n");

	/**Invoking the static round robin scheduling policy.*/
	static_round_robin_scheduling();

	/** Condition check for producer unloading flag set or not.*/
	if (flag == 0){
		/** Setting the delayed work execution for the provided rate */
		q_status = queue_delayed_work(scheduler_wq, &scheduler_hdlr, time_quantum*HZ);
	}
	else
		printk(KERN_ALERT "Scheduler instance: scheduler is unloading\n");
}
```

queue_delayed_work 启动 scheduler_hdlr ，开启下一次调度。

```c
int static_round_robin_scheduling(void)
{
	/**Storage class variable to detecting the process state change.*/
	int ret_process_state=-1;

	printk(KERN_INFO "Static Round Robin Scheduling scheme.\n");
	
	/**Removing all terminated process from the queue.*/
	remove_terminated_processes_from_queue();

	/**Check if the current process id is INVALID or not.*/
	if(current_pid != -1) {
		/**Add the current process to the process queue.*/	
		add_process_to_queue(current_pid);	
	}

	/** Obtaining the first process in the wait queue.*/
	current_pid = get_first_process_in_queue();
	/**
		Check if the obtained process id is invalid or not. If Invalid indicates,
		the queue does not contain any active process.
	*/
	if(current_pid != -1) {
		/**Change the process state of the obtained process from queue to running.*/
		ret_process_state = change_process_state_in_queue(current_pid, eRunning);
		/**Remove the process from the waiting queue.*/
		remove_process_from_queue(current_pid);
	}
	
	printk(KERN_INFO "Currently running process: %d\n", current_pid);

	/**Check if there no processes active in the scheduler or not.*/
	if(current_pid != -1) {
		printk(KERN_INFO "Current Process Queue...\n");
		/**Print the wait queue.*/
		print_process_queue();
		printk(KERN_INFO "Currently running process: %d\n", current_pid);
	}
	
	
	/** Successful execution of the method. */
	return 0;
}
```



