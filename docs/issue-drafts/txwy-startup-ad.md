# [Bug] 繁中服 Txwy 启动广告遮挡 START，MuMu 6.3.1.0 下「开始唤醒」无限重试；已验证可由 Android 输入层关闭

<details><summary>Checkboxes</summary>

### 请确认自己完成了下列必选项之后再进行勾选，若未完成必选项或勾选了"我未仔细阅读"选项将视为自愿接受被直接关闭 Issue

- [x] 我理解 Issue 是用于反馈和解决问题的，而非吐槽评论区，将尽可能提供更多信息帮助问题解决
- [ ] 我未仔细阅读这些内容，只是一键已读所有内容，并相信这不会影响问题的处理
- [x] 我填写了简短且清晰明确的标题，以便开发者在翻阅 Issue 列表时能快速确定大致问题
- [x] 我使用的是当前更新版本的最新版，并已查看更新内容及尚未发布的 Pull Requests，未发现该问题已修复
- [x] 我已检查常见问题、公告、活跃议题及已关闭议题。现象与 #17466 相似，但本案例为 MuMu + Txwy，并提供完整日志及 Android 输入层验证，因此不是单纯重复雷电模拟器案例

</details>

## 问题描述

在繁中服（`client_type: txwy`）启动页，游戏会弹出一张跨游戏推广广告。广告位于明日方舟启动画面上层，右上角有 `X`，下方仍能看见游戏原本的黄色 `START` 按钮。

执行 MAA 的「开始唤醒」后：

1. MAA 能持续识别底层的 `GameStart.png`；
2. MaaTouch 持续点击底层 START；
3. 由于广告仍在上层，点击没有进入游戏；
4. `GameStart` 的执行次数持续累积，最终日志中达到 `exec_times: 2065`，而 `max_times` 为 `2147483647`；
5. 任务因此长时间停留在同一个画面，并不会主动识别或关闭广告右上角的 `X`。

这不是截图通道或触控通道完全失效：MuMuExtras 截图正常，MaaTouch 也正常。问题是当前 `StartUpThemes` 中没有该繁中服启动广告的识别与关闭规则，而且 `GameStart` 排在其前方并会持续命中底层 START。

### 与 #17466 的差异

#17466 是雷电模拟器内嵌广告案例，且原报告日志上传失败。本案例具有以下不同点：

- 模拟器：MuMu（MAA 配置名 `MuMuEmulator12`），模拟器版本 `6.3.1.0`
- 客户端：繁中服 `txwy`
- 截图：MuMuExtras，可完整截到广告；最快截图约 `35 ms`
- 触控：MuMuExtras / MaaTouch 已启用
- 提供完整 `asst.log`、`gui.log`、`gui.new.json`
- 已通过 `adb shell input tap` 实验，证明广告可由 Android 内部输入事件关闭，并非只能由 Windows 宿主层操作的广告

## Version

```text
UI Version: v6.15.1
Core Version: v6.15.1
Resource Version: 永不落幕
Build Time: 2026/7/30 00:11:39
Resource Time: 2026/7/27 01:10:51
```

## 日志和配置文件

上传附件：

```text
report_07-31_startup-ad_minimal.zip
```

该压缩包约 6 MB，包含：

```text
debug/asst.log
debug/gui.log
config/gui.new.json
evidence/maa_before.png
evidence/maa_after.png
evidence/TxwyStartAdClose.png
```

关键日志：

```text
name: MuMuEmulator12
extras: {"path":"C:\\Program Files\\Netease\\MuMuPlayer","touch":true,"client_type":"txwy"}
address: 127.0.0.1:5555
TouchMode: MumuExtras
```

MAA 已识别底层 START：

```text
match_templ | GameStart.png [opencv] score: 0.997266
rect: [ 611, 652, 58, 58 ]
roi: [ 550, 600, 200, 120 ]
```

重复执行：

```text
"exec_times":2065
"max_times":2147483647
"task":"GameStart"
"pre_task":"StartUp@GameStart"
```

MaaTouch 实际仍在点击 START 区域：

```text
maatouch click: (815, 820)
maatouch click: (826, 866)
maatouch click: (783, 826)
```

同一画面 OCR 可读取广告文字：

