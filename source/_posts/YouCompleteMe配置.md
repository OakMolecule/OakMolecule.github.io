---
title: YouCompleteMe 配置
date: 2023-9-19 08:56:00
comments: false
categories:
  - Vim
tags:
  - Vim
  - YouCompleteMe
  - C/C++
  - clang-tidy
  - LSP
---

本文记录 YouCompleteMe（YCM）的安装与 C/C++ 补全配置，并留存 `.vimrc`、`compile_commands.json` 等要点。行文按「先装起来 → 再配工程 → 再调 clangd → 最后理解 LSP」的顺序展开。安装方式基于 **vim-plug + `install.py --clangd-completer`**；Neovim 用户可将插件目录改为 `~/.local/share/nvim/plugged`。YCM 详见 [ycm-core/YouCompleteMe](https://github.com/ycm-core/YouCompleteMe)，C/C++ 语言服务详见 [clangd](https://clangd.llvm.org/)。

<!-- more -->

## 环境要求

YCM 对 Vim 版本与 Python 支持有硬性要求：

- Vim **≥ 7.4.1578**
- 编译时启用 **Python 3** 接口

检查方式：

```bash
vim --version | head -n 5
```

在 Vim 命令模式下执行：

```vim
:echo has('python') || has('python3')
```

输出为 `1` 表示支持；为 `0` 则需[从源码编译 Vim](https://github.com/ycm-core/YouCompleteMe/wiki/Building-Vim-from-source)（并不复杂，按官方 Wiki 操作即可）。

## 安装 vim-plug

[vim-plug](https://github.com/junegunn/vim-plug) 是轻量的 Vim 插件管理器，安装前需已安装 `git` 与 `curl`。

```bash
curl -fLo ~/.vim/autoload/plug.vim --create-dirs \
    https://raw.githubusercontent.com/junegunn/vim-plug/master/plug.vim
```

编辑 `~/.vimrc`，在文件开头加入（插件默认安装到 `~/.vim/plugged/`）：

```vim
call plug#begin('~/.vim/plugged')

Plug 'ycm-core/YouCompleteMe'

call plug#end()

filetype plugin indent on
```

保存后安装插件，任选其一：

```bash
vim +PlugInstall +qall
```

或在 Vim 内执行 `:PlugInstall`。

首次批量安装后可能出现配色丢失、退格键异常等现象，属常见情况，完成后续配置后一般会恢复正常。

## 安装 YouCompleteMe

若上一步已在 `~/.vimrc` 中加入 `Plug 'ycm-core/YouCompleteMe'`，保存后执行 `:PlugInstall` 即可拉取源码。

> 说明：原仓库 [Valloric/YouCompleteMe](https://github.com/Valloric/YouCompleteMe) 已归档，维护方现为 [ycm-core/YouCompleteMe](https://github.com/ycm-core/YouCompleteMe)。

### 运行安装脚本

`:PlugInstall` 完成后，进入插件目录并执行官方安装脚本。C/C++ 补全建议带上 **`--clangd-completer`**，会一并安装 [clangd](https://clangd.llvm.org/)（后文详述其作用与配置）：

```bash
cd ~/.vim/plugged/YouCompleteMe
./install.py --clangd-completer
```

脚本会自动拉取并编译所需组件；若缺少依赖，按终端提示安装 `cmake`、`python3` 等即可。执行成功后，YCM 本体即安装完成。

## 配置 Vim（可选）

YCM 安装不难，日常更费时间的是 Vim 整体配置。我当时的完整配置托管在 GitHub：

- [OakMolecule/dot_vimrc](https://github.com/OakMolecule/dot_vimrc)

可按需 fork 或摘取片段合并进自己的 `~/.vimrc`。

## C/C++ 工程：compile_commands.json

要让 C/C++ 补全和静态检查「认得」你的工程，需要一份 **`compile_commands.json`**：其中记录每个源文件真实的编译命令（头文件路径、宏、`-std` 等）。clangd 与 YCM 会从当前文件所在目录**向上查找**该文件；找不到时往往只能做语法级猜测，补全和检查质量都会变差。官方说明见 [clangd：Project setup](https://clangd.llvm.org/installation#project-setup)。

下面介绍两种常见生成方式。

### 使用 CMake 生成

在配置阶段打开 `CMAKE_EXPORT_COMPILE_COMMANDS`，构建后会在构建目录生成该文件。建议在工程根目录做符号链接，便于编辑器查找：

```bash
cmake -B build -DCMAKE_EXPORT_COMPILE_COMMANDS=ON
cmake --build build
ln -sf build/compile_commands.json .
```

若使用较旧的 CMake 写法，也可在 `CMakeLists.txt` 中加入：

```cmake
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)
```

### 使用 Bear 生成（Makefile 项目）

没有 CMake、只有 **Makefile** 时，可用 [Bear](https://github.com/rizsotto/Bear) 拦截 `make` 过程中的编译命令并写出 `compile_commands.json`（Arch 等发行版包名一般为 `bear`）：

```bash
bear -- make clean
bear -- make -j"$(nproc)"
```

生成的文件位于执行 `bear` 时的当前目录（通常是工程根目录）。若构建在子目录，注意在该目录执行，或将 `compile_commands.json` 链接到工程根。

> 提示：更换构建方式或切换分支后，记得重新生成或更新 `compile_commands.json`，必要时在 Vim 中执行 `:YcmRestartServer`。

## 补全由 clangd 提供

安装时若使用了 `--clangd-completer`，YCM 在编辑 C/C++ 时会启动 **[clangd](https://clangd.llvm.org/)** 作为语言后端，负责补全、跳转、悬停等；这与早年 YCM 直接绑 libclang 的路径不同。

使用上只需：

1. 按上文准备好工程根目录的 `compile_commands.json`；
2. 在工程内用 Vim 打开源文件，正常触发补全（如 `Ctrl+N` / YCM 默认映射）。

clangd 会读取编译数据库，按真实编译环境解析类型与符号。若补全不准，优先检查 `compile_commands.json` 是否存在、是否过期。

## 配置 clangd：clang-tidy 与启动参数

[clangd](https://clangd.llvm.org/) 可集成 **clang-tidy** 做静态检查（风格、现代化改写、潜在 bug 等），诊断与补全共用同一份 `compile_commands.json`。系统需安装 **clang-tidy**（常与 `clang` 同包，如 Arch 的 `clang`）。

在 `~/.vimrc` 中通过 YCM 向 clangd 传入启动参数（`g:ycm_clangd_args`）：

```vim
let g:ycm_clangd_args = ['-background-index', '-clang-tidy']
```

| 参数 | 作用 |
|------|------|
| `-background-index` | 后台索引工程，大项目补全更跟手 |
| `-clang-tidy` | 启用 clang-tidy 诊断 |

工程根目录可再放 **`.clang-tidy`**，细粒度控制检查项，例如：

```yaml
Checks: '-*,readability-*,modernize-*'
WarningsAsErrors: ''
HeaderFilterRegex: '.*'
```

保存文件后，YCM 经 clangd 刷新诊断。常用命令：

- `:YcmDiags` — 查看问题列表
- `:YcmRestartServer` — 更新 `compile_commands.json` 或修改 `g:ycm_clangd_args` 后重启语言服务

更多 clangd 配置项（如 `config.yaml`）见 [clangd 配置文档](https://clangd.llvm.org/config)。

## clangd 如何提供服务：LSP 简述

读到这里，可以把前文串起来：**Vim + YCM** 负责界面与快捷键，**clangd** 负责「懂代码」；二者之间走的是 **LSP**（Language Server Protocol，语言服务器协议）。

**LSP** 是微软提出的一套标准：让编辑器与语言服务用 JSON-RPC 通信，而不必为「每种编辑器 × 每种语言」各写一套插件。可以把它理解成固定分工：

| 角色 | 在本文中的对应 | 职责 |
|------|----------------|------|
| **客户端（Client）** | Vim + YCM | 展示补全菜单、诊断下划线、跳转；把打开文件、编辑、保存等事件发给服务端 |
| **语言服务器（Server）** | [clangd](https://clangd.llvm.org/) | 解析工程；返回补全候选、诊断、定义位置等 |

典型交互如下：

``` mermaid
flowchart LR
  User[用户编辑代码] --> Vim[Vim 编辑器]
  Vim --> YCM[YCM 客户端]
  YCM <-->|LSP 请求/响应| Clangd[clangd 语言服务器]
  Clangd --> DB[(compile_commands.json 等)]
```

LSP 统一描述的能力包括：**补全**、**诊断**（含 clang-tidy）、**跳转定义/引用**、**悬停文档**、**重命名**等。同一种服务器可被多种客户端复用——clangd 既可为 YCM 服务，也可用于 Neovim 的 `nvim-lspconfig`、VS Code 的 C/C++ 扩展等。

因此，本文的配置链路可以概括为：

1. **安装** YCM，并用 `install.py --clangd-completer` 拉起 clangd；
2. **工程侧** 用 CMake 或 Bear 生成 `compile_commands.json`，让 clangd 知道如何编译每个文件；
3. **编辑器侧** 用 `g:ycm_clangd_args`、`.clang-tidy` 调整 clangd 行为；
4. **底层** YCM 通过 LSP 与 clangd 对话，你在 Vim 里看到补全与诊断。

## 延伸阅读

- [clangd 官方站点](https://clangd.llvm.org/)
- [如何在 Linux 下利用 Vim 搭建 C/C++ 开发环境？——韦易笑的回答](https://www.zhihu.com/question/47691414/answer/373700711)
