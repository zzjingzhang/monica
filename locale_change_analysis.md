# 用户修改Locale生效机制分析

## 1. 入口：PreferencesController 和 PreferencesLocaleController

### 1.1 PreferencesController - 偏好设置页面入口

[PreferencesController.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Settings/ManageUserPreferences/Web/Controllers/PreferencesController.php) 负责渲染偏好设置页面：

```php
public function index()
{
    return Inertia::render('Settings/Preferences/Index', [
        'layoutData' => VaultIndexViewHelper::layoutData(),
        'data' => UserPreferencesIndexViewHelper::data(Auth::user()),
    ]);
}
```

**作用**：
- 通过 `GET /settings/preferences` 路由访问
- 使用 Inertia 渲染 Vue 组件 `Settings/Preferences/Index`
- 传递当前用户的偏好数据（包括当前 locale）给前端

### 1.2 PreferencesLocaleController - 修改Locale的API入口

[PreferencesLocaleController.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Settings/ManageUserPreferences/Web/Controllers/PreferencesLocaleController.php) 处理locale修改请求：

```php
public function store(Request $request)
{
    $data = [
        'account_id' => Auth::user()->account_id,
        'author_id' => Auth::id(),
        'locale' => $request->input('locale'),
    ];

    $user = (new StoreLocale)->execute($data);

    return response()->json([
        'data' => UserPreferencesIndexViewHelper::dtoLocale($user),
    ], 200);
}
```

**关键要点**：
- 通过 `POST /settings/preferences/locale` 路由调用
- 接收前端提交的 `locale` 参数
- 调用 `StoreLocale` 服务执行实际的修改逻辑
- 返回更新后的用户locale数据

---

## 2. StoreLocale：验证、写入和设置Locale

[StoreLocale.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Domains/Settings/ManageUserPreferences/Services/StoreLocale.php) 是核心服务类。

### 2.1 验证 supported_locales

通过 Laravel 验证规则确保 locale 有效：

```php
public function rules(): array
{
    return [
        'account_id' => 'required|uuid|exists:accounts,id',
        'author_id' => 'required|uuid|exists:users,id',
        'locale' => [
            'required',
            'string',
            'max:5',
            Rule::in(config('localizer.supported_locales')),
        ],
    ];
}
```

**验证来源**：[localizer.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/config/localizer.php) 配置文件定义了支持的语言列表：

```php
'supported_locales' => ['en', 'ar', 'bn', 'ca', 'da', 'de', 'es', 'el',
    'fr', 'he', 'hi', 'it', 'ja', 'ml', 'nl', 'nn', 'pa', 'pl', 'pt',
    'pt_BR', 'ro', 'ru', 'sv', 'te', 'tr', 'ur', 'vi', 'zh_CN', 'zh_TW',
],
```

### 2.2 写入 user.locale 字段

```php
private function updateUser(): void
{
    $this->author->locale = $this->data['locale'];
    $this->author->save();

    App::setLocale($this->data['locale']);
}
```

**执行流程**：
1. 将新 locale 写入用户模型的 `locale` 属性
2. 保存到数据库
3. **立即调用** `App::setLocale()` 设置当前请求的应用locale

---

## 3. bootstrap/app.php：中间件顺序的意义

[bootstrap/app.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/bootstrap/app.php) 定义了web中间件组的顺序：

```php
$middleware->web(append: [
    \CodeZero\Localizer\Middleware\SetLocale::class,           // 第1位：设置Locale
    \Illuminate\Routing\Middleware\SubstituteBindings::class,  // 第2位：路由绑定
    \App\Http\Middleware\HandleInertiaRequests::class,         // 第3位：Inertia处理
    \Illuminate\Http\Middleware\AddLinkHeadersForPreloadedAssets::class,
]);
```

### 顺序的关键意义

1. **`SetLocale` 必须在最前面**
   - Localizer 中间件负责检测和设置应用的 locale
   - 后续所有中间件和控制器逻辑都依赖正确的 locale 设置
   - 包括：翻译、日期格式化、数字格式化等

2. **`SubstituteBindings` 在中间**
   - 路由模型绑定需要在 locale 设置之后（如果模型属性需要翻译）
   - 在 Inertia 之前（因为 Inertia 可能需要绑定的数据）

