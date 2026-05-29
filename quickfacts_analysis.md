# Quick Facts 全链路分析文档

## 1. VaultQuickFactsTemplate 为什么属于 Vault

### 数据库表结构与模型关系

[VaultQuickFactsTemplate.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Models/VaultQuickFactsTemplate.php) 模型通过以下方式明确归属 Vault：

1. **`vault_id` 外键字段**：在模型的 `$fillable` 属性中包含 `vault_id`，表明每个模板记录都关联到特定的 Vault

2. **`vault()` 关联方法**：
```php
public function vault(): BelongsTo
{
    return $this->belongsTo(Vault::class);
}
```

3. **Vault 模型的反向关联**：在 [Vault.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Models/Vault.php#L251-L254) 中定义了一对多关系：
```php
public function quickFactsTemplateEntries(): HasMany
{
    return $this->hasMany(VaultQuickFactsTemplate::class);
}
```

### 设计意图

- **Vault 级别的模板复用**：同一 Vault 内的所有联系人共享一套 Quick Facts 模板
- **数据隔离**：不同 Vault 之间的模板完全隔离，符合 Monica 的多租户（Vault）架构
- **集中管理**：模板在 Vault 级别定义，联系人只需引用模板 ID 并存储具体内容

---

## 2. CreateQuickFact 如何验证 template 和 contact 同属一个 Vault

[CreateQuickFact.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/ManageQuickFacts/Services/CreateQuickFact.php) 通过**两层验证机制**确保 template 和 contact 同属一个 Vault：

### 第一层：权限系统的基础验证

在 `permissions()` 方法中定义了以下权限检查：
```php
public function permissions(): array
{
    return [
        'author_must_belong_to_account',
        'vault_must_belong_to_account',
        'author_must_be_vault_editor',
        'contact_must_belong_to_vault',  // 确保 contact 属于给定的 vault_id
    ];
}
```

这些权限由 `BaseService` 在执行时自动验证，确保：
- Contact 属于传入的 `vault_id` 对应的 Vault

### 第二层：Template 归属验证

在 `validate()` 方法中，通过 Vault 的关联关系来查找模板：
```php
private function validate(): void
{
    $this->validateRules($this->data);

    $this->vault->quickFactsTemplateEntries()
        ->findOrFail($this->data['vault_quick_facts_template_id']);
}
```

**关键逻辑**：
- 不是直接通过 `VaultQuickFactsTemplate::find($templateId)` 查找
- 而是通过 `$this->vault->quickFactsTemplateEntries()` 关联关系查找
- 如果 template 不属于该 vault，`findOrFail` 会抛出 `ModelNotFoundException`

### 验证流程总结

```
传入参数: vault_id, contact_id, vault_quick_facts_template_id
    ↓
权限检查: contact_must_belong_to_vault → 确保 contact ∈ vault
    ↓
模板检查: vault->quickFactsTemplateEntries()->findOrFail(template_id) → 确保 template ∈ vault
    ↓
两者都通过 → template 和 contact 同属一个 vault
```

---

## 3. ContactQuickFactController 的 show/store/update/destroy 依赖分析

[ContactQuickFactController.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/ManageQuickFacts/Web/Controllers/ContactQuickFactController.php)

### 3.1 show 方法

```php
public function show(Request $request, string $vaultId, string $contactId, int $templateId): JsonResponse
{
    $contact = Contact::find($contactId);
    $template = $contact->vault->quickFactsTemplateEntries()->findOrFail($templateId);

    return response()->json([
        'data' => ContactModuleQuickFactViewHelper::data($contact, $template),
    ], 200);
}
```

**依赖**：
- **模型**：`Contact`（直接查询）
- **关联关系**：`$contact->vault`、`$contact->vault->quickFactsTemplateEntries()`
- **视图辅助类**：[ContactModuleQuickFactViewHelper::data()](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/ManageQuickFacts/Web/ViewHelpers/ContactModuleQuickFactViewHelper.php#L11-L32)

**ViewHelper::data() 返回的数据结构**：
- `template`: id, label, store url
- `quick_facts`: 该联系人在该模板下的所有 quick facts（通过 `$contact->quickFacts()` 查询）

### 3.2 store 方法

```php
public function store(Request $request, string $vaultId, string $contactId, int $templateId): JsonResponse
{
    $data = [
        'account_id' => Auth::user()->account_id,
        'author_id' => Auth::id(),
        'vault_id' => $vaultId,
        'contact_id' => $contactId,
        'vault_quick_facts_template_id' => $templateId,
        'content' => $request->input('content'),
    ];

    $quickFact = (new CreateQuickFact)->execute($data);

    return response()->json([
        'data' => ContactModuleQuickFactViewHelper::dto($quickFact),
    ], 201);
}
```

**依赖**：
- **认证**：`Auth::user()`、`Auth::id()`
- **服务类**：[CreateQuickFact](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/ManageQuickFacts/Services/CreateQuickFact.php)
- **视图辅助类**：[ContactModuleQuickFactViewHelper::dto()](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/ManageQuickFacts/Web/ViewHelpers/ContactModuleQuickFactViewHelper.php#L34-L56)

### 3.3 update 方法

```php
public function update(Request $request, string $vaultId, string $contactId, int $templateId, int $quickFactId): JsonResponse
{
    $data = [
        'account_id' => Auth::user()->account_id,
        'author_id' => Auth::id(),
        'vault_id' => $vaultId,
        'contact_id' => $contactId,
        'quick_fact_id' => $quickFactId,
        'content' => $request->input('content'),
    ];

    $quickFact = (new UpdateQuickFact)->execute($data);

    return response()->json([
        'data' => ContactModuleQuickFactViewHelper::dto($quickFact),
    ], 200);
}
```

**依赖**：
- **认证**：`Auth::user()`、`Auth::id()`
- **服务类**：[UpdateQuickFact](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/ManageQuickFacts/Services/UpdateQuickFact.php)
  - 验证逻辑：通过 `$this->contact->quickFacts()->findOrFail($quick_fact_id)` 确保 quickFact 属于该 contact
- **视图辅助类**：`ContactModuleQuickFactViewHelper::dto()`

### 3.4 destroy 方法

```php
public function destroy(Request $request, string $vaultId, string $contactId, int $templateId, int $quickFactId)
{
    $data = [
        'account_id' => Auth::user()->account_id,
        'author_id' => Auth::id(),
        'vault_id' => $vaultId,
        'contact_id' => $contactId,
        'quick_fact_id' => $quickFactId,
    ];

    (new DestroyQuickFact)->execute($data);

    return response()->json([
        'data' => true,
    ], 200);
}
```

**依赖**：
- **认证**：`Auth::user()`、`Auth::id()`
- **服务类**：[DestroyQuickFact](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/ManageQuickFacts/Services/DestroyQuickFact.php)
  - 验证逻辑：通过 `$this->contact->quickFacts()->findOrFail($quick_fact_id)` 确保 quickFact 属于该 contact

---

## 4. ToggleQuickFactModule 改变的字段及影响

### 4.1 改变的是联系人字段

[ToggleQuickFactModule.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/ManageQuickFacts/Services/ToggleQuickFactModule.php) 的核心逻辑：

```php
private function update(): void
{
    $this->contact->show_quick_facts = ! $this->contact->show_quick_facts;
    $this->contact->save();
}
```

**明确结论**：ToggleQuickFactModule 改变的是 **联系人（Contact）的 `show_quick_facts` 字段**，不是模板字段。

该字段在 [Contact.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Models/Contact.php#L53) 中定义：
- 属于 `$fillable` 属性，可批量赋值
- 在 `$casts` 中被转换为 `boolean` 类型

### 4.2 对同一 Vault 内其他联系人的影响

**答案：完全没有影响**

原因如下：

1. **字段级别**：`show_quick_facts` 是 Contact 表的字段，每个联系人有独立的布尔值
   - 表结构：`contacts` 表包含 `show_quick_facts` 列
   - 每个联系人记录有自己的值

2. **代码逻辑**：ToggleQuickFactModule 只操作当前传入的 contact_id 对应的记录
   ```php
   // 只修改 $this->contact（通过 contact_id 查找到的特定联系人）
   $this->contact->show_quick_facts = ! $this->contact->show_quick_facts;
   ```

3. **模板独立性**：Vault 的 Quick Facts 模板是共享的，但每个联系人可以独立决定是否显示 Quick Facts 模块

### 4.3 影响范围总结

| 影响对象 | 是否受影响 | 说明 |
|---------|-----------|------|
| 当前操作的联系人 | ✅ 是 | 切换其 show_quick_facts 值 |
| 同一 Vault 内其他联系人 | ❌ 否 | 每个联系人有独立的开关 |
| Vault Quick Facts 模板 | ❌ 否 | 模板不受开关影响 |
| 已创建的 Quick Fact 记录 | ❌ 否 | 数据保留，只是前端显示与否 |

---

## 5. 全链路架构图

```
Vault 级别
└── VaultQuickFactsTemplate (模板定义，vault_id 外键)
    └── 属于特定 Vault

Contact 级别
├── show_quick_facts (开关字段，每个联系人独立)
└── QuickFact (具体事实记录)
    ├── contact_id (关联联系人)
    ├── vault_quick_facts_template_id (关联模板)
    └── content (具体内容)

数据流向:
ContactController.show() → 通过 contact->vault->templates 找模板
                      → 通过 contact->quickFacts 找该模板下的事实
                      → ViewHelper 组装数据
```

---

## 6. 关键文件索引

| 功能 | 文件路径 |
|-----|---------|
| 模板模型 | [VaultQuickFactsTemplate.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Models/VaultQuickFactsTemplate.php) |
| 事实记录模型 | [QuickFact.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Models/QuickFact.php) |
| 控制器 | [ContactQuickFactController.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/ManageQuickFacts/Web/Controllers/ContactQuickFactController.php) |
| 创建服务 | [CreateQuickFact.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/ManageQuickFacts/Services/CreateQuickFact.php) |
| 更新服务 | [UpdateQuickFact.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/ManageQuickFacts/Services/UpdateQuickFact.php) |
| 删除服务 | [DestroyQuickFact.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/ManageQuickFacts/Services/DestroyQuickFact.php) |
| 模块开关服务 | [ToggleQuickFactModule.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/ManageQuickFacts/Services/ToggleQuickFactModule.php) |
| 视图辅助类 | [ContactModuleQuickFactViewHelper.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/ManageQuickFacts/Web/ViewHelpers/ContactModuleQuickFactViewHelper.php) |
| Vault 模型 | [Vault.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Models/Vault.php) |
| Contact 模型 | [Contact.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Models/Contact.php) |
