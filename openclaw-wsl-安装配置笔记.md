# 今日笔记：OpenClaw 在 WSL 的安装与配置

> 日期：2026-03-10  
> 目标：在 Windows + WSL2 环境中完成 OpenClaw 的源码获取、依赖安装、编译运行与常见问题排查。

## 1. 环境前置

- Windows 11（建议最新补丁）
- WSL2（Ubuntu 22.04 或 24.04）
- Git、CMake、GCC/G++、SDL2（以及音频/图像相关开发包）

先确认 WSL 版本：

```bash
wsl -l -v
```

如果不是 WSL2，可执行（在 PowerShell）：

```powershell
wsl --set-version <发行版名称> 2
```

---

## 2. 在 WSL 中准备基础工具链

更新并安装常用构建工具：

```bash
sudo apt update
sudo apt install -y \
  build-essential \
  cmake \
  git \
  pkg-config \
  ninja-build
```

安装 OpenClaw 常见依赖（按 Ubuntu 包名给出，实际以项目 CMake 输出为准）：

```bash
sudo apt install -y \
  libsdl2-dev \
  libsdl2-image-dev \
  libsdl2-mixer-dev \
  libsdl2-ttf-dev \
  libopenal-dev \
  zlib1g-dev
```

> 说明：不同分支/版本可能依赖略有差异，优先看 OpenClaw 仓库的 README / BUILD 文档。

---

## 3. 获取源码

```bash
git clone <OpenClaw 仓库地址> ~/openclaw
cd ~/openclaw
```

如果仓库包含子模块：

```bash
git submodule update --init --recursive
```

---

## 4. 配置与编译

推荐 out-of-source 构建：

```bash
cmake -S . -B build -G Ninja -DCMAKE_BUILD_TYPE=Release
cmake --build build -j
```

如需调试版本：

```bash
cmake -S . -B build-debug -G Ninja -DCMAKE_BUILD_TYPE=Debug
cmake --build build-debug -j
```

---

## 5. 运行方式（WSLg / X11）

### 5.1 WSLg（Windows 11 默认较方便）

直接运行可执行文件：

```bash
./build/<可执行文件名>
```

### 5.2 传统 X Server（如 VcXsrv）

若不是 WSLg，需在 Windows 侧启动 X Server，并在 WSL 配置 DISPLAY：

```bash
export DISPLAY=$(grep nameserver /etc/resolv.conf | awk '{print $2}'):0
```

可写入 `~/.bashrc` 持久化。

---

## 6. 资源文件与运行目录

很多游戏项目会依赖资源目录（如 data/assets）。常见约定：

- 可执行文件在 build 下
- 资源路径按相对路径查找

若启动时报“找不到资源”，可尝试：

1. 在项目根目录运行程序（确保相对路径正确）
2. 查看是否需要复制资源到可执行目录
3. 检查是否有 `-DASSET_DIR=...` 之类的 CMake 选项

---

## 7. 声音/输入常见问题

### 7.1 无声音

- 检查 `libsdl2-mixer-dev` / `libopenal-dev` 是否安装
- WSLg 下确认 Windows 音频设备正常
- 运行前尝试：

```bash
echo $PULSE_SERVER
```

若为空，可重启 WSL：

```powershell
wsl --shutdown
```

### 7.2 键盘/手柄异常

- SDL 在 WSL 的设备映射可能受限
- 优先先验证键盘输入，再排查手柄
- 部分 USB 外设需要 usbipd 转发到 WSL

---

## 8. 性能与稳定性建议

- Release 构建优于 Debug
- 尽量把源码放在 WSL Linux 文件系统（例如 `~/openclaw`），避免放 ` /mnt/c ` 导致 I/O 慢
- 大项目构建可加并行参数 `-j`
- 遇到奇怪链接错误先全量清理：

```bash
rm -rf build build-debug
```

---

## 9. 一套可复用的最小流程

```bash
# 1) 安装依赖
sudo apt update && sudo apt install -y build-essential cmake git ninja-build \
  libsdl2-dev libsdl2-image-dev libsdl2-mixer-dev libsdl2-ttf-dev libopenal-dev zlib1g-dev

# 2) 拉取代码
git clone <OpenClaw 仓库地址> ~/openclaw
cd ~/openclaw
git submodule update --init --recursive

# 3) 编译
cmake -S . -B build -G Ninja -DCMAKE_BUILD_TYPE=Release
cmake --build build -j

# 4) 运行
./build/<可执行文件名>
```

---

## 10. 今日结论

1. WSL2 + WSLg 是目前最省事的 OpenClaw Linux 运行方式。  
2. 成功关键点在于：**依赖包完整 + 资源路径正确 + 图形音频转发正常**。  
3. 若后续要长期开发，建议补充：`clang-format`、`gdb`、`valgrind`、`ccache` 等工具以提升调试与构建效率。
