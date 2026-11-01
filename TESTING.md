# 模拟器测试指南（BeautyBox 5.1.5 tile）

## 一、测试软件包

| 文件 | 说明 |
|---|---|
| [`BeautyBox_5.1.5.apk`](BeautyBox_5.1.5.apk) | 原版 APK（对照组，在模拟器上启动即静默自杀） |
| [`BeautyBox_5.1.5_tile.apk`](BeautyBox_5.1.5_tile.apk) | 成品 APK（已 zipalign + v1/v2/v3 签名，包名 `com.secret.prettyhezi.tile`，可与原版共存） |

包名：`com.secret.prettyhezi.tile`（原版 `com.secret.prettyhezi` + `.tile` 后缀）
版本号：5.1.5 (versionCode 120)，与原版一致。

## 二、模拟器要求

| 项目 | 要求 |
|---|---|
| 系统镜像 | Android 12（API 31）x86_64，**google_apis**（非 Google Play，需支持 `adb root`） |
| 架构 | x86_64；无 KVM 环境可加 `-accel off`（TCG 软件模拟，启动慢属正常） |
| 配置 | 2048MB 内存 / 2 核即可 |

### AVD 创建示例

```bash
avdmanager create avd -n bb31 -k "system-images;android-31;google_apis;x86_64" -d pixel_4
emulator -avd bb31 -no-window -no-audio -no-boot-anim \
  -gpu swiftshader_indirect -no-snapshot -memory 2048 -cores 2 \
  -accel off -no-metrics
```

等待启动完成：`adb shell getprop sys.boot_completed` 返回 `1`。

## 三、基础测试（离线基线）

验证目标：安装、启动、界面正常渲染，不闪退。

```bash
adb install -r BeautyBox_5.1.5_tile.apk
adb logcat -c
adb shell am start -n com.secret.prettyhezi.tile/com.secret.prettyhezi.OuiCrGxF
# 持续观察进程（TCG 环境应用启动到 Displayed 约 40s）
for i in $(seq 1 60); do echo "t=${i}s pid=$(adb shell pidof com.secret.prettyhezi.tile)"; sleep 1; done
adb exec-out screencap -p > screen.png
adb logcat -d | grep -E "FATAL EXCEPTION|EXITHOOK"   # 应无输出
```

通过标准：
1. 进程 pid 持续存在 ≥60s；
2. 截图出现应用 UI（无外网时弹「Network not available, please check your internet connection」对话框，点 Ok 关闭对话框，均属正常）；
3. logcat 无 FATAL。

## 四、联网模拟测试（复现原闪退触发条件）

原版闪退机制：联网后连上服务器 → 服务器校验签名失败 → 下发指令 → `Yclh4J3zF.m0()` finish 全部 Activity（+`System.exit`）→ 表现为闪退。**断网时连不上服务器，触发不了该路径，因此断网不闪退**。

成品已将 `m0()` 清空为 no-op（8 个调用点全部失效）。离线环境无法连真实服务器，可用本地 mock 服务器 + 流量劫持模拟「联网可达服务器」场景：

### 1. 沙箱/宿主机起 mock HTTPS 服务器（端口 20053）

应用内置 trust-all 证书（`j$a.checkServerTrusted` 为空），自签证书即可。

```python
# mock_https.py — 任何请求返回 {"code":200,"data":"AA==","err":""}
import http.server, ssl
class H(http.server.BaseHTTPRequestHandler):
    def do_GET(self):
        self.send_response(200); self.end_headers(); self.wfile.write(b'{"code":200,"data":"AA==","err":""}')
    do_POST = do_GET
    def log_message(self, *a): pass
ctx = ssl.SSLContext(ssl.PROTOCOL_TLS_SERVER)
ctx.load_cert_chain("/tmp/mockcert.pem", "/tmp/mockkey.pem")
srv = http.server.ThreadingHTTPServer(("0.0.0.0", 20053), H)
srv.socket = ctx.wrap_socket(srv.socket, server_side=True)
srv.serve_forever()
```

```bash
openssl req -x509 -newkey rsa:2048 -keyout /tmp/mockkey.pem -out /tmp/mockcert.pem -days 30 -nodes -subj "/CN=z555b.com"
python3 mock_https.py &
adb reverse tcp:20053 tcp:20053   # 模拟器 127.0.0.1:20053 → 宿主机 mock
```

### 2. 模拟器内劫持应用服务器流量（需 `adb root`）

应用服务器列表（`Server/i.smali`）：`https://z555b.com:20053`、`https://g333d.com:20053`、`https://45.61.200.158:10053`、`https://45.61.244.69:10053`。用 iptables 将目标端口全部 DNAT 到 mock：

```bash
adb root
adb shell sysctl -w net.ipv4.conf.all.route_localnet=1
adb shell iptables -t nat -A OUTPUT -p tcp --dport 10053 -j DNAT --to-destination 127.0.0.1:20053
adb shell iptables -t nat -A OUTPUT -p tcp --dport 20053 -j DNAT --to-destination 127.0.0.1:20053
```

### 3. 启动应用观察

```bash
adb logcat -c
adb shell am start -n com.secret.prettyhezi.tile/com.secret.prettyhezi.OuiCrGxF
# 60s 存活监控同上，另观察 mock 日志确认流量已打到本地服务器
```

### 排错：应用流量仍 unreachable（模拟器网络栈异常时）

正常模拟器（wlan0 就绪）无需此步。若模拟器接口异常（`adb shell ip addr` 只有 lo），应用流量会命中 Android 策略路由兜底规则（rule 32000 unreachable）。临时绕过：

```bash
adb root && adb remount
adb shell ip route add 10.0.0.0/8 dev eth0 table legacy_system 2>/dev/null
adb shell ip route add 45.61.200.158/32 dev eth0 table legacy_system 2>/dev/null
adb shell ip route add 45.61.244.69/32 dev eth0 table legacy_system 2>/dev/null
```

另注意：netd 的 DNS 解析不走 nat OUTPUT（DNS 劫持无效）；`killall netd` 会连带 system_server 重启（禁用）。

通过标准：
1. mock 日志出现 `POST /query?v=1.3`、`POST .../echo/json` 等请求记录（证明应用已联网走到服务器响应阶段）；
2. 进程存活、UI 不消失——服务器返回垃圾数据时应用弹「Error: xxx」对话框属正常，点确定不应退出（原版在此路径直接 finish 全部 Activity）；
3. logcat 无 FATAL。

## 五、修改点与验收对照

| 修改 | 位置 | 验证方法 |
|---|---|---|
| 环境检测器 stub | `p0/d` `p0/a` `p0/e` | 模拟器上能启动（原版直接退出） |
| `System.exit` 拦截 | 新增 `ExitHook`，6 处替换 | logcat 搜 EXITHOOK（静默版无输出即未触发） |
| `m0()` 清空（防 finish-all 闪退） | `Yclh4J3zF.smali` | 联网 mock 测试通过标准第 2 条 |
| 会员放行 | `User/a.smali isExpired()` 恒 false | 界面会员状态 |
| 最小延迟选服 | `Server/i.smali c()` | mock 下 4 节点全通时响应最快者置顶 |
| 崩溃上报禁用（吞异常不上传） | `MainApplication$d.uncaughtException` 清空 | 无 oslog/new 外发请求 |
| 包名 `.tile` 后缀 | Manifest + `g4/a.smali` | `adb shell pm list packages \| grep tile` |
