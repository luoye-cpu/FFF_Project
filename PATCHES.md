# 3FCompare 内核补丁索引（PATCHES）

> 基线：上游 master `ea3ce05`（2026-09-17，含已合并的 PR #8 = issue #7 修复）。
> 本地分支：`3fcompare/integrate-issue7`，当前归档 tag `3fcompare-kernel-2026.9.17.1`（`b6b96a6`）。
> （历史分支 `3fcompare/zoom-viewport-cover` 保留，停在 `0fe33c4`。）
>
> **升级记录**
> | 日期 | 归档 tag | 上游 | 说明 |
> |---|---|---|---|
> | 2026-09-11 | `3fcompare-kernel-2026.9.11.1`（`6bc8d61`） | `f25c28f`（上游 9.12） | 首次 re-port |
> | 2026-09-15 | `3fcompare-kernel-2026.9.14.1`（`025198f`） | `d8b2c03`（上游 9.14） | 仅 2 个 vbproj 变化，4 项扩展无需重移植；**尚未构建验证** |
> | 2026-09-16 | `3fcompare-kernel-2026.9.14.2`（`68e1965`） | `d8b2c03`（上游 9.14） | 新增 A11 `preferredAdapterIndex`（PlayerApiVersion 14→15）。首次真正用 MSVC 构建本基线并通过实机验证，同时补齐了 9.14.1 缺失的构建验证 |
> | 2026-09-16 | `3fcompare-kernel-2026.9.14.3`（`0fe33c4`） | `d8b2c03`（上游 9.14） | 追加 issue #7 修复（PresentTimedText 与交换链改写竞态），为上游 `824093d` 的 cherry-pick |
> | 2026-09-17 | `3fcompare-kernel-2026.9.17.1`（`b6b96a6`） | `ea3ce05`（上游 PR #8 合并后 master） | **上游已合并 PR #8**（即 issue #7 修复）。本基线 = 上游 master `ea3ce05` 与我方扩展 `68e1965` 的合并提交。`FFF.Native/` 源码与上一基线 `0fe33c4` **逐字节一致**（差异仅 README 随上游 `f551384`），故无功能差异、无 ABI 破坏——本次是**基线溯源归正**，使本地内核重新挂在上游 master 上 |
> 本文档固化"哪些补丁必须在每次上游更新后重放"的清单，避免合并时靠记忆裁决。
> 原则：**上游优先**——上游已有等价实现的一律不重放；仅托管 API 硬依赖且上游无等价的扩展保留。

## 一、必须重放（托管 API 硬依赖，上游无等价）

每次上游更新后需人工核对签名/上下文并重移植，按维护成本从低到高排列：

| 补丁 | 锚点 | 托管侧依赖 | 重移植要点 |
|---|---|---|---|
| SetPresentConfig / SetPacingConfig（最小 shim） | `VideoRenderer.cpp/.h`，PlayerApi 导出 `FFF3FP_SetPresentConfig` / `FFF3FP_SetPacingConfig` | `Fff3FpEngine`、`MainWindow` | tearing 存偏好位；pacing 为 no-op（上游无周期 keepalive present 可抑制） |
| GetRenderTargetInfo（K4 诊断） | `VideoRenderer.cpp/.h` 的 `RenderTargetInfo` 结构 + `lastDestX_/Y/Width/Height_` atomics；PlayerApi 导出 `FFF3FP_GetRenderTargetInfo` | `Fff3FpEngine` 轮询诊断 | `lastDest*` 由 `DrawCachedVideo` 成功路径记录——上游若重写该函数需重新锚定记录点 |
| ReadPixelRegion（原 patch 0004，批量像素回读） | `VideoRenderer.cpp/.h`；PlayerApi 导出 `FFF3FP_ReadVideoPixelRegion` | `Fff3FpEngine.TryReadPixelRegion`（缩略图/取色） | 单次 staging 拷贝 + Map，替代逐像素 GPU 往返；依赖 `AcquireBackBufferTarget` / `DrawCachedVideo` / `swapOutputBits_`，上游渲染器重构时签名可能漂移 |
| SetViewTransform 直写原子路径（原 0006 rev5） | `PlayerSession.cpp` `SetViewTransform`（绕过 Enqueue 命令队列直写渲染器原子量 + Redraw） | `Fff3FpEngine.SetViewTransform` ← UI 平移/缩放主路径 | 动机：命令队列在 Worker（解码）线程上执行，HD/HDR 播放时平移命令延迟数十 ms（"水平平移失效+卡顿"）。重放时保留上游的 disc 保护分支 |
| **PreferredAdapterIndex（A11 多显卡指定解码）** | `FFF.Player.Api.h` 的 `FFF3FPConfiguration` **末尾** + `PlayerApiVersion` **递增** + `PlayerApi.cpp` 范围校验；`VideoRenderer.h/.cpp` 成员 `preferredAdapterIndex_` + `SetPreferredAdapterIndex()` + `EnsureDevice()` 指定索引优先分支；`PlayerSession.cpp` 构造期接线 | `Fff3FpEngine.ConfigVersion` **必须同步递增**（托管 `Fff3FpConfiguration` 同步加字段）；`GpuEnumeration` 必须走 **DXGI `EnumAdapters1`** | ⚠ **ABI 破坏性变更**：`FFF3FP_Create` 校验 `version != PlayerApiVersion` **严格相等**且 `size >= sizeof(config)` ⇒ 内核与托管**必须同批次发布**，错开一个版本就会让全部会话创建失败。字段**只能追加在结构体末尾**，不得移动/插入既有字段（否则静默错位）。失败时必须回落到内置 monitor 匹配策略 |

