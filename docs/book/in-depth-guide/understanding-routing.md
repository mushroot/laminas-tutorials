# 了解路由

我们的模块顺利进行的同时，我们并不能做到完全的比配；
确切的说，我们总是在整个页面上展示 *整个* 博客。
本章节中，你将会学习到关于 `Router` 的所有用法，
以便我们能够导航到控制器以及方法中，用来展示一个独立的模块，
添加一个新的帖子，编辑一个存在的帖子，删除一个帖子这些功能。

## 不通的路由类型

在详细介绍我们的应用之前，让我们了解几个常用的路由类型。

### Literal 路由

如之前章节展示的那样， 一个 literal 路由表示匹配与指定字符串完全匹配的路径。
下面是一些使用 literal 路由的示例：

- `http://domain.com/blog`
- `http://domain.com/blog/add`
- `http://domain.com/about-me`
- `http://domain.com/my/very/deep/page`

配置一个 literal 路由需要提供一个匹配的路径以及一个 "defaults" 属性用来返回匹配参数；
一般是用来指定控制器以及其操作方法。如下：

```php
'router' => [
    'routes' => [
        'about' => [
            'type' => \Laminas\Router\Http\Literal::class,
            'options' => [
                'route'    => '/about-me',
                'defaults' => [
                    'controller' => 'AboutMeController',
                    'action'     => 'aboutme',
                ],
            ],
        ],
    ],
],
```

### Segment 路由

Segment 允许你使用变量参数来定义路由；通常哟弄过来在路径中指定ID。
segment 路由的示例 URLs 如下所示：

- `http://domain.com/blog/1` (parameter "1" is dynamic)
- `http://domain.com/blog/details/1` (parameter "1" is dynamic)
- `http://domain.com/blog/edit/1` (parameter "1" is dynamic)
- `http://domain.com/blog/1/edit` (parameter "1" is dynamic)
- `http://domain.com/news/archive/2014` (parameter "2014" is dynamic)
- `http://domain.com/news/archive/2014/january` (parameter "2014" and "january"
   are dynamic)

配置一个 segment 路由与 literal 类似。
不同点在于：

- 路由可能会有一个或者多个 `:<变量>` 段，用于匹配变量。
  `:<变量>` 应为一个字符串，并且需要作为在路由分配成功的时候引用变量的标识。
- 路由同样 *可能* 包含 *可选* 的部分，并使用大括号 (`[]`) 将其包含在其中，
  且可以在其中使用字符以及可选的部分。
- "defaults" 中可以包含可变部分的名称；一遍在没有传递参数的时候使用默认的参数。
  (定义的时候应该也保持独立，例如，"controller" 通常就不建议包含在可选部分中。)
- 同时你可以在 "constraints" 中对每个变量进行约束；
  每个约束条件就是一个正则表达式，并且会在路由匹配成功的时候触发。

举个例子，我们需要一个路由指定 "year" 部分为变量，并且限制其必须为4个数字；
当匹配的时候，我们将使用 `ArchiveController` 的 `byYear` 操作：

```php
'router' => [
    'routes' => [
        'archives' => [
            'type' => \Laminas\Router\Http\Segment::class,
            'options' => [
                'route'    => '/news/archive[/:year]',
                'defaults' => [
                    'controller' => ArchiveController::class,
                    'action'     => 'byYear',
                    'year'       => date('Y'),
                ],
                'constraints' => [
                    'year' => '\d{4}',
                ],
            ],
        ],
    ],
],
```

当前配置定义了一个可以匹配类似 `//example.com/news/archive/2014` 的路由。
路由指定了变量段 `:year`，并定义了其约束条件为  `\d{4}`，
表示其只有当其为 4 个数字的时候才会匹配。
例如：`//example.com/news/archive/123` 将不能匹配，
但是 `//example.com/news/archive/1234` 就可以。

该定义标记了一个用 `[/:year]` 表示的可选部分。这有两种含义。
首先意味着它可以匹配如下的URL：

- `//example.com/news/archive`
- `//example.com/news/archive/`

