# ContactController 技术分析报告

## 一、整体架构与依赖关系

[ContactController](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/ManageContact/Web/Controllers/ContactController.php) 通过 **服务层（Services）** 和 **视图辅助器（ViewHelpers）** 两层架构来保持联系人状态管理。

### 依赖关系图

```
ContactController
├── 服务层依赖
│   ├── CreateContact      # 创建联系人
│   ├── UpdateContact      # 更新联系人
│   ├── DestroyContact     # 删除联系人
│   └── UpdateContactView  # 更新访问次数
└── 视图辅助器依赖
    ├── ContactIndexViewHelper  # 列表页数据
    ├── ContactCreateViewHelper # 创建页数据
    ├── ContactShowViewHelper   # 详情页数据
    └── ContactEditViewHelper   # 编辑页数据
```

### 五个动作的状态流转

| 动作 | 服务层 | 视图辅助器 | 状态保持方式 |
|------|--------|------------|------------|
| **列表（index）** | 无直接服务调用 | ContactIndexViewHelper | 通过分页查询 + 用户排序偏好 |
| **创建（create/store）** | CreateContact | ContactCreateViewHelper | 服务创建记录 + Feed Item 记录 |
| **展示（show）** | UpdateContactView | ContactShowViewHelper | 更新访问次数 + 模板数据渲染 |
| **跳转（页面跳转）** | 无 | ContactShowViewHelper | 模板缺失时跳转blank页面 |
| **删除（destroy）** | DestroyContact | 无 | 级联删除文件 + 软删除联系人 |

---

## 二、模板缺失时跳转blank页面逻辑分析

### 核心代码位置

