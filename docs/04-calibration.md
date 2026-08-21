# 4. 标定与自检

本章做**一次性**的两件事:夹爪标定,和追踪器自检。
整条链路是否凑齐,在开录前用[预览](05-data-collection.md#preview)确认。

## 4.1 夹爪标定(零点 + 行程) {#41}

### 4.1.1 什么时候需要标

两种情况需要标,一种程序会替你发现,另一种只能你自己看出来:

| 情况 | 表现 | 谁发现 |
|---|---|---|
| **从没标过** | 采集程序**拒绝连接**,报错里直接带标定命令 | 程序,漏不掉 |
| **标过,但值和实际行程对不上** | 能连上,但张到机械极限时 `gripper.pos` 明显顶不到 `1.0`(例如停在 `0.8`) | **只能你在预览里看出来**,见 [4.1.3](#413) |

第二种常见于**拆装过编码器或动过机械限位**之后——程序只知道存过值,不知道值还准不准,
所以不报错。除这两种情况外不用重标:标定值写在 MCU flash 里,断电不丢、换主机也不用重标,
**一台标一次就够**。数据集里的 `gripper.pos` 是**归一化开度**,`0.0` 完全闭合、`1.0` 完全张开,
这两个端点就是标定写进 flash 的两个数:

| 端点 | 来源 | 由哪一步写入 |
| --- | --- | --- |
| `0.0` 闭合 | 编码器零点 | 标定第 1 步(完全闭合) |
| `1.0` 张开 | 该夹爪的行程上限 | 标定第 2 步(张到机械极限,固件 ≥ V2.1) |

**没标行程上限会怎样。**主夹爪连接时直接报错退出,报错里就写着要跑哪条命令:

```text
This leader gripper has no encoder-max calibration, so its jaw travel is unknown
and gripper.pos cannot be computed (...).

Calibrate it once, then re-run:

    python third_party/taccap-gripper/python/examples/calibrate.py <left|right>
```

!!! danger "双夹爪只标一侧,比两侧都不标更糟"
    两侧都没标时刻度至少一致;只标一侧会让 `left_gripper.pos` 和 `right_gripper.pos`
    落在**不同刻度**上——同一个握持动作左右读数不同,而数据里看不出任何异常。
    **要标就两侧都标。**

### 4.1.2 怎么标

按**左右**指定要标的夹爪,**两台各跑一次**:

```bash
python third_party/taccap-gripper/python/examples/calibrate.py left
python third_party/taccap-gripper/python/examples/calibrate.py right
```

左右由固件烧录的 SN 判定,和采集时 `left_gripper.pos` 用的是同一条规则,所以
`calibrate.py left` 标的一定就是 `left` 那一台。脚本会把解析出的
固件 SN 连同**扫到的全部夹爪**一起打印,方便在写 flash 之前确认没选错。想显式锁定某台时,
仍可直接传固件 SN(`calibrate.py TCGU01A28Z0024m`,SN 为示例,换成你自己那台的)。

!!! danger "报 `needs command set >= V2.1` → 先刷固件,再回来标定"
    脚本先验固件版本,不够就**原样退出、什么都不改**,并打印当前版本和该跑的命令:

    ```text
    ✗ encoder-max calibration needs command set >= V2.1 (leader >= 1.2.0); this gripper reports 1.1.0.
      Nothing was changed. Flash it first: ...
    ```

    刷写步骤见 **[固件 OTA 升级](versions.md#ota)**:镜像**按角色选**、且**先升 SDK 再刷固件**。

一条命令走完两步,按提示操作:

1. **保持夹爪完全闭合** → 回车。锁存为编码器零点,随后复读校验残差(容差 ±0.01 rad)。
2. **完全张开(顶到机械极限)** → 回车。采样此时的角度并**直接写入** MCU flash 作为 encoder max
   (没有二次确认,按回车前请先张到位),随后进入 10 Hz 实时读数便于目视核对。

输出形如:

```text
================================================================
  TacCap leader-gripper encoder calibration
================================================================
  requested    : left  (resolved by side)
  firmware SN  : TCGU01A28Z0031m
  side         : Left
  mcu serial   : 5C96089694
  mcu device   : /dev/serial/by-id/usb-1a86_USB_Dual_Serial_5C96089694-if02
  visible      : TCGU01A28Z0032m (Right), TCGU01A28Z0031m (Left)

Step 1/2: hold the gripper FULLY CLOSED.
  → press [Enter] when held closed:
  post-latch reading: raw=+0.0058 rad (+0.33°)   cooked=+0.0058
  ✓ zero latched OK (|raw post-zero| ≤ 0.010 rad)

Step 2/2: open the gripper to its MECHANICAL LIMIT.
  → press [Enter] when fully open:
  fully-open reading: +1.1486 rad  (+65.81°)
  ✓ stored: max_rad = 1.1486 rad (65.81°)
```

若这台之前标过,抬头还会多一行 `existing span: … — will be overwritten`。

!!! warning "先夹到位,再按 Enter"
    固件在收到命令的瞬间锁存当时的原始计数。按 Enter 前夹爪必须**已经**在目标开合位置
    (第 1 步完全闭合、第 2 步完全张开),中途再动就白标了。

!!! tip "闭合恒为 0"
    零点写在固件里,**没有 `gripper_closed_rad` 配置**。`position_rad` 保持输出原始弧度,
    归一化只是新增 `position` 字段,不改变原有读数。

### 4.1.3 确认标定生效 {#413}

**一、看启动日志。**标定生效时,采集程序连接每侧会打印:

```text
[left]  Jaw normalised by the firmware's encoder-max calibration
```

没打印这一行就不用往下看了——**未标定的主夹爪根本连不上**,程序会带着标定命令报错退出。

**二、在 Rerun 里看曲线。**开 `--display_data=true`,标量面板里找 `gripper.pos`:

| 动作 | 期望 |
| --- | --- |
| 完全张开 | 顶到 **1.0** |
| 完全闭合 | 落到 **0.0** |

能连上就说明行程上限已经生效,所以这一步查的是**存的值还准不准**。

!!! warning "张到底明显够不到 1.0 → 重新标定这一台"
    能连上、但完全张开只到 `0.8` 上下,说明 flash 里存的行程上限和这台夹爪的实际行程已经
    对不上了(常见于拆装过编码器或动过机械限位之后)。程序**不会**为此报错——它只知道有
    存过值,不知道值还准不准,所以这一条只能靠你在预览里看出来。

    重新跑一遍标定即可,命令和第一次完全一样。

### 4.1.4 适用范围

- **手动标定仅 leader(主夹爪)。**这套两步标定是主夹爪专有的,从夹爪不接受该命令——
  它走的是固件的**上电自动标定**(见下方说明)。采集时从夹爪的 `gripper.pos` 仍按
  `gripper_open_rad` 归一化,与旧算法一致。
- **需要固件命令集 ≥ V2.1**(即构建 leader **≥ 1.2.0** / follower **≥ 1.1.0**,[区别](versions.md#v21))**。**
  更低版本不支持行程标定:`calibrate.py` 会**原样退出、
  不改动任何东西**,采集时主夹爪则直接报错退出,并提示先做 OTA 升级。
  **低于 V2.1 的夹爪必须先升级固件**,镜像随 SDK 附带 → [固件 OTA 升级](versions.md#ota)。
  **先升 SDK 再刷固件**——刷写要用 0.1.7 及以上的 SDK。
- **标定是一次性的。**值写在 MCU flash 里,断电不丢,换主机不用重标。需要重做的只有两类
  情况:拆装编码器、更换机械限位或擦除固件之后;以及预览时发现**张到底明显够不到 1.0**
  ——后者说明存的值已经和实际行程对不上,见 [4.1.3](#413)。

!!! note "从夹爪的上电自动标定不替代这一步"
    **从夹爪**自 V1.9 起支持上电自动标定:上电时闭合到堵转取零点、张开到堵转取行程上限。
    **主夹爪没有这项功能**,而采集用的正是主夹爪,所以本节的手动标定仍然必要。

## 4.2 Pico4 Ultra 企业版追踪器自检

!!! note "追踪器没有标定这一步,这条命令只读不写"
    追踪器不需要标定:安装变换是**内置**的(见下),左右侧别由 SN 自动匹配。这条命令
    **不写入任何东西**——只是把位姿打印出来给你看。本节纯粹是确认链路通了、装配没错。

```bash
python -m lerobot.robots.taccap_gripper.check_tracker
# 指定某个 tracker SN(格式见 3.3,形如 PC2310MLL3200496G):
python -m lerobot.robots.taccap_gripper.check_tracker <tracker SN>
# 应用该侧内置的 tracker→TCP 安装变换:
python -m lerobot.robots.taccap_gripper.check_tracker --side right
```

以 10 Hz 打印 `raw`(追踪器自身位姿)与 `ee`(经刚性安装变换后的 TCP)。挥动夹爪,
`raw xyz` 应平滑变化、SN 与预期一致。

!!! note "安装变换是内置的,不需要你测"
    追踪器拧在夹爪上,它报的是**追踪器**的位姿,不是我们要记的 TCP。两者之间的刚性偏移
    由 `ee_transform.tracker_to_tcp` 内置(取自 CAD 装配实测),**左右两侧各自实测**——
    两侧接近镜像但不完全相同(旋转差 0.03°、平移差 1.27 mm),所以左值不是把右值镜像出来的。

    `--side` 决定套用哪一侧的内置值;**不带 `--side` 时变换是单位阵**,`ee` 会完全跟随 `raw`。
    需要覆盖(如重新加工过的安装座)才设 `--robot.tracker_to_ee_pos` /
    `--robot.tracker_to_ee_quat`,两者独立,可以只钉平移、旋转仍用内置值。

!!! tip "支点检查:不用任何额外硬件就能验证安装变换"
    把夹爪**两指的中点**抵在一个固定点上,握着手柄尽可能多地变换姿态摆动。
    **`ee xyz` 应基本不动,而 `raw xyz` 大幅摆动**——这就是全部测试内容,
    看到的漂移量即该变换的误差。**左右两侧都要测**;若左侧的值镜像方向错了,
    表现为 `ee` 的摆动幅度约为应有的两倍。

四元数出现半球翻转(符号跳变)时,位姿读取内部已有连续性修正;**若仍看到跳变请报 bug**。

标定与自检通过后,即可开始正式采集。

下一步 → [5. 数据采集](05-data-collection.md)