这两种情况下，我们同样可以获取可选部分 `:year` 的值，
因为我们为其定义了一个默认的参数 `date('Y')` （返回当前的年份）。

Segment 路由允许你动态的匹配路径，并且提供了多种方式来定义这些路由，
定义默认值，以及提供约束条件。

## 不同路由的概念

当我们规划整个应用的时候，你可能会有很多的路由需要去定义。
编写这些路由的时候我们有两种选择：

- 花更少的时间来编写通用的路由，但是这样可能会导致匹配的速度变慢。
- 编写一个非常明确的路由，这样可以使其有更快的匹配速度，但是可能会在定义路由上花费大量的时间。

### 通用路由

通用路由是非常贪婪的，他会常识去匹配尽可能多的 URLs。
一种常用的方法是编写自动匹配控制器和操作的路由：

```php
'router' => [
    'routes' => [
        'default' => [
            'type' => \Laminas\Router\Http\Segment::class,
            'options' => [
                'route'    => '/[:controller[/:action]]',
                'defaults' => [
                    'controller' => Application\Controller\IndexController::class,
                    'action'     => 'index',
                ],
                'constraints' => [
                    'controller' => '[a-zA-Z][a-zA-Z0-9_-]*',
                    'action'     => '[a-zA-Z][a-zA-Z0-9_-]*',
                ],
            ],
        ],
    ],
],
```

来我们来详细的了解下此配置中定义的内容。
`route` 部分定义了两个可选的部分，`controller` 和 `action`.
`action` 部分只有在在  `controller` 部分匹配的情况才才会生效。
两部分都设置了约束条件以确保其允许满足 PHP 规范的类和方法名。

这种方式最大的优点就在于在开发的时候可以节省大量的时间；
只需要一个路由你就可以直接开始创建控制器并且向其中添加方法。

其缺点就在于。

为了确保其可以运作，你需要在你定义控制器的时候使用别名，
这些简短的别名将命名空间省略为符合规范的控制器名称；
这样做的话就可能因为在不同模块中定义了相同的别名而引发冲突。

其次，使用嵌套的匹配方式可能会引起不必要的性能开销。

再次，此类路由不会其他的附加参数，从而导致了控制器会忽略动态路由部分。
而是需要通过查询字段来获取参数 &mdash; 也就是说参数的验证就留给了控制器。

最后，不能保证有效的匹配到理想的控制器和操作。
例如，一个请求为 `//example.com/strange/nonExistent`，
但是没有指向 `strange` 的控制器，或 `nonExistentAction()` 方法，
应用将会耗费更多的时间和资源来判断是否出错。
这既是性能上的考虑，也是处于安全方面的考虑，因为攻击者可能会利用这种情况来实施攻击。

### 基本路由

现在，你应该就能明白尽量避免使用通用的路由了，尽管其很方便。
这就意味着你需要定义明确的路由。

最好的方法是为每一种情况定义一个路由：

```php
'router' => [
    'routes' => [
        'news' => [
            'type' => \Laminas\Router\Http\Literal::class,
            'options' => [
                'route'    => '/news',
                'defaults' => [
                    'controller' => NewsController::class,
                    'action'     => 'showAll',
                ],
            ],
        ],
        'news-archive' => [
            'type' => \Laminas\Router\Http\Segment::class,
            'options' => [
                'route'    => '/news/archive[/:year]',
                'defaults' => [
                    'controller' => NewsController::class,
                    'action'     => 'archive',
                ],
                'constraints' => [
                    'year' => '\d{4}',
                ],
            ],
        ],
        'news-single' => [
            'type' => \Laminas\Router\Http\Segment::class,
            'options' => [
                'route'    => '/news/:id',
                'defaults' => [
                    'controller' => NewsController::class,
                    'action'     => 'detail',
                ],
                'constraints' => [
                    'id' => '\d+',
                ],
            ],
        ],
    ],
],
```

