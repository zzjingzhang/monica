# Vault 搜索与最常查看联系人功能分析

## 1. 三个控制器的入口差异

### 1.1 VaultSearchController — 全局搜索

| 维度 | 说明 |
|------|------|
| 文件 | [VaultSearchController.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Vault/Search/Web/Controllers/VaultSearchController.php#L12-L32) |
| 路由 | `GET /vaults/{vaultId}/search` → `index`（渲染 Inertia 页面）<br>`POST /vaults/{vaultId}/search` → `show`（返回 JSON） |
| 路由名 | `vault.search.index` / `vault.search.show` |
| 视图辅助器 | [VaultSearchIndexViewHelper](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Vault/Search/Web/ViewHelpers/VaultSearchIndexViewHelper.php#L13-L88) |
| 搜索范围 | **Contact + Note + Group 三种模型**，全部使用 Scout 全文搜索 |
| 输入参数 | `searchTerm`（可选，为空时返回空数组） |
| 输出格式 | `index` 返回 Inertia 渲染页面，`show` 返回 JSON `{ data: { query, contacts, notes, groups, url } }` |
| 使用场景 | Vault 主搜索页面，用户输入关键词后同时搜索联系人、笔记和分组 |

**关键逻辑**：`VaultSearchIndexViewHelper::data()` 在 `searchTerm` 为 `null` 时，contacts/notes/groups 均返回空数组；只有在提供了搜索词时才调用 `Model::search($term)->where('vault_id', $vault->id)->get()`。

### 1.2 VaultContactSearchController — 模块内联系人搜索

| 维度 | 说明 |
|------|------|
| 文件 | [VaultContactSearchController.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Vault/Search/Web/Controllers/VaultContactSearchController.php#L14-L24) |
| 路由 | `POST /vaults/{vaultId}/search/user/contacts` |
| 路由名 | `vault.user.search.index` |
| 视图辅助器 | [VaultContactSearchViewHelper](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Vault/Search/Web/ViewHelpers/VaultContactSearchViewHelper.php#L9-L31) |
| 搜索范围 | **仅 Contact 模型**，使用 Scout 全文搜索 |
| 输入参数 | `searchTerm`（必需，参数类型为 `string`，非 nullable） |
| 输出格式 | JSON `{ data: Collection }`，最多返回 **5 条**结果 |
| 排序 | `orderBy('first_name')->orderBy('last_name')` |
| 使用场景 | Activity、Loans 等模块中需要快速选择联系人时的下拉搜索 |

**关键差异**：
- 仅搜索 Contact，不涉及 Note 和 Group
- 结果限制为 5 条并按姓名排序
- 没有页面渲染入口，只返回 JSON
- `searchTerm` 为必需参数（`string` 类型，非 `?string`）

### 1.3 VaultMostConsultedContactsController — 最常查看联系人

| 维度 | 说明 |
|------|------|
| 文件 | [VaultMostConsultedContactsController.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Vault/Search/Web/Controllers/VaultMostConsultedContactsController.php#L12-L26) |
| 路由 | `GET /vaults/{vaultId}/search/user/contact/mostConsulted` |
| 路由名 | `vault.user.search.mostconsulted` |
| 视图辅助器 | [VaultMostConsultedViewHelper](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Vault/Search/Web/ViewHelpers/VaultMostConsultedViewHelper.php#L11-L39) |
| 搜索范围 | **不使用 Scout 搜索**，直接查询 `contact_vault_user` 表 |
| 输入参数 | 无搜索词，仅依赖当前认证用户 `Auth::user()` |
| 输出格式 | JSON `{ data: Collection }`，最多返回 **5 条**结果 |
| 排序 | 按 `number_of_views` 降序 |
| 使用场景 | 显示当前用户在当前 Vault 中最常查看的联系人快捷列表 |

**关键差异**：
- 完全不涉及 Scout/搜索引擎，是基于访问计数的纯数据库查询
- 结果是用户维度的（每个用户看到自己的最常查看列表）
- 不需要搜索关键词

---

## 2. Contact、Note、Group 模型如何参与搜索

三个模型均使用 Laravel Scout 的 `Searchable` trait，但索引字段和可搜索条件各不相同。

### 2.1 Contact 模型

| 项目 | 详情 |
|------|------|
| 文件 | [Contact.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Models/Contact.php#L25-L107) |
| Searchable trait | `use Searchable`（第 28 行） |
| 软删除 | `use SoftDeletes`（第 29 行） |
| 索引内容 | `toSearchableArray()` 返回：`id, vault_id, created_at, updated_at`（来自 ScoutHelper::id）+ `first_name, last_name, middle_name, nickname, maiden_name` |
| 全文索引标注 | `#[SearchUsingFullText(['first_name', 'last_name', 'middle_name', 'nickname', 'maiden_name'], ['expanded' => true])]` |
| 可搜索条件 | `shouldBeSearchable()` → 返回 `$this->listed`（第 104-107 行） |
| 索引更新守卫 | `searchIndexShouldBeUpdated()` → `ScoutHelper::isActivated()`（第 134-137 行） |
| 删除联动 | `boot()` 中注册 `deleting` 事件：删除联系人时将其关联 Notes 从索引中移除（`$model->notes()->unsearchable()`）（第 124-127 行） |
| 活跃范围 | `scopeActive()` → `where('listed', 1)` |

**搜索参与方式**：
- `VaultSearchIndexViewHelper::contacts()`：`Contact::search($term)->where('vault_id', $vault->id)->get()`
- `VaultContactSearchViewHelper::data()`：`Contact::search($term)->where('vault_id', $vault->id)->orderBy('first_name')->orderBy('last_name')->take(5)->get()`

### 2.2 Note 模型

| 项目 | 详情 |
|------|------|
| 文件 | [Note.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Models/Note.php#L13-L55) |
| Searchable trait | `use Searchable`（第 17 行） |
| 软删除 | 无 |
| 索引内容 | `toSearchableArray()` 返回：`id, vault_id, created_at, updated_at`（来自 ScoutHelper::id）+ `contact_id, title, body` |
| 全文索引标注 | `#[SearchUsingFullText(['title', 'body'], ['expanded' => true])]` |
| 可搜索条件 | 无 `shouldBeSearchable()` 方法，**默认始终可搜索** |
| 索引更新守卫 | `searchIndexShouldBeUpdated()` → `ScoutHelper::isActivated()` |

**搜索参与方式**：
- 仅在 `VaultSearchIndexViewHelper::notes()` 中被搜索：`Note::search($term)->where('vault_id', $vault->id)->get()`
- 搜索结果中会关联加载 `note->contact` 以显示所属联系人信息

### 2.3 Group 模型

| 项目 | 详情 |
|------|------|
| 文件 | [Group.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Models/Group.php#L16-L61) |
| Searchable trait | `use Searchable`（第 21 行） |
| 软删除 | `use SoftDeletes`（第 22 行） |
| 索引内容 | `toSearchableArray()` 返回：`id, vault_id, created_at, updated_at`（来自 ScoutHelper::id）+ `name` |
| 全文索引标注 | `#[SearchUsingFullText(['name'], ['expanded' => true])]` |
| 可搜索条件 | 无 `shouldBeSearchable()` 方法，**默认始终可搜索** |
| 索引更新守卫 | `searchIndexShouldBeUpdated()` → `ScoutHelper::isActivated()` |

**搜索参与方式**：
- 仅在 `VaultSearchIndexViewHelper::groups()` 中被搜索：`Group::search($term)->where('vault_id', $vault->id)->get()`

### 2.4 三模型搜索对比总结

| 特性 | Contact | Note | Group |
|------|---------|------|-------|
| 索引字段 | 姓名五字段 | title + body + contact_id | name |
| `shouldBeSearchable` | `$this->listed`（归档则不可搜索） | 无（始终可搜索） | 无（始终可搜索） |
| 软删除 | 有 | 无 | 有 |
| 删除联动 | 删除时移除关联 Notes 索引 | 被联系人删除联动移除 | 无 |
| 搜索入口 | VaultSearch + VaultContactSearch | 仅 VaultSearch | 仅 VaultSearch |

---

## 3. ScoutHelper 和 config/scout 如何影响索引后端

### 3.1 ScoutHelper 三个核心方法

文件：[ScoutHelper.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Helpers/ScoutHelper.php#L8-L85)

#### `isActivated()`
根据 `config('scout.driver')` 判断搜索引擎是否已激活配置：
- `algolia`：检查 `scout.algolia.id` 非空
- `meilisearch`：检查 `scout.meilisearch.key` 非空
- `typesense`：检查 `scout.typesense.client-settings.api_key` 非空
- `database` / `collection`：始终返回 `true`
- 其他：返回 `false`

**影响**：所有三个模型的 `searchIndexShouldBeUpdated()` 方法都调用 `ScoutHelper::isActivated()`。当返回 `false` 时，模型更新不会触发索引同步，减少不必要的写入。

#### `isIndexed()`
判断驱动是否为外部索引引擎（Algolia/Meilisearch/Typesense），`database` 和 `collection` 返回 `false`。

**影响**：此方法在代码库中用于区分是否需要维护外部索引。

#### `isFullTextIndex()`
检查 `config('scout.full_text_index')` 为 `true` 且数据库驱动为 MySQL 或 PostgreSQL。

**影响**：在 [contacts 迁移文件](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/database/migrations/2020_04_25_133132_create_contacts_table.php#L52-L58) 中，当 `isFullTextIndex()` 为 `true` 时，会为 Contact 的 `first_name, last_name, middle_name, nickname, maiden_name` 字段创建数据库全文索引，供 `database` 驱动使用。

#### `id()`
为索引文档生成基础字段（`id, vault_id, created_at, updated_at`）。当驱动为 `database` 时返回空数组（因为数据库驱动不需要这些冗余字段）。

### 3.2 config/scout 配置结构

文件：[scout.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/config/scout.php#L1-L321)

| 配置项 | 值 | 影响 |
|--------|-----|------|
| `driver` | `env('SCOUT_DRIVER', 'algolia')` | 决定使用哪个搜索引擎后端 |
| `prefix` | `env('SCOUT_PREFIX', '')` | 索引名称前缀，多租户场景使用 |
| `queue` | `env('SCOUT_QUEUE', false)` | 是否异步同步索引 |
| `soft_delete` | `false` | **Scout 不保留软删除记录的索引** |
| `full_text_index` | `env('FULL_TEXT_INDEX', true)` | 是否创建数据库全文索引 |
| `meilisearch.index-settings` | Contact/Group/Note 各自的 filterable/sortable 属性 | Meilisearch 索引配置 |
| `typesense.model-settings` | 三种模型的 collection-schema 和 search-parameters | Typesense 索引配置 |

### 3.3 四种后端驱动的行为差异

| 驱动 | 搜索方式 | 索引存储 | 全文索引支持 | `isActivated` |
|------|----------|----------|-------------|---------------|
| `algolia` | 外部 API | Algolia 云端 | 由 Algolia 处理 | 需配置 APP_ID |
| `meilisearch` | 外部 API | Meilisearch 实例 | 由 Meilisearch 处理 | 需配置 KEY |
| `typesense` | 外部 API | Typesense 实例 | 由 Typesense 处理 | 需配置 API_KEY |
| `database` | SQL WHERE + 全文索引 | 本地数据库表 | 需要 `full_text_index=true` + MySQL/PgSQL | 始终 `true` |
| `collection` | 内存集合过滤 | 无持久化 | 无 | 始终 `true`（仅用于测试） |

### 3.4 Typesense 的搜索参数配置

在 `config/scout.php` 的 `typesense.model-settings` 中：

- **Contact**：`query_by` = `first_name,last_name,middle_name,nickname,maiden_name`，`default_sorting_field` = `updated_at`，包含 `__soft_deleted` 字段（`int32, optional`）
- **Group**：`query_by` = `name`，`default_sorting_field` = `updated_at`，包含 `__soft_deleted` 字段
- **Note**：`query_by` = `title,body`，`default_sorting_field` = `updated_at`，无 `__soft_deleted` 字段（Note 模型未使用 SoftDeletes）

---

## 4. UpdateContactView 写入 contact_vault_user 及 most consulted 视图消费

### 4.1 写入端：UpdateContactView 服务

文件：[UpdateContactView.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/ManageContact/Services/UpdateContactView.php#L9-L73)

**触发时机**：用户查看联系人详情页时，由两个控制器调用：

1. [ContactPageController::show()](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/ManageContact/Web/Controllers/ContactPageController.php#L41-L46) — 联系人模板页面
2. [ContactController::show()](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/ManageContact/Web/Controllers/ContactController.php#L115-L120) — 联系人默认页面

**写入逻辑**：

```
1. 验证 account_id, vault_id, author_id, contact_id 四个必填参数
2. 权限检查：author 必须属于 account、vault、且在 vault 中、contact 必须属于 vault
3. 查询 contact_vault_user 表是否已存在 (contact_id, vault_id, user_id) 记录
   - 已存在 → number_of_views + 1 (increment)
   - 不存在 → 插入新记录，number_of_views = 1
```

**表结构**（[迁移文件](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/database/migrations/2020_04_25_133132_create_contacts_table.php#L69-L76)）：

| 字段 | 类型 | 说明 |
|------|------|------|
| `contact_id` | UUID (FK) | 被查看的联系人 |
| `vault_id` | UUID (FK) | 所属 Vault |
| `user_id` | UUID (FK) | 查看者 |
| `number_of_views` | integer | 累计查看次数 |
| `is_favorite` | boolean | 是否收藏（默认 false） |
| `created_at` / `updated_at` | timestamp | 时间戳 |

**注意**：三元组 `(contact_id, vault_id, user_id)` 构成了逻辑唯一键，但迁移中没有显式声明 unique 约束。

### 4.2 消费端：VaultMostConsultedViewHelper

文件：[VaultMostConsultedViewHelper.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Vault/Search/Web/ViewHelpers/VaultMostConsultedViewHelper.php#L11-L39)

**查询逻辑**：

```php
DB::table('contact_vault_user')
    ->where('vault_id', $vault->id)
    ->where('user_id', $user->id)
    ->orderBy('number_of_views', 'desc')
    ->select('contact_id')
    ->limit(5)
    ->get()
```

**后续处理**：

```php
foreach ($records as $record) {
    $contact = Contact::find($record->contact_id);
    // 构建 { id, name, url } 数组
}
```

**数据流完整链路**：

```
用户访问联系人页面
  → ContactPageController / ContactController
    → UpdateContactView::execute()
      → contact_vault_user.number_of_views 自增
        → GET /vaults/{id}/search/user/contact/mostConsulted
          → VaultMostConsultedContactsController::index()
            → VaultMostConsultedViewHelper::data()
              → 查询 contact_vault_user 按 number_of_views DESC
              → Contact::find() 逐条加载联系人
              → 返回前 5 名最常查看联系人
```

---

## 5. 软删除或归档联系人是否应出现在搜索结果中

### 5.1 归档联系人（listed = false）

[ToggleArchiveContact](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/ManageContact/Services/ToggleArchiveContact.php#L44-L56) 将 `listed` 字段取反：`$this->contact->listed = ! $this->contact->listed`。

**对搜索的影响**：

- Contact 模型的 `shouldBeSearchable()` 返回 `$this->listed`
- 当 `listed = false`（已归档）时，`shouldBeSearchable()` 返回 `false`
- Scout 在同步索引时会跳过 `shouldBeSearchable() = false` 的记录，并从索引中移除已存在的记录
- **结论：归档联系人不会出现在 Scout 搜索结果中**

**对最常查看的影响**：

- `VaultMostConsultedViewHelper` 使用 `Contact::find($record->contact_id)` 加载联系人
- `Contact::find()` **不会**查询软删除的记录，但**会**查询 `listed = false` 的记录
- **结论：归档联系人仍然可能出现在最常查看列表中** — 这是一个潜在的设计缺陷

### 5.2 软删除联系人（deleted_at IS NOT NULL）

Contact 和 Group 模型使用了 `SoftDeletes` trait。

**对搜索的影响**：

- Laravel Scout 在模型使用 `SoftDeletes` 时的行为取决于 `config('scout.soft_delete')` 配置
- 当前配置 `scout.soft_delete = false`，这意味着软删除的记录会从搜索索引中**移除**
- 在 Typesense 的 collection-schema 中，Contact 和 Group 都定义了 `__soft_deleted` 字段（`int32, optional`），这是 Scout 自动管理的——当 `soft_delete = true` 时，Scout 会在索引中保留软删除记录并标记 `__soft_deleted`
- **当前配置下结论：软删除联系人不会出现在搜索结果中**

**对最常查看的影响**：

- `VaultMostConsultedViewHelper` 使用 `Contact::find()` 而非 `Contact::withTrashed()->find()`
- `find()` 会自动排除软删除记录，返回 `null`
- **但是**：代码中没有对 `Contact::find()` 返回 `null` 的情况做防护（[第 26 行](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Vault/Search/Web/ViewHelpers/VaultMostConsultedViewHelper.php#L26)），如果联系人被软删除，`$contact->id` 会触发 `Trying to get property of non-object` 错误
- **结论：软删除联系人不会出现在最常查看列表中，但会导致运行时错误**

### 5.3 Note 和 Group 的特殊情况

- **Note**：没有 `SoftDeletes`，也没有 `shouldBeSearchable()`，始终可搜索。但 Note 的关联联系人被归档或软删除时，Note 本身仍在索引中，搜索结果中的 `$note->contact` 可能返回 `null` 或被归档的联系人
- **Group**：有 `SoftDeletes` 但没有 `shouldBeSearchable()`，软删除时行为同 Contact（因 `scout.soft_delete = false` 而从索引移除），但没有归档机制

### 5.4 问题总结与改进建议

| 场景 | 当前行为 | 是否合理 | 建议改进 |
|------|----------|----------|----------|
| 归档联系人出现在 Scout 搜索 | 不会出现 | ✅ 合理 | — |
| 归档联系人出现在最常查看 | 会出现 | ❌ 不合理 | `VaultMostConsultedViewHelper` 应过滤 `listed = false` 的联系人 |
| 软删除联系人出现在 Scout 搜索 | 不会出现 | ✅ 合理 | — |
| 软删除联系人出现在最常查看 | 不会出现，但导致运行时错误 | ❌ 有 Bug | `Contact::find()` 返回 null 时需防护，或改用 `Contact::withTrashed()->find()` 后检查 `trashed()` |
| Note 的关联联系人被归档 | Note 仍可搜索，但 contact 信息可能不完整 | ⚠️ 边界情况 | 搜索结果中应检查 `$note->contact` 是否为 null 或 `listed = false` |
| contact_vault_user 孤儿记录 | 联系人删除后因 FK 级联删除自动清理 | ✅ 合理 | — |

---

## 6. 整体架构图

```
┌─────────────────────────────────────────────────────────────┐
│                        Routes (web.php)                      │
│  GET  /vaults/{id}/search                     → VSC::index  │
│  POST /vaults/{id}/search                     → VSC::show   │
│  GET  /vaults/{id}/search/user/contact/mostConsulted         │
│                                               → VMCC::index │
│  POST /vaults/{id}/search/user/contacts       → VCSC::index │
└──────────────────────┬──────────────────────────────────────┘
                       │
          ┌────────────┼────────────────┐
          ▼            ▼                ▼
   VaultSearch    VaultContact    VaultMostConsulted
   Controller     SearchController ContactsController
   (Contact+Note  (Contact only)  (no Scout)
    +Group)                        (DB query)
          │            │                │
          ▼            ▼                ▼
   VaultSearch    VaultContact     VaultMostConsulted
   IndexView      SearchView       ViewHelper
   Helper         Helper           ┌──────────────────┐
   ┌─────────┐    ┌──────────┐     │ contact_vault_user│
   │  Scout   │    │  Scout   │     │ .where(vault_id)  │
   │  Search  │    │  Search  │     │ .where(user_id)   │
   │  across  │    │  Contact │     │ .orderBy(views)   │
   │  3 models│    │  only    │     │ .limit(5)         │
   └────┬────┘    └────┬─────┘     └────────┬─────────┘
        │              │                     │
        ▼              ▼                     ▼
   ┌─────────────────────────┐        Contact::find()
   │   Scout Engine Layer    │              │
   │ (Algolia/Meilisearch/   │              ▼
   │  Typesense/Database)    │         Contact Model
   │                         │        (skip soft-deleted)
   │  Contact::search($term) │
   │  ->where('vault_id')    │
   │  Note::search($term)    │
   │  ->where('vault_id')    │
   │  Group::search($term)   │
   │  ->where('vault_id')    │
   └─────────────────────────┘

                    ┌──────────────────────────────┐
                    │     Index Data Source         │
                    │                              │
                    │  Contact.toSearchableArray() │
                    │    → listed = true 才索引     │
                    │    → 5个姓名字段              │
                    │  Note.toSearchableArray()     │
                    │    → 始终索引                 │
                    │    → title + body             │
                    │  Group.toSearchableArray()    │
                    │    → 始终索引                 │
                    │    → name                     │
                    │                              │
                    │  ScoutHelper::isActivated()  │
                    │    → 控制索引是否同步          │
                    │  ScoutHelper::isFullTextIndex()│
                    │    → 控制DB全文索引创建        │
                    └──────────────────────────────┘
```
