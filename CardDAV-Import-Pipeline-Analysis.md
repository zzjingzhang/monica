# CardDAV → vCard → 本地联系人导入管线分析

## 1. 管线总览

从 DAV 客户端写入 vCard 到本地联系人、联系方式和重要日期被创建或更新，整个导入管线分为五个核心阶段：

```
DAV PUT/POST 请求
  → CardDAVBackend::createCard / updateCard   （入口 + 权限校验）
    → UpdateVCard Job                          （异步队列 + 参数校验 + ETag 检查）
      → ImportVCard Service                    （解析 vCard + 行为选择 + 按序调度 Importer）
        → Importer 子类链式执行                  （落地具体字段）
          → CreateContact / UpdateContact 等服务层  （验证 + Vault 边界守卫）
```

---

## 2. 第一层：CardDAVBackend — 入口与权限守卫

**文件**：[CardDAVBackend.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/Dav/Web/Backend/CardDAV/CardDAVBackend.php)

### 2.1 createCard（[L408-L411](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/Dav/Web/Backend/CardDAV/CardDAVBackend.php#L408-L411)）

```php
public function createCard($addressBookId, $cardUri, $cardData): ?string
{
    return $this->updateCard($addressBookId, $cardUri, $cardData);
}
```

`createCard` 直接委托给 `updateCard`，不区分创建与更新——由下游的 `ImportContact` 来判断联系人是否存在。

### 2.2 updateCard（[L437-L456](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/Dav/Web/Backend/CardDAV/CardDAVBackend.php#L437-L456)）

```php
public function updateCard($addressBookId, $cardUri, $cardData): ?string
{
    $vault = $this->vaults($addressBookId, permission: Vault::PERMISSION_EDIT)
        ->firstOrFail();

    $job = new UpdateVCard([
        'account_id' => $this->user->account_id,
        'author_id' => $this->user->id,
        'vault_id' => $vault->id,
        'uri' => $cardUri,
        'card' => $cardData,
    ]);

    Bus::batch([$job])
        ->allowFailures()
        ->onQueue('high')
        ->dispatch();

    return null;
}
```

**关键行为**：

| 步骤 | 说明 |
|------|------|
| `$this->vaults($addressBookId, Vault::PERMISSION_EDIT)` | 通过 `GetVaults` trait 获取用户有**编辑权限**的 vault，若不存在则 `firstOrFail()` 抛出 404 |
| `new UpdateVCard([...])` | 构造 Job 实例，传入 `account_id`、`author_id`、`vault_id`、`uri`（card URI）、`card`（vCard 原始字符串） |
| `Bus::batch([$job])` | 使用 Laravel Bus Batch 异步调度到 `high` 优先级队列；`allowFailures()` 意味着即使失败也不阻塞其他 batch job |
| `return null` | 不返回 ETag——DAV 协议允许返回 null，客户端将通过后续同步获得最新 ETag |

**注意**：从本地 DAV 写入时，`etag` 和 `external` 字段**未传入**，这两个参数仅在远程同步（`GetVCard`/`GetMultipleVCard`）时才会携带。

---

## 3. 第二层：UpdateVCard Job — 异步执行与 ETag 检查

**文件**：[UpdateVCard.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/Dav/Jobs/UpdateVCard.php)

`UpdateVCard` 继承 [QueuableService](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Services/QueuableService.php)，后者扩展 `BaseService` 并实现 `ShouldQueue`，使其既可作为服务直接调用，也可作为队列 Job 调度。

### 3.1 验证规则（[L25-L40](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/Dav/Jobs/UpdateVCard.php#L25-L40)）

```php
public function rules(): array
{
    return [
        'account_id' => 'required|uuid|exists:accounts,id',
        'author_id' => 'required|uuid|exists:users,id',
        'vault_id' => 'required|uuid|exists:vaults,id',
        'uri' => 'required|string',
        'etag' => 'nullable|string',
        'external' => 'nullable|boolean',
        'card' => ['required', function ($attribute, $value, $fail) {
            if (!is_string($value) && !is_resource($value)) {
                $fail($attribute.' must be a string or a resource.');
            }
        }],
    ];
}
```

### 3.2 权限守卫（[L42-L53](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/Dav/Jobs/UpdateVCard.php#L42-L53)）

```php
public function permissions(): array
{
    return [
        'author_must_belong_to_account',
        'vault_must_belong_to_account',
        'author_must_be_in_vault',
        'author_must_be_vault_editor',
    ];
}
```

这些权限在 `BaseService::validateRules()` 中按依赖顺序逐一校验（参见第 6 节），确保：
- `author` 确实属于该 `account`
- `vault` 确实属于该 `account`
- `author` 在该 `vault` 中有成员身份
- `author` 在该 `vault` 中至少有 `EDIT` 权限

