# Vault 权限架构与CRUD操作分析

## 一、权限系统整体架构

Monica采用**三层权限防护**机制，确保Web和API两条入口共享相同的服务层权限规则：

| 层级 | 技术实现 | 作用位置 | 主要职责 |
|------|----------|----------|----------|
| 第一层 | Gate + Policy | Web路由层 | 基于模型的粗粒度权限检查 |
| 第二层 | Sanctum Abilities | API路由层 | Token级别的读写能力检查 |
| 第三层 | BaseService 权限验证 | 服务层 | 细粒度业务逻辑权限检查（Web和API共享） |

---

## 二、核心权限概念

### 2.1 权限常量定义

在 [Vault.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Models/Vault.php#L17-L23) 中定义了三个权限等级，数值采用**倒序排列**（数值越小，权限越高）：

```php
public const PERMISSION_VIEW = 300;      // 查看者
public const PERMISSION_EDIT = 200;      // 编辑者
public const PERMISSION_MANAGE = 100;    // 管理者
```

**设计意图**：采用倒序数值便于使用 `<=` 运算符进行权限比较。高权限（如MANAGE=100）自然满足低权限的检查条件。

### 2.2 用户与Vault的Pivot关联

用户与Vault通过 `user_vault` 中间表建立多对多关系，pivot表包含 `permission` 字段存储用户在该Vault中的权限值。

在 [TestCase.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/tests/TestCase.php#L66-L80) 的测试辅助方法中可以看到典型的关联创建方式：

```php
public function setPermissionInVault(User $user, int $permission, Vault $vault): Vault
{
    $contact = Contact::factory()->create([
        'vault_id' => $vault->id,
    ]);
    $vault->users()->save($user, [
        'permission' => $permission,
        'contact_id' => $contact->id,
    ]);
    return $vault;
}
```

### 2.3 Gate 定义

在 [AuthServiceProvider.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Providers/AuthServiceProvider.php#L32-L59) 中定义了三个Gate，使用 `<=` 运算符实现权限继承：

```php
// 任何在Vault中的用户都通过
Gate::define('vault-viewer', function (User $user, $vault): bool {
    return $user->vaults()
        ->wherePivotIn('vault_id', [static::id($vault)])
        ->exists();
});

// 权限值 <= 200 (EDIT + MANAGE)
Gate::define('vault-editor', function (User $user, $vault): bool {
    return $user->vaults()
        ->wherePivotIn('vault_id', [static::id($vault)])
        ->wherePivot('permission', '<=', 200)
        ->exists();
});

// 权限值 <= 100 (仅MANAGE)
Gate::define('vault-manager', function (User $user, $vault): bool {
    return $user->vaults()
        ->wherePivotIn('vault_id', [static::id($vault)])
        ->wherePivot('permission', '<=', 100)
        ->exists();
});
```

### 2.4 VaultPolicy 策略类

[VaultPolicy.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Policies/VaultPolicy.php) 将Gate封装为模型策略：

```php
public function view(User $user, Vault $vault): bool
{
    return Gate::forUser($user)->allows('vault-viewer', $vault);
}

public function update(User $user, Vault $vault): bool
{
    return Gate::forUser($user)->allows('vault-editor', $vault);
}

public function delete(User $user, Vault $vault): bool
{
    return Gate::forUser($user)->allows('vault-manager', $vault);
}
```

---

## 三、Web与API入口的权限共享机制

### 3.1 Web入口流程

**路由配置**：[web.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/routes/web.php#L181-L192)

```php
Route::middleware([
    'auth:sanctum',
    config('jetstream.auth_session'),
    'verified',
])->group(function () {
    Route::prefix('vaults')->group(function () {
        Route::get('', [VaultController::class, 'index'])->name('vault.index');
        Route::post('', [VaultController::class, 'store'])->name('vault.store');
        Route::put('{vault}', [VaultController::class, 'update'])->name('vault.update');
        Route::delete('{vault}', [VaultController::class, 'destroy'])->name('vault.destroy');
    });
});
```

**Web控制器**：[Web/VaultController.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Vault/ManageVault/Web/Controllers/VaultController.php#L21-L132)

```php
public function __construct()
{
    $this->authorizeResource(Vault::class, 'vault');  // 自动应用VaultPolicy
}
```

`authorizeResource` 会自动为每个CRUD操作应用对应的Policy方法，这是**第一层防护**。

### 3.2 API入口流程

**路由配置**：[api.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/routes/api.php#L17-L25)

```php
Route::middleware('auth:sanctum')->name('api.')->group(function () {
    Route::apiResource('vaults', VaultController::class);
});
```

**API控制器**：[Api/VaultController.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Vault/ManageVault/Api/Controllers/VaultController.php#L14-L126)

```php
public function __construct()
{
    $this->middleware('abilities:read')->only(['index', 'show']);
    $this->middleware('abilities:write')->only(['store', 'update', 'delete']);
    parent::__construct();
}
```

API入口使用Sanctum的 `abilities` 中间件（别名定义在 [bootstrap/app.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/bootstrap/app.php#L30)）进行Token能力检查，这是**第二层防护**。

### 3.3 共享服务层（核心）

无论Web还是API入口，最终都会调用相同的Service类，而所有Service都继承自 [BaseService.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Services/BaseService.php)，这是**第三层防护**，也是真正的共享权限层。

**权限验证流程**：

每个Service通过 `permissions()` 方法声明所需权限，`validateRules()` 方法自动按依赖顺序执行验证：

```php
// BaseService.php 中的权限依赖定义
private static array $permissionDependencies = [
    'author_must_belong_to_account' => [],
    'author_must_be_account_administrator' => ['author_must_belong_to_account'],
    'vault_must_belong_to_account' => [],
    'author_must_be_vault_manager' => ['vault_must_belong_to_account', 'author_must_belong_to_account'],
    'author_must_be_vault_editor' => ['vault_must_belong_to_account', 'author_must_belong_to_account'],
    'author_must_be_in_vault' => ['vault_must_belong_to_account', 'author_must_belong_to_account'],
    'contact_must_belong_to_vault' => ['vault_must_belong_to_account', 'author_must_belong_to_account'],
    'group_must_belong_to_vault' => ['vault_must_belong_to_account', 'author_must_belong_to_account'],
];
```

**核心验证方法** [validateUserPermissionInVault](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Services/BaseService.php#L187-L200)：

```php
public function validateUserPermissionInVault(int $permission): void
{
    $exists = $this->author->vaults()
        ->where('vaults.id', $this->vault->id)
        ->wherePivot('permission', '<=', $permission)
        ->exists();

    if (! $exists) {
        throw new NotEnoughPermissionException;
    }
}
```

该方法接收权限常量值，使用 `<=` 比较实现权限继承。

---

## 四、CRUD操作权限规则

### 4.1 Create（创建Vault）

**权限要求**：[CreateVault.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Vault/ManageVault/Services/CreateVault.php#L37-L42)

```php
public function permissions(): array
{
    return [
        'author_must_belong_to_account',  // 只需属于账户，无需vault权限
    ];
}
```

**创建流程** [execute](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Vault/ManageVault/Services/CreateVault.php#L48-L63) 方法包含6个步骤：

1. **创建Vault本身** [createVault](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Vault/ManageVault/Services/CreateVault.php#L65-L78)：
   - 设置默认模板为账户中的第一个模板（如果存在）
   ```php
   $template = $this->account()->templates()->first();
   $this->vault = Vault::create([
       'account_id' => $this->data['account_id'],
       'type' => $this->data['type'],
       'name' => $this->data['name'],
       'description' => $this->valueOrNull($this->data, 'description'),
       'default_template_id' => $template ? $template->id : null,
   ]);
   ```

2. **创建用户默认联系人** [createUserContact](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Vault/ManageVault/Services/CreateVault.php#L80-L94)：
   - 为创建者创建一个不可删除的联系人记录（代表自己）
   - 建立用户与Vault的关联，权限设为 `PERMISSION_MANAGE`
   ```php
   $contact = Contact::create([
       'vault_id' => $this->vault->id,
       'first_name' => $this->author->first_name,
       'last_name' => $this->author->last_name,
       'can_be_deleted' => false,
       'template_id' => $this->vault->default_template_id,
   ]);
   $this->vault->users()->save($this->author, [
       'permission' => Vault::PERMISSION_MANAGE,
       'contact_id' => $contact->id,
   ]);
   ```

3. **填充默认重要日期类型** [populateDefaultContactImportantDateTypes](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Vault/ManageVault/Services/CreateVault.php#L96-L115)：
   - 生日（Birthdate）
   - 逝世日期（Deceased date）

4. **填充情绪追踪参数** [populateMoodTrackingParameters]

5. **填充默认人生事件分类** [populateDefaultLifeEventCategories]：
   - 交通类（Transportation）：骑车、驾车、步行、公交、地铁
   - 社交类（Social）：吃饭、喝水、去酒吧、看电影等
   - 运动类（Sport）：高尔夫、网球等
   - 工作类（Work）：入职、离职、被解雇、升职等

6. **填充默认Quick Fact模板** [populateDefaultQuickVaultTemplateEntries]：
   - 爱好（Hobbies）
   - 饮食偏好（Food preferences）

### 4.2 Read（读取Vault）

**权限要求**：
- Web层：`view` → `vault-viewer`（只需在Vault中）
- 服务层：`author_must_be_in_vault` → 验证 `<= PERMISSION_VIEW (300)`

### 4.3 Update（更新Vault）

**权限要求**：[UpdateVault.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Vault/ManageVault/Services/UpdateVault.php#L30-L38)

```php
public function permissions(): array
{
    return [
        'author_must_belong_to_account',
        'vault_must_belong_to_account',
        'author_must_be_vault_manager',  // 需 <= 100
    ];
}
```

**注意**：更新Vault基本信息需要 `MANAGE` 权限（100），而不是 `EDIT`（200）。这是因为Vault元数据属于敏感配置。

### 4.4 Delete（删除Vault）

**权限要求**：[DestroyVault.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Vault/ManageVault/Services/DestroyVault.php#L29-L41)

```php
public function permissions(): array
{
    return [
        'author_must_belong_to_account',
        'vault_must_belong_to_account',
        'author_must_be_vault_manager',  // 需 <= 100
    ];
}
```

**删除流程** [execute](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Vault/ManageVault/Services/DestroyVault.php#L45-L52)：

```php
public function execute(array $data): void
{
    $this->validateRules($data);

    // 逐个删除文件以触发FileDeleted事件，清理存储空间
    File::where('vault_id', $data['vault_id'])->chunk(100, function ($files) {
        $files->each(function ($file) {
            $file->delete();
        });
    });

    $this->vault->delete();
}
```

**存储清理事件机制**：

1. [File.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Models/File.php#L55-L59) 模型配置了删除事件：
   ```php
   protected $dispatchesEvents = [
       'deleted' => FileDeleted::class,
   ];
   ```

2. [FileDeleted](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/ManageDocuments/Events/FileDeleted.php) 事件被触发

3. [DeleteFileInStorage](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/ManageDocuments/Listeners/DeleteFileInStorage.php) 监听器处理：
   ```php
   public function handle(FileDeleted $event)
   {
       $this->file = $event->file;
       $this->checkAPIKeyPresence();
       $this->getFileFromUploadcare();
       $this->deleteFile();  // 调用Uploadcare API删除远程文件
   }
   ```

**设计意图**：不直接删除File记录，而是通过 `chunk` + 逐个 `delete()` 的方式，确保每个文件都触发Model事件，从而正确清理Uploadcare等云存储中的实际文件。

---

## 五、权限常量数值大小对GetVaults过滤逻辑的影响

### 5.1 倒序设计的核心原理

权限常量采用**倒序排列**是为了支持**权限继承**：

```
MANAGE (100) < EDIT (200) < VIEW (300)
```

当检查某权限时，使用 `<=` 运算符：
- 检查 `<= 100`（MANAGE）：只有MANAGE通过
- 检查 `<= 200`（EDIT）：MANAGE和EDIT都通过
- 检查 `<= 300`（VIEW）：MANAGE、EDIT、VIEW都通过

### 5.2 GetVaults中的应用

在 [GetVaults.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Contact/Dav/Web/Backend/GetVaults.php) trait 中，`vaults()` 方法使用权限过滤：

```php
public function vaults(?string $collectionId = null, int $permission = Vault::PERMISSION_VIEW): Collection
{
    $vaults = $this->user->vaults()
        ->wherePivot('permission', '<=', $permission);

    if ($collectionId !== null) {
        $vaults = $vaults->where('id', $collectionId);
    }

    return $vaults->get();
}
```

**数值大小的影响**：

| 传入参数 | 过滤条件 | 返回结果 |
|---------|----------|----------|
| `PERMISSION_MANAGE (100)` | `<= 100` | 仅返回用户是管理者的Vault |
| `PERMISSION_EDIT (200)` | `<= 200` | 返回用户是管理者或编辑者的Vault |
| `PERMISSION_VIEW (300)` | `<= 300` | 返回用户有权访问的所有Vault |

### 5.3 其他地方的应用

在 [BaseService.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Services/BaseService.php#L141-L151) 的 `validatePermission` switch中：

```php
case 'author_must_be_vault_manager':
    $this->validateUserPermissionInVault(Vault::PERMISSION_MANAGE);  // <= 100
    break;
case 'author_must_be_vault_editor':
    $this->validateUserPermissionInVault(Vault::PERMISSION_EDIT);    // <= 200
    break;
case 'author_must_be_in_vault':
    $this->validateUserPermissionInVault(Vault::PERMISSION_VIEW);    // <= 300
    break;
```

---

## 六、完整权限校验链路

以 **Web删除Vault** 为例：

```
请求 → auth:sanctum (认证) → authorizeResource (Policy层)
    → delete() 方法 → Gate::allows('vault-manager') 
    → 检查 pivot.permission <= 100
        → DestroyVault::execute()
            → validateRules()
                → validateAuthorBelongsToAccount()
                → validateVaultExists()
                → validateUserPermissionInVault(100)  [服务层二次校验]
                    → 检查 pivot.permission <= 100
                        → 删除文件（触发FileDeleted事件）
                        → 删除Vault
```

以 **API创建Vault** 为例：

```
请求 → auth:sanctum (Token认证) → abilities:write (Token能力检查)
    → store() 方法
        → CreateVault::execute()
            → validateRules()
                → validateAuthorBelongsToAccount()  [服务层校验]
                    → 创建Vault + 默认数据
```

---

## 七、设计优势总结

1. **单一职责**：权限逻辑集中在Service层，避免Web和API重复实现
2. **深度防御**：三层防护（Policy/Abilities → Service）确保安全
3. **权限继承**：倒序数值设计 + `<=` 运算符，天然支持高权限包含低权限
4. **事件驱动**：File删除通过Model事件触发存储清理，确保数据一致性
5. **依赖明确**：`permissionDependencies` 数组确保权限检查按正确顺序执行
