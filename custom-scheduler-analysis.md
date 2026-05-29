# Monica 自定义调度系统设计分析

## 一、为什么实现自定义 Schedule、CronEvent 和 Cron 模型

### 1.1 Laravel 默认调度器的局限性

Laravel 默认的调度器（`Illuminate\Console\Scheduling\Schedule`）主要存在以下局限性：

- **无持久化状态**：默认调度器仅在内存中运行，任务的上次运行时间不会持久化存储。如果服务器重启或调度进程中断，任务执行历史会丢失。
- **多实例重复执行风险**：默认调度器本身不提供跨实例的协调机制。在多实例部署下，如果每个实例都运行 `schedule:run`，同一任务可能会被多次执行。
- **审计追踪能力弱**：默认调度器没有内置的任务执行历史记录，难以追踪任务何时运行、是否成功执行。
- **自定义频率计算**：默认调度器的频率计算基于 CRON 表达式，而项目需要更灵活的基于分钟/天数的频率计算，并能与数据库持久化状态结合。

### 1.2 自定义调度系统的设计目标

通过自定义 [Schedule](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Console/Scheduling/Schedule.php)、[CronEvent](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Console/Scheduling/CronEvent.php) 和 [Cron](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Models/Instance/Cron.php) 模型，项目实现了：

- **持久化状态管理**：每个任务的上次运行时间存储在数据库中，重启后可恢复状态。
- **基于数据库的去重**：通过 `last_run_at` 字段判断任务是否需要运行。
- **可审计性**：可追踪每个任务的执行历史。
- **与 Laravel 调度器兼容**：自定义 Schedule 作为门面，底层仍使用 Laravel 的调度器，只是增加了 `when()` 条件判断。

---

## 二、routes/console.php 中注册的任务

[routes/console.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/routes/console.php) 中注册了以下 8 个定时任务：

| 任务命令/类 | 频率 | 说明 |
|------------|------|------|
| `model:prune` | daily | 清理过期的模型数据 |
| `queue:prune-batches` | daily | 清理 48 小时前完成的批次、72 小时未完成/取消的批次 |
| `queue:prune-failed` | daily | 清理 48 小时前的失败队列任务 |
| `telescope:prune` | daily | 清理 Telescope 监控数据（仅启用时） |
| `UpdateAddressBooks::class` | hourly | 每小时更新地址簿订阅同步 |
| `ProcessScheduledContactReminders::class` | every 1 minute | 每分钟处理联系人提醒通知 |
| `CleanSyncToken::class` | daily | 每日清理过期的 DAV 同步令牌 |
| `CleanLogs::class` | daily | 每日清理 15 天前的日志 |

这些任务覆盖了：
- **系统维护**：模型清理、队列清理、日志清理
- **业务功能**：联系人提醒、地址簿同步、DAV 同步令牌管理
- **监控运维**：Telescope 数据清理

---

## 三、Schedule 如何创建或查询 Cron 记录

### 3.1 Schedule 类的工作原理

[Schedule](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Console/Scheduling/Schedule.php) 是一个门面类，通过 `__callStatic` 魔术方法处理静态调用：

```php
public static function __callStatic(string $method, array $args): void
{
    $command = array_shift($args);
    $frequency = array_shift($args);
    
    ScheduleFacade::$method($command)->when(function () use ($command, $frequency, $args) {
        $event = CronEvent::command($command);
        if ($frequency !== null) {
            $event = $event->{$frequency}(...$args);
        }
        return $event->isDue();
    });
}
```

### 3.2 关键流程分析

1. **参数解析**：从 `$args` 中提取 `command`（命令或任务类名）和 `frequency`（频率）。

2. **注册到 Laravel 调度器**：调用 `ScheduleFacade::$method($command)` 将任务注册到 Laravel 默认调度器。

3. **添加条件判断**：通过 `when()` 方法添加一个回调，该回调决定任务是否实际执行。

4. **创建 CronEvent 实例**：在回调中调用 `CronEvent::command($command)`，该方法内部会：

   ```php
   public static function command(string $command): self
   {
       $cron = Cron::firstOrCreate(['command' => $command]);
       return new self($cron);
   }
   ```

5. **数据库操作**：使用 `Cron::firstOrCreate(['command' => $command])` 查询或创建 Cron 记录：
   - 如果该命令已有记录，则查询返回现有记录
   - 如果是新命令，则在数据库中创建一条新记录，`last_run_at` 默认为 `null`

### 3.3 Cron 数据库表结构

