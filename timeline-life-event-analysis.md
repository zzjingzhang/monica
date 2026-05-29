# Timeline Event & Life Event 链路分析报告

## 1. 架构概述

TimelineEvent 与 LifeEvent 是一对多的关系：
- 一个 TimelineEvent（时间线事件）可以包含多个 LifeEvent（生命事件）
- TimelineEvent 代表一个大的事件周期（如"旅行"），LifeEvent 代表具体的子事件（如"驾车100公里"、"吃披萨"）
- 两者都有 `collapsed` 字段控制UI可见性

---

## 2. 创建链路分析

### 2.1 控制器层：ContactModuleTimelineEventController::store

**文件位置**：[ContactModuleTimelineEventController.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/ManageLifeEvents/Web/Controllers/ContactModuleTimelineEventController.php#L33-L84)

**参数传递流程**：

1. **创建 TimelineEvent 的数据**（第35-41行）：
```php
$data = [
    'account_id' => Auth::user()->account_id,
    'author_id' => Auth::id(),
    'vault_id' => $vaultId,
    'label' => $request->input('label'),           // ✅ 正确获取 label
    'started_at' => $request->input('started_at'),
];
$timelineEvent = (new CreateTimelineEvent)->execute($data);
```

2. **当前联系人加入 participant_ids**（第48-52行）：
```php
$participants = collect($request->input('participants'))
    ->push(['id' => $contactId])       // 将当前联系人推入参与者数组
    ->unique('id')                     // 去重，避免重复添加
    ->pluck('id')                      // 只提取 id 字段
    ->toArray();
```

3. **创建 LifeEvent 的数据**（第56-76行）：
```php
$data = [
    'account_id' => Auth::user()->account_id,
    'author_id' => Auth::id(),
    'vault_id' => $vaultId,
    'label' => $request->input('label'),           // ⚠️  传递了 label 但 CreateLifeEvent 不处理
    'timeline_event_id' => $timelineEvent->id,
    'life_event_type_id' => $request->input('lifeEventTypeId'),
    'summary' => $request->input('summary'),
    // ... 其他字段
    'participant_ids' => $participants,
];
$lifeEvent = (new CreateLifeEvent)->execute($data);
```

### 2.2 CreateTimelineEvent 服务

**文件位置**：[CreateTimelineEvent.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/ManageLifeEvents/Services/CreateTimelineEvent.php)

**验证规则**（第18-27行）：
```php
public function rules(): array
{
    return [
        'account_id' => 'required|uuid|exists:accounts,id',
        'vault_id' => 'required|uuid|exists:vaults,id',
        'author_id' => 'required|uuid|exists:users,id',
        'label' => 'nullable|string|max:255',      // ✅ 规则定义了 label
        'started_at' => 'required|date|date_format:Y-m-d',
    ];
}
```

**⚠️ 关键 Bug - label 字段丢失**（第60-67行）：
```php
private function store(): void
{
    $this->timelineEvent = TimelineEvent::create([
        'vault_id' => $this->data['vault_id'],
        'label' => $this->valueOrNull($this->data, 'summary'),  // ❌ BUG: 使用了 'summary' 而非 'label'
        'started_at' => $this->data['started_at'],
    ]);
}
```

**问题分析**：
- 控制器传递的是 `label` 字段
- 验证规则也期望 `label` 字段
- 但写入数据库时错误地读取了 `summary` 字段（该字段在 CreateTimelineEvent 的 data 中不存在）
- `valueOrNull()` 会因 `summary` 不存在而返回 `null`，导致 label 始终为空

### 2.3 CreateLifeEvent 服务

**文件位置**：[CreateLifeEvent.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/ManageLifeEvents/Services/CreateLifeEvent.php)

**验证逻辑**（第77-103行）：

1. **事件类型分类验证**（第81-87行）：
```php
$lifeEventType = LifeEventType::findOrFail($this->data['life_event_type_id']);

$this->timelineEvent = $this->vault->timelineEvents()
    ->findOrFail($this->data['timeline_event_id']);

// 验证该事件类型所属的分类属于当前 vault
$this->vault->lifeEventCategories()
    ->findOrFail($lifeEventType->lifeEventCategory->id);
```

2. **支付联系人验证**（第89-92行）：
```php
if (! is_null($this->data['paid_by_contact_id'])) {
    $this->vault->contacts()
        ->findOrFail($this->data['paid_by_contact_id']);  // 验证支付人属于当前 vault
}
```

3. **货币验证**（第94-97行）：
```php
if (! is_null($this->data['currency_id'])) {
    $this->account()->currencies()
        ->findOrFail($this->data['currency_id']);  // 验证货币属于当前 account
}
```

4. **参与者验证**（第101-102行）：
```php
$this->partipantsCollection = collect($this->data['participant_ids'])
    ->map(fn (string $participantId): Contact => $this->vault->contacts()->findOrFail($participantId));
```
> 每个参与者都必须属于当前 vault，否则抛出 `ModelNotFoundException`

**关联参与者**（第113-119行）：
```php
private function associateParticipants(): void
{
    foreach ($this->partipantsCollection as $participant) {
        $participant->lifeEvents()->attach($this->lifeEvent->id);                     // life_event_participants 表
        $participant->timelineEvents()->syncWithoutDetaching($this->timelineEvent->id); // timeline_event_participants 表
    }
}
```

**⚠️ 注意**：CreateLifeEvent 的 rules 中没有 `label` 字段，控制器传递的 `label` 会被忽略。

---

## 3. 更新链路分析

### 3.1 控制器层：ContactModuleLifeEventController::edit

**文件位置**：[ContactModuleLifeEventController.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/ManageLifeEvents/Web/Controllers/ContactModuleLifeEventController.php#L60-L102)

**参数传递**（第73-94行）：
```php
$data = [
    'account_id' => Auth::user()->account_id,
    'author_id' => Auth::id(),
    'vault_id' => $vaultId,
    'timeline_event_id' => $timelineEventId,
    'life_event_id' => $lifeEventId,
    'label' => $request->input('label'),           // ⚠️  传递了 label 但 UpdateLifeEvent 不处理
    'life_event_type_id' => $request->input('lifeEventTypeId'),
    // ... 其他字段
    'participant_ids' => $participants,
];
$lifeEvent = (new UpdateLifeEvent)->execute($data);
```

**参与者处理**（第65-69行）：
```php
$participants = collect($request->input('participants'))
    ->push(['id' => $contactId])       // 同样确保当前联系人在参与者中
    ->unique('id')
    ->pluck('id')
    ->toArray();
```

### 3.2 UpdateLifeEvent 服务

**文件位置**：[UpdateLifeEvent.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/ManageLifeEvents/Services/UpdateLifeEvent.php)

**执行流程**（第67-77行）：
```php
public function execute(array $data): LifeEvent
{
    $this->data = $data;
    $this->validate();
    $this->update();               // 更新 LifeEvent 字段
    $this->dissociateParticipants();  // ❶ 先移除所有旧参与者
    $this->associateParticipants();   // ❷ 再关联新参与者（重建关系）
    $this->updateLastEditedDate();
    return $this->lifeEvent;
}
```

**重建参与者关系**：

1. **解除旧关联**（第135-138行）：
```php
private function dissociateParticipants(): void
{
    $this->lifeEvent->participants()->detach();  // 从 life_event_participants 表移除所有关联
}
```

2. **建立新关联**（第140-146行）：
```php
private function associateParticipants(): void
{
    foreach ($this->partipantsCollection as $participant) {
        $participant->lifeEvents()->attach($this->lifeEvent->id);
        $participant->timelineEvents()->syncWithoutDetaching($this->timelineEvent->id);
    }
}
```

**⚠️ 问题**：UpdateLifeEvent 的 rules 中也没有 `label` 字段，控制器传递的 `label` 会被忽略。TimelineEvent 的 label 无法通过此接口更新。

### 3.3 UpdateTimelineEvent 服务

**文件位置**：[UpdateTimelineEvent.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/ManageLifeEvents/Services/UpdateTimelineEvent.php)

该服务存在但**从未被控制器调用**，路由中也没有对应的端点。其 `update()` 方法正确处理了 label：
```php
private function update(): void
{
    $this->timelineEvent->label = $this->valueOrNull($this->data, 'label');  // ✅ 正确使用 label
    $this->timelineEvent->started_at = $this->data['started_at'];
    $this->timelineEvent->save();
}
```

---

## 4. 切换显示链路分析

### 4.1 ToggleTimelineEvent

**控制器**：[ToggleTimelineEventController.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/ManageLifeEvents/Web/Controllers/ToggleTimelineEventController.php)

**服务**：[ToggleTimelineEvent.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/ManageLifeEvents/Services/ToggleTimelineEvent.php)

**改变的可见性层级**：
```php
private function update(): void
{
    $this->timelineEvent->collapsed = ! $this->timelineEvent->collapsed;  // TimelineEvent 层级
    $this->timelineEvent->save();
}
```
> 控制整个 TimelineEvent 的展开/折叠，影响其下所有 LifeEvent 的显示

### 4.2 ToggleLifeEvent

**控制器**：[ToggleLifeEventController.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/ManageLifeEvents/Web/Controllers/ToggleLifeEventController.php)

**服务**：[ToggleLifeEvent.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/ManageLifeEvents/Services/ToggleLifeEvent.php)

**改变的可见性层级**：
```php
private function update(): void
{
    $this->lifeEvent->collapsed = ! $this->lifeEvent->collapsed;  // LifeEvent 层级
    $this->lifeEvent->save();
}
```
> 仅控制单个 LifeEvent 的展开/折叠，不影响 TimelineEvent 和其他 LifeEvent

---

## 5. 删除链路分析

### 5.1 删除 TimelineEvent

**控制器**：[ContactModuleTimelineEventController::destroy](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/ManageLifeEvents/Web/Controllers/ContactModuleTimelineEventController.php#L86-L100)

**服务**：[DestroyTimelineEvent.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/ManageLifeEvents/Services/DestroyTimelineEvent.php)

```php
public function execute(array $data): void
{
    $this->data = $data;
    $this->validate();
    $this->timelineEvent->delete();  // 直接删除，级联删除其下的 LifeEvent（数据库外键约束）
}
```

### 5.2 删除 LifeEvent

**控制器**：[ContactModuleLifeEventController::destroy](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/ManageLifeEvents/Web/Controllers/ContactModuleLifeEventController.php#L104-L119)

**服务**：[DestroyLifeEvent.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/ManageLifeEvents/Services/DestroyLifeEvent.php)

```php
public function execute(array $data): void
{
    $this->data = $data;
    $this->validate();
    $this->lifeEvent->delete();
    $this->deleteTimelineEvent();  // 如果是最后一个 LifeEvent，同时删除 TimelineEvent
}

private function deleteTimelineEvent(): void
{
    if ($this->timelineEvent->lifeEvents()->count() === 0) {
        $this->timelineEvent->delete();
    }
}
```

---

## 6. Label 字段丢失回归分析

### 6.1 根本原因

**文件**：[CreateTimelineEvent.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/ManageLifeEvents/Services/CreateTimelineEvent.php#L64)

**第64行代码错误**：
```php
'label' => $this->valueOrNull($this->data, 'summary'),  // ❌ 应为 'label'
```

### 6.2 回归路径

```
前端传递 label 字段
    ↓
ContactModuleTimelineEventController::store 正确接收 label
    ↓
$data['label'] = $request->input('label')  ✅ 正确
    ↓
CreateTimelineEvent::rules() 定义 'label' 规则  ✅ 正确
    ↓
CreateTimelineEvent::store() 读取 'summary'  ❌ Bug
    ↓
valueOrNull 因 'summary' 不存在返回 null
    ↓
数据库 label 字段为 null  ⚠️  数据丢失
```

### 6.3 相关问题

1. **CreateLifeEvent 忽略 label**：控制器传递了 label，但服务没有处理
2. **UpdateLifeEvent 无法更新 TimelineEvent 的 label**：没有调用 UpdateTimelineEvent
3. **UpdateTimelineEvent 服务闲置**：存在但没有路由和控制器调用

---

## 7. 修复建议

### 7.1 紧急修复 CreateTimelineEvent

修改 [CreateTimelineEvent.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/ManageLifeEvents/Services/CreateTimelineEvent.php#L64) 第64行：
```php
// 修复前
'label' => $this->valueOrNull($this->data, 'summary'),

// 修复后
'label' => $this->valueOrNull($this->data, 'label'),
```

### 7.2 完善更新链路

在 [ContactModuleLifeEventController::edit](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/ManageLifeEvents/Web/Controllers/ContactModuleLifeEventController.php#L96) 中添加对 UpdateTimelineEvent 的调用：
```php
// 在 UpdateLifeEvent 调用后添加
$timelineData = [
    'account_id' => Auth::user()->account_id,
    'author_id' => Auth::id(),
    'vault_id' => $vaultId,
    'timeline_event_id' => $timelineEventId,
    'label' => $request->input('label'),
    'started_at' => $request->input('started_at'),
];
(new UpdateTimelineEvent)->execute($timelineData);
```

### 7.3 清理冗余代码

从 [ContactModuleLifeEventController.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/ManageLifeEvents/Web/Controllers/ContactModuleLifeEventController.php) 和 [ContactModuleTimelineEventController.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/ManageLifeEvents/Web/Controllers/ContactModuleTimelineEventController.php) 的 CreateLifeEvent 数据中移除无用的 `label` 字段传递，或在 CreateLifeEvent 中添加对 TimelineEvent label 的更新逻辑。

---

## 8. 关键验证点总结

| 验证项 | CreateLifeEvent | UpdateLifeEvent | 验证层级 |
|--------|-----------------|-----------------|----------|
| 事件类型分类 | ✅ 第86-87行 | ✅ 第91-92行 | Vault 层级 |
| 支付联系人 | ✅ 第89-92行 | ✅ 第94-97行 | Vault 层级 |
| 货币 | ✅ 第94-97行 | ✅ 第99-102行 | Account 层级 |
| 参与者 | ✅ 第101-102行 | ✅ 第104-105行 | Vault 层级 |
| TimelineEvent 归属 | ✅ 第83-84行 | ✅ 第83-84行 | Vault 层级 |

| 切换操作 | 影响层级 | 改变字段 |
|----------|----------|----------|
| ToggleTimelineEvent | TimelineEvent 层 | `timeline_events.collapsed` |
| ToggleLifeEvent | LifeEvent 层 | `life_events.collapsed` |
