# ORCA Hand Retarget 测试对比

ORCA v1 右手 · MediaPipe · Adaptive Analytical · MuJoCo 仿真

本仓库仅用于分享 **YAML 参数与测试视频**。先看配置改动，再按场景比较视频。视频尚未放入；下方文件链接在加入对应文件后可用，当前不代表已有测试结果。

## 配置与改动说明

### A：原始配置

**文件：[adaptive_v1_test.yaml](configs/adaptive_v1_test.yaml)**

- 用途：原始对照组，保留额外指尖偏移。
- `w_pos: 1.0`，`w_dir: 10.0`，`w_full_hand: 1.0`。
- `norm_delta: 0.04`，`lp_alpha: 1.0`（无输出低通平滑）。
- 补充说明：待填写。

### B：零偏移配置

**文件：[adaptive_v1_tipframe_test.yaml](configs/adaptive_v1_tipframe_test.yaml)**

- 相对 A：五指 `fingertip_offsets_m` 均为 `[0, 0, 0]`，直接使用 URDF 指尖 frame 原点。
- 其他 YAML 参数与 A 一致。
- 修改目的：避免在已定义的指尖位置上额外叠加指尖长度。
- 注意：该修改也影响自动尺度标定，不是固定尺度下的纯几何对照。
- 补充说明：待填写。

### C：参数实验配置

**文件：[adaptive_v1_experiment.yaml](configs/adaptive_v1_experiment.yaml)**

- 当前相对 B：`w_full_hand: 1.0 → 0.5`。
- 修改目的：降低全手形状约束的权重，观察是否改善捏合，同时检查姿态是否退化。
- 其他改动：无（之后修改 YAML 时，同步更新这里）。
- 补充说明：待填写。

### 其他 YAML

[adaptive_v2_test.yaml](configs/adaptive_v2_test.yaml)：历史 v2 配置，统一存放于 configs，本页三个场景暂不比较它。

> B/C 需要 retarget 代码支持 `fingertip_offsets_m`；未修改的上游代码不一定读取此字段。本页是实验展示，不是完整软件环境。

## 如何下载和运行 YAML

### 1. YAML 放在哪里？

在本仓库点击 **Code → Download ZIP**，解压后将 `configs/` 放到你本机 Orca 项目的 `orca_adaptive_test/` 中，保持如下结构。也可以在单个 YAML 页面点击 Raw 下载对应文件。

```text
Orcahand/
├── orca_teleop/
│   ├── .venv/
│   └── scripts/teleop_sim.py
├── orca_sim/
├── orcahand_description/
│   └── v1/models/urdf/orcahand_right.urdf
└── orca_adaptive_test/
    └── configs/
        ├── adaptive_v1_test.yaml
        ├── adaptive_v1_tipframe_test.yaml
        ├── adaptive_v1_experiment.yaml
        └── adaptive_v2_test.yaml
```

不要放进 `.venv` 或 Python 的 site-packages。已有同名实验文件时先保留旧版本，避免旧视频失去对应参数。

### 2. 运行前提

下列命令用于 **macOS + 已安装依赖的 Orca 仿真环境**。下载本仓库本身不会安装 orca_teleop、orca_sim、MuJoCo、MediaPipe、Pinocchio 或 NLopt；队友需先准备同版本项目与虚拟环境。这里不使用实体机器人。

B/C 的零偏移字段需要代码支持。打开 `orca_teleop/src/orca_teleop/retargeting/adaptive_analytical.py`，在 `_build_frame_indices()` 中检查是否已经读取 `fingertip_offsets_m`。本次测试使用的代码已有此支持。

如果没有，将函数末尾设置 `_frame_offsets` 的那一小段替换为以下代码（放在函数内部，保持缩进），其他代码不动：

```python
        self._frame_offsets = [np.zeros(3, dtype=np.float64) for _ in self._computed_frame_names]
        tip_offsets = self._retarget_cfg.get("fingertip_offsets_m", {})
        for finger, frame_name in zip(FINGERS, self._frame_map.tip, strict=True):
            frame_idx = self._computed_frame_names.index(frame_name)
            offset = np.asarray(
                tip_offsets.get(finger, FINGERTIP_OFFSETS_M[finger]), dtype=np.float64
            )
            if offset.shape != (3,) or not np.all(np.isfinite(offset)):
                raise ValueError(f"fingertip_offsets_m.{finger} must contain three finite values")
            self._frame_offsets[frame_idx] = offset
```

没有此支持时，单纯下载 B/C YAML 不能保证零偏移生效。A 未设置该字段，仍使用旧偏移。确保运行环境导入的是你修改的源码。

### 3. 选择 A、B 或 C，然后启动

先进入 `orca_teleop`。本机路径如下；队友替换为自己的实际路径：

```bash
cd /Users/su/Code/Orcahand/orca_teleop
```

三选一，在同一个终端设置本次配置：

```bash
# A：原始配置
RETARGET_CONFIG=../orca_adaptive_test/configs/adaptive_v1_test.yaml
```

```bash
# B：五指零偏移
RETARGET_CONFIG=../orca_adaptive_test/configs/adaptive_v1_tipframe_test.yaml
```

```bash
# C：当前参数实验（w_full_hand=0.5）
RETARGET_CONFIG=../orca_adaptive_test/configs/adaptive_v1_experiment.yaml
```