根据 [2019_05_05_194746_create_crons.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/database/migrations/2019_05_05_194746_create_crons.php)，`crons` 表包含：

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | increments | 主键 |
| `command` | string (unique) | 命令名称，唯一索引 |
| `last_run_at` | timestamp (nullable) | 上次运行时间 |
| `created_at` | timestamp | 创建时间 |
| `updated_at` | timestamp | 更新时间 |

---

## 四、CronEvent 如何决定任务是否到期、运行、记录状态和避免重复

### 4.1 核心方法：isDue()

[CronEvent](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Console/Scheduling/CronEvent.php) 的 `isDue()` 方法是整个调度系统的核心：

```php
public function isDue(): bool
{
    $now = now();

    if ($this->cron->last_run_at !== null) {
        $t = $this->cron->last_run_at;

        if ($this->minutes !== 0) {
            $nextRun = Carbon::create($t->year, $t->month, $t->day, $t->hour, (int) floor($t->minute / $this->minutes) * $this->minutes, 0)
                ->addMinutes($this->minutes);
        } elseif ($this->days !== 0) {
            $nextRun = Carbon::create($t->year, $t->month, (int) floor($t->day / $this->days) * $this->days, 0, 0, 0)
                ->addDays($this->days);
        }

        if (! isset($nextRun) || $nextRun > $now) {
            return false;
        }
    }

    $this->cron->update(['last_run_at' => $now]);
    return true;
}
```

### 4.2 频率设置方法

CronEvent 提供了以下方法设置运行频率：

| 方法 | minutes | days | 说明 |
|------|---------|------|------|
| `hourly()` | 60 | 0 | 每小时运行一次 |
| `daily()` | 0 | 1 | 每天运行一次 |
| `weekly()` | 0 | 7 | 每周运行一次 |
| `minutes($n)` | $n | 0 | 每 $n 分钟运行一次 |

### 4.3 到期判断逻辑

#### 场景 1：首次运行（last_run_at 为 null）

- 跳过时间计算，直接更新 `last_run_at` 为当前时间
- 返回 `true`，任务执行

#### 场景 2：按分钟频率（如每分钟、每小时）

以每 15 分钟为例：
1. 上次运行时间：`2024-01-01 10:07:00`
2. 计算对齐后的分钟：`floor(7 / 15) * 15 = 0`
3. 对齐时间：`2024-01-01 10:00:00`
4. 下次运行时间：`2024-01-01 10:00:00 + 15分钟 = 2024-01-01 10:15:00`
5. 如果当前时间 >= 下次运行时间，则返回 `true`

#### 场景 3：按天数频率（如每天、每周）

以每天为例：
1. 上次运行时间：`2024-01-15 14:30:00`
2. 计算对齐后的天数：`floor(15 / 1) * 1 = 15`
3. 对齐时间：`2024-01-15 00:00:00`
4. 下次运行时间：`2024-01-15 00:00:00 + 1天 = 2024-01-16 00:00:00`
5. 如果当前时间 >= 下次运行时间，则返回 `true`

### 4.4 状态记录和避免重复机制

1. **更新时机**：在返回 `true` 之前，立即执行 `$this->cron->update(['last_run_at' => $now])` 更新数据库。
2. **原子性尝试**：更新操作是针对单条记录的 UPDATE，数据库会保证行级锁。
3. **避免重复**：一旦 `last_run_at` 被更新，后续的 `isDue()` 检查会计算出新的 `nextRun`，在下次到期前都会返回 `false`。

### 4.5 测试验证

[CronEventTest.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/tests/Unit/Commands/Scheduling/CronEventTest.php) 中的测试用例验证了以下场景：
- 首次调用 `isDue()` 返回 `false`，但会更新 `last_run_at`
- 时间推进后再次调用返回 `true`
- 每小时、每天频率的正确性验证

---

## 五、CleanLogs 与 CleanSyncToken 等维护任务的调度方式

### 5.1 CleanLogs 任务

[CleanLogs](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Logging/CleanLogs.php) 是日志清理任务：

```php
class CleanLogs extends QueuableService implements ServiceInterface
{
    public function execute(array $data): void
    {
        Log::where('created_at', '<', Carbon::now()->subDays(15))->delete();
    }
}
```

- **调度方式**：`Schedule::job(CleanLogs::class, 'daily')`
- **执行频率**：每日一次
- **功能**：删除 15 天前的日志记录
- **执行方式**：继承 `QueuableService`，通过队列执行

### 5.2 CleanSyncToken 任务

[CleanSyncToken](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/Dav/Jobs/CleanSyncToken.php) 是 DAV 同步令牌清理任务：