```text
上線7天
立即
釋放專屬必殺技
```

这说明广告已进入 MAA 的截图画面，而不是只存在于 Windows 窗口外层。

## 配置信息

```text
操作系统：Windows 11
模拟器：MuMu（MuMu 12 系列）
模拟器版本：6.3.1.0
MAA 连接配置：MuMuEmulator12
ADB：C:\Program Files\Netease\MuMuPlayer\nx_device\15.0\shell\adb.exe
ADB 地址：127.0.0.1:5555
客户端：txwy（繁中服）
ADB 截图分辨率：1600 × 900，16:9
画面显示帧率：60
MuMu 截图增强：开启
MuMu 触控增强：开启
触控模式：MumuExtras / MaaTouch
GPU 加速推理：开启
GPU：NVIDIA GeForce GTX 1660 Ti
```

启动时曾出现一次 ADB 短暂断线，但之后已经重连成功，MuMuExtras 截图和 MaaTouch 均正常工作，因此该连接事件不是持续卡住的主因。

## 截图或录屏

### 1. `maa_before.png`

由 Android `screencap` 取得的原图。可以完整看到广告、右上角 `X`，以及广告下方仍存在的游戏 START。

### 2. `maa_after.png`

使用 Android 输入事件点击右上角 `X` 后，再由 Android `screencap` 取得。广告已经消失，游戏原始启动页和 START 正常显示。

## Android 输入层验证

完整复现实验使用 PowerShell：

```powershell
$adb = 'C:\Program Files\Netease\MuMuPlayer\nx_device\15.0\shell\adb.exe'
$device = '127.0.0.1:5555'
$desktop = [Environment]::GetFolderPath('Desktop')

# 连接并确认设备
& $adb connect $device
& $adb -s $device shell wm size

# 取得点击前的 Android 原始截图
& $adb -s $device shell screencap -p /sdcard/maa_before.png
& $adb -s $device pull /sdcard/maa_before.png "$desktop\maa_before.png"

# 点击广告右上角 X；当前设备分辨率为 1600 × 900
& $adb -s $device shell input tap 1380 132

Start-Sleep -Seconds 2

# 取得点击后的 Android 原始截图
& $adb -s $device shell screencap -p /sdcard/maa_after.png
& $adb -s $device pull /sdcard/maa_after.png "$desktop\maa_after.png"
```

实验结果：

- `maa_before.png` 的 Android 原始截图中存在完整广告；
- `adb shell input tap 1380 132` 能成功关闭广告；
- `maa_after.png` 中广告消失；
- 游戏启动页仍正常存在。

因此可以确定：

> 该广告位于 Android 渲染层及 Android 输入层内，可被 ADB/MaaTouch 操作。当前 MAA 无法处理的原因是缺少识别及点击规则，而不是该广告在宿主层、MAA 无法点到。

## 预期行为

「开始唤醒」在识别并点击 `GameStart` 之前，应优先检测该繁中服启动广告；识别成功时点击右上角 `X`，等待广告消失，再继续执行 `GameStart`。

同时建议为 `GameStart` 增加合理的异常上限或阻挡检测，避免被未知模态弹窗遮挡时以接近无限的次数持续点击。

## 建议实现

以下为可直接测试的完整资源层实现方案。建议最终放在 `txwy` 客户端专用资源中，以避免影响其他客户端；若暂时放入通用 `resource/tasks/tasks.json`，应保留严格 ROI 和模板阈值。

### 1. 修改 `StartUpThemes`

将 `TxwyStartAdClose` 放在 `GameStart` 之前：

```json
"StartUpThemes": {
    "algorithm": "JustReturn",
    "next": [
        "BiliBiliNoMorePromptsNextTime",
        "LeidianGoogleFrameworkConfirm",
        "TxwyStartAdClose",
        "GameStart",
        "StartToWakeUp",
        "StartToWakeUpOCR",
        "StartLoginBServer",
        "StartUpConnectingFlag",
        "MainThemes#next",
        "CloseAnnos#next",
        "OfflineConfirm",
        "GameStartCheckResourceOCR",
        "GameStartUpdateOCR"
    ]
}
```

### 2. 新增任务 `TxwyStartAdClose`

