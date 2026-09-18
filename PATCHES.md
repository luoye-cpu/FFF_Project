# 3FCompare 内核补丁索引（PATCHES）

> 基线：上游 master `440e662`（2026-09-18，含已合并的 PR #8 与 **PR #9**）。
> 本地分支：`3fc/integrate-issue7`。
>
> **2026-09-18 重大变更：上游合并了 PR #9。** 该 PR 把我们长期自行维护的 5 项扩展
> （A11 显卡指定、批量像素回读、渲染目标诊断、SetViewTransform 直写、16F/HDR 越界修复）
> 连同版本资源一并吸收。**重移植负担从 8 项降到 4 项**（类别一 7→3，类别二 1 项不变）。
>
> 本文档固化"哪些补丁必须在每次上游更新后重放"的清单，避免合并时靠记忆裁决。
> 原则：**上游优先**——上游已有等价实现的一律不重放；仅托管 API 硬依赖且上游无等价的扩展保留。

## 升级记录

| 日期 | 归档 tag | 上游 | 说明 |
|---|---|---|---|
| 2026-09-11 | `3fcompare-kernel-2026.9.11.1`（`6bc8d61`） | `f25c28f`（上游 9.12） | 首次 re-port |
| 2026-09-15 | `3fcompare-kernel-2026.9.14.1`（`025198f`） | `d8b2c03`（上游 9.14） | 仅 2 个 vbproj 变化，扩展无需重移植 |
| 2026-09-16 | `3fcompare-kernel-2026.9.14.2`（`68e1965`） | `d8b2c03`（上游 9.14） | 新增 A11 `preferredAdapterIndex`（API 14→15），首次 MSVC 构建验证 |
| 2026-09-16 | `3fcompare-kernel-2026.9.14.3`（`0fe33c4`） | `d8b2c03`（上游 9.14） | 追加 issue #7 修复（上游 `824093d` 的 cherry-pick） |
| 2026-09-17 | `3fcompare-kernel-2026.9.17.1`（`b6b96a6`） | `ea3ce05`（PR #8 合并后） | 基线溯源归正，`FFF.Native/` 与上一基线逐字节一致 |
| 2026-09-18 | `3fcompare-kernel-2026.9.18.1`（`ba6d875`） | `ea3ce05`（PR #8 合并后） | 新增 16F/HDR 越界修复；**PR #9 提交/合并前的最后本地态** |
| **2026-09-18** | *（本次合并，待打 tag）* | **`440e662`（PR #9 合并后）** | **上游吸收 5 项扩展。本地残留 = 类别一 3 项 + 类别二 1 项** |

## 一、必须重放（托管 API 硬依赖，上游无等价）—— 3 项

> ⚠ **术语澄清**：`SetPresentConfig` 仅存偏好位（内核几乎无行为），但**不是废弃项**——
> 托管侧有活跃的 P/Invoke 调用链（`Fff3FpEngine`、`MainWindow.SelfTest`）。
> 称其为 "shim" 只是指**对上游无价值**（上游无对应机制）；删除导出会立即
> `EntryPointNotFoundException`。

| 补丁 | 锚点 | 托管侧依赖 | 重移植要点 |
|---|---|---|---|
| **Redraw（K5 导出）** | `FFF.Player.Api.h` 声明 + `PlayerApi.cpp` 实现 + `PlayerSession::Redraw` + `PlayerVideoRenderer::Redraw` | `PlayerSurface.SubclassedWndProc`（子 HWND resize 后调用）→ `Fff3FpEngine.Redraw` | 让 presenter 感知尺寸变化并 swapchain resize + 重绘一帧。**上游无此导出** ⇒ 缺失时本地 app 直接 `EntryPointNotFoundException` |
| **SetLogCallback（F-LOG）** | `FFF.Player.Api.h` 的 `FFF3FPLogCallback` typedef + `PlayerApi.cpp` 的 `g_logSink` / `g_logContext` / `FFF3FP_KernelLogImpl` | `AppLog`（内核日志落盘到 `logs/app-*.log`） | 内核日志回调注册。**上游无此导出**。<br>⚠ `VideoRenderer.cpp` 的 `EnsureDevice()` 诊断行会 `extern` 声明并调用 `FFF3FP_KernelLogImpl`；该函数定义在 `PlayerApi.cpp`，重移植时不要漏 |
| **SetPresentConfig（tearing 偏好位）** | `VideoRenderer.h/.cpp` 的 `SetPresentConfig(bool)` + `PlayerSession` 转发 + `PlayerApi` 导出 | `Fff3FpEngine`、`MainWindow` | 只写 `swapAllowTearing_`（该成员是**上游既有**），不新增渲染器状态。原配对的 `SetPacingConfig` 已于 2026-09-18 全链路移除 |

