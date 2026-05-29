# 关系模块 SetRelationship 与 UnsetRelationship 深度分析

## 一、控制器参数传入与流向

### 1.1 控制器层入口

关系操作的控制器入口位于 [ContactRelationshipsController.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/ManageRelationships/Web/Controllers/ContactRelationshipsController.php)，主要包含两个关键方法：

#### store() 方法 - 建立关系
```php
public function store(Request $request, string $vaultId, string $contactId)
```

**核心参数流向（第67-74行）：**
```php
(new SetRelationship)->execute([
    'account_id' => Auth::user()->account_id,
    'author_id' => Auth::id(),
    'vault_id' => $vaultId,
    'relationship_type_id' => $request->input('relationship_type_id'),
    'contact_id' => $request->input('base_contact_id') === $contactId ? $contactId : $otherContactId,
    'other_contact_id' => $request->input('base_contact_id') === $contactId ? $otherContactId : $contactId,
]);
```

**关键逻辑：**
- 通过 `base_contact_id` 参数决定关系方向（谁是"from"，谁是"to"）
- 支持两种模式：选择已有联系人，或创建新联系人（`choice !== 'contact'` 时）
- 新创建的联系人可设置 `listed` 属性（第59行 `'listed' => $request->input('create_contact_entry')`）

#### update() 方法 - 解除关系
```php
public function update(Request $request, string $vaultId, string $contactId, int $relationshipId)
```

**核心参数流向（第88-95行）：**
```php
(new UnsetRelationship)->execute([
    'account_id' => Auth::user()->account_id,
    'author_id' => Auth::id(),
    'vault_id' => $vaultId,
    'relationship_type_id' => $relationship->relationship_type_id,
    'contact_id' => $relationship->contact_id,
    'other_contact_id' => $relationship->related_contact_id,
]);
```

### 1.2 数据流向总图

```
前端请求
    ↓
ContactRelationshipsController::store()
    ├─ 选择已有联系人 / 创建新联系人（CreateContact）
    ├─ 确定 base_contact_id 决定关系方向
    └─ 调用 SetRelationship::execute()
        ├─ 参数校验（rules）
        ├─ 权限校验（permissions）
        ├─ 业务逻辑校验（account、vault、group）
        └─ 写入 relationships 表

前端删除请求
    ↓
ContactRelationshipsController::update()
    ├─ 从 DB 查询原始 relationship 记录
    └─ 调用 UnsetRelationship::execute()
        ├─ 参数校验（rules）
        ├─ 权限校验（permissions）
        ├─ 业务逻辑校验
        ├─ 双向 detach
        ├─ 删除 unlisted 联系人
        └─ 更新 last_edited_date
```

---

## 二、SetRelationship 校验机制深度分析

### 2.1 服务类基本结构

[SetRelationship.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/ManageRelationships/Services/SetRelationship.php) 继承自 [BaseService.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Services/BaseService.php)，采用"规则+权限"的双层校验模式。

### 2.2 第一层：参数规则校验（rules）

```php
public function rules(): array
{
    return [
        'account_id' => 'required|uuid|exists:accounts,id',
        'vault_id' => 'required|uuid|exists:vaults,id',
        'author_id' => 'required|uuid|exists:users,id',
        'relationship_type_id' => 'required|integer|exists:relationship_types,id',
        'contact_id' => 'required|uuid|exists:contacts,id',
        'other_contact_id' => 'required|uuid|exists:contacts,id',
    ];
}
```

**校验要点：**
- 所有 ID 必须存在于对应表中
- account_id、vault_id、contact_id、other_contact_id 必须是 UUID 格式
- relationship_type_id 是整数类型

### 2.3 第二层：权限校验（permissions）

```php
public function permissions(): array
{
    return [
        'author_must_belong_to_account',
        'vault_must_belong_to_account',
        'author_must_be_vault_editor',
        'contact_must_belong_to_vault',
    ];
}
```

**权限依赖链（BaseService 第41-67行）：**

