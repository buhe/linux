obj-m += sched_fils.o
obj-m += sched_main.o
obj-m += sched_queue.o
all:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules
clean:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
