obj-m += sched_file.o
obj-m += sched_main.o
obj-m += sched_queue.o
all:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules
clean:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
install:
	sudo insmod sched_queue.ko
	sudo insmod sched_main.ko
	sudo insmod sched_file.ko
uninstall:
	sudo rmmod sched_queue.ko
	sudo rmmod sched_main.ko
	sudo rmmod sched_file.ko
look:
	lsmod