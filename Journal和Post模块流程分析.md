# Journal和Post模块流程分析

## 一、PostController 中 posts/template/{template} 创建草稿分析

### 1. 路由与控制器方法

在 [PostController.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Vault/ManageJournals/Web/Controllers/PostController.php) 中，`store` 方法处理从模板创建文章的请求。

### 2. 创建草稿的原因

从模板创建文章时立即创建草稿的设计意图：

- **即时创建**：用户访问页面时立即创建，而不是等到用户填写内容后再创建
- **草稿状态**：通过 `published = false 标记为草稿
- **预填充 Sections**：根据模板预先创建所有的 post sections

关键代码（第45-76行）：

```php
public function store(Request $request, string $vaultId, int $journalId, int $templateId)
{
    $vault = Vault::findOrFail($vaultId);
    $journal = Journal::findOrFail($journalId);

    try {
        PostTemplate::where('account_id', $vault->account_id)
            ->findOrFail($templateId);
    } catch (ModelNotFoundException $e) {
        return redirect()->route('post.choose_template', [
            'vault' => $vaultId,
            'journal' => $journalId,
        ]);
    }

    $post = (new CreatePost)->execute([
        'account_id' => Auth::user()->account_id,
        'author_id' => Auth::id(),
        'vault_id' => $vaultId,
        'journal_id' => $journalId,
        'post_template_id' => $templateId,
        'title' => null,
        'published' => false,  // 关键：标记为草稿
        'written_at' => Carbon::now()->format('Y-m-d'),
    ]);

    return redirect()->route('post.edit', [
        'vault' => $vaultId,
        'journal' => $journalId,
        'post' => $post->id,
    ]);
}
```

### 3. 设计意图说明

- **用户体验优化**：用户选择模板后立即跳转到编辑页面，所有 sections 已准备好
- **自动保存**：创建草稿后，用户可以直接编辑，无需手动保存
- **内容填充**：模板 sections 已复制好，用户可以直接开始填写内容

---

## 二、CreatePost 复制模板 Section 分析

### 1. 服务类位置

[CreatePost.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Vault/ManageJournals/Services/CreatePost.php)

### 2. 执行流程

```
execute()
    ├── validate()          // 验证数据和权限
    ├── create()            // 创建 Post 记录
    └── createPostSections()  // 复制模板 sections
```

### 3. createPostSections 方法（第95-108行）：

```php
private function createPostSections(): void
{
    $postTemplateSections = $this->postTemplate->postTemplateSections()
        ->orderBy('position')
        ->get();

    foreach ($postTemplateSections as $postTemplateSection) {
        PostSection::create([
            'post_id' => $this->post->id,
            'position' => $postTemplateSection->position,
            'label' => $postTemplateSection->label,
        ]);
    }
}
```

### 4. 复制逻辑说明

- **来源**：从 `PostTemplateSection` 表获取模板定义
- **目标**：写入 `PostSection` 表
- **复制字段**：
  - `position`：保持顺序位置
  - `label`：section 标题
- **内容字段**：`content` 初始为 null，等待用户填写

---

## 三、UpdatePost 更新 Section 内容分析

### 1. 服务类位置

[UpdatePost.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Vault/ManageJournals/Services/UpdatePost.php)

### 2. 执行流程

```
execute()
    ├── validate()      // 验证数据和权限
    ├── update()        // 更新 Post 基本信息
    └── updateSections()  // 更新 sections 内容
```

### 3. updateSections 方法（第82-95行）：

```php
private function updateSections(): void
{
    foreach ($this->data['sections'] as $section) {
        if (! array_key_exists('content', $section)) {
            continue;
        }

        $this->post->postSections()
            ->find($section['id'])
            ->update([
                'content' => $section['content'],
            ]);
    }
}
```

### 4. 更新逻辑说明

- **遍历传入的 sections 数组**：从请求中获取所有 sections
- **检查 content 字段**：只更新包含 content 的 section
- **通过 ID 查找**：`$this->post->postSections()->find($section['id'])
- **更新内容**：只更新 `content` 字段

### 5. 控制器中的调用

