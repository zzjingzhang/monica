# 联系人目标模块逻辑分析

## 一、CreateGoal：验证 contact 归属并写入目标

### 源码位置

[CreateGoal.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/ManageGoals/Services/CreateGoal.php)

### 执行流程

`CreateGoal::execute()` 方法按顺序执行以下步骤：

1. **验证（validate）** → 2. **创建目标（create）** → 3. **创建动态条目（createFeedItem）** → 4. **更新联系人时间戳**

### 验证 contact 归属的完整链路

验证并非在 `CreateGoal` 自身完成，而是通过继承 [BaseService](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Services/BaseService.php) 的权限系统来实现。`CreateGoal::permissions()` 声明了四项权限约束：

```php
return [
    'author_must_belong_to_account',
    'vault_must_belong_to_account',
    'author_must_be_vault_editor',
    'contact_must_belong_to_vault',
];
```

BaseService 按 `$permissionDependencies` 中定义的依赖拓扑顺序逐项校验：

| 权限约束 | 校验逻辑 | 失败时抛出异常 |
|---|---|---|
| `author_must_belong_to_account` | `User::where('account_id', $accountId)->findOrFail($authorId)` | `ModelNotFoundException` |
| `vault_must_belong_to_account` | `Vault::where('account_id', $accountId)->findOrFail($vaultId)` | `ModelNotFoundException` |
| `author_must_be_vault_editor` | 检查用户在 vault 中的 pivot permission ≤ `Vault::PERMISSION_EDIT` | `NotEnoughPermissionException` |
| `contact_must_belong_to_vault` | `$this->vault->contacts()->findOrFail($contactId)` + 二次验证 `contact.vault_id === vault.id` | `ModelNotFoundException` |

**关键点**：`contact_must_belong_to_vault` 依赖前两项先通过（因为需要先加载 `$this->vault` 和 `$this->author`）。校验成功后，`BaseService` 会将 `$this->contact`、`$this->author`、`$this->vault` 三个模型实例挂载到服务对象上，供后续逻辑直接使用。

### 写入目标

校验通过后执行 `create()` 方法：

```php
$this->goal = Goal::create([
    'contact_id' => $this->contact->id,
    'author_id'  => $this->author->id,
    'name'       => $this->data['name'],
    'active'     => true,
]);
```

新创建的 Goal 默认 `active = true`。随后创建一条 `ContactFeedItem`（action = `ACTION_GOAL_CREATED`），并通过 morphOne 关联到该 Goal。最后将联系人的 `last_updated_at` 更新为当前时间。

### 数据库表结构

根据 [migration](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/database/migrations/2022_06_02_011219_create_goals_table.php)：

- **goals 表**：`id`, `contact_id`（外键，级联删除）, `name`, `active`（默认 false）, `timestamps`
- **streaks 表**：`id`, `goal_id`（外键，级联删除）, `happened_at`（datetime）, `timestamps`

---

## 二、ContactModuleGoalController 与 ContactModuleStreakController 的分工

### 路由定义

根据 [web.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/routes/web.php) 中的路由注册：

| HTTP 方法 | URI | 控制器 | 路由名 |
|---|---|---|---|
| POST | `/vaults/{vault}/contacts/{contact}/goals` | ContactModuleGoalController@store | `contact.goal.store` |
| PUT | `/vaults/{vault}/contacts/{contact}/goals/{goal}/streaks` | ContactModuleStreakController@update | `contact.goal.streak.update` |

### ContactModuleGoalController

[ContactModuleGoalController.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/ManageGoals/Web/Controllers/ContactModuleGoalController.php)

**职责**：处理目标的**创建**。

- 从 `Auth` 获取当前用户的 `account_id` 和 `author_id`
- 从请求获取 `vault_id`、`contact_id`、`name`
- 调用 `CreateGoal` 服务完成创建
- 创建后重新查询 Contact，调用 `ModuleGoalsViewHelper::dto($contact, $goal)` 构建响应
- 返回 HTTP 201