| 权限项 | 依赖项 | 校验逻辑（BaseService） |
|--------|--------|------------------------|
| `author_must_belong_to_account` | 无 | 第159-163行：验证用户属于指定账户 |
| `vault_must_belong_to_account` | 无 | 第180-184行：验证 vault 属于指定账户 |
| `author_must_be_vault_editor` | `vault_must_belong_to_account`<br>`author_must_belong_to_account` | 第190-200行：验证用户在 vault 中有 EDIT 权限 |
| `contact_must_belong_to_vault` | `vault_must_belong_to_account`<br>`author_must_belong_to_account` | 第205-215行：验证 contact 属于指定 vault |

### 2.4 第三层：业务逻辑校验

在 `execute()` 方法中还有额外的业务校验：

#### 校验1：另一端联系人属于同一 vault（第51-52行）
```php
$otherContact = $this->vault->contacts()
    ->findOrFail($data['other_contact_id']);
```

> **关键点**：`contact_must_belong_to_vault` 权限只校验了 `contact_id`，这里额外校验 `other_contact_id` 也必须属于同一 vault。

#### 校验2：relationship type 属于当前账户（第54-57行）
```php
$relationshipType = RelationshipType::findOrFail($data['relationship_type_id']);
if ($relationshipType->groupType->account_id !== $data['account_id']) {
    throw new ModelNotFoundException;
}
```

**关联模型结构：**

[RelationshipType.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Models/RelationshipType.php) 第53-56行：
```php
public function groupType(): BelongsTo
{
    return $this->belongsTo(RelationshipGroupType::class, 'relationship_group_type_id');
}
```

[RelationshipGroupType.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Models/RelationshipGroupType.php) 第51-54行：
```php
public function account(): BelongsTo
{
    return $this->belongsTo(Account::class);
}
```

**校验层级：**
```
relationship_type_id
    ↓ belongsTo
relationship_group_type (RelationshipGroupType)
    ↓ has account_id
account_id 校验
```

### 2.5 校验完整性总结

| 校验对象 | 校验位置 | 校验内容 |
|---------|---------|---------|
| contact_id | permissions 层 | 属于指定 vault |
| other_contact_id | execute() 方法 | 属于指定 vault |
| relationship_type_id | execute() 方法 | 所属 group 属于指定 account |
| vault_id | permissions 层 | 属于指定 account |
| author_id | permissions 层 | 属于指定 account，且有 vault 编辑权限 |

---

## 三、Inverse Relationship 注释与实际实现的矛盾

### 3.1 源码注释的承诺

[SetRelationship.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/ManageRelationships/Services/SetRelationship.php) 第42-46行注释明确写道：

```php
/**
 * Set a relationship between two contacts.
 * When a relationship is created (father -> son), we need to create
 * the inverse relationship (son -> father) as well.
 */
```

> 注释承诺：当创建（父亲→儿子）关系时，**同时创建**反向关系（儿子→父亲）。

### 3.2 实际实现

第59-72行的实际代码：

```php
// create the relationships
$this->setRelationship($this->contact, $otherContact, $relationshipType);

private function setRelationship(Contact $contact, Contact $otherContact, RelationshipType $relationshipType): void
{
    $contact->relationships()->syncWithoutDetaching([
        $otherContact->id => [
            'relationship_type_id' => $relationshipType->id,
        ],
    ]);
}
```

**只执行了一次写入！**

### 3.3 关联关系定义

[Contact.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Models/Contact.php) 第184-187行：

```php
public function relationships(): BelongsToMany
{
    return $this->belongsToMany(Contact::class, 'relationships', 'contact_id', 'related_contact_id');
}
```

**数据库表结构（推断）：**
- 表名：`relationships`
- 字段：
  - `id`（主键，整数）
  - `contact_id`（源联系人 UUID）
  - `related_contact_id`（目标联系人 UUID）
  - `relationship_type_id`（关系类型 ID）
  - 可能还有 `created_at`、`updated_at`

