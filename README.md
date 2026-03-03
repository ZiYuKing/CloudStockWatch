# CloudStockWatch

CloudStockWatch 是一个面向 Windows 的 WinForms 桌面工具，用于监控狐蒂云（`szhdy.com`）活动页服务器库存，并在补货时触发通知或自动购买流程。

## 1. 项目定位

- 技术栈：`.NET 10` + `WinForms` + `WebView2` + `RestSharp`
- 运行平台：`Windows`
- 核心目标：
  - 自动抓取活动列表
  - 周期性刷新服务器库存状态
  - 库存由“售罄”变“可购买”时发送通知
  - 支持一键购买与自动购买（可选自动跳支付）

## 2. 功能总览

### 2.1 活动与库存监控

- 拉取 `https://www.szhdy.com/newestactivity.html` 解析活动列表
- 每个活动创建独立 Tab，并展示服务器明细表
- 解析字段包括：
  - 状态（可购买/售罄/未知/错误）
  - 现价、原价、折扣
  - 余量（剩余台数）
  - 配置（核心、内存、系统盘、带宽、时长、温馨提示、融合怪脚本测试）

### 2.2 登录与 Cookie 管理

- 支持点击“未登录，点击登录”打开 `LoginWebViewForm`
- 在 WebView2 登录成功后自动抓取 `Cookie` 并写入本地配置
- 支持“退出登录”：
  - 清空配置中的 `HdyCookie`
  - 尝试清理 WebView2 下 `szhdy.com` 域 Cookie

### 2.3 通知能力

补货触发条件：某商品状态从 `SoldOut` 切换为 `InStock`。

支持两种通知通道：
- 企业微信机器人（Webhook Key）
- KOOK 机器人（Bot Token + TargetId）

设置界面提供“通知测试”按钮。

### 2.4 购买与自动购买

- 表格“购买”按钮可直接打开购买页（WebView2）
- 每个商品可单独配置 OS（系统/版本）
- 启用“自动购买”后，满足条件会自动触发购买流程
- 可选“购买后自动跳转支付”
  - 支持支付方式：`AliPay` / `AliPayH5` / `WxPay`

## 3. 界面说明

主界面（`Main`）关键区域：

- 顶部参数：
  - 活动列表刷新间隔（分钟，默认 `5`）
  - 服务器刷新间隔（秒，默认 `30`）
- 操作按钮：`开始` / `停止` / `刷新` / `设置` / `退出登录`
- 中央：按活动分组的 Tab + DataGridView
- 底部：状态栏 + 下次刷新倒计时

表格关键列：
- 自动购买
- 配置（按钮）
- 购买（按钮）
- 状态/价格/库存/配置信息/时间戳等

## 4. 运行流程（实际代码行为）

1. 程序启动后会自动执行一次：
   - 拉取活动列表
   - 初始化每个活动商品行
   - 刷新一次库存状态
2. 用户点击“开始”后进入定时监控：
   - 高频刷新各活动库存（秒级）
   - 低频刷新活动列表（分钟级，发现新活动自动加新 Tab）
3. 若网络错误且命中瞬时错误（502/503/504），会按 5 秒间隔自动重试，最多 5 次。
4. 触发补货事件时：
   - 系统提示音
   - 发送配置中的机器人通知
5. 若启用自动购买且存在购买链接，则打开购买 WebView 流程。

## 5. 环境要求

- Windows 10/11
- .NET SDK `10.0`（开发构建）
- WebView2 Runtime（通常系统已安装，未安装时 WebView2 窗口无法正常初始化）

## 6. 本地开发与运行

### 6.1 还原与构建

```powershell
dotnet restore .\CloudStockWatch.csproj
dotnet build .\CloudStockWatch.csproj -c Debug
```

可执行文件位置：
- `bin\Debug\net10.0-windows\CloudStockWatch.exe`

### 6.2 直接运行

```powershell
dotnet run --project .\CloudStockWatch.csproj
```

## 7. 发布（Release）

仓库已提供文件夹发布配置：`Properties/PublishProfiles/FolderProfile.pubxml`