### ContactModuleStreakController

[ContactModuleStreakController.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/ManageGoals/Web/Controllers/ContactModuleStreakController.php)

**职责**：处理连续打卡记录的**切换**。

- 从 `Auth` 获取当前用户信息
- 从请求获取 `vault_id`、`contact_id`、`goal_id`、`happened_at`
- 调用 `ToggleStreak` 服务完成切换
- 切换后重新查询 Contact 和 Goal，调用 `ModuleGoalsViewHelper::dto($contact, $goal)` 构建响应
- 返回 HTTP 200

### 分工对比

| 维度 | ContactModuleGoalController | ContactModuleStreakController |
|---|---|---|
| 操作对象 | Goal（目标本身） | Streak（打卡记录） |
| HTTP 方法 | POST | PUT |
| 核心服务 | CreateGoal | ToggleStreak |
| 响应状态码 | 201 | 200 |
| URL 参数 | 不需要 goalId | 需要 goalId |
| 额外请求参数 | `name` | `happened_at`（日期字符串 Y-m-d） |

此外还有一个 [ContactGoalController](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/ManageGoals/Web/Controllers/ContactGoalController.php)，负责目标的 show/update/destroy 操作，与 Module 控制器共同构成完整的 CRUD 体系。Module 控制器专注于模块化视图内的轻量操作（创建 + streak 切换），而 ContactGoalController 则处理完整页面级的展示和编辑。

---

## 三、ToggleStreak：按日期创建或删除 Streak 并更新联系人

### 源码位置

[ToggleStreak.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/ManageGoals/Services/ToggleStreak.php)

### 执行流程

```
execute()
  ├── validate()
  │     ├── validateRules(data)          // 校验输入参数
  │     └── contact->Goals()->findOrFail(goal_id)  // 校验 goal 归属 contact
  ├── 查询当日 streak
  │     └── goal->streaks()->whereDate('happened_at', date)->first()
  ├── 判断分支
  │     ├── 存在 → destroyStreak(entry)  // 删除
  │     └── 不存在 → createStreak()      // 创建
  └── contact->last_updated_at = now(); contact->save()
```

### 验证逻辑详解

ToggleStreak 声明了与 CreateGoal 相同的四项权限约束，外加一步额外的归属校验：

```php
$this->goal = $this->contact->Goals()
    ->findOrFail($this->data['goal_id']);
```

这行代码通过 `contact->Goals()` 关系查询（即 `hasMany(Goal::class)`），确保 `goal_id` 对应的 Goal 确实属于当前 contact。如果 Goal 属于其他联系人，`findOrFail` 会抛出 `ModelNotFoundException`。注意代码中使用的是大写的 `Goals()` 方法名，实际调用的是 Contact 模型中定义的 `goals()` HasMany 关系。

### 切换逻辑

核心判断：

```php
$entry = $this->goal->streaks()
    ->whereDate('happened_at', $this->data['happened_at'])
    ->first();

if ($entry) {
    $this->destroyStreak($entry);   // 已有记录 → 删除
} else {
    $this->createStreak();          // 无记录 → 创建
}
```

- **创建**：向 `streaks` 表写入 `goal_id` + `happened_at`
- **删除**：直接调用 `$streak->delete()` 删除已有记录
- 无论创建还是删除，最后都会将 `contact.last_updated_at` 更新为当前时间

### 单元测试验证

[ToggleStreakTest.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/tests/Unit/Domains/Contact/ManageGoals/Services/ToggleStreakTest.php) 中的 `executeService` 方法明确验证了 toggle 行为：

```php
// 第一次调用 → 创建 streak
(new ToggleStreak)->execute($request);
$this->assertDatabaseHas('streaks', [...]);

// 第二次调用 → 删除 streak
(new ToggleStreak)->execute($request);
$this->assertDatabaseMissing('streaks', [...]);
```

---