```php
class CleanSyncToken extends QueuableService implements ServiceInterface
{
    public function execute(array $data): void
    {
        $this->timefix = now()->addDays(-1 * intval(config('dav.sync_token_keep_days')));
        
        DB::table('sync_tokens')
            ->orderBy('user_id')
            ->groupBy('user_id', 'name')
            ->select(DB::raw('user_id, name, max(timestamp) as timestamp'))
            ->chunk(200, function ($tokens) {
                foreach ($tokens as $token) {
                    $this->handleUserToken($token->user_id, $token->name, $token->timestamp);
                }
            });
    }
    
    private function handleUserToken(string $userId, string $tokenName, string $timestamp): void
    {
        $tokens = SyncToken::where([
            ['user_id', $userId],
            ['name', $tokenName],
            ['timestamp', '<', Carbon::parse($timestamp)],
            ['timestamp', '<', $this->timefix],
        ])
            ->orderByDesc('timestamp')
            ->get();
            
        foreach ($tokens as $token) {
            TokenDeleteEvent::dispatch($token);
            $token->delete();
        }
    }
}
```

- **调度方式**：`Schedule::job(CleanSyncToken::class, 'daily')`
- **执行频率**：每日一次
- **功能**：
  1. 按用户和令牌名称分组，找出每组最大的 timestamp
  2. 删除每个用户每组中除最新以外的令牌
  3. 只删除超过保留天数（由 `dav.sync_token_keep_days` 配置）的令牌
  4. 删除前触发 `TokenDeleteEvent` 事件
- **分块处理**：使用 `chunk(200)` 分批处理，避免内存溢出

### 5.3 其他关键任务

#### ProcessScheduledContactReminders

[ProcessScheduledContactReminders](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/ManageReminders/Jobs/ProcessScheduledContactReminders.php)：

- **调度方式**：`Schedule::job(ProcessScheduledContactReminders::class, 'minutes', 1)`
- **执行频率**：每分钟一次
- **功能**：查询所有到期的联系人提醒，发送通知（邮件、Telegram 等），然后重新调度下次提醒
- **容错处理**：单个通知失败不会影响其他提醒，会记录错误日志并增加失败计数，超过阈值后自动禁用该通知渠道

#### UpdateAddressBooks

[UpdateAddressBooks](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/DavClient/Jobs/UpdateAddressBooks.php)：

- **调度方式**：`Schedule::job(UpdateAddressBooks::class, 'hourly')`
- **执行频率**：每小时一次
- **功能**：检查所有活跃的地址簿订阅，根据订阅的频率（每个订阅有自己的 `frequency` 字段）决定是否需要同步
- **二级调度**：任务本身每小时运行，但内部会检查每个订阅的 `last_synchronized_at`，只调度真正到期的订阅进行同步

---

## 六、多实例部署下的锁和幂等性考虑

### 6.1 当前设计存在的问题

