# 用户通知渠道完整流程分析

## 概述

本文档分析 Monica CRM 系统中用户通知渠道从创建、验证、启用、全量重排提醒到实际发送失败退避的完整生命周期。

---

## 一、通知渠道生命周期总览

```
创建渠道 → 验证渠道 → 启用渠道 → 全量重排提醒 → 发送提醒
                          ↓
                  发送成功 → 重新排期
                  发送失败 → 计数重试 → 达到上限 → 停用渠道 + 删除排期
```

---

## 二、Email 与 Telegram 验证路径对比

### 2.1 Email 渠道验证路径

**关键文件：**
- [CreateUserNotificationChannel.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Settings/ManageNotificationChannels/Services/CreateUserNotificationChannel.php)
- [SendVerificationEmailChannel.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Settings/ManageNotificationChannels/Jobs/SendVerificationEmailChannel.php)
- [VerifyUserNotificationChannelEmailAddress.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Settings/ManageNotificationChannels/Services/VerifyUserNotificationChannelEmailAddress.php)

**验证流程：**
1. **创建渠道**：用户在前端填写邮箱信息，调用 `CreateUserNotificationChannel`
2. **生成验证Token**：系统自动生成 `verification_token` (UUID格式)
3. **发送验证邮件**：触发 `SendVerificationEmailChannel` 队列任务，发送包含验证链接的邮件
4. **用户点击验证**：用户点击邮件中的链接，调用 `VerifyUserNotificationChannelEmailAddress`
5. **标记验证成功**：设置 `verified_at = Carbon::now()`
6. **全量重排提醒**：调用 `ScheduleAllContactRemindersForNotificationChannel`

### 2.2 Telegram 渠道验证路径

**关键文件：**
- [TelegramNotificationsController.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Settings/ManageNotificationChannels/Web/Controllers/TelegramNotificationsController.php)
- [TelegramWebhookController.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Settings/ManageNotificationChannels/Web/Controllers/TelegramWebhookController.php)

**验证流程：**
1. **创建渠道**：用户点击"添加Telegram"，调用 `CreateUserNotificationChannel`，此时 `content = 'tbd'`
2. **生成验证Token**：系统自动生成 `verification_token`
3. **用户与Bot交互**：系统引导用户在Telegram中发送 `/start {token}` 给Bot
4. **Telegram Webhook回调**：Telegram服务器调用 `TelegramWebhookController@store`
5. **解析消息**：校验消息格式 `/^\/start\s[A-Za-z0-9-]{36}$/`，提取 `chat_id`
6. **更新渠道信息**：设置 `content = $chatId` 和 `active = true`
7. **全量重排提醒**：调用 `ScheduleAllContactRemindersForNotificationChannel`

### 2.3 核心差异对比

| 维度 | Email 渠道 | Telegram 渠道 |
|------|-----------|--------------|
| **验证方式** | 邮件链接点击验证 | Bot消息Webhook回调 |
| **content初始值** | 真实邮箱地址 | `'tbd'` 占位符 |
| **content更新时机** | 创建时即确定 | Webhook回调时更新为chat_id |
| **active状态** | 需验证后启用 | Webhook回调时直接设为true |
| **verified_at字段** | 用于标记验证时间 | 不使用此字段 |
| **触发重排提醒时机** | Verify服务中触发 | Webhook控制器中触发 |

---

## 三、CreateUserNotificationChannel 重复 content 校验分析

### 3.1 校验实现