### 3.4 数据存储模式

**实际存储（有向图）：**

| 操作 | 数据行 | 说明 |
|-----|--------|------|
| A → B（父亲→儿子） | `contact_id=A, related_contact_id=B, relationship_type_id=父亲类型` | 只存储一行 |
| B → A 的反向关系 | 不存在 | **没有**自动创建 |

**注释期望的存储（双向图）：**

| 操作 | 数据行 | 说明 |
|-----|--------|------|
| A → B（父亲→儿子） | `contact_id=A, related_contact_id=B, relationship_type_id=父亲类型` | 正向 |
| B → A（儿子→父亲） | `contact_id=B, related_contact_id=A, relationship_type_id=儿子类型` | 反向（注释期望但未实现） |

### 3.5 矛盾的根源：视图层的补偿

查看 [ModuleRelationshipViewHelper.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/ManageRelationships/Web/ViewHelpers/ModuleRelationshipViewHelper.php) 第30-55行的查询逻辑：

```php
$relations = DB::table('relationships')
    ->join('contacts as contact1', 'relationships.contact_id', '=', 'contact1.id')
    ->join('contacts as contact2', 'relationships.related_contact_id', '=', 'contact2.id')
    ->join('relationship_types', 'relationships.relationship_type_id', '=', 'relationship_types.id')
    ->select(...)
    ->where('relationships.relationship_type_id', $relationshipType->id)
    ->where(function ($query) use ($contact) {
        $query->where('relationships.contact_id', $contact->id)
            ->orWhere('relationships.related_contact_id', $contact->id);
    })
    ->get();

foreach ($relations as $relation) {
    if ($relation->contact_id === $contact->id) {
        // 当前联系人是源，显示反向关系名称
        $relatedContact = Contact::find($relation->related_contact_id);
        $relationshipName = $relationshipType->name_reverse_relationship;
    } else {
        // 当前联系人是目标，显示正向关系名称
        $relatedContact = Contact::find($relation->contact_id);
        $relationshipName = $relationshipType->name;
    }
}
```

**视图层的补偿逻辑：**
1. 查询时**双向搜索**：`contact_id = X OR related_contact_id = X`
2. 根据当前联系人在关系中的位置（源/目标）动态决定显示正向还是反向关系名称
3. 使用 [RelationshipType.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Models/RelationshipType.php) 中的 `name` 和 `name_reverse_relationship` 字段

### 3.6 设计模式分析

这是典型的 **"单向存储，双向查询"** 模式：

| 维度 | 设计选择 | 理由 |
|-----|---------|------|
| 写入 | 只存一次 | 节省存储空间，避免数据不一致 |
| 查询 | 双向搜索 | 通过 OR 条件确保能找到所有关联 |
| 展示 | 动态切换关系名称 | 根据查询结果动态决定显示正向/反向名称 |
| 注释 | 描述用户感知 | 从用户角度描述为"创建双向关系" |

---

## 四、UnsetRelationship 的双向 detach 与 unlisted 联系人删除

### 4.1 UnsetRelationship 核心逻辑

[UnsetRelationship.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/ManageRelationships/Services/UnsetRelationship.php) 第47-71行：

```php
public function execute(array $data): void
{
    $this->validateRules($data);

    $otherContact = $this->vault->contacts()
        ->findOrFail($data['other_contact_id']);

    $this->relationshipType = RelationshipType::findOrFail($data['relationship_type_id']);
    if ($this->relationshipType->groupType->account_id !== $data['account_id']) {
        throw new ModelNotFoundException;
    }

    // 双向 detach
    $this->unsetRelationship($this->contact, $otherContact);
    $this->unsetRelationship($otherContact, $this->contact);

    // 删除 unlisted 联系人
    if (! $this->contact->listed) {
        $this->contact->delete();
    }

    if (! $otherContact->listed) {
        $otherContact->delete();
    }

    $this->updateLastEditedDate();
}
```

### 4.2 为什么要双向 detach？

