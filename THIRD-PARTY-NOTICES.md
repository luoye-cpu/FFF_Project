# 第三方组件与许可清单 / Third-Party Notices（3FP 内核与播放器）

本文件覆盖 `3fp/kernel` 这条发布链**随包分发**的东西：`FFF.Native.dll`、`FFF.Player.exe`
及其运行期依赖（FFmpeg / libass 一族 / LakeUI / NuGet 件）。
每项许可标识都注明**证据来源**；读不到的写 **未判定**，不猜。
是否合规、要不要授权 → **需法务**，本文件只交事实。

- 与主仓的对应文件：`C:\PLAN\3FCompare\THIRD-PARTY-NOTICES.md`（那条链是 3FCompare 比对工具，
  载荷表不同，两处都要维护）。本文件由 `tools\发布3FP.ps1` 的许可步骤强制随包，缺席即发布失败。
- 许可文本原件：由同一条步骤从 vcpkg / FFmpeg / LakeUI 的在盘位置收进包内 `licenses\`；
  读不到即中止（可用 `FFF_ALLOW_MISSING_LICENSES=1` 显式降级）。

---

## 1. FFmpeg 运行期件（13 个 DLL）

| 项 | 读数 |
|---|---|
| 在包件 | `avcodec-63` `avformat-63` `avutil-61` `avfilter-12` `swscale-10` `swresample-7` `avdevice-63` + 伴生件（`libc++` `libunwind` `libwinpthread-1` `libomp` `libshaderc_shared` `libSPIRV-Tools-shared`） |
| 取得方式 | `tools\准备FFmpeg.ps1` 下载后放入 `third_party\ffmpeg\`（编译期）与 `runtime\`（运行期） |
| 盘上实测变体（本树 `runtime\`，即播放器实际带的这一套） | **12 件里 6 件带 configuration 串，这 6 件全部含 `--enable-nonfree`**，同时含 `--enable-gpl --enable-version3 --enable-libx264 --enable-libx265 --enable-libfdk-aac`，`--prefix=/home/dominic/ffmpeg/bin`（BtbN 构建农场）。逐件偏移：avcodec 67 441 853 / avfilter 10 144 029 / avformat 6 515 309 / avutil 1 309 255 / swresample 361 117 / swscale 2 235 900 |
| 同一判据扫主仓发货源（3FCompare 那条链，`publish\PLAN\ffmpeg-full`） | 13 件里 7 件带串、7 件全 nonfree（多出来的一件是 `avdevice-63.dll`） |
| 链接方式 | **动态 + 延迟加载**：`FFF.Native.vcxproj:79-81/105-107` 链的是 DLL 导入库（`.def` 首行 `LIBRARY "avcodec-63"`），运行期由 `Shared/Ffmpeg/FfmpegRuntime.cpp:89-116` 的自定义 delay hook 按 `prefix*.dll` 定位 ⇒ `FFF.Native.dll` 不内嵌 FFmpeg 机器码 |
| 仓内许可文本 | 只有 `COPYING.LGPLv2.1`、`COPYING.LGPLv3`；**无 GPLv2/GPLv3、无 `LICENSE.md`** |

⚠ **未解决**：分发的是 gpl+nonfree 组合而 GPL 文本缺席；该组合的分发条件需回上游原文核对（**需法务**）。

## 2. libass 字幕栈（vcpkg 传递依赖，随包 12 个 DLL）

证据源：`third_party\vcpkg_installed\x64-windows\share\<port>\vcpkg.spdx.json` 的 `licenseConcluded`
（`licenseDeclared` 全为 `NOASSERTION`）+ 该 port 的 `copyright` 文件首行。

| port | 许可 | 随包件 |
|---|---|---|
| libass | **ISC** | `ass-9.dll` |
| libbluray | **LGPL-2.1-or-later** | `bluray-3.dll` |
| libudfread | **LGPL-2.1-or-later** | `udfread-3.dll` |
| libiconv | **LGPL**（其 `COPYING.LIB` 自述；vcpkg 未映射 SPDX） | `iconv-2.dll`、`charset-1.dll` |
| fribidi | **LGPL-2.1-or-later** | `fribidi-0.dll` |
| freetype | **FTL OR GPL-2.0-or-later** | `freetype.dll` |
| harfbuzz | **MIT-Modern-Variant** | `harfbuzz.dll` |
| brotli | **MIT** | `brotlicommon.dll`、`brotlidec.dll` |
| bzip2 | **bzip2-1.0.6** | `bz2.dll` |
| libpng | **libpng-2.0** | `libpng16.dll` |
| zlib | **Zlib** | `z.dll` |
| libxml2 | **MIT** | `libxml2.dll` |

`vcpkg.json` 只声明 `libass` + `libbluray`，其余是传递依赖。

## 3. 静态链进 `FFF.Native.dll` 的 libiamf 一族

`FFF.Native.vcxproj` 的 `HaveLibiamf` 分支链入 `iamf.lib; oar.lib; obr_capi.lib; obr.lib; pffft.lib;
FLAC.lib; opus.lib` + **59** 个 `absl_*.lib`（磁盘 59 个逐名相符）。这些 `.lib` 均为 `!<arch>` 静态归档、
体内 0 个 `.dll` 名引用；PDB 内 `FLAC__` 180 / `IAMF` 183 / `opus_multistream` 7 命中。

| 组件 | 许可（按上游；原文收集状态 2026-11-07 更新，文件在本仓 `licenses\`） |
|---|---|
| libiamf | BSD-3-Clause Clear（Alliance for Open Media，2022）；原文已收：`licenses\libiamf-LICENSE.txt`（上游 main）+ `licenses\libiamf-PATENTS.txt`（同仓专利声明）；构建副本为 `third_party\libiamf\LICENSE.libiamf.txt`（钉定版本 `b276f437`） |
| OAR | BSD-3-Clause Clear（AOM，2022）；原文已收：`licenses\oar-LICENSE.txt`（自 libiamf 钉定版本的 oar 子模块 `ca14b543` 收取，与上游 main 现行文本存在版本差异，main 版留档 `licenses\oar-LICENSE-main-branch.txt`） |
| OBR | 随 oar 仓分发（`oar\src\renderer\obr\obr_capi\obr`，README 自述 Open Binaural Renderer）；**上游无独立 LICENSE 文件**，按宿主仓 oar 的 BSD-3-Clause Clear 对待；具体授权含义**未判定** |
| FLAC | BSD-3-Clause（Xiph）；**未判定**（无原文；libiamf 的 `dep_codecs\` 只给 bin/include/lib 预编译件，无许可文本） |
| opus | BSD-3-Clause（Xiph）；**未判定**（无原文，同上） |
| pffft | FFTPACK/UCAR 条款（Julien Pommier 2013 原作 + Hayati Ayguen 2020 修改）；原文头部已收：`licenses\pffft-license-header.txt`（hayguen/pffft master；libiamf 树内无 pffft 源码与许可件，bazel 经 `external\pffft.BUILD` 拉取） |
| abseil-cpp（59 件） | Apache-2.0；原文已收：`licenses\abseil-cpp-LICENSE` |

**`fdk-aac.lib` 链接行状态（2026-11-07 复核，推翻下文旧陈述）**：旧句"已于 2026-10-15 从本树摘除、
摘除前 `iamf.lib` 无 FDK 引用"经实测**不成立**，本树（KF）三处实况互相一致——`FFF.Native.vcxproj:122`
链 `fdk-aac.lib`、`tools\准备Libiamf.ps1:151` staging 它、磁盘 `lib\x64\fdk-aac.lib` 在（2026-11-06
0:21，4,170,630 B）。证据链：

- git 史（`git log -S fdk-aac`，三个触碰点）：`d3f0945`（2026-10-12 上游自带，链接行含 fdk-aac）→
  `d19d198`（2026-10-15 从 vcxproj 摘除**但未动准备脚本**，同时写下本段的"已摘除"陈述）→
  `b27c959`（2026-11-05 重新加回链接行，现行状态）。
- 2026-11-07 `dumpbin /symbols` 实测（磁盘 iamf.lib 为 11-06 0:26 构建，SHA256 `F671CAF5…FF91`；
  注意 `3fp\kernel\third_party` 是 junction，指向 `third_party\fff_project\third_party`，两树引用
  同一物理件）：iamf.lib 共 **41 个归档成员**，其中 `aac_multistream_decoder.obj` 的 UNDEF External
  符号含**全部 7 个 FDK API**（`aacDecoder_{Open,Close,Fill,SetParam,ConfigRaw,DecodeFrame,GetStreamInfo}`），
  整档 aac/fdk 相关 UNDEF **12 个**；二进制串旁证 `aacDecoder_` 在 iamf.lib 7 命中、fdk-aac.lib 201 命中、
  oar/obr/obr_capi/pffft/FLAC/opus 全 0（FDK 引用仅来自 iamf.lib）。fdk-aac.lib 恰好定义这 7 个符号
  （各 1 个 Defined External）。当年"EXTERNREF 0 命中"的口径作废——dumpbin 输出中本无 EXTERNREF 字样，
  实际要看的字段是 UNDEF External。
- **机制**：静态库按需拉取。引用 FDK 的成员目前不在链接闭包内 ⇒ TP（链接行无 fdk-aac.lib，10-15 摘除后
  `MSBUILD_NATIVE_RC=0`）与 KF（链接行有）都能链接通过；2026-11-07 复测 11-06 构建的成品
  `FFF.Native.dll`（6,449,152 B）内 `aacDecoder_DecodeFrame` 串 **0 命中** ⇒ fdk-aac 代码**零进包**。
- **处置（2026-11-07 裁决：保持链接不动）**：链接行与 staging 均不改——防御性保留，若未来内核改动使
  `aac_multistream_decoder.obj` 进入抽取闭包，缺件会以 LNK2019 立即暴露而非静默出空。代价只是链接时
  多读一个 4 MB 归档，零产物影响。
- **许可**：fdk-aac 的许可原文**不在** libiamf 仓内（`code\dep_codecs\` 仅 bin/include/lib 预编译件），
  **未判定**；因其代码零进包，不构成随包分发义务；若未来闭包变化使 FDK 代码进包，必须先收原文再发布。

## 4. LakeUI —— **GPL-3.0-only，且作者声明商用需取得许可**

- `third_party\LakeUI\LICENSE.txt` 35 149 B，正文头两行 `GNU GENERAL PUBLIC LICENSE / Version 3, 29 June 2007`，无运行库例外条款。
- `third_party\LakeUI\README.md:84` 原文："请注意，LakeUI 是收费软件，要在公开发布的产品中使用必须取得对应许可"；`:35` 为购买链接。
- 引用点：`FFF.Player.vbproj:294` `<HintPath>..\..\LakeUI\LakeUI\bin\Debug\…\LakeUI.dll</HintPath>`（**Debug 配置产物**）；
  `FFF.Recorder.vbproj:76` `<PackageReference Include="LakeUI.Notifications" Version="1.0.0" />`（许可 **未判定**：本机 NuGet 缓存无包可读）。

⚠ 本产物链把 GPL-3.0-only 的库静态引用进宿主 exe。是否必须先取得作者许可、以及由此产生的整体许可后果：**需法务**。

## 5. NuGet 直接引用（`FFF.Player.vbproj:77-82`）

| 包 | 许可（`.nuspec` 的 `<license>` 原文） |
|---|---|
| `Svg 3.4.8` | **MS-PL** |
| `Vortice.Direct2D1 / Direct3D11 / DirectComposition / DirectX / DXGI 3.8.3` | **MIT** ×5 |

## 6. 运行时形态与内核源码本体

- `tools\发布3FP.ps1:45` 为 `--self-contained false`（框架依赖，需本机 .NET 10 桌面运行时）；
  `tools\发布3FP单文件.ps1:40` 为 `PublishSingleFile + IncludeNativeLibrariesForSelfExtract`，产物 ~45 MB < 单个 `avcodec` 76 MB ⇒ FFmpeg 未内嵌。
- **本仓源码的归属**：`HEAD` 251 个被跟踪文件里 **212 个（84.5%）与上游根提交逐字节相同**（作者 `1059 Studio <vb1059@outlook.com>`）；本仓自研部分为 33 改 / 6 增 / 3 删。
- `LICENSE.txt:3` 是**未填写的 MIT 模板**（`Copyright (c) [year] [fullname]`）⇒ "本仓按 MIT 授权"缺少可读回的版权人声明；`FFF.Native.rc` 的 `LegalCopyright` 值为 `"MIT License"`。两者的口径差异在此登记，修正需要版权人本人署名（**不由本仓使用者代填**）。

## 7. 随包发布仍需处理（缺口，不是结论）

1. §1 的 GPL 文本与"对应源码"要约（gpl+nonfree 构建）——未取得前不应对外分发。
2. §3 的许可**原文**缺口（2026-11-07 更新）：已收 libiamf（LICENSE+PATENTS）、OAR、pffft（头部声明）、
   abseil 四族入 `licenses\`（`tools\发布3FP.ps1` 的许可步骤需同步收集该目录）；仍缺 **FLAC / opus**
   （Xiph 原文）、**OBR 单独文本**（上游无独立文件）、**fdk-aac**（libiamf 仓 `dep_codecs\` 只给预编译件；
   其代码当前零进包，见 §3 处置）。
3. §4 LakeUI 的许可取得状态待确认。
4. 本文件与 §2 各 port 的 `copyright` 由 `tools\发布3FP.ps1` 的许可步骤随包；缺件默认中止。
