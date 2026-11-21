# 本地改动（相对上游 v1.0.1）

基于上游 `Devilstore/Glados-Railgun-checkin` v1.0.1（`8ca67db1`），只改了
`extension/background.js`。重新合并上游新版本时，按下面的清单重新应用即可。

## 背景

2026-11-21 09:34 的自动签到里，ysp 账号「页面加载超时」失败，20 秒后 lcx 账号
在同一网络下成功。日志显示签到页实际加载耗时在 12–20 秒之间波动，原来的
20 秒阈值卡在正常耗时尾部；而失败会直接进入 30 分钟自动重试退避。

`waitTabComplete` 原本只认 `tab.status === 'complete'`，也就是字体、统计脚本、
图片等所有子资源都要到位，任何慢请求都会把整次签到拖成超时。

## 改动

1. **超时阈值 20s → 45s**
   `OPEN_TAB_TIMEOUT_MS = 45000`，`openSessionTab` 与 `probeBrowserLogin` 两处使用。

2. **等待条件放宽**
   加载中会探一次 `document.readyState`，只要不是 `loading` 就放行；签到只依赖
   DOM 可交互和 `/api/` 请求，不需要等广告或统计脚本。
   加了 `onRealPage` 判断，避免导航未提交时在 `about:blank` 上误判（空白页的
   `readyState` 天生是 `complete`）。签到按钮另有最多 12 秒的等待循环，提前放行
   不会点到还没渲染的页面。

3. **打开失败本地重试一次**
   首次失败后关掉标签页、等 2 秒重开；仍失败才返回「页面加载超时」，交给原有的
   30 分钟退避。单次网络抖动不再付出半小时。

## 验证

- `node --check extension/background.js` 通过
- `waitTabComplete` 5 组桩测试：慢页面（interactive 即放行）、about:blank 陷阱、
  正常完成、标签页被关、探针抛错后恢复 —— 全部符合预期
- `openSessionTab` 3 组桩测试：一次成功（开 1 关 0）、重试成功（开 2 关 1）、
  两次失败（ok=false）—— 全部符合预期

## 加载方式

`chrome://extensions` → 开发者模式 → 「加载已解压的扩展程序」→ 选择
`extension/` 目录。
