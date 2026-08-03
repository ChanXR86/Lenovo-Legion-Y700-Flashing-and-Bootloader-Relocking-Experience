# Lenovo Legion Y700 2023 / TB320FC 国行 17.0.478 官方回锁技术报告

> 本文是一次实机恢复和回锁复盘，不是通用刷机教程。  
> 设备相关的 `sn.img`、邮箱、完整序列号、GSN 等均不应公开提交到 GitHub。本文已用占位符脱敏。

## 结论

这台 Lenovo Legion Y700 2023 / TB320FC 最终恢复到官方国行 CN 系统：

```text
TB320FC_CN_OPEN_USER_Q00031.0_V_ZUI_17.0.478_ST_251224
```

并完成普通 Bootloader 与 Critical Bootloader 回锁。最终验收状态：

```text
Android boot completed: yes
Active slot: _b
Verified Boot state: green
flash.locked: 1
vbmeta.device_state: locked
veritymode: enforcing
Fastboot secure: yes
Device unlocked: false
Device critical unlocked: false
Slot B successful: yes
Slot B unbootable: no
```

本案例的关键点不是“强刷后直接回锁”，而是先确认 OTA 包来源、签名、目标版本、A/B 槽状态、Virtual A/B 合并状态、AVB/verity 状态，再执行回锁。

## 设备和范围

实测设备：

```text
Model: TB320FC
Product name: TB320FC_PRC
Product device: TB320FC
Region/build line: CN_OPEN_USER
Android: 15
Final display build: ZUXOS_1.1.478_ST_251224
Final incremental build: TB320FC_CN_OPEN_USER_Q00031.0_V_ZUI_17.0.478_ST_251224
Serial/GSN: <redacted>
```

本文不适用于：

- ROW / NEC / JPN 区域包混刷后直接回锁。
- Root、Magisk、禁用 AVB、修改 vbmeta 后直接回锁。
- 修改 OTA metadata、重签名 OTA、绕过 Recovery 校验。
- 不确认当前槽位 successful/unbootable 状态就回锁。
- 不确认 Virtual A/B snapshot merge 状态就回锁。

## 时间线概览

1. 设备先恢复到官方 CN 17.0.350，Bootloader 和 Critical Bootloader 保持解锁。
2. 下载并只读鉴定 17.0.478 候选 OTA 包。
3. 尝试原厂 Recovery 的 `Apply update from ADB`，被 Recovery 正常拒绝，未绕过。
4. 通过联想系统更新 App 的可见本地更新入口安装同一 17.0.478 OTA。
5. 设备自动重启后进入 17.0.478，当前槽从 `_a` 切到 `_b`。
6. 在 17.0.478 上做双重启动、Fastboot 只读槽位、AVB、Root/Magisk、Virtual A/B 检查。
7. 先回锁 Critical Bootloader，再回锁普通 Bootloader。
8. 回锁后再次验证 Android 和 Fastboot 状态，确认 AVB green、两把锁均 locked。

## 17.0.478 OTA 候选包鉴定

候选包来源：

```text
https://otanew-cdn.lenovo.com/firmware/2025122515380343-5747.zip
```

下载后记录：

```text
Size:   330170688 bytes
MD5:    38388ca20caa1f9d1a5be61e8b1479cf
SHA256: 4923c3c7bdc8be62aa451217de60f6914196081083bafd93984a040d9700a18d
ZIP CRC: passed
```

`META-INF/com/android/metadata` 关键字段：

```text
ota-type=AB
pre-device=TB320FC
pre-build=Lenovo/TB320FC/TB320FC:15/AQ3A.240812.002/TB320FC_CN_OPEN_USER_Q00031.0_V_ZUI_17.0.350_ST_250418:user/release-keys
pre-build-incremental=TB320FC_CN_OPEN_USER_Q00031.0_V_ZUI_17.0.350_ST_250418
post-build=Lenovo/TB320FC/TB320FC:15/AQ3A.240812.002/TB320FC_CN_OPEN_USER_Q00031.0_V_ZUI_17.0.478_ST_251224:user/release-keys
post-build-incremental=TB320FC_CN_OPEN_USER_Q00031.0_V_ZUI_17.0.478_ST_251224
post-sdk-level=35
post-security-patch-level=2025-10-05
post-timestamp=1766593326
```

判断：

- 这是 AB OTA。
- `pre-device` 是 `TB320FC`。
- 来源版本精确要求 `17.0.350_ST_250418`。
- 目标版本是 `17.0.478_ST_251224`。
- `post-build` 属于 `TB320FC_CN_OPEN_USER`，不是 ROW / NEC / JPN。
- 未发现 `POWERWASH=1`。
- 未发现 `ota-wipe=yes`。
- 未发现绑定公开可见序列号的 metadata 字段。

Payload 校验：

