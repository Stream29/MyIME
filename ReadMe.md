# MyIME

## 概要

MyIME 记录当前部署在 Ubuntu 桌面端和 Android 移动端的一套跨设备输入方案。该方案包含文字输入、Rime 用户数据同步和语音输入三部分。

- 桌面端和移动端都以 Fcitx 5 与 Rime 作为中文输入基础。
- 中文方案和基础词库使用雾凇拼音。
- 桌面端日语输入使用 Mozc。
- 移动端日语输入使用 Rime Kagiroi 方案。
- 英文输入使用 Fcitx 提供的英文键盘布局。
- Rime 同步数据通过阿里云 OSS Bucket `rime-data` 在设备之间传递。
- 桌面端和移动端的语音识别都使用阿里云百炼的 `qwen3-asr-flash-realtime`。
- 桌面端使用键盘上的 Copilot 键触发语音识别。
- 输入法、同步工具和语音客户端的源代码仓库均公开；阿里云 OSS 和百炼是外部托管服务。

## 整体结构

| 能力 | Ubuntu 桌面端 | Android 移动端 | 共享服务 |
| --- | --- | --- | --- |
| 中文输入 | Fcitx 5、fcitx5-rime、Rime、雾凇拼音 | Fcitx5 for Android、Rime 插件、雾凇拼音 | 无 |
| 日语输入 | fcitx5-mozc | Rime Kagiroi | 无 |
| 英文输入 | `keyboard-us` | Fcitx 英文键盘布局 | 无 |
| 用户数据同步 | Rime 原生同步、ossutil、systemd 用户定时器 | Rime 原生同步、RimeSync、RSAF | 阿里云 OSS `rime-data` |
| 语音输入 | fcitx5-vinput、PipeWire | DashVoice | 阿里云百炼 |

## 桌面端

当前桌面环境为 Ubuntu 26.04 LTS、GNOME Wayland 和 Fcitx 5.1.19。

Fcitx 输入法组包含：

- `keyboard-us`
- `rime`
- `mozc`

默认输入法为 `rime`。Rime 用户目录为：

```text
~/.local/share/fcitx5/rime
```

该目录包含雾凇拼音配置、Rime 用户词库和 `sync/` 同步目录。

### 桌面端同步

桌面端同步脚本位于：

```text
~/.local/bin/rime-oss-sync
```

同步流程为：

1. 使用 ossutil 将 `oss://rime-data/` 拉取到本地状态目录。
2. 将其他 installation ID 的目录复制到 Rime 的 `sync/` 目录。
3. 通过 Fcitx D-Bus 接口调用 Rime 用户数据同步。
4. 将本机 installation ID 对应的同步目录上传到 `oss://rime-data/<installation_id>/`。

`rime-oss-sync.timer` 在用户会话启动两分钟后首次执行，此后每十分钟执行一次。每次任务通过文件锁避免并发运行。

### 桌面端语音输入

桌面端使用 fcitx5-vinput 2.3.5。它由 Fcitx 插件和 systemd 用户服务 `vinput-daemon.service` 组成。

语音输入流程为：

1. Fcitx 插件从当前输入上下文接收语音触发键。
2. 后台服务通过 PipeWire 采集 16 kHz 单声道 PCM。
3. Bailian Streaming Provider 将音频发送到阿里云百炼。
4. 中间识别结果作为 Fcitx preedit 显示。
5. 最终识别结果提交到当前输入上下文。

当前模型为 `qwen3-asr-flash-realtime`。语言参数为空，由模型返回检测到的语言。

桌面端以键盘上的 Copilot 键作为语音识别触发键。在当前 Fn Lock 状态下，直接按 Copilot 键向系统发送 `Shift+Super+XF86Assistant`。

## Android 移动端

Android 默认输入法为 Fcitx5 for Android。中文输入由 Rime 插件和雾凇拼音提供；日语输入由同一个 Rime 实例中的 Kagiroi 方案提供。

Rime 同步目录位于 Fcitx5 for Android 的应用专用外部存储目录：

```text
/storage/emulated/0/Android/data/org.fcitx.fcitx5.android/files/data/rime/sync
```

### 移动端同步

移动端同步链路由 RimeSync 和 RSAF 组成。