在 [PostController.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Vault/ManageJournals/Web/Controllers/PostController.php) 第109-146行的 `update` 方法中：

```php
$post = (new UpdatePost)->execute([
    'account_id' => Auth::user()->account_id,
    'author_id' => Auth::id(),
    'vault_id' => $vaultId,
    'journal_id' => $journalId,
    'post_id' => $postId,
    'title' => $request->input('title'),
    'sections' => $request->input('sections'),
    'written_at' => Carbon::parse($request->input('date'))->format('Y-m-d'),
]);
```

---

## 四、AddContactToPost 同时写 post_contact 和联系人 Feed Item 分析

### 1. 服务类位置

[AddContactToPost.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Vault/ManageJournals/Services/AddContactToPost.php)

### 2. 执行流程

```
execute()
    ├── validate()                     // 验证数据和权限
    ├── $post->contacts()->syncWithoutDetaching()  // 写 post_contact 表
    ├── createFeedItem()                 // 创建 ContactFeedItem
    └── updateLastEditedDate()           // 更新联系人最后编辑时间
```

### 3. 关联联系人（第53行）：

```php
$this->post->contacts()->syncWithoutDetaching($this->contact);
```

- **表**：`contact_post`（多对多关联表
- **方法**：`syncWithoutDetaching` - 同步但不删除已有的关联
- **作用**：建立 Post 和 Contact 的关联关系

### 4. 创建 Feed Item（第78-87行）：

```php
private function createFeedItem(): void
{
    $feedItem = ContactFeedItem::create([
        'author_id' => $this->author->id,
        'contact_id' => $this->contact->id,
        'action' => ContactFeedItem::ACTION_ADDED_TO_POST,
        'description' => $this->post->title,
    ]);
    $this->post->feedItem()->save($feedItem);
}
```

- **表**：`contact_feed_items
- **多态关联**：通过 `feedable` 多态关系关联到 Post
- **记录动作**：`ACTION_ADDED_TO_POST` 表示添加到文章的动作
- **描述**：保存文章标题

### 5. 模型关系

在 [Post.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Models/Post.php) 中定义：

```php
// 多对多关联联系人
public function contacts(): BelongsToMany
{
    return $this->belongsToMany(Contact::class);
}

// 多态关联 Feed Item
public function feedItem(): MorphOne
{
    return $this->morphOne(ContactFeedItem::class, 'feedable');
}
```

### 6. 控制器中的调用

在 [PostController.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Vault/ManageJournals/Web/Controllers/PostController.php) 第124-141行：

```php
$post->contacts()->detach();