然后执行统一启动命令。直接使用虚拟环境中的 mjpython，无需先 activate：

```bash
.venv/bin/mjpython scripts/teleop_sim.py \
  --env right \
  --version v1 \
  --hand right \
  --local \
  --show-video \
  --retargeter adaptive_analytical \
  --urdf_path ../orcahand_description/v1/models/urdf/orcahand_right.urdf \
  --retarget-config "$RETARGET_CONFIG"
```

每行末尾的反斜杠后不要加空格。v2 YAML 不适用于这条 v1 命令。

启动后将右手放入摄像头，保持一致的标定姿势，等待 `auto-scale calibrated` 后开始 S01/S02/S03 与手动录屏。切换 YAML 或修改参数后，在启动终端按 Ctrl+C 完整退出，再重新启动；不要同时开启两个使用同一端口的仿真。若绿色骨架正常但仿真不动，检查 `Publisher connected` 日志，并用 `lsof -nP -iTCP:50051 -sTCP:LISTEN` 查看是否有旧实例残留。

## S01：张手、半握、握拳

动作：张手 → 半握 → 握拳 → 张开，各姿态保持约 3 秒，重复 5 次。

| A：原始配置 | B：零偏移 | C：参数实验 |
| --- | --- | --- |
| [查看视频](videos/S01/A.mp4) | [查看视频](videos/S01/B.mp4) | [查看视频](videos/S01/C.mp4) |
| 待填写观察 | 待填写观察 | 待填写观察 |

**同屏对比视频：** 待添加。
<!-- 在下一行粘贴 GitHub 上传生成的视频附件地址；推荐 A/B/C 三栏合成视频。 -->

**本场景结论：** 待填写。重点比较姿态完成程度、保持抖动和突然跳变。

## S02：拇指—食指慢速捏合与释放

动作：慢速靠近 → 捏合保持约 3 秒 → 慢速释放，重复 5 次。

| A：原始配置 | B：零偏移 | C：参数实验 |
| --- | --- | --- |
| [查看视频](videos/S02/A.mp4) | [查看视频](videos/S02/B.mp4) | [查看视频](videos/S02/C.mp4) |
| 待填写观察 | 待填写观察 | 待填写观察 |

**同屏对比视频：** 待添加。
<!-- 在下一行粘贴 GitHub 上传生成的视频附件地址；推荐 A/B/C 三栏合成视频。 -->

**本场景结论：** 待填写。重点比较指尖间隙、错位、保持稳定性和释放。视觉闭合不等于已验证物理接触。

## S03：三指抓持与释放

动作：拇指—食指靠近 → 中指加入 → 三指保持约 3 秒 → 中指退出 → 释放，重复 5 次。

| A：原始配置 | B：零偏移 | C：参数实验 |
| --- | --- | --- |
| [查看视频](videos/S03/A.mp4) | [查看视频](videos/S03/B.mp4) | [查看视频](videos/S03/C.mp4) |
| 待填写观察 | 待填写观察 | 待填写观察 |

**同屏对比视频：** 待添加。
<!-- 在下一行粘贴 GitHub 上传生成的视频附件地址；推荐 A/B/C 三栏合成视频。 -->

**本场景结论：** 待填写。重点比较中指加入是否破坏已有捏合、拇指跳转和释放。空手测试仅验证抓持姿态，不代表能抓住物体。

## 如何添加视频

1. 每个场景的三个视频分别命名为 `A.mp4`、`B.mp4`、`C.mp4`，放进对应的 `videos/S01/`、`S02/`、`S03/`。如果是 MOV，修改上方链接后缀即可，不要只重命名冒充 MP4。
2. 表格提供三个配置并排的视频入口。**普通仓库视频链接不等于 README 内嵌播放器，README 也不提供三个独立播放器的一键同步控制。**
3. 若需要“同时播放三个结果”，将 A/B/C 合成一条带配置标签的三栏视频，保持原始速度，以动作开始点对齐。上传到 GitHub Markdown 编辑区，把生成的视频附件地址单独粘贴到对应“同屏对比视频”处；用 Preview 确认实际播放效果。
4. 每次录像同时显示人手与仿真，固定标定手势和光照。更新 YAML 时也更新本页说明；已有视频仍对应旧参数时，新增配置编号，不覆盖旧参数。

GitHub 视频附件的操作见[官方说明](https://docs.github.com/en/get-started/writing-on-github/working-with-advanced-formatting/attaching-files)。较大原片可保存在团队共享存储，在本页链接；不必全部放进 Git 历史。

## 文件结构

```text
configs/                 所有分享用 YAML
videos/S01/              A.mp4、B.mp4、C.mp4
videos/S02/              A.mp4、B.mp4、C.mp4
videos/S03/              A.mp4、B.mp4、C.mp4
README.md                参数说明与场景视频对比
```

configs 中是整理时复制的 YAML。之后分享实验以 configs 文件为准：修改此处并在运行时显式指定 `--config configs/文件名.yaml`（run_test.py），或 `--retarget-config configs/文件名.yaml`（teleop_sim.py，路径相对于运行目录）。根目录旧 YAML 和本机工具未删除，但不会上传，也不会自动与 configs 同步。