```text
payload FILE_SIZE match: true
payload FILE_HASH match: true
payload METADATA_HASH match: true
```

证书比对：

```text
Known official 350 OTA MD5: 6657fd37815887fce9fc8e708ed47bfc
350/478 otacert SHA256 equal: true
otacert SHA256: 113cd885fbef873f39427a064159444362b8551febac6011eda82a007be705d1
350/478 certificate DER SHA256 equal: true
350/478 public key SHA256 equal: true
CERT.RSA: absent in both packages
```

证书信息：

```text
Subject: C=CN, ST=BJ, L=Beijing View, O=LENOVO, OU=Mobile, CN=LENOVO, emailAddress=lenovo@lenovo.com
Issuer:  C=CN, ST=BJ, L=Beijing View, O=LENOVO, OU=Mobile, CN=LENOVO, emailAddress=lenovo@lenovo.com
Serial:  B291CE264E88FB08
SHA256 fingerprint:
6E:FF:C3:C0:25:98:B0:C4:9A:C1:29:CC:CA:66:E1:B5:A1:5F:36:EA:CB:32:06:74:EC:BD:DB:AB:A6:AA:09:C4
```

结论：478 包具备进一步评估资格，但这一步本身不等于允许安装或允许回锁。

## Recovery ADB Sideload 被拒绝

在官方 Recovery 中尝试 `Apply update from ADB` 后，Recovery 返回：

```text
Package is for source build
TB320FC_CN_OPEN_USER_Q00031.0_V_ZUI_17.0.350_ST_250418

but expected
TB320FC_S000087_250418_DRV

Install from ADB completed with status 1.
Installation aborted.
```

这个错误说明 Recovery 使用的源版本标识与 OTA metadata 的 `pre-build` 不一致。这里没有做以下操作：

- 修改 OTA metadata。
- 修改 `pre-build` / `pre-build-incremental`。
- 重打包或重签名 OTA。
- 调用 `update_engine_client` 强行安装。
- 再次 sideload 硬试。
- wipe / factory reset。
- set-active / flash / erase / format。

这个拒绝是安全机制，不应该绕过。

## 联想本地更新路径

Recovery 拒绝后，设备正常回到 17.0.350：

```text
Build: TB320FC_CN_OPEN_USER_Q00031.0_V_ZUI_17.0.350_ST_250418
Slot: _a
sys.boot_completed: 1
verifiedbootstate: orange
flash.locked: 0
vbmeta.device_state: unlocked
```

系统更新 App 信息：

```text
Package: com.lenovo.ota
Version: V9.2.1.250407
APK: /system/priv-app/LenovoOTA/LenovoOTA.apk
Local update activity observed in package dump:
com.lenovo.row.ota.core.d.ui.UpdateFromExternalStorageActivity
Permission:
lenovo.permission.ota.exported
```

系统更新页面初始菜单没有本地安装入口，只显示：

```text
更新设置
更新日志
我要反馈
```

后续通过正常可见 UI 操作，菜单出现：

```text
本地安装更新包
```

安装路径：

```text
PC OTA path:     <local-workdir>/2025122515380343-5747.zip
Device OTA path: /storage/emulated/0/ota.zip
```

设备侧重新计算 size / MD5 / SHA256 均与 PC 文件一致。点击可见入口后，联想 OTA 客户端直接进入“正在安装”，没有观察到文件选择器，也没有观察到 wipe / downgrade / ROW / NEC / JPN 等危险提示。

安装日志中的关键证据：

```text
InstallPlanAction ErrorCode::kSuccess
overall progress 10%
overall progress 40%
overall progress 50%
Lenovo/UDSEngine progress reached 36%
```

计划中原本希望停在“立即重启”确认前，但本机联想 OTA 客户端未显示明确重启确认，而是自动重启。PC 没有发送重启命令。

自动重启后状态：

```text
Display ID:  TB320FC_CN_OPEN_USER_Q00031.0_V_ZUXOS_1.1.478_ST_251224
Incremental: TB320FC_CN_OPEN_USER_Q00031.0_V_ZUI_17.0.478_ST_251224
Slot: _b
sys.boot_completed: 1
verifiedbootstate: orange
flash.locked: 0
vbmeta.device_state: unlocked
```

## 回锁前只读验收

在 17.0.478 上先做 Android 侧检查：

```text
ro.product.model=TB320FC
ro.product.name=TB320FC_PRC
ro.product.device=TB320FC
ro.build.version.incremental=TB320FC_CN_OPEN_USER_Q00031.0_V_ZUI_17.0.478_ST_251224
ro.build.fingerprint=Lenovo/TB320FC_PRC/TB320FC:15/AQ3A.240812.002/ZUXOS_1.1.478_251224_PRC:user/release-keys
ro.build.version.security_patch=2025-10-05
ro.boot.slot_suffix=_b
ro.boot.verifiedbootstate=orange
ro.boot.flash.locked=0
ro.boot.vbmeta.device_state=unlocked
ro.boot.veritymode=enforcing
ro.boot.avb_version=1.2
sys.boot_completed=1
```

