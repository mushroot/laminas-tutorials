# 模块

laminas-mvc 使用一个模块系统来组织每个模块中主应用程序特定的代码。skeleton 内置的
`Application` 模块提供整个系统的启动，错误，路由配置信息。通常被用来作为应用主页
的控制器，当时当前教程我们不准备使用默认主页，我们希望专辑列表的注册存在于我们自己
的模块中。

我们要把所有包含控制器、模型、表单和视图，以及配置的的代码放入 Album 模块当中。我们还将
根据需要调整 `Application` 模块。

先从目录结构开始。

## 创建 Album 模块

首先在 `module` 目录下创建一个名为 `Album` 的目录并且按照下面的模式创建好子目录:

```text
laminas-mvc-tutorial/
    /module
        /Album
            /config
            /src
                /Controller
                /Form
                /Model
            /view
                /album
                    /album
```

我们将为不同类型的文件在 `Album` 模块中分离出不同的目录。在 `Album` 命名空间下的 PHP 
文件在 `src/` 目录下。视图目录也存在一个名为 `album` 的子目录用来放置我们模块的视图脚本。

为了加载并配置一个模块，Laminas 提供一个 `ModuleManager` 用来在指定的模块命名
空间(i.e., `Album`)中寻找 `Module` 类。在我们的新模块中，即 `module/Album/src/Module.php`
中的类 `Album\Module` 。

现在我们按照下面的内容来创建这个文件：

```php
namespace Album;

use Laminas\ModuleManager\Feature\ConfigProviderInterface;

class Module implements ConfigProviderInterface
{
    public function getConfig()
    {
        return include __DIR__ . '/../config/module.config.php';
    }
}
```

The `ModuleManager` will call `getConfig()` automatically for us.

### 自动加载

Zend Framework 的 [zend-loader](https://laminas.org.cn/zend-loader) 组件
提供自动加载功能，我们建议使用  Composer 的自动加载功能。例如，我们需要通知 Composer 
一个新命名空间，并让其加载。

在您项目的根目录中打开 `composer.json` ，找到 `autoload` 项；
默认如下所示：

```json
"autoload": {
    "psr-4": {
        "Application\\": "module/Application/src/"
    }
},
```

我们将添加一个新的模块到列表中，添加后如下：

```json
"autoload": {
    "psr-4": {
        "Application\\": "module/Application/src/",
        "Album\\": "module/Album/src/"
    }
},
```

一旦我们修改了配置，需要运行如下的代码来更新自动记载规则：

```bash
$ composer dump-autoload
```

## 配置

注册了自动加载后，再来查看 `Album\Module` 中的 `getConfig()`，当前方法加载 我们模块
根目录中的 `config/module.config.php` 文件。

在 `zf2-tutorial/module/Album/config/` 中创建名为 `module.config.php` 的文件：

```php
namespace Album;

use Laminas\ServiceManager\Factory\InvokableFactory;

return [
    'controllers' => [
        'factories' => [
            Controller\AlbumController::class => InvokableFactory::class,
        ],
    ],
    'view_manager' => [
        'template_path_stack' => [
            'album' => __DIR__ . '/../view',
        ],
    ],
];
```

配置信息通过 `ServiceManager` 传递给相应的组件。我们需要初始化 `controllers` 和
`view_manager` 两个部分。controllers部分列出模块内可被调用的控制器列表。我们现在
需要使用 `AlbumController` 控制器；我们将使用完整类名来引用他，并且使用zend-servicemanager
的 `InvokableFactory` 来为他创建一个实例。

`view_manager` 部分中，我们将添加我们的视图目录到 `TemplatePathStack` 配置中。
这将允许程序通过 `Album` 模块的 `view/` 目录找到视图文件。

## 将新模块告知给应用

我们需要告知 `ModuleManager` 新模块的存在。这需要在骨架中提供的 `config/modules.config.php`
文件中操作。更新其中的数组，使其包含`Album`模块，如下所示：

( 更改提示行，文件原始注释信息已省略 )

```php
return [
    'Laminas\Form',
    'Laminas\Db',
    'Laminas\Router',
    'Laminas\Validator',
    'Application',
    'Album',          // <-- Add this line
];
```

如此，我们已经在 `Application` 模块下添加了一个 `Album` 列表。

现在，我们已经将我们自己的代码添加到了模块中。