路由是作为一个堆栈存储的，也就意味着其先进后出（LIFO）.
那么我们建议你首先定义最普通的路由，然后再定义特定的路由。
在上面的例子中，我们最普通的路由指向路径 `/news`。
然后我们添加了两个特定的路由，一个匹配 `/news/archive`（附带一个参数 year）
另一个匹配 `/news/:id`。这些路由具有一些重复的部分：

- 为了防止名称间的冲突，我们对每个路由都使用前缀 `news-`。
- 每个路由都包含字符串 `/news`。
- 每个路由都有相同的控制器。

很明显，这样就显得比较乏味。另外，如果你具有多个这样重复的路由，
就需要特别注意堆栈，可能会出现路由器的重叠，同时还可能会影响性能（如果堆栈变大的话）。

### 子路由

为了解决上面出现的问题，laminas-router 允许定义“子路由”。
子路由继承了各自父路由的所有 `options`；
这就意味着如果存在和父路由相同的配置，那么就不需要重复去定义了。

另外，子路由需要 *相对* 匹配父路由。者也提供了一些优化：

- 你不用重复定义公共路径部分。
- *除非父路由匹配*，否则将会直接忽略子路由，这可以在路由匹配过程中提供巨大的优势。

让我们看下和上面一样的效果，但是使用了子路由的配置：

```php
'router' => [
    'routes' => [
        'news' => [
            // First we define the basic options for the parent route:
            'type' => \Laminas\Router\Http\Literal::class,
            'options' => [
                'route'    => '/news',
                'defaults' => [
                    'controller' => NewsController::class,
                    'action'     => 'showAll',
                ],
            ],

            // The following allows "/news" to match on its own if no child
            // routes match:
            'may_terminate' => true,

            // Child routes begin:
            'child_routes' => [
                'archive' => [
                    'type' => \Laminas\Router\Http\Segment::class,
                    'options' => [
                        'route'    => '/archive[/:year]',
                        'defaults' => [
                            'action' => 'archive',
                        ],
                        'constraints' => [
                            'year' => '\d{4}',
                        ],
                    ],
                ],
                'single' => [
                    'type' => \Laminas\Router\Http\Segment::class,
                    'options' => [
                        'route'    => '/:id',
                        'defaults' => [
                            'action' => 'detail',
                        ],
                        'constraints' => [
                            'id' => '\d+',
                        ],
                    ],
                ],
            ],
        ],
    ],
],
```

最基本的操作是，我们和之前一样定义一个父路由，然后添加属性 `child_routes`，
这是基本的路由配置，用于在在父路由匹配时再去匹配其他路由。

`may_terminate` 配置用于确定是否允许父路由自行匹配；
换句话说，如果没有子路由满足条件，那么是否父路由就为有效的匹配路由？
其默认值为 `false`，将其修改为 `true` 即可允许父路由自行匹配。

`child_routes` 就其本身来说也是一个标准的路由；同样的其可以拥有自己的子路由！
需要注意的是，任何路由的字符串都是相对于其父路由的。
因此，以上的路由可以匹配下面的任一一种路由：

- `/news`
- `/news/archive`
- `/news/archive/2014`
- `/news/42`

（如果将 `may_terminate` 设置了为 `false`，第一个路径 `/news` 将不会匹配）

你可能会注意到，上面所有的子路由都未指定默认 `controller`。
子路由从父路由 *继承* 了参数，这就意味着所有的路由都与父路由使用相同的控制器！

使用子路由的优势包括：

- 明确的路由意味着在与控制器与方法的匹配中产生的错误情况更少。
- 性能；除非父路由匹配，否则子路由将会被直接忽略。
- 删除重复数据；父路由就包括了公共的前缀和选项。
- 组织；你可以一目了然的看见所有以公共路径开头的路由定义。

主要缺点就在于配置的复杂性。

## 以我们的模块为实例的例子

现在我们知晓了如何去配置路由，我们首先来创建一个仅用于显示单条博文的路由。
其 ID 为可变参数，我们需要构建一个 segment 路由。
此外，我们知道该路由需要使用相同的路径前缀 `/blog`，
所以我们将其视为现路由的子路由。更新配置如下：