#### 原因1：数据库中可能存在双向记录

虽然 `SetRelationship` 只写一次，但考虑以下场景：
1. 用户先设置 A → B（父亲）
2. 用户又单独设置 B → A（儿子）
3. 此时数据库中存在**两条独立记录**

如果只 detach 一个方向，另一个方向会成为**孤儿关系**。

#### 原因2：与视图层查询逻辑一致

视图层是双向查询，删除也必须双向删除，否则会出现：
- A 的关系列表中看不到 B
- 但 B 的关系列表中仍然能看到 A

#### 原因3：确保幂等性

无论数据库中是单向还是双向记录，执行一次 `UnsetRelationship` 都能**彻底清除**两个方向的关系。

### 4.3 为什么要删除 unlisted 联系人？

#### 什么是 unlisted 联系人？

[Contact.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Models/Contact.php) 第58行：
```php
'listed' => 'boolean',
```

第104-107行：
```php
public function shouldBeSearchable()
{
    return $this->listed;
}
```

第112-115行：
```php
public function scopeActive(Builder $query): Builder
{
    return $query->where('listed', 1);
}
```

**unlisted 联系人的特性：**
- 不会出现在搜索结果中（`shouldBeSearchable()` 返回 false）
- 不会出现在联系人列表中（`scopeActive()` 过滤）
- 只能通过直接访问 URL 或关系链接查看
- 通常是为了某个关系而临时创建的"附属"联系人

#### 删除逻辑的设计意图

回到控制器 [ContactRelationshipsController.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/ManageRelationships/Web/Controllers/ContactRelationshipsController.php) 第47-61行：

```php
if ($request->input('choice') !== 'contact') {
    $otherContact = (new CreateContact)->execute([
        ...
        'listed' => $request->input('create_contact_entry'),
        ...
    ]);
}
```

**场景示例：**
1. 用户正在查看联系人"张三"
2. 用户想添加"父亲"关系，但系统中没有"张父"这个联系人
3. 用户选择"创建新联系人"，并勾选"不创建独立联系人条目"（`create_contact_entry = false`）
4. 创建的"张父"是 unlisted 联系人，仅作为关系的另一端存在
5. 当删除"张三-父亲-张父"关系时，"张父"失去了存在的意义，应该被自动清理

### 4.4 与 SetRelationship 的不对称性

| 操作 | 写入/删除方向 | 处理 unlisted |
|-----|-------------|--------------|
| SetRelationship | **单向**写入（只写 A→B） | 创建时可选 unlisted |
| UnsetRelationship | **双向**删除（删 A→B 和 B→A） | 删除时自动清理 unlisted |

---

## 五、不对称设计的影响分析

### 5.1 对图谱展示的影响

#### 单向存储 + 双向查询 = 正确的展示

[ModuleRelationshipViewHelper.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/ManageRelationships/Web/ViewHelpers/ModuleRelationshipViewHelper.php) 的查询逻辑确保了展示正确性：

```php
->where(function ($query) use ($contact) {
    $query->where('relationships.contact_id', $contact->id)
        ->orWhere('relationships.related_contact_id', $contact->id);
})
```

**展示逻辑：**
- 查看 A 时，能找到 A→B 的记录，显示 "A 的父亲是 B"
- 查看 B 时，通过 `related_contact_id = B` 也能找到同一条记录，显示 "B 的儿子是 A"

#### 潜在问题：关系方向丢失

如果用户需要**明确知道**关系是"谁设置的"或"谁指向谁"，单向存储会丢失这个元信息。

### 5.2 对重复建立的影响

#### SetRelationship 使用 syncWithoutDetaching

```php
$contact->relationships()->syncWithoutDetaching([
    $otherContact->id => [
        'relationship_type_id' => $relationshipType->id,
    ],
]);
```

**syncWithoutDetaching 的特性：**
- 如果关系不存在：创建新关系
- 如果关系已存在：**更新** relationship_type_id
- 不会删除其他已有关系

#### 重复建立的场景分析