当前自定义调度系统在多实例部署下存在**竞态条件**风险。问题出在 [CronEvent::isDue()](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Console/Scheduling/CronEvent.php#L104-L127) 方法中：

```php
public function isDue(): bool
{
    $now = now();
    
    if ($this->cron->last_run_at !== null) {
        // ... 计算 nextRun
        if (! isset($nextRun) || $nextRun > $now) {
            return false;
        }
    }
    
    // 问题：这里存在时间窗口
    $this->cron->update(['last_run_at' => $now]);
    return true;
}
```

#### 竞态条件场景

假设有两个实例 A 和 B，在同一时刻运行 `schedule:run`：

| 时间 | 实例 A | 实例 B |
|------|--------|--------|
| T1 | 读取 `last_run_at = 10:00` | 读取 `last_run_at = 10:00` |
| T2 | 计算 `nextRun = 10:05`，当前 10:05，到期 | 计算 `nextRun = 10:05`，当前 10:05，到期 |
| T3 | 更新 `last_run_at = 10:05` | （等待行锁） |
| T4 | 开始执行任务 | 获取锁，更新 `last_run_at = 10:05` |
| T5 | | 开始执行任务 |

**结果**：同一个任务被实例 A 和 B 各执行一次。

### 6.2 需要增加的锁机制

#### 方案 1：使用数据库乐观锁

修改 `Cron` 模型和 `isDue()` 方法，增加版本号或使用条件更新：

```php
// 修改 update 为带条件的更新
$updated = Cron::where('id', $this->cron->id)
    ->where('last_run_at', $this->cron->last_run_at)  // 关键：确保读取的值未被修改
    ->update(['last_run_at' => $now]);

if ($updated === 0) {
    return false;  // 其他实例已更新，本次不执行
}
return true;
```

#### 方案 2：使用 Laravel 缓存锁

利用 Laravel 的 `Cache::lock()` 机制：

```php
public function isDue(): bool
{
    $lock = Cache::lock("cron:{$this->cron->command}", 60);
    
    if (! $lock->get()) {
        return false;
    }
    
    try {
        // ... 原有逻辑
    } finally {
        $lock->release();
    }
}
```

#### 方案 3：使用 Laravel 内置的 `onOneServer()`

Laravel 调度器本身提供 `onOneServer()` 方法，可以确保任务只在一台服务器上运行：

```php
ScheduleFacade::$method($command)
    ->onOneServer()
    ->when(...);
```

但需要注意：
- 需要配置缓存驱动为支持锁的驱动（Redis、Memcached 等）
- 该方法是在任务级别防止重复，而不是在 `isDue()` 判断级别

#### 方案 4：使用 `withoutOverlapping()`

防止同一个任务的上一次执行还未结束时，本次又开始执行：

```php
ScheduleFacade::$method($command)
    ->withoutOverlapping()
    ->when(...);
```

### 6.3 任务幂等性分析

即使有了锁机制，每个任务仍应设计为幂等（多次执行产生相同结果）。以下是各任务的幂等性分析：

| 任务 | 当前幂等性 | 风险 | 改进建议 |
|------|-----------|------|---------|
| `model:prune` | 高 | 低 | 基于时间删除，重复执行只是 DELETE 0 条 |
| `queue:prune-batches` | 高 | 低 | 同上 |
| `queue:prune-failed` | 高 | 低 | 同上 |
| `telescope:prune` | 高 | 低 | 同上 |
| `UpdateAddressBooks` | 中 | 中 | 可能触发多次同步，但同步本身有 `last_synchronized_at` 检查 |
| `ProcessScheduledContactReminders` | **低** | **高** | 发送重复通知给用户。需要在发送前再次检查 `triggered_at` 字段 |
| `CleanSyncToken` | 高 | 低 | 删除操作天然幂等 |
| `CleanLogs` | 高 | 低 | 删除操作天然幂等 |

#### ProcessScheduledContactReminders 幂等性改进

当前代码在 [ProcessScheduledContactReminders::handle()](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/ManageReminders/Jobs/ProcessScheduledContactReminders.php#L30-L79) 中查询时未排除已触发的记录：

```php
$scheduledContactReminders = DB::table('contact_reminder_scheduled')
    ->where('scheduled_at', '<=', $currentDate)
    ->get();
```

应增加 `triggered_at is null` 条件：

```php
$scheduledContactReminders = DB::table('contact_reminder_scheduled')
    ->where('scheduled_at', '<=', $currentDate)
    ->whereNull('triggered_at')  // 增加此行
    ->get();
```

### 6.4 调度入口配置

从 [docker-compose.yml](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/docker-compose.yml#L35) 和 Dockerfile 可以看出，调度通过系统 cron 每分钟执行一次：

```
* * * * * php /var/www/html/artisan schedule:run -v
```

在多实例部署下：
- 如果每个实例都配置此 cron，则需要上述的锁机制
- 或者只在一个特定实例上运行 cron，但这会引入单点故障
- 推荐方案：每个实例都运行 cron，但通过 `onOneServer()` 或数据库锁确保只有一个实例执行任务

### 6.5 总结改进建议

1. **立即修复**：在 `isDue()` 方法中使用条件更新（乐观锁），解决竞态条件
2. **增加幂等性**：为 `ProcessScheduledContactReminders` 增加 `triggered_at is null` 查询条件
3. **利用 Laravel 特性**：在 Schedule 注册时增加 `->onOneServer()->withoutOverlapping()`
4. **监控告警**：增加任务执行时间过长或失败的监控
5. **考虑使用独占锁**：对于关键任务，使用 `Cache::lock()` 确保同一时间只有一个实例在执行

---

## 七、总结

Monica 项目自定义调度系统的设计体现了对持久化状态和多实例部署的考量，通过 `Cron` 模型存储任务执行状态，`CronEvent` 计算到期时间，`Schedule` 门面与 Laravel 调度器无缝集成。然而，当前设计在多实例环境下存在竞态条件问题，需要通过乐观锁、缓存锁或 Laravel 内置的 `onOneServer()` 等机制来增强，同时关键任务需要确保幂等性，以避免重复执行带来的副作用。
