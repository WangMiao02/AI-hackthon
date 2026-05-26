---
name: heartbeat
description: >
  定时调度总表。heartbeat 本身不含业务逻辑，只负责按计划触发各 Skill。
  Agent 启动时加载本文件，按调度表注册所有定时任务。
---

# heartbeat — 定时调度总表

## 调度表

| 时间 | Skill | trigger_type | 说明 |
|------|-------|-------------|------|
| 每日 09:00 | `trip_reserver` | `scheduled` | 出行前三天触发，有待预订行程时推送 |
| 每日 09:00 | `expense_reimbursement` | `scheduled` | 按用户配置频率（每天/每周）推送报销提醒 |
| 每日 10:00 | `coupon_claim` | `scheduled` | 当日存在匹配券时推送领券卡片 |
| 每日 11:00 | `food_recommend` | `scheduled` | 推送午餐外卖推荐卡片，用户确认后触发 skill_order |
| 每月1号 09:00 | `medicine_reminder` | `scheduled_check` | 后台静默检查，计算药品耗尽日，写入推送队列 |
| 每月1号 09:00 | `supplies_reminder` | `scheduled_check` | 后台静默检查，计算日用品耗尽日，写入推送队列 |
| 动态日期 | `medicine_reminder` | `scheduled_push` | 由月度检查写入队列，耗尽日-3天触发推送 |
| 动态日期 | `supplies_reminder` | `scheduled_push` | 由月度检查写入队列，耗尽日-3天触发推送 |
| 每日 22:00 | `profile_updater` | `scheduled` | 后台静默运行，总结会话、更新用户画像 |

---

## 调度规则说明

**静默任务**（用户无感知，不产生任何推送）：
- `medicine_reminder` — `scheduled_check`
- `supplies_reminder` — `scheduled_check`
- `profile_updater` — 所有触发均静默

**条件触发**（满足条件才推送，不满足静默结束）：
- `trip_reserver`：存在未预订的近期行程才推送
- `expense_reimbursement`：存在未报销记录才推送
- `coupon_claim`：存在匹配可领券才推送
- `food_recommend`：每日无条件推送，用户点击「暂时不定」后当日不再推送
- `medicine_reminder` — `scheduled_push`：由队列驱动，写入时即已满足条件
- `supplies_reminder` — `scheduled_push`：同上

---

## 用户可修改项

| 配置项 | 默认值 | 修改入口 |
|--------|--------|---------|
| 报销提醒频率 | 每周五 | 报销设置 |
| 药品月度检查日 | 每月1号 | 健康设置 |
| 日用品月度检查日 | 每月1号 | 家庭设置 |

> 其余触发时间为系统固定值，用户不可修改。