```php
// In module/Blog/config/module.config.php:
namespace Blog;

use Laminas\Router\Http\Literal;
use Laminas\Router\Http\Segment;
use Laminas\ServiceManager\Factory\InvokableFactory;

return [
    'service_manager' => [ /* ... */ ],
    'controllers'     => [ /* ... */ ],
    'router'          => [
        'routes' => [
            'blog' => [
                'type' => Literal::class,
                'options' => [
                    'route'    => '/blog',
                    'defaults' => [
                        'controller' => Controller\ListController::class,
                        'action'     => 'index',
                    ],
                ],
                'may_terminate' => true,
                'child_routes'  => [
                    'detail' => [
                        'type' => Segment::class,
                        'options' => [
                            'route'    => '/:id',
                            'defaults' => [
                                'action' => 'detail',
                            ],
                            'constraints' => [
                                'id' => '[1-9]\d*',
                            ],
                        ],
                    ],
                ],
            ],
        ],
    ],
    'view_manager'    => [ /* ... */ ],
];
```

这样我们就创建了一个用于显示单条博文的路由。并定义了一个参数 `id`,
并限制其为一个或者读个整数，且不能为0.

这些路由使用与父路由相同的 `controller`，但是使用了 `detailAction()` 方法。
进去浏览器输入链接：`http://localhost:8080/blog/2`，你将会看见如下错误：

```text
A 404 error occurred

Page not found.

The requested controller was unable to dispatch the request.

Controller:
Blog\Controller\ListController

No Exception available
```

那是因为控制器中的 `detailAction()` 方法不存在。
我们需要在 `ListController` 中按照如下方式添加这个方法，其会返回一个空的视图

```php
// In module/Blog/src/Controller/ListController.php:

/* .. */

class ListController extends AbstractActionController
{
    /* ... */

    public function detailAction()
    {
        return new ViewModel();
    }
}
```

刷新浏览器，你将会看见一个熟悉的消息界面，即无法呈现视图模板。

我们现在来创建这个模板并注入 `Post` 实例用于显示博客内容，
创建一个内容如下的文件 `module/Blog/view/blog/list/detail.phtml`:

```php
<h1>Post Details</h1>

<dl>
    <dt>Post Title</dt>
    <dd><?= $this->escapeHtml($this->post->getTitle()) ?></dd>

    <dt>Post Text</dt>
    <dd><?= $this->escapeHtml($this->post->getText()) ?></dd>
</dl>
```

上面的模板接收 `$post` 变量在视图中作为 `Post` 的实例。
并更新 `ListController` 内容如下：

```php
public function detailAction()
{
    $id = $this->params()->fromRoute('id');

    return new ViewModel([
        'post' => $this->postRepository->findPost($id),
    ]);
}
```

现在刷新浏览器，理论上将会看见我们的 `Post` 内容了。
但是，这样有个问题，当我们的库去查找一个不存在的帖子的时候会抛出一个 `InvalidArgumentException` 异常，我们并没有在控制器中处理这种情况。

打开浏览器，输入 `http://localhost:8080/blog/99`; 你将会看见如下信息：

```text
An error occurred

An error occurred during execution; please try again later.

Additional information:
InvalidArgumentException

File:
{projectPath}/module/Blog/src/Model/LaminasDbSqlRepository.php:{lineNumber}

Message:
Blog post with identifier "99" not found.
```

这种方式是不够友好的，因此我们的 `ListController` 需要对  `PostService`
抛出的 `InvalidArgumentException` 异常进行预处理。
将页面重定位到博客的列表页。

首先，在 `ListController` 中引入类：

```php
use InvalidArgumentException;
```

现在在 `detailAction()` 中添加 try-catch 代码块：

```php
public function detailAction()
{
    $id = $this->params()->fromRoute('id');

    try {
        $post = $this->postRepository->findPost($id);
    } catch (\InvalidArgumentException $ex) {
        return $this->redirect()->toRoute('blog');
    }

    return new ViewModel([
        'post' => $post,
    ]);
}
```

现在，当我们访问一个不存在的帖子的时候，我们将会重定位到路由 `blog`，展示我们的博客列表。
