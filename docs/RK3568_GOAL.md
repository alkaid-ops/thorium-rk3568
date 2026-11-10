# RK3568 硬解收敛计划（GOAL）

更新时间：2026-11-10
状态：M1/M2/M3 基本达成（2026-11-10 验证），M4 待决策

## 目标（Definition of Done）

在 `/home/hsy/project/thorium-rk3568`（GitHub 仓库，Chromium `156.0.8078.80`）产出的
**arm64 deb**，安装到 Sunniwell Z96A (RK3568) 后，达到与旧 `.28` 验证版**同等**的硬解能力：

1. H.264 1080p60 走 **V4L2 rkvdec 零拷贝**（解码进程持续占用 `/dev/video1`，
   日志出现 `S264` 输入 / `NV12` 输出，且无 `ImageProcessor error`）。
2. HEVC 1080p60 同样走硬解。
3. 播放 25s/60s 验收样本，稳态 `droppedVideoFrames` 不持续增长
   （先记录基线，是否追求 0 丢帧见 M4）。
4. 产物可复现：同一 args + 源码改动可重建出等价 deb。

## 已知事实（问题基线）

| 项 | 结论 |
|---|---|
| 新 deb 安装/启动 | ✅ 正常，`156.0.8078.80` |
| V4L2 初始化 | ✅ 自动选中 `/dev/video1`，`S264→NV12`，17/6 缓冲 |
| 硬解出帧 | ❌ `Importing NV12 buffers is not supported` → libyuv 映射失败 → 回落软解 |
| 运行中 `/dev/video1` | ❌ 无进程占用；CPU 83%+17%（软解） |
| 根因 | ① X11 ozone 仅允许 ANGLE → 无 `EGL_EXT_image_dma_buf_import`；② 原生 EGL 生效后，SharedImage 工厂仍不接受 `Y_UV/NV12` 的 NATIVE_PIXMAP（GBM 无法分配/导入 NV12）；③ GPU 进程因此 context lost → 解码器 disconnect |
| 已补 | 原生 EGL 允许列表 + `enable_validating_command_decoder` + X11 `CanCreateNativePixmapForFormat`/`IsFormatSupportedForTexturing` 支持 GL 可导入的多平面格式 + `CreateNativePixmapFromHandle` 直接包装 DMA-BUF |
| 待验证 | 原生 EGL 生效后 V4L2 已走到 `S264→NV12`；SharedImage Y_UV 修复重建中 |

## 旧树改动清单（来自 `old/` 文档与 wrapper，按需恢复）

| # | 文件 / 配置 | 作用 | 状态 |
|---|---|---|---|
| 1 | `ui/ozone/platform/x11/x11_surface_factory.cc` | 允许原生 EGL（`kGLImplementationEGLGLES2`） | 已改，待验证 |
| 2 | `rk3568_args.gn`: `enable_validating_command_decoder=true` | Panfrost 原生 EGL 需要 | 已加，重建中 |
| 3 | wrapper: `--use-gl=egl` + `--hardware-video-device-path` | 启用原生 EGL / 指定设备 | 待恢复（旧包装已提取） |
| 4 | `gpu/command_buffer/service/shared_image/ozone_image_backing_factory.cc` | X11 NV12 SharedImage 零拷贝 | 备用，若 M1 失败 |
| 5 | `gpu/ipc/service/gpu_init.cc` | 原生 EGL 下 NV12 能力上报 | 备用 |
| 6 | `media/base/media_switches.cc` | V4L2-only 默认开 `AcceleratedVideoDecoder` | ✅ 已加（工作区） |
| 7 | `media/gpu/sandbox/...hook_linux.cc` | 放行 media-controller 设备 | 未验证是否必要（当前解码已打开设备） |
| 8 | `media/base/media_switches.*` + v4l2 device 选择 | `--hardware-video-device-path` 支持 V4L2 | 暂不需要（自动选择可用） |
| 9 | `video_frame_submitter.cc` ack=2、`delay_based_time_source.cc` lead | 丢帧优化 | M4，暂不动 |

## 里程碑

### M1 原生 EGL + NV12 零拷贝（当前）
- 动作：x11 允许列表 + `enable_validating_command_decoder=true` + 重建 → 装板 → 测 H.264
- 通过标准：
  - 日志无 `Importing NV12 buffers is not supported`；
  - `supports_nv12_gl_native_pixmap` / renderable `NV12` 生效；
  - `lsof /dev/video1` 有解码进程；
  - 25s 播放 `currentTime` 正常推进。
- 失败后分支：补 #4 `ozone_image_backing_factory` → 再测；仍失败再补 #5。

### M2 wrapper 与包一致性
- 把 RK3568 wrapper（含 `--use-gl=egl`、解析 `/dev/videoN`）作为 RK3568 profile 的
  安装件；确保默认启动即硬解，不依赖用户手加参数。