## 四、ModuleGoalsViewHelper：构建最近 7 天的状态

### 源码位置

[ModuleGoalsViewHelper.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/ManageGoals/Web/ViewHelpers/ModuleGoalsViewHelper.php)

### data() 方法

`data(Contact $contact)` 方法构建整个模块的视图数据：

1. 查询联系人的所有 Goal
2. 按活跃状态分组：`activeGoals` 和 `inactiveGoals`
3. 对每个活跃 Goal 调用 `dto()` 构建 DTO
4. 返回包含 `active_goals`、`inactive_goals_count`、`url` 的数组

### dto() 方法

`dto(Contact $contact, Goal $goal)` 构建单个 Goal 的数据传输对象，包含：

| 字段 | 来源 |
|---|---|
| `id` | Goal 主键 |
| `name` | Goal 名称 |
| `active` | Goal 活跃状态 |
| `streaks_statistics` | `GoalHelper::getStreakData($goal)` 的返回值 |
| `last_7_days` | `self::getLast7Days($goal)` 的返回值 |
| `url` | 各操作路由的 URL 集合（show/update/streak_update/destroy） |

### getLast7Days() 方法详解

这是构建 7 天状态视图的核心方法：

```php
private static function getLast7Days(Goal $goal): array
{
    // 1. 查询最近 7 天内的 streaks（一次性加载）
    $streaks = $goal->streaks()
        ->whereDate('happened_at', '<=', Carbon::now())
        ->whereDate('happened_at', '>=', Carbon::now()->copy()->subDays(7))
        ->get();

    // 2. 生成 7 天的日期列表（从今天往前推 7 天）
    $last7DaysCollection = collect();
    for ($i = 0; $i < 7; $i++) {
        $date = Carbon::now()->subDays($i);

        // 3. 在已加载的 streaks 中匹配当前日期
        $streak = $streaks->first(function ($streak) use ($date) {
            return $streak->happened_at->format('Y-m-d') === $date->format('Y-m-d');
        });

        $last7DaysCollection->push([
            'id'           => $i,
            'day'          => DateHelper::formatShortDay($date),
            'day_number'   => $date->format('d'),
            'happened_at'  => $date->format('Y-m-d'),
            'active'       => $streak ? true : false,
        ]);
    }

    // 4. 按 id 降序排列（即最旧的日期排在前面）
    return $last7DaysCollection->sortByDesc('id')->values()->all();
}
```

**执行步骤**：

1. **预加载 streaks**：一次性从数据库查询该 Goal 在 `now()-7天` 到 `now()` 范围内的所有 streak 记录，避免 N+1 查询
2. **构建 7 天日期序列**：循环 i=0（今天）到 i=6（6 天前），生成每天的日期对象
3. **匹配 streak**：在预加载的集合中，用 `Y-m-d` 格式比对每条 streak 的 `happened_at` 与当前循环日期。匹配到则 `active = true`，否则 `active = false`
4. **排序输出**：按 `id`（即偏移天数）降序排列，最终结果从最旧日期排到最新日期

**示例输出**（假设今天为 2026-05-28，且在 5/26 和 5/28 有 streak）：

```json
[
  { "id": 6, "day": "Thu", "day_number": "22", "happened_at": "2026-05-22", "active": false },
  { "id": 5, "day": "Fri", "day_number": "23", "happened_at": "2026-05-23", "active": false },
  { "id": 4, "day": "Sat", "day_number": "24", "happened_at": "2026-05-24", "active": false },
  { "id": 3, "day": "Sun", "day_number": "25", "happened_at": "2026-05-25", "active": false },
  { "id": 2, "day": "Mon", "day_number": "26", "happened_at": "2026-05-26", "active": true  },
  { "id": 1, "day": "Tue", "day_number": "27", "happened_at": "2026-05-27", "active": false },
  { "id": 0, "day": "Wed", "day_number": "28", "happened_at": "2026-05-28", "active": true  }
]
```