Root / Magisk 检查：

```text
which su: no result
Magisk package count: 0
Magisk process count: 0
```

Virtual A/B / snapshot 检查：

```text
ro.build.ab_update=true
ro.virtual_ab.enabled=true
uptime at check: about 56 minutes
dumpsys update_engine: service not found
bootctl: inaccessible or not found from adb shell
actual MERGING/SNAPSHOTTED/merge-in-progress lines: 0
snapshot/update_verifier error lines: 0
snapuserd process: not present
dm-user snapshot mounts: not present
```

因为 ADB shell 无法直接读取 merge state，所以采用保守判断：设备已稳定运行超过 20 分钟，未发现 snapshot merge 进行中、snapuserd、update_verifier 错误或 dm-user snapshot 挂载。

之后进入 Fastboot，仅做只读查询：

```text
fastboot getvar product
fastboot getvar current-slot
fastboot getvar slot-count
fastboot getvar slot-successful:a
fastboot getvar slot-unbootable:a
fastboot getvar slot-retry-count:a
fastboot getvar slot-successful:b
fastboot getvar slot-unbootable:b
fastboot getvar slot-retry-count:b
fastboot getvar secure
fastboot getvar unlocked
fastboot oem device-info
fastboot getvar all
```

Fastboot 结果：

```text
product: taro
current-slot: b
slot-count: 2

slot-successful:a: yes
slot-unbootable:a: yes
slot-retry-count:a: 6

slot-successful:b: yes
slot-unbootable:b: no
slot-retry-count:b: 6

secure: yes
unlocked: yes
Device unlocked: true
Device critical unlocked: true
Verity mode: true
build.display.id: TB320FC_CN_OPEN_USER_Q00031.0_V_ZUI_17.0.478_ST_251224
```

注意：A 槽状态被原样记录，没有执行 `set_active` 或修复槽位。回锁依据是当前活动 B 槽满足：

```text
current-slot=b
slot-successful:b=yes
slot-unbootable:b=no
Verity mode=true
secure=yes
Fastboot build=17.0.478
```

Fastboot 查询后重启回系统，再观察约 5 分钟，未出现 Recovery 循环、Fastboot 循环、AVB 红屏、系统崩溃或自动回退到 A 槽。

## 最终回锁

执行回锁前，明确接受：

```text
可能清空数据
可能无法再次解锁
回锁失败可能需要 9008 / QFIL 救援
```

回锁顺序：

```powershell
fastboot flashing lock_critical
fastboot flashing lock
```

第一步 Critical 回锁结果：

```text
command: fastboot flashing lock_critical
result: OKAY
post Android boot: passed
post Fastboot:
  Device unlocked=true
  Device critical unlocked=false
  Verity mode=true
  current-slot=b
```

第二步普通 Bootloader 回锁结果：

```text
command: fastboot flashing lock
result: OKAY
post Android boot: passed
post Fastboot: passed
second Android boot: passed
```

最终 Android 状态：

```text
ro.build.version.incremental=TB320FC_CN_OPEN_USER_Q00031.0_V_ZUI_17.0.478_ST_251224
ro.boot.slot_suffix=_b
sys.boot_completed=1
ro.boot.verifiedbootstate=green
ro.boot.flash.locked=1
ro.boot.vbmeta.device_state=locked
ro.boot.veritymode=enforcing
ro.serialno=<redacted>
ro.odm.lenovo.gsn=<redacted>
```

最终 Fastboot 状态：

```text
currentSlot=b
slotSuccessfulB=yes
slotUnbootableB=no
secure=yes
deviceUnlocked=false
deviceCriticalUnlocked=false
verityMode=true
```

结论：这台 TB320FC 已在官方 17.0.478 上完成普通与 Critical 回锁，双重锁定启动验收通过。

## 后处理

回锁后做了两类后处理。

应用恢复：

```text
Manifest packages: 47
Missing before restore: 25
Restored successfully: 25
Restore failures: 0
Remaining missing: 0
```

说明：只找到了 APK 备份，没有找到 app 私有数据备份。因此能恢复的是安装包，不是登录态、聊天记录、游戏数据等私有数据。

Google Play：

```text
Play Store package: com.android.vending
Version: 52.4.41-24 [0] [PR] 953053140
Launcher activity observed: com.android.vending/.AssetBrowserActivity
```

系统更新关闭：

```text
com.lenovo.ota disabled: true
OTA activity resolve: No activity found
ota_disable_automatic_update=1
ota_network_permission=0
lenovo_ota_process=0
lenovo_ota_new_version_found=0
```