- [ContactController.php#L108-L113](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/ManageContact/Web/Controllers/ContactController.php#L108-L113)
- [ContactPageController.php#L30-L35](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/ManageContact/Web/Controllers/ContactPageController.php#L30-L35)
- [ContactNoTemplateController.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/ManageContact/Web/Controllers/ContactNoTemplateController.php)

### 跳转原因分析

```php
// ContactController@show
if (! $contact->template_id) {
    return redirect()->route('contact.blank', [
        'vault' => $vaultId,
        'contact' => $contactId,
    ]);
}
```

**为什么需要blank页面：**

1. **模板驱动设计**：Monica采用模板驱动的联系人展示方式，所有联系人信息的展示都依赖于模板（Template）定义的页面结构和模块配置。

2. **数据完整性保障**：[ContactShowViewHelper](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/ManageContact/Web/ViewHelpers/ContactShowViewHelper.php) 在渲染时需要访问 `$contact->template->pages()`（[第43行](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/ManageContact/Web/ViewHelpers/ContactShowViewHelper.php#L43)），如果没有模板会导致空指针异常。

3. **用户引导**：Blank页面提供了更新模板的入口，引导用户为联系人选择合适的模板后再进行正常展示。

4. **路由定义**：在 [routes/web.php#L275](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/routes/web.php#L275) 中定义了 `contact.blank` 路由，指向 `ContactNoTemplateController@show`。

---

## 三、CreateContact 默认模板选择与Feed Item写入

### 默认模板选择逻辑

**代码位置**：[CreateContact.php#L86-L113](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/ManageContact/Services/CreateContact.php#L86-L113)

```php
private function createContact(): void
{
    // template - if no template is provided, we should use the default
    // template that is in the vault - if it exists.
    $templateId = $this->valueOrNull($this->data, 'template_id');
    if (! $templateId) {
        $templateId = $this->vault->default_template_id;
    }

    $this->contact = Contact::create([
        'template_id' => $templateId,
        // ... 其他字段
    ]);
}
```

**选择规则：**

1. **优先级1**：用户在前端通过 [ContactCreateViewHelper](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/ManageContact/Web/ViewHelpers/ContactCreateViewHelper.php#L32-L38) 选择的模板（`$request->input('template_id')`）

2. **优先级2**：如果用户未选择，则使用 Vault 的默认模板（`$this->vault->default_template_id`）

3. **视图辅助器支持**：ContactCreateViewHelper 在模板列表中自动标记 vault 默认模板为选中状态：
   ```php
   'selected' => $template->id === $vault->default_template_id,
   ```

### Feed Item 写入机制

**代码位置**：[CreateContact.php#L121-L128](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/ManageContact/Services/CreateContact.php#L121-L128)

```php
private function createFeedItem(): void
{
    ContactFeedItem::create([
        'author_id' => $this->author->id,
        'contact_id' => $this->contact->id,
        'action' => ContactFeedItem::ACTION_CONTACT_CREATED,
    ]);
}
```

**执行流程：**
1. `execute()` 方法按顺序调用：`validate()` → `createContact()` → `updateLastEditedDate()` → `createFeedItem()`
2. Feed Item 记录了创建者、联系人ID和动作类型，用于后续在 Feed 模块中展示活动历史

---

## 四、show方法更新contact_vault_user访问次数

**代码位置**：
- [ContactController.php#L115-L120](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/ManageContact/Web/Controllers/ContactController.php#L115-L120)
- [UpdateContactView.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/ManageContact/Services/UpdateContactView.php)

### 调用时机

在 `show()` 方法渲染页面前调用：
```php
(new UpdateContactView)->execute([
    'account_id' => Auth::user()->account_id,
    'vault_id' => $vaultId,
    'author_id' => Auth::id(),
    'contact_id' => $contactId,
]);
```

### 更新逻辑

```php
private function updateView(): void
{
    $contact = [
        'contact_id' => $this->data['contact_id'],
        'vault_id' => $this->data['vault_id'],
        'user_id' => $this->data['author_id'],
    ];

    $exists = DB::table('contact_vault_user')
        ->where($contact)
        ->exists();

    if ($exists) {
        DB::table('contact_vault_user')
            ->where($contact)
            ->increment('number_of_views');
    } else {
        DB::table('contact_vault_user')->insert($contact + [
            'number_of_views' => 1,
        ]);
    }
}
```

**设计要点：**

1. **多对多关联表**：使用中间表 `contact_vault_user` 记录每个用户在每个 vault 中对每个联系人的访问次数

2. **原子操作**：使用数据库 `increment()` 方法保证计数的原子性，避免并发问题

3. **首次访问处理**：如果记录不存在则插入新记录并设访问次数为1

4. **权限检查**：服务层通过 permissions() 方法确保用户在 vault 中（`author_must_be_in_vault`）

---

## 五、DestroyContact 先删除文件与禁止删除当前用户逻辑

### 先删除文件的原因

**代码位置**：[DestroyContact.php#L37-L59](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/ManageContact/Services/DestroyContact.php#L37-L59)

```php
public function execute(array $data): void
{
    $this->validateRules($data);

    if (! $this->contact->can_be_deleted) {
        throw new CantBeDeletedException;
    }

    $this->destroyFiles();  // 先删文件
    $this->contact->delete(); // 再删联系人
}

private function destroyFiles(): void
{
    $files = $this->contact->files;
    foreach ($files as $file) {
        $file->delete();
    }
}
```

**为什么先删文件：**

1. **数据一致性**：Contact 与 File 是多态关联关系（`morphMany`），如果先删除 Contact，可能导致关联文件成为孤儿记录

2. **级联删除控制**：手动遍历删除文件可以触发 File 模型的删除事件和观察者，确保文件资源（包括云存储）被正确清理

3. **事务安全性**：虽然代码中没有显式事务，但先删除依赖项再删除主记录是标准做法，即使中间失败也不会留下孤立的主记录

### 禁止删除当前用户自己的联系人

**两处检查点：**

1. **数据库层检查** - [DestroyContact.php#L44-L46](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/ManageContact/Services/DestroyContact.php#L44-L46)：
   ```php
   if (! $this->contact->can_be_deleted) {
       throw new CantBeDeletedException;
   }
   ```
   Contact 模型有 `can_be_deleted` 布尔字段（[Contact.php#L52](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Models/Contact.php#L52)），代表当前用户自己的联系人此字段为 `false`。

2. **视图层控制** - [ContactShowViewHelper.php#L64-L65](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/ManageContact/Web/ViewHelpers/ContactShowViewHelper.php#L64-L65)：
   ```php
   'options' => [
       'can_be_archived' => $user->getContactInVault($contact->vault)->id !== $contact->id,
       'can_be_deleted' => $user->getContactInVault($contact->vault)->id !== $contact->id,
   ],
   ```

**设计意图：**

- **完整性保护**：每个用户在 vault 中有一个代表自己的联系人记录，删除此记录会导致用户身份引用断裂
- **防止误操作**：在视图层面隐藏删除按钮，在服务层面抛出异常，形成双重保护

---

## 六、Contact模型搜索索引更新条件

**代码位置**：[Contact.php#L87-L137](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Models/Contact.php#L87-L137)

### 搜索索引字段

```php
#[SearchUsingFullText(['first_name', 'last_name', 'middle_name', 'nickname', 'maiden_name'], ['expanded' => true])]
public function toSearchableArray(): array
{
    return array_merge(ScoutHelper::id($this), [
        'first_name' => $this->first_name ?? '',
        'last_name' => $this->last_name ?? '',
        'middle_name' => $this->middle_name ?? '',
        'nickname' => $this->nickname ?? '',
        'maiden_name' => $this->maiden_name ?? '',
    ]);
}
```

### 是否可搜索条件

```php
public function shouldBeSearchable()
{
    return $this->listed;
}
```

- 只有 `listed = true` 的联系人才会被加入搜索索引
- 归档联系人（`listed = false`）不会出现在搜索结果中

### 索引更新条件

```php
public function searchIndexShouldBeUpdated()
{
    return ScoutHelper::isActivated();
}
```

**更新条件：**

1. **Scout服务激活**：通过 `ScoutHelper::isActivated()` 检查搜索服务是否启用
2. **删除时清理关联**：在模型 boot 方法中定义了删除时的联动清理：
   ```php
   static::deleting(function (self $model) {
       $model->notes()->unsearchable();
   });
   ```
3. **字段变化触发**：当姓名相关字段（first_name, last_name, middle_name, nickname, maiden_name）变化时，Laravel Scout 会自动更新索引

---

## 七、总结

### 架构设计特点

| 层面 | 设计模式 | 优势 |
|------|---------|------|
| Controller | 薄控制器 | 只负责路由和请求响应，业务逻辑委托给服务层 |
| Service | 命令模式 | 每个操作独立封装，可复用可测试 |
| ViewHelper | 表示模式 | 视图数据集中组装，保持模板纯净 |
| Model | 领域模型 | 包含搜索、关联、访问器等领域逻辑 |

### 关键设计决策

1. **模板驱动展示**：所有联系人展示都依赖模板，保证UI一致性
2. **访问统计**：通过中间表记录细粒度访问数据
3. **分层保护**：重要操作在视图层和服务层都有检查
4. **搜索优化**：仅索引活跃联系人，降低索引体积
