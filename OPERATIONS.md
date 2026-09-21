# SEUSC 集群使用手册：人和 CLI agent 都可直接使用

本文是独立操作手册，**无需安装 Skill、无需阅读 SKILL.md，也无需使用 Codex 或 MCP**。只要能读取文本、创建本地文件并执行终端命令，就可以按本文使用集群。完整模板也直接列在文末，只有这一个文档时仍可使用。

## 阅读导航

- [先理解在哪里执行命令](#execution-model)
- [首次运行：从本地文件到下载结果](#first-job)
- [必须遵守的平台规则](#platform-rules)
- [按需查阅完整操作章节](#operations)
- [完整 CPU/GPU 提交模板](#templates)
- [给任意 agent 的任务描述和交接记录](#agent-handoff)

后续操作章节涵盖连接与账户、资源选择、存储、传输、编码、软件环境、提交、多进程、交互调试、监控恢复、故障处理和事实来源。

<a id="execution-model"></a>
## 先理解在哪里执行命令

```text
本地电脑（你的代码和 agent）
  ├─ ssh seusc ── 本机 SEU SC Bridge ── 集群登录节点
  │                                      └─ sbatch/srun ── 计算节点
  └─ scp / sftp ── 本机 SEU SC Bridge ── 共享文件存储
```

| 位置/概念 | 作用 | 怎么识别 |
| --- | --- | --- |
| 本地终端 | 准备文件、上传下载、发起 SSH | Windows PowerShell 或 macOS/Linux shell |
| 登录节点 | 查看集群、准备环境、提交和管理作业 | `ssh seusc hostname`，例如 login01 |
| 计算节点 | 实际执行训练、仿真、编译等计算任务 | Slurm 分配；在作业内用 hostname 查看 |
| 共享目录 | 保存代码、数据、环境和结果，供节点访问 | 以远端 pwd / $HOME 和实际挂载为准 |
| Partition | 资源分区/队列，决定可选节点类型 | sinfo / scontrol show partition |
| Account / QoS | 账户关联、资源和调度策略 | sacctmgr；不等于本机用户名 |
| Job ID | Slurm 作业编号 | 实际提交确认返回，用于后续查询 |

本机 `seusc status` 是桥接客户端命令；远端 `sbatch` 是集群调度命令，两者运行位置不同。每次 `ssh seusc '命令'` 是独立会话，前一次 cd、module load、conda activate 不会自动保留到下一次。把依赖步骤放在同一个短命令中，或写入上传的脚本。

本文带 `powershell` 的代码块在本地 Windows 执行；标明“集群内”的 Bash 代码在交互 SSH 或已上传的远端脚本中执行。`JOBID`、`YYYY-MM-DD`、`实际节点名` 是占位符，必须换为真实返回。不要把文档中的历史账户、预计开始时间当作当前事实。

<a id="first-job"></a>
## 首次运行：从本地文件到下载结果

此示例只输出主机、时间和资源信息，不依赖 Python。**第 4 步会实际申请 1 CPU、最多 2 分钟的作业**；若只需检查连接/提交配置，完成第 3 步即可。示例步骤相互依赖：任何一步失败先解决，不能继续当作成功；SSH 提交响应不确定时先查重，不重新提交。

### 1. 检查本地连接和远端入口

安装并登录支持 SFTP 的 [SEU SC Bridge](https://github.com/PureStudyer/SEU-SC-Bridge)，满足学校网络/VPN条件。在本地执行（PowerShell 和 POSIX shell 均可）：

```sh
ssh -o BatchMode=yes -o ConnectTimeout=15 seusc 'hostname; whoami; pwd; type sbatch; sinfo --version'
ssh seusc 'sinfo -p normal_test; squeue -u $(id -un)'
```

预期看到远端登录节点、自己的集群用户名和家目录、Slurm版本，以及 sbatch 的真实入口。本文命令中的 `/usr/bin/sbatch_wrapper` 是实测路径，若 `type sbatch` 显示不同，使用当前平台指定入口。认证失败时在桥接应用中完成登录；不从 agent 中读取登录凭据。

### 2. 在本地创建脚本和唯一工作目录

使用编辑器/agent 的文件写入能力，将下列内容保存为本地 `smoke.slurm`，采用 UTF-8 无 BOM、LF 换行。它是完整文件，不需要从其他仓库下载模板。

```bash
#!/bin/bash
#SBATCH --job-name=seusc-smoke
#SBATCH --partition=normal_test
#SBATCH --nodes=1
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=1
#SBATCH --time=00:02:00
#SBATCH --output=%x-%j.out
#SBATCH --error=%x-%j.err

set -e
cd "$SLURM_SUBMIT_DIR"
printf 'job=%s node=%s cpus=%s\n' "$SLURM_JOB_ID" "$(hostname)" "$SLURM_CPUS_PER_TASK"
date -Is
```

在包含 smoke.slurm 的本地目录操作，选择自己系统的一组命令。

**Windows PowerShell：**

```powershell
$run = 'smoke-' + [DateTime]::UtcNow.ToString('yyyyMMddTHHmmssZ') + '-' + [Guid]::NewGuid().ToString('N').Substring(0,8)
# $run 由上面一行生成，只含安全字符；不要直接用未经转义的任意输入替代
ssh seusc "mkdir -p ~/jobs && mkdir ~/jobs/$run"
if ($LASTEXITCODE -ne 0) { throw '创建远端目录失败' }
scp ./smoke.slurm "seusc:jobs/$run/smoke.slurm"
if ($LASTEXITCODE -ne 0) { throw '上传失败' }
New-Item -ItemType Directory -Path "./downloads/$run" -Force | Out-Null
$run | Set-Content -Encoding ascii ./last-run.txt
```

**macOS / Linux 本地 shell：**

```bash
run="smoke-$(date -u +%Y%m%dT%H%M%SZ)-$$"
ssh seusc "mkdir -p ~/jobs && mkdir ~/jobs/$run" || exit 1
scp ./smoke.slurm "seusc:jobs/$run/smoke.slurm" || exit 1
mkdir -p "./downloads/$run"
printf '%s\n' "$run" > ./last-run.txt
```

如果复制到交互终端，失败后不要继续下一段。`jobs/...` 是 SFTP 家目录下的相对路径，与 SSH 中的 `~/jobs/...` 对应。`mkdir` 的最终目录不带 -p，同名冲突就换 run-id，不覆盖已有运行。

### 3. 检查语法和调度请求

以下命令在本地执行，两种 shell 均使用上一步的 `$run`：

```sh
ssh seusc "cd ~/jobs/$run && bash -n smoke.slurm && /usr/bin/sbatch_wrapper --test-only --job-name=$run smoke.slurm"
```

检查命令退出码：PowerShell用 `$LASTEXITCODE`，POSIX shell用 `$?`。必须为0。`bash -n`验证shell语法；`--test-only`检查调度参数但不执行脚本。输出里的 `Job N to start ...` **不是提交确认**，不记录它为实际Job ID。预计开始日期可能不可靠。

### 4. 真正提交一次，保存 Job ID

仅在任务要求实际运行时执行：

```sh
ssh seusc "cd ~/jobs/$run && /usr/bin/sbatch_wrapper --job-name=$run smoke.slurm"
```

成功通常有 `Submitted batch job 123456`。记下该实际编号，保留原始返回、run-id、提交时间、目录。平台可能附加其他文字，不把整段输出当作数字。不要直接重复执行此步骤。

若终端超时或断开，用以下命令查询，再按操作章节的 sacct 历史查重；没有足够证据时报告“提交状态未知”：

```sh
ssh seusc "squeue --name=$run"
```

### 5. 查询运行状态和最终结果

把下面的 `JOBID` 替换为第 4 步确认的数字，在本地执行：

```sh
ssh seusc 'squeue -j JOBID'
ssh seusc 'scontrol show job JOBID'
ssh seusc 'sacct -j JOBID --format=JobID,JobName,State,ExitCode,Elapsed -P'
```

PENDING是排队，不是失败；RUNNING表示已经开始；COMPLETED且ExitCode为0:0才满足调度层成功条件。短作业可能很快从squeue消失，此时看sacct和文件；历史暂未出现就稍后再查，不重新提交。

本地把 `JOBID` 换成确认的数字，查看日志（本段保留已生成的 `$run`）：

```sh
ssh seusc "cd ~/jobs/$run && tail -n 40 $run-JOBID.out && tail -n 40 $run-JOBID.err"
```

输出应有实际计算节点名、Job ID、CPU数和时间。登录节点名不能被当成计算已经运行的证据；确认是作业日志中的输出。

### 6. 下载结果并核对

作业完成后，在本地执行：

```sh
scp "seusc:jobs/$run/$run-JOBID.out" "./downloads/$run/"
scp "seusc:jobs/$run/$run-JOBID.err" "./downloads/$run/"
ssh seusc "cd ~/jobs/$run && sha256sum $run-JOBID.out"
```

Windows本地比较 `Get-FileHash -Algorithm SHA256 "./downloads/$run/$run-JOBID.out"`；Linux用 `sha256sum`，macOS用 `shasum -a 256`。记录结果目录。不需要成功后立即删远端日志，排错和复现实验时仍有价值。

这套流程适用于真实任务：把 smoke.slurm 换成本文末尾的 CPU/GPU 模板，先上传代码和数据、配置依赖，再传入程序入口。复杂运行流程保存为 run.sh，交给模板执行 `bash run.sh`。

<a id="platform-rules"></a>
## 必须遵守的平台规则

本文已包含原 Skill 的操作约束，不需要额外读取 Skill 才能得到这些规则。

| 规则 | 对操作的影响 |
| --- | --- |
| 登录节点不运行计算 | 训练、仿真、重编译等通过 sbatch/srun 分配资源 |
| --wrap 被禁用 | 提交真实脚本文件，不能用 sbatch --wrap |
| --mem 被拒绝 | 按CPU核数自动配内存，查看DefMemPerCPU；高内存任务换分区/规划核数 |
| normal_test 默认且限时 | 正式任务明确选择其他分区；平台另提示每30分钟清理测试任务 |
| sbatch 有平台wrapper | 保留配额检查，不调用裸sbatch绕过平台入口 |
| GPU按卡配CPU | V100每卡3核、gpuB每卡12核；不自行覆盖CUDA_VISIBLE_DEVICES |
| shell状态不跨会话持久化 | 在同一脚本里cd、加载module、激活Conda、启动程序 |
| stdin管道不受支持 | 本地文件通过scp/SFTP传输，不使用cat管道或rsync传输 |
| exec约3KB、输出合流 | 长逻辑写成脚本上传，处理ANSI/CRLF；不能假定独立stderr |
| 仅现代scp/SFTP | 不用scp -O、-p或追加/续传、链接创建/完整POSIX语义 |
| 本机暂存传输文件 | 留出空间，等100%之后的合并/关闭与成功返回 |
| 不支持完整SSH转发 | 无ssh -L/-R/-D、agent/X11转发或VS Code Remote SSH |
| 提交响应不确定先查重 | 唯一名称+时间+目录核对squeue/sacct，不盲目再提交 |

这些是2026-09-21/22的实测与文档事实，运行时查询优先。dry-run、实际提交、程序完成是不同状态。

<a id="operations"></a>
## 完整操作章节

后面章节中的 demo/run-001、demo-env、train.py 等是说明性示例；它们不是首次运行示例自动创建的文件。应用到当前任务时，统一替换为实际的路径、环境、程序和Job ID。每一步检查退出码，依赖上一步成功后继续。

## 1. 连接与账户

本机需安装并登录 SEU SC Bridge，并具有校园网/VPN访问条件。OpenSSH 提供 ssh/scp/sftp。先查：

```powershell
ssh -G seusc
ssh -o BatchMode=yes -o ConnectTimeout=15 seusc 'hostname; whoami; pwd; type sbatch; sinfo --version'
ssh seusc 'sacctmgr -nP show assoc where user=$(id -un) format=Cluster,Account,User,Partition,QOS,DefaultQOS'
ssh seusc 'squeue -u $(id -un)'
```

`ssh -G` 的 user 是桥接用户，不能代替集群 `whoami`。不写死以前账户的家目录或 Slurm Account。多账户时选择用户要求的账户，并在提交参数指定 `--account=实际账户`。查询不到关联或限制时报告可见性问题，不将空值解释成无限制。

连接问题使用 `seusc status`、`seusc doctor`。Windows 未加入 PATH 时通常可用：

```powershell
& "$env:LOCALAPPDATA\Programs\SEUSC\seusc.exe" status
```

macOS 通常为 `/Applications/SEU SC Bridge.app/Contents/MacOS/seusc`。先确认安装路径。验证码/失效登录由用户在应用中处理，不读取密码、Cookie、Bearer 或浏览器 profile。重启会影响活动会话，仅在故障需要时使用；例行检查不 logout、不删除状态、不覆盖 SSH 配置。

用户只要求查询、准备脚本或写文档时不实际启动计算。用户要求运行明确任务时，完成必要的准备、传输、提交和结果检查，无需反复确认；资源规模不清楚且影响成本/可行性时先检查程序和输入，再询问关键缺失信息。

## 2. 资源选择

```powershell
ssh seusc 'sinfo -o "%P %a %l %D %c %m %G"'
ssh seusc 'sinfo -o "%P %D %t %C"'
ssh seusc 'scontrol show partition 6240-36C-192G'
```

`%C` 是 allocated/idle/other/total CPU。节点数量包括不可用节点；CPU 空闲不代表 GPU 空闲。需要 GPU 详情时用 `sinfo -N -p gpu_v100` 选出实际节点名，再 `scontrol show node 节点名` 查 GRES 配置和已分配资源。在已分配 GPU 作业内用 `nvidia-smi` 确认实际设备。

按账户查询出的 QoS 检查限制，例如集群内：

```bash
# 将 ogsp 替换为自己的实际 QoS
sacctmgr -nP show qos where name=ogsp format=Name,MaxWall,MaxTRESPerJob,MaxTRESPerUser,MaxJobsPerUser,MaxSubmitJobsPerUser
```

参考分区（2026-09-22 查询，不固定节点数或长期可用性）：

| 需求 | 分区候选 | 注意 |
| --- | --- | --- |
| 快速验证 | normal_test | 默认分区、最长30分钟；平台另提示定期清理任务 |
| 常规 CPU | 6240-36C-192G、6126-24C-192G、2680v4-28C-256G | 根据核数、内存和排队选择 |
| 海光 CPU | 7495-128C-384G | 检查二进制/库适配；部分模块有 hygon 版本 |
| 大内存 | 7542-64C-512G、6126-24C-768G、9462-64C-1024G | 查看实际 DefMemPerCPU |
| V100 GPU | gpu_v100 | 每 GPU 自动分配3核CPU |
| 大显存 GPU | gpuB | 每 GPU 自动分配12核CPU；官方描述为单机任务 |

实际型号、内存和显存不能仅由分区名字推断。官方文档描述 V100 每卡32G显存、gpuB 每卡80G；当前 Slurm GRES 只显示通用 gpu，不能据此认定 gpuB 的具体型号。

`--mem` 被拒绝，内存跟随 CPU 核数自动配置。高内存任务依据 `DefMemPerCPU` 估算核数或选择大内存分区，不把整机内存当作每个作业额度，也不改用其他内存选项绕过政策。不要无需求地 `--exclusive` 占整节点。

官方说明 gpuB 节点无法访问外网，其他节点也受白名单/校园网络条件限制。代码、包、数据和模型提前准备到共享存储；只针对任务所需地址验证连通性，不假设 GitHub/Hugging Face 等一直可达。

## 3. 存储与目录

```powershell
ssh seusc 'pwd; df -h "$HOME"; ls -ld /seu_share /seu_share2 /seu_nvme'
```

家目录以 pwd 为准，不能据旧用户名拼接。其他存储不一定已有当前用户可写目录。`df` 表示整个文件系统空间，不是组/个人剩余额度。官方默认政策为 share 组免费2T、nvme组免费500G，实际配额需在网页“团队管理 → 配额管理”核实。曾在 login01 上遇到 mmlsquota 报 GPFS daemon 不可用，这不证明共享存储不可访问，不尝试管理或修复集群文件系统。

每次运行使用独立工作目录，例如 `~/jobs/<project>/<run-id>`。环境和大型只读输入可复用，不必每次复制。记录源码版本/哈希、输入、资源、目录、提交时间、Job ID及结果路径。示例选择一个尚未存在的 run-id：

```powershell
ssh seusc 'mkdir -p ~/jobs/demo && mkdir ~/jobs/demo/run-001'
```

第二个 mkdir 不带 -p，避免静默复用已有目录。用户要求继续现有任务时可复用其目录，不强制新建。

## 4. 文件传输和完整性

### 单文件

```powershell
scp .\train.py seusc:~/jobs/demo/run-001/train.py
scp .\job.slurm seusc:~/jobs/demo/run-001/job.slurm
scp seusc:~/jobs/demo/run-001/result.json .\downloads\result.json
```

先确保源文件和目标父目录存在；本地下载目录要先创建。scp 可覆盖同名文件，选择独立结果目录或确认覆盖属于当前任务。现代 scp 默认使用 SFTP，不加 `-O` 或 `-p`。

### 多文件项目

示例文件必须存在，只打包本次任务需要的输入：

```powershell
tar -czf payload.tar.gz train.py requirements.txt configs
tar -tzf payload.tar.gz
scp .\payload.tar.gz seusc:~/jobs/demo/run-001/payload.tar.gz
Get-FileHash -Algorithm SHA256 .\payload.tar.gz
ssh seusc 'cd ~/jobs/demo/run-001 && sha256sum payload.tar.gz'
# 比较哈希后再解压
ssh seusc 'cd ~/jobs/demo/run-001 && tar -xzf payload.tar.gz'
```

不默认打包 .git、凭据、登录状态、无关数据和本地虚拟环境。外来归档先检查绝对路径、..、链接等条目，在独立目录解压。Windows Conda/venv通常不能直接复制到Linux运行。

目录传输优先归档，不把 scp -r 当作完整权限、链接和文件系统镜像。需要执行位/链接时可在合适系统生成 tar，再远端解包；SFTP本身不提供完整POSIX语义。大型归档的高负载压缩也应申请计算资源。

### SFTP批处理

本机创建 UTF-8无BOM的 transfer.sftp，Windows本地路径用正斜杠，空格路径加引号。例如上传阶段：

```text
put "D:/work/train.py" "jobs/demo/run-001/train.py"
bye
```

```powershell
sftp -b .\transfer.sftp seusc
if ($LASTEXITCODE -ne 0) { throw 'SFTP failed' }
```

结果阶段另建包含 `get "jobs/demo/run-001/result.json" "D:/work/downloads/result.json"` 的批处理。支持 put/get/ls/mkdir/rename/rm/rmdir；不使用追加、续传、chmod、保留权限/时间戳、链接创建。关键命令不加忽略失败前缀。

文件在桥接本机暂存，预留与文件相当的空间；100%后可能仍在远端合并/改名，等待关闭确认与成功退出码。内置下载大小校验不是内容哈希验证，关键输入/结果要比 SHA-256。正在写的日志可能因变化导致下载失败，可先 SSH tail 或等任务结束。

## 5. 编码、命令和退出码

远端命令外用单引号，避免 PowerShell 展开 Bash 变量：

```powershell
ssh seusc 'printf "%s\n" "$HOME"; squeue -u $(id -un)'
```

动态路径按对应 shell 转义；JSON.stringify不是shell转义。复杂逻辑生成本地脚本再上传，避免约3KB的exec长度限制。stdin管道不可用，不用 `Get-Content | ssh`、`cat | ssh`、`ssh ... < script.sh` 或 rsync。远端命令内部使用管道则不受此限制。

shell/slurm 文件用UTF-8无BOM、LF。PowerShell5.1默认编码可能不符，写文件时指定：

```powershell
# $content 是已准备好的脚本文本，$path 是本地绝对输出路径
$utf8 = New-Object System.Text.UTF8Encoding($false)
[IO.File]::WriteAllText($path, $content.Replace("`r`n", "`n"), $utf8)
```

`sbatch job.slurm` 或 `bash run.sh` 不需要执行位；必须直接运行时，远端对该文件 chmod。macOS/Linux本机同样用scp上传脚本和短ssh命令调用即可。

桥接返回远端命令退出码，但stdout/stderr合流且可能带ANSI/CRLF。解析前清理显示控制字符，同时保留原始返回。多步骤依赖用 `&&` 或脚本正确传播失败；不能仅看最后一个无关查询成功就认为前面提交成功。

## 6. 软件和环境

```powershell
ssh seusc 'MODULES_PAGER=cat module -t avail 2>&1'
ssh seusc 'MODULES_PAGER=cat module show anaconda3-2024.10-1 2>&1'
ssh seusc 'ls /seu_share/home/examples'
```

现有模块包括Anaconda、CUDA、GCC、OpenMPI、Intel/oneAPI、Singularity及科研软件；名称/版本要现查。官方python_cuda.sh曾引用已下线模块，不原样照搬。科研软件优先参考其对应样例的模块组合和MPI启动方式。

### Python环境（集群内）

优先查找已有适合环境，新建时遵循项目Python/依赖版本。下面demo-env、Python3.11均为例子：

```bash
module load anaconda3-2024.10-1
source "$(conda info --base)/etc/profile.d/conda.sh"
conda env list
conda create -n demo-env python=3.11 -y
conda activate demo-env
python -m pip install -r ~/jobs/demo/run-001/requirements.txt
```

检查 conda envs_dirs/pkgs_dirs 位于用户可写共享存储；不要修改共享base环境。编译大量扩展、需要GPU检测的安装应在分配资源后执行。不要默认修改.bashrc；普通CLI流程在作业脚本中初始化即可，网页环境选择器有需要时才按官方说明配置。

CPU/GPU模板支持 `SEUSC_CONDA_ENV` 环境名或路径；未设置时使用模块默认Python，不保证有项目依赖。作业中显式激活环境，因为sbatch_wrapper可能清理提交进程继承的Conda状态。安装PyTorch等遵循项目锁定版本、驱动和设备兼容性，不盲目组合CUDA/GCC模块。

自己安装软件用用户共享目录，如 `cmake -DCMAKE_INSTALL_PREFIX=...` 或 `./configure --prefix=...`；无root权限，不执行sudo/yum/apt安装系统包。编译密集任务通过Slurm运行，之后显式设置PATH/库路径。

### 容器

平台网页有Enroot镜像/模型开发入口；命令行可查Singularity模块，二者格式不同，不把Enroot镜像当作.sif。以下是已分配GPU的作业脚本示例，模块/镜像须先确认：

```bash
module load singularity-4.1.2
singularity exec --nv --bind "$PWD:/work" project.sif python /work/train.py
```

不假定root Docker daemon可用。网页实例的SSH/Jupyter入口与本地桥接不同，不能据此使用 `ssh -L` 或VS Code Remote SSH连接seusc。容器/MPI/深度学习环境仍需任务级运行验证。

## 7. 脚本、dry-run与提交

本仓库提供 `assets/smoke.slurm`（1CPU/2分钟验证）、`assets/cpu.slurm`（单进程4CPU）、`assets/gpu.slurm`（1V100/3CPU）。后两者接收命令argv，支持可选 `SEUSC_CONDA_ENV`。它们不写死Account、不指定内存、不发邮件。

复制模板为job.slurm，修改分区、资源、时间和唯一名称。`#SBATCH`应在第一个实际shell命令之前；指令里的$变量不会按普通shell展开，动态值用sbatch命令选项或预先生成具体内容。

本机示例：

```powershell
scp .\job.slurm seusc:~/jobs/demo/run-001/job.slurm
ssh seusc 'cd ~/jobs/demo/run-001 && bash -n job.slurm'
ssh seusc 'cd ~/jobs/demo/run-001 && /usr/bin/sbatch_wrapper --test-only --job-name=demo-run-001 job.slurm python -u train.py'
```

wrapper路径以 `type sbatch` 为准。不要用裸/usr/bin/sbatch绕过配额检查。test-only只验证调度参数，不运行脚本体，不验证依赖、数据、程序结果或保证计费状态。其 `Job N to start ...` 不是已提交作业，不是可靠的开始时间承诺。

用户要求运行后，提交一次：

```powershell
ssh seusc 'cd ~/jobs/demo/run-001 && SEUSC_CONDA_ENV=demo-env /usr/bin/sbatch_wrapper --job-name=demo-run-001 job.slurm python -u train.py'
```

保存完整返回，寻找 `Submitted batch job N` 并查询该ID。wrapper可能附带排队序号，不把整段输出当数字；--parsable也不能未经验证就假定纯数字返回。没有明确确认时按名称、时间、目录查重，不猜ID或盲目再次提交。

模板用 `"$@"` 执行argv。若程序需要管道/重定向/多步shell，另写run.sh，提交 `job.slurm bash run.sh`；不要把整段shell文本当成单个可执行参数。保留程序非零退出状态，不用 `|| true` 隐藏错误，自定义含管道脚本应正确处理管道失败。

## 8. 多核、批量、MPI和多GPU

- 单进程多线程：1 task、N cpus-per-task；模板设置OMP线程数，CPU模板另设置MKL/OpenBLAS，实际并行仍由程序实现。
- 独立批量任务：按需求设计array，先查支持和限制，以 `%并发数` 限流，不因输入多就自动申请大量资源。
- MPI：按对应软件样例选择MPI模块及mpirun/srun，申请实际task数量；只有程序支持跨节点时才增加nodes，运行库与编译库保持兼容。
- 单节点PyTorch DDP：2张V100可设计为 `--ntasks=1 --cpus-per-task=6 --gres=gpu:2`，由 `torchrun --standalone --nnodes=1 --nproc_per_node=2 train.py` 启动。程序必须支持DDP，这只是设计例子，本手册未实测训练。
- gpuB每卡12核，官方描述为单机任务。跨节点训练需单独核实网络、策略、程序支持与启动方式。

多进程/多节点不能直接照用单进程模板。新增资源配置先dry-run；不硬编码CUDA_VISIBLE_DEVICES、不手工挑选别人的GPU。

## 9. 交互调试

先在本机 `ssh seusc`，再在集群终端申请：

```bash
srun -p normal_test -N 1 -n 1 -c 1 -t 00:10:00 --pty /bin/bash
hostname
# 调试完毕释放allocation
exit
```

GPU用gpu_v100、1task、3CPU、--gres=gpu:1和短时限。agent须有交互PTY；无人值守任务优先sbatch。不直接ssh到计算节点占资源，不把登录节点作为训练或重计算节点。交互会话断开后先查自己的allocation，不能假定恢复原来的PTY。

## 10. 监控、恢复、取消与取回

```bash
# 集群内：JOBID替换为已确认的数字
squeue -u "$USER"
sjob -j JOBID
scontrol show job JOBID
sacct -j JOBID --format=JobID,JobName,State,ExitCode,Elapsed,AllocCPUS,MaxRSS -P
speek JOBID
speek -e JOBID
```

非交互环境没有shist alias时直接用sacct。查看MaxRSS时保留step记录；批作业行为空不代表内存使用为零。历史记账可能延迟或权限受限。

| 状态/现象 | 动作 |
| --- | --- |
| PENDING / Resources / Priority | 正常排队，检查请求和原因，不重复提交 |
| QOS/Account/Partition限制 | 查账户关联和策略，保留原报错 |
| 队列消失 | 查sacct与日志，不能当作成功 |
| COMPLETED + ExitCode 0:0 | 再核对预期结果文件及内容 |
| OUT_OF_MEMORY | 查step和内存配比，调整分区/核数/程序，不加--mem |
| TIMEOUT | 检查时限和测试分区清理；正式分区、checkpoint按需使用 |
| FAILED / 非零退出 | 查输入、依赖和错误日志，修复原因再重试 |

Python用-u或PYTHONUNBUFFERED避免输出缓冲。运行中日志用短SSH `tail -n 80 实际日志路径`，不启动无法结束的tail轮询。合理间隔查询；需要持续监控时用agent宿主可用机制，没有则说明最后观察到的状态，不承诺离开会话后自行通知。

Batch任务通常独立于SSH连接。若提交响应丢失，先查重：

```bash
squeue -u "$USER" -n demo-run-001
# 日期换为本次实际提交日期，必要时缩小窗口
sacct -S YYYY-MM-DD -u "$USER" --name=demo-run-001 --format=JobID,JobName,Submit,State,WorkDir -P
```

以唯一名称、时间和目录匹配。历史尚未出现不一定未提交，不能无依据重复提交；无法判断就报告不确定状态。

作业完成后下载已稳定产物和日志：

```powershell
scp seusc:~/jobs/demo/run-001/result.json .\downloads\result.json
scp seusc:~/jobs/demo/run-001/demo-run-001-JOBID.out .\downloads\
```

从scontrol/实际目录确认日志文件名，提前创建本地目录，关键产物验证SHA-256。用户要求或当前任务需要时仅 `scancel JOBID`；不用scancel -u取消其全部作业。清理仅针对明确属于本任务的文件，保留待诊断任务日志和输入。结束报告区分：已准备、dry-run通过、已提交、运行中、成功完成。

## 11. 故障速查

| 问题 | 处理 |
| --- | --- |
| --wrap被禁用 | 上传脚本文件提交 |
| --mem被拒绝 | 去掉，按CPU自动内存配比规划 |
| /bin/bash^M | LF、UTF-8无BOM重新上传 |
| Conda环境不存在 | 同一Anaconda模块，显式source conda.sh，查环境名/绝对路径 |
| CUDA不可用 | 在分配到的节点查设备/包，不覆盖CUDA_VISIBLE_DEVICES |
| SFTP subsystem不可用 | 确认运行中桥接已升级，不回退scp -O |
| 100%后等待 | 等上游合并关闭及成功退出码 |
| 磁盘满/配额超限 | 检查本机暂存与远端组配额，不只看df |
| 下载size mismatch | 文件仍在写，SSH tail或等结束 |
| 权限/时间戳设置失败 | 去除保留属性选项，必要chmod远端执行 |
| 欠费/组配额报错 | 平台核实，不绕过wrapper |
| bashrc造成协议异常 | 避免无条件输出，不覆盖用户现有配置 |

## 12. 来源与验证边界

- 2026-09-22检查了[SEU SC Bridge](https://github.com/PureStudyer/SEU-SC-Bridge) 的 README、docs/guide.md 及 internal/sftpserver/server.go。SFTP通过Finder网页接口，SSH命令通过WebShell PTY。
- 2026-09-21/22查询login01与Slurm23.11.10，验证了账户查询、分区与模块发现，以及禁用参数/GPU核数配比的dry-run返回。后续以现场查询为准。
- [官方总说明](https://sc.seu.edu.cn/docs/)
- [CPU、Slurm、存储](https://sc.seu.edu.cn/docs/hpc/contents.html)
- [GPU、环境、容器](https://sc.seu.edu.cn/docs/ai/contents.html)
- [软件与样例](https://sc.seu.edu.cn/docs/software/contents.html)
- [常见问题](https://sc.seu.edu.cn/docs/faq/contents.html)
- [费用管理](https://sc.seu.edu.cn/docs/fee/contents.html)

完整工作流不等于所有科研软件、容器、多节点训练或费用策略已实测。具体任务仍需验证运行时与预期结果；不将文档设计示例标成已经执行成功。

### 原始模板与传输验证（2026-09-22）

使用当前运行中的桥接完成了4KiB随机二进制的scp上传、SFTP批量下载，本地原件、下载件及远端SHA-256一致。三个模板均通过远端bash -n与平台sbatch_wrapper --test-only。测试使用独立目录，文件已通过SFTP逐项删除并移除空目录；没有实际提交计算作业。

GPU dry-run曾给出异常遥远的预计时间，说明预计开始时间不能当作可靠承诺；本次只把返回用作调度请求校验，不声称GPU已分配或程序已运行。


<a id="templates"></a>
## 附录 A：完整 CPU/GPU 模板

以下代码块可直接另存为相应文件，UTF-8无BOM、LF换行。与本仓库 assets 中的模板一致；如果只拿到本手册，不需要安装Skill或读取其他文件。

### CPU 单进程模板：cpu.slurm

```bash
#!/bin/bash
#SBATCH --job-name=seusc-cpu
#SBATCH --partition=6240-36C-192G
#SBATCH --nodes=1
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=4
#SBATCH --time=01:00:00
#SBATCH --output=%x-%j.out
#SBATCH --error=%x-%j.err

set -e
cd "$SLURM_SUBMIT_DIR"
module load anaconda3-2024.10-1
if [ -n "${SEUSC_CONDA_ENV:-}" ]; then
    source "$(conda info --base)/etc/profile.d/conda.sh"
    conda activate "$SEUSC_CONDA_ENV"
fi
export OMP_NUM_THREADS="$SLURM_CPUS_PER_TASK"
export MKL_NUM_THREADS="$SLURM_CPUS_PER_TASK"
export OPENBLAS_NUM_THREADS="$SLURM_CPUS_PER_TASK"
export PYTHONUNBUFFERED=1
if [ "$#" -eq 0 ]; then
    echo 'Provide a command: sbatch cpu.slurm python -u main.py' >&2
    exit 2
fi
"$@"
```

本地上传cpu.slurm和项目文件后，在集群内的实际工作目录使用：

```bash
SEUSC_CONDA_ENV=demo-env /usr/bin/sbatch_wrapper --test-only --job-name=实际唯一名称 cpu.slurm python -u main.py
# 只有任务要求实际运行才执行下一行
SEUSC_CONDA_ENV=demo-env /usr/bin/sbatch_wrapper --job-name=实际唯一名称 cpu.slurm python -u main.py
```

先在第6章准备demo-env，或替换为已有环境。没有设置SEUSC_CONDA_ENV时只加载默认Anaconda，不代表依赖已经安装。4CPU只是初始例子，程序必须自己支持并行才能用满。main.py必须确实存在。

### 单 GPU 模板：gpu.slurm

```bash
#!/bin/bash
#SBATCH --job-name=seusc-gpu
#SBATCH --partition=gpu_v100
#SBATCH --nodes=1
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=3
#SBATCH --gres=gpu:1
#SBATCH --time=01:00:00
#SBATCH --output=%x-%j.out
#SBATCH --error=%x-%j.err

set -e
cd "$SLURM_SUBMIT_DIR"
module load anaconda3-2024.10-1
if [ -n "${SEUSC_CONDA_ENV:-}" ]; then
    source "$(conda info --base)/etc/profile.d/conda.sh"
    conda activate "$SEUSC_CONDA_ENV"
fi
export OMP_NUM_THREADS="$SLURM_CPUS_PER_TASK"
export PYTHONUNBUFFERED=1
nvidia-smi
if [ "$#" -eq 0 ]; then
    echo 'Provide a command: sbatch gpu.slurm python -u train.py' >&2
    exit 2
fi
"$@"
```

准备环境、代码、数据后，在集群内实际工作目录使用：

```bash
SEUSC_CONDA_ENV=demo-env /usr/bin/sbatch_wrapper --test-only --job-name=实际唯一名称 gpu.slurm python -u train.py
# 只有任务要求实际运行才执行下一行
SEUSC_CONDA_ENV=demo-env /usr/bin/sbatch_wrapper --job-name=实际唯一名称 gpu.slurm python -u train.py
```

GPU模板调用nvidia-smi并运行传入的命令，不会自动安装CUDA/PyTorch，也不会自动把普通Python程序改造成GPU程序。两张V100需按程序和第8章调整卡数、CPU数及torchrun启动方式；改gpuB时按每卡12CPU配置。

### SBATCH字段怎么选

| 字段 | 含义 | 实际选择 |
| --- | --- | --- |
| --job-name | 作业名 | 一次运行一个唯一名，便于查重 |
| --partition | 分区 | 正式CPU/GPU任务明确选择 |
| --nodes | 节点数 | 单机程序保持1 |
| --ntasks | 进程/task数量 | 普通单进程保持1；MPI按启动方式设计 |
| --cpus-per-task | 每个task的CPU | 多线程/数据加载需要多少就申请多少，受平台GPU配比约束 |
| --gres=gpu:N | GPU卡数 | CPU作业不写此项 |
| --time | 最大运行时限 | HH:MM:SS，按任务规模和队列限制设置 |
| --output / --error | 日志路径 | %x是作业名，%j是Job ID；日志目录必须提前存在 |
| --account | Slurm账户 | 有明确账户要求时，用实际关联中选出的值 |

`SLURM_SUBMIT_DIR` 是执行sbatch时的目录，不一定是脚本存放位置，所以必须先cd到任务目录再提交。模板的最后一行 `"$@"` 让程序退出状态成为脚本退出状态；不要在其后追加掩盖错误的成功命令。

<a id="agent-handoff"></a>
## 附录 B：给任意 agent 的任务描述和交接记录

无需技能系统的任务提示示例：

```text
先完整阅读本地 OPERATIONS.md，按其中的平台规则使用 ssh seusc。
项目目录：<本地路径>；程序入口：<命令>；依赖：<文件或已有环境>。
输入数据：<位置>；资源：<CPU核数/单卡或多卡/目标分区>；最长运行时间：<时限>。
本次要求：<仅准备并dry-run / 实际提交运行 / 排查已有Job ID>。
需要取回：<产物与本地保存位置>。
提交后保存Job ID和工作目录；响应丢失先查重，完成后检查日志和结果再下载。
```

文件/入口能从项目确定时，agent应自行检查，不要求用户重复填写所有信息。未知且影响资源成本或实验正确性的参数才需要澄清；不因本文包含提交命令就替用户启动未请求的计算。

建议每次运行保存普通Markdown记录（本地，或与项目日志同目录）：

```text
任务目标：
本地项目和源码版本/哈希：
远端用户 / Account / QoS：
远端工作目录 / 输入位置：
环境和依赖版本：
分区 / 节点 / tasks / CPU / GPU / 时限：
作业脚本与提交命令：
提交时间和时区：
提交原始返回 / 确认的Job ID：
最后查询时间 / State / ExitCode：
输出与错误日志路径：
产物、校验结果和本地下载位置：
尚未验证/待处理的问题：
```

用户已要求运行且资源清楚时，正常准备、上传、校验和提交不需要反复征求相同授权。失败重试需先确定原因；破坏现有输入、取消其他作业、扩大资源规模不属于默认的错误恢复。

中途停止时交接实际状态；“文件已上传”“dry-run通过”“已提交”“运行中”“已成功并取回结果”分别报告。没有后台监控能力时说明最后观察到的状态，不承诺离开当前会话后继续跟踪。
