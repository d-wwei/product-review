# 注意事项 & Setup 子命令

> 本文件由 SKILL.md 按需加载。仅当用户执行 setup 子命令时加载对应段落。
> 注意事项在所有执行流程开始前加载。

## 注意事项

- 报告语言跟随用户习惯（默认中文）
- 每次体验按日期和版本独立存储，方便追踪产品迭代
- subagent 探索时如果遇到需要登录的功能，先尝试其他入口；如确实需要登录，通过 AskUserQuestion 询问用户是否手动输入；用户选择跳过时报告为"需要登录"并记录到未探索入口清单
- **敏感输入人机接力**：遇到交易密码、登录、银行账户绑定、KYC 等敏感信息输入时，agent 暂停并通过 AskUserQuestion 请求用户协助，绝不自行输入或猜测密码
- **数据准确性"所见即所录"**：报告中的所有数据必须来自 App 中实际看到的内容。禁止根据部分信息推断总数或全貌。遇到下拉菜单和折叠面板必须展开后截图记录完整列表
- **主 agent context 保护**：subagent 的完整报告写入 `exploration-reports/` 目录下的独立文件，返回给主 agent 的只是不超过 20 行的摘要。报告生成 subagent 直接从文件读取探索数据
- 如果 App 崩溃或无响应，记录问题并尝试重新启动
- 探索时间较长的产品，定期向用户汇报进度
- **所有产品数据必须来自 App 内或官方渠道，严禁使用第三方网站数据**
- **每个页面必须滚动到底，点击每个入口，不要只看第一屏**

---

## PlayCover Setup 子命令

用于在 Mac 上通过 PlayCover 快速配置 iOS app 以 iPhone 模式运行，为产品体验做好环境准备。

**触发**: `/product-review playcover-setup [App名称或bundleID]`

### 执行流程

1. **查找 app**: 在 `~/Library/Containers/io.playcover.PlayCover/Applications/` 下查找匹配的 `.app` bundle
2. **读取当前配置**: 读取 `~/Library/Containers/io.playcover.PlayCover/App Settings/<bundleID>.plist`
3. **应用 iPhone 模式配置**:
   ```bash
   plutil -replace resolution -integer 0 "<App Settings plist path>"
   plutil -replace customScaler -float 3 "<App Settings plist path>"
   ```
4. **重启 PlayCover + App**:
   ```bash
   pkill -f "<app process name>" 2>/dev/null || true; sleep 1
   pkill -f "PlayCover" 2>/dev/null || true; sleep 3
   open -g "<app .app path>"
   ```
5. **验证**: 等待 app 加载后截图确认 iPhone UI 渲染正确

### 配置说明

| 参数 | 值 | 作用 |
|---|---|---|
| `resolution` | `0` | 关闭自适应显示，app 使用实际窗口尺寸渲染，避免渲染宽度与窗口不匹配 |
| `customScaler` | `3` | 3x Retina 渲染，接近真实 iPhone 像素密度，解决模糊问题 |

### 原理

PlayTools（PlayCover 注入框架）在 `resolution≥1` 时，向 app 报告的屏幕宽度与 macOS 实际窗口宽度存在固定 ~23% 缩放差，导致 app 的渲染视口超出窗口边界、UI 被截断。`resolution=0` 绕过此问题，让 app 直接使用实际窗口尺寸。

### 已知限制

- 窗口固定为 ~288×541pt（在 4K 显示器上约等于真实 iPhone 物理尺寸）
- 弹窗位置可能略有偏移（坐标映射限制）
- 窗口大小不可通过 windowWidth/Height 调整（resolution=0 会覆盖）
- 如需彻底解决，需升级 PlayTools 到支持 resolution=6（resizable）的版本

---

## Android Setup 子命令

快速设置 Android 模拟器环境，为产品体验做好准备。

**触发**: `/product-review android-setup [App名称或APK路径]`

### 执行流程

1. **检查环境**:
   ```bash
   # Android SDK
   echo $ANDROID_HOME  # 通常 ~/Library/Android/sdk/
   $ANDROID_HOME/emulator/emulator -version
   adb version
   ```
   - 如未安装，指导用户安装 Android Studio 或通过 `sdkmanager` 安装命令行工具
