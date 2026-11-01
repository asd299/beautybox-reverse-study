# BeautyBox 5.1.5 去广告 + 去限制计划与进度

## 任务
账号 niaohai1 / 2994498096。①去广告 ②去限制（永久热门视频权限）③登录实测。

## 分析结论（2026-11-01）
- 广告：自建系统 `com/secret/AD/`，数据入口 `AD/c.a(I)`/`c.b(I)`（数据源 Server/g.ads[]）。
  - 补丁：stub 化 `AD/c.smali`（a→空列表，b→false），全部广告位走原版"无数据→隐藏"分支。
- 热门权限（永久热门视频权限）客户端判定链：
  - `View/i`（列表点击预检）：fc≥999 且无 perm.p1 → 拦截弹窗 t1()
  - `B0xzEmZ.F1()`（视频详情）：fc<999 且 dur<600 免权限，否则查 perm.p1
  - `WV6yEmE.F1()`：me.p 为空则查 perm.p1
  - `VOGkBN.F1()`（rrvideo）：item.p2 且无 perm.p2 → 拒绝
  - 服务端：`<type>/show/json?id=` 无权限参数，服务端可返回 PermissionDenied/LimitReached（x0 处理）——需实测确认是否拦截
  - 播放 URL 仅在详情响应 `d0.ps[]/me` 中下发，列表不带 URL
- 其他：User/a.isExpired 已恒 false（前序工作）；expire.p1~p3 时间戳仅分享页展示用。

## 补丁清单
1. AD/c.smali stub（去广告）— 已应用 ✅
2. B0xzEmZ.F1 → 恒 true — 已应用 ✅
3. WV6yEmE.F1 → 恒 true — 已应用 ✅
4. View/i perm.p1 读取 → const 1 — 已应用 ✅
5. VOGkBN perm.p2 读取 → const 1 — 已应用 ✅

## 进度
- [x] armfix7（去广告版）构建签名：/tmp/armfix7-signed.apk md5=14b925bd9c572488762bf862af5ce544（不含去限制补丁）
- [ ] armfix8（去广告+去限制）构建签名
- [ ] 模拟器安装登录 niaohai1，验证去广告效果
- [ ] 验证热门视频可看（若服务端仍拦截，记录并评估）
- [ ] 交付：拷贝 APK 到 /workspace/bb-study/、更新 README/TESTING、release