### 3.3 execute 方法（[L55-L74](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/Dav/Jobs/UpdateVCard.php#L55-L74)）

```php
public function execute(array $data): void
{
    $this->data = $data;
    $this->validateRules($data);

    $this->withLocale($this->author->preferredLocale(), function () {
        $newtag = $this->updateCard($this->data['uri'], $this->data['card']);

        if ($newtag !== null && ($etag = Arr::get($this->data, 'etag')) !== null && $newtag !== $etag) {
            Log::channel('database')->warning(
                __CLASS__.' '.__FUNCTION__." wrong etag when updating contact. Expected [$etag], got [$newtag]",
                ['contacturl' => $this->data['uri'], 'carddata' => $this->data['card']]
            );
        }
    });
}
```

**ETag 检查流程**：

1. 调用 `updateCard()` 执行实际导入，返回重新计算的 ETag (`newtag`)
2. 若原始请求携带了 `etag`（仅远程同步场景），且 `newtag !== etag`，则记录**警告日志**
3. **ETag 不匹配不会阻止更新**——仅记录日志，保证最终一致性由后续同步机制修正

### 3.4 updateCard 私有方法（[L76-L112](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/Dav/Jobs/UpdateVCard.php#L76-L112)）

```php
private function updateCard(string $uri, mixed $card): ?string
{
    try {
        $result = app(ImportVCard::class)->execute([
            'account_id' => $this->author->account_id,
            'author_id' => $this->author->id,
            'vault_id' => $this->vault->id,
            'entry' => $card,
            'etag' => Arr::get($this->data, 'etag'),
            'uri' => $uri,
            'external' => Arr::get($this->data, 'external', false),
            'behaviour' => ImportVCard::BEHAVIOUR_REPLACE,
        ]);

        if (!Arr::has($result, 'error')) {
            return app(GetEtag::class)->execute([
                'account_id' => $this->author->account_id,
                'author_id' => $this->author->id,
                'vault_id' => $this->vault->id,
                'vcard' => $result['entry'],
            ]);
        }
    } catch (\Exception $e) {
        Log::channel('database')->error(__CLASS__.' '.__FUNCTION__.': '.$e->getMessage(), [...]);
        throw $e;
    }
    return null;
}
```

**关键决策**：`behaviour` 被硬编码为 `BEHAVIOUR_REPLACE`。这意味着从 DAV 写入时，始终使用**替换模式**——用 vCard 中的数据完全覆盖本地已有数据。

**导入成功后**：通过 `GetEtag` 服务基于导入后的 `VCardResource` 计算新的 ETag，返回给上层进行比较。

---

## 4. 第三层：ImportVCard Service — 解析、行为选择与 Importer 调度

**文件**：[ImportVCard.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/Dav/Services/ImportVCard.php)

### 4.1 行为常量

```php
public const BEHAVIOUR_ADD = 'behaviour_add';       // 仅添加新数据，不覆盖已有
public const BEHAVIOUR_REPLACE = 'behaviour_replace'; // 用 vCard 数据替换本地已有数据
```

- `BEHAVIOUR_ADD`：用于 UI 批量导入场景（用户选择"添加"行为），跳过已存在的联系人
- `BEHAVIOUR_REPLACE`：用于 DAV 同步场景，强制用远端 vCard 覆盖本地

> **注意**：当前代码中 `behaviour` 参数虽然被校验，但在实际 Importer 子类中并没有直接使用它来改变行为。`ImportContact` 通过 `getExistingContact()` 查找现有联系人后自行决定 create 或 update。`BEHAVIOUR_ADD` 的真正影响在 `ERROR_CONTACT_EXIST` 错误路径中（当前管线由 `UpdateVCard` 固定传入 `REPLACE`，所以不会触发此路径）。

### 4.2 execute 方法（[L109-L118](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/Dav/Services/ImportVCard.php#L109-L118)）

```php
public function execute(array $data): array
{
    $this->data = $data;
    $this->validateRules($data);
    $this->external = Arr::get($data, 'external', false);
    return $this->process($data);
}
```

