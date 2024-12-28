# GDB 小技巧

### 各种缩写
| command      | shortcut | function               |
| ------------ | -------- | ---------------------- |
| print        | p        | print                  |
| frame        | f        | print current location |
| step         | s        | step in                |
| next         | n        | next line              |
| finish       | fin      | finish function        |
| continue     | c        | continue               |
| break        | b        | set breakpoint         |
| delete <idx> | d        | delete breakpoint      |
| run [args]   | r        | start program          |
| list [lines] | l        | finish function        |
| info         | i        | look infos             |
| focus <next> | fs       | focus next             |
| quit         | q        | quit gdb               |

### 两种运行带参数程序的方法
```
Method-I:
  $ gdb --args program arg1 arg2 ... argN  
  (gdb) r

Method-II:
  $ gdb program  
  (gdb) r arg1 arg2 ... argN

#Note - Run GDB with <--silent> option to hide the extra output it emits on the console.
```

### commands 命令
commands 命令可在一个断点处设置触发此断点后需要执行的一系列命令。

使用 `commands` 命令：
```
(gdb) commands <breakpoint-number>
> command1
> command2
> ...
> end
```

### 一种打印数组局部的方法
```
 例如打印 arr 96-100 的元素
 (gdb) p *&arr[96]@5
```

### ignore breakpoints
`ignore` 命令可跳过指定 breakpoints 指定次数
```
(gdb) ignore 1 1000  # 跳过 breakpoints 1 1000 次
```
