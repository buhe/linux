## 编译加载内核模块

内核模块能使用内核的数据结构，也不用编译内核。

例如 simple_mod 

```
make
```

编译

### 安装

```
sudo insmod simple_mod.ko
```

### 查看日志

```
dmesg
```

### 卸载

```
sudo rmmod simple_mod.ko
```

