# VIctory 进度跟踪（Xbox Series X 版 GTA6 → Windows）

个人跟踪 [Belobug/VIctory-Belobug-AMD](https://github.com/Belobug/VIctory-Belobug-AMD) 的笔记：现状、我和上游的交互、本机验证结果。
上游不是我的项目；这里记录的是**我读到的**和**我验证过的**，两类分开写。

最后更新：2026-11-26（北京时间）。

---

## 一、上游现状（读自仓库 README 与 docs/，非我实测）

- 仓库：`Belobug/VIctory-Belobug-AMD`，默认分支 `rx9070xt`，HEAD `81ebb7b`（"RX 9070 XT fixes: magenta filter, VRAM eviction, keep_shown guard, lock-free compiler VEH"），★1,744 / fork 314。
- 它是 VIctory（Xbox 兼容层）的一个分支，专门在 AMD RX 9070 XT 上调；作者自述 cloned from `avatarmanz/VIctory-7900XTX`。

性能（作者机器 i9-12900K / 64GB / RX 7900 XTX / 1440p / Series X 档位）：

| 场景 | 每帧 | 折算 |
|---|---|---|
| 菜单 | — | 30 fps |
| 序章室内 | 50–75 ms | 13–20 fps |
| 首个开船任务 | 46–53 ms | 19–22 fps |
| 街头追车 | 100–165 ms | 6–10 fps |
| 营地步行 | 60–90 ms | — |
| 飞行 | 42–57 ms | — |

- 最长一次 41 分钟不崩；私有内存 40–46 GiB，建议 64GB 内存。
- 同一套场景两天前是 280 ms/帧，现在是 63 ms 左右（README 原话），优化链见 `docs/findings.md`：280 → 124 → 103 → 75 → 68 → ~63 ms。

已知毛病（上游 README）：
- 户外树木、植物、远处建筑闪烁；
- 不加 `-RayTracing` 时暗房间过暗；
- 流式加载的纹理来得晚；
- 第一次跑时驱动建管线会卡（`victory warm` 预热可缓解）；
- 在世界内录帧会触发 title 自己的 watchdog 退出。

光追（`docs/fork-ray-tracing.md`）：`VICTORY_BVH_BUILDER=1` 才有间接光（light-10/11/n19），但每帧多约 40 ms 卡时间，所以默认关；`VICTORY_AMBIENT=1` 是廉价替身。**已知未解**：动态 BVH 结构会互相覆盖——light-10 里 65 个结构只有 8 个是好的，light-15 里 275 个只有 48 个（title 先问压缩后大小、1–2 帧后才读，而我们的 submission 最多落后 2 帧，于是读到旧 slot）。

未合入的补丁（全部以 issue 形式排队，11-26 12:40–14:09 UTC 创建，多条注明 "Codex helped implement"）：#250 帧节奏、#251 间接 clamp 后清绑定缓存、#253 `in_parallel` 改用常驻线程、#255–#258 上传堆/纹理扫描/scratch 重试/`wait_fence` 锁、#260 测试跳过 16 lane 的 iGPU、#261 monitor 解析、#263 RX 9070 XT 手部五彩纸屑纹理、#264 测试报告模板。

---

## 二、我做的（2026-11-26）

### 实现上游 issue #104：`victory system`

不加载 title、不需要游戏文件，直接打印本机事实和阻塞项，让测试报告能和 build 对上。

- 提交：`942fef2`，分支 `system`，9 文件 / +880 行
- 上游 PR：**[Belobug/VIctory-Belobug-AMD#268](https://github.com/Belobug/VIctory-Belobug-AMD/pull/268)**

改了哪些：

| 文件 | 内容 |
|---|---|
| `crates/victory-gpu/src/system.rs` | `Facts` / `Cpu` / `Memory` / `Gpu` 类型、渲染、阻塞项规则、9 个单测（合成机器数据，不需要 GPU） |
| `crates/victory-gpu/src/system/gather.rs` | Windows 采集：DXGI 枚举、每个 adapter 建 `D3D12CreateDevice(FL_12_0)`、`CheckFeatureSupport`（waves / SM / mesh tier / binding tier / upload heap）、按 `probe_view_limit` 同款三档尺寸试 descriptor heap、`GlobalMemoryStatusEx`、注册表 `CurrentBuild`+`UBR`、`ProcessorNameString`、`ntdll` 的 `wine_get_version` |
| `apps/victory/src/main.rs` | `system` 子命令；`--version` 带 commit |
| `apps/victory/build.rs` | 从 `git rev-parse --short=7 HEAD` 生成 `VICTORY_COMMIT` / `VICTORY_VERSION` |
| `README.md` | 挨着其它命令加一段说明 |

验证（本机全部真跑）：

- `cargo test -p victory-gpu --lib system::` → 9 passed
- `cargo test -p victory` → 2 passed
- 本文件新增的代码在 clippy 1.98 下零告警（仓库里既有的 25 条 `manual_is_multiple_of` 类告警与本次无关）

---

## 三、本机实测（RTX 5060 Laptop）

```
VIctory 0.1.0 (81ebb7b), debug build
Windows 11, build 26100.3624
CPU: AMD Ryzen 9 8945HX with Radeon Graphics, 32 threads, AVX2 yes
Memory: 31.8 GiB RAM (17.7 GiB free), commit 42.6 GiB free of 75.7 GiB
GPU (used): NVIDIA GeForce RTX 5060 Laptop GPU [10de:2d59], driver 32.0.15.9621, 7899 MiB dedicated, 16280 MiB shared
    shader model 6.3, mesh shaders yes, waves of 32 to 32 lanes, resource binding tier 3, view heap of 1000000 views, enough for the title's, GPU upload heap yes
Known blockers:
- Waves of 32 to 32 lanes: console wave64 compute programs are dropped where their cross-lane code cannot be split (#9)
- Shader model 6.3: the title's shaders are written for 6.6 and above
- 42.6 GiB of commit free: the title asks for about 43 GiB and Windows refuses 1455 below that
```

三条读法：

1. **32 lane 的 wave** 是 NVIDIA 的常态，上游 wave64 模拟（#9）决定这里能不能跑 compute；这是跨厂商必须面对的一项。
2. **shader model 6.3** 是这台机器 `D3D12_FEATURE_SHADER_MODEL` 的真实回答（问 6.4–6.9 一律回 6.3），而 `crates/victory-shader/src/emit/host.rs:107` 自己的门要求 ≥6.6。这条是否与 #259（RTX 5070 Laptop 创 compute PSO 报 `E_INVALIDARG`）同源，等报 #259 的人贴这份输出就能确认。
3. **commit 42.6 GiB free** 低于 title 要的约 43 GiB，Windows 会直接 `1455`——这解释了内存小的机器为什么起不来。

---

## 四、下一步可做的方向（按我判断的收益排序）

1. **wave64 compute 模拟（#9）**：NVIDIA 上 32 lane，跨 lane 的 console compute 会被丢。要把 wave 内操作（ballot/any/all/shuffle/dpp）在 32 lane 硬件上等价展开，翻译器层面可离线验证（现成 `valu_tests.rs`/`wave64_tests.rs` 风格）。
2. **光追结构互相覆盖**（`docs/fork-ray-tracing.md` 的 Open 项）：需要 title 才能验证，但可以先把「按 slot 给地址表」的 host 侧脚手架和在 replay 里复现的用例做出来。
3. **内存 40–46 GiB**：lists 分配器 9GB、windows 7GB、uploads 4GB（`VICTORY_MEMORY_MAP=1` 可看）。与小内存机器直接相关。
4. **5 个 shadow-mask trace 程序翻不过**：从堆里按计算索引取 descriptor 再读结构地址，需要 host 按 slot 提供地址表；目前 title 没创建这些程序，收益为 0，属于「先修好，等 title 用」。

---

## 五、如何在本机复现

本机没有 MSVC、不是管理员，用的是 mingw + rustup 的自包含工具链：

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
cargo build -p victory
.\target\debug\victory.exe system
```

已知限制：`cargo test --workspace` 里只有 `victory-loader --test load_probe` 会失败，原因是 GNU ld 不认 MSVC 的 `/DELAYLOAD:` 链接选项（环境限制，不是仓库 bug）；其余全部通过（`victory-gpu` 411 个测试里 278 个真跑在 D3D12 上，耗时约 6.5 分钟）。