2. **查找/创建 AVD**:
   - 列出现有: `$ANDROID_HOME/emulator/emulator -list-avds`
   - 如无合适 AVD:
     ```bash
     sdkmanager "system-images;android-34;google_apis;arm64-v8a"
     avdmanager create avd -n product_review_phone -k "system-images;android-34;google_apis;arm64-v8a" -d "pixel_7"
     ```
3. **启动模拟器**:
   ```bash
   $ANDROID_HOME/emulator/emulator -avd <avd_name> -no-snapshot-load &
   adb wait-for-device
   ```
4. **安装 App**: 如提供 APK → `adb install <path.apk>`；否则提示用户提供
5. **验证**: `mobile_list_available_devices` 确认模拟器可用，启动 app 截图确认

### 推荐配置

| 项 | 值 | 说明 |
|---|---|---|
| 设备 | Pixel 7 | 标准尺寸 |
| 系统 | Android 14 (API 34) | 主流版本 |
| 架构 | arm64-v8a | Apple Silicon Mac 必须用 ARM |

### 常见问题

- **模拟器卡住**: `emulator -avd <name> -no-snapshot-load -gpu swiftshader_indirect`
- **adb 连不上**: `adb kill-server && adb start-server`
- **Apple Silicon**: 必须用 arm64-v8a 镜像，不能用 x86_64

---

## iOS Simulator Setup 子命令

快速设置 iOS 模拟器环境，为产品体验做好准备。

**触发**: `/product-review ios-sim-setup [App名称]`

### 执行流程

1. **检查环境**:
   ```bash
   xcode-select -p
   xcrun simctl list devices available
   xcrun simctl list runtimes
   ```
2. **查找/创建模拟器**:
   - 列出可用: `xcrun simctl list devices available`
   - 创建新模拟器:
     ```bash
     xcrun simctl list devicetypes | grep iPhone
     xcrun simctl list runtimes | grep iOS
     xcrun simctl create "ProductReview-iPhone15Pro" "com.apple.CoreSimulator.SimDeviceType.iPhone-15-Pro" "com.apple.CoreSimulator.SimRuntime.iOS-18-2"
     ```
3. **启动模拟器**:
   ```bash
   xcrun simctl boot <device_udid>
   open -a Simulator
   ```
4. **安装 App**:
   - .app/.zip → `xcrun simctl install booted <path>`
   - .ipa 不能装模拟器（真机包），需要 .app 的模拟器版本
   - App Store 下载: 提示用户手动在模拟器中操作
5. **验证**: `mobile_list_available_devices` 确认，启动 app 截图

### 推荐配置

| 项 | 值 |
|---|---|
| 设备 | iPhone 15 Pro |
| iOS | 最新可用（`xcrun simctl list runtimes`） |

### 常见问题

- **无运行时**: Xcode → Settings → Platforms 下载
- **.ipa 装不了**: 模拟器需要 .app（模拟器架构），.ipa 是真机包
- **App 闪退**: 部分 app 不支持模拟器架构

---

## iOS Device Setup 子命令

连接 iOS 真机进行产品体验测试。

**触发**: `/product-review ios-device-setup [设备名]`

### 执行流程

1. **检查连接**:
   ```bash
   xcrun devicectl list devices
   # 或
   system_profiler SPUSBDataType | grep -A5 iPhone
   ```
   - 未出现 → 提示: 解锁设备、点击"信任此电脑"、换 USB 线/端口
2. **开发者模式**（iOS 16+）:
   - 设置 → 隐私与安全性 → 开发者模式 → 开启 → 重启
   - 如选项不可见: 先用 Xcode 连接设备一次
3. **配对**:
   ```bash
   xcrun devicectl manage pair --device <device_udid>
   ```
4. **安装 App**（如需）:
   ```bash
   xcrun devicectl device install app --device <device_udid> <path.ipa>
   ```
5. **验证**:
   - `mobile_list_available_devices` 确认设备出现
   - `mobile_take_screenshot` 验证 mobile-mcp 可控制
6. **输出设备信息摘要**

### Wi-Fi 调试（可选）

```bash
# USB 连接后启用
xcrun devicectl manage enableNetworkDebugging --device <device_udid>
```

### 常见问题

- **不被识别**: 换端口/线，确保设备已解锁
- **"信任"不弹出**: 还原位置与隐私设置（设置 → 通用 → 传输或还原 → 还原）
- **mobile-mcp 看不到**: 确认已解锁、已信任、开发者模式已开
- **安装失败**: 检查签名是否有效（Ad Hoc / TestFlight / 企业证书）
