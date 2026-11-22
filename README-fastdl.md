# 64Gram — Configurable Download Speed Boost (`fastdl`)

[English](#english) | [中文说明](#中文说明)

---

## English

### Why this fork exists

On **high-RTT links** (e.g., when your Telegram account's DC is on another continent or routed through a distant hop with 300–500 ms RTT), Telegram Desktop / 64Gram video streaming and downloads hit a hard client-side throughput ceiling regardless of your actual bandwidth:

- **Default streaming window**: `128 KB` chunks × `4` (official Telegram Desktop) or `8` (64Gram) concurrent requests = only `0.5–1 MB` in flight.
- At ~420 ms RTT (+ server processing ≈ 720 ms per request), throughput caps around **0.65–1.4 MB/s**, even on a 300 Mbps connection where the mobile Telegram client (which uses `1 MiB` chunks) easily achieves **10–20+ MB/s** on the exact same network.

This branch adds a **restart-applied "Download speed boost"** setting in **Settings → 64Gram → Network** (right next to the existing *Upload speed boost*) with 3 tiers:

| Tier | Chunk size | Streaming concurrency | Streaming preload ahead | In-flight window | Recommended for |
|---|---:|---:|---:|---:|---|
| **Disabled (Default)** | 128 KB | 8 | 8 parts (1 MB) | 1 MB | Low-RTT links / slow connections |
| **Medium** | 1 MB | 8 | 4 parts (4 MB) | 8 MB | General high-RTT links / lower RAM / non-Premium |
| **High** | 1 MB | 16 | 8 parts (8 MB) | 16 MB | High-RTT links + high-bitrate 4K video streaming |

### Key engineering fixes included

1. **Startup-frozen chunk size**: Active chunk size and streaming limits are captured once at startup so changing the setting right before restart cannot mutate active `Reader` instances or cache serialization.
2. **CDN multi-chunk SHA-256 verification**: `DownloadMtprotoTask::checkCdnFileHash()` validates `1 MB` CDN payloads in `fileHash.limit` (128 KB) sub-blocks and fetches missing hashes at the exact missing sub-offset, avoiding `Wrong cdnFileHash` failures on CDN-redirected channels.
3. **Cache safety across mode switches**: Streaming slice/header byte limits stay capped at `8 MB` / `10 MB` (within `Storage::Cache::Database` `maxDataSize = 10 MB`), mismatched part-size entries from previous modes are cleanly rejected as cache misses on switch, `VideoPreload` caps serialized size below `kMaxFileInMemory`, and `DownloadManagerMtproto` bounds per-session windows to max `4 MB` (`32 MB` across 8 sessions).

### Upstream PR & maintenance status

- **Pure C++ PR branch**: [`feat/download-speed-boost`](https://github.com/2763981847/tdesktop/tree/feat/download-speed-boost) (contains only source & localization changes, zero CI changes).
- **Maintenance policy**: Once merged into upstream [`TDesktop-x64/tdesktop`](https://github.com/TDesktop-x64/tdesktop), Releases here will be archived in favor of official 64Gram builds (which also cover macOS and Linux). Until then, Windows x64 portable builds are provided on a best-effort basis.

### Trust, security & verification

Never blindly trust an unsigned binary with your Telegram account. Every Release here is:

1. **Built exclusively by public GitHub Actions** directly from this repository's [`win.yml`](.github/workflows/win.yml) (with `-D DESKTOP_APP_DISABLE_AUTOUPDATE=ON` and the same open-source API credentials `611335` used by 64Gram nightly).
2. **Verifiable via SHA-256 & GitHub Build Provenance**:
   ```bash
   # Verify SHA-256 checksum
   sha256sum -c SHA256SUMS.txt

   # Verify GitHub Actions build provenance attestation (requires GitHub CLI)
   gh attestation verify 64Gram-x64-portable.zip --repo 2763981847/tdesktop
   ```
3. **100% self-buildable**: Fork this repository → open the **Actions** tab → select **Windows.** → click **Run workflow** on branch `fastdl` to produce your own binary directly under your own GitHub account.

> **Notes**:
> - Windows SmartScreen may warn because the binary is unsigned (open-source personal build without a commercial EV certificate).
> - Auto-update is disabled at compile time so official 64Gram updates won't silently overwrite this build; update manually via GitHub Releases.
> - Licensed under **GPLv3 with OpenSSL exception** (see [LICENSE](LICENSE) and [LEGAL](LEGAL)).

### Test scope

- Tested on **Windows 11 x64** with a **Telegram Premium** account over ~420 ms RTT links (sustained **15–25 MB/s** vs **0.65 MB/s** on official Telegram Desktop).
- Feedback from **non-Premium** accounts and slower networks is welcome via GitHub Issues.

---

## 中文说明

### 解决什么问题

当你与 Telegram 账号所在数据中心（DC）之间存在**较高往返延迟（高 RTT，例如跨洲 300–500 ms）**时，即使本地带宽充足、同一网络下手机端能跑 10–20+ MB/s，电脑版 Telegram Desktop / 64Gram 的视频流式播放与下载也会卡在极低速度：

- 原版流式下载在途窗口写死为 `128 KB × 4`（官方 TG Desktop）或 `128 KB × 8`（64Gram），全局仅 `0.5–1 MB` 在途；
- 在 ~420 ms RTT 下，速度上限被锁死在 **0.65–1.4 MB/s** 左右。

本分支在 **设置 → 64Gram → 网络（Settings → 64Gram → Network）** 中、紧挨现有的「上传加速」新增了独立的**「下载加速（Download speed boost）」**选项（默认关闭，切换后重启生效）：

| 档位 | 分片大小 | 流式并发请求上限 | 播放预加载 | 最大在途窗口 | 适用场景 |
|---|---:|---:|---:|---:|---|
| **默认（原版）** | 128 KB | 8 | 8 片（1 MB） | 1 MB | 低延迟网络 / 低速网络 |
| **中档（Medium）** | 1 MB | 8 | 4 片（4 MB） | 8 MB | 一般高延迟链路 / 兼顾内存与非会员 |
| **高档（High）** | 1 MB | 16 | 8 片（8 MB） | 16 MB | 高延迟链路 + 高码率 4K 在线播/下载 |

### 同时修复的关键边界问题

1. **启动期冻结参数**：分片大小与流式上限在启动时一次性读取并冻结，切换选项触发重启前不会影响正在播放的流式 Reader 或缓存落盘。
2. **CDN 分段哈希校验**：修正 `checkCdnFileHash()`，使 `1 MB` 分片能按 Telegram CDN 返回的 `128 KB` 哈希块逐段完成 SHA-256 校验，并精确从缺失偏移补拉哈希，避免热门公开频道触发 CDN 重定向时报 `Wrong cdnFileHash`。
3. **旧缓存兼容与内存封顶**：保持流式 slice / header 字节上限严格 ≤ `8 MB / 10 MB`（不触发缓存库 `maxDataSize = 10 MB` 断言）；在 128 KB 与 1 MB 档位间来回切换时，不匹配分片尺寸的旧缓存条目会安全按未命中处理；`VideoPreload` 序列化上限受 `kMaxFileInMemory` 约束；普通文件下载单会话窗口封顶 `4 MB`（8 会话最高 `32 MB`）。

### 使用方式、安全校验与退场说明

- **仅提供 Windows x64 便携版**：解压 `64Gram-x64-portable.zip` 得到 `64Gram.exe`（可与原 64Gram 便携目录的 `tdata` 共用；已编译期关闭自动更新，不会误被官方更新覆盖）。macOS / Linux 将在上游 PR 合并后由 64Gram 官方 CI 统一提供。
- **不信任第三方未签名程序？完全理解！** 你可以：
  1. 核对 `SHA256SUMS.txt` 并用 `gh attestation verify 64Gram-x64-portable.zip --repo 2763981847/tdesktop` 验证该文件确由本仓库公开 GitHub Actions 构建产出；
  2. 或直接 **Fork 本仓库 → 进入 Actions → 选择 `Windows.` → 点击 `Run workflow`（分支选 `fastdl`）**，在自己账号下零信任一键自编译。
- **已知测试范围**：目前主要在 **Windows 11 x64 + Telegram Premium** 下完成高延迟回归验证（~420 ms RTT 下由 0.65 MB/s 提升至 15–25 MB/s）；非会员账号及弱网环境下的体感欢迎在 Issue 反馈。
- **退场策略**：本改动同时以纯净分支 [`feat/download-speed-boost`](https://github.com/2763981847/tdesktop/tree/feat/download-speed-boost) 向上游 64Gram 提交 PR；一旦上游合并，本仓库 Release 将归档并引导回官方 64Gram。