### M3 HEVC + 4K 验收
- HEVC 1080p60 用同样方法验证；4K 只记录（旧结论是显示/合成上限，非解码）。

### M4 丢帧对齐（可选/按需）
- 若 M1~M3 硬件解码确认，再评估是否移植旧树的 ack=2 与 `BeginFrameTickLead`，
  目标：稳态丢帧不持续增长；是否追求 0 丢帧单独决策。

### M5 上游化收敛
- 输出两套改动：
  - A. Thorium 可接受（RK3568 args/docs/wrapper profile）
  - B. Chromium 上游候选（V4L2-only 默认开启加速解码、X11 原生 EGL/NV12 import、
    sandbox media-controller、device selection）
- 每项标注影响面与验证证据。

## 验收脚本（板端固定流程）

```sh
# 1) 干净启动
pkill -x thorium
rm -rf /tmp/th-test

# 2) 播放（X11）
DISPLAY=:0 XAUTHORITY=/tmp/slim.auth \
  /usr/bin/thorium-browser --user-data-dir=/tmp/th-test --no-first-run \
  --remote-debugging-port=9222 --enable-logging=stderr --v=1 \
  --vmodule='*v4l2*=3' \
  "file:///home/user/rk3568-thorium/video-only.html?video=<样本>&loop=1"

# 3) 判据
sudo lsof /dev/video1              # 必须出现解码进程
grep -c 'ImageProcessor error'     # 必须为 0
# CDP 读 totalVideoFrames / droppedVideoFrames
```

## 风险与约束

- 旧 Chromium 本地改动已随 `version.py` reset 丢失，只能按文档/日志/旧二进制重建；
  重建期间不得再执行 `version.py`、`git checkout -f`、`git clean -ffd`。
- 磁盘/网络：GCS 依赖按 DEPS pin 逐个补齐并校验 sha256。
- 编译并行度上限 24。

## 验证结果（2026-11-10）

产物：`chromium/src/out/thorium/thorium-browser_156.0.8078.80_arm64.deb` / `.rpm`

| 项 | H.264 1080p60 | HEVC 1080p60 |
|---|---|---|
| 解码器 | V4L2 stateless `S264` | V4L2 stateless `S265` |
| CAPTURE | NV12，10 buffers | NV12，10 buffers |
| OOP utility | `media.mojom.VideoDecoderFactoryProcess` 持有 `/dev/video1` | 同 |
| GPU 崩溃 | 无 | 无 |
| 播放 | 1920x1080，loop 正常 | 同 |

丢帧 A/B（同一 X11 测试条件，probe 12s 窗口）：

```text
旧 .28 二进制：decoded +801/+803，dropped +365/+387
新 .80 二进制：decoded +831/+822，dropped +419/+418
```

两者几乎一致（丢帧率约 46–50%）。旧文档里的“0 丢帧”来自其 KMS/direct-scanout
基准环境和丢帧优化补丁，不属于基础硬解能力，对应本计划的 M4。

## 本轮恢复的改动（9 项）

1. `ui/ozone/platform/x11/x11_surface_factory.cc`：允许原生 EGL；NV12/P010 可导入即视为可创建/可贴图；句柄导入直接包装 DMA-BUF
2. `rk3568_args.gn`: `enable_validating_command_decoder = true`
3. `media/media_options.gni`: Linux+V4L2 开启 OOP 解码
4. `media/mojo/services/mojo_cdm_service_context.{cc,h}`: ChromeOS 守卫
5. `media/mojo/services/gpu_mojo_media_client_linux.cc`: GPU 侧返回 kOutOfProcess
6. `media/gpu/sandbox/hardware_video_decoding_sandbox_hook_linux.cc`: 放行 `/dev/video0..255`、`/dev/media0..9`、补 `base/logging.h`
7. `media/gpu/v4l2/v4l2_video_decoder.cc`: CAPTURE 缓冲下限 10
8. `media/base/media_switches.cc`: V4L2-only 时默认开 `AcceleratedVideoDecoder`
9. `other/thor_ver_linux/wrapper-raspi`: 检测 `/dev/video-rkvdec` 时加 `--use-gl=egl`

## 未恢复 / 明确不需要

- `gpu/ipc/service/gpu_init.cc`、`ozone_image_backing_factory.cc`：实测当前路径已覆盖
- `--hardware-video-device-path`：V4L2-only 构建无此开关，自动选设备已验证可用
- `is_rk3568` 编译器调优：为上游化主动移除
- M4 丢帧补丁：`video_frame_submitter.cc`（ack=2）、`delay_based_time_source.cc`（BeginFrameTickLead）、renderer 诊断等
