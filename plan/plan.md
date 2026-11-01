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
- [x] armfix7（去广告版）构建签名：md5=14b925bd9c572488762bf862af5ce544（不含去限制补丁）
- [x] armfix8（去广告+去限制）构建签名：md5=0b874af1608791020e14c274c1de576f
- [x] 模拟器实测（2026-11-01）：安装 armfix8 → 登录 niaohai1（图片验证码人工点选通过）→ 登录成功
- [x] 去广告验证：启动页无广告、主界面无广告位，公告/跑马灯（应用自身系统）正常
- [x] 去限制验证：短视频区切 AllT 排序，fc=999 热门视频点开无"需要热门权限"弹窗，详情页正常加载，画面变化+CDN 连接确认视频播放中；第二个热门视频复现通过
- [x] 结论：热门视频限制为客户端预检（View/i 列表点击 + F1 详情判定），补丁后服务端 show/play 均正常放行，无需服务端绕过

## 遗留说明
- 服务端 PermissionDenied/LimitReached 分支保留（真实服务端拒绝时仍按原版提示）
- 模拟器 TCG 软渲染偶发 System UI ANR，属模拟器性能问题，与应用无关（真机无此现象）
