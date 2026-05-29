# DAV服务端架构分析

本文档详细解释Monica内置DAV服务端如何将Laravel认证、Vault权限、CardDAV地址簿、联系人vCard和同步令牌连接起来。

---

## 1. 概述

Monica的DAV服务端基于SabreDAV库构建，通过一系列自定义后端（Backend）和插件（Plugin）将Laravel的认证体系、Vault权限系统与CardDAV协议无缝集成。核心架构如下：

```
┌─────────────────────────────────────────────────────────────────┐
│                      DAVServiceProvider                          │
│  ┌─────────────┐  ┌──────────────────────────────────────────┐  │
│  │   Nodes     │  │               Plugins                    │  │
│  │ - Principal │  │ - AuthPlugin → AuthBackend               │  │
│  │ - AddressBook│ │ - CardDAVPlugin                          │  │
│  │ - Calendar  │  │ - SyncPlugin → SyncDAVBackend            │  │
│  └─────────────┘  │ - ACLPlugin                              │  │
│                   └──────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────┐
│                        CardDAVBackend                           │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌────────────────┐  │
│  │ GetVaults│  │  CRUD    │  │  ETag    │  │ SyncDAVBackend │  │
│  │  权限控制 │  │  操作    │  │  生成    │  │  增量同步      │  │
│  └──────────┘  └──────────┘  └──────────┘  └────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. DAVServiceProvider - 注册节点和插件

[DAVServiceProvider.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Providers/DAVServiceProvider.php) 是DAV功能的核心配置文件，负责注册DAV节点和Sabre插件。

### 2.1 节点（Nodes）注册

在 [nodes()](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Providers/DAVServiceProvider.php#L42-L56) 方法中注册三个核心节点：

```php
private function nodes(): array
{
    $user = Auth::user();

    $principalBackend = app(PrincipalBackend::class, ['user' => $user]);
    $carddavBackend = app(CardDAVBackend::class)->withUser($user);
    $caldavBackend = app(CalDAVBackend::class)->withUser($user);

    return [
        new PrincipalCollection($principalBackend),
        new AddressBookRoot($principalBackend, $carddavBackend),
        new CalendarRoot($principalBackend, $caldavBackend),
    ];
}
```

**节点说明：**

| 节点类型 | 作用 | 依赖后端 |
|---------|------|---------|
| `PrincipalCollection` | 管理用户主体（Principal），DAV ACL的基础 | [PrincipalBackend](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/Dav/Web/DAVACL/PrincipalBackend.php) |
| `AddressBookRoot` | CardDAV地址簿根节点，对外暴露为地址簿集合 | [CardDAVBackend](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/Dav/Web/Backend/CardDAV/CardDAVBackend.php) |
| `CalendarRoot` | CalDAV日历根节点 | `CalDAVBackend` |

**关键设计：**
- 每个后端通过 `withUser($user)` 方法注入当前认证用户
- 使用Laravel的服务容器解析后端实例，支持依赖注入
- [PrincipalBackend::getPrincipalUser()](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/Dav/Web/DAVACL/PrincipalBackend.php#L27-L30) 将用户映射为 `principals/{email}` 格式的DAV主体URI

### 2.2 插件（Plugins）注册

在 [plugins()](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Providers/DAVServiceProvider.php#L63-L92) 方法中注册SabreDAV插件：

```php
private function plugins()
{
    // 认证插件
    $authBackend = new AuthBackend;
    yield new AuthPlugin($authBackend);

    // CardDAV插件
    yield new CardDAVPlugin;
    yield new VCFExportPlugin;

    // CalDAV插件
    yield new CalDAVPlugin;
    yield new ICSExportPlugin;

    // 同步插件 (RFC 6578)
    yield new SyncPlugin;

    // ACL插件
    $aclPlugin = new AclPlugin;
    $aclPlugin->allowUnauthenticatedAccess = false;
    $aclPlugin->hideNodesFromListings = true;
    yield $aclPlugin;

    // 环境相关插件
    if (App::environment('local')) {
        yield new BrowserPlugin(false);
    } else {
        yield new DAVRedirect;
    }
}
```

**插件说明：**

| 插件 | 作用 | 关键配置 |
|-----|------|---------|
| `AuthPlugin` | 处理DAV认证，使用自定义 [AuthBackend](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/Dav/Web/Auth/AuthBackend.php) | 复用Laravel认证 |
| `CardDAVPlugin` | 实现CardDAV协议支持 | - |
| `VCFExportPlugin` | 支持vCard导出功能 | - |
| `SyncPlugin` | 实现RFC 6578同步报告，支持增量同步 | - |
| `AclPlugin` | 访问控制列表 | `allowUnauthenticatedAccess = false` 禁止匿名访问<br>`hideNodesFromListings = true` 无权限节点不可见 |

---

## 3. AuthBackend - 复用Laravel认证

[AuthBackend.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/Dav/Web/Auth/AuthBackend.php) 实现了SabreDAV的 `BackendInterface`，将Laravel的认证体系桥接到DAV服务。

### 3.1 核心实现

```php
class AuthBackend implements BackendInterface
{
    public function check(RequestInterface $request, ResponseInterface $response): array
    {
        if (! Auth::check()) {
            return [false, 'User is not authenticated'];
        }

        return [true, PrincipalBackend::getPrincipalUser(Auth::user())];
    }

