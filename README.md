# SEUSC Cluster Skill

让 CLI agent 通过 `ssh seusc` 使用东南大学超算集群的完整 Skill。

依赖 [SEU SC Bridge](https://github.com/PureStudyer/SEU-SC-Bridge) 提供本地 SSH、SFTP/scp 入口，覆盖从准备代码到回收计算结果的完整流程。适用于能读取 Skill/Markdown、执行本地终端命令的 agent。

## 能做什么

- 检查登录、账户、Slurm 分区、QoS、软件模块和存储配额。
- 选择 CPU/GPU 资源，上传代码、数据和提交脚本。
- 准备 Conda、编译软件或容器环境。
- 编写、校验、提交和管理 Slurm 作业；提供 CPU/GPU 模板及 MPI/DDP 指引。
- 查看日志、分析失败、处理断线与不确定提交，避免重复创建作业。
- 检查完成状态、校验产物完整性并下载结果。

集群特有规则已纳入：禁止 `sbatch --wrap` 和 `--mem`、平台提交 wrapper、GPU/CPU 自动配比、测试分区限时，以及桥接的 stdin/PTY/文件传输限制。可变信息仍要求现场查询。

## 前提

1. 安装并登录支持文件传输的 SEU SC Bridge。
2. 满足学校网络/VPN访问条件，具有自己的集群账户。
3. 本机具备 OpenSSH，确认 `ssh seusc hostname` 可以返回登录节点。

本仓库不包含桥接程序、账号或登录凭据。创建 Skill、查看资源或校验脚本不会自动授权运行计算任务。

## 安装

安装到 agent 的 Skills 目录，文件夹名使用 `seusc-cluster`。以下是默认 Codex Skills 目录；若设置了 `CODEX_HOME`，请改用其下的 `skills` 目录。若目标已存在，请先检查现有版本，不要直接覆盖自定义内容。

Windows PowerShell：

```powershell
git clone https://github.com/PureStudyer/seusc-cluster-skill.git "$env:USERPROFILE/.codex/skills/seusc-cluster"
```

macOS / Linux：

```bash
git clone https://github.com/PureStudyer/seusc-cluster-skill.git ~/.codex/skills/seusc-cluster
```

也可下载 Release 压缩包，将其中的 `seusc-cluster` 文件夹放到对应 Skills 目录。重新打开或刷新 agent 会话后使用其 Skill 发现机制。

其他 CLI agent 可直接读取本仓库的 [SKILL.md](SKILL.md)，并按其中链接读取参考文档和模板；不依赖 Codex 专有工具或 MCP。

## 使用示例

```text
使用 $seusc-cluster，检查当前账户可以使用的 CPU/GPU 分区和队列情况。
```

```text
使用 $seusc-cluster，为当前项目准备单 GPU 作业，先检查依赖并做 dry-run，不实际提交。
```

```text
使用 $seusc-cluster，把当前项目上传到集群，使用一张 V100 运行训练，
时限两小时，检查作业结果并下载产物。
```

```text
使用 $seusc-cluster，检查作业 JOBID 失败的原因，修复后按原资源配置重跑。
```

## 文件结构

| 文件 | 用途 |
| --- | --- |
| [SKILL.md](SKILL.md) | agent 入口、关键约束与决策流程 |
| [references/workflow.md](references/workflow.md) | 完整操作参考：连接、资源、传输、环境、作业和排错 |
| [assets/smoke.slurm](assets/smoke.slurm) | 最小验证作业，1 CPU、2 分钟 |
| [assets/cpu.slurm](assets/cpu.slurm) | 单进程 CPU 模板，命令通过参数传入 |
| [assets/gpu.slurm](assets/gpu.slurm) | 单 GPU 模板，命令通过参数传入 |
| [agents/openai.yaml](agents/openai.yaml) | Skill 展示元数据 |

模板使用前应按任务调整分区、资源、时限和运行环境。模板本身不是适用于所有程序的性能配置。

## 验证范围

2026-09-22，在 Windows 上通过当前 SEU SC Bridge 和 SEU 登录节点验证：

- Skill 格式、文件链接、UTF-8 无 BOM / LF 格式检查通过。
- 4 KiB 二进制文件 scp 上传、SFTP 批处理下载；本地原件、远端文件、下载件 SHA-256 一致。
- 三个 Slurm 模板通过远端 `bash -n` 和平台 `sbatch_wrapper --test-only`。
- 测试文件已清理，没有实际启动计算作业。

CPU/GPU 程序运行、MPI/DDP、容器与科研软件仍需针对具体任务验证。分区、配额、模块和平台策略可能变化，使用时以实际查询及[官方文档](https://sc.seu.edu.cn/docs/)为准。

## 项目关系与许可

这是独立的集群使用 Skill；[SEU SC Bridge](https://github.com/PureStudyer/SEU-SC-Bridge) 是连接工具。两者均为非官方项目，未经东南大学官方背书。

[MIT License](LICENSE) · Copyright © 2026 PureStudyer
