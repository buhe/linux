## 改变 os id

编辑 .config

```
CONFIG_LOCALVERSION="-v7l-bugu-os"
```

然后编译

```
uname -r
```

返回类似

```
5.15.32-v7l-bugu-os+
```

成功改变 os id

