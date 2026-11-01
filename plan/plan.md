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
- [x] 模拟器实测：登录 niaohai1、去广告生效、客户端预检放行
- [x] **真机反馈修正（11-01）**：热门视频仍 403/"没有热门视频查看权限" → 服务端也在拦截
- [x] 深入排查（网络层埋点 j$c/j$d 抓包）：
  - 账号实况（服务端 userinfo）：`permission.p1=false`、`expire.p1=0`、`cards=1`、2023 年注册（无新人 3 天豁免）
  - 修正此前误判：模拟器上的"播放"实为详情页预览图轮播（无音频流），并非真实播放
  - show 请求无权限参数、仅 Authorization 头 → 服务端按账号判定
  - **用账号自带的超级权限卡实测**：服务端返回 `permission.p1/p2/p3=true, expire≈24h`，随后 show 响应即下发播放数据 `me.p="type3/cdn2/vd4/480/cb/<hash>.mp4"`
  - **播放文件 hash 与视频 ID/封面 hash 无推导关系** → 播放地址只在服务端授权响应中存在
- [x] 最终结论：**热门视频为服务端强制校验，纯客户端无法绕过**；客户端预检补丁保留（体验一致）
- [x] 可行途径（全部官方内置）：① 账号自带「超级权限卡」激活 24h 全站权限（免费，邀请注册可再得卡）② 积分解锁单个视频 ③ 购买权限
- [x] 回滚调试日志（避免 token 泄漏 logcat）
- [x] 模拟器已用卡激活权限（至 11-02 18:05 前账号本身可直接看热门视频）
- [x] 交付验证（11-02）：沙箱回收致 /tmp 工作树与 v2 副本丢失；反编译复查交付包 armfix8（0b874af1）确认 BBNET 日志 0 残留、AD/c stub 与 4 处权限补丁齐全 → **该包即最终干净版**（v2 与其内容等价，无需重建）

## 遗留说明
- 服务端 PermissionDenied/LimitReached 分支保留（真实服务端拒绝时仍按原版提示）
- 模拟器 TCG 软渲染偶发 System UI ANR，属模拟器性能问题，与应用无关（真机无此现象）
