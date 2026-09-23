---
layout: post
title: "当 kill(pid, 0) 开始说谎：一次 Linux TID 碰撞认尸实录"
date: 2026-09-23
---

*TL;DR: `kill(pid, 0)` 在 Linux 上可能对着一个早已死去的进程说“还活着”——因为复用这个号码的可能是一个线程。本文记录一次容器内幽灵进程的完整取证：现象、误诊、`/proc` 指纹、以及打给上游的三行补丁。Companion to [posit-dev/positron#16167](https://github.com/posit-dev/positron/issues/16167) and fix [#16176](https://github.com/posit-dev/positron/pull/16176). Ops scripts: [kallichore-watchman](https://github.com/Liang-Psych/kallichore-watchman).*

## 现象：删不掉的名单

Positron 的“运行中的内核 Supervisor”列表里，躺着几条删不掉的死条目：状态 `Status unavailable: connect ENOENT /tmp/kc-442.sock`，点 Shut Down 弹窗报错，关掉还在。重启容器？还在——名单存在持久化卷里，借尸还魂。

## 误诊：进程明明死了，探活却说活着

修剪逻辑长这样：`process.kill(pid, 0)` 成功即判活。Linux 上线程号（TID）和进程号（PID）共用一个编号池——442 号 supervisor 死后，Node/Tokio 的工作线程顺手拿走这个号，`kill(442, 0)` 照样返回成功。前台以为客人还在，从不打扫。

## 取证：/proc 指纹

```bash
cat /proc/442/comm    # 真身：kcserver
cat /proc/2001/comm   # 冒牌货：tokio-rt-worker
```

名字一对，冒牌当场现形。更阴的死角： supervisor 若是被 `kill -9`/OOM 直接拍死的，连 socket 都来不及删——此时“文件存在 + kill 成功”双双失明，只有 `comm` 指纹能定案。

## 三行补丁（已交上游）

1. 判活加两道：socket 必须存在；Linux 下 `/proc/<pid>/comm` 必须是 `kcserver`（非 Linux 原样放行）。
2. Shut Down 的 `catch` 里，`ENOENT`/`ECONNREFUSED` 照样 `removeByPid`——门都没了还不销账，留着过年吗。
3. 运维侧：`kc-sweeper` 定时给 socket 做握手探活，审计先行，观察期后再动手删。

## 教训

- `kill(pid, 0)` 只能证明“这个号码有人接”，证明不了“接的是本人”。
- 容器重启会重置号码池、持久卷会保留名单——两者一乘就是认尸工厂，桌面端攒不出的 bug，服务器上周周见。
- 取证先于编码：`/proc` 实测、`pid_max` 死刑验证，全是动手前先验尸。
