# 世贸定制中控屏折腾全记录

> 一台 Android 8.1 老旧中控屏的完整"刷机式"改造实录：从无线 ADB 接入、桌面与导航栏替换、Root 尝试（失败）、到 IPTV / TVBox 影音方案落地，以及用一加手机远控这块屏。
>
> 记录时间：2026-11-23 ~ 2026-11-24

---

## 目录

- [设备档案](#设备档案)
- [阶段一：接入与初步控制](#阶段一接入与初步控制)
- [阶段二：桌面与导航栏改造](#阶段二桌面与导航栏改造)
- [阶段三：Root 尝试（最终放弃）](#阶段三root-尝试最终放弃)
- [阶段四：影音与 IPTV 方案](#阶段四影音与-iptv-方案)
- [阶段五：手机远控（VNC / scrcpy）](#阶段五手机远控vnc--scrcpy)
- [阶段六：TVBox 点播 + 直播](#阶段六tvbox-点播--直播)
- [阶段七：后台占用排查与清理](#阶段七后台占用排查与清理)
- [最终软件清单](#最终软件清单)
- [踩坑与技术结论](#踩坑与技术结论)

---

## 设备档案

| 项目 | 参数 |
|---|---|
| 设备 | 世贸定制中控屏（酒店/商用定制机） |
| 系统 | Android 8.1 / API 27 |
| 芯片 | MediaTek MT8167，型号 T6770_L37RME_32 |
| 架构 | armeabi-v7a（32 位） |
| 分辨率 | 1024 × 600 |
| 内存 | 1 GB（可用长期紧张，仅数十 MB） |
| 网络地址 | `192.168.31.114` |
| 无线 ADB | `192.168.31.114:5555`（开机自启脚本持久化） |
| USB 序列号 | `LZMA2Y000090354` |
| Bootloader | 已解锁（Root 未成功，保持解锁状态） |
| 附带设备 | 一加 13T（PKX110，Android 17，arm64，`192.168.31.151:5555`） |

---

## 阶段一：接入与初步控制

**目标**：搞到这块屏的控制权，替换掉定制桌面。

- 设备在内网 `192.168.31.114`，先尝试无线 ADB，后用 USB 有线接入（`LZMA2Y000090354`）。
- 用 `CleanYourTV` 类工具 + 手动 `pm disable` 清理定制项目和预装应用。
- 亮度通过系统设置调到 50%。
- 全机 APK 打包备份（`pm list packages` + `pm path` + `pull`），留作回滚。

**无障碍导航栏**：因原生返回键被定制系统占用，改用开源项目 **VirtualSoftKeys**（`tw.com.daxia.virtualsoftkeys`）做悬浮导航栏。改造点：

- 弹出后自动隐藏时间改为 3 秒；
- 默认触屏宽度 30%；
- 播放视频时自动隐藏：把 `windowManager` 的隐藏从 `setVisibility(GONE)` 改为 `removeView()`，否则隐藏后的透明 View 会继续吞掉触摸事件，与 video app 冲突。
- 产物：`vsk/virtualsoftkeys-release.apk`

---

## 阶段二：桌面与导航栏改造

尝试了多个 GitHub 苹果 TV 风格桌面：

| 桌面 | 结果 |
|---|---|
| **Veil Lite** (`com.veil.launcher`) | ✅ 最终采用，支持触屏、稳定 |
| Aerial | ❌ 磁贴无触摸响应，判定为 MTK 固件兼容问题 |
| 若干近期更新的 Leanback / Apple-TV 风格启动器 | ❌ 图标缺失、体验差，弃用 |

- B 站第三方客户端 **BBLL** (`com.xx.blbl`) 安装成功。
- **hongguo-tv** (`rocks.aston.hongguotv`，来自 `aston314/hongguo-tv`) 安装成功。
- 网络排障：下载慢时发现云界线代理配置问题；代理出口 `127.0.0.1:7892` 可用，而 WorkBuddy 注入的 `127.0.0.1:60338` 会黑洞 `github.com` 主站（详见踩坑章节）。

---

## 阶段三：Root 尝试（最终放弃）

**目标**：Root 后禁用更多系统组件。

过程：

1. 确认 `oem unlock` 选项 → 进 fastboot，用 Zadig 装 WinUSB 驱动成功，UAC 无法弹窗则用注册表静默提权。
2. `fastboot oem unlock` 成功解锁 Bootloader。
3. 试图提取 `boot.img` / `preloader`：MTKClient / BROM / DA 多路径尝试，机器反复卡在 fastboot mode、boot mode、机器人倒地"无命令"状态，多次拔插/按键/重启。
4. **始终无法稳定提取 boot.img，Root 放弃**，Bootloader 保持解锁。

重启恢复后：重新推入桌面与导航栏，删除原定制桌面与不需要的启动项/应用。

> 结论：MT8167 这块老机的 boot/preloader 提取是真正的拦路虎，"一键 Root"在这里不现实。

---

## 阶段四：影音与 IPTV 方案

### 无线 ADB 持久化

自研 `adbpersist` App（`com.example.adbpersist`）：注册 `BOOT_COMPLETED` / `LOCKED_BOOT_COMPLETED` 广播，开机执行

```java
Runtime.getRuntime().exec(new String[]{"setprop","service.adb.tcp.port","5555"});
Runtime.getRuntime().exec(new String[]{"ctl.restart","adbd"});
```

踩坑：纯 receiver 无 Activity 会被系统置为 stopped 状态收不到广播 → 加一个 `MainActivity`，安装后手动启动一次解除 stopped。

> 现成方案（Termux:Boot、adb-auto-enable 需 Android 14）在此机均不适用，故自研。

### IPTV：mytv-android + 纯直链精选源

- 播放器：`mytv-android` v7a (`com.github.mytv.android`)，用内置 Web API (`http://192.168.31.114:10591`) 写入本地源。
- 源不稳定排查链：
  - ChinaIPTV 多为网页跳转 + 失效；
  - vbskycn 央视不全；
  - dainifei 多为移动 IPv6，联通跨网不通；
  - mgtv / 央视频 WebView 线路在老固件 OMX 解码报错——**IJK、TBS X5 都救不了**；
  - Arirang 纯直链 30fps 正常 → 证明播放器解码本身没问题，问题在 WebView 线路。
- 最终用 `build_strict.py` 严格检测真实 m3u8、排除网页跳转，生成 **258 频道纯直链精选源** `iptv_pure.m3u`，实测可播。

> 核心结论：**这台老机只能可靠播原生 m3u8 直链，播不了需要网页解析的 WebView 线路。**

---

## 阶段五：手机远控（VNC / scrcpy）

**目标**：像 VNC 一样用一加手机控制中控屏。

| 方案 | 结果 |
|---|---|
| droidVNC-NG (`net.christianbeier.droidvnc_ng`) | ❌ 系统缺 `MediaProjectionPermissionActivity`，无障碍手动写入被清、启用即"已停止运行"；fallback 仅 SDK≥30，此机 API27 不可用 |
| **scrcpy** (`org.client.scrcpy`) | ✅ 一加手机装客户端，输入 `192.168.31.114` 无线 ADB 直连成功，实测 `Remote wm size: 600x1024`，远控可用 |

> scrcpy 是目前唯一实际可用的手机控制方案。

---

## 阶段六：TVBox 点播 + 直播

**本体**：TVBoxOSC 俊于版 32 位 python 包 `tvbox-python32.apk`（`com.github.tvbox.osc.jun`），SHA256 与官方发布一致。

> 注：FongMi/TV 原仓库已删（404）；该版本为**单仓配置**，一次只能用一个配置地址。

### 点播源（配置历史可切换）

- ✅ **主源** `https://xiaolong69.github.io/tv/1.json`（47 站点）——端到端验证：搜索「两京十五日」返回 28/33 源、21 条结果；m3u8 直链成功出画面和弹幕。
- 备用源 `https://raw.githubusercontent.com/Lightconer/tvbox-ysc-config/main/output/ouge.json`（12 站，spider 有效）。

配置方式：TVBox 管理页 `http://192.168.31.114:9978` 的 `POST /action`（`do=api` / `do=liveApi`）+ UI 确定按钮。

### 直播源

- `https://xiaolong69.github.io/tv/lives/1.txt`：1208 频道 + EPG 节目单，频道列表正常（CCTV5+ 有德甲/澳网节目单）。
- 央视/卫视多为移动内网线路，联通网络下转圈 → **直播主力仍用 mytv-android 的 258 纯直链源**，TVBox 直播作补充（国际台/CGTN 可播）。
- TVBox 直播接口只吃 http(s) 在线地址，`file:///` 本地路径不识别。

---

## 阶段七：后台占用排查与清理

设备 1GB 内存长期紧张（清理前已用 941MB，仅剩 35MB）。清理动作：

| 动作 | 效果 |
|---|---|
| `am force-stop com.github.mytv.android` | 回收主进程 + :player 子进程 ≈ 102 MB（不卸载应用） |
| 卸载临时输入法 ADBKeyBoard | 回收 ≈ 13 MB |
| force-stop 系统设置/documentsui/packageinstaller/Termux 残留 | 回收 ≈ 40 MB |

**结果**：可用内存（MemAvailable）从 ~35 MB 回升到 **447 MB**，空闲 98 MB。前台 TVBox 播放不打断。

> 关于输入法：搜狗 TV 输入法会把 URL 的 `:/` 转成中文全角 `：／`，导致配置地址填错。临时装 `ADBKeyBoard`（`com.android.adbkeyboard`）用广播 `ADB_INPUT_TEXT` 注入纯 ASCII URL，配完切回搜狗并卸载。清理输入框的可靠方式：`input keyevent KEYCODE_MOVE_END` 后批量发 `67`（DEL）。

---

## 最终软件清单

| 用途 | 应用 | 包名 |
|---|---|---|
| 点播 | TVBox（俊于版） | `com.github.tvbox.osc.jun` |
| 直播 | mytv-android | `com.github.mytv.android` |
| B站 | BBLL | `com.xx.blbl` |
| 影视 | hongguo-tv | `rocks.aston.hongguotv` |
| 桌面 | Veil Lite | `com.veil.launcher` |
| 导航栏 | VirtualSoftKeys | `tw.com.daxia.virtualsoftkeys` |
| ADB 持久化 | adbpersist（自研） | `com.example.adbpersist` |
| 远控（手机端） | scrcpy client | `org.client.scrcpy` |

---

## 踩坑与技术结论

1. **ADB 守护进程会被沙箱回收**：每条命令必须 `start-server` + `connect` + 操作写在同一条命令里。
2. **路径转换坑**：Git Bash 会转换 `/sdcard/...` 参数，需 `export MSYS_NO_PATHCONV=1 MSYS2_ARG_CONV_EXCL='*'`；`adb pull/push` 前尤其注意。
3. **代理坑**：WorkBuddy 注入的代理会黑洞 `github.com` 主站，`api.github.com` 正常。GitHub release 下载走 `api.github.com/releases/assets/<id>` + `Accept: application/octet-stream`，用 `-k` 时须核对 SHA256。
4. **视图缓存坑**：ADB 截图连续看会被视觉工具判为"unchanged"，换文件名 / 裁剪放大可规避。
5. **老固件解码**：能播原生 m3u8 直链，播不了 mgtv/央视频 WebView 线路；与运营商跨网（移动内网）+ OMX 解码共同导致。
6. **32 位设备**：`armeabi-v7a` 只能装 32 位包；TVBox 需选 `python32`/`java32` 构建。

> 本站记录为个人设备折腾笔记，涉及的内网 IP 为私有地址。第三方源/订阅地址来自公开网络，可能随时间失效，请自行甄别。
