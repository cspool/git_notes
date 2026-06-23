---
layout: post
title: "project-docker-runner：容器启动挂载与权限策略"
date: 2026-06-23 16:30:00 +0800
categories: notes
---

# project-docker-runner：容器启动挂载与权限策略

`project-docker-runner` 是一个很小但边界很明确的 Codex skill：输入一个 Docker 镜像名，基于当前项目目录生成 `docker/run_project_gpu.sh`。脚本自动从项目目录名推导 `PROJECT_NAME`，默认容器名为 `${PROJECT_NAME}_dev`，并把 GPU、网络、IPC、CUDA、Claude/Codex 配置、模型目录和缓存目录整理成一套可重复启动的开发容器配置。

它解决的问题不是“怎么把容器跑起来”这么简单，而是把容器启动时最容易混乱的三件事固定下来：

- 哪些目录应该给容器写权限；
- 哪些 host 目录只能只读共享；
- 哪些 host 用户配置必须保留，但不能把整个 host HOME 暴露给容器。

下面以当前项目 `Qwen35_27B_on_2GPU` 的默认生成结果为例说明。变量展开后，项目目录是：

```text
PROJECT_NAME=Qwen35_27B_on_2GPU
CONTAINER_NAME=Qwen35_27B_on_2GPU_dev
IMAGE_NAME=qwen35-27b-llama-cpp:cuda12.9
CONTAINER_HOME=/home/descfly
CACHE_DIR=/data3/docker_cache/Qwen35_27B_on_2GPU
MODEL_DIR=/data3/docker_model/Qwen35_27B_on_2GPU
WORKDIR=/workspace/Qwen35_27B_on_2GPU
```

## 1. 总体原则

这个启动脚本的核心原则是：容器内开发体验尽量完整，但 host 侧暴露面尽量窄。

容器默认可以 root 登录，方便安装依赖、调试 CUDA、运行 agent 工具；但它不会挂载整个 host HOME，也不会让容器 root 随意改 host 的包环境。真正可写的目录只有项目、容器自己的 cache/HOME/npm、模型目录，以及明确需要写状态的 Claude/Codex 配置。

只读共享目录则分成三类：

- host 工具路径：目前只保留 `/opt/nvidia`；
- host 用户主包入口：目前只保留 `~/.local` 的只读旁路挂载；
- skill 源目录：只读挂载 Claude/Codex 相关 skill root，避免容器修改共享 skill 仓库。

`.bashrc` 不再直接 bind mount。脚本每次启动前会从 host `.bashrc` 中按关键词提取 Claude 相关 export 行，写入容器自己的 HOME cache 里的 `.bashrc`。这样容器内能拿到 Claude Code 所需环境变量，同时避免 host `.bashrc` 中的 conda、CUDA、cargo、bun、PATH 等宿主机初始化逻辑污染容器。

## 2. 可写挂载：容器工作的持久层

这类挂载的目标是让容器能正常开发、写缓存、安装 npm CLI、保存模型或推理文件。它们都是当前项目或项目专属缓存，不是 host 的真实 HOME。

| 分类 | Host 路径 | 容器路径 | 权限 | 目的 |
|---|---|---|---|---|
| 项目工作区 | `/data3/Projects/Qwen35_27B_on_2GPU` | `/workspace/Qwen35_27B_on_2GPU` | `rw` | 代码、脚本、实验输出直接写回项目 |
| 容器 HOME | `/data3/docker_cache/Qwen35_27B_on_2GPU/home/descfly` | `/home/descfly` | `rw` | 容器用户 HOME，不等于 host HOME |
| 项目 cache | `/data3/docker_cache/Qwen35_27B_on_2GPU` | 同路径 | `rw` | Hugging Face、Torch、Triton、XDG cache 等 |
| npm global | `/data3/docker_cache/Qwen35_27B_on_2GPU/npm-global` | 同路径 | `rw` | 安装和复用 `codex`、`claude` 等 npm CLI |
| 模型目录 | `/data3/docker_model/Qwen35_27B_on_2GPU` | 同路径 | `rw` | 模型权重、量化文件、推理资产 |