| **Redraw（K5 导出）** | `FFF.Player.Api.h:570` 声明 + `PlayerApi.cpp:167` 实现，导出 `FFF3FP_Redraw` | `PlayerSurface.SubclassedWndProc`（子 HWND resize 后调用）→ `Fff3FpEngine.Redraw` | 让 presenter 感知尺寸变化并执行 swapchain resize + 重绘一帧。**上游无此导出** ⇒ 缺失时本地 app 直接 `EntryPointNotFoundException` | 
| **SetLogCallback（F-LOG）** | `FFF.Player.Api.h:443` 声明 + `PlayerApi.cpp:29` 实现，导出 `FFF3FP_SetLogCallback` | `AppLog`（内核日志经 `FFF3FP_KernelLogImpl` 落盘到 `logs/app-*.log`） | 内核日志回调注册。**上游无此导出** | 

## 二、本地专用（有效，但不推上游）

| 补丁 | 说明 |
|---|---|
| 8204c02 音频缓冲 250ms | `PlayerSession.cpp` 的 `TargetAudioBuffer100ns`（音频包**投喂阈值**）由 120ms 改为 250ms。性质是"延迟换抗欠载"：对视频对比工具合适（延迟不敏感、抗欠载优先），**但会增加延迟 ⇒ 不推上游**。<br>⚠ 别与 WASAPI 缓冲区混淆：`WasapiRenderer.cpp` 的 bufferDuration 上游已改为自适应 `clamp(sharedDefaultPeriod*3, 50ms, 200ms)`，那一处跟随上游即可。<br>若高码率多声道仍欠载，优先评估调上游 clamp 上限，而非继续加大此值。 |

## 三、已移除（有意不保留）

| 补丁 | 移除原因 | 重引入条件 |
|---|---|---|
| P3 原生变速 SetSpeed（`7e85c99`：时钟斜率 + Wasapi `speed_` 缩放 + `SpeedChanged` 事件 + 渲染器 speedBits shim） | 内核完整但托管侧 0 绑定（死代码）；与 UI 伪变速（每秒 Seek）语义冲突；每次上游更新白付重移植税。2026-09-13 从 PlayerApi 导出、API 头声明/枚举、PlayerSession、WasapiRenderer、VideoRenderer 全链路移除 | 托管侧正式接线时（导出 `FFF3FP_SetSpeed`、删除 UI 伪变速、加声画漂移测试 ≤100ms），从历史提交 `7e85c99` 整体重移植 |

## 四、历史留档（已过时 / 已被上游吸收 —— **禁止重放**）

> **2026-09-18 复核：以下补丁均已不在当前代码中。**
> 判据：我方 HEAD 与上游 `ea3ce05` 对 `FF_THREAD_FRAME` / `SetMaximumFrameLatency` /
> `Present(0, 0)` 的**命中数完全相同**（1/1、1/1、1/1），说明这些符号全是上游自己的代码，
> 我方补丁无残留 —— 即已随上游重构被吸收，或在重移植时放弃。
>
> 保留仅作历史记录。**严禁按此重放**：会与上游现有实现重复甚至冲突。
> 当前真实存在的本地补丁 = 类别一 5 项 + 类别二 1 项（音频）+ 类别五 2 项。

