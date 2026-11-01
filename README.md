# beautybox-reverse-study

BeautyBox 5.1.5 逆向分析靶场（本地学习用）。

## 仓库内容

- `BeautyBox_5.1.5.apk` — 原版 APK（靶场样本）
- `BeautyBox_5.1.5_tile.apk` — 成品 APK（原版服务器列表 + 全部修复）
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

## 修改清单（Release 产物）

| 修改 | 位置 | 内容 |
|---|---|---|
| 闪退修复 | `p0/d`、`p0/a`、`p0/e`、`ExitHook`（新增） | 检测器 stub + exit 拦截 |
| 闪退修复（联网） | `Yclh4J3zF.m0()` 清空 | m0 原逻辑=逆序 finish 全部 Activity + 清栈 + exit。服务器校验签名失败下发指令走 8 个调用点触发 m0 → 界面全关=闪退观感（断网连不上服务器故不闪退）。清空后 8 个调用点全部 no-op |
| 联网 native 闪退修复 | `Server/e.smali` `c()`/`a()` | 请求加密 `ea5`/响应解密 `da5`（libali.so）改为 UTF-8 明文直通。ARM 库在 x86_64 转译层（libndk_translation）下 ea5 必 SIGABRT——这是「断网正常、联网必闪退」的真因（详见分析结论 3） |
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
