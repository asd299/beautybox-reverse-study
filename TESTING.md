# 模拟器测试指南（BeautyBox 5.1.5 tile）

## 一、测试软件包

| 文件 | md5 | 说明 |
|---|---|---|
| [`BeautyBox_5.1.5.apk`](BeautyBox_5.1.5.apk) | — | 原版 APK（对照组，在模拟器上启动即静默退出） |
| [`BeautyBox_5.1.5_tile.apk`](BeautyBox_5.1.5_tile.apk) | `b91b580b823119a3b0f53b631be021f5` | 成品 APK（原版服务器列表 + 全部修复，zipalign + v1/v2/v3 测试签名） |
| [`BeautyBox_5.1.5_tile_mocktest.apk`](BeautyBox_5.1.5_tile_mocktest.apk) | `ed8b411109438d93ef28e950b011aa4b` | mock 测试专用包：仅将 `Server/i.smali` 的 4 个服务器硬编码为 `http://10.0.2.2:20999`，其余与成品完全一致 |

包名：`com.secret.prettyhezi.tile`（原版 `com.secret.prettyhezi` + `.tile` 后缀）
版本号：5.1.5 (versionCode 120)，与原版一致。两个 tile 包与原版签名不同，混装前先卸载旧包。

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
# 持续观察进程（TCG 环境应用 fork 后 40s 才有 pid，属正常）
for i in $(seq 1 60); do echo "t=${i}s pid=$(adb shell pidof com.secret.prettyhezi.tile)"; sleep 1; done
adb exec-out screencap -p > screen.png
adb logcat -d | grep -E "FATAL EXCEPTION|EXITHOOK"   # 应无输出
```

通过标准：
1. 进程 pid 持续存在 ≥60s；
2. 截图出现应用 UI（无外网时弹「Network not available, please check your internet connection」对话框，点 Ok 关闭对话框，均属正常）；
3. logcat 无 FATAL。

## 四、联网闪退的真因与修复验证（mock 联网测试）

### 1. 真因（tombstone 定位）

断网正常、联网必闪退的根因**不在 Java 层**：

- 所有请求/响应都过 `libali.so`：`Server.e.c(String,int)` → `c.abc.ea5([BI)[B`（请求加密），`Server.e.a([BI)` → `c.abc.da5([BI)[B`（响应解密）。
- x86_64 模拟器的 ARM 转译层（libndk_translation）执行 `ea5` 时，JNI `CallVoidMethodA` 收到 null 对象 → SIGABRT。
- 断网时 `h.a()`（ConnectivityManager）为 false，请求链不启动 → 不闪退；联网后**第一个请求加密即 abort，发生在任何网络包发出之前**。
- 成品修复：`Server/e.smali` 的 `c()`/`a()` 改为 UTF-8 明文直通（传输层本来就是 OkHttp POST，明文化不影响流程）。

### 2. 本地 mock 服务器（明文 HTTP，端口 20999）

成品包保留 https 与真实服务器列表，无法直连 mock；**mocktest 包**已把 4 个服务器硬编码为 `http://10.0.2.2:20999`（模拟器内 10.0.2.2 即宿主机回环）。应用 manifest 本身 `usesCleartextTraffic="true"`，明文可用。

```python
# mock_http20999.py — 记录请求并返回通用 JSON
import http.server, time
class H(http.server.BaseHTTPRequestHandler):
    def do_GET(self):
        body = b'{"code":200,"data":"AA==","err":""}'
        self.send_response(200)
        self.send_header("Content-Type", "application/json")
        self.send_header("Content-Length", str(len(body)))
        self.end_headers()
        self.wfile.write(body)
    do_POST = do_GET
    def log_message(self, fmt, *a):
        with open("/tmp/mock_https.log", "a") as f:
            f.write(time.strftime("[%H:%M:%S] ") + (fmt % a) + "\n")
print("mock plain http on 20999", flush=True)
http.server.ThreadingHTTPServer(("0.0.0.0", 20999), H).serve_forever()
```

```bash
python3 mock_http20999.py        # 宿主机上运行
# 验证：宿主机 curl http://127.0.0.1:20999 应返回 {"code":200,...}
```

> 为什么不用 https mock：部分沙箱/容器环境（含本仓库测试环境）对 qemu slirp 转发出的 TLS 握手有按协议的拦截（明文 HTTP 可通过、ClientHello 被丢弃），且应用自签证书链在 OkHttp 侧还需额外绕过 hostname 校验。mocktest 包直接用明文 http 最省事。

### 3. 模拟器网络栈准备（wipe-data/冷启动后需重做）

```bash
adb root
# eth0 静态地址（slirp 固定网段；若 DHCP 已自动配好可跳过）
adb shell "ip addr add 10.0.2.15/24 dev eth0; ip link set eth0 up"

# WiFi：关闭强制门户验证（netsim WiFi 出不了网，验证必然失败会导致默认网络为 none）
adb shell "settings put global captive_portal_detection_enabled 0; settings put global captive_portal_mode 0"
adb shell "svc wifi enable; sleep 12; cmd wifi status | grep 'Wifi is'"   # 应显示 connected to "AndroidWifi"
adb shell "cmd wifi connect-network AndroidWifi open"                     # 未自动连时手动连

# 关键路由：应用按 fwmark 查 per-network 路由表，需把 10.0.2.2 指到 eth0（netsim WiFi 的 TCP 转发不可用）
adb shell "ip route add 10.0.2.2/32 dev eth0 table wlan0"
adb shell "ip route add 10.0.2.2/32 dev eth0 table legacy_system"

# 端到端自检：应返回 {"code":200,...}
adb shell "printf 'GET /probe HTTP/1.0\r\n\r\n' | nc -w 6 10.0.2.2 20999"
```

### 4. 安装 mocktest 包并启动

```bash
adb uninstall com.secret.prettyhezi.tile        # 与成品包同签名体系，先卸旧
adb install -r BeautyBox_5.1.5_tile_mocktest.apk
adb logcat -c; : > /tmp/mock_https.log
adb shell am start -n com.secret.prettyhezi.tile/com.secret.prettyhezi.OuiCrGxF

# 监控 180s：pid 存活 + mock 收包
for i in $(seq 1 30); do
  echo "t=$((i*6))s pid=$(adb shell pidof com.secret.prettyhezi.tile) reqs=$(grep -c POST /tmp/mock_https.log)"
  sleep 6
done
adb exec-out screencap -p > mocktest.png
adb logcat -d | grep -cE "FATAL EXCEPTION|SIGABRT|JNI DETECTED"   # 应为 0
```

### 5. 通过标准（本仓库实测结果）

1. **mock 日志出现 4 条 `POST /query?v=1.3 HTTP/1.1" 200`**——4 个服务器探测全部到达 mock（实测 t≈36s 出现，13:27:53–54）；
2. **进程 150s+ 存活、logcat 0 次 FATAL/SIGABRT**——修复前：联网后第一个请求即 SIGABRT，pid 50s 内消失；修复后全程存活；
3. UI 弹「**Server not available, please try later**」对话框属**预期正常**——mock 返回的是假数据（`data=AA==`），应用收到响应、明文解码、解析失败后优雅降级为对话框（不再走 m0 闪退链）。对话框提供 Exit App / Latest Site 两个按钮，点任一都不会闪退。

> 该测试证明的是「联网→发包→收响应→解析」全链路不再触发 native 闪退；应用与真实服务器的完整业务交互不在靶场范围内。

### 排错

| 现象 | 原因与处理 |
|---|---|
| 弹「Network not available」 | WiFi 没连上或默认网络为 none：`cmd wifi status` 查状态；`settings put global captive_portal_detection_enabled 0` 后重开 WiFi |
| mock 零收包但 `nc 10.0.2.2 20999` 通 | 应用流量走了 per-network 表：确认 `ip route show table wlan0` 里有 `10.0.2.2/32 dev eth0`；WiFi 重连后 netd 会重建网络，需重新加路由 |
| `nc` 通但 https mock 永远零连接 | 宿主环境对 slirp 转发的 TLS 拦截（本仓库实测：明文 HTTP 过、TLS ClientHello 被丢弃）。改用明文 mock + mocktest 包 |
| WiFi 反复掉线 | `svc wifi disable && sleep 3 && svc wifi enable`，再 `cmd wifi connect-network AndroidWifi open` |
| 屏幕卡在「xxx isn't responding」 | TCG 过载导致 SystemUI/Launcher ANR，点「Wait」即可，应用本身仍在运行 |
| 应用的 DNS 解析全失败 | hosts 文件在 Android 12 解析路径中不被读取；如需劫持域名用 iptables DNAT udp 53 → 宿主 DNS mock（mocktest 包硬编码 IP，无需 DNS） |

## 五、修改点与验收对照

| 修改 | 位置 | 验证方法 |
|---|---|---|
| 环境检测器 stub | `p0/d` `p0/a` `p0/e` | 模拟器上能启动（原版直接退出） |
| `System.exit` 拦截 | 新增 `ExitHook`，6 处替换 | logcat 搜 EXITHOOK（静默版无输出即未触发） |
| `m0()` 清空（防 finish-all 闪退） | `Yclh4J3zF.smali` | mock 测试中弹「Server not available」对话框且点按钮不退出 |
| 联网 native 闪退修复（ea5/da5 明文直通） | `Server/e.smali` `c()`/`a()` | mock 联网测试通过标准第 1、2 条（修复前第一个请求即 SIGABRT） |
| 会员放行 | `User/a.smali isExpired()` 恒 false | 界面会员状态 |
| 最小延迟选服 | `Server/i.smali c()` | mock 下 4 节点全通时响应最快者置顶 |
| 崩溃上报禁用（吞异常不上传） | `MainApplication$d.uncaughtException` 清空 | 无 oslog/new 外发请求 |
| 包名 `.tile` 后缀 | Manifest + `g4/a.smali` | `adb shell pm list packages \| grep tile` |