3. **`HandleInertiaRequests` 在最后**
   - 确保 Inertia 共享数据时，应用的 locale 已经正确设置
   - 这样 `trans()`、`__()` 等翻译函数才能使用新 locale

### Localizer Detectors 执行顺序

[localizer.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/config/localizer.php#L26-L35) 定义了 locale 检测顺序：

```php
'detectors' => [
    CodeZero\Localizer\Detectors\RouteActionDetector::class,   // 1. 路由动作
    CodeZero\Localizer\Detectors\UrlDetector::class,           // 2. URL
    CodeZero\Localizer\Detectors\OmittedLocaleDetector::class, // 3. 省略locale
    CodeZero\Localizer\Detectors\UserDetector::class,          // 4. 用户模型 ← 主要来源
    CodeZero\Localizer\Detectors\SessionDetector::class,       // 5. Session
    CodeZero\Localizer\Detectors\CookieDetector::class,        // 6. Cookie
    CodeZero\Localizer\Detectors\BrowserDetector::class,       // 7. 浏览器
    CodeZero\Localizer\Detectors\AppDetector::class,           // 8. 应用默认
],
```

**关键点**：`UserDetector` 排在第4位，会从 `user.locale` 字段读取locale，这是用户修改后持久化的来源。

---

## 4. HandleInertiaRequests：共享数据分析

[HandleInertiaRequests.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/app/Http/Middleware/HandleInertiaRequests.php) 是 Inertia 的核心中间件。

### 4.1 share() 方法共享的数据

```php
public function share(Request $request)
{
    $this->storeCurrentUrl($request);

    return [
        ...parent::share($request),
        
        // Help 相关数据
        'help_links' => fn () => config('monica.help_links'),
        'help_url' => fn () => config('monica.help_center_url'),
        
        // Footer（包含翻译）
        'footer' => fn () => $this->footer(),
        
        // WebAuthn 密钥检测函数
        'hasKey' => fn () => function () use ($request) {
            if (! $user = $request->user()) {
                return null;
            }
            return (bool) optional($user->webauthnKeys())->count() > 0;
        },
        
        // Ziggy 路由信息
        'ziggy' => fn () => [
            ...(new Ziggy)->toArray(),
            'location' => $request->url(),
        ],
        
        // Sentry 配置
        'sentry' => fn () => [
            'dsn' => config('sentry.dsn'),
            'tunnel' => config('sentry-tunnel.tunnel-url'),
            'release' => config('sentry.release'),
            'environment' => config('sentry.environment'),
            'sendDefaultPii' => config('sentry.send_default_pii'),
            'tracesSampleRate' => config('sentry.traces_sample_rate'),
        ],
    ];
}
```

### 4.2 Footer 中的 locale 影响

`footer()` 方法使用了 `trans()` 翻译函数：

```php
private function footer(): string
{
    $commit = config('monica.commit');
    $params = [
        'version' => config('monica.app_version'),
        'short' => substr(config('monica.commit'), 0, 7),
    ];
    if ($commit === null) {
        $message = trans('Version :version.', $params);
    } else {
        $params['url'] = Str::finish(config('monica.repository', 'https://github.com/monicahq/monica/'), '/').'commit/'.config('monica.commit');
        $message = trans('Version :version — commit [:short](:url).', $params);
    }

    return Str::markdownExternalLink($message, 'underline text-xs dark:text-gray-100 hover:text-gray-900 hover:dark:text-gray-200');
}
```

**重要**：由于 `SetLocale` 中间件在 `HandleInertiaRequests` 之前执行，`trans()` 调用会使用正确的 locale。

---

## 5. 前端：app.js 中的 i18nVue 初始化

[app.js](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/resources/js/app.js) 是 Vue 应用的入口文件。

### 5.1 i18nVue 配置

```javascript
import { i18nVue } from 'laravel-vue-i18n';

createInertiaApp({
  // ...
  setup({ el, App, props, plugin }) {
    return createSSRApp({
      // ...
    })
      .use(plugin)
      .use(ZiggyVue, window.Ziggy)
      .use(i18nVue, {
        resolve: (lang) => resolvePageComponent(`../../lang/${lang}.json`, import.meta.glob('../../lang/*.json')),
      })
      .use(sentry, props.initialPage.props.sentry)
      .mixin({ methods: { route: window.route, ...methods } })
      .mount(el);
  },
});
```

### 5.2 语言文件加载机制

**语言文件位置**：[lang/*.json](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/lang/)，包括：
- `en.json`, `zh_CN.json`, `zh_TW.json`, `fr.json` 等共28种语言

**加载流程**：
1. `laravel-vue-i18n` 插件检测当前的 locale
2. 通过动态导入 `import.meta.glob('../../lang/*.json')` 预加载所有语言文件
3. 根据当前 locale 加载对应的 `{locale}.json` 文件
4. 初始化 Vue i18n 系统

### 5.3 Locale 变更的前端处理

当用户通过 API 修改 locale 后：
1. 后端返回新的 locale 数据
2. 前端需要调用 `i18nVue` 的 `loadLanguageAsync(locale)` 方法重新加载语言文件
3. 所有使用 `__()` 或 `trans()` 的组件会自动更新翻译

---

## 6. 哪些请求会仍然使用旧 Locale？

### 6.1 API 请求（routes/api.php）

[api.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/monica/routes/api.php) 定义的 API 路由：

```php
$middleware->api(prepend: [
    \Laravel\Sanctum\Http\Middleware\EnsureFrontendRequestsAreStateful::class,
]);
```

**问题**：API 中间件组**不包含** `SetLocale` 中间件！

**影响**：
- API 请求不会自动从用户模型读取 locale
- 始终使用应用默认 locale（通常是 `en`）
- 如果 API 响应需要翻译，会使用默认语言

### 6.2 修改 Locale 请求本身的响应

当调用 `POST /settings/preferences/locale` 时：
1. 请求进入时，`SetLocale` 中间件使用**旧** locale（从用户旧数据读取）
2. `StoreLocale` 服务执行，更新用户 locale 并调用 `App::setLocale()`
3. 响应返回时，应用的 locale 已是新值
4. 但这个请求的**中间件阶段**使用的是旧 locale

### 6.3 并发请求

如果在修改 locale 的同时有其他请求在处理：
- 那些在 locale 写入数据库**之前**启动的请求会使用旧 locale
- 数据库事务隔离级别可能影响读取时机

### 6.4 队列任务 / 命令行

- 队列任务和 Artisan 命令不会经过 HTTP 中间件
- 需要手动设置 locale：`App::setLocale($user->locale)`

### 6.5 WebDAV / 其他特殊路由

- `/dav` 路由可能有独立的中间件配置
- 检查是否排除在 web 中间件组之外

---

## 7. 完整的 Locale 变更流程图

```
用户在前端选择新语言
        ↓
[前端] 发起 POST /settings/preferences/locale
        ↓
[请求进入] SetLocale 中间件 → 读取 user.locale（旧值）
        ↓
PreferencesLocaleController::store()
        ↓
StoreLocale::execute()
  ├─ 验证：Rule::in(supported_locales) ✓
  ├─ 更新 user.locale 到数据库 ✓
  └─ App::setLocale(新locale) ✓
        ↓
返回更新后的用户数据
        ↓
[前端] 收到响应
  ├─ 调用 i18nVue.loadLanguageAsync(新locale)
  └─ 重新加载翻译文件，UI 更新
        ↓
[下一个页面请求]
  ├─ SetLocale 中间件读取新的 user.locale
  ├─ HandleInertiaRequests 使用新 locale 翻译 footer
  └─ 整个应用使用新语言
```

---

## 8. 总结：关键点

| 层面 | 生效时机 | 机制 |
|------|---------|------|
| **后端应用** | StoreLocale 执行后立即生效 | `App::setLocale()` |
| **后续 HTTP 请求** | 下一个请求 | SetLocale 中间件从 user.locale 读取 |
| **Inertia 共享数据** | 下一个请求 | HandleInertiaRequests 中的翻译函数 |
| **前端 UI 翻译** | API 响应后前端主动调用 | `i18nVue.loadLanguageAsync()` |
| **API 接口** | ❌ 不生效 | API 中间件组缺少 SetLocale |
| **队列任务** | ❌ 不生效 | 需要手动设置 |

