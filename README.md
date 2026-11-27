# VIctory 进度跟踪：GTA6（Xbox Series X 版）在 Windows 上跑到哪了

这份笔记跟着 [Belobug/VIctory-Belobug-AMD](https://github.com/Belobug/VIctory-Belobug-AMD) 走。上游不是我的项目，我只是提 PR、在自己的笔记本上跑。内容分两种：从上游 README 和 docs 里读到的，和我自己跑出来的。前者会写明是读来的，没写的就是我这边实测。

最后更新：2026-11-27 晚上。

## 现在的状态

上游现在最新的公开线是 `a-publier`（`4eb6cfd`，11-27 06:27 +08），比之前的 `cd8dfc0` 多 49 个提交；`main` 是另一条更旧的线。仓库 HEAD 也指向它，直接按它拉。

我把 fork 的 6 个提交 rebase 到 `4eb6cfd` 上，`system` 现在到 `12fc45e`，旧 tip 留在 `backup-before-rebase2`（`bb666a6`）。只有 `gameinput.rs` 一处冲突：上游把单 pad 的兜底从 `stand_in` 改成了 `fallback(&keys)`，我保留上游语义，在上面叠加 `VICTORY_PAD_LOG` 的诊断输出。

真正的变化是改用优化构建跑。之前所有 run 都是 `cargo build -p victory` 的 debug 构建，工作区代码 opt-level 0；换成 `--profile diag --features diagnostics`（optimized + debuginfo）后，同样的世界场景帧时间大约减半，list call 记录从 40,344 次 9.3 秒降到 42,286 次 2.1 秒，等管线创建的 1,154 ms 也归零。最近三组实测（都是 diag 构建）：

| 构建 | 世界帧时间中位数 | 稳态标题栏 |
|---|---|---|
| fork（旧基线 `cd8dfc0`） | 515 ms | 1.9 fps / 514 ms |
| 纯 a-publier | 485.6 ms | 2 fps / 470 ms |
| fork rebase 到 a-publier | 503.1 ms | 2 fps / 474.6 ms |

三组差异都在场景波动范围内，说明 rebase 没带来性能回退。画面是真出图的：天空、楼房、成排电线杆、亮着的窗、路灯、树，HUD 也在。现在这台机器上更早撞到的是内存和显存，不是 wave64。

## 上游是什么

`Belobug/VIctory-Belobug-AMD` 是 VIctory 的一个分支。VIctory 本身是 Xbox 兼容层，目标是把 Xbox Series X 版的 GTA6 跑在 Windows 上。这个分支主要在 RX 9070 XT 上调，作者说他是从 `avatarmanz/VIctory-7900XTX` clone 下来的，我看到的 star 是 1,744。

作者自己机器（i9-12900K / 64G / 7900 XTX / 1440p / Series X 档位）的成绩：

| 场景 | 每帧 |
|---|---|
| 序章室内 | 50–75 ms |
| 首个开船任务 | 46–53 ms |
| 街头追车 | 100–165 ms |
| 营地步行 | 60–90 ms |
| 飞行 | 42–57 ms |

菜单是 30 fps，最长一次 41 分钟没崩，私有内存 40–46 GiB，所以建议 64G 内存。作者说他这套场景两天前还是 280 ms 一帧，现在是 63 ms 左右，过程记在 `docs/findings.md`：280 → 124 → 103 → 75 → 68 → ~63 ms。

已知毛病也是他写的：户外树和远处建筑会闪；不加 `-RayTracing` 时暗房间太黑；流式纹理来得晚；第一次跑驱动建管线会卡，`victory warm` 可以预热；在世界里录帧会触发 title 自己的 watchdog 退出。

光追方面，`VICTORY_BVH_BUILDER=1` 才有间接光，但每帧多 40 ms 左右，所以默认关，`VICTORY_AMBIENT=1` 是便宜的替代。有个没解决的问题：动态 BVH 结构会互相覆盖，light-10 里 65 个结构只有 8 个是好的，light-15 里 275 个只有 48 个。原因是 title 先问压缩后的大小、隔一两帧才来读，而我们的 submission 最多落后两帧，读到的就是旧 slot。

另有一批补丁以 issue 形式排着队，都是 11-26 12:40–14:09 UTC 提的，好几条注明 Codex 帮忙实现：#250 帧节奏、#251、#253、#255–#258、#260、#261、#263、#264。

## 我提的东西

比较完整的一件是上游 issue #104 的实现：`victory system`。它不加载 title、也不需要游戏文件，直接把本机能不能跑、会撞到什么打出来，这样测试报告能和具体的 build 对上。PR 在 [#268](https://github.com/Belobug/VIctory-Belobug-AMD/pull/268)。

主要动了这几处：`crates/victory-gpu/src/system.rs`（`Facts` / `Cpu` / `Memory` / `Gpu` 和阻塞项规则，9 个单测，用合成数据，不需要 GPU）、`system/gather.rs`（DXGI 枚举、每块卡建 `D3D12CreateDevice(FL_12_0)` 再 `CheckFeatureSupport` 问 waves / SM / mesh tier / binding tier / upload heap，按三档尺寸试 descriptor heap，加注册表里的 `CurrentBuild`+`UBR`、`ProcessorNameString`、`ntdll` 的 `wine_get_version`）、`apps/victory/src/main.rs`（子命令和带 commit 的 `--version`）、`apps/victory/build.rs`（从 `git rev-parse --short=7 HEAD` 生成版本号）。

跑过的：`cargo test -p victory-gpu --lib system::` 9 个通过，`cargo test -p victory` 2 个通过。clippy 1.98 下新增代码没告警，仓库里原有的 25 条 `manual_is_multiple_of` 跟这次无关。

之后跑真机又补了三次修改：

- `--version` 里的 commit 一直是旧的。`build.rs` 只盯 `.git/HEAD`，可提交之后那个文件内容不变，变的是 `.git/refs/heads/system`。现在也看分支文件和 `packed-refs`。
- 内存那条原来写成了判决，说低于 43 GiB 就会 1455。但我在 40.5 GiB 空闲时照样跑到了世界，说明这个门槛我测不出来。现在改成「标题工作集据称 40–46 GiB，长会话可能撞 1455」，数字保留。
- 加了 `VICTORY_PAD_LOG=1`，标题来要输入时把 pad 读数打出来，好区分「前端不反应」和「前端根本没来问」。本机没有 D3D12 调试层（`0x887A002D`），驱动消息拿不到。

fork 上现在比上游多 6 个提交：`12fc45e` `8d79a5d` `1bdbd8f` `033d902` `5d419fd` `579d17a`。rebase 前的旧哈希 `bb666a6` 那一串保存在 `backup-before-rebase2` 分支里。

## 这台机器

```
VIctory 0.1.0 (12fc45e), diag build（optimized + debuginfo）
Windows 11, build 26100.3624
CPU: AMD Ryzen 9 8945HX with Radeon Graphics, 32 threads, AVX2 yes
Memory: 31.8 GiB RAM (18.1 GiB free), commit 42.5 GiB free of 75.7 GiB
GPU (used): NVIDIA GeForce RTX 5060 Laptop GPU [10de:2d59], driver 32.0.15.9621, 7899 MiB dedicated, 16280 MiB shared
    shader model 6.7, mesh shaders yes, waves of 32 to 32 lanes, resource binding tier 3, view heap of 1000000 views, enough for the title's, GPU upload heap yes
Known blockers:
- Waves of 32 to 32 lanes: console wave64 compute programs are dropped where their cross-lane code cannot be split (#9)
- 42.5 GiB of commit free: the title's own working set is said to run 40-46 GiB, so a long session may meet 1455 (#15)
```

内存空闲量两次运行之间会差一两个 GiB，别当固定值看。

关于这台机器有四点值得说。32 lane 是 NVIDIA 的常态，能不能跑 compute 全看上游 #9 的 wave64 模拟。着色器模型这里踩过一个坑：`D3D_SHADER_MODEL_6_0` 是 `0x60` 不是 100，所以设备回的 `0x67` 就是 6.7；早先版本把本机报成 6.3，还顺手写了「标题的 shader 要 6.6」这条假阻塞，改完以后输出 6.7，运行时 mesh pipeline 也确实建得起来，两边对上了。commit 数字那条见上面。最后是 view heap：`victory system` 说 100 万个 `enough for the title's`，但真跑时标题要 124.5 万个，这句判断是错的，后面按实测改。

## 真机跑到哪

游戏文件在 `E:\GTA6.Xbox.v1.0.2.1-CYBERLEEK`，208.6 GB、110 个文件（`levels.rpf` 70.92 GB、`audio.rpf` 26.58 GB、`movies.rpf` 8.23 GB）。

三个环境变量必须设，因为 `C:\Users\Y\AppData\Local` 在本机不可写，默认缓存目录会直接失败：

```powershell
$env:VICTORY_CACHE_DIR='E:\Deep\victory-data\cache'
$env:VICTORY_DATA_DIR='E:\Deep\victory-data\data'
$env:VICTORY_GAME_DIR='E:\GTA6.Xbox.v1.0.2.1-CYBERLEEK'
```

标题画面是 Xbox 账号选择（`ACCOUNT PICKER (Y) / SCREEN NARRATION (OFF) (X) / PLAYER (A)`），30 fps、13 draws/frame。键盘能用，但窗口必须在最前：`SetForegroundWindow` 单独调会被拒，先敲一下 Alt 再调就成。按键映射在 `crates/victory-hle/src/modules/gameinput.rs`，Enter=A、Backspace=B、Esc=Menu、Tab=View、方向键=DPad。首次启动的语言、导航体验、法律声明、隐私政策和服务条款两个勾选、Confirm，这一串都能过去。

然后是世界流式加载，日志开始打开 `core_restofworld.rpf`、`story000_restofworld.rpf`。用 diag 构建最近一次的稳态：2 fps，470–515 ms 一帧，4,775–4,886 draws/frame；加载初期会到 1.5–1.9 秒一帧、5,000–10,796 draws/frame。掉帧 172–220/frame 来自 ray-tracing structure builder，另有 7 个 wave64 compute。私有内存 21.6–22.5 GB，系统只剩 0.9–1.5 GB 空闲；显存 in use 7,414 MB，预算 6,582 MB（8G 卡）。画面是完整的世界，不是黑屏，截图 `run8_b.png`。

## 现在卡在哪

**wave64。** 之前那条 `host pipeline refused error=E_INVALIDARG shaders=[compute 02384a60fda6f2d0]` 我一开始当成驱动或 PSO 故障，其实是 #9 在 PSO 创建这一步露头。查法是这样的：清空缓存用 `victory translate --warm` 重翻那个 shader，缓存里是 `VSH2` 包装（magic + 源长度 + source + stage + DXIL 长度 + DXIL），解出来的 DXIL 用 `dxv.exe` 校验是 `Validation succeeded`，说明字节码合法；再用 `dxc.exe -dumpbin` 看，`NumThreads=(128,1,1)`、`WaveSize=64`。本机 wave 只有 32 宽，`emit/lower.rs:1841-1852` 只在能被 32 半波切开时才降到 32，切不开就留着 64，`CreateComputePipelineState` 只能报参数错误。

我在 `pipeline/host.rs` 加了个判断：`victory_shader::dxbc::wave_size` 从 `PSV0` 读容器声明的宽度（`min == max` 才算硬声明），跟 `host_settings()[0]` 比，超过就直接 `return None`，打一行

```
no host pipeline: the compute program asks for wider waves than the host runs
    shaders=[compute 02384a60fda6f2d0] wave=64 host_wave=32
```

改完再跑，`host pipeline refused` 不再出现，改成 7 个不同的 compute 程序各打一行 `wider waves`（run13：`02384a60`、`369cbe90`、`2da4beb3`、`6f225b64`、`5dc3b259`、`926c46ba`、`6b82bc7d`），世界照常推进到 10,796 draws/frame。这个改动在 `12fc45e`，PR #268 下面补了复现步骤。

**ray-tracing stage 没翻。** 最近几次的日志是 `a stage of the ray-tracing structure builder, which is off`（`VICTORY_BVH_BUILDER=1` 才开），run11 是 220 drops/frame、run12 是 172；run8 早期一次量到 677.6 draws/frame（更早 1,548）。仍是世界内最大的单项掉帧。

**view heap 上限。** 启动时 `host view heap capped at 1000000; views past it read a null slot count=1245184`：标题要 124.5 万个 view，NVIDIA 的 1,000,000 上限差约 24.5 万个，读到的就是 null。`no free dynamic descriptor slots` 在大场景超 2^18 张表时也会出现，`VICTORY_DYNAMIC_SLOTS` 能调，但余量同样被这 100 万压死。

**内存和显存。** run13 私有内存 21.6 GB、系统剩 877 MB 空闲（run11 是 22.5 GB）；`host copies` 4.5–4.7 GB、`host textures` 4.4 GB、`shared buffers` 274,376 个共 7.5 GB。显存这边 a-publier 新加的统计会直接说超预算：`the card's own: budget 6582 MB, in use 7414 MB`。在这台 31.8 GiB / 8G 显存的机器上，这条比 RT 更早成为限制：run11 最后系统只剩 1.5 GB 空闲，进程随后消失，日志里没留下明确的原因行；run13 是我在只剩 877 MB 空闲时手动停掉的。

另外几个小的：`placed unordered texture view not mirrored`（run13 里 11 次）、按索引取的纹理在上传后变了（38 次）、`constant buffer not 256-byte aligned`（1 次）、`buffer view not on an element of its host copy`（1 次）、`a texture copied back could not be tiled`（1 次），还有 `CreatePipelineState` 被调 29 次全部返回 E_NOTIMPL。另有一次 `method not implemented object="command queue" slot=0xb8`，对应注释里的 Suspend。

还有 5 个 shadow-mask trace 程序翻不过：要从堆里按计算索引取 descriptor 再读结构地址，需要 host 按 slot 提供地址表。目前 title 没创建这些程序，收益是 0，属于先修好等着的类型。

到现在为止只是能渲染、能看到画面和 HUD，还没真正玩。

## 本机环境与跑法

没有 MSVC、不是管理员，用的是 mingw + rustup 的自包含工具链：

```powershell
# 1) Rust（gnu 目标，绕开 rust-toolchain.toml 里的 msvc）
$env:CARGO_HOME='E:\Deep\_toolchain\cargo'; $env:RUSTUP_HOME='E:\Deep\_toolchain\rustup'
$env:RUSTUP_DIST_SERVER='https://rsproxy.cn'
E:\Deep\_toolchain\rustup-init.exe -y --default-host x86_64-pc-windows-gnu --default-toolchain 1.98.1 --profile minimal --no-modify-path

# 2) mingw-w64（tuna msys2 ucrt64 包解到 E:\Deep\_toolchain\msys）
#    gcc 16.2.0 + binutils + gmp/isl/mpc/mpfr + zlib + zstd + gettext + winpthreads
$env:PATH='E:\Deep\_toolchain\cargo\bin;E:\Deep\_toolchain\msys\ucrt64\bin;' + $env:PATH
$env:RUSTUP_TOOLCHAIN='1.98.1-x86_64-pc-windows-gnu'
$env:CARGO_TARGET_X86_64_PC_WINDOWS_GNU_LINKER='E:\Deep\_toolchain\msys\ucrt64\bin\x86_64-w64-mingw32-gcc.exe'
$env:TMP='E:\Deep\_toolchain\tmp'; $env:TEMP=$env:TMP   # 系统 TEMP 不可写

# 3) 构建与运行
cd E:\Deep\VIctory-Belobug-AMD
cargo build -p victory                                   # 日常开发用，跑起来慢但编译快
cargo build --profile diag -p victory --features diagnostics   # 测性能/看掉帧原因用这个
.\target\debug\victory.exe system
.\target\diag\victory.exe system
```

测帧时间一定用 `diag`，`debug` 下工作区代码是 opt-level 0，同样的场景会慢一倍以上。`diag` 保留 diagnostics 和 debuginfo，掉帧原因、每帧统计都还在。

测试这边，`cargo test --workspace` 只有 `victory-loader --test load_probe` 会挂，原因是 GNU ld 不认 MSVC 的 `/DELAYLOAD:`，属于环境限制不是仓库 bug。其余全过，`victory-gpu` 411 个测试里有 278 个真跑在 D3D12 上，大约 6.5 分钟。

## 时间线

- 11-26 晚：实现 #104，开 PR #268。
- 11-27 凌晨：跑通首次启动流程到世界加载，修掉 shader model 解码、`--version` 的 commit、加 pad 日志。
- 11-27 上午：上游 force-push 到 `cd8dfc0`，rebase 跟上，世界画面出来；定位 wave64，`bb666a6` 让它报得清楚。
- 11-27 晚上：拉最新的 `a-publier`（`4eb6cfd`，比基线多 49 个提交），rebase `system` 到 `12fc45e`；改用 `diag` 优化构建，帧时间约减半；确认这台机器当前最早的墙是内存和显存。