| 补丁 | 为何过时 |
|---|---|
| 6e7469f DWM 修复（ResizeBuffers 后 `Present(0,0)` 解除 DWM 停滞） | 上游 `EnsureSwapChain` 失败恢复路径自带 `swapChain_->Present(0, 0)` |
| a4a7ab0 FLAC 多线程解码（FF_THREAD_FRAME） | 上游已实现（或作者不认为音频解码是瓶颈）；不在净差异中 |
| P2 lock-free Render 快路径（稳态 try_to_lock 跳过 `deviceMutex_`） | 上游 2026.9 渲染器重构后已有等价实现。<br>⚠ 另注：经核对 `interactiveMove_` + try_lock **本就是上游自己的代码**，历史上曾被误当作我方补丁，**不要计为本地补丁** |
| c941da3 HDR 元数据去重 + `SetMaximumFrameLatency(1→2)` | 两边 `SetMaximumFrameLatency` 同为 **1**，该改动未保留 |
| patch 0007 zoom viewport cover（b0ff668） | 上游 shader 已删除 ViewZoom/ViewPan 常量，UV 空间缩放无处生效；**从未重放** |

## 五、纯增量（随分支走，无重放成本）

- `2fc46e9` 版本资源 `FFF.Native.rc`（FileVersion 主段 = PlayerApiVersion，**当前 15**；**升内核 API 时同步递增**，供托管 `NativeRuntime.ExtractEmbeddedDll` 版本比较）。
  ⚠ 本行此前长期写着"当前 13"，实际当时已是 14（托管 `Fff3FpEngine.ConfigVersion` 才是真源），现已更正。
- `11b7f6d` `.gitignore` 忽略 `vcpkg_installed/`。

## 六、本地补丁与 issue #7（多路随机崩溃）的关系 —— **无关**（2026-09-17 实测）

用户提出质疑：本地有大量补丁，崩溃是否由它们引入？**实测结论：不是。**

**唯一的嫌疑项与证伪过程。** 本地补丁中，唯一会触及"交换链改写"的是
`SetViewTransform` 直写路径（类别一 0006 rev5）——它绕过 `Enqueue` 后，
`Redraw()` → `EnsureSwapChain()` 可从任意调用线程发起，而我们已证实
"Present 撞上交换链改写"正是崩溃触发条件（把它串行化后 8 路崩溃率 56% → 12.5%）。

于是做了回退实验（分支 `3fc/exp-revert-svt`，提交 `2b90a87`，产物 `kernel_revert_svt.dll`）：
把 `PlayerSession::SetViewTransform` 改回上游 `Enqueue` 版并编译——
**8 路崩溃 5/8，同批次基线 4/8，没有下降 ⇒ 假设证伪。**

（事后看也合理：zoom 只改变绘制矩形、不改变 swapchain 尺寸，`EnsureSwapChain`
通常会 early-return，所以这条路径很少真的触发改写。）

**其余补丁逐一排除：**
- A11 `preferredAdapterIndex` —— 只在 `EnsureDevice()` 建设备时生效，运行时不参与；
- `ReadPixelRegion` / `GetRenderTargetInfo` —— 持锁与上游既有 `ReadPixel` 一致，
  且 `--multitest` 根本不调用它们；
- `TargetAudioBuffer100ns` 250ms —— 只影响音频包投喂，不碰 GPU/DXGI；
- `SetPresentConfig` / `SetPacingConfig` —— 后者是 no-op；
- 类别四三项 —— 已不在当前代码中（见该节）。

⇒ **不要再往"本地补丁导致崩溃"这个方向排查。** 根因是内核里跨渲染器并发 Present
这一**上游既有设计**问题，与我们的扩展无关。

## 上游更新操作流程

1. `git fetch origin`，对照本文档逐类核对；
2. 类别一 4 项重移植（以 PlayerApi 导出面为完成判据：`FFF3FP_SetPresentConfig` / `SetPacingConfig` / `GetRenderTargetInfo` / `ReadVideoPixelRegion` 可编译可链接）；
3. 类别二 2 项复核上游是否仍覆盖；
4. 类别四逐项决策（默认：FLAC 多线程与 P2 保到不能再保）；
5. MSBuild Release x64 构建 + 托管 `dotnet build` + `3FCompare.Core.Tests` 全绿；
6. 打新归档 tag（`3fcompare-kernel-<上游版本>.<序号>`）。
