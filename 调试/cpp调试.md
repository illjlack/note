# C++ 调试文档

## 基本调试操作

- **F9**: 设置/取消断点
- **F5**: 开始调试或继续执行，运行到下一个断点

## 附加进程调试 (Attach to Process Debugging)

附加进程调试允许你将调试器连接到一个正在运行的进程，而不是在启动时附加调试器。这对于调试已经运行的应用程序特别有用。

## 条件断点

右键 -> 编辑断点 -> 表达式 (如 `n == 95`)

## VSCode 调试控制台

可以在调试控制台使用 `-exec` 命令执行 GDB 命令辅助调试，例如：

```sh
-exec x /20bx 0xd865bff940
```

这将以16进制打印当前地址后的20个字节。

## GDB 调试

### 断点

- **break (b)**: 设置断点
  - 格式：行号, 文件:行号, 条件, 函数名
  - 示例：`b 40 if ab.a == 1`

- **watch**: 设置监视断点
  - 格式：`watch 表达式`

- **info (i)**: 显示断点信息

- **clear**: 删除行号或函数名断点
- **delete (d, del)**: 删除所有断点，断点范围，或指定编号的断点

- **disable (dis)**: 禁用断点
- **enable (en, ena)**: 启用断点
  - `enable once 断点编号`
  - `enable count 数量 断点编号`

- **ignore**: 忽略指定编号的断点一定次数

### 保存和读取断点

- **save breakpoint 文件名**: 保存断点
- **gdb xx -x 文件名**: 读取断点

### 远程调试

`gdbserver` 用于远程调试：

1. **在目标机器上启动 gdbserver**：

   调试新进程：
   ```sh
   gdbserver <ip>:<port> ./运行文件
   ```

   附加到已运行的进程：
   ```sh
   gdbserver <ip>:<port> --attach <进程ID>
   ```

2. **在调试机器上启动 GDB 并连接到 gdbserver**：

   ```sh
   gdb
   (gdb) target remote <ip>:<port>
   ```

### 执行控制

- **run (r)**: 启动程序
- **continue (c)**: 继续执行
- **kill**: 停止程序
- **quit (q)**: 退出 GDB
- **where**: 显示当前执行位置和调用堆栈 (bt 的别名)
- **step (s)**: 单步进入
- **next (n)**: 单步跳过
- **finish**: 运行到函数结束

### 查看信息

- **backtrace (bt)**: 显示调用堆栈信息
  - `bt 栈帧数`: 显示指定数量的栈帧
  - `bt -栈帧数`: 反向显示指定数量的栈帧
  - `backtrace full`: 显示所有栈帧的局部变量

- **frame (f)**: 显示当前栈帧
  - `f 帧编号`: 切换到指定栈帧

- **up n**: 向上移动 n 个栈帧
- **down n**: 向下移动 n 个栈帧

- **info locals**: 查看当前帧的局部变量

- **list (l)**: 查看当前源码
  - `l 行号`: 显示指定行号代码
  - `l 函数名`: 显示指定函数的代码
  - `l -`: 显示上几行代码
  - `l 开始,结束`: 显示指定范围的代码
  - `l 文件名:行号`: 显示指定文件的行号代码

- **set listsize 数字**: 设置一次显示的代码行数
- **show listsize**: 查看当前一次显示的代码行数

### 查看内存和变量

- **x 地址**: 查看指定地址的内存
  
  - 格式：`x /nfu 地址`
    - `n`: 单位数量
    - `f`: 格式化（如十进制 `i`, 16 进制 `x`, 字符 `c`）
    - `u`: 单位大小（`b` 字节, `h` 双字节, `w` 四字节, `g` 八字节）
  
  ```
  比如：
  (gdb) p a         
  $3 = {1, 2, 3, 4}
  (gdb) x /4xw &a
  0x70ad1ff940:   0x00000001      0x00000002      0x00000003      0x00000004
  ```
  
  
  
- **print (p)**: 查看变量(表达式，甚至函数，call也行，可以改变变量)
  
  - `p 变量名`
  - `p 文件名::变量名`
  
- **ptype**: 查看变量类型
  
  - `ptype 变量`
  - `ptype 数据类型`
  
- **set print pretty**: 格式化结构体输出

- **set print array on/off**: 数组友好显示与否

### 在没有 `-g` 参数的情况下调试 Release 版本的方法

1. **编译 Release 版本**
   
   编译时使用适当的优化选项（如 `-O2`, `-O3`），但不包含调试符号表 `-g`：

   ```sh
   gcc -O2 -o release_version source.c
   ```

2. **生成调试符号表**

   对调试版本（带有 `-g` 参数的版本）生成调试符号表，并将其作为符号源使用：

   ```sh
   objcopy --only-keep-debug debug_version debug_version.debug
   ```

3. **调试 Release 版本**

   使用 GDB 调试 Release 版本时，指定调试符号文件以及 Release 版本的可执行文件：

   ```sh
   gdb --symbol=debug_version.debug --exec=release_version
   ```

### 直接使用调试版作为符号源的方法

如果有权限访问调试版本的可执行文件，也可以直接将其作为符号源来调试 Release 版本：

```sh
gdb --symbol=debug_version --exec=release_version
```

这种方法省去了生成和维护单独的调试符号文件的步骤，但需要确保可以获取到调试版本的可执行文件。