### GoalHelper::getStreakData 辅助统计

[GoalHelper.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Helpers/GoalHelper.php) 提供连续打卡统计：

- 按 `happened_at` 排序获取所有 streak 日期
- 遍历相邻两天，如果连续则 `currentStreak++`，否则归零
- 记录 `maxStreak`（历史最长连续天数）
- 如果最后一条 streak 不是今天，`currentStreak` 归零
- 返回 `max_streak` 和 `current_streak`

---

## 五、同一天重复点击的数据库状态变化

### 场景分析

假设用户对同一个 Goal，在同一天（如 `2026-05-28`）连续多次点击 streak 切换按钮，每次请求都会走 `ContactModuleStreakController::update` → `ToggleStreak::execute` 的完整流程。

### 逐次状态变化

| 点击次数 | streaks 表状态 | contacts 表变化 | 前端显示 |
|---|---|---|---|
| 第 1 次 | 无记录 → **创建**一条 `(goal_id, 2026-05-28)` | `last_updated_at` 更新 | 该天 `active: true` |
| 第 2 次 | 找到记录 → **删除**该条 | `last_updated_at` 更新 | 该天 `active: false` |
| 第 3 次 | 无记录 → **创建**一条 `(goal_id, 2026-05-28)` | `last_updated_at` 更新 | 该天 `active: true` |
| 第 N 次 | 奇数次创建，偶数次删除 | 每次都更新 `last_updated_at` | 奇数 true / 偶数 false |

### 本质：幂等的 Toggle 操作

ToggleStreak 是典型的**幂等切换（Toggle）**模式——同一个日期的记录要么存在要么不存在，不存在就创建，存在就删除，形成 0↔1 的翻转。

### contacts 表的副作用

每次 toggle 操作（无论创建还是删除）都会更新 `contact.last_updated_at`。这意味着即使最终数据库状态与点击前完全一致（如点击了 2 次回到原点），`last_updated_at` 也会被更新两次，导致联系人出现在"最近更新"列表中的排序发生变化。

### 潜在的并发问题

在 `streaks` 表的 migration 中，`(goal_id, happened_at)` 组合**没有唯一索引约束**。在正常单线程操作中不会有问题（因为 `whereDate->first()` 只匹配一条），但在并发场景下：

1. 请求 A 查询无记录，准备创建
2. 请求 B 查询也无记录，准备创建
3. 两个请求都执行创建，导致同一天出现**两条** streak 记录
4. 下一次 toggle 只会 `first()` 删除其中一条，另一条残留

这意味着在极端并发下，可能出现同一 Goal 同一日期存在多条 Streak 记录的数据异常。不过 `whereDate('happened_at', date)->first()` 只取第一条，所以后续的删除操作每次只会移除一条，需要多次 toggle 才能清空所有重复记录。

---

## 六、整体架构概览

```
前端 Goals.vue 组件
    │
    ├── POST /vaults/{v}/contacts/{c}/goals              → ContactModuleGoalController@store
    │                                                        └── CreateGoal (创建目标)
    │
    └── PUT /vaults/{v}/contacts/{c}/goals/{g}/streaks    → ContactModuleStreakController@update
                                                                 └── ToggleStreak (切换打卡)
                                                                 
两个控制器共用 ModuleGoalsViewHelper::dto() 构建响应:
    ┌── GoalHelper::getStreakData()     → streak 统计数据
    └── ModuleGoalsViewHelper::getLast7Days() → 最近 7 天打卡状态
```

### 数据流向

```
Request → Controller → Service(BaseService 权限校验) → 数据库写入
                     → Controller → ModuleGoalsViewHelper → JSON Response
```

Service 层负责所有的权限校验和业务逻辑，Controller 层仅负责参数组装和响应构建，ViewHelper 负责数据的展示格式化。这种分层设计使得权限逻辑集中在 BaseService 中统一管理，业务逻辑在各 Service 中独立实现，视图构建则与业务逻辑完全解耦。
