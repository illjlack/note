```
sudo apt update
sudo apt install clang clangd
```

vscode下载clangd插件

用clang编译项目

比如把make里的gcc换成clang

```
# CC = gcc
CC = clang
```



就能在linux里跳转、找定义了



`clangd` 需要一个 `compile_commands.json` 文件来理解项目的编译设置。这个文件通常由 CMake 生成：

```
cmake -DCMAKE_EXPORT_COMPILE_COMMANDS=ON .
```



用make可以：

**安装 `Bear`**：

- 在 Ubuntu 上安装：

  ```
  sudo apt-get install bear
  ```

- 在 macOS 上安装：

  ```
  brew install bear
  ```

**使用 `Bear` 生成 `compile_commands.json`**：

使用 `Bear` 包装 `make` 命令来生成 `compile_commands.json`：

```
bear -- make
```

`Bear` 工具会监视 `make` 执行的编译命令，并生成 `compile_commands.json` 文件。