# 高级配置技巧

laminas-mvc 应用的配置分如下几个步骤：

- 初始化配置并将其传递给 `Application` 实例，
  同时分发给  `ModuleManager` 和 `ServiceManager`。
  本教程中，我们将这种配置称之为 **系统配置**。
- `ModuleManager` 的 `ConfigListener` 会将配置进行聚合并在模块加载的时候合并。
  在本教程中，我们将其称之为 **应用配置**。
- 一旦从所有模块中聚合了配置，`ConfigListener`
  还会将制定目录（ 通常是 `config/autoload/`）的应用配置合并。
- 最后在将应用的配置传递给 `ServiceManager` 之前，
  还会将其传递给制定的 `EVENT_MERGE_CONFIG` 事件，以允许将来做其他更改。

在本教程中，我们将研究具体的排列，以及如何将其绑定到应用中。

## 系统配置

模块加载的时候，我们需要告诉 `Application` 实例哪些模块需要加载，
同时还可以向默认模块的监听器提供一个信息
（例如：应用配置的位置以及要加载的文件；配置是否缓存以及缓存的位置等等）
还可以选择将配置传递给 `ServiceManager`。
在本教程中，我们称之为 ***系统配置**。

当使用骨架应用的时候， **系统配置** 默认在
`config/application.config.php` 文件中。
默认内容如下：

```php
return [
    // Retrieve list of modules used in this application.
    'modules' => require __DIR__ . '/modules.config.php',

    // These are various options for the listeners attached to the ModuleManager
    'module_listener_options' => [
        // use composer autoloader instead of laminas-loader
        'use_laminas_loader' => false,
        
        // An array of paths from which to glob configuration files after
        // modules are loaded. These effectively override configuration
        // provided by modules themselves. Paths may use GLOB_BRACE notation.
        'config_glob_paths' => [
            realpath(__DIR__) . '/autoload/{{,*.}global,{,*.}local}.php',
        ],

        // Whether or not to enable a configuration cache.
        // If enabled, the merged configuration will be cached and used in
        // subsequent requests.
        'config_cache_enabled' => true,

        // The key used to create the configuration cache file name.
        'config_cache_key' => 'application.config.cache',

        // Whether or not to enable a module class map cache.
        // If enabled, creates a module class map cache which will be used
        // by in future requests, to reduce the autoloading process.
        'module_map_cache_enabled' => true,

        // The key used to create the class map cache file name.
        'module_map_cache_key' => 'application.module.cache',

        // The path in which to cache merged configuration.
        'cache_dir' => 'data/cache/',

        // Whether or not to enable modules dependency checking.
        // Enabled by default, prevents usage of modules that depend on other modules
        // that weren't loaded.
        // 'check_dependencies' => true,
    ],

    // Used to create an own service manager. May contain one or more child arrays.
    // 'service_listener_options' => [
    //     [
    //         'service_manager' => $stringServiceManagerName,
    //         'config_key'      => $stringConfigKey,
    //         'interface'       => $stringOptionalInterface,
    //         'method'          => $stringRequiredMethodName,
    //     ],
    // ],

    // Initial configuration with which to seed the ServiceManager.
    // Should be compatible with Laminas\ServiceManager\Config.
    // 'service_manager' => [],
];
```

系统配置适用于在应用程序就绪之前配置 MVC 的相关细节。
这些配置通常比较简短。

此外，系统配置会被 *立即* 使用，并且不会和其他配置合并
&mdash; 这就意味着除了 `service_manager` 中的值意外，他将不能被别的模块重写。

这里我们将提供一个技巧：如何提供特定环境的配置？

### 特定系统环境的配置

当你在开发环境的时候想要对模块进行一些配置如何操作呢？
或者如果需要在开发环境中禁用配置缓存？

这就是为什么我们的骨架应用提供的摸摸人配置是 PHP;
提供 PHP 配置就意味着你可以通过编程的方式来做修改。

例如，我们可能有如下需求：

- 我们只想要在开发环境中使用 `Laminas\\DeveloperTools` 模块。
- 我们只想在生产环境中开启配置缓存。

[laminas/laminas-development-mode](https://github.com/laminas/laminas-development-mode)
提供了一个简洁且公认的方法来对生产环境和开发环境进行切换。
当前包默认在 3+ 版本进行了安装，在 V2 版本中可以通过如下方式安装：

```bash
$ composer require laminas/laminas-development-mode
```

它采用如下的实现方式：

- 在 `config/application.config.php` 中提供生产环境的信息。
- 在 `config/development.config.php.dist` 中配置开发环境的配置信息以覆盖引导层的设置，如模块及缓存信息。我们同样可以选择在 `config/autoload/development.local.php.dist` 覆盖应用的配置信息。
- 引导脚本（`public/index.php`）将检查 `config/development.config.php`，如果存在则将其与应用的配置文件合并并传递给 `应用`。

接下来我们执行：

```bash
$ ./vendor/bin/laminas-development-mode enable
```

`.dist` 文件将会被复制为一个不带后缀的版本；这样做的目的是保证其能被应用所引用。

因此为了实现我们的最终目标，我们将做如下操作：

- 在 `config/development.config.php.dist` 中添加 `ZendDeveloperTools` 到模块列表中：

  ```php
  'modules' => [
      'LaminasDeveloperTools',
  ],
  ```

- 同样我也可以在 `config/development.config.php.dist` 中禁用缓存：

  ```php
  'config_cache_enable' => false,
  ```


- 在 `config/application.config.php` 中启用缓存：

  ```php
  'config_cache_enable' => true,
  ```


启用开发模式将会启用指定的模块，并禁用配置文件的缓存；禁用开发模式将会开启配置缓存（同样，缓存文件也讲会被清理）。


如果你希望继续扩展环境，可以通过扩展 laminas-development-mode 来实现。

### 特定环境的应用配置

有时间你需要修改应用的配置文件以加载数据库适配器，日志记录，缓存适配器等等。
这通常在服务管理器中定义，同时可能在模块中定义。
你可以用过 `Laminas\ModuleManager\Listener\ConfigListener` 
在应用层来进行修改。通过**系统配置**的一个特殊的全局参数 &mdash; 之前示例中的 
`module_listener_options.config_glob_paths`。

其默认参数为 `config/autoload/{{,*.}global,{,*.}local}.php`。
这将意味着在 `config/autoload` 目录中寻找如下所示的 **应用配置**：

- `global.php`
- `*.global.php`
- `local.php`
- `*.local.php`

这种方式允许你在 "全局" 配置文件中定义应用层的默认值，
以便你可以将其提交到版本控制系统中，
转而在特定环境中使用 "local" 文件来保存参数，
以便其不会被提交到版本控制系统中。

> ### 为开发模式增加 glob 匹配模式
>
> 当我们使用 laminas-development-mode 时，
> 如上面的章节所述，`config/development.config.php.dist` 
> 文件为特定的开发模式提供了一个 glob 匹配模式:
>
> - `config/autoload/{,*.}{global,local}-development.php`
>
> 这将匹配如下这种类型的文件：
>
> - database.global-development.php
> - database.local-development.php
>
> 这将只在开发模式下引用！

这对开发来说是一个比较好的解决方案，
这样就可以让你随时切换为指定的配置,
特别对于开发环境来说就避免了需要小心翼翼的来配置了。

然而，如果你想要更多的环境 &mdash; 
例如 "测试" 或 "临时" 环境 &mdash; 
也可以通过同样的方式来实现么？

为了实现这个目的，我们需要在web服务器的配置中设置一个 *环境变量*，`APP_ENV`。
在 Apache中，你需要添加如下命令在系统全局 apache.conf 
或 http.conf 或者 virtual honst 文件中;
甚至可以将其放置在 .htaccess 文件中。

```apacheconf
SetEnv "APP_ENV" "development"
```

对于其他web服务器来说，可以参考其文档而定义环境变量。

一般来说，如果环境变量不存在，我们就假设其为 "production"。

有了它，我们就可以在系统配置文件中更改 glob 路径，如下：

```php
'config_glob_paths' => [
    realpath(__DIR__) . sprintf('/autoload/{,*.}{global,%s,local}.php', getenv('APP_ENV') ?: 'production')
],
```

上面的代码允许你为每个配置环境定义一份系统的配置信息；而且 *只* 在环境生效的时候生效！


参照如下单三个配置文件：

```text
config/
    autoload/
        global.php
        local.php
        users.development.php
        users.testing.php
        users.local.php
```

如果 `$env` 判定为 `testing`，如下文件将会合并：

```text
global.php
users.testing.php
local.php
users.local.php
```

注意 `users.development.php` 将不会被加载 &mdash; 因为其不符合 glob 规则！

另外，由于加载顺序不同，你可以知道哪个文件的参数会覆盖另一个文件的参数，
也可以选择性的覆盖一些值作为 debug 使用。

> #### 配置合并顺序
>
> `config/autoload/` 中的文件会在模块的配置 *之后* 合并，
> 详细的会在接下来的章节中讨论。
> 这里我们详细介绍了在**系统配置**（`config/application.config.php`）
> 中配置**应用配置**的 glob 路径。

## 模块配置

模块需要向应用提供他们自己的配置信息的。模块有两种机制来实现。

**首先**，模块需要继承 `Laminas\ModuleManager\Feature\ConfigProviderInterface`
并实现一个 `getConfig()` 方法用来返回其配置信息。通常， 
`getConfig()` 方法的实现方式如下：

```php
public function getConfig()
{
    return include __DIR__ . '/config/module.config.php';
}
```

`module.config.php` 返回一个 PHP 数组。通过这个数组，可以提供一些基本的配置，以及 ServiceManager 提供的 `Manager` 类的配置。 参见
[配置列表名录](#configuration-mapping-table) 查看哪些名次对应哪些 `Manager`。

**其次**，模块可以继承一些接口和服务管理器相关的方案或查件管理器的配置信息。
你可以在[配置列表名录](#configuration-mapping-table) 中找到所有接口以及其
对应的模块配置功能。

大多数的结构都在 `Laminas\ModuleManager\Feature` 命名空间下（少数移到了私人模块中），
都将返回一个为服务管理器的数组，如
[默认服务配置](docs.laminas.dev/laminas-mvc/services/#servicemanager).

## 配置列表名录

Manager name               | Interface name                      | Module method name            | Config key name
-------------------------- | ----------------------------------- | ----------------------------- | ---------------
`ControllerPluginManager`  | `ControllerPluginProviderInterface` | `getControllerPluginConfig()` | `controller_plugins`
`ControllerManager`        | `ControllerProviderInterface`       | `getControllerConfig()`       | `controllers`
`FilterManager`            | `FilterProviderInterface`           | `getFilterConfig()`           | `filters`
`FormElementManager`       | `FormElementProviderInterface`      | `getFormElementConfig()`      | `form_elements`
`HydratorManager`          | `HydratorProviderInterface`         | `getHydratorConfig()`         | `hydrators`
`InputFilterManager`       | `InputFilterProviderInterface`      | `getInputFilterConfig()`      | `input_filters`
`RoutePluginManager`       | `RouteProviderInterface`            | `getRouteConfig()`            | `route_manager`
`SerializerAdapterManager` | `SerializerProviderInterface`       | `getSerializerConfig()`       | `serializers`
`ServiceLocator`           | `ServiceProviderInterface`          | `getServiceConfig()`          | `service_manager`
`ValidatorManager`         | `ValidatorProviderInterface`        | `getValidatorConfig()`        | `validators`
`ViewHelperManager`        | `ViewHelperProviderInterface`       | `getViewHelperConfig()`       | `view_helpers`
`LogProcessorManager`      | `LogProcessorProviderInterface`     | `getLogProcessorConfig`       | `log_processors`
`LogWriterManager`         | `LogWriterProviderInterface`        | `getLogWriterConfig`          | `log_writers`

## 优先配置

想象下在模块配置文件中有多个服务配置文件，如何来判定优先权？

其默认合并顺序为：

- 模块中的各种服务配置方法返回的配置
- `getConfig()` 返回的配置

换句话说，`getConfig()` 优先级高于各种服务配置方法。
另外，需要特别注意的是：这些方法返回的配置是 *无法* 缓存的。

> ### 服务配置方法用法
>
> 当你在定义闭包或工厂回调，静态工厂以及初始化应用程序的时候需要各种服务配置方法。
> 这样就可以避免缓存的问题，同时还能通过其他的标记格式写入配置。

## 操作合并的配置

有时，你不仅要覆盖应用程序指定配置的键，而且要删除它。
由于合并操作并不会删除当前键，那么我们需要如何来实现呢？

`Laminas\ModuleManager\Listener\ConfigListener` 将会在合并所有的事件之后
触发一个特殊的事件，`Laminas\ModuleManager\ModuleEvent::EVENT_MERGE_CONFIG`，
且在传递给 `ServiceManager` 之前触发。通过监听这个事件，你可以检查并操作合并的配置。

`ConfigListener` 本身在合并配置的时候监听这个事件的优先级为 1000 （你可以理解为很高）。你可以通过当前模块的 `init()` 方法对合并后的配置进行修改。

```php
namespace Foo;

use Laminas\ModuleManager\ModuleEvent;
use Laminas\ModuleManager\ModuleManager;

class Module
{
    public function init(ModuleManager $moduleManager)
    {
        $events = $moduleManager->getEventManager();

        // Registering a listener at default priority, 1, which will trigger
        // after the ConfigListener merges config.
        $events->attach(ModuleEvent::EVENT_MERGE_CONFIG, [$this, 'onMergeConfig']);
    }

    public function onMergeConfig(ModuleEvent $e)
    {
        $configListener = $e->getConfigListener();
        $config         = $configListener->getMergedConfig(false);

        // Modify the configuration; here, we'll remove a specific key:
        if (isset($config['some_key'])) {
            unset($config['some_key']);
        }

        // Pass the changed configuration back to the listener:
        $configListener->setMergedConfig($config);
    }
}
```

在这里，合并后的应用配置就不会包含 `some_key` 了。

> ### 缓存配置的合并
>
> 如果 `ModuleManager` 缓存了配置，`EVENT_MERGE_CONFIG` 事件将不会被触发。
> 但是这也就意味着配置将会是监听器最初处理的内容。

## 合并配置的工作流程

在本教材结束的的时候，让我们来回顾下配置是何时何地定义以及合并的。


- **系统配置**
  - 在 `config/application.config.php` 中定义
  - 无合并需求
  - 可编程，以及：
    - 根据计算状态更换标签
    - 根据计算状态更换配置的 glob 路径。
    - 配置将会传递给 `Application` 实例，紧接着 `ModuleManager` 将会用来初始化系统。
- **应用配置**
  - `ModuleManager` 遍历 **系统配置** 中的每个模块类
    - 汇总 `Module` 类中定义的服务配置
    - 汇总 `Module::getConfig()` 返回的配置
    - 从**服务配置** 的 `config_glob_paths` 中查找配置文件，并按照他们再全局路径中的顺序进行合并。
    - `ConfigListener` 触发 `EVENT_MERGE_CONFIG` 事件:
      - `ConfigListener` 合并配置
      - 处理其他监听器对配置的操作
    - 最后通过 `ServiceManager` 合并配置