    public function challenge(RequestInterface $request, ResponseInterface $response): void
    {
        $auth = new \Sabre\HTTP\Auth\Bearer(
            $this->realm,
            $request,
            $response
        );
        $auth->requireLogin();
    }
}
```

### 3.2 认证流程

1. **认证检查**：`check()` 方法调用Laravel的 `Auth::check()` 验证用户是否已通过Laravel认证（通常通过API token或session）
2. **主体映射**：认证通过后，返回DAV主体URI `principals/{user_email}`
3. **认证挑战**：未认证时，`challenge()` 方法设置 `WWW-Authenticate: Bearer` 响应头，返回401状态码

**关键设计：**
- 不处理具体的认证逻辑（如密码验证、token解析），完全复用Laravel已有的认证体系
- 支持Bearer认证方式，适合API客户端
- 返回的主体URI与 [PrincipalBackend](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/Dav/Web/DAVACL/PrincipalBackend.php) 保持一致，为后续权限检查提供基础

---

## 4. GetVaults - 按权限返回地址簿

[GetVaults.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/Dav/Web/Backend/GetVaults.php) 是一个Trait，被 `CardDAVBackend` 使用，负责根据用户权限过滤Vault（地址簿）。

### 4.1 核心实现

```php
trait GetVaults
{
    public function vaults(?string $collectionId = null, int $permission = Vault::PERMISSION_VIEW): Collection
    {
        $vaults = $this->user->vaults()
            ->wherePivot('permission', '<=', $permission);

        if ($collectionId !== null) {
            $vaults = $vaults->where('id', $collectionId);
        }

        return $vaults->get();
    }
}
```

### 4.2 权限模型

[Vault.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Models/Vault.php) 中定义了三级权限：

```php
public const PERMISSION_VIEW = 300;    // 查看
public const PERMISSION_EDIT = 200;    // 编辑
public const PERMISSION_MANAGE = 100;  // 管理
```

**权限过滤逻辑：**
- 使用 `wherePivot('permission', '<=', $permission)` 进行权限过滤
- 数值越小，权限越高：`MANAGE(100) < EDIT(200) < VIEW(300)`
- 请求 `<= PERMISSION_VIEW` 返回所有有权限的Vault
- 请求 `<= PERMISSION_EDIT` 仅返回有编辑或管理权限的Vault

### 4.3 地址簿映射

在 [CardDAVBackend::getAddressBooksForUser()](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/Dav/Web/Backend/CardDAV/CardDAVBackend.php#L78-L83) 中，Vault被映射为CardDAV地址簿：

```php
public function getAddressBooksForUser($principalUri): array
{
    return $this->vaults()
        ->map(fn (Vault $vault) => $this->getAddressBookDetails($vault))
        ->toArray();
}
```

**地址簿属性映射：**

| DAV属性 | 来源 | 说明 |
|--------|------|------|
| `id` | `$vault->id` | Vault的UUID |
| `uri` | `$vault->name` | 地址簿URL路径 |
| `principaluri` | `principals/{email}` | 所属用户主体 |
| `{DAV:}displayname` | `$vault->name` | 显示名称 |
| `{DAV:}sync-token` | `SyncToken->id` | 同步令牌 |
| `{http://calendarserver.org/ns/}getctag` | `SYNCTOKEN_PREFIX{id}` | 集合变更标记 |

---

## 5. CardDAVBackend - CRUD操作与领域服务

[CardDAVBackend.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/Dav/Web/Backend/CardDAV/CardDAVBackend.php) 继承自SabreDAV的 `AbstractBackend`，实现了CardDAV协议的核心操作。它使用了三个Trait：
- `GetVaults` - 权限控制
- `SyncDAVBackend` - 同步支持
- `WithUser` - 用户上下文

### 5.1 数据模型基础

[IDavResource](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/Dav/IDavResource.php) 是资源接口，[VCardResource](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/Dav/VCardResource.php) 是抽象基类，被 `Contact` 和 `Group` 模型继承：

```php
abstract class VCardResource extends Model implements IDavResource
{
    use SoftDeletes;
}
```

**关键属性：**
- `vcard` - 缓存的vCard序列化字符串
- `distant_etag` - 外部服务的ETag（用于双向同步）
- `distant_uuid` - 外部UUID
- `created_at` / `updated_at` / `deleted_at` - 时间戳（用于同步检测）

### 5.2 读取（Read）操作

#### 5.2.1 获取单个Card - [getCard()](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/Dav/Web/Backend/CardDAV/CardDAVBackend.php#L374-L382)

```php
public function getCard($addressBookId, $cardUri)
{
    $card = $this->getObject($addressBookId, $cardUri);
    return $card === null ? false : $this->prepareCard($card);
}
```

#### 5.2.2 获取多个Card - [getCards()](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/Dav/Web/Backend/CardDAV/CardDAVBackend.php#L352-L360)

```php
public function getCards($addressbookId): array
{
    $cards = $this->getObjects($addressbookId);
    return $cards
        ->map(fn (VCardResource $card) => $this->prepareCard($card))
        ->toArray();
}
```

#### 5.2.3 准备Card数据 - [prepareCard()](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/Dav/Web/Backend/CardDAV/CardDAVBackend.php#L180-L215)

```php
public function prepareCard(VCardResource $resource): array
{
    $carddata = $resource->vcard;
    $timestamp = $this->rev($carddata);

    if ($carddata === null || empty($carddata) || $timestamp === null || $timestamp < $resource->updated_at) {
        $carddata = $this->refreshObject($resource);
    }

    $etag = app(GetEtag::class)->execute([
        'account_id' => $this->user->account_id,
        'author_id' => $this->user->id,
        'vault_id' => $resource->vault_id,
        'vcard' => $resource->refresh(),
    ]);

    return [
        'contact_id' => $resource->id,
        'uri' => $this->encodeUri($resource),
        'carddata' => $carddata,
        'etag' => $etag,
        'distant_etag' => $resource->distant_etag,
        'lastmodified' => $resource->updated_at->timestamp,
    ];
}
```

**读取流程调用的领域服务：**

| 服务 | 作用 | 文件 |
|-----|------|------|
| [ReadVObject](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/Dav/Services/ReadVObject.php) | 解析vCard字符串，读取REV字段判断缓存有效性 | `rev()` 方法调用 |
| [ExportVCard](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/Dav/Services/ExportVCard.php) | 从数据库导出完整vCard（缓存失效时） | `refreshObject()` 方法调用 |
| [GetEtag](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/Dav/Services/GetEtag.php) | 计算vCard的ETag哈希值 | 直接调用 |

**智能缓存机制：**
1. 优先使用数据库缓存的 `$resource->vcard`
2. 通过 `REV` 字段（vCard修订时间）与 `updated_at` 比较判断缓存是否有效
3. 缓存失效时重新导出vCard并更新缓存

### 5.3 创建（Create）操作

[createCard()](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/Dav/Web/Backend/CardDAV/CardDAVBackend.php#L408-L411) 直接委托给 `updateCard()`：

```php
public function createCard($addressBookId, $cardUri, $cardData): ?string
{
    return $this->updateCard($addressBookId, $cardUri, $cardData);
}
```

### 5.4 更新（Update）操作

[updateCard()](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/Dav/Web/Backend/CardDAV/CardDAVBackend.php#L437-L456) 处理Card的创建和更新：

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

**更新流程调用的领域服务/作业：**

| 组件 | 作用 | 文件 |
|-----|------|------|
| [GetVaults::vaults()](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/Dav/Web/Backend/GetVaults.php#L15-L25) | 权限检查，要求 `PERMISSION_EDIT` | 方法第一行 |
| [UpdateVCard](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/Dav/Jobs/UpdateVCard.php) | 异步处理vCard导入的队列作业 | 核心处理逻辑 |
| [ImportVCard](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/Dav/Services/ImportVCard.php) | 解析vCard并写入数据库 | UpdateVCard内部调用 |
| [GetEtag](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/Dav/Services/GetEtag.php) | 导入完成后计算新的ETag | UpdateVCard内部调用 |

**关键设计：**
- **异步处理**：通过队列异步执行，避免DAV请求阻塞
- **权限前置**：在排队前就检查 `PERMISSION_EDIT` 权限
- **ETag检查**：在 [UpdateVCard::execute()](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/Dav/Jobs/UpdateVCard.php#L58-L74) 中验证客户端传入的ETag

### 5.5 删除（Delete）操作

[deleteCard()](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/Dav/Web/Backend/CardDAV/CardDAVBackend.php#L464-L489) 处理Card删除：

```php
public function deleteCard($addressBookId, $cardUri): bool
{
    $obj = $this->getObject($addressBookId, $cardUri);

    if ($obj !== null && $obj instanceof Contact) {
        DestroyContact::dispatch([
            'account_id' => $this->user->account_id,
            'author_id' => $this->user->id,
            'vault_id' => $obj->vault_id,
            'contact_id' => $obj->id,
        ])->onQueue('high');
        return true;
    } elseif ($obj !== null && $obj instanceof Group) {
        DestroyGroup::dispatch([
            'account_id' => $this->user->account_id,
            'author_id' => $this->user->id,
            'vault_id' => $obj->vault_id,
            'group_id' => $obj->id,
        ])->onQueue('high');
        return true;
    }

    return false;
}
```

**删除流程调用的领域服务：**

| 作业 | 作用 | 类型 |
|-----|------|------|
| `DestroyContact` | 软删除联系人 | 队列作业 |
| `DestroyGroup` | 软删除分组 | 队列作业 |

**关键设计：**
- 支持两种资源类型：`Contact` 和 `Group`
- 使用软删除（`SoftDeletes`），保留删除记录供同步使用
- 异步执行，保持DAV响应快速

---

## 6. SyncDAVBackend - 同步令牌与变化记录

[SyncDAVBackend.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/Dav/Web/Backend/SyncDAVBackend.php) 是一个Trait，实现了RFC 6578同步协议，支持增量同步。

### 6.1 同步令牌模型

[SyncToken.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Models/SyncToken.php) 存储同步令牌：

```php
class SyncToken extends Model
{
    protected $fillable = [
        'account_id',
        'user_id',
        'name',       // backendId，如 "contacts-{vaultId}"
        'timestamp',  // 令牌创建时间
    ];
}
```

### 6.2 令牌管理

#### 获取当前令牌 - [getCurrentSyncToken()](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/Dav/Web/Backend/SyncDAVBackend.php#L19-L28)

```php
public function getCurrentSyncToken(?string $collectionId): ?SyncToken
{
    return SyncToken::where([
        'account_id' => $this->user->account_id,
        'user_id' => $this->user->id,
        'name' => $this->backendId($collectionId),
    ])
        ->orderBy('created_at', 'desc')
        ->first();
}
```

#### 刷新令牌 - [refreshSyncToken()](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/Dav/Web/Backend/SyncDAVBackend.php#L33-L42)

```php
public function refreshSyncToken(?string $collectionId): SyncToken
{
    $token = $this->getCurrentSyncToken($collectionId);

    if (! $token || $token->timestamp < $this->getLastModified($collectionId)) {
        $token = $this->createSyncTokenNow($collectionId);
    }

    return $token;
}
```

**令牌刷新逻辑：**
- 比较令牌时间戳与集合最后修改时间
- 有新变更时创建新令牌
- 每个用户-地址簿组合独立维护令牌序列

### 6.3 增量变化检测

[getChanges()](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/Dav/Web/Backend/SyncDAVBackend.php#L131-L154) 是同步协议的核心：

```php
public function getChanges(?string $collectionId, ?string $syncToken): ?array
{
    $token = null;
    $timestamp = null;
    if ($syncToken !== null && $syncToken !== '') {
        $token = $this->getSyncToken($collectionId, (int) $syncToken);

        if ($token === null) {
            return null;  // 令牌无效，返回null触发全量同步
        }

        $timestamp = $token->timestamp;
    }

    $objs = $this->getObjects($collectionId);

    return [
        'syncToken' => $this->refreshSyncToken($collectionId)->id,
        'added' => $this->getAdded($objs, $timestamp),
        'modified' => $this->getModified($objs, $timestamp),
        'deleted' => $this->getDeleted($collectionId, $timestamp),
    ];
}
```

### 6.4 变化分类算法

#### 新增（Added）- [getAdded()](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/Dav/Web/Backend/SyncDAVBackend.php#L159-L167)

```php
private function getAdded(Collection $objs, ?Carbon $timestamp): array
{
    return $objs->filter(fn (IDavResource $obj): bool => 
        $timestamp === null || $obj->created_at >= $timestamp
    )
    ->map(fn ($obj): string => $this->encodeUri($obj))
    ->values()
    ->toArray();
}
```

**判定条件**：`created_at >= 令牌时间戳`

#### 修改（Modified）- [getModified()](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/Dav/Web/Backend/SyncDAVBackend.php#L172-L185)

```php
private function getModified(Collection $objs, ?Carbon $timestamp): array
{
    return $objs->filter(fn (IDavResource $obj): bool => 
        $timestamp !== null &&
        $obj->updated_at > $timestamp &&
        $obj->created_at < $timestamp
    )
    ->map(function (IDavResource $obj): string {
        $this->refreshObject($obj);
        return $this->encodeUri($obj);
    })
    ->values()
    ->toArray();
}
```

**判定条件**：
- `created_at < 令牌时间戳`（不是新增）
- `updated_at > 令牌时间戳`（有修改）

**关键细节**：修改的对象会先调用 `refreshObject()` 确保vCard缓存是最新的。

#### 删除（Deleted）- [getDeleted()](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/Dav/Web/Backend/SyncDAVBackend.php#L190-L199)

```php
private function getDeleted(string $collectionId, ?Carbon $timestamp): array
{
    return $this->getDeletedObjects($collectionId)
        ->filter(fn (IDavResource $obj): bool => 
            $timestamp === null || $obj->deleted_at >= $timestamp
        )
        ->map(fn (IDavResource $obj): string => $this->encodeUri($obj))
        ->values()
        ->toArray();
}
```

**判定条件**：`deleted_at >= 令牌时间戳`（使用软删除记录）

### 6.5 同步令牌与地址簿关联

在 [getAddressBookDetails()](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/Dav/Web/Backend/CardDAV/CardDAVBackend.php#L85-L107) 中，同步令牌被添加到地址簿属性：

```php
private function getAddressBookDetails(Vault $vault): array
{
    $token = $this->getCurrentSyncToken((string) $vault->id);

    $des = [
        'id' => $vault->id,
        'uri' => $vault->name,
        'principaluri' => PrincipalBackend::getPrincipalUser($this->user),
        '{DAV:}displayname' => $vault->name,
    ];
    
    if ($token) {
        $des += [
            '{DAV:}sync-token' => $token->id,
            '{'.SabreServer::NS_SABREDAV.'}sync-token' => $token->id,
            '{'.CalDAVPlugin::NS_CALENDARSERVER.'}getctag' => DAVSyncPlugin::SYNCTOKEN_PREFIX.$token->id,
        ];
    }

    return $des;
}
```

---

## 7. 错误处理 - ETag不匹配与权限不足

### 7.1 ETag机制

[GetEtag.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/Dav/Services/GetEtag.php) 生成ETag：

```php
public function execute(array $data): string
{
    // ... 验证权限 ...
    
    if ($entry->distant_etag) {
        return $entry->distant_etag;
    }
    
    return '"'.hash('sha256', $entry->vcard).'"';
}
```

**ETag生成规则：**
1. 优先使用 `distant_etag`（外部同步时使用）
2. 否则对vCard内容做SHA-256哈希
3. 结果用双引号包裹（符合HTTP ETag规范）

### 7.2 ETag不匹配处理

在 [UpdateVCard::execute()](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/Dav/Jobs/UpdateVCard.php#L58-L74) 中检测ETag不匹配：

```php
public function execute(array $data): void
{
    $this->withLocale($this->author->preferredLocale(), function () {
        $newtag = $this->updateCard($this->data['uri'], $this->data['card']);

        if ($newtag !== null && ($etag = Arr::get($this->data, 'etag')) !== null && $newtag !== $etag) {
            Log::channel('database')->warning(
                __CLASS__.' '.__FUNCTION__." wrong etag when updating contact. Expected [$etag], got [$newtag]",
                [
                    'contacturl' => $this->data['uri'],
                    'carddata' => $this->data['card'],
                ]
            );
        }
    });
}
```

**ETag不匹配行为：**
1. **不阻止更新**：即使ETag不匹配，仍然执行更新操作
2. **记录警告日志**：在database日志通道记录警告，包含期望ETag、实际ETag、URI和vCard内容
3. **客户端后果**：由于DAV请求已返回200/201，客户端不会收到冲突错误。但后续同步时客户端会收到更新后的ETag，最终一致性由同步机制保证

### 7.3 无编辑权限处理

#### 场景1：更新/创建时权限检查

在 [updateCard()](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/Dav/Web/Backend/CardDAV/CardDAVBackend.php#L439-L440) 中：

```php
$vault = $this->vaults($addressBookId, permission: Vault::PERMISSION_EDIT)
    ->firstOrFail();
```

**行为**：`firstOrFail()` 抛出 `ModelNotFoundException`，SabreDAV将其转换为 **404 Not Found** 响应。

#### 场景2：删除时权限检查

删除时通过 [getObject()](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/Dav/Web/Backend/SyncDAVBackend.php#L235-L242) → [getObjectUuid()](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/Dav/Web/Backend/CardDAV/CardDAVBackend.php#L257-L279) 间接检查权限：

```php
public function getObjectUuid(?string $collectionId, string $uuid): ?VCardResource
{
    $vault = $this->vaults($collectionId)
        ->first();

    if (! $vault) {
        throw new NotEnoughPermissionException;
    }

    // ... 查询对象 ...
}
```

**行为**：抛出 `NotEnoughPermissionException`，SabreDAV将其转换为 **403 Forbidden** 响应。

#### 场景3：地址簿列表权限

`AclPlugin` 的 `hideNodesFromListings = true` 配置确保：
- 用户无权限的Vault不会出现在地址簿列表中
- 即使知道URL，访问无权限的地址簿也会返回403

---

## 8. 完整调用链总结

### 8.1 认证流程

```
DAV请求 → AuthPlugin → AuthBackend::check() 
    → Laravel Auth::check() → 返回主体URI → ACL检查
```

### 8.2 地址簿列表流程

```
PROPFIND请求 → CardDAVPlugin → CardDAVBackend::getAddressBooksForUser()
    → GetVaults::vaults(PERMISSION_VIEW) → 过滤有权限的Vault
    → getCurrentSyncToken() → 获取同步令牌
    → 返回地址簿列表（含sync-token）
```

### 8.3 联系人读取流程

```
GET /addressbook/contact.vcf → CardDAVBackend::getCard()
    → getObject() → getObjectUuid() → 权限检查
    → prepareCard()
        → ReadVObject::execute() 解析缓存vCard
        → 比较REV与updated_at，判断缓存有效性
        → 缓存失效 → ExportVCard::execute() 重新导出
        → GetEtag::execute() 计算ETag
    → 返回vCard数据 + ETag + Last-Modified
```

### 8.4 联系人更新流程

```
PUT /addressbook/contact.vcf → CardDAVBackend::updateCard()
    → GetVaults::vaults(PERMISSION_EDIT) → 权限检查（失败返回404）
    → 创建UpdateVCard作业 → 入队列
    → 返回204 No Content
    
    （异步）队列处理 → UpdateVCard::execute()
        → ImportVCard::execute() → 解析vCard → 写入数据库
        → GetEtag::execute() → 计算新ETag
        → ETag比较 → 不匹配则记录警告日志
```

### 8.5 增量同步流程

```
REPORT sync-collection → SyncPlugin → CardDAVBackend::getChangesForAddressBook()
    → SyncDAVBackend::getChanges()
        → 验证syncToken（无效返回null → 客户端全量同步）
        → getObjects() → 获取所有活动对象
        → getAdded() → 新增对象（created_at >= timestamp）
        → getModified() → 修改对象（updated_at > timestamp && created_at < timestamp）
            → refreshObject() → 确保vCard最新
        → getDeleted() → 删除对象（deleted_at >= timestamp，软删除记录）
        → refreshSyncToken() → 生成新令牌
    → 返回 added/modified/deleted 列表 + 新syncToken
```

---

## 9. 核心设计模式总结

| 模式 | 应用场景 | 优势 |
|-----|---------|------|
| **Trait组合** | `GetVaults`、`SyncDAVBackend`、`WithUser` | 代码复用，灵活组合功能 |
| **领域服务** | `ExportVCard`、`ImportVCard`、`GetEtag` | 单一职责，可测试性强 |
| **队列异步** | `UpdateVCard`、`DestroyContact`、`DestroyGroup` | 响应快速，解耦DAV协议与业务逻辑 |
| **软删除** | `VCardResource` 使用 `SoftDeletes` | 保留删除历史，支持增量同步 |
| **缓存优化** | vCard缓存 + REV时间戳比较 | 减少重复导出，提升性能 |
| **桥接模式** | `AuthBackend` 桥接Laravel与SabreDAV | 复用现有认证体系，避免重复实现 |

---

## 10. 关键文件索引

| 模块 | 文件路径 | 核心职责 |
|-----|---------|---------|
| 服务提供者 | [DAVServiceProvider.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Providers/DAVServiceProvider.php) | 注册节点和插件 |
| 认证后端 | [AuthBackend.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/Dav/Web/Auth/AuthBackend.php) | 复用Laravel认证 |
| 权限过滤 | [GetVaults.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/Dav/Web/Backend/GetVaults.php) | 按权限返回Vault |
| CardDAV后端 | [CardDAVBackend.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/Dav/Web/Backend/CardDAV/CardDAVBackend.php) | CardDAV CRUD操作 |
| 同步后端 | [SyncDAVBackend.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/Dav/Web/Backend/SyncDAVBackend.php) | 增量同步支持 |
| 主体后端 | [PrincipalBackend.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/Dav/Web/DAVACL/PrincipalBackend.php) | 用户主体映射 |
| Vault模型 | [Vault.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Models/Vault.php) | 权限常量定义 |
| 同步令牌 | [SyncToken.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Models/SyncToken.php) | 同步令牌存储 |
| vCard导出 | [ExportVCard.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/Dav/Services/ExportVCard.php) | 导出vCard |
| vCard导入 | [ImportVCard.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/Dav/Services/ImportVCard.php) | 导入vCard |
| ETag生成 | [GetEtag.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/Dav/Services/GetEtag.php) | 计算ETag |
| vCard解析 | [ReadVObject.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/Dav/Services/ReadVObject.php) | 解析vCard |
| 更新作业 | [UpdateVCard.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/Dav/Jobs/UpdateVCard.php) | 异步更新vCard |
| 资源接口 | [IDavResource.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/Dav/IDavResource.php) | DAV资源接口 |
| 资源基类 | [VCardResource.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/Dav/VCardResource.php) | vCard资源抽象基类 |
