# beautybox-reverse-study

BeautyBox 5.1.5 逆向分析靶场（本地学习用）。

## 仓库内容

- `BeautyBox_5.1.5.apk` — 原版 APK（靶场样本）
- `BeautyBox_5.1.5_tile.apk` — 成品 APK（原版服务器列表 + 全部修复，明文直通版，适合模拟器）
- `BeautyBox_5.1.5_armfix-signed.apk` — 真机专用 APK（原生加密保留 + 防闪退补丁，包名 `.tile`，详见结论 5）
- `BeautyBox_5.1.5_tile_mocktest.apk` — 模拟器 mock 测试专用包（4 个服务器硬编码为 `http://10.0.2.2:20999`，配合本地 mock 验证联网链路）
- `角色/逆向工程师.md` — 分析方法论
- Releases — 分析产物 APK

## 分析结论（smali 证据）

### 1. 启动闪退根因

入口 `OuiCrGxF.onCreate` → 基类 `Yclh4J3zF.onCreate` 依次调用 4 个环境检测器，任一命中即走 `Yclh4J3zF.m0()`（finish 全部 Activity + 清栈）→ `System.exit(0)` 静默退出：

| 检测器 | 位置 | 检测内容 |
|---|---|---|
| `p0/d.d(Context,c)` | smali/p0/d | 模拟器特征（baseband `1.0.0.0`、`ro.build.flavor=sdk_gphone`、goldfish/nox） |
| `p0/a.c(Context)` | smali/p0/a | 模拟器包名（bluestacks/mumu/nox 等） |
| `p0/e.d()` | smali/p0/e | `ro.secure`/su 文件（root 检测） |
| `p0/e.f()` | smali/p0/e | Xposed 调用栈特征 |

`m0()` 共 8 个调用点：3 处检测器回调（`OuiCrGxF:997/1115/1130`）、1 处 Xposed 检测（`Yclh4J3zF:3103`）、3 处服务器响应对话框（`OuiCrGxF$p$a` / `Yclh4J3zF$i` / `t$a`）、1 处登录流程（`User/CfTs9fWO:614`）。真机上 3 秒闪退为服务器校验签名后下发指令触发同一条退出链。

### 2. 退出链防御

- `ExitHook.smali`（新增类）：`block(I)` 静默 no-op。
- 6 处 `System.exit` 调用点（对话框退出 `OuiCrGxF$c/$d$a/$h$a/$i$a`、`Yclh4J3zF:2833`、缓存清理 `ZIgnJ$c`）全部替换为 `ExitHook.block`。
- 4 个检测器方法体 stub 为恒 `false`。
- 实测：Android 12 模拟器（API 31，无 KVM，TCG 软件渲染）进程存活 60s+，零退出尝试，Activity 正常 Displayed。

### 3. 联网闪退真因（tombstone 定位）

断网正常、联网必闪退的根因**不在 Java 层**，而是 native 崩溃：

- 应用所有请求/响应都经 `libali.so` 加解密：`Server.e.c(String,int)` → `c.abc.ea5([BI)[B`（请求加密），`Server.e.a([BI)` → `c.abc.da5([BI)[B`（响应解密）。
- 在 x86_64 模拟器的 ARM 转译层（libndk_translation）下执行 `ea5` 时，JNI `CallVoidMethodA` 收到 null 对象 → SIGABRT（tombstone：`Abort message: 'JNI DETECTED ERROR IN APPLICATION: calling GetObjectClass on null'` 一类）。
- 断网时 `h.a()`（ConnectivityManager 判定）为 false，请求链根本不启动，`ea5` 不执行 → 不闪退；联网后第一个请求加密即 abort，此时还没有任何数据包发出。
- **修复**：`Server/e.smali` 的 `c()`（请求方向）改为 `String.getBytes("UTF-8")` 明文直通、`a()`（响应方向）改为 `new String(bytes,"UTF-8")` 明文直通，绕过 `libali.so`。传输层本身是标准 OkHttp POST，与 native 无关，明文化不影响模拟器上的通信流程。

### 4. 启动页加载完后闪退真因

联网闪退修复后实测出现第二段闪退：启动页动画正常走完，随后崩溃。定位：

- splash 探测（`Server/i.c()`）无论成败都会回调 `OuiCrGxF.T0()` 发主配置请求（`Server/v.b + v.c`，即 `POST /v13/1`）。
- 响应回调 `OuiCrGxF$q.run()` **没有任何 try/catch**：直接 `f.d()`（Gson.fromJson）解析响应体，再裸取 `code`、`data.sys` 等字段。
- 原版协议里响应体是加密二进制（本仓库已把解密改成明文直通），服务器返回的加密内容被当 UTF-8 解出乱码 → Gson 抛 `JsonSyntaxException`（或解析出 `data:null` 后 NPE）→ **UI 线程未捕获异常 → 闪退**。
- 同一请求链的持久化/回读（`MainApplication.w()/k()`）走 `Server/e.d()/e.b()` → `libali.so` 的 `ea2/da2`，与 `ea5` 同属转译层下的 abort 风险点。
- **修复**：① `OuiCrGxF$q.run()` 整体 try/catch(Throwable)，异常一律降级到 `a1()`（「Server not available」对话框）并打 `BBFIX` 日志；② `Server/e.b()/e.d()` 改恒等直通（本地配置明文存取自洽，`ea2/da2` 不再被调用）；③ echo 回调 `OuiCrGxF$m.c()` 同样加 catch 兜底。
- 实测：配置响应不可解析时应用存活、弹降级对话框、logcat 0 FATAL（见 TESTING.md 第四节）。

### 5. 原版包名触发的隐藏退出（真机版必须改包名）