| 场景 | 行为 | 结果 |
|-----|------|------|
| 重复建立 A→B（同类型） | 同步但无变化 | 数据不变 |
| 重复建立 A→B（不同类型） | 更新 relationship_type_id | 类型被覆盖 |
| 先建立 A→B，再建立 B→A | 创建第二条独立记录 | 数据库中存在双向两条记录 |

#### 问题：双向记录的不一致

当数据库中存在 A→B 和 B→A 两条记录时：
1. 两者的 relationship_type_id 可能**不一致**
2. 视图层查询会返回两条记录
3. 用户可能看到重复的关系（"A是B的父亲" 和 "B是A的儿子"）

### 5.3 删除后的孤儿数据问题

#### UnsetRelationship 的清理机制

```php
$this->unsetRelationship($this->contact, $otherContact);
$this->unsetRelationship($otherContact, $this->contact);

if (! $this->contact->listed) {
    $this->contact->delete();
}
if (! $otherContact->listed) {
    $otherContact->delete();
}
```

#### 孤儿数据的定义

**孤儿关系：**
- 关系的一端联系人已被删除，但关系记录仍存在
- 或关系记录存在，但 relationship_type 已被删除

**孤儿联系人：**
- unlisted 联系人失去了所有关系，但未被自动删除

#### 潜在的孤儿数据场景

**场景1：跨关系类型删除不完整**

```
UnsetRelationship 只删除指定 relationship_type_id 的关系
    ↓
如果 A 和 B 之间还有其他类型的关系
    ↓
这些关系不会被删除
    ↓
如果被删除的是 unlisted 联系人，其他关系仍然引用它
    ↓
但 unlisted 联系人已经被删除了！
    ↓
产生孤儿关系
```

**场景2：删除时不检查其他关系**

```php
if (! $this->contact->listed) {
    $this->contact->delete();
}
```

> **问题**：只检查了 `listed` 属性，没有检查该联系人是否还有**其他关系**。

**修复建议：**
```php
if (! $this->contact->listed && $this->contact->relationships()->count() === 0) {
    $this->contact->delete();
}
```

**场景3：软删除与关系的一致性**

[Contact.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Models/Contact.php) 第29行使用了 `SoftDeletes` trait，而视图层查询（ModuleRelationshipViewHelper.php 第36-37行）也过滤了已删除的联系人：

```php
->where('contact1.deleted_at', null)
->where('contact2.deleted_at', null)
```

这确保了软删除的联系人不会出现在关系列表中，但关系记录本身**不会被自动清理**，仍然留在数据库中。

### 5.4 不对称性总结表

| 影响维度 | 具体表现 | 风险等级 | 建议 |
|---------|---------|---------|------|
| **图谱展示** | 视图层补偿确保正确性，但方向元信息丢失 | 中 | 如需要可在展示时增加方向标识 |
| **重复建立** | 先A→B再B→A会产生两条独立记录，可能不一致 | 高 | SetRelationship 应检查反向记录并保持同步 |
| **孤儿关系** | 删除 unlisted 联系人时不检查其他关系 | 高 | 删除前检查 relationships()->count() === 0 |
| **软删除孤儿** | 联系人软删除后关系记录仍存在 | 中 | 可考虑在模型 deleting 事件中清理关系 |
| **数据一致性** | 单向存储降低了冗余但增加了查询复杂度 | 中 | 评估是否改为双向存储 |

---

## 六、核心代码溯源

### 6.1 SetRelationship 完整流程