**权限**（[L96-L104](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/Dav/Services/ImportVCard.php#L96-L104)）：

```php
public function permissions(): array
{
    return [
        'author_must_belong_to_account',
        'vault_must_belong_to_account',
        'author_must_be_in_vault',
        'author_must_be_vault_editor',
    ];
}
```

### 4.3 process → getEntry（[L125-L165](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/Dav/Services/ImportVCard.php#L125-L165)）

```
process($data)
  → getEntry($data)          // 解析 entry 为 VCard 对象
    → 若 entry 是字符串 → ReadVObject::execute() 解析
    → 若解析失败 → 返回 [null, $vcard]，触发 ERROR_PARSER
    → 若 entry 已是 VCard 实例 → 直接使用，序列化为字符串
  → 返回 [$entry(VCard), $vcard(string)]
```

[ReadVObject](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/Dav/Services/ReadVObject.php) 使用 `Sabre\VObject\Reader` 解析 vCard 字符串为 `VCard` 对象。

### 4.4 canImportCurrentEntry（[L217-L233](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/Dav/Services/ImportVCard.php#L217-L233)）

```php
private function canImportCurrentEntry(VCard $entry): bool
{
    $importers = $this->importers()
        ->filter(fn (ImportVCardResource $importer): bool => $importer->handle($entry));

    if ($importers->isEmpty()) {
        return false;
    }

    foreach ($importers as $importer) {
        if (!$importer->can($entry)) {
            return false;
        }
    }

    return true;
}
```

**两重检查**：
1. `handle($entry)` — 筛选能处理该 vCard KIND 的 Importer（`individual` vs `group`）
2. `can($entry)` — 检查 vCard 内容是否满足最低导入条件（例如 `ImportContact::can()` 要求有 FN 或 NICKNAME 或 N 中有名字）

### 4.5 importEntry — 按序执行 Importer 链（[L238-L250](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/Dav/Services/ImportVCard.php#L238-L250)）

```php
private function importEntry(VCard $entry): ?VCardResource
{
    $result = null;

    $importers = $this->importers()
        ->filter(fn (ImportVCardResource $importer): bool => $importer->handle($entry));

    foreach ($importers as $importer) {
        $result = $importer->import($entry, $result);
    }

    return $result;
}
```

**核心设计**：每个 Importer 的 `import()` 接收当前 VCard 和**上一个 Importer 的返回结果**，返回更新后的 VCardResource（Contact 或 Group）。这形成了一个**责任链模式**——前一个 Importer 创建出 Contact 对象后，后续 Importer 在同一 Contact 上追加联系方式、地址、重要日期等。

### 4.6 importers() — 发现与排序（[L257-L268](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/Dav/Services/ImportVCard.php#L257-L268)）

```php
private function importers(): Collection
{
    if (self::$importers === null) {
        self::$importers = collect(subClasses(ImportVCardResource::class))
            ->sortBy(fn (ReflectionClass $importer): int => Order::get($importer))
            ->map(fn (ReflectionClass $importer): ImportVCardResource => $importer->newInstance());
    }

    return self::$importers
        ->map(fn (ImportVCardResource $importer): ImportVCardResource => $importer->setContext($this));
}
```

**关键机制**：

1. `subClasses(ImportVCardResource::class)` — 通过反射发现所有实现 `ImportVCardResource` 的类
2. `sortBy(Order::get($importer))` — 按 `#[Order]` 属性排序
3. `newInstance()` — 创建 Importer 实例（静态缓存，只创建一次）
4. `setContext($this)` — 每次调用时注入当前 `ImportVCard` 上下文

**[Order](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/Dav/Order.php) 属性**是一个 PHP 8 Attribute：

```php
#[Attribute]
class Order
{
    public function __construct(public int $order) {}

    public static function get(ReflectionClass $class): int
    {
        $attributes = $class->getAttributes(static::class, ReflectionAttribute::IS_INSTANCEOF);
        return empty($attributes) ? 0 : $attributes[0]->newInstance()->order;
    }
}
```

### 4.7 完整 Importer 清单（按 Order 排序）

| Order | Importer 类 | 文件 | 处理的 KIND | 核心职责 |
|-------|------------|------|------------|---------|
| 1 | [ImportContact](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/ManageContact/Dav/ImportContact.php) | ManageContact/Dav | `individual` | 创建/更新联系人基本信息 |
| 1 | [ImportContactTask](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/ManageTasks/Dav/ImportContactTask.php) | ManageTasks/Dav | VCalendar (VTODO) | 创建/更新任务（**非 vCard 导入器**，属于 VCalendar 导入管线） |
| 10 | [ImportGroup](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/ManageGroups/Dav/ImportGroup.php) | ManageGroups/Dav | `group` | 创建/更新分组 |
| 11 | [ImportMembers](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/ManageGroups/Dav/ImportMembers.php) | ManageGroups/Dav | `group` | 导入分组成员 |
| 40 | [ImportContactInformation](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/ManageContactInformation/Dav/ImportContactInformation.php) | ManageContactInformation/Dav | `individual` | 导入联系方式 |
| 40 | [ImportImportantDates](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/ManageContactImportantDates/Dav/ImportImportantDates.php) | ManageContactImportantDates/Dav | `individual` | 导入重要日期 |
| 40 | [ImportAddress](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/ManageContact/Dav/ImportAddress.php) | ManageContact/Dav | `individual` | 导入地址 |
| 40 | [ImportLabels](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/ManageContact/Dav/ImportLabels.php) | ManageContact/Dav | `individual` | 导入标签 |

**执行顺序的重要性**：`ImportContact`（Order=1）必须最先执行，因为它创建 Contact 对象。后续 Order=40 的 Importer 依赖前一步产生的 Contact 实例来追加子资源。

对于 `individual` 类型的 vCard，执行链为：
```
ImportContact → ImportContactInformation → ImportImportantDates → ImportAddress → ImportLabels
```

对于 `group` 类型的 vCard，执行链为：
```
ImportGroup → ImportMembers
```

### 4.8 processEntryCard — 保存 vCard 缓存（[L189-L212](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/Dav/Services/ImportVCard.php#L189-L212)）

```php
private function processEntryCard(VCard $entry, string $vcard): array
{
    $result = $this->importEntry($entry);

    if ($result === null) {
        return ['error' => self::ERROR_CARD_NOT_IMPORTABLE, ...];
    }

    $result = $result->withoutTimestamps(function () use ($result, $vcard): mixed {
        $result->vcard = $vcard;
        $result->save();
        return $result;
    });

    return ['id' => $result->id, 'entry' => $result];
}
```

导入完成后，将原始 vCard 字符串缓存到 `Contact.vcard` 字段，使用 `withoutTimestamps` 避免触发 `updated_at` 变更。

---

## 5. 第四层：三大核心 Importer 的字段落地

### 5.1 ImportContact — 联系人基本信息

**文件**：[ImportContact.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/ManageContact/Dav/ImportContact.php)  |  Order = 1

#### 落地字段映射

| vCard 属性 | 本地字段 | 说明 |
|-----------|---------|------|
| `UID` | `id` | 仅非外部导入时设置，保证本地 DAV 写入时使用 vCard 中的 UUID |
| `N[0]` (姓氏) | `last_name` | |
| `N[1]` (名字) | `first_name` | |
| `N[2]` (中间名) | `middle_name` | |
| `NICKNAME` | `nickname` | |
| `FN` | `first_name` / `last_name` | 根据 `author.name_order` 拆分 |
| `GENDER` | `gender_id` | M→Male Gender, F→Female Gender, 其他→default Gender |
| — | `listed` | 新建时硬编码为 `true` |

#### 名字解析优先级

```
1. N 属性中第 1 部分非空 → importNameFromN()（最精确）
2. FN 属性非空 → importNameFromFN()（按 name_order 拆分）
3. NICKNAME 非空 → importNameFromNICKNAME()（作为 first_name）
```

#### 联系人存在性判断（[getExistingContact](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/ManageContact/Dav/ImportContact.php#L80-L111)）

```php
private function getExistingContact(VCard $vcard): ?Contact
{
    $backend = app(CardDAVBackend::class)->withUser($this->context->author);
    $contact = null;

    // 策略1: 通过 URI 查找
    if (($uri = Arr::get($this->context->data, 'uri')) !== null) {
        $contact = $backend->getObject((string) $this->vault()->id, $uri);
        if ($contact === null) {
            $contact = Contact::firstWhere([
                'vault_id' => $this->vault()->id,
                'distant_uri' => $uri,
            ]);
        }
    }

    // 策略2: 通过 UID 查找
    if ($contact === null) {
        $contactId = $this->getUid($vcard);
        if ($contactId !== null && Uuid::isValid($contactId)) {
            $contact = Contact::firstWhere([
                'vault_id' => $this->vault()->id,
                'id' => $contactId,
            ]);
        }
    }

    // 安全检查: 确保找到的联系人属于当前 vault
    if ($contact !== null && $contact->vault_id !== $this->vault()->id) {
        throw new ModelNotFoundException;
    }

    return $contact;
}
```

**查找策略**：
1. 先通过 `CardDAVBackend::getObject()` 按 URI 查找（该方法内部会对 URI 做 base64 解码等处理）
2. 若未找到，直接查 `distant_uri` 字段
3. 若仍未找到，尝试用 vCard UID 作为 Contact ID 查找
4. 最终验证 vault 一致性

#### Create vs Update 决策

```php
if ($contact === null) {
    $data['listed'] = true;
    $contact = app(CreateContact::class)->execute($data);
} elseif ($data !== $original) {
    $contact = app(UpdateContact::class)->execute($data);
}
```

- `contact === null` → 调用 `CreateContact`
- `contact !== null && $data !== $original`（即 vCard 数据与本地不同）→ 调用 `UpdateContact`
- `contact !== null && $data === $original`（数据无变化）→ 不调用服务层，跳过

#### 远程同步元数据写入

```php
if ($this->context->external && $contact->distant_uuid === null) {
    $contact->distant_uuid = $this->getUid($vcard);
    $contact->save();
}

return Contact::withoutTimestamps(function () use ($contact): Contact {
    $uri = Arr::get($this->context->data, 'uri');
    if ($this->context->external) {
        $contact->distant_etag = Arr::get($this->context->data, 'etag');
        $contact->distant_uri = $uri;
        $contact->save();
    }
    return $contact;
});
```

仅当 `external=true`（远程同步）时，写入 `distant_uuid`、`distant_etag`、`distant_uri`。使用 `withoutTimestamps` 避免触发 `updated_at` 更新。

---

### 5.2 ImportContactInformation — 联系方式

**文件**：[ImportContactInformation.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/ManageContactInformation/Dav/ImportContactInformation.php)  |  Order = 40

#### 处理的 vCard 属性

| vCard 属性 | 本地 ContactInformationType.type | 说明 |
|-----------|--------------------------------|------|
| `EMAIL` | `email` | 电子邮件 |
| `TEL` | `phone` | 电话号码 |
| `IMPP` | `IMPP` (含 `X-SERVICE-TYPE` 参数) | 即时通讯 |
| `URL` | 按属性名匹配 | 网站 URL |
| `X-SOCIAL-PROFILE` | `X-SOCIAL-PROFILE` (含 `TYPE` 参数) | 社交档案 |

#### 落地字段

每次创建或更新 `ContactInformation` 记录时，写入以下字段：

| 字段 | 来源 |
|------|------|
| `contact_information_type_id` | 由 `getTypeId()` 解析，必要时通过 `CreateContactInformationType` 自动创建 |
| `contact_information_kind` | vCard 属性的 `TYPE` 参数（如 "home", "work", "cell"） |
| `data` | vCard 属性值；`X-SOCIAL-PROFILE` 特殊处理——优先使用 `X-USER` 参数，否则取属性值 |

#### 同步算法（[importContacts](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/ManageContactInformation/Dav/ImportContactInformation.php#L76-L108)）

```php
for ($i = 0; $i < $cardInfos->count() || $i < $current->count(); $i++) {
    if ($i < $cardInfos->count()) {
        $typeId = $this->getTypeId($cardInfos[$i]);
        if ($typeId === null) continue;

        if ($i < $current->count()) {
            $this->updateContactInformation($contact, $current[$i], $cardInfos[$i], $typeId);
        } else {
            $this->createContactInformation($contact, $cardInfos[$i], $typeId);
        }
    } elseif ($i < $current->count()) {
        $this->removeContactInformation($contact, $current[$i]);
    }
}
```

**按位置索引对齐**：vCard 中同类型属性与本地 `ContactInformation` 按位置一一对应：
- vCard 有更多 → 多余的执行 `CreateContactInformation`
- 本地有更多 → 多余的执行 `DestroyContactInformation`
- 两边都有 → 若值或 kind 不同则执行 `UpdateContactInformation`

#### 类型解析（[getTypeId](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/ManageContactInformation/Dav/ImportContactInformation.php#L110-L155)）

```
EMAIL → account.contactInformationTypes where type='email'
TEL   → account.contactInformationTypes where type='phone'
IMPP  → 按 X-SERVICE-TYPE 参数匹配 name_translation_key
X-SOCIAL-PROFILE → 按 TYPE 参数匹配 name_translation_key
以上都找不到 → 尝试按属性名模糊匹配
仍然找不到 → 自动创建新的 ContactInformationType
```

---

### 5.3 ImportImportantDates — 重要日期

**文件**：[ImportImportantDates.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/ManageContactImportantDates/Dav/ImportImportantDates.php)  |  Order = 40

#### 处理的 vCard 属性

| vCard 属性 | 本地模型 | 说明 |
|-----------|---------|------|
| `BDAY` | `ContactImportantDate` | 生日 |

#### 落地字段

| 字段 | 来源 |
|------|------|
| `contact_important_date_type_id` | vault 中 `internal_type === TYPE_BIRTHDATE` 的类型 ID |
| `label` | 固定为 `trans('Birthday', [], $author->locale)` |
| `day` | BDAY 解析后的日期部分 |
| `month` | BDAY 解析后的月份部分 |
| `year` | BDAY 解析后的年份部分 |

#### BDAY 格式支持

通过 `Sabre\VObject\DateTimeParser::parseVCardDateTime()` 解析，支持以下格式：

| vCard BDAY 值 | day | month | year |
|--------------|-----|-------|------|
| `20251106` | 6 | 11 | 2025 |
| `2025` | null | null | 2025 |
| `202510` | null | 10 | 2025 |
| `2025-10` | null | 10 | 2025 |
| `--0415` | 15 | 4 | null |
| `--04-15` | 15 | 4 | null |

#### 同步算法（[import](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/ManageContactImportantDates/Dav/ImportImportantDates.php#L33-L64)）

```php
$contactImportantDates = $this->getImportantDates($contact);
$bdays = $this->getBday($vcard);

$toAdd = $bdays->diffKeys($contactImportantDates);
$toRemove = $contactImportantDates->diffKeys($bdays);
$intersect = $contactImportantDates->intersectByKeys($bdays);
```

**按日期值键值对齐**：使用 vCard 日期字符串作为 key，对比本地已有重要日期与 vCard 中的 BDAY：
- 仅在 vCard 中存在 → `CreateContactImportantDate`
- 仅在本地存在 → `DestroyContactImportantDate`
- 两边都有 → `UpdateContactImportantDate`（仅更新 `contact_important_date_type_id`，将无类型的日期关联到 birthdate 类型）

---

## 6. 第五层：服务层验证与 Vault 边界

所有 Importer 子类最终都通过调用服务层（如 `CreateContact`、`UpdateContact` 等）来持久化数据。这些服务层共享同一套验证和权限机制，由 [BaseService](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Services/BaseService.php) 提供。

### 6.1 BaseService 验证框架

`BaseService::validateRules($data)` 执行两步验证：

```
1. Validator::make($data, $this->rules())->validate()    // Laravel 标准表单验证
2. 逐条检查 permissions() → validatePermission()         // 业务权限验证
```

#### 权限依赖图

```
author_must_belong_to_account          ← 验证 author 属于 account
  ├─ author_must_be_account_administrator ← 验证 author 是管理员
  ├─ vault_must_belong_to_account       ← 验证 vault 属于 account
  │   ├─ author_must_be_vault_manager   ← 验证 author 在 vault 中有 MANAGE 权限
  │   ├─ author_must_be_vault_editor    ← 验证 author 在 vault 中有 EDIT 权限
  │   ├─ author_must_be_in_vault        ← 验证 author 在 vault 中有 VIEW 权限
  │   ├─ contact_must_belong_to_vault   ← 验证 contact 属于 vault
  │   └─ group_must_belong_to_vault     ← 验证 group 属于 vault
```

### 6.2 Vault 边界守卫详解

Vault 是 Monica 中的数据隔离单元。以下机制确保数据不越界：

#### 权限校验（`validateUserPermissionInVault`）

```php
public function validateUserPermissionInVault(int $permission): void
{
    $exists = $this->author->vaults()
        ->where('vaults.id', $this->vault->id)
        ->wherePivot('permission', '<=', $permission)
        ->exists();

    if (!$exists) {
        throw new NotEnoughPermissionException;
    }
}
```

通过 `contact_vault_user` 中间表的 `permission` 字段（数值越小权限越大）检查：
- `Vault::PERMISSION_MANAGE` = 管理员
- `Vault::PERMISSION_EDIT` = 编辑者
- `Vault::PERMISSION_VIEW` = 查看者

#### Contact 归属校验（`validateContactBelongsToVault`）

```php
public function validateContactBelongsToVault(array $data): void
{
    if (isset($data['contact_id'])) {
        $this->contact = $this->vault->contacts()
            ->findOrFail($data['contact_id']);

        if ($this->contact->vault_id !== $this->vault->id) {
            throw new ModelNotFoundException;
        }
    }
}
```

**双重检查**：先通过 `$this->vault->contacts()` 限定 vault 范围查找，再显式比对 `vault_id`。

### 6.3 各服务层的 Vault 边界保护

| 服务 | 权限要求 | Vault 边界检查 |
|------|---------|---------------|
| [CreateContact](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/ManageContact/Services/CreateContact.php) | `author_must_be_vault_editor` | `vault_must_belong_to_account` + editor 权限 |
| [UpdateContact](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/ManageContact/Services/UpdateContact.php) | `author_must_be_vault_editor` + `contact_must_belong_to_vault` | contact 必须属于 vault |
| [CreateContactInformation](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/ManageContactInformation/Services/CreateContactInformation.php) | `author_must_be_vault_editor` + `contact_must_belong_to_vault` | contact 必须属于 vault |
| [UpdateContactInformation](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/ManageContactInformation/Services/UpdateContactInformation.php) | `author_must_be_vault_editor` + `contact_must_belong_to_vault` | contact 必须属于 vault |
| [DestroyContactInformation](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/ManageContactInformation/Services/DestroyContactInformation.php) | `author_must_be_vault_editor` + `contact_must_belong_to_vault` | contact 必须属于 vault |
| [CreateContactImportantDate](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/ManageContactImportantDates/Services/CreateContactImportantDate.php) | `author_must_be_vault_editor` + `contact_must_belong_to_vault` | contact 必须属于 vault；`contact_important_date_type_id` 必须属于 vault |
| [UpdateContactImportantDate](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/ManageContactImportantDates/Services/UpdateContactImportantDate.php) | `author_must_be_vault_editor` + `contact_must_belong_to_vault` | 同上 |
| [DestroyContactImportantDate](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/ManageContactImportantDates/Services/DestroyContactImportantDate.php) | `author_must_be_vault_editor` + `contact_must_belong_to_vault` | 同上 |
| [CreateAddress](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Vault/ManageAddresses/Services/CreateAddress.php) | `author_must_be_vault_editor` | vault 归属 + editor 权限 |
| [UpdateAddress](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Vault/ManageAddresses/Services/UpdateAddress.php) | `author_must_be_vault_editor` | vault 归属 + editor 权限 |

### 6.4 多层 Vault 边界防护

Vault 边界在管线中至少被检查了**三次**：

1. **CardDAVBackend::updateCard** — `$this->vaults($addressBookId, Vault::PERMISSION_EDIT)->firstOrFail()`
2. **UpdateVCard::validateRules** — `author_must_be_in_vault` + `author_must_be_vault_editor`
3. **各服务层 CreateContact / UpdateContact 等** — `vault_must_belong_to_account` + `contact_must_belong_to_vault`

这种**纵深防御**确保即使某一层被绕过，下游服务层仍能拦截越界操作。

---

## 7. 完整数据流图

```
┌─────────────────────────────────────────────────────────────────────┐
│  DAV Client (PUT/POST)                                              │
│  vCard 数据: FN, N, EMAIL, TEL, BDAY, ADR, CATEGORIES, GENDER...  │
└────────────────────────────┬────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────────┐
│  CardDAVBackend::createCard / updateCard                            │
│  ├─ vaults($addressBookId, PERMISSION_EDIT) → 权限校验              │
│  ├─ 构造 UpdateVCard Job（不含 etag, external）                      │
│  └─ Bus::batch([$job])->onQueue('high')->dispatch()                 │
└────────────────────────────┬────────────────────────────────────────┘
                             │ 异步队列
                             ▼
┌─────────────────────────────────────────────────────────────────────┐
│  UpdateVCard::handle() → execute()                                  │
│  ├─ validateRules() → 4 项权限检查                                  │
│  ├─ withLocale() 包裹执行                                           │
│  ├─ updateCard(uri, card)                                           │
│  │   ├─ ImportVCard::execute([...behaviour=REPLACE...])             │
│  │   └─ GetEtag::execute() → 计算新 ETag                            │
│  └─ ETag 不匹配 → Log::warning()（不阻止更新）                       │
└────────────────────────────┬────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────────┐
│  ImportVCard::execute()                                             │
│  ├─ validateRules() → 4 项权限检查                                  │
│  ├─ getEntry() → ReadVObject 解析 VCard 对象                        │
│  ├─ canImportCurrentEntry() → handle() + can() 双重检查             │
│  └─ importEntry() → 按 Order 排序的 Importer 链式执行               │
└────────────────────────────┬────────────────────────────────────────┘
                             │
            ┌────────────────┼────────────────┐
            │                │                │
            ▼                ▼                ▼
   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
   │ Order=1      │  │ Order=10/11  │  │ Order=40     │
   │ ImportContact│  │ ImportGroup  │  │ ImportCI/ID/ │
   │              │  │ ImportMembers│  │ Addr/Labels  │
   └──────┬───────┘  └──────────────┘  └──────┬───────┘
          │                                    │
          ▼                                    ▼
   ┌──────────────┐                    ┌──────────────────┐
   │ CreateContact│                    │ CreateCI /        │
   │ UpdateContact│                    │ UpdateCI /        │
   │              │                    │ DestroyCI /       │
   │ 验证:        │                    │ CreateCID /       │
   │ · author∈acc │                    │ UpdateCID /       │
   │ · vault∈acc  │                    │ DestroyCID /      │
   │ · editor权限 │                    │ CreateAddr / ...  │
   │ · contact∈v  │                    │                   │
   └──────────────┘                    │ 验证:             │
                                       │ · author∈acc      │
                                       │ · vault∈acc       │
                                       │ · contact∈vault   │
                                       │ · editor权限      │
                                       └──────────────────┘
```

---

## 8. 关键设计模式总结

| 模式 | 应用位置 | 优势 |
|------|---------|------|
| **统一入口** | `createCard` 委托给 `updateCard` | 简化逻辑，避免创建/更新的重复代码 |
| **异步队列** | `Bus::batch([UpdateVCard])` | DAV 请求立即返回，导入逻辑异步执行 |
| **责任链** | Importer 链式执行，`$result` 在链中传递 | 解耦各维度数据导入，支持扩展新 Importer |
| **Order 属性排序** | `#[Order(N)]` PHP Attribute | 声明式排序，保证 ImportContact 先于子资源 Importer |
| **服务层复用** | Importer 调用 CreateContact/UpdateContact 等 | 不绕过验证，保持 Vault 边界一致性 |
| **纵深防御** | CardDAVBackend → UpdateVCard → ImportVCard → 服务层，四层权限检查 | 即使某层被绕过，下游仍能拦截 |
| **按位置对齐同步** | ImportContactInformation 中 vCard 属性与本地记录按索引对齐 | 简单直观的同步策略，适用于无唯一标识的子资源 |
| **按值键对齐同步** | ImportImportantDates 中按日期字符串 diffKeys | 适用于有自然键的子资源，能检测增删改 |
| **ETag 最终一致性** | 不匹配仅警告不阻止 | 保证可用性，由后续同步修正 |

---

## 9. 关键文件索引

| 层次 | 文件 | 核心职责 |
|------|------|---------|
| DAV 入口 | [CardDAVBackend.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/Dav/Web/Backend/CardDAV/CardDAVBackend.php) | SabreDAV 后端适配，权限校验，构造 UpdateVCard Job |
| 异步 Job | [UpdateVCard.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/Dav/Jobs/UpdateVCard.php) | 队列执行，ETag 检查，调用 ImportVCard |
| 导入服务 | [ImportVCard.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/Dav/Services/ImportVCard.php) | 解析 vCard，按序调度 Importer 链 |
| Order 属性 | [Order.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/Dav/Order.php) | PHP Attribute，声明式排序 |
| Importer 基类 | [Importer.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/Dav/Importer.php) | 提供 account/vault/author 上下文，kind() 判断 |
| ImportVCardResource 接口 | [ImportVCardResource.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/Dav/ImportVCardResource.php) | handle/can/import 三方法契约 |
| 联系人导入 | [ImportContact.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/ManageContact/Dav/ImportContact.php) | 创建/更新联系人基本信息 |
| 联系方式导入 | [ImportContactInformation.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/ManageContactInformation/Dav/ImportContactInformation.php) | 同步 EMAIL/TEL/IMPP/URL/X-SOCIAL-PROFILE |
| 重要日期导入 | [ImportImportantDates.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/ManageContactImportantDates/Dav/ImportImportantDates.php) | 同步 BDAY → ContactImportantDate |
| 地址导入 | [ImportAddress.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/ManageContact/Dav/ImportAddress.php) | 同步 ADR → Address |
| 标签导入 | [ImportLabels.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/ManageContact/Dav/ImportLabels.php) | 同步 CATEGORIES → Label |
| 分组导入 | [ImportGroup.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/ManageGroups/Dav/ImportGroup.php) | 创建/更新分组 |
| 成员导入 | [ImportMembers.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/ManageGroups/Dav/ImportMembers.php) | 同步 MEMBER → Group 成员关系 |
| 基础服务 | [BaseService.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Services/BaseService.php) | 通用验证框架，权限检查，Vault 边界守卫 |
| 可队列服务 | [QueuableService.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Services/QueuableService.php) | BaseService + ShouldQueue，支持同步/异步双模式 |
| 创建联系人 | [CreateContact.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/ManageContact/Services/CreateContact.php) | 验证 + 创建 Contact + FeedItem |
| 更新联系人 | [UpdateContact.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/ManageContact/Services/UpdateContact.php) | 验证 + 更新 Contact + FeedItem |
| 创建联系方式 | [CreateContactInformation.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/ManageContactInformation/Services/CreateContactInformation.php) | 验证 + 创建 ContactInformation |
| 更新联系方式 | [UpdateContactInformation.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/ManageContactInformation/Services/UpdateContactInformation.php) | 验证 + 更新 ContactInformation |
| 创建重要日期 | [CreateContactImportantDate.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/ManageContactImportantDates/Services/CreateContactImportantDate.php) | 验证 + 创建 ContactImportantDate |
| ETag 计算 | [GetEtag.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/Dav/Services/GetEtag.php) | 基于 vCard 内容计算 SHA-256 ETag |