## 二、本地专用（有效，但不推上游）—— 1 项

| 补丁 | 说明 |
|---|---|
| 音频缓冲 250ms | `PlayerSession.cpp` 的 `TargetAudioBuffer100ns`（音频包**投喂阈值**）由 120ms 改为 250ms。性质是"延迟换抗欠载"：对视频对比工具合适（延迟不敏感、抗欠载优先），**但会增加延迟 ⇒ 不推上游**。<br>⚠ 别与 WASAPI 缓冲区混淆：`WasapiRenderer.cpp` 的 bufferDuration 上游已改为自适应 `clamp(sharedDefaultPeriod*3, 50ms, 200ms)`，那一处跟随上游即可。<br>若高码率多声道仍欠载，优先评估调上游 clamp 上限，而非继续加大此值。 |

## 三、已移除（有意不保留）

| 补丁 | 移除原因 | 重引入条件 |
|---|---|---|
| P3 原生变速 SetSpeed（`7e85c99`） | 内核完整但托管侧 0 绑定（死代码）；与 UI 伪变速（每秒 Seek）语义冲突；每次上游更新白付重移植税。已从 PlayerApi 导出、API 头声明/枚举、PlayerSession、WasapiRenderer、VideoRenderer 全链路移除 | 托管侧正式接线时（导出 `FFF3FP_SetSpeed`、删除 UI 伪变速、加声画漂移测试 ≤100ms），从历史提交 `7e85c99` 整体重移植 |

## 四、已被上游 PR #9 吸收（2026-09-18）—— **禁止重放**

> PR #9 = 上游 `a6c74b3`（Native）+ `7deabbd`（Player），合并为 `440e662`。
> 以下各项**已在上游 `origin/master` 中**，本地保留的只是合并结果。
> ⚠ **严禁按历史提交重放**：已实测会产生重复定义（`PlayerSession::ReadVideoPixelRegion`、
> `PlayerApi.cpp` 的 `FFF3FP_GetRenderTargetInfo` 都重复过一次）。
> 上游版本在若干点上**比我们的更强**（见右列），合并时**优先取上游**。

| 补丁 | 本地原提交 | 上游改进（相对我们的版本） |
|---|---|---|
| A11 `preferredAdapterIndex`（多显卡指定） | `68e1965` | 成员改为 `= -1` 默认初始化，并注释点明"**0 是合法索引、不是未设置**" ⇒ 调用方若零值会静默钉死到 adapter #0。**托管侧仍必须显式置 -1**（`IPlayerEngine.PreferredAdapterIndex` 默认已是 -1，`AppSettings.Normalize` 钳制 -1..15） |
| `FFF3FP_ReadVideoPixelRegion`（原 patch 0004） | 随 re-port 带入 | ① `dstFloatCount` 比较改 64 位（防 uint32 溢出）；② 越界**拒绝** `InvalidArgument`（原为静默截断返回 Success）；③ 拷贝尺寸直接取请求尺寸 |
| `FFF3FP_GetRenderTargetInfo`（K4 诊断） | 随 re-port 带入 | ① 校验 `size` / `version`；② 无 swapchain 时返回 `InvalidState`（原为全零 Success，与真实 0x0 目标不可区分） |
| `SetViewTransform` 直写原子路径（原 0006 rev5） | 随 re-port 带入 | 用 `std::atomic<bool> discOpened_` 替代跨线程裸读 `disc_`（消除 data race / use-after-free 窗口） |
| 16F/HDR HALF 越界修复 | `ba6d875` | 上游 `ReadPixelRegion` 的 16F 分支本身就是 `HALF*` + `XMConvertHalfToFloat`，修复随 PR #9 一起进入上游 |
| `FFF.Native.rc` 版本资源 | `2fc46e9` | 上游已采纳（VER_API 15）。⚠ 这是**纯本地产物被 PR #9 一并带上去了**，现已是上游的一部分，无需再维护 |
| `.gitignore` 忽略 `vcpkg_installed/` | `11b7f6d` | 同上，已进上游 |

> **附带说明**：PR #9 还带上了我们在评审中提出的加固项（越界拒绝、64 位比较、RTInfo 校验、
> `EnsureDevice()` 内改用固定栈缓冲以避免 noexcept 下 `std::terminate`、`discOpened_` 原子）。
> 这些已是上游代码，不再属于本地补丁。

