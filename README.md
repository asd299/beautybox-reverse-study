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

### 5. 原生层签名校验（真机版的真正障碍与破解）

真机版（原生加密 + 防闪退补丁）的闪退根因是 **`libali.so` 内置的签名/包名完整性校验**，行为分两种：

- **包名 = 原版 `com.secret.prettyhezi`**：初始化（`c.abc.abe(Context)`，splash `OuiCrGxF` 调用）即校验 `getPackageInfo` 签名，与内置证书不符 → 静默 `System.exit(0)`（离线启动 ~2.7s，Java 层 0 处 exit）。
- **包名 ≠ 原版（`.tile`）**：初始化校验跳过（模拟器离线 60s+ 存活），但联网后首次调用 `ea5/da5` 时，native 校验失败分支把 null 对象传给 `CallVoidMethodA` → ART `JniAbort`（tombstone：`JNI DETECTED ERROR IN APPLICATION: obj == null`，帧 `libndk_translation DoTrampoline_JNIEnv_CallVoidMethodV` 之下——null 来自 libali.so 自身，不是转译层引入）→ **真机 ARM 同样 abort**，表现为启动页后快速闪退。
- so 内 strings 佐证：`()Landroid/content/pm/PackageManager;`、`(Ljava/lang/String;I)Landroid/content/pm/PackageInfo;`——native 通过 JNI 回调 `getPackageManager().getPackageInfo(...)` 取签名。

**破解（Java 层签名 spoof，`bbfix/Spoof`）**：`OuiCrGxF` 的 `abe()` 调用点改为 `Spoof.init(Context)`——包一层 `ContextWrapper`：`getPackageName()` 返回原版包名、`getPackageManager()` 返回代理（仅 `getPackageInfo*` 三个重载把 `signatures` 替换为**从原版 APK 提取的 X.509 证书 DER**，其余 94 个抽象方法全量委托真实 pm）→ native 校验读到「原版包名 + 原版签名」→ 通过。

- 模拟器实测（联网，真实服务器）：`BBSPOOF: abe() invoked with spoofed context`，splash → **正常进入登录界面**，75s+ 存活，0 exit / 0 abort / 0 FATAL（对照：无 spoof 同条件 30s 内必 SIGABRT 于 AsyncTask #1）。
- **原包名版本验证**：Spoof 同样适用于原版包名——`com.secret.prettyhezi` + Spoof 离线 60s+ 存活 0 exit（对照无 Spoof 同条件 2.7s 必死）、联网进登录界面。当前真机版即原包名（安装前须卸载原版/旧包，同包名互斥）。
- 实现注记：Spoof 类以 `classes2.dex` 之外的**主 dex smali**（`smali/bbfix/`）打包；曾尝试 zip 追加独立 `classes2.dex` 导致覆盖原包 desugar 类（`j$/util/*`）→ gson `<clinit>` NoClassDefFoundError。

## 真机版（armfix）说明

`BeautyBox_5.1.5_armfix-signed.apk` 与 `tile.apk` 的区别：

| | tile.apk（模拟器/明文直通版） | armfix-signed.apk（真机/原生加密版） |
|---|---|---|
| 请求加密 `ea5`/响应解密 `da5`、配置存取 `ea2/da2` | UTF-8 明文直通 | **保留 `libali.so` native 原版调用**（与真实服务器协议兼容） |
| 服务器列表/选服 | 原版 4 节点 + 智能选服补丁 | 原版 4 节点、原版逻辑（未动） |
| 防闪退补丁（14 文件） | 有 | 有（`ExitHook` 新增 + `MainApplication$d`、`OuiCrGxF$c/$d$a/$h$a/$i$a/$m/$q`、`User/a`、`Yclh4J3zF`、`ZIgnJ$c`、`p0/a`、`p0/d`、`p0/e`） |
| 包名 | `com.secret.prettyhezi.tile` | **原版包名** `com.secret.prettyhezi`（Spoof 对 native 层呈现原版包名+签名；与原版 APK 同包名互斥，安装前先卸载原版） |
| 适用环境 | 真机 + x86 模拟器 | **仅真机（ARM）**；x86_64 模拟器上 `ea5` 转译层必 SIGABRT |
| 会员 | 放行（`isExpired()` 恒 false） | 同左 |