if ($request->input('contacts')) {
    if (count($request->input('contacts')) > 0) {
        foreach ($request->input('contacts') as $contact) {
            $data = [
                'account_id' => Auth::user()->account_id,
                'author_id' => Auth::user()->id,
                'vault_id' => $vaultId,
                'journal_id' => $journalId,
                'post_id' => $postId,
                'contact_id' => $contact['id'],
            ];

            (new AddContactToPost)->execute($data);
        }
    }
}
```

---

## 五、IncrementPostReadCounter 执行时机分析

### 1. 服务类位置

[IncrementPostReadCounter.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Vault/ManageJournals/Services/IncrementPostReadCounter.php)

### 2. 执行时机

**在文章展示页面被访问时执行**

在 [PostController.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Vault/ManageJournals/Web/Controllers/PostController.php) 第78-95行的 `show` 方法中：

```php
public function show(Request $request, string $vaultId, int $journalId, int $postId)
{
    $vault = Vault::findOrFail($vaultId);
    $post = Post::findOrFail($postId);

    (new IncrementPostReadCounter)->execute([
        'account_id' => Auth::user()->account_id,
        'author_id' => Auth::id(),
        'vault_id' => $vaultId,
        'journal_id' => $journalId,
        'post_id' => $postId,
    ]);

    return Inertia::render('Vault/Journal/Post/Show', [
        'layoutData' => VaultIndexViewHelper::layoutData($vault),
        'data' => PostShowViewHelper::data($post, Auth::user()),
    ]);
}
```

### 3. 核心逻辑（第65-68行）：

```php
private function increment(): void
{
    $this->post->increment('view_count');
}
```

### 4. 模型字段

在 [Post.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Models/Post.php) 第30行：

```php
protected $fillable = [
    'journal_id',
    'slice_of_life_id',
    'title',
    'view_count',  // 阅读计数字段
    'published',
    'written_at',
    'updated_at',
];
```

### 5. 触发场景

- **每次访问文章详情页**时自动增加阅读计数
- **用户查看文章**时执行
- **不区分用户**：同一用户多次查看也会增加计数

---

## 六、AddPostToSliceOfLife 验证同一 Vault 分析

### 1. 服务类位置

[AddPostToSliceOfLife.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Vault/ManageJournals/Services/AddPostToSliceOfLife.php)

### 2. 验证逻辑

通过 **Journal 作为中间层确保 Post 和 SliceOfLife 属于同一 Vault

### 3. 核心验证代码（第57-71行）：

```php
private function validate(): void
{
    $this->validateRules($this->data);

    // 1. 通过 vault 找到 journal（确保 journal 属于正确的 vault
    $journal = $this->vault->journals()
        ->findOrFail($this->data['journal_id']);

    // 2. 通过 journal 找到 post（确保 post 属于正确的 journal）
    $this->post = $journal->posts()
        ->findOrFail($this->data['post_id']);

    // 3. 通过同一个 journal 找到 slice of life（确保 slice 属于同一个 journal）
    if ($this->valueOrNull($this->data, 'slice_of_life_id')) {
        $this->slice = $journal->slicesOfLife()
            ->findOrFail($this->data['slice_of_life_id']);
    }
}
```

### 4. 数据模型关系

```
Vault
  └── Journal (属于 Vault)
        ├── Post (属于 Journal)
        └── SliceOfLife (属于 Journal)
```

**关键验证链：

1. **`$this->vault->journals()**：确保 Journal 属于正确的 Vault
2. **`$journal->posts()**：确保 Post 属于该 Journal（即属于同一 Vault）
3. **`$journal->slicesOfLife()`：确保 SliceOfLife 也属于同一个 Journal

由于 Post 和 SliceOfLife 都属于同一个 Journal，而 Journal 属于 Vault，因此它们必然属于同一个 Vault。

### 5. 模型关系定义

在 [SliceOfLife.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Models/SliceOfLife.php) 中：

```php
// SliceOfLife 属于 Journal
public function journal(): BelongsTo
{
    return $this->belongsTo(Journal::class);
}
```

在 [Post.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Models/Post.php) 中：

```php
// Post 属于 SliceOfLife
public function sliceOfLife(): BelongsTo
{
    return $this->belongsTo(SliceOfLife::class);
}
```

### 6. 最终关联

验证通过后执行关联（第51-52行）：

```php
$this->post->slice_of_life_id = optional($this->slice)->id;
$this->post->save();
```

---

## 七、整体流程总结

### 1. 从模板创建文章流程

```
用户选择模板
    ↓
PostController::store()
    ├── 验证模板存在
    └── CreatePost::execute()
          ├── 创建 Post (published=false)
          └── 复制模板 Sections 到 PostSections
    ↓
跳转到编辑页面
```

### 2. 保存草稿/更新内容流程

```
用户编辑文章
    ↓
PostController::update()
    ├── UpdatePost::execute()
    │     ├── 更新 Post 基本信息
    │     └── 更新每个 Section 的 content
    ├── 移除所有联系人关联
    └── 重新添加联系人
          └── AddContactToPost::execute()
                ├── syncWithoutDetaching 关联
                ├── 创建 ContactFeedItem
                └── 更新联系人 last_updated_at
```

### 3. 展示阅读流程

```
用户查看文章
    ↓
PostController::show()
    ├── IncrementPostReadCounter::execute()
    │     └── increment view_count
    └── 渲染文章详情页
```

### 4. 关联到 Slice Of Life 流程

```
用户选择 SliceOfLife
    ↓
AddPostToSliceOfLife::execute()
    ├── 验证: vault → journal → post
    ├── 验证: vault → journal → sliceOfLife
    └── 关联 post.slice_of_life_id
```