```json
"TxwyStartAdClose": {
    "doc": "关闭繁中服启动页的跨游戏推广广告。模板仅匹配右上角关闭按钮，避免按广告素材内容识别。",
    "algorithm": "MatchTemplate",
    "template": "TxwyStartAdClose.png",
    "action": "ClickSelf",
    "roi": [1060, 60, 100, 80],
    "templThreshold": 0.85,
    "postDelay": 1000,
    "maxTimes": 5,
    "next": ["StartUpThemes#next"]
}
```

说明：

- MAA 的 ROI 以 `1280 × 720` 为基准自动缩放；
- 实测设备为 `1600 × 900`；
- 点击坐标 `(1380, 132)` 对应约 `(1104, 106)` 的 1280 × 720 坐标；
- ROI `[1060, 60, 100, 80]` 只覆盖右上方关闭按钮附近，降低误触风险；
- 必须排在 `GameStart` 前面，否则 `GameStart.png` 会先匹配到底层 START 并继续无效点击。

### 3. 生成模板图片

从附件 `maa_before.png` 生成 `TxwyStartAdClose.png`：

```python
from pathlib import Path
from PIL import Image

source = Path("maa_before.png")
target = Path("TxwyStartAdClose.png")

image = Image.open(source).convert("RGB")

# MAA 任务坐标以 1280 × 720 为基准
image = image.resize((1280, 720), Image.Resampling.LANCZOS)

# 截取右上角 X；输出为 32 × 32 模板
# 原图 1600 × 900 中约位于 x=1360~1400、y=98~136
image.crop((1088, 78, 1120, 110)).save(target)

print(f"saved: {target.resolve()}")
```

建议模板路径：

```text
resource/template/TxwyStartAdClose.png
```

若采用繁中服专用覆盖资源，则放入相应 `txwy` template 目录，并在该客户端的 tasks 覆盖文件中加入上述任务。

### 4. 可应用的 unified diff（资源任务部分）

```diff
diff --git a/resource/tasks/tasks.json b/resource/tasks/tasks.json
--- a/resource/tasks/tasks.json
+++ b/resource/tasks/tasks.json
@@
     "StartUpThemes": {
         "algorithm": "JustReturn",
         "next": [
             "BiliBiliNoMorePromptsNextTime",
             "LeidianGoogleFrameworkConfirm",
+            "TxwyStartAdClose",
             "GameStart",
             "StartToWakeUp",
             "StartToWakeUpOCR",
@@
         ]
     },
+    "TxwyStartAdClose": {
+        "doc": "关闭繁中服启动页的跨游戏推广广告。模板仅匹配右上角关闭按钮，避免按广告素材内容识别。",
+        "algorithm": "MatchTemplate",
+        "template": "TxwyStartAdClose.png",
+        "action": "ClickSelf",
+        "roi": [1060, 60, 100, 80],
+        "templThreshold": 0.85,
+        "postDelay": 1000,
+        "maxTimes": 5,
+        "next": ["StartUpThemes#next"]
+    },
     "LeidianGoogleFrameworkConfirm": {
```

## 建议测试矩阵

1. **有广告**：先关闭广告，再进入 `GameStart`。
2. **无广告**：不得误点启动页其他右上角控件。
3. **同一壳层、不同广告素材**：应继续匹配关闭按钮，而不依赖广告正文。
4. **分辨率**：至少测试 `1280×720`、`1600×900`、`1920×1080`。
5. **触控模式**：MuMuExtras/MaaTouch 与普通 ADB input 各测试一次。
6. **客户端隔离**：优先只在 `txwy` 资源中启用，避免影响官服、B服及 YoStar 客户端。
7. **异常保护**：连续点击 `GameStart` 多次但画面不变时，应保存截图并退出或转入阻挡弹窗检测，而不是无限累积 `exec_times`。

## 还有别的吗？

该问题不是模拟器无法被 MAA 操作，也不是广告只能通过 Windows 鼠标关闭。两张 ADB 原始截图及输入实验已经建立完整证据链：

```text
MAA/MuMuExtras 能看到广告
        ↓
Android screencap 能看到广告
        ↓
Android input tap 能关闭广告
        ↓
关闭后游戏启动页正常
        ↓
缺陷定位为 StartUp 缺少该弹窗识别规则
```