如需恢复系统更新入口，可重新启用：

```powershell
adb shell pm enable com.lenovo.ota
adb shell settings put system ota_network_permission 1
```

## 踩坑和判断标准

### 不要低版本回锁

本机之前尝试过较低版本路线，例如 15.0.549 / 15.0.677。结果出现过 Recovery 循环、AVB 红色错误、无法稳定启动等问题，最后需要救援流程恢复。低版本固件存在，不等于低版本可安全回锁。

已回锁在 17.0.478 后，不建议再为了“低版本回锁”解锁、降级、再回锁。这样会重新引入：

- 再次解锁失败风险。
- 数据清空风险。
- 防回滚 / AVB 校验失败风险。
- 9008 救援风险。

### 不要忽略 Recovery 拒绝

Recovery 报：

```text
Package is for source build ... but expected ...
```

时，正确处理是停止并分析版本标识差异，而不是修改 ZIP 或绕过签名。

### 不要在 orange 状态下凭感觉回锁

解锁状态下 `verifiedbootstate=orange` 是预期；回锁后应变为 `green`。  
但回锁前必须确认：

```text
Root/Magisk: absent
verity: enforcing / true
current slot: target slot
slot-successful: yes
slot-unbootable: no
build: official release-keys
region/product: expected CN TB320FC
```

### Virtual A/B 设备要等稳定

Virtual A/B OTA 后，即使已经进入新系统，也可能存在 snapshot merge。无法直接读取 merge state 时，至少要等待足够长的稳定运行时间，并检查：

```text
snapuserd
dm-user mounts
update_verifier
update_engine
snapshot
merge in progress
MERGING / SNAPSHOTTED
```

## 可复用的只读检查命令

Android 侧：

```powershell
adb devices -l
adb shell getprop ro.product.model
adb shell getprop ro.product.name
adb shell getprop ro.product.device
adb shell getprop ro.build.version.incremental
adb shell getprop ro.build.fingerprint
adb shell getprop ro.boot.slot_suffix
adb shell getprop ro.boot.verifiedbootstate
adb shell getprop ro.boot.flash.locked
adb shell getprop ro.boot.vbmeta.device_state
adb shell getprop ro.boot.veritymode
adb shell getprop sys.boot_completed
adb shell which su
adb shell pm list packages | findstr /i magisk
adb shell ps -A | findstr /i magisk
```

Fastboot 侧：

```powershell
adb reboot bootloader
fastboot devices
fastboot getvar product
fastboot getvar current-slot
fastboot getvar slot-count
fastboot getvar slot-successful:a
fastboot getvar slot-unbootable:a
fastboot getvar slot-retry-count:a
fastboot getvar slot-successful:b
fastboot getvar slot-unbootable:b
fastboot getvar slot-retry-count:b
fastboot getvar secure
fastboot getvar unlocked
fastboot oem device-info
fastboot getvar all
fastboot reboot
```

## 日志索引

本机原始日志目录如下。公开仓库中建议只提交脱敏后的报告和必要摘要，不要提交 `sn.img`、邮件截图、完整 bugreport 或包含序列号的原始日志。

```text
D:\Codex\Y700_ZUI_17.0.478_CANDIDATE_VERIFY
D:\Codex\Y700_ZUI_17.0.478_STOCK_RECOVERY_SIDELOAD\20260802_100026
D:\Codex\Y700_ZUI_17.0.478_LOCAL_UPDATE_PRECHECK\20260802_101742
D:\Codex\Y700_ZUI_17.0.478_LOCAL_UPDATE_INSTALL\20260802_104131
D:\Codex\Y700_ZUI_17.0.478_RELOCK_PREFLIGHT\20260802_121545
D:\Codex\Y700_ZUI_17.0.478_FINAL_RELOCK\20260802_124815
D:\Codex\Y700_APP_RESTORE_AND_OTA_OFF_20260802_134945
```

## 参考资料

- Lenovo 社区：<https://mclub.lenovo.com.cn/thread-7917183-1-1.html>
- XDA Y700 2023 regional ROM discussion: <https://xdaforums.com/t/y700-2023-gen_2-regional-rom-flashing-guide.4685115/>
- Y700 OTA/community update index: <https://t.me/s/Y700_updates>
- Android Platform Tools: <https://developer.android.com/tools/releases/platform-tools>
- Lenovo Software Fix / Rescue and Smart Assistant: <https://support.lenovo.com/us/en/downloads/ds101291>

## 最后说明

本案例能成功回锁的前提是：目标系统是官方 CN 17.0.478、签名链与已验证官方 OTA 一致、当前活动槽 B 通过 Fastboot 只读验收、未发现 Root/Magisk、verity 正常、Virtual A/B 未见合并异常，并且用户明确接受回锁风险。

只要其中任何一项不满足，就不应该执行回锁。