[SetRelationship.php:47-63](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/ManageRelationships/Services/SetRelationship.php#L47-L63)

```
validateRules(data)
    ↓
validatePermission（4项权限检查）
    ↓
otherContact = vault.contacts.findOrFail(other_contact_id)
    ↓
relationshipType = RelationshipType.findOrFail(relationship_type_id)
    ↓
relationshipType.groupType.account_id === data.account_id ?
    ↓ 是
setRelationship(contact, otherContact, relationshipType)
    ↓
syncWithoutDetaching（单向写入）
    ↓
updateLastEditedDate()
```

### 6.2 UnsetRelationship 完整流程

[UnsetRelationship.php:47-71](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/ManageRelationships/Services/UnsetRelationship.php#L47-L71)

```
validateRules(data)
    ↓
validatePermission（4项权限检查）
    ↓
otherContact = vault.contacts.findOrFail(other_contact_id)
    ↓
relationshipType 校验（同 SetRelationship）
    ↓
unsetRelationship(contact, otherContact)  → detach A→B
    ↓
unsetRelationship(otherContact, contact)  → detach B→A
    ↓
!contact.listed ? contact.delete()
    ↓
!otherContact.listed ? otherContact.delete()
    ↓
updateLastEditedDate()
```

### 6.3 关键模型关联

**Contact → Contact（多对多）：**
[Contact.php:184-187](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Models/Contact.php#L184-L187)
```php
return $this->belongsToMany(Contact::class, 'relationships', 'contact_id', 'related_contact_id');
```

**RelationshipType → RelationshipGroupType（多对一）：**
[RelationshipType.php:53-56](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Models/RelationshipType.php#L53-L56)
```php
return $this->belongsTo(RelationshipGroupType::class, 'relationship_group_type_id');
```

**RelationshipGroupType → Account（多对一）：**
[RelationshipGroupType.php:51-54](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Models/RelationshipGroupType.php#L51-L54)
```php
return $this->belongsTo(Account::class);
```

---

## 七、改进建议

### 7.1 修复 UnsetRelationship 中的孤儿数据风险

```php
// 原代码
if (! $this->contact->listed) {
    $this->contact->delete();
}

// 建议修改
if (! $this->contact->listed && $this->contact->relationships()->count() === 0) {
    $this->contact->delete();
}
```

### 7.2 SetRelationship 应同步反向关系类型

```php
private function setRelationship(Contact $contact, Contact $otherContact, RelationshipType $relationshipType): void
{
    $contact->relationships()->syncWithoutDetaching([
        $otherContact->id => [
            'relationship_type_id' => $relationshipType->id,
        ],
    ]);
    
    // 可选：确保反向关系类型一致（如果存在反向类型定义）
    // if ($relationshipType->name_reverse_relationship) { ... }
}
```

### 7.3 考虑使用模型事件清理关系

在 Contact 模型中添加：

```php
protected static function boot(): void
{
    parent::boot();
    
    static::deleting(function (self $model) {
        // 删除联系人时清理所有关系
        $model->relationships()->detach();
        // 清理作为目标的关系
        DB::table('relationships')
            ->where('related_contact_id', $model->id)
            ->delete();
    });
}
```

### 7.4 修复注释与实现不一致

更新 SetRelationship 类的注释，准确描述实际行为：

```php
/**
 * Set a relationship between two contacts.
 * Stores the relationship as a directed edge (contact -> otherContact)
 * with the given relationship type. The inverse relationship is not
 * stored separately, but is resolved at query time by checking both
 * contact_id and related_contact_id fields.
 */
```

---

## 八、总结

本分析揭示了 Monica 关系模块的核心设计模式：**"单向存储，双向查询，注释描述用户感知"**。

| 设计决策 | 优点 | 缺点 |
|---------|------|------|
| SetRelationship 单向写入 | 存储高效，避免冗余 | 注释与实现不一致，方向元信息丢失 |
| UnsetRelationship 双向删除 | 确保彻底清除，幂等操作 | 与写入不对称，可能造成认知困惑 |
| unlisted 联系人自动清理 | 避免垃圾数据 | 存在孤儿关系风险（未检查其他关系） |
| 视图层双向查询 + 动态名称 | 展示正确，用户体验好 | 查询逻辑复杂，性能开销 |

最严重的问题是 **UnsetRelationship 删除 unlisted 联系人前未检查是否还有其他关系**，这可能导致孤儿关系数据。建议优先修复此问题。
