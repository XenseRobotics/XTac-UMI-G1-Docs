# 版本与支持

本手册对应的版本基线、如何查版本、如何升级,以及遇到问题怎么反馈。

## 必须升级到最新版本 {#required}

!!! danger "这不是可选项——采集前请先把四项都升到下表版本"
    整条链路是**配套**的:固件带[命令集 V2.1](#v21) 才支持行程标定,SDK 0.1.7 才能安全刷这版固件,
    而 `gripper.pos` 的 0–1 归一化要固件、SDK、标定三者齐了才成立。主夹爪缺标定或固件过低时
    **采集程序会直接拒绝连接**,不会让你采出刻度错的数据;但版本不配套的其它组合仍可能
    **"正常"跑完并落盘**,只是数据和别人对不上——事后从数据里看不出来。

| 组件 | 最低要求版本 | 怎么查 |
|---|---|---|
| `xense-taccap-lerobot` | `0.5.1+xtac.0.0.4` | `pip show lerobot` 或看 `pyproject.toml` |
| `xense.taccap` SDK | **0.1.7** | `python -c "import xense.taccap as t; print(t.__version__)"` |
| 夹爪固件 | **命令集 V2.1**,即构建 leader **≥ 1.2.0** / follower **≥ 1.1.0**([区别](#v21)) | 跑 [`calibrate.py`](04-calibration.md#41),版本不够会打印当前版本并退出;想直接读版本见[下面这条命令](#v21) |
| 每台 leader 的编码器标定 | 零点 + 行程上限已写入 flash | [4.1 夹爪标定](04-calibration.md#41) |

### 三套编号:V2.1 是命令集,不是固件版本 {#v21}

夹爪固件涉及**三套互不相干的编号**,长得像但含义完全不同:

| 编号 | 当前值 | 是什么 |
|---|---|---|
| 帧格式(wire framing) | `V1.8` | 字节怎么打包成帧(字节填充、CRC)。极少变动,本手册基本不提 |
| **命令集(command set)** | **`V2.1`** | 固件**实现了哪些命令**。每条命令按引入时的版本标注 |
| 固件构建号 | leader `≥ 1.2.0`<br>follower `≥ 1.1.0` | 具体那一版镜像。**这是最小值,不是必须精确匹配的值**——1.2.1 之类的更高版本同样支持 V2.1,不需要为此回刷 |

**本手册里说的「V2.1」一律指命令集**,因为行程标定用的 `EncoderMaxCal` 命令正是 V2.1 引入的
(`V2.0` 引入鱼眼标定,`V1.9` 引入 LED 与私有电机参数)。所以"标定要求固件 ≥ V2.1"说的是
**命令集**要够新。

对应关系是**门槛**而非等号:**命令集 V2.1 从 leader `1.2.0` / follower `1.1.0` 起开始支持**,
更高的构建号(如 leader `1.2.1`)同样支持。所以看到自己是 1.2.1 不必疑惑,也不需要回刷到
1.2.0——**"我们跑的版本"和"我们要求的版本"是两件事**。主从两个号不同,只是因为它们本来就是
两份不同的固件,不是谁更新。

一条命令查完 SDK 版本和每只夹爪的固件构建号。**固件版本不在 SN 里**,SN 只给出侧别与角色,
版本要用 `GetVersion` 向固件问:

```bash
python - <<'EOF'
import xense.taccap as t
from xense.taccap import scan_grippers, LeaderGripper, FollowerGripper, Cmd
print("xense.taccap", t.__version__, "(需要 >= 0.1.7)")
for ep in scan_grippers():
    cls = LeaderGripper if ep.firmware_sn.endswith("m") else FollowerGripper
    g = cls(mcu_device=ep.mcu_device)          # 只开 MCU;相机默认不开
    ack = g.transport.send_cmd(Cmd.GetVersion, b"", 500)
    print(f"  {ep.firmware_sn}  {ep.side.name:5}  fw={ack.data[0]}.{ack.data[1]}.{ack.data[2]}")
EOF
```

```text
  TCGU01A28Z0023m  Left   fw=1.2.1
  TCGU01A28Z0024m  Right  fw=1.2.1
```

!!! note "本文出现的序列号和版本号都是**示例**"
    上面这两行是某台设备上的实际输出,写在这里只是让你知道**输出长什么样**。你手上这台的
    SN 一定不同,版本号也可能更高——本文后面凡是出现具体 SN(如 `TCGU01A28Z0023m`)的地方,
    都是同一个意思:那是举例,以你自己跑出来的结果为准。

    SN 里唯一需要你去读的是**最后一个字符**:`m` = 主夹爪,`s` = 从夹爪。这决定刷哪个镜像,
    见 [固件 OTA 升级](#ota)。

!!! note "为什么这样构造夹爪对象"
    只传 `mcu_device`:不开腕相机,`normalize_position` 也保持默认的 `False`——**没做过行程标定的
    夹爪照样能读到版本**,不会在构造时因为读不到 encoder-max 而抛错。这条也不依赖 SDK 的
    `examples/`,升级前后都能跑。

    ACK 里其实有**第 4 个字节 `build`**,固件把它恒定写 0,SDK 各处一律按
    `MAJOR.MINOR.PATCH` 显示。别把它写进版本比较——`1.2.1.0` 这种四段写法是过时格式。

**升级顺序不能乱**,每一步都依赖上一步:

```mermaid
flowchart LR
    A[拉仓库 + 子模块] --> B[重新编译 xense.taccap] --> C[刷固件 OTA] --> D[每台 leader 标定]
```

1. [拉仓库与子模块](#repo-update) —— 子模块要跟着一起更新。
2. **重新编译 SDK**,否则新旧文件不配套,`import xense.taccap` 直接失败。
3. [固件 OTA 升级](#ota) —— **必须在第 2 步之后**,理由见该节。
4. [夹爪标定](04-calibration.md#41) —— 升级本身**不产生标定值**,不标的主夹爪连不上。

!!! note "已经采过的数据不用重采"
    这里要求的是**今后采集**用统一版本。历史数据保持原样即可;若要和升级后的数据混用,
    先确认两批的 `gripper.pos` 是否落在同一刻度上(升级+标定前后不一定一致)。

## 版本兼容基线

本页将“支持范围”和“已验证基线”分开列出。实际命令与字段仍以本地 checkout 为准。

| 组件 | 支持范围 / 约束 | 已验证基线 |
|---|---|---|
| 操作系统 / 架构 | Ubuntu 22.04 / 24.04,**amd64** | Ubuntu 22.04.5 LTS / 24.04.4 LTS,x86_64 |
| Linux 内核 | 不构成约束 | 6.8 / 6.14 / 7.0 系列均已验证 |
| 采集主机(CPU / 内存 / 硬盘) | **最低**:12 代 i7、8 GB 内存、512 GB SSD;**推荐**:13/14 代 i7/i9、32 GB 内存、1 TB NVMe SSD。两档明细见[采集主机配置要求](02-environment.md#host-spec) | — |
| NVIDIA GPU / 驱动 | **最低 RTX 3060 / 8 GB 显存**(推荐 RTX 4070 / 12 GB 及以上),**驱动 ≥ 570.144**。没有 NVIDIA 显卡的机器只能[降级录制](05-data-collection.md#no-gpu),效率明显下降 | 驱动 570.144 / 580.126.09 |
| Python | **≥ 3.12**(`pyproject.toml` 的 `requires-python`;`conda_environment.yaml` 固定 `python=3.12`) | 3.12.13 |
| PyTorch | `torch>=2.2.1,<2.11.0`;`torchvision>=0.21.0,<0.26.0` | 2.10.0 / torchvision 0.25.0 |
| `torchcodec` | `>=0.2.1,<0.11.0`,由 `setup_env.sh` **按当前 torch 版本自动对齐**(不匹配会强制重装) | 0.10.0 |
| PyAV | `av>=15.0.0,<16.0.0`,安装脚本固定装 **15.1.0** | 15.1.0 |
| `rerun-sdk` | `>=0.24.0,<0.27.0`(`--display_data` 用) | 0.26.2 |
| `opencv-python` | 固定 `==4.12.0.88`(XenseRobotics 各 SDK 统一) | 4.12.0.88 |
| NumPy | `>=1.26.4` | 2.2.6 |
| `xense-taccap-lerobot` | 基于 lerobot 0.5.1 定制;版本号 `0.5.1+xtac.0.0.6`(与文档版本同步) | `main@a7b0d57b` |
| `xense.taccap`(`taccap-gripper` SDK) | 与主仓库子模块版本配套 | 0.1.7(子模块 `83314c8`) |
| 夹爪固件命令集 | **V2.1**(帧格式另计,为 V1.8;区别见[三套编号](#v21)) | 命令集 V2.1 |
| 夹爪固件构建 | leader **≥ 1.2.0** / follower **≥ 1.1.0** 即支持命令集 V2.1 | 当前基线附带 leader **1.2.1** / follower **1.1.1**(固件源码分支 `hw_v1.1.0`);镜像版本随 SDK 走,以 `firmware/manifest.json` 为准,见 [固件 OTA 升级](#ota) |
| `xensesdk` | 由安装脚本提供 | 2.1.1 |
| XenseVR PC Service(`.deb` 守护进程) | ≥ **v0.2.0**;**装机请直接用 v0.2.1**,理由见下 | v0.2.1 |
| `xensevr_pc_service_sdk`(Python 接口) | 绑定在主仓库内(不再是子模块),链接 `.deb` 里的 C SDK | 0.2.1 —— **版本号取自 `.deb`**,见下 |

!!! note "`.deb` 现在是 Pico4 C SDK 的唯一来源"
    以前为了编译 `libPXREARobotSDK.so` 要克隆 `third_party/XenseVR-PC-Service` 子模块;
    现在这个库直接从已安装的 `.deb`(`/opt/apps/roboticsservice/SDK`)里取,**子模块已删除**。
    带来两件事:

    - **递归克隆从约 33 MiB 降到约 1.6 MiB**,安装时也不再链接 gRPC 静态库。
    - **这个 C SDK 的更新随新版 `.deb` 分发**,重跑 `--install` 不会重新编译它。

    脚本会跳过"版本已经一致"的包,所以停在 **v0.2.0** 的机器会拿旧 SDK 去编绑定——
    v0.2.1 是同一个守护进程配**重新编译过的 SDK**,装机请直接用它。

!!! note "`pip show xensevr-pc-service-sdk` 跟着 `.deb` 走"
    这个包在构建时向 `dpkg` 问 `xensevr-pc-service` 的版本,所以显示的就是 `.deb` 的版本
    (如 `0.2.1`)。

    **判断有没有头显相机支持,请看接口在不在**——它反映的是实际加载的那个模块:

    ```bash
    python -c "import xensevr_pc_service_sdk as xrt; print(hasattr(xrt, 'has_pico_camera_frame'))"
    ```

    服务本体的版本用 `dpkg -s xensevr-pc-service` 查(见[如何查版本](#check-versions))。

!!! note "PC Service v0.2.0 只影响头显相机"
    v0.2.0 相对 v0.1.0 只多做一件事:**转发头显相机的视频帧**,这是
    [头显相机](05-data-collection.md#56)画面的通路。老版本头显 APK 配 v0.2.0 也照常工作,
    所以**不用头显相机就没有任何行为变化**。

!!! note "版本以本地为准"
    命令与字段应以你本地这一版主仓库、以及夹爪 SDK 附带的设备说明为准。

!!! note "头显相机需要 `ffc94d53` 之后的版本"
    本手册中的[头显相机](05-data-collection.md#56)、Insight 链路移除、以及
    [`/world` 3D 视图](05-data-collection.md#world-view)的改动,来自
    [PR #9](https://github.com/Vertax42/xense-taccap-lerobot/pull/9),已合入 `main`(`ffc94d53`)。

    checkout 停在更早的版本时:`--robot.enable_head_camera` 在单夹爪上还不存在,
    双夹爪上它指的是旧的 Insight 相机,Rerun 里也仍然画着 TRACKER 坐标系和虚线。
    执行 `git pull --recurse-submodules` + `./setup_env.sh --install` 即与本手册一致。

!!! warning "子模块已改用 HTTPS——`ffc94d53` 及更早的版本需要 GitHub SSH key"
    早期版本的 `.gitmodules` 用的是 `git@github.com:` 形式,机器上没配 GitHub SSH key 时
    [克隆仓库与子模块](02-environment.md) 那一步会失败(顶层仓库能克隆,拉子模块时报错)。
    现在已全部改为 `https://`,任何机器都能直接拉。Docker 路径同样要过这一关——镜像构建
    只校验子模块是否已就位,拉取仍由宿主机在构建前完成。

    **已经升级到新版本、但拉子模块仍然要 SSH key**:`.gitmodules` 只是模板,老机器
    `.git/config` 里记的还是旧 URL,`git pull` 不会改写它。同步一次即可:

    ```bash
    git submodule sync --recursive
    git submodule update --init --recursive --progress
    ```

    **必须停在旧版本时**,全局改写 URL 绕过:

    ```bash
    git config --global url."https://github.com/".insteadOf "git@github.com:"
    ```

!!! note "主夹爪拒绝连接的行为需要 `4fb5b79b` 之后的版本"
    从这一版起,**未标定的主夹爪会被直接拒绝连接**,不用你自己判断标没标过。
    停在更早的版本时没有这道拦截,**务必自己按 [4.1.3](04-calibration.md#41) 确认
    `gripper.pos` 张到底能到 `1.0`** 再开录。

!!! note "头显位姿进 action、显示默认不压缩,需要 `f491cae5` 之后的版本"
    两项变化都在这之后:`head_camera.*` 从"仅观测"变成**同时也是动作**
    (见 [5.6 位姿与可视化](05-data-collection.md#56));`--display_compressed_images`
    的默认值从 `true` 改为 **`false`**,并新增 `--display_image_every_n`。

    停在更早的版本时:头显位姿只作为观测落盘,策略不会被要求复现它;Rerun 显示默认走 JPEG
    压缩,开 `--display_data=true` 更容易出现 `[slow_frame]`。

!!! note "固件镜像 1.2.1 / 1.1.1 需要 `fc9e9b93` 之后的版本"
    子模块升到 SDK `83314c8` 后,`firmware/` 里附带的镜像是 leader **1.2.1** / follower
    **1.1.1**;更早的基线附带 1.2.0 / 1.1.0。**两者都支持命令集 V2.1,不需要为此重刷**
    ——1.2.1 只改了 LED 的颜色与闪烁周期。要求门槛仍是 leader ≥ 1.2.0 / follower ≥ 1.1.0。

!!! danger "`--robot.id` 变成必填,需要 `e8146c4e` 之后的版本"
    从这一版起,`lerobot-record` / `lerobot-teleoperate` **必须**带 `--robot.id`,
    漏了在解析命令行时就退出;录制时还会往数据集里写一份
    [`meta/hardware.json`](05-data-collection.md#robot-id)(夹爪固件 SN + 触觉 SN)。

    停在更早的版本时:`--robot.id` 可以不填,数据集里也**没有 `meta/hardware.json`**——
    那批数据说不出自己是哪套硬件采的,只能靠[采集台账](data-management.md)人工记。
    **本手册的命令一律按新版书写。**

!!! note "`--robot.id` 可以只填数字,需要 `04812536` 之后的版本"
    `--robot.id=0` 会按 `--robot.type` 自动补前缀(`taccap_0` / `bi_taccap_0`)。
    停在更早的版本时,前缀要自己写全:`--robot.id=taccap_0`。
    **两个版本都接受写全的形式**,所以本手册里的 `--robot.id=0` 在旧版本上要改写,反之不必。

!!! note "无 GPU 主机上的编码器选择,建议用 `3b9d2deb` 之后的版本"
    从这一版起,`--dataset.vcodec=auto` 通过**真开一次编码会话**来判断有没有硬件编码器,
    所以没有 NVIDIA 驱动的机器上会自动落到 `libsvtav1`,不需要指定。更早的版本上,
    这类主机请显式写 `--dataset.vcodec=libsvtav1`。见
    [没有 NVIDIA GPU 的主机怎么录](05-data-collection.md#no-gpu)。

!!! note "Pico4 的 C SDK 改由 `.deb` 提供,需要 `42b44066` 之后的版本"
    `third_party/XenseVR-PC-Service` 子模块在这一版删除,`libPXREARobotSDK.so` 改从
    `.deb` 里取,同时 `.deb` 基线提到 **v0.2.1**(见上面的基线表)。

    停在更早的版本时:克隆仍需拉这个子模块,`--install` 仍会现编译这个库;
    [2.2 克隆仓库与子模块](02-environment.md#22) 里"只有一个子模块"的说法对不上你的 checkout。

!!! note "`setup_env.sh` 的 apt 预检查需要 `2892929a` 之后的版本"
    从这一版起,`--install` 会先查 `build-essential` / `cmake` / `pkg-config` / `git` / `curl`
    在不在,缺了直接停下并打印该敲的 apt 命令(见
    [前置:系统依赖包](02-environment.md#apt))。停在更早的版本时没有这道检查,缺包要到
    编译阶段才报出来——照那一节先把包装齐即可。

!!! note "Docker 交付镜像需要 `9387ef05` 之后的版本"
    [2. 环境部署](02-environment.md#docker) 里那条 Docker 路径——交付目录、
    `install_customer.sh`、`compose.yaml`——来自这次提交,更早的版本里没有。

!!! note "Docker 改为默认从 GHCR 拉取,需要 `854d4cdf` 之后的版本"
    从这一版起 `compose.yaml` 的默认镜像就是 `ghcr.io/vertax42/xense-taccap-lerobot`,
    `.env` 里**只需要写 tag 一行**;`.tar` 离线包仍然支持,但不再是默认路径。

    停在更早的版本时:默认镜像是**本机构建**的名字,要拉 GHCR 就必须同时写
    `LEROBOT_IMAGE` 和 `LEROBOT_IMAGE_TAG` 两行,只改 tag 不生效。

!!! note "容器里的图形能力需要 `93beb2aa` 之后的版本"
    `compose.yaml` 从 `gpus: all` 改成了 `runtime: nvidia`。停在更早的版本时,容器能跑 CUDA、
    `nvidia-smi` 也正常,但 NVIDIA 的 Vulkan ICD 不会被注入,**Rerun 窗口起不来**
    (`Failed to create surface`)。处理见
    [容器里的图形界面](02-environment.md#docker-gui)。

    改用 `runtime: nvidia` 后要求 NVIDIA runtime 已注册进 Docker;没注册会直接报
    `Unknown runtime specified nvidia`,见[故障排查](troubleshooting.md#docker)。

!!! danger "语音播报不再中断录制,需要 `94597ba2` 之后的版本;`0.0.5` 镜像**早于**它"
    每集开始会语音播报一句,而播报用的 `spd-say` 不在镜像里。修复前:第一集播报就抛异常,
    收尾那句又抛一次,进程直接崩掉——**在容器里等于录不了**。修好后播报失败只告警一次。

    **`0.0.6` 镜像已经包含这个修复**,升级后不必再带 `--play_sounds=false`;仍然钉在
    `0.0.5` 及更早的机器请一律带上,见[故障排查](troubleshooting.md#docker)。
    Mamba 路径上装了 `speech-dispatcher` 的主机不受影响。

!!! note "容器里的语音播报是静音的,而且是刻意如此"
    `0.0.6` 的镜像装了 `speech-dispatcher`,但**没有装语音合成器模块**,所以播报只是被安全地
    跳过。补上合成器并不能让你听到声音——容器里没有可用的音频输出,`spd-say` 会从"立刻失败"
    变成"一直挂着",把录制卡在收尾那一步。为此 `d1ad7140` 之后的版本给这条阻塞调用加了 10 秒
    上限;`0.0.6` 镜像早于它,但只要不自己往镜像里装合成器就遇不到。**要听到提示音,在宿主机上录。**

!!! note "录出来的视频对非 root 可读,需要 `dac15f74` 之后的版本;`0.0.5` 镜像**早于**它"
    修复前视频落盘为 `-rw------- root`(拼接用的临时文件是 `0600`,移动时保留了权限),
    而同目录的元数据是正常 `0644`。后果只在**导出**时显现:非 root 拷贝时只有 `.mp4` 报
    `Permission denied`,看着像个别文件坏了。

    **`0.0.6` 镜像已经包含这个修复**。但权限取决于**当初录数据的那版镜像**,升级不会改写
    已经录好的文件——老数据仍按 [数据放在哪](02-environment.md#docker-data) 的写法
    **以 root 拷、拷完再 `chown`**。

!!! note "`LEROBOT_DATA_DIR` 把数据落到宿主机目录,需要 `89239c71` 之后的版本"
    在 `.env` 里设这一项,数据集就直接写进你指定的宿主机目录,不用再从容器里拷出来
    (见 [让数据直接落在宿主机目录](02-environment.md#docker-data-dir))。不设时行为与之前
    完全一致,仍是具名卷 `lerobot-data`。

    停在更早的版本时:只能用具名卷,想落到宿主机目录得改 `compose.yaml`——**不建议**,
    那是仓库跟踪的文件,写死绝对路径会在下次 `git pull` 冲突,换台机器还会挂到不存在的路径。

!!! note "头显相机默认 640x480,需要 `4b5f5cea` 之后的版本;`0.0.6` 镜像**早于**它"
    头显 APP 的「分辨率」有 `640` / `1024` / `1280` 三档,默认 `640`(每眼 640x480)——
    但采集端原先只认 `1024x768` 和 `1280x960`,默认 `1024x768`。也就是说,头显停在出厂默认时
    开 `--robot.enable_head_camera=true` 会因首帧尺寸不符连不上,`--robot.head_camera_width=640`
    也会被判错。`4b5f5cea` 把 640x480 加进白名单并设为默认,两边默认这才对得上。

    停在更早的版本(含 `0.0.6` 镜像)时:头显相机只能跑 `1024` 或 `1280`——**先在头显里把
    分辨率调上去**,`1024` 用命令行默认即可,`1280` 加
    `--robot.head_camera_width=1280 --robot.head_camera_height=960`。本页其余部分与
    [5.6 头显相机](05-data-collection.md#56)按新默认(640x480)描述。

!!! note "`--dryrun` 已删除,`d46fcf66` 之后的版本不再接受它"
    这个参数只打印一句"动作不会下发给机器人",实际从不生效——设备照常运行。已删掉,
    脚本里带着它的话现在会报未知参数,直接去掉即可。本手册从未使用过它。

## 如何查版本 {#check-versions}

一条命令把上表里跑在 Python 环境里的都打出来:

```bash
python - <<'EOF'
import importlib.metadata as M
for p in ("lerobot", "taccap-gripper", "xensesdk", "torch", "torchvision",
          "torchcodec", "av", "rerun-sdk", "opencv-python", "numpy"):
    try:
        print(f"{p:16} {M.version(p)}")
    except M.PackageNotFoundError:
        print(f"{p:16} 未安装")
EOF
```

```bash
# xense-taccap / SDK
python -c "import xense.taccap as t; print('xense.taccap', t.__version__)"
python -c "import xensesdk, xensevr_pc_service_sdk; print('xensesdk/pc_service OK')"

# NVIDIA 驱动(装了 NVIDIA GPU 时需 >= 570.144)
nvidia-smi --query-gpu=driver_version,name --format=csv,noheader

# XenseVR PC Service 守护进程的 deb 版本(服务本体的版本以这个为准)
dpkg -s xensevr-pc-service 2>/dev/null | grep -E '^(Package|Version|Architecture):'
# 是否带头显相机接口(需要 PC Service v0.2.0;包版本号不会变,只能看接口)
python -c "import xensevr_pc_service_sdk as xrt; print('pico camera API:', hasattr(xrt, 'has_pico_camera_frame'))"

# 固件 SN(含固件方案信息;role/side 由 SN 解析,但 SN 里没有版本)
python -c "from xense.taccap import scan_grippers
for g in scan_grippers(): print(g.side.name, g.role.name, repr(g.firmware_sn))"

# 夹爪固件构建号(向固件问 GetVersion;不依赖 SDK 的 examples/)
python -c "
from xense.taccap import scan_grippers, LeaderGripper, FollowerGripper, Cmd
for ep in scan_grippers():
    cls = LeaderGripper if ep.firmware_sn.endswith('m') else FollowerGripper
    g = cls(mcu_device=ep.mcu_device)
    ack = g.transport.send_cmd(Cmd.GetVersion, b'', 500)
    print(f'{ep.firmware_sn}  {ep.side.name:5}  fw={ack.data[0]}.{ack.data[1]}.{ack.data[2]}')
"

# 视频编解码
python -c "import torchcodec; print('torchcodec', torchcodec.__version__)"
```

## 升级与更新

### 仓库 + 子模块 {#repo-update}

```bash
git pull --recurse-submodules
git submodule update --init --recursive --progress
./setup_env.sh --install     # 重新对齐依赖
```

!!! danger "拉完子模块必须重新编译 `xense.taccap`"
    `git submodule update` 只更新文件,不会重新编译。见 [2.2 克隆仓库与子模块](02-environment.md)。

### 固件 OTA 升级 {#ota}

### 我需要刷吗? {#ota-when}

**不用自己查版本号——跑一次 [`calibrate.py`](04-calibration.md#41) 就知道。**它在写任何东西
之前先验固件,不够会原样退出并打印这台当前的版本:

```text
✗ encoder-max calibration needs command set >= V2.1 (leader >= 1.2.0); this gripper reports 1.1.0.
  Nothing was changed. Flash it first: ...
```

看到以下**任意一条**就需要刷:

| 现象 | 出处 |
|---|---|
| `calibrate.py` 报 `needs command set >= V2.1` 并原样退出 | [4.1 夹爪标定](04-calibration.md#41) |
| 主夹爪连不上,报错里提示先做 OTA 升级 | [4.1.1](04-calibration.md#41) |
| 夹爪固件低于命令集 **V2.1**(即 leader < 1.2.0 / follower < 1.1.0) | 上面的[基线表](#版本兼容基线) |

都没遇到就**不用刷**。固件不会自己退化,刷过一次之后除非换主板或擦除固件,不需要再刷。

!!! warning "但 V2.1 只是**能用**的底线,不代表**没有已知缺陷**"
    上面几条判断的是"命令集够不够用"。而 leader `1.2.2` / follower `1.1.5` 之后修掉了三个
    实打实的缺陷,它们在更早的固件上都存在——包括跑在 leader `1.2.0` / follower `1.1.0` 上、
    完全满足 V2.1 的那些:

    | 缺陷 | 表现 |
    |---|---|
    | 命令通道活锁 | 持续高速率下发命令后,夹爪**再也不响应任何命令**,而数据流看起来一切正常(帧率、错误计数全对)。只能断电恢复 |
    | 日志阻塞实时任务 | 固件空载时就在以约 35 KB/s 打日志,而日志发送是阻塞的,会拖住发出它的那个任务 |
    | 启动时越界写 | 每次上电都会在一个数组末尾之外写一个字节 |

    第一条最值得注意:**它坏掉的样子和正常几乎无法区分**——设备还在正常推流,只是不再接受
    任何命令。如果你的采集流程里有"发命令 → 等响应"的环节,遇到过莫名其妙的卡住,值得升上来。

    这三条对主从夹爪都成立(它们在两种角色共用的代码里)。判断办法还是看
    [`manifest.json`](#ota) 里附带镜像的版本号,比你手上这台高就值得刷。

### 怎么刷

**所有夹爪都要升到 V2.1**。低于 V2.1 的固件不支持行程标定,
[夹爪标定](04-calibration.md#41)的第 2 步做不了,主夹爪也就无法连接。

SDK 自 0.1.7 起**随仓库附带已发布的固件镜像**,直接刷即可:

| 镜像 | 适用角色 |
|---|---|
| `tc-gu-01-master.bin` | 主夹爪(SN 末位 **`m`**) |
| `tc-gu-01-slave.bin` | 从夹爪(SN 末位 **`s`**) |

路径 `third_party/taccap-gripper/firmware/`,该目录只保留当前发布版。

!!! note "镜像版本随 SDK 版本走,以 `manifest.json` 为准"
    **这里不写死版本号**——附带的镜像版本取决于你这一版 SDK。同目录的 `manifest.json`
    记录了每个镜像的版本、字节数与 CRC32,是唯一出处:

    ```bash
    cat third_party/taccap-gripper/firmware/manifest.json
    ```

    只要不低于 leader `1.2.0` / follower `1.1.0`,就已经支持命令集 V2.1。

!!! warning "顺序:**先升级 SDK,再刷固件**"
    **刷写与刷后校验请用 0.1.7 及以上的 SDK**,更低版本不要用来刷固件。

    新 SDK 与旧固件通信不变,所以**先升 SDK 总是安全的**。

**按角色选镜像,不是按左右手。**角色看固件 SN 的**最后一个字符**:
`TCGU01A28Z0023m`(示例 SN) → 末位 `m` → 主夹爪,用 `tc-gu-01-master.bin`。同一套设备上的两只夹爪常常**都是主夹爪**。

```bash
# 1. 确认每只夹爪的角色
python -c "from xense.taccap import scan_grippers
for g in scan_grippers(): print(g.firmware_sn, '->', 'master' if g.firmware_sn.endswith('m') else 'slave')"

# 2. 刷写(镜像只写文件名即可,脚本会去 SDK 的 firmware/ 里找)
python third_party/taccap-gripper/python/examples/ota_update.py \
    tc-gu-01-master.bin --side left

# 3. 确认:GetVersion 返回固件编译进去的常量,读回的版本就是实际刷上去的版本
python -c "
from xense.taccap import scan_grippers, LeaderGripper, Cmd
for ep in scan_grippers():
    g = LeaderGripper(mcu_device=ep.mcu_device)
    ack = g.transport.send_cmd(Cmd.GetVersion, b'', 500)
    print(f'{ep.firmware_sn}  {ep.side.name:5}  fw={ack.data[0]}.{ack.data[1]}.{ack.data[2]}')
"
```

第 3 步读回的号必须**不低于** leader 1.2.0 / follower 1.1.0;刷的是随 SDK 附带的镜像时,
读回的通常比这高(某次实测两台示例设备均为 `1.2.1`;你读到的可能更高),这是正常的,
见[三套编号](#v21)。上面用 `LeaderGripper` 是因为这一步刷的是 master 镜像;从夹爪换
`FollowerGripper`,其余不变。

!!! tip "`--target-version` 是可选的"
    它只是给固件的安装后校验日志和分区元数据打个标记,**不影响刷什么内容**——刷进去的是哪一版
    由镜像文件本身决定。需要在日志里标注时才加,填 `manifest.json` 里那个版本号即可:

    ```bash
    python third_party/taccap-gripper/python/examples/ota_update.py \
        tc-gu-01-master.bin --side left --target-version 1.2.1
    ```

约 1 秒写完,夹爪重启并重新识别约 1–3 秒。新固件写在**备用分区**,校验通过之前不会覆盖
正在运行的那一份——所以传输失败不会把夹爪刷坏。

!!! danger "刷完之后**必须断电重插一次**,这是升级流程的一步,不是排障手段"
    上面那次重启是**软复位**:MCU 重新初始化了,但夹爪上的 USB 转串口芯片从头到尾没断过电。
    设备回来之后会停在一个**降级状态**,而这个状态从任何看得见的地方都分辨不出来——版本号
    正确、数据流在跑、错误计数全是 0,**唯一的症状是它在悄悄丢状态帧**。

    实测同一只夹爪、同一个固件、同一条线,60 秒一轮:

    | | 丢失的状态帧 |
    |---|---|
    | 只做了 OTA | 每轮 35~39 帧,连续三轮 |
    | 断电重插之后 | **0**,连续三轮 |

    所以顺序是:**刷写 → 断电重插 → 再做第 3 步的版本确认和后续标定**。在断电之前测到的
    任何数据都不可信。

    这和上面"升级期间不要断电"不冲突:**升级过程中**(蓝灯闪烁时)断电会写坏;**升级完成、
    夹爪已经重启之后**断电重插则是必需的。

!!! danger "刷错角色会导致夹爪无法启动,需返厂恢复"
    `ota_update.py` 会按 CRC32 与 `manifest.json` 比对识别镜像,**角色不匹配时直接拒绝**
    (不是告警),`--force` 才能强制。手工编译的镜像识别不出来,会带一条提示放行。

    升级期间**不要断电或拔线**(此时夹爪指示灯蓝色闪烁,见 [硬件介绍](hardware.md))。

!!! note "镜像只写文件名,不用管在哪个目录运行"
    `ota_update.py` 会依次尝试:你给的原始路径 → SDK 根目录下的同名路径 →
    SDK `firmware/` 下的同名文件。所以 `tc-gu-01-master.bin` 在主仓库根目录、
    SDK 目录里、以及其它任何位置都能解析到同一个镜像,不需要按仓库拼前缀。
    路径在**连接设备之前**就检查,写错文件名不会白等一次设备发现。

升到 V2.1 之后,回到 [4.1 夹爪标定](04-calibration.md#41) 把零点和行程上限标上——
升级本身不产生标定值,标定完成之前主夹爪仍然连不上。

## 支持与反馈 {#support}

遇到问题:

1. 先查[故障排查](troubleshooting.md)与[常见问题](07-faq-reference.md)。
2. 文档内容、链接或示例问题可提交到[文档仓库 Issues](https://github.com/XenseRobotics/XTac-UMI-G1-Docs/issues)。
3. 硬件、固件、标定材料或返修问题请通过设备交付 / 售后渠道反馈,并提供设备 SN。

反馈时请附带:

- 完整报错与相关日志,不要截断。
- `scan_grippers` 的 side / role / firmware_sn 输出。
- 本页“如何查版本”命令的输出。
- 复现步骤、完整命令、单夹爪 / 双夹爪、是否启用追踪器。
- 如涉及相机或硬件装配,附设备连接和异常画面照片。

## 兼容性与发布维护

- 当前站点文档版本为 `v0.0.6`;内容变更可通过文档仓库 Git 提交历史追踪。
- 主仓库版本号与本页文档版本对齐:`xense-taccap-lerobot` 的 `pyproject.toml` 记 `0.5.1+xtac.0.0.6`,其中 `0.5.1` 是 lerobot 官方基线,`xtac.0.0.6` 是与本文档同步的产品版本。
- **0.0.6 相对 0.0.5:三条都是 Docker 路径上踩过的坑,建议所有机器升级**——录制不再因为语音播报崩溃(不用再带 `--play_sounds=false`);录出来的视频是 `0644` 而不是 `0600 root`,导出时不再只有 `.mp4` 失败;容器里 `mamba activate` 不再提示 `Shell not initialized`。**采集程序本身的行为、数据格式和三个 SDK 与 0.0.5 相同**,升级不改变已录数据,也不需要重新标定。
- **0.0.5 相对 0.0.4:只有安装方式变了**——Docker 改为默认从 GHCR 在线拉取,不再需要 tar 交付包;镜像里的采集程序和三个 SDK 与 0.0.4 相同。已经装好 0.0.4 的机器**没有必须升级的理由**,尤其是采集进行到一半时不要动。
- 精确兼容关系以主仓库依赖锁定文件、子模块 commit 和本页“已验证基线”为准,不要仅按包名猜测兼容性。
- 升级主仓库、SDK、固件或 XenseVR PC Service 后,应重新执行环境验证、设备自检和一条短 episode 校验。