[CreateUserNotificationChannel.php#L57-L67](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Settings/ManageNotificationChannels/Services/CreateUserNotificationChannel.php#L57-L67)

```php
private function validate(): void
{
    $this->validateRules($this->data);

    $exists = UserNotificationChannel::where('content', $this->data['content'])
        ->exists();

    if ($exists) {
        throw ValidationException::withMessages(['content' => trans('The email is already taken. Please choose another one.')]);
    }
}
```

### 3.2 校验作用

1. **防止重复绑定**：确保同一个邮箱/Telegram ID 不会被多个账号绑定
2. **避免重复发送**：防止同一个接收方收到多份相同提醒
3. **数据一致性**：保证 `content` 字段在全局范围内的唯一性

### 3.3 校验局限

#### 局限1：Telegram渠道创建时绕过校验
在 [TelegramNotificationsController.php#L18-L28](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Settings/ManageNotificationChannels/Web/Controllers/TelegramNotificationsController.php#L18-L28) 中：
```php
$data = [
    'type' => UserNotificationChannel::TYPE_TELEGRAM,
    'content' => 'tbd',  // 固定值
];
```
- 创建时 `content = 'tbd'`，第一个用户创建后，后续用户无法创建Telegram渠道
- 虽然Webhook回调时会更新为真实 `chat_id`，但创建阶段的校验存在问题

#### 局限2：全局唯一性校验过于严格
- 校验是**全局**的，而非按用户隔离
- 理论上，同一邮箱可以被不同用户使用（只要他们都拥有该邮箱）
- 实际场景中可能限制了合法的多账户使用

#### 局限3：错误信息不匹配
- 错误消息硬编码为 "The email is already taken"
- 对于Telegram渠道，错误信息会误导用户

#### 局限4：不区分渠道类型
- 邮箱和Telegram使用同一字段 `content` 存储
- 理论上可能出现邮箱地址与Telegram chat_id 冲突（虽然概率极低）

---

## 四、状态变更机制分析

### 4.1 SendVerificationEmailChannel 的作用

[SendVerificationEmailChannel.php#L42-L52](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Settings/ManageNotificationChannels/Jobs/SendVerificationEmailChannel.php#L42-L52)

```php
public function handle()
{
    if ($this->channel->type !== UserNotificationChannel::TYPE_EMAIL) {
        return;
    }

    Mail::to($this->channel->content)
        ->send(new UserNotificationChannelEmailCreated($this->channel));
}
```

**状态影响：**
- **不直接修改数据库状态**
- 仅负责发送验证邮件
- 邮件包含验证链接，引导用户完成验证流程
- 是Email渠道验证流程的**触发点**，而非状态变更点

### 4.2 VerifyUserNotificationChannelEmailAddress 的状态变更

[VerifyUserNotificationChannelEmailAddress.php#L60-L73](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Settings/ManageNotificationChannels/Services/VerifyUserNotificationChannelEmailAddress.php#L60-L73)

```php
private function verify(): void
{
    $this->userNotificationChannel->verified_at = Carbon::now();
    $this->userNotificationChannel->save();
}

private function rescheduledReminders(): void
{
    (new ScheduleAllContactRemindersForNotificationChannel)->execute([...]);
}
```

**状态变更：**

| 字段 | 变更前 | 变更后 | 含义 |
|------|--------|--------|------|
| `verified_at` | `null` | 当前时间 | 标记邮箱已验证 |
| `active` | 保持不变 | 保持不变 | 依赖数据库默认值(true) |

**注意事项：**
- 仅设置 `verified_at`，不直接设置 `active`
- 依赖数据库默认值 `active = true`（需确认迁移文件）
- 验证后立即触发全量重排提醒

### 4.3 TelegramWebhookController 的状态变更

[TelegramWebhookController.php#L51-L60](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Settings/ManageNotificationChannels/Web/Controllers/TelegramWebhookController.php#L51-L60)

```php
$channel->content = $chatId;
$channel->active = true;
$channel->save();

(new ScheduleAllContactRemindersForNotificationChannel)->execute([...]);
```

**状态变更：**

| 字段 | 变更前 | 变更后 | 含义 |
|------|--------|--------|------|
| `content` | `'tbd'` | Telegram chat_id | 存储真实的接收目标 |
| `active` | 保持不变 | `true` | 显式标记渠道为启用状态 |
| `verified_at` | 不使用 | 不使用 | Telegram渠道不使用此字段 |

---

## 五、ScheduleAllContactRemindersForNotificationChannel 补齐机制分析

### 5.1 补齐逻辑实现

[ScheduleAllContactRemindersForNotificationChannel.php#L62-L100](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Settings/ManageNotificationChannels/Services/ScheduleAllContactRemindersForNotificationChannel.php#L62-L100)

```php
private function schedule(): void
{
    // 1. 获取账户下所有vault
    $vaults = $this->account()->vaults()->pluck('id')->toArray();

    // 2. 查询所有联系人提醒
    $contactReminders = DB::table('contact_reminders')
        ->join('contacts', ...)
        ->join('vaults', ...)
        ->whereIn('vaults.id', $vaults)
        ->select('contact_reminders.id', 'day', 'year', 'month')
        ->get();

    // 3. 遍历每个提醒，计算下次触发时间
    foreach ($contactReminders as $contactReminder) {
        // 计算upcomingDate...
        
        // 4. 插入排期记录
        DB::table('contact_reminder_scheduled')->insert([
            'user_notification_channel_id' => $this->userNotificationChannel->id,
            'contact_reminder_id' => $contactReminder->id,
            'scheduled_at' => $upcomingDate->tz('UTC'),
        ]);
    }
}
```

### 5.2 触发场景

根据代码注释和调用位置，补齐机制在以下场景触发：

1. **新渠道创建并验证后**
   - Email：`VerifyUserNotificationChannelEmailAddress` 中调用
   - Telegram：`TelegramWebhookController` 中调用

2. **渠道重新激活时**（代码注释提到但需确认调用位置）
   - 当渠道从非活跃状态切换到活跃状态时
   - 确保新激活的渠道不会错过已存在的提醒

### 5.3 补齐逻辑详解

#### 步骤1：获取范围
- 查询账户下所有 vault
- 查询这些 vault 中所有联系人的所有提醒

#### 步骤2：计算下次触发日期
- **无年份提醒**（如生日）：使用 `1900-{month}-{day}` 作为基准
- **有年份提醒**（如纪念日）：使用具体日期
- **过去日期处理**：
  - 如果是过去的日期，先尝试设为今年
  - 如果今年也已过去，则设为明年

#### 步骤3：应用用户偏好时间
- 使用渠道用户的时区（或系统默认时区）
- 设置为渠道配置的 `preferred_time`（小时和分钟）
- 最终转换为 UTC 时间存储

#### 步骤4：批量插入排期记录
- 直接向 `contact_reminder_scheduled` 表插入记录
- **注意：不做去重检查**，如果重复调用会产生重复排期

### 5.4 潜在问题

1. **缺少幂等性保护**
   - 如果服务被重复调用，会产生重复的排期记录
   - 建议在插入前检查是否已存在相同的 `(channel_id, reminder_id)` 组合

2. **性能问题**
   - 使用循环逐条插入，提醒数量多时性能较差
   - 建议使用 `insert()` 批量插入

3. **时区转换逻辑**
   - `shiftTimezone()` vs `setTimezone()` 的使用需要仔细验证

---

## 六、ProcessScheduledContactReminders 失败退避机制分析

### 6.1 失败处理流程

[ProcessScheduledContactReminders.php#L54-L77](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/ManageReminders/Jobs/ProcessScheduledContactReminders.php#L54-L77)

```php
catch (\Exception $e) {
    // 1. 记录错误日志
    Log::error('Error sending reminder', [...]);
    
    // 2. 记录发送失败记录
    UserNotificationSent::create([
        'user_notification_channel_id' => $userNotificationChannel->id,
        'sent_at' => Carbon::now(),
        'subject_line' => '',
        'error' => $e->getMessage(),
    ]);

    // 3. 增加失败计数
    $userNotificationChannel->refresh();
    $userNotificationChannel->fails += 1;

    // 4. 达到上限时停用渠道
    if ($userNotificationChannel->fails >= config('monica.max_notification_failures', 10)) {
        $userNotificationChannel->active = false;
        $userNotificationChannel->contactReminders->each->delete();
    }

    $userNotificationChannel->save();
}
```

### 6.2 为什么要停用渠道

**设计意图分析：**

1. **防止无限重试风暴**
   -  cron每5分钟执行一次，如果渠道持续失败，会产生大量错误日志
   -  停用渠道可以避免无效的资源消耗

2. **保护发送服务信誉**
   - 对于Email渠道：持续发送失败可能被邮件服务商标记为垃圾邮件发送者
   - 对于Telegram：频繁失败可能触发API限流或封禁

3. **故障隔离**
   - 单个渠道的失败不应影响其他渠道的正常发送
   - 停用有问题的渠道，保证系统整体可用性

4. **提示用户介入**
   - 渠道被停用后，用户在界面上会看到异常状态
   - 促使用户检查并修复问题（如邮箱无效、Telegram chat_id变更等）

### 6.3 为什么要删除 scheduled rows

**设计意图分析：**

1. **清理无效数据**
   - 渠道已停用，这些排期记录永远不会被执行
   - 删除可以保持数据库整洁，减少无用数据

2. **避免恢复后重复发送**
   - 如果用户后来修复并重新启用渠道
   - 保留旧记录可能导致过期提醒被发送
   - 重新启用时会调用 `ScheduleAllContactRemindersForNotificationChannel` 重新计算

3. **状态一致性**
   - 渠道停用 = 不发送任何提醒
   - 删除排期记录是这一状态的直接体现

4. **防止"幽灵提醒"**
   - 如果渠道在停用时被修改（如更换邮箱）
   - 旧的排期记录可能发送到错误的目标

### 6.4 退避参数

- **默认失败阈值**：`config('monica.max_notification_failures', 10)` = 10次
- **重试间隔**：由cron调度决定（每5分钟执行一次）
- **重置机制**：代码中未见失败计数的重置逻辑

### 6.5 潜在改进点

1. **缺少指数退避**
   - 当前是固定5分钟间隔重试
   - 建议实现指数退避：1分钟 → 5分钟 → 15分钟 → 30分钟 → ...

2. **缺少成功后的计数重置**
   - 成功发送后应重置 `fails` 计数器
   - 否则累计10次失败后即使后续成功也会被停用

3. **缺少通知机制**
   - 渠道被停用后应通知用户（如发送系统通知邮件）
   - 用户可能不知道渠道已停止工作

4. **软删除而非硬删除**
   - 当前是直接 `delete()` 排期记录
   - 可以考虑软删除或保留历史记录用于排查

---

## 七、总结

### 7.1 流程完整闭环

```
用户创建渠道
    ↓
CreateUserNotificationChannel
    ├─ 重复content校验（全局唯一）
    ├─ 生成verification_token
    └─ Email: 分发SendVerificationEmailChannel
       Telegram: content='tbd'
    ↓
用户验证
    ├─ Email: 点击链接 → VerifyUserNotificationChannelEmailAddress
    │      └─ 设置verified_at
    └─ Telegram: Bot Webhook → TelegramWebhookController
           ├─ 设置content=chat_id
           └─ 设置active=true
    ↓
ScheduleAllContactRemindersForNotificationChannel
    └─ 遍历所有提醒，计算下次时间，插入排期表
    ↓
ProcessScheduledContactReminders (cron每5分钟)
    ├─ 成功: 调用RescheduleContactReminderForChannel重新排期
    └─ 失败: fails++
            └─ fails >= 10: active=false + 删除所有排期
```

### 7.2 关键设计决策

| 决策 | 优点 | 缺点 |
|------|------|------|
| **content全局唯一校验** | 防止重复绑定 | Telegram创建有bug，限制多账号共享 |
| **验证后全量重排** | 新渠道立即生效 | 缺少幂等性，可能重复插入 |
| **失败10次自动停用** | 防止重试风暴 | 缺少成功重置，缺少用户通知 |
| **停用时删除排期** | 状态一致，避免幽灵提醒 | 重新启用需重新计算 |

### 7.3 建议优化方向

1. **修复Telegram创建时的content校验问题**
   - 创建Telegram渠道时跳过content校验
   - 或在Webhook回调时做chat_id唯一性校验

2. **增强Schedule服务的幂等性**
   - 插入前检查排期是否已存在
   - 使用 `upsert` 替代 `insert`

3. **完善失败退避机制**
   - 实现指数退避策略
   - 成功发送后重置失败计数
   - 渠道停用时通知用户

4. **优化性能**
   - 批量插入排期记录
   - 考虑使用队列处理大量提醒

---

*文档生成时间: 2026-05-28*