注意模型目录默认已经去掉最后一级冗余的 `models`。如果确实需要子目录，可以启动时显式设置：

```bash
MODEL_NAME=models bash docker/run_project_gpu.sh
```

默认值则是：

```text
MODEL_DIR=/data3/docker_model/Qwen35_27B_on_2GPU
```

## 3. Agent 配置挂载：少量 rw，保证工具可用

Claude/Codex 这类 agent 工具需要保存登录态、本地配置、插件状态和运行时 metadata。如果全部只读，容器里工具经常无法更新状态；如果挂整个 HOME，又会暴露太多 host 文件。因此脚本只选择性挂载几类 agent 配置。

| Host 路径 | 容器路径 | 权限 | 目的 |
|---|---|---|---|
| `/home/descfly/.claude` | `/home/descfly/.claude` | `rw` | Claude Code 配置、登录态、插件状态 |
| `/home/descfly/.claude.json` | `/home/descfly/.claude.json` | `rw` | Claude Code 顶层配置 |
| `/home/descfly/.codex` | `/home/descfly/.codex` | `rw` | Codex 配置、状态、插件信息 |
| `/home/descfly/.agents` | `/home/descfly/.agents` | `rw` | 本地 agent skill/config |
| `/home/descfly/.orchestra` | `/home/descfly/.orchestra` | `rw` | 本地 orchestra skill/config |

`.claude/plugins/cache` 和 `.codex/plugins/cache` 不再被当成通用 host package cache 做额外挂载；它们跟随 `.claude` 和 `.codex` 的现有挂载策略。这样可以避免 `node_modules` 或 pnpm symlink 被错误拆成一堆只读 mount，也避免出现空 target mount spec。

## 4. 只读 host 工具挂载：保留必要能力，不扩散 host 包环境

当前策略下，只读 host package 共享被压缩到最小集合。

| Host 路径 | 容器路径 | 权限 | 目的 |
|---|---|---|---|
| `/home/descfly/.local` | `/home/descfly/.host-home/.local` | `ro,rshared` | 复用 host 用户级主要包和命令入口 |
| `/opt/nvidia` | `/opt/nvidia` | `ro,rshared` | 复用 Nsight Systems、Nsight Compute 等 NVIDIA 工具 |

以前常见的 `.cache`、`.cargo`、`.conda`、`.npm`、`.rustup`、`miniconda3`、`node_modules` 等只读共享现在默认不再启用。原因很直接：这些目录体量大、状态复杂，而且常常包含 host 专属路径、解释器版本或包管理器 metadata。容器里只读看到它们，很多时候并不比看不到更安全，反而容易让 PATH 或运行时状态变得不清楚。

`~/.local` 是保留项，因为它经常承载用户级安装的命令或 Python 包入口；但它不会覆盖容器自己的 `${HOME}/.local`，而是挂到：

```text
/home/descfly/.host-home/.local
```

也就是说，容器自己的 HOME 仍然是可写 cache HOME，host `.local` 只是一个只读旁路。

## 5. Skill 只读挂载：共享能力，不共享写权限

脚本会扫描 `.claude`、`.codex`、`.agents`、`.orchestra` 中的 symlink。如果 symlink 指向 skill root，就把对应 skill 根目录只读挂进容器。

规则是：

```text
只挂 skill root
跳过非 skill symlink target
跳过已经在 .claude/.codex/.agents/.orchestra 挂载覆盖内的目标
```

这保证了两件事：

- 容器内 Claude/Codex 能解析绝对 symlink 或外部 skill 路径；
- 容器不会因为 root 权限修改共享 skill 仓库。

一个固定的共享 skill 路径也会在存在时挂载：

```text
/data3/paper_analysis/.claude/skills -> /data3/paper_analysis/.claude/skills:ro
```

这类挂载的定位是“读能力”，不是“写配置”。如果需要修改 skill，应在 host 或对应项目仓库中明确编辑，而不是让容器启动脚本默认开放写权限。

## 6. 不直接挂载 `.bashrc`：只提取 Claude 环境变量

host `.bashrc` 通常会包含 conda init、CUDA PATH、cargo、bun、代理函数、输入法变量等宿主机初始化逻辑。直接把它挂进容器会带来两个问题：