- RimeSync 使用 Android Storage Access Framework 访问 Fcitx 的 Rime `sync/` 目录。
- RimeSync 在 Rime `sync/` 目录和用户选定的远端目录之间复制同步文件。
- RSAF 内置 rclone，并将配置好的阿里云 OSS 远端暴露为 Storage Access Framework 文档提供者。
- RSAF 中选定的远端对应 OSS Bucket `rime-data`。
- 移动端同步由用户手动触发。
- Fcitx 的 Rime 用户数据同步操作负责把同步文件合并回本机用户词库；配置重载由用户手动执行。

各设备使用不同的 Rime installation ID。OSS Bucket 中的一级目录以 installation ID 区分设备，设备不直接并发写入同一个目录。

### 移动端语音输入

移动端语音客户端为 [DashVoice](https://github.com/Stream29/DashVoice)，包名为：

```text
io.github.stream29.dashvoice
```

DashVoice 使用 Kotlin 和 Jetpack Compose 实现，并包含：

- Android `RecognitionService`
- 独立的 Android `InputMethodService`
- 基于 Room 的本地设置存储
- 基于 Kotlin 协程、Flow 和 kotlinx.serialization 的实时会话

DashVoice 使用 `qwen3-asr-flash-realtime`，不固定识别语言。默认服务端 VAD 阈值为 `0.0`，默认静音结束时间为 `400 ms`。识别完成后，独立语音输入法会返回之前使用的输入法。

## 同步数据

跨设备传递的是 Rime 原生同步生成的 `sync/` 目录内容，不是正在使用中的 LevelDB 用户数据库目录。

每个设备将自己的用户数据导出到：

```text
sync/<installation_id>/
```

其他设备的目录被放入本机 `sync/` 后，由 Rime 原生同步逻辑读取并合并。OSS 负责保存和传输这些按设备分目录的数据。

## 配置与凭据

- 桌面端 OSS 凭据保存在 `~/.config/rime-oss/ossutil.conf`。
- 桌面端百炼配置保存在 `~/.config/vinput/config.json`，该文件当前权限为 `600`。
- Android 端百炼 API Key 和 Base URL 以明文保存在 DashVoice 的 Room 数据库中。
- API Key、OSS AccessKey 和其他凭据不存放在本仓库中。
- 语音音频会发送到阿里云百炼服务。
- Rime 同步文件会存放在阿里云 OSS。

## 组件与源码

| 组件 | 用途 | 源码 | 许可证状态 |
| --- | --- | --- | --- |
| Fcitx 5 | 桌面输入法框架 | [fcitx/fcitx5](https://github.com/fcitx/fcitx5) | LGPL-2.1-or-later |
| fcitx5-rime | Fcitx 的 Rime 前端 | [fcitx/fcitx5-rime](https://github.com/fcitx/fcitx5-rime) | LGPL-2.1-or-later |
| Fcitx5 for Android | Android 输入法框架 | [fcitx5-android/fcitx5-android](https://github.com/fcitx5-android/fcitx5-android) | LGPL-2.1 |
| librime | Rime 输入法引擎 | [rime/librime](https://github.com/rime/librime) | BSD-3-Clause |
| 雾凇拼音 | 中文方案和词库 | [iDvel/rime-ice](https://github.com/iDvel/rime-ice) | GPL-3.0-only |
| Kagiroi | Rime 日语方案 | [rimeinn/rime-kagiroi](https://github.com/rimeinn/rime-kagiroi) | GPL-3.0 |
| Mozc | 桌面日语输入 | [google/mozc](https://github.com/google/mozc) | BSD-3-Clause |
| RSAF | Android rclone 文档提供者 | [chenxiaolong/RSAF](https://github.com/chenxiaolong/RSAF) | RSAF 为 GPL-3.0-only；内置 rclone 为 MIT |
| RimeSync | Android SAF 同步桥 | [zuiwuchang/android-rimesync](https://github.com/zuiwuchang/android-rimesync) | 仓库当前未包含许可证文件 |
| ossutil | 桌面端 OSS 文件同步 | [aliyun/ossutil](https://github.com/aliyun/ossutil) | MIT |
| fcitx5-vinput | 桌面端语音输入 | [xifan2333/fcitx5-vinput](https://github.com/xifan2333/fcitx5-vinput) | GPL-3.0 |
| DashVoice | Android 端语音输入 | [Stream29/DashVoice](https://github.com/Stream29/DashVoice) | 仓库当前未包含许可证文件 |

阿里云 OSS 和阿里云百炼属于外部托管服务，不属于上述开放源码软件组件。