为在真机上保留原生加密协议，曾构建「原版树 + 仅 14 个防闪退补丁、包名保持 `com.secret.prettyhezi`」的 armfix 包，结果**离线**状态启动约 2.7 秒即静默退出：

- logcat 证据：`cret.prettyhez: System.exit called, status: 0` + `Process ... has died: fg TOP`，发生在 splash 阶段、**无任何网络交互**（离线）→ 排除服务器下发指令。
- Java 层 grep 确认补丁树 **0 处 `System.exit`** 残留（6 处原调用点已换 `ExitHook.block`）；签名读取点排查（`p0/a`、`b0/c`、`g4/a`）均非退出源。退出点藏在反射调用/native `libali.so`/动态加载 dex 中，未定位。
- **对照实验**：同一棵树仅把包名改为 `com.secret.prettyhezi.tile`（`apktool.yml` 的 `renameManifestPackage` + FileProvider authorities 同步改名），离线启动 60s+ 存活、0 exit、0 FATAL；联网后仅剩 `ea5` 在 x86_64 转译层的已知 SIGABRT（真机 ARM 不存在）。
- **结论**：原版包名是某条隐藏完整性校验的触发条件（大概率校验原版签名，包名不同则校验分支直接跳过——此前 `.tile` 系列包从未触发）。真机版必须使用 `.tile` 包名。

## 真机版（armfix）说明

`BeautyBox_5.1.5_armfix-signed.apk` 与 `tile.apk` 的区别：

| | tile.apk（模拟器/明文直通版） | armfix-signed.apk（真机/原生加密版） |
|---|---|---|
| 请求加密 `ea5`/响应解密 `da5`、配置存取 `ea2/da2` | UTF-8 明文直通 | **保留 `libali.so` native 原版调用**（与真实服务器协议兼容） |
| 服务器列表/选服 | 原版 4 节点 + 智能选服补丁 | 原版 4 节点、原版逻辑（未动） |
| 防闪退补丁（14 文件） | 有 | 有（`ExitHook` 新增 + `MainApplication$d`、`OuiCrGxF$c/$d$a/$h$a/$i$a/$m/$q`、`User/a`、`Yclh4J3zF`、`ZIgnJ$c`、`p0/a`、`p0/d`、`p0/e`） |
| 包名 | `com.secret.prettyhezi.tile` | 同左（**必须**，绕过隐藏 exit，见结论 5） |
| 适用环境 | 真机 + x86 模拟器 | **仅真机（ARM）**；x86_64 模拟器上 `ea5` 转译层必 SIGABRT |
| 会员 | 放行（`isExpired()` 恒 false） | 同左 |

真机安装：直接安装即可（包名为 `.tile`，与原版可共存；如需与原版隔离数据可先卸载原版）。签名为本仓库测试签名，与原版签名不同属预期。若启动后仍提示「服务连接不上」，说明服务器端对请求载荷中的设备/签名字段做了校验，属协议层问题，非本包闪退缺陷。

## 修改清单（Release 产物）

| 修改 | 位置 | 内容 |
|---|---|---|
| 闪退修复 | `p0/d`、`p0/a`、`p0/e`、`ExitHook`（新增） | 检测器 stub + exit 拦截 |
| 闪退修复（联网） | `Yclh4J3zF.m0()` 清空 | m0 原逻辑=逆序 finish 全部 Activity + 清栈 + exit。服务器校验签名失败下发指令走 8 个调用点触发 m0 → 界面全关=闪退观感（断网连不上服务器故不闪退）。清空后 8 个调用点全部 no-op |
| 联网 native 闪退修复 | `Server/e.smali` `c()`/`a()` | 请求加密 `ea5`/响应解密 `da5`（libali.so）改为 UTF-8 明文直通。ARM 库在 x86_64 转译层（libndk_translation）下 ea5 必 SIGABRT——这是「断网正常、联网必闪退」的真因（详见分析结论 3） |
| 启动页后闪退修复 | `OuiCrGxF$q.run()`、`OuiCrGxF$m.c()`、`Server/e.smali` `b()`/`d()` | 配置响应解析全程 try/catch，异常降级「Server not available」对话框（BBFIX 日志）；`da2/ea2` 恒等直通（本地配置明文存取自洽）；详见分析结论 4 |
| 会员放行 | `User/a.smali` | `isExpired()` 恒返回 `false` |
| 智能选服 | `Server/i.smali` `c()` | 原「按数组顺序取第一个可达」改为：探测全部 4 个节点，选延迟最小者交换到首位 |
| 崩溃吞掉不外传 | `MainApplication$d.uncaughtException` 清空 | 保留 handler 挂载（维持原版吞异常行为），删除 `oslog/new` 上报 |
| 包名后缀 | `AndroidManifest.xml` + `g4/a.smali` | `com.secret.prettyhezi` → `com.secret.prettyhezi.tile`，FileProvider authorities 同步修改，可与原版共存 |

测试方法见 [TESTING.md](TESTING.md)。

## 构建流程

```bash
# 反编译（已注入 debuggable 供调试，成品已移除）
java -jar apktool_2.11.1.jar d BeautyBox_5.1.5.apk -o decomp

# 回编
java -jar apktool_2.11.1.jar b decomp -o BeautyBox_5.1.5_tile.apk

# 对齐 + 签名（v1/v2/v3，测试签名库 testsign.keystore）
zipalign -f -p 4 BeautyBox_5.1.5_tile.apk out.apk && mv out.apk BeautyBox_5.1.5_tile.apk
apksigner sign --ks testsign.keystore --ks-pass pass:android --key-pass pass:android BeautyBox_5.1.5_tile.apk
```

## 免责声明

本仓库仅用于本地靶场的 Android 逆向学习，所有分析与产物不得用于其他用途。