真机安装：**先卸载已装的原版/旧包再安装**（原版包名，签名不同互斥）。签名为本仓库测试签名，与原版签名不同属预期（native 校验已由 Spoof 穿透）。若启动后仍提示「服务连接不上」，说明服务器端对请求载荷中的设备/签名字段做了校验，属协议层问题，非本包闪退缺陷。

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
| 真机版签名 spoof | `bbfix/Spoof`（新增）+ `OuiCrGxF` abe 调用点 | 原生加密版专供：ContextWrapper + PackageManager 代理返回原版包名/证书，穿透 `libali.so` 原生签名校验（详见结论 5） |

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

## 七、去广告与去限制（armfix8，2026-11-01）

在 armfix6（原包名+Spoof）基础上叠加两组补丁，产物 `BeautyBox_5.1.5_armfix8-signed.apk`（md5 `0b874af1608791020e14c274c1de576f`），模拟器实测通过。

### 去广告
自建广告系统（无第三方 SDK）：`com/secret/AD/`，数据入口 `AD/c.a(I)`（取广告列表）/`c.b(I)`（判断某类型广告存在），数据源为服务器配置 `Server/g.ads[]`。

**补丁**：stub 化 [smali/com/secret/AD/c.smali](../decomp/smali/com/secret/AD/c.smali)——`a(I)` 恒返回空 `ArrayList`、`b(I)` 恒返回 `0`。所有广告位（splash/信息流/视频页/游戏页）走原版"无数据 → setVisibility(GONE)"正常分支，不碰流程回调，无残留逻辑。

### 去限制（永久热门视频权限）
"999 热门"= 收藏数 fc≥999 的内容。原版需开通热门权限（日/月/年/永久），否则列表点击与服务端 show/play 均受限。权限模型：

- 服务端登录响应 `Server/a0.permission (a0$g{p1,p2,p3})` 下发布尔位，`p1`=热门权限；
- 客户端预检：`View/i`（列表点击，fc≥999 且无 p1 → 弹"需要热门权限"）、`B0xzEmZ.F1()`（视频：fc<999 且时长<600s 免权限，否则查 p1）、`WV6yEmE.F1()`、`VOGkBN.F1()`（rrvideo 查 p2）。

**补丁**（4 处，全部强制放行）：
1. `View/i` perm.p1 读取 → `const/4 v0, 1`
2. `B0xzEmZ.F1()` → 恒 `return true`
3. `WV6yEmE.F1()` → 恒 `return true`
4. `VOGkBN.F1()` perm.p2 读取 → `const/4 v0, 1`

服务端 `show/play` 请求本身不带权限参数（仅 Authorization 头），**权限判定在服务端按账号状态执行**（网络层埋点实测：`permission.p1=false` 的账号 show/play 被拒；播放文件地址 `me.p="type3/cdn2/vd4/480/<hash>.mp4"` 仅在授权响应中下发，且 hash 与视频 ID/封面 hash 无推导关系，**纯客户端无法绕过**）。客户端补丁的作用是移除本地预检弹窗、让流程直达服务端校验。

**热门视频观看的可行途径（官方内置）**：① 账号自带的「超级权限卡」（免费，激活后 24h 全站权限，邀请他人注册可再得卡）——实测激活成功；② 积分解锁单个视频；③ 购买权限。

### 实测结论（模拟器，测试账号登录）
- 启动页/主界面无广告；公告、跑马灯（应用自身系统）正常；
- 客户端预检放行生效：热门视频点开不再弹本地权限弹窗，直达详情页；
- 服务端仍按账号状态强制校验：`permission.p1=false` 的账号 show/play 被拒（403/「没有热门视频查看权限」）；用账号自带的超级权限卡激活（24h 全站权限）后 show 即下发播放数据。详见 TESTING.md 第七节。

## 免责声明

本仓库仅用于本地靶场的 Android 逆向学习，所有分析与产物不得用于其他用途。
