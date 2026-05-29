# 联系人重要日期创建与提醒机制分析

## 1. 三种日期输入类型的解析处理

### 1.1 日期类型常量定义

三种日期类型在 [ContactImportantDate.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Models/ContactImportantDate.php#L35-L39) 中定义：

```php
public const TYPE_FULL_DATE = 'full_date';
public const TYPE_MONTH_DAY = 'month_day';
public const TYPE_YEAR = 'year';
```

### 1.2 Controller 层的 date parts 解析

[ContactImportantDatesController.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/ManageContactImportantDates/Web/Controllers/ContactImportantDatesController.php#L115-L139) 中的 `getDateParts()` 方法负责解析前端传入的三种日期格式：

```php
private function getDateParts(Request $request): array
{
    $day = '';
    $month = '';
    $year = '';

    switch ($request->input('choice')) {
        case ContactImportantDate::TYPE_FULL_DATE:
            $year = Carbon::parse($request->input('date'))->year;
            $month = Carbon::parse($request->input('date'))->month;
            $day = Carbon::parse($request->input('date'))->day;
            break;
        case ContactImportantDate::TYPE_MONTH_DAY:
            $month = $request->input('month');
            $day = $request->input('day');
            break;
        case ContactImportantDate::TYPE_YEAR:
            $year = Carbon::now()->subYears($request->input('age'))->format('Y');
            break;
        default:
            throw new \InvalidArgumentException('Invalid date type');
    }

    return [$day, $month, $year];
}
```

**解析逻辑说明：**

| 输入类型 | 前端参数 | 解析逻辑 | 输出结果 |
|---------|---------|---------|---------|
| **完整日期 (full_date)** | `date` (完整日期字符串) | 使用 `Carbon::parse()` 解析，分别提取年、月、日 | `day`、`month`、`year` 均有值 |
| **仅月日 (month_day)** | `month`、`day` | 直接使用传入值，年份留空 | `day`、`month` 有值，`year` 为空字符串 |
| **仅年龄/年份 (year)** | `age` (年龄数值) | 使用 `Carbon::now()->subYears($age)` 计算出生年份，仅保留年份 | `year` 有值，`day`、`month` 为空字符串 |

> **注意**：`TYPE_YEAR` 类型通过年龄反向推算年份，但不记录具体月日。例如：输入年龄 30，当前年份 2026，则推算年份为 1996。

### 1.3 解析结果在 store/update 中的使用

解析后的 `[$day, $month, $year]` 会同时传递给两个服务：

1. **创建/更新重要日期**：传递给 `CreateContactImportantDate` 或 `UpdateContactImportantDate`
2. **创建/更新关联提醒**（如果用户勾选了提醒）：传递给 `CreateContactReminder` 或 `UpdateContactReminder`

## 2. CreateContactImportantDate 服务分析

### 2.1 类型归属验证

[CreateContactImportantDate.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/ManageContactImportantDates/Services/CreateContactImportantDate.php#L72-L81) 中的 `validate()` 方法验证日期类型的归属：

```php
private function validate(): void
{
    $this->validateRules($this->data);

    // make sure the vault matches
    if (! is_null($this->valueOrNull($this->data, 'contact_important_date_type_id'))) {
        $this->vault->contactImportantDateTypes()
            ->findOrFail($this->data['contact_important_date_type_id']);
    }
}
```

**验证逻辑：**

- 首先通过 `validateRules()` 验证基础字段规则（account_id、vault_id、author_id、contact_id 等必须存在且有效）
- `contact_important_date_type_id` 是可选的，如果提供了该值，必须确保它属于当前 vault 下的日期类型
- 通过 `$this->vault->contactImportantDateTypes()->findOrFail()` 确保类型归属正确，防止越权访问其他 vault 的日期类型

### 2.2 日期存储

[CreateContactImportantDate.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/ManageContactImportantDates/Services/CreateContactImportantDate.php#L57-L64) 中创建记录：

```php
$this->date = ContactImportantDate::create([
    'contact_id' => $data['contact_id'],
    'contact_important_date_type_id' => $this->valueOrNull($data, 'contact_important_date_type_id'),
    'label' => $data['label'],
    'day' => $this->valueOrNull($data, 'day'),
    'month' => $this->valueOrNull($data, 'month'),
    'year' => $this->valueOrNull($data, 'year'),
]);
```

> **关键处理**：使用 `valueOrNull()` 方法将空字符串转换为 `null` 存入数据库，确保数据库中不会存储空字符串。

### 2.3 Feed Item 生成

[CreateContactImportantDate.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/ManageContactImportantDates/Services/CreateContactImportantDate.php#L89-L99) 中的 `createFeedItem()` 方法：

```php
private function createFeedItem(): void
{
    $feedItem = ContactFeedItem::create([
        'author_id' => $this->author->id,
        'contact_id' => $this->contact->id,
        'action' => ContactFeedItem::ACTION_IMPORTANT_DATE_CREATED,
        'description' => $this->date->label.' '.ImportantDateHelper::formatDate($this->date, $this->author),
    ]);

    $this->date->feedItem()->save($feedItem);
}
```

**Feed Item 特点：**

- 动作类型为 `ACTION_IMPORTANT_DATE_CREATED`
- 描述由标签和格式化后的日期组成，使用 `ImportantDateHelper::formatDate()` 根据用户偏好格式化日期
- 通过多态关联 `feedItem()` 与重要日期建立关联

## 3. ImportantDateHelper 类型判断与年龄计算

### 3.1 类型判断逻辑

[ImportantDateHelper.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Helpers/ImportantDateHelper.php#L83-L110) 中的 `determineType()` 方法根据存储的字段值反推日期类型：

```php
public static function determineType(ContactImportantDate $date): ?string
{
    // case: date is empty
    if (! $date->day && ! $date->month && ! $date->year) {
        return null;
    }

    // case: full date
    if ($date->day && $date->month && $date->year) {
        $type = ContactImportantDate::TYPE_FULL_DATE;
    }

    // case: only know the year
    if (! $date->day && ! $date->month && $date->year) {
        $type = ContactImportantDate::TYPE_YEAR;
    }

    // case: only know the month and day
    if ($date->day && $date->month && ! $date->year) {
        $type = ContactImportantDate::TYPE_MONTH_DAY;
    }

    return $type;
}
```

**类型判断规则：**

| day | month | year | 推断类型 |
|-----|-------|------|---------|
| 有值 | 有值 | 有值 | `TYPE_FULL_DATE` |
| 有值 | 有值 | null | `TYPE_MONTH_DAY` |
| null | null | 有值 | `TYPE_YEAR` |
| null | null | null | `null` |

### 3.2 年龄计算

[ImportantDateHelper.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Helpers/ImportantDateHelper.php#L14-L38) 中的 `getAge()` 方法：

```php
public static function getAge(ContactImportantDate $date): ?int
{
    $age = 0;

    if (self::determineType($date) === ContactImportantDate::TYPE_FULL_DATE) {
        $age = Carbon::parse($date->year.'-'.$date->month.'-'.$date->day)->age;
    }

    if (self::determineType($date) === ContactImportantDate::TYPE_YEAR) {
        $currentDate = (string) $date->year;
        $age = Carbon::createFromFormat('Y', $currentDate)->age;
    }

    if (self::determineType($date) === ContactImportantDate::TYPE_MONTH_DAY) {
        return null;
    }

    if (! $date->day && ! $date->month && ! $date->year) {
        return null;
    }

    return $age;
}
```

**年龄计算规则：**

- **完整日期**：使用 `Carbon::parse()->age` 计算精确年龄（考虑了生日是否已过）
- **仅年份**：使用 `Carbon::createFromFormat('Y', $year)->age` 计算，结果可能比实际大 1 岁（如果生日还没到）
- **仅月日**：无法计算年龄，返回 `null`
- **空日期**：返回 `null`

### 3.3 日期格式化

[ImportantDateHelper.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Helpers/ImportantDateHelper.php#L44-L74) 中的 `formatDate()` 方法根据用户偏好格式化日期：

```php
public static function formatDate(ContactImportantDate $date, User $user): ?string
{
    if (self::determineType($date) === ContactImportantDate::TYPE_FULL_DATE) {
        $dateAsString = Carbon::parse($date->year.'-'.$date->month.'-'.$date->day)->isoFormat($user->date_format);
    }

    if (self::determineType($date) === ContactImportantDate::TYPE_YEAR) {
        $dateAsString = $date->year;
    }

    if (self::determineType($date) === ContactImportantDate::TYPE_MONTH_DAY) {
        $date = Carbon::parse('1900-'.$date->month.'-'.$date->day);
        $dateAsString = match ($user->date_format) {
            'MMM DD, YYYY' => Carbon::parse($date)->isoFormat('MMM DD'),
            'DD MMM YYYY' => Carbon::parse($date)->isoFormat('DD MMM'),
            'YYYY/MM/DD' => Carbon::parse($date)->isoFormat('MM/DD'),
            default => Carbon::parse($date)->isoFormat('DD/MM'),
        };
    }

    return $dateAsString;
}
```

**格式化特点：**

- **完整日期**：使用用户设置的 `date_format` 完整格式化
- **仅年份**：直接显示年份数字
- **仅月日**：使用 1900 年作为假年份构造 Carbon 对象，根据用户日期格式偏好去掉年份部分显示

## 4. 提醒创建与调度机制

### 4.1 提醒创建流程

当用户在创建重要日期时勾选了提醒选项（`$request->input('reminder')` 为 true），[ContactImportantDatesController.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/ManageContactImportantDates/Web/Controllers/ContactImportantDatesController.php#L50-L63) 会调用 `CreateContactReminder` 服务：

```php
if ($request->input('reminder')) {
    (new CreateContactReminder)->execute([
        'account_id' => Auth::user()->account_id,
        'author_id' => Auth::id(),
        'vault_id' => $vaultId,
        'contact_id' => $contactId,
        'label' => $request->input('label'),
        'day' => $day,
        'month' => $month,
        'year' => $year,
        'type' => $request->input('reminderChoice'),
        'frequency_number' => 1,
    ]);
}
```

### 4.2 CreateContactReminder 服务

[CreateContactReminder.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/ManageReminders/Services/CreateContactReminder.php#L51-L93) 执行流程：

```php
public function execute(array $data): ContactReminder
{
    $this->validateRules($data);
    $this->data = $data;

    $this->createContactReminder();
    $this->updateLastUpdatedDate();
    $this->scheduledReminderForAllUsersInVault();

    return $this->reminder;
}
```

**关键步骤：**

1. **创建提醒记录**：存储 `label`、`day`、`month`、`year`、`type`（提醒类型）、`frequency_number`
2. **更新联系人最后编辑时间**
3. **为 vault 中所有用户调度提醒**：调用 `scheduledReminderForAllUsersInVault()`

```php
private function scheduledReminderForAllUsersInVault(): void
{
    $users = $this->vault->users()->get();

    foreach ($users as $user) {
        (new ScheduleContactReminderForUser)->execute([
            'contact_reminder_id' => $this->reminder->id,
            'user_id' => $user->id,
        ]);
    }
}
```

> **重要特性**：提醒是按用户调度的，每个 vault 中的用户都会有独立的调度记录，支持按各自时区和偏好时间发送。

### 4.3 ScheduleContactReminderForUser 调度服务

[ScheduleContactReminderForUser.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/ManageReminders/Services/ScheduleContactReminderForUser.php#L39-L89) 是核心调度逻辑。

#### 4.3.1 日期解析

```php
private function getDate(): void
{
    if (! $this->contactReminder->year) {
        $this->upcomingDate = Carbon::parse('1900-'.$this->contactReminder->month.'-'.$this->contactReminder->day);
    } else {
        $this->upcomingDate = Carbon::parse($this->contactReminder->year.'-'.$this->contactReminder->month.'-'.$this->contactReminder->day);
    }
}
```

**日期处理策略：**

- **有完整日期**：直接解析完整日期
- **仅有月日**：使用 1900 年作为占位年份构造日期（后续会调整到正确年份）

#### 4.3.2 时区与首选时间处理

```php
private function schedule(): void
{
    // is the date in the past? if so, we need to schedule the reminder
    // for next year
    if ($this->upcomingDate->isPast()) {
        $this->upcomingDate->year = Carbon::now()->year;
        if ($this->upcomingDate->isPast()) {
            $this->upcomingDate->year = Carbon::now()->addYear()->year;
        }
    }

    $notificationChannels = $this->user->notificationChannels;
    foreach ($notificationChannels as $channel) {
        $this->upcomingDate->shiftTimezone($this->user->timezone ?? config('app.timezone'));
        $this->upcomingDate->hour = $channel->preferred_time->hour;
        $this->upcomingDate->minute = $channel->preferred_time->minute;

        $this->contactReminder->userNotificationChannels()->syncWithoutDetaching([$channel->id => [
            'scheduled_at' => $this->upcomingDate->tz('UTC'),
        ]]);
    }
}
```

**调度逻辑详解：**

1. **年份调整**：
   - 如果日期已过（使用占位年份 1900 时必然已过），先设置为当前年份
   - 如果设置为当前年份后仍然已过（说明今年的这个日期已经过了），则设置为明年

2. **时区转换**：
   - 使用 `shiftTimezone()` 将日期转换到用户时区（`$this->user->timezone`），如果用户未设置时区则使用系统默认时区

3. **设置首选时间**：
   - 从用户的通知通道（`UserNotificationChannel`）获取 `preferred_time`（用户偏好的提醒发送时间）
   - 将日期的小时和分钟设置为用户偏好时间

4. **存储调度时间**：
   - 最后将日期转换回 UTC 时区，存储到 `contact_reminder_scheduled` 中间表的 `scheduled_at` 字段
   - 使用 `syncWithoutDetaching()` 确保不会重复创建关联

**调度示例：**

假设：
- 当前时间：2026-05-28
- 重要日期：3月15日（无年份，生日）
- 用户时区：Asia/Shanghai (UTC+8)
- 用户偏好时间：09:00

调度过程：
1. 初始日期：1900-03-15 → 已过
2. 调整年份：2026-03-15 → 已过（今年生日已过）
3. 调整年份：2027-03-15 → 未来
4. 转换时区：2027-03-15 00:00 Asia/Shanghai
5. 设置偏好时间：2027-03-15 09:00 Asia/Shanghai
6. 转回 UTC 存储：2027-03-15 01:00 UTC

## 5. Update/Destroy 中的 TODO 与一致性缺口

### 5.1 Update 操作中的问题

[ContactImportantDatesController.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/ManageContactImportantDates/Web/Controllers/ContactImportantDatesController.php#L72-L113) 中的 `update()` 方法：

```php
public function update(Request $request, string $vaultId, string $contactId, string $dateId)
{
    [$day, $month, $year] = $this->getDateParts($request);

    // ... 更新重要日期 ...

    if ($request->input('reminder')) {
        // TODO - not working yet
        (new UpdateContactReminder)->execute([
            'account_id' => Auth::user()->account_id,
            'author_id' => Auth::id(),
            'vault_id' => $vaultId,
            'contact_id' => $contactId,
            'contact_reminder_id' => $request->input('contact_reminder_id'),
            'label' => $request->input('label'),
            'day' => $day,
            'month' => $month,
            'year' => $year,
            'type' => $request->input('reminderChoice'),
            'frequency_number' => 1,
        ]);
    }

    // ...
}
```

**存在的问题：**

1. **TODO 标记**：代码明确标注 `// TODO - not working yet`，说明更新提醒的功能尚未完善
2. **缺少重新调度逻辑**：即使 `UpdateContactReminder` 更新了提醒的基本信息，但不会重新调用 `ScheduleContactReminderForUser` 来重新计算下一次触发时间
3. **前端依赖**：依赖前端传入 `contact_reminder_id`，如果前端未传入或传入错误，会导致更新失败
4. **单向更新**：只有当 `$request->input('reminder')` 为 true 时才会尝试更新提醒，但如果用户取消了提醒勾选，代码不会删除已有的提醒

### 5.2 Destroy 操作中的问题

[ContactImportantDatesController.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/ManageContactImportantDates/Web/Controllers/ContactImportantDatesController.php#L141-L158) 中的 `destroy()` 方法：

```php
public function destroy(Request $request, string $vaultId, string $contactId, string $dateId)
{
    $data = [
        'account_id' => Auth::user()->account_id,
        'author_id' => Auth::id(),
        'vault_id' => $vaultId,
        'contact_id' => $contactId,
        'contact_important_date_id' => $dateId,
    ];

    (new DestroyContactImportantDate)->execute($data);

    // TODO - delete the reminder if it exists

    return response()->json([
        'data' => true,
    ], 200);
}
```

**存在的问题：**

1. **孤儿提醒**：重要日期被删除后，关联的提醒记录仍然存在于 `contact_reminders` 表中，成为"孤儿"数据
2. **无效调度**：提醒对应的调度记录（`contact_reminder_scheduled`）也仍然存在，可能导致已经删除的重要日期仍然触发提醒通知
3. **Feed Item 不一致**：`DestroyContactImportantDate` 会创建删除动作的 Feed Item，但提醒的删除没有对应的 Feed Item

### 5.3 一致性缺口总结

| 场景 | 问题 | 数据一致性影响 |
|------|------|--------------|
| **更新重要日期+日期变更** | 提醒的日期信息可能未同步更新，调度时间未重新计算 | 用户可能在错误的日期收到提醒 |
| **更新重要日期+取消提醒** | 没有代码处理取消提醒的场景，原有提醒仍然存在 | 用户仍然会收到已取消的提醒 |
| **更新重要日期+新增提醒** | 依赖前端传入 `contact_reminder_id`，但新增时应该没有这个 ID | 可能导致创建失败或创建重复提醒 |
| **删除重要日期** | 关联的提醒和调度记录不会被删除 | 数据库中存在孤儿数据，可能触发无效通知 |
| **多用户场景** | 更新/删除时不会通知其他 vault 用户 | 其他用户的调度记录不会同步更新/删除 |

### 5.4 缺失的关联关系

查看 [ContactImportantDate.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Models/ContactImportantDate.php) 和 [ContactReminder.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Models/ContactReminder.php) 的模型关系：

- `ContactImportantDate` 模型中没有定义与 `ContactReminder` 的关联关系
- `ContactReminder` 模型中也没有定义与 `ContactImportantDate` 的关联关系
- 两者仅通过 `contact_id` + `label` + `day` + `month` + `year` 隐式关联

这种隐式关联导致：
- 无法通过模型关系方便地获取重要日期对应的提醒
- 删除重要日期时无法利用数据库外键或模型事件自动删除提醒
- 更新时需要手动匹配两者，容易出错

## 6. 总结

### 6.1 完整创建流程（正常路径）

```
用户选择日期类型和输入
        ↓
ContactImportantDatesController::getDateParts() 解析为 day/month/year
        ↓
CreateContactImportantDate 执行：
  - 验证类型归属
  - 创建重要日期记录
  - 生成 Feed Item
        ↓
如果用户勾选提醒：
  CreateContactReminder 执行：
    - 创建提醒记录
    - 为 vault 中每个用户调用 ScheduleContactReminderForUser
      - 解析日期（处理无年份情况）
      - 调整年份（确保是未来日期）
      - 按用户时区转换
      - 设置用户偏好时间
      - 转回 UTC 存储调度时间
```

### 6.2 数据一致性风险

当前实现中，创建流程相对完整，但更新和删除流程存在明显的 TODO 标记和逻辑缺失，导致重要日期与提醒之间的数据一致性无法保证。主要风险点包括：

1. **更新重要日期**：提醒不会重新调度，可能在错误时间触发
2. **删除重要日期**：提醒不会被级联删除，可能产生无效通知
3. **关联缺失**：两个实体间没有显式关联，依赖隐式匹配容易出错

建议在后续迭代中：
- 为 `ContactImportantDate` 和 `ContactReminder` 建立显式关联（如添加 `contact_important_date_id` 外键到 `contact_reminders` 表）
- 完善 `update()` 方法，处理提醒的创建、更新、取消三种场景
- 在 `destroy()` 方法中添加删除关联提醒的逻辑
- 确保更新日期后重新调用 `ScheduleContactReminderForUser` 计算新的触发时间