## 五、历史留档（已过时 —— **禁止重放**）

> **2026-09-18 复核：以下补丁均已不在当前代码中。**
> 判据：我方 HEAD 与上游对 `FF_THREAD_FRAME` / `SetMaximumFrameLatency` / `Present(0, 0)`
> 的命中数完全相同，说明这些符号全是上游自己的代码，我方补丁无残留。

| 补丁 | 为何过时 |
|---|---|
| `6e7469f` DWM 修复（ResizeBuffers 后 `Present(0,0)`） | 上游 `EnsureSwapChain` 失败恢复路径自带 `swapChain_->Present(0, 0)` |
| `a4a7ab0` FLAC 多线程解码（FF_THREAD_FRAME） | 上游已实现；不在净差异中 |
| P2 lock-free Render 快路径 | 上游 2026.9 渲染器重构后已有等价实现。<br>⚠ `interactiveMove_` + try_lock **本就是上游自己的代码**，曾被误当作我方补丁，**不要计为本地补丁** |
| `c941da3` HDR 元数据去重 + `SetMaximumFrameLatency(1→2)` | 两边同为 **1**，该改动未保留 |
| patch 0007 zoom viewport cover（`b0ff668`） | 上游 shader 已删除 ViewZoom/ViewPan 常量；**从未重放** |
| `SetPacingConfig`（A9 媒体率呈现节奏） | **2026-09-18 全链路移除**：内核实现为空操作，纯占导出位；托管侧 P/Invoke、`AppSettings.VrrPacingEnabled`、设置窗口复选框一并删除。导出数 83→82 |

## 六、纯增量（随分支走，无重放成本）

**已清空。** 原两项（`FFF.Native.rc`、`.gitignore`）已随 PR #9 进入上游，见类别四。

## 七、本地补丁与 issue #7（多路随机崩溃）的关系 —— **无关**（2026-09-17 实测）

用户曾质疑：本地有大量补丁，崩溃是否由它们引入？**实测结论：不是。**

**唯一的嫌疑项与证伪过程。** 本地补丁中唯一触及"交换链改写"的是
`SetViewTransform` 直写路径（现已属上游）——把它改回上游 `Enqueue` 版并编译后，
**8 路崩溃 5/8，同批次基线 4/8，没有下降 ⇒ 假设证伪。**
（分支 `3fc/exp-revert-svt`，提交 `2b90a87`。事后看也合理：zoom 只改绘制矩形、
不改 swapchain 尺寸，`EnsureSwapChain` 通常 early-return。）

**其余补丁逐一排除：**
- A11 —— 只在 `EnsureDevice()` 建设备时生效，运行时不参与；
- `ReadPixelRegion` / `GetRenderTargetInfo` —— 持锁与上游既有 `ReadPixel` 一致，且 `--multitest` 不调用；
- 音频 250ms —— 只影响音频包投喂，不碰 GPU/DXGI；
- `SetPresentConfig` —— 仅偏好位；
- 类别五各项 —— 已不在当前代码中。

⇒ **不要再往"本地补丁导致崩溃"方向排查。** 根因是内核跨渲染器并发 Present 这一
**上游既有设计**问题。

## 八、上游更新操作流程（2026-09-18 实测修订）

1. `git fetch origin --prune`，对照本文档逐类核对；
2. `git merge --no-ff --no-commit origin/master`，**逐文件解决冲突**；
3. ⚠ **合并必产生重复定义**（两侧都新增过同一函数）⇒ 解决完冲突后**必须扫描重名**：
   `grep -oP '^\s*\w+\s+(PlayerSession|PlayerVideoRenderer)::\w+' <file> | sort | uniq -d`；
4. 以 PlayerApi 导出面为完成判据：Redraw / SetLogCallback / SetPresentConfig 三个本地
   专属导出必须还在，导出总数应为 **82**，API 版本 **15**；
5. 构建（见下）+ 托管 `dotnet build` + `3FCompare.Core.Tests` 全绿；
6. 打新归档 tag（`3fcompare-kernel-<上游版本>.<序号>`），同步更新
   `tools/构建全部.ps1` 的 `$KernelBaselineSha` / `$KernelBaselineTag` 与 `.3fc_kernel_sha`。

> ⚠ **MSBuild.exe 在本机被安全策略拦截** ⇒ 用 `tools/build_kernel_manual.py`
> （复放 `FFF.Native.tlog` 中 MSBuild 真实下发的 cl/rc/link 命令，输出到新 obj 目录
> 以避免删除既有 obj）。产物校验：`tools/check_kernel_exports.py <dll>`。
