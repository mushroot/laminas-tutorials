# 介绍博客模块

现在我们已经知晓了 zend-mvc 框架应用的基本操作，
接下来让我们继续创建我们自己的模块，我们将创建一个名为 "Blog" 的模块。
这个模块将会显示一个表示单个博客帖子的数据库条目的列表，每个帖子将会有三个属性
`id`, `text`, and `title`。我们将会创建一个表单用来天机的新的帖子到数据库中
以及用来编辑一个存在的帖子。此外我们将使用最佳方案来实现这个教程。

## 编写一个新的模块

让我们在 `/module` 目录下创建一个名为 `Blog` 的新目录，并建立如下的目录结构：

```text
module/
    Blog/
        config/
        src/
        view/
```

在能够被
[ModuleManager](https://docs.laminas.dev/laminas-modulemanager/intro/),
识别为一个模块之前，我们需要做三件事：

- 告诉 Composer 怎么自动去加载我们新模块的类。
- 在 `Blog` 命名空间内创建一个 `Module` 类。
- 将新模块告知给应用。

让我们告知 Composer 我们的新模块。打开项目根目录中的 `composer.json` 文件，
编辑 `autoload` 选项，为 `Blog` 模块添加一个新的 PSR-4 实体；完成后将会和如下格式一样：

```json
"autoload": {
   "psr-4": {
        "Application\\": "module/Application/src/",
        "Album\\": "module/Album/src/",
        "Blog\\": "module/Blog/src/"
   }
}
```

一旦完成上面的操作，我们就需要跟新 Composer 自动加载项：

```bash
$ composer dump-autoload
```

接下来，我们将在 `Blog` 命名空间下创建一个 `Module` 类。创建一个如下内容的文件
`module/Blog/src/Module.php`：

```php
<?php
namespace Blog;

class Module
{
}
```

现在我们已经配置好模块可以被
[ModuleManager](https://docs.laminas.dev/laminas-modulemanager/intro/).
检测到了。接下来我们将模块添加到应用中。尽管我们的模块现在不能进行任何操作，
同时也仅仅只存在 `Module.php` 类能够被模块管理器加载。
我们需要在 `config/modules.config.php` 文件中添加 `Blog` 实体到模块数组中：

```php
<?php
// In config/modules.config.php:

return [
    /* ... */
    'Application',
    'Album',
    'Blog',
];
```

如果你现在刷新你的应用你讲不会看见任何的改变（但是也没有任何的错误出现）。

这时我们可以退一步来讨论下什么事模块。简单来说，模块就是对你应用一个功能集合的封装。
正如你看见的那样一个模块可以向应用中添加一个功能，就像我们的 `Blog` 模块那样；
或者他可以一个后台的功能共应用中其他的模块来调用，就像与第三方 API 交互那样。

组织你的代码到模块中使其在其他应用中可以很方便的重用，或者使用由社区编写的模块。

## 配置模块

接下来我们将为我们的应用添加一个路由，以便我们可以通过链接 `localhost:8080/blog`
来访问我们的模块。我们通过在添加路由配置到我们的模块中来实现这个需求，但是首先我们需要告知
`ModuleManager` 以及配置好了需要加载。

我们通过在 `Module` 类中添加一个 `getConfig()` 方法来返回配置信息。
（这个方法已经在 `ConfigProviderInterface` 中定义，虽然实现这个接口是可选的。）
这个方法需要返回一个 `array` 或者一个 `Traversable` 对象。
我们继续编辑 `module/Blog/src/Module.php`：

```php
// In /module/Blog/Module.php:
class Module
{
    public function getConfig()
    {
        return [];
    }
}
```

这样，我们就可以开始配置我们的模块了。配置文件可能变得比较庞大，而且在所有的配置都放在
`getConfig()` 方法中也不是一个最优的解决方案。为了保证我们项目的可维护性，
我们将将我们的配置文件放入一个独立的文件中。创建
`module/Blog/config/module.config.php` 文件：

```php
<?php
return [];
```

现在我们重写 `getConfig()` 函数来包含我们新创建的文件来返回一个数组：

```php
<?php
// In /module/Blog/Module.php:

public function getConfig()
{
    return include __DIR__ . '/../config/module.config.php';
}
```

刷新你的引用，你任然还是看不见任何的变化。接下来我们将添加一个新的路由到我们的配置文件中：

```php
// In /module/Blog/config/module.config.php:
namespace Blog;

use Laminas\Router\Http\Literal;

return [
    // 该行开始为路由管理器配置路由信息
    'router' => [
        // 开始配置所有可能的路由
        'routes' => [
            // 定义一个名为 "blog" 的路由
            'blog' => [
                // 定义路由类型为 "literal" :
                'type' => Literal::class,
                // 配置路由信息
                'options' => [
                    // 监听 "/blog" :
                    'route' => '/blog',
                    // 定义当路由匹配的时候默认的控制器和操作
                    'defaults' => [
                        'controller' => Controller\ListController::class,
                        'action'     => 'index',
                    ],
                ],
            ],
        ],
    ],
];
```

我们定义了一个名为 `blog` 的路由来监听链接 `localhost:8080/blog`。
当有人访问这个路由的时候，`Blog\Controller\ListController` 类的
`indexAction()` 函数将会被执行。然而现在这个控制器还不存在，
因此当你刷星这个页面的时候，你将会看见如下的错误信息：

```text
A 404 error occurred
Page not found.
The requested controller could not be mapped by routing.

Controller:
Blog\Controller\ListController(resolves to invalid controller class or alias: Blog\Controller\ListController)
```

现在我们需要告诉我们的模块在哪里可以找到名 `Blog\Controller\ListController` 的控制器。
我们需要在 `module/Blog/config/module.config.php` 这个配置文件中添加
`controllers` 这个配置信息来实现这个功能。

```php
namespace Blog;

use Laminas\ServiceManager\Factory\InvokableFactory;

return [
    'controllers' => [
        'factories' => [
            Controller\ListController::class => InvokableFactory::class,
        ],
    ],
    /* ... */
];
```

这个配置使用 zend-servicemanager `InvokableFactory`
为 `Blog\Controller\ListController` 控制器类定义了一个工厂
（内部会对类进行一个不带参数的实例化）。重载这个页面我们将会看见如下信息：

```text
Fatal error: Class 'Blog\Controller\ListController' not found in {projectPath}/vendor/laminas/laminas-servicemanager/src/Factory/InvokableFactory.php on line 32
```

这个错误信息告诉我们应用已经知晓需要去加载这个类了，但是不能找到这个类。
在这个实例中，我们已经设置了自动加载，但是我们还没有定义这个控制器类！

根据如下内容创建文件  `module/Blog/src/Controller/ListController.php`：

```php
<?php
namespace Blog\Controller;

class ListController
{
}
```

重载页面我们将会看见一个新的页面，这个心得错误信息如下：

```html
A 404 error occurred
Page not found.
The requested controller was not dispatchable.

Controller:
Blog\Controller\List(resolves to invalid controller class or alias: Blog\Controller\List)

Additional information:
Laminas\ServiceManager\Exception\InvalidServiceException

File:
{projectPath}/vendor/laminas/laminas-mvc/src/Controller/ControllerManager.php:{lineNumber}

Message:
Plugin of type "Blog\Controller\ListController" is invalid; must implement Laminas\Stdlib\DispatchableInterface
```

这是因为我们的控制器必须继承接口
[DispatchableInterface](https://github.com/laminas/laminas-stdlib/blob/0e66240daec93f19ae104084e369c393453fc901/src/DispatchableInterface.php)
以便 laminas-mvc 分发（或者运行）。 laminas-mvc 提供了一个基本的接口实现
[AbstractActionController](https://github.com/laminas/laminas-mvc/blob/9fc7f2cf569760d6a0cdc62f56e042a0aee3a4c9/src/Controller/AbstractActionController.php),
供我们使用。现在我们修改我们的控制器：

```php
// In /module/Blog/src/Blog/Controller/ListController.php:

namespace Blog\Controller;

use Laminas\Mvc\Controller\AbstractActionController;

class ListController extends AbstractActionController
{
}
```

让我们再次刷新站点，你将会看见如下的错误信息：

```text
An error occurred

An error occurred during execution; please try again later.

Additional information:

Laminas\View\Exception\RuntimeException

File:
{projectPath}/vendor/laminas/laminas-view/src/Renderer/PhpRenderer.php:{lineNumber}

Message:
Laminas\View\Renderer\PhpRenderer::render: Unable to render template "blog/list/index"; resolver could not resolve to a file
```

现在应用告诉你不能渲染视图脚本，这是预料中的，因为我们还没有创建视图脚本文件。
应用默认的位置是 `module/Blog/view/blog/list/index.phtml`。
让我们创建这个文件，并且添加啊一些输出信息：

```html
<!-- Filename: module/Blog/view/blog/list/index.phtml -->
<h1>Blog\Controller\ListController::indexAction()</h1>
```

在我们继续下一步之前，让我们看看这个文件所处的位置。提示信息显示的是文件不能在 `/view`
子目录下被找到，不是作为一个存在于 `/src` 目录下的 PHP 的类文件，
而是作为一个被渲染为 HTML 的模板文件。然而，这里需要做一些说明。
首先是瞎写的命名空间名称，其次是小写的控制器名称（不带 'controller' 后缀），
最后紧接的是我访问的操作名（不包含 'action' 后缀）。你可以认为他是
`view/{namespace}/{controller}/{action}.phtml` 这样的模板字符串。
这是一个社区的标准，但是如果你需要的话你也可以自定去定义路径。

然而单单创建这个文件是不足以完成这部分教程的。我们需要让我们的应用知晓在哪里去寻找视图文件。
我们需要在 `module.config.php` 配置文件中去实现这个配置。

```php
// In module/Blog/config/module.config.php:

return [
    'controllers' => [ /** Controller Configuration */ ],
    'router'      => [ /** Route Configuration */ ]
    'view_manager' => [
        'template_path_stack' => [
            __DIR__ . '/../view',
        ],
    ],
];
```

上面的配置信息告诉应用 `module/Blog/view/` 中存在满足
`view/{namespace}/{controller}/{action}.phtml`
规则的视图文件。
值得注意的是 `view_manager` 配置不仅仅允许你为模块配置视图文件，
还可以重写其他模块的视图文件。

刷新站点。最终我们将会看见一个没有错误信息的显示了！
你将会在视图中看见 **Blog\Controller\ListController::indexAction()**。

祝贺你，
你不但创建一个简单的 "Hello World" 模块，同时你也知晓了一些常见的错误以及其处方法。
如果你学起来不是那么费劲的话，就跟着教程，创建一个能做真实事情的模块。