- 容器 Bash 会 source host 初始化逻辑，可能覆盖镜像里的 CUDA、conda、PATH；
- `.bashrc` 里很多路径只在 host 上成立，进入容器后反而产生错误状态。

因此当前脚本不再执行：

```text
/home/descfly/.bashrc -> /home/descfly/.bashrc:ro
```

它改为每次启动前生成容器自己的 `.bashrc`，只抽取和 Claude Code 相关的 export 行：

```text
ANTHROPIC_*
CLAUDE_CODE_*
MAX_MCP_OUTPUT_TOKENS
```

生成位置是：

```text
/data3/docker_cache/Qwen35_27B_on_2GPU/home/descfly/.bashrc
```

容器里看到的路径仍然是：

```text
/home/descfly/.bashrc
```

但内容来自脚本按关键词提取后的最小配置，不包含 host `.bashrc` 的全部内容。

## 7. PATH：只放容器路径和明确旁路

容器初始 PATH 由脚本显式设置。关键点是不要出现真实 host HOME 的可写路径：

```text
不包含：/home/descfly/.local/bin
包含：  /home/descfly/.host-home/.local/bin
```

默认 PATH 结构是：

```text
<detected /opt/nvidia tool dirs>
/home/descfly/.host-home/.local/bin
/data3/docker_cache/Qwen35_27B_on_2GPU/npm-global/bin
/opt/conda/bin
/opt/conda/condabin
/usr/local/cuda-12.9/bin
/usr/local/cuda/bin
/usr/local/bin
/usr/bin
/bin
```

其中 `/opt/nvidia` 部分是按 host 上实际存在目录动态生成的，例如：

```text
/opt/nvidia/nsight-systems/<version>/bin
/opt/nvidia/nsight-compute/<version>
```

启动命令进入容器后还会执行 CUDA 选择逻辑。默认 `CUDA_DEFAULT=system`，脚本会选择镜像内可用的 CUDA toolkit，并将 `${CUDA_HOME}/bin` 再 prepend 到 PATH。

## 8. 运行时访问示例

启动容器：

```bash
bash docker/run_project_gpu.sh
```

如果已有旧容器，需要重建才能应用新的 mount：

```bash
RECREATE=1 bash docker/run_project_gpu.sh
```

进入容器后检查项目、模型、cache：

```bash
docker exec -it Qwen35_27B_on_2GPU_dev /bin/bash

pwd
ls /workspace/Qwen35_27B_on_2GPU

echo "$MODEL_DIR"
ls "$MODEL_DIR"

echo "$HF_HOME"
echo "$NPM_CONFIG_PREFIX"
```

检查 host 只读旁路和 PATH：

```bash
ls ~/.host-home/.local
printf '%s\n' "${PATH//:/\\n}"

case ":$PATH:" in
  *":$HOME/.local/bin:"*) echo "unexpected host HOME local path" ;;
  *) echo "real HOME .local/bin is not in PATH" ;;
esac
```

检查 Claude/Codex 配置：

```bash
ls ~/.claude
ls ~/.codex
which claude
which codex
```

检查 `.bashrc` 只包含提取后的 Claude 相关 export：

```bash
sed -n '1,80p' ~/.bashrc
```

检查 NVIDIA 工具：

```bash
ls /opt/nvidia
which nsys || true
which ncu || true
nvidia-smi
```

## 9. 这个 skill 的边界

`project-docker-runner` 的输入只有一个必填项：镜像名。它不应该替用户猜镜像、不应该在生成时随意改 Docker runtime shape，也不应该把当前 host 的所有便利路径都塞进容器。

它负责的是把项目级 GPU 开发容器的启动策略固定下来：

- 项目和容器 cache 可写；
- 模型目录项目级可写；
- Claude/Codex 状态可写；
- host HOME 不整体挂载；
- host `.local` 和 `/opt/nvidia` 只读旁路；
- skill root 只读共享；
- `.bashrc` 只提取 Claude 环境变量；
- PATH 不混入真实 host HOME 的 `.local/bin`。

这套规则的目标是让容器足够好用，同时让“容器 root 修改 host 环境”的风险停在明确授权的目录边界内。