关键参数：
- `RuntimeIdentifier=win-x64`
- `SelfContained=true`
- `PublishSingleFile=true`
- 输出目录：`bin\Release\net10.0-windows\publish\win-x64\`

发布命令：

```powershell
dotnet publish .\CloudStockWatch.csproj -c Release -r win-x64 --self-contained true /p:PublishSingleFile=true
```

## 8. 配置项说明（Properties.Settings）

| 键名 | 类型 | 默认值 | 说明 |
|---|---|---|---|
| `IntervalTime` | `Int64` | `30` | 服务器库存刷新间隔（秒） |
| `ActivityIntervalTime` | `Int64` | `5` | 活动列表刷新间隔（分钟） |
| `HdyCookie` | `String` | 空 | 狐蒂云登录 Cookie |
| `LoginExpiredNotice` | `Boolean` | `False` | 登录失效时是否发送通知 |
| `WecomBotEnabled` | `Boolean` | `False` | 企业微信通知开关 |
| `WecomBotKey` | `String` | 空 | 企业微信机器人 Key |
| `KookBotEnabled` | `Boolean` | `False` | KOOK 通知开关 |
| `KookBotToken` | `String` | 空 | KOOK Bot Token |
| `KookBotTargetId` | `String` | 空 | KOOK 接收者 ID |
| `BuyToCartEnabled` | `Boolean` | `False` | 购买后自动跳转支付 |
| `AutoPayMethod` | `String` | `AliPay` | 自动支付方式（AliPay/AliPayH5/WxPay） |

## 9. 代码结构

```text
CloudStockWatch/
  Program.cs                    # 程序入口
  Main.cs                       # 主流程：监控、解析、通知触发、购买流程
  Main.Designer.cs              # 主界面布局
  SettingForm.cs                # 设置页（Cookie/通知/自动支付）
  LoginWebViewForm.cs           # 手动登录并抓取 Cookie
  EmbeddedWebViewForm.cs        # 购买页嵌入 WebView + 自动加购/支付脚本
  ProductOsConfigForm.cs        # 商品 OS 预设选择
  NotificationService.cs        # 企业微信与 KOOK 发送实现
  HdyParser.cs                  # HTML 解析辅助（热卖解析）
  Models/                       # 领域模型（WatchRow/ActivityInfo/...）
  Properties/Settings.settings  # 用户配置定义
  Resources/                    # 图标等资源
```

## 10. 自动购买脚本机制说明

`EmbeddedWebViewForm` 在导航完成后执行 JS 脚本，核心动作：

- 识别购买页表单并尝试选择目标 OS 版本
- 勾选协议/条款
- 按配置选择支付方式
- 尝试点击“加入购物车/结算/提交”等按钮
- 循环最多 10 次，每次间隔 700ms

说明：该逻辑依赖目标站点 DOM 结构，页面改版后可能需要同步调整脚本选择器。

## 11. 已知问题与风险

### 11.1 当前构建警告（2026-03-03 本地验证）

执行 `dotnet build -c Debug` 可成功，但有警告：

- `MSB3277`：`WindowsBase` 版本冲突（4.0.0.0 与 5.0.0.0）
- 多处 `CS8600/CS8602` 空引用相关警告（`Main.cs`、`LoginWebViewForm.cs`、`EmbeddedWebViewForm.cs`）

这些警告不阻塞构建，但建议后续清理以降低运行时风险。

### 11.2 解析稳定性

HTML 解析大量使用正则表达式，若 `szhdy.com` 页面结构变化，可能出现：
- 活动解析不到
- 状态误判
- 购买链接提取失败

### 11.3 Cookie 安全

`HdyCookie` 为本地用户配置明文存储，请勿将配置文件提交到仓库或共享给他人。

## 12. 常见使用建议

- 首次使用优先走“手动登录”抓取 Cookie，避免手填格式错误。
- 将“活动列表刷新间隔”设为较低频（例如 3~10 分钟），“服务器刷新间隔”按需求设置（例如 10~30 秒）。
- 开启自动购买前，先单次手动走通“配置 + 购买”流程，确认 OS 与支付方式匹配。
- 开启通知测试并确认消息可达后，再进行长时监控。

## 13. 免责声明

本项目仅用于技术学习与自动化流程实践。请遵守目标平台服务条款与当地法律法规，风险由使用者自行承担。
