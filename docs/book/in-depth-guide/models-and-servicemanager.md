# 模块和服务管理器

上一个章节我们了解了如何使用 zend-mvc 来创建一个 "Hello World" 应用。
这是个好的开始，但是这个应用还不能完成任何工作。
在这个章节中我们将介绍模型（models）的改变，同时，也会介绍 zend-servicemanager。

## 什么是模型?

模型封装了应用的逻辑。我们的模型种通常包含 *实体* 或者 *值*
对象所代表的特定 *内容*，还包含用户储存与更新这些对象的 *库*。

为了尽力去完善我们的博客模块，我们需要去完善检索和保存博客帖子的功能。
帖子的内容就是我们的实体，库就是我们用来保存并检索帖子的。
模型将帮助我们从资源中获取数据；当我们编写模型的时候不必去关心资源是实际来源。
模型将会针对性的提供一些接口以便我们将来继承的时候来实现。

## 编写帖子库（PostRepository）

当我们编写一个库的时候，常见的最佳方案是先定义一个接口。
接口是一种非常好的方式来确保其他程序员也能很容易的简历他们自己的实现方式。
换句话说，他们可以编写具有相同函数名称的类，但是他们内部可以以不同的方式来实现，
却可以返回我们预想的结果。

在我们的例子中，我们想创建一个 `PostRepository`。这意味着首先我们需要去定义一个
`PostRepositoryInterface` 接口。我们这个库的作用是为我们提供博客帖子的数据。
目前，我们暂时只关注读方面的事情：我们定义了一个方法来返回我们所有的帖子，
另一个方法返回一个单独的帖子。

接下来让我们在 `module/Blog/src/Model/PostRepositoryInterface.php` 中创建接口

```php
namespace Blog\Model;

interface PostRepositoryInterface
{
    /**
     * Return a set of all blog posts that we can iterate over.
     *
     * Each entry should be a Post instance.
     *
     * @return Post[]
     */
    public function findAllPosts();

    /**
     * Return a single blog post.
     *
     * @param  int $id Identifier of the post to return.
     * @return Post
     */
    public function findPost($id);
}
```

第一个方法 `findAllPosts()`，将会返回所有的帖子信息，第二个方法 `findPost($id)`
将会返回与传入的 `$id` 所匹配的帖子。这里我们实际上定义了一个不存在的返回值，
我们会在接下来的类中去定义它；目前，我们将创建  `PostRepository` 类。

在 `module/Blog/src/Model/PostRepository.php` 中创建 `PostRepository` 类：
并确保其继承了 `PostRepositoryInterface` 类及其定义的方法（我们将在后面补充）。
你创建的类应该具有如下内容：

```php
namespace Blog\Model;

class PostRepository implements PostRepositoryInterface
{
    /**
     * {@inheritDoc}
     */
    public function findAllPosts()
    {
        // TODO: Implement findAllPosts() method.
    }

    /**
     * {@inheritDoc}
     */
    public function findPost($id)
    {
        // TODO: Implement findPost() method.
    }
}
```

## 创建实体

因为我们的 `PostRepository` 将会返回 `Post` 实体，我们同样需要创建这个类。
让我们在 `module/Blog/src/Model/Post.php` 创建如下内容：

```php
namespace Blog\Model;

class Post
{
    /**
     * @var int
     */
    private $id;

    /**
     * @var string
     */
    private $text;

    /**
     * @var string
     */
    private $title;

    /**
     * @param string $title
     * @param string $text
     * @param int|null $id
     */
    public function __construct($title, $text, $id = null)
    {
        $this->title = $title;
        $this->text = $text;
        $this->id = $id;
    }

    /**
     * @return int|null
     */
    public function getId()
    {
        return $this->id;
    }

    /**
     * @return string
     */
    public function getText()
    {
        return $this->text;
    }

    /**
     * @return string
     */
    public function getTitle()
    {
        return $this->title;
    }
}
```

注意我们仅仅创建了 getter 方法；那是因为每个实例都是不能被改变的，
必要的时候可以允许我们将实例缓存到库中。

## 理解 PostRepository

现在我们拥有了自己的实体，我们可以进一步了解 `PostRepository` 类.
为了使得这个库容易理解，我们我们将直接从 `PostRepository` 类中返回写死的内容。
我们在 `PostRepository` 中创建一个名为 `$data` 的数组作为 `Post` 类型。
编辑 `PostReepository` 如下：

```php
namespace Blog\Model;

class PostRepository implements PostRepositoryInterface
{
    private $data = [
        1 => [
            'id'    => 1,
            'title' => 'Hello World #1',
            'text'  => 'This is our first blog post!',
        ],
        2 => [
            'id'    => 2,
            'title' => 'Hello World #2',
            'text'  => 'This is our second blog post!',
        ],
        3 => [
            'id'    => 3,
            'title' => 'Hello World #3',
            'text'  => 'This is our third blog post!',
        ],
        4 => [
            'id'    => 4,
            'title' => 'Hello World #4',
            'text'  => 'This is our fourth blog post!',
        ],
        5 => [
            'id'    => 5,
            'title' => 'Hello World #5',
            'text'  => 'This is our fifth blog post!',
        ],
    ];

    /**
     * {@inheritDoc}
     */
    public function findAllPosts()
    {
        // TODO: Implement findAllPosts() method.
    }

    /**
     * {@inheritDoc}
     */
    public function findPost($id)
    {
        // TODO: Implement findPost() method.
    }
}
```

现在我们已经拥有了一些数据，让我们修改 `find*()` 函数来返回适当的内容：

```php
namespace Blog\Model;

use DomainException;

class PostRepository implements PostRepositoryInterface
{
    private $data = [
        1 => [
            'id'    => 1,
            'title' => 'Hello World #1',
            'text'  => 'This is our first blog post!',
        ],
        2 => [
            'id'    => 2,
            'title' => 'Hello World #2',
            'text'  => 'This is our second blog post!',
        ],
        3 => [
            'id'    => 3,
            'title' => 'Hello World #3',
            'text'  => 'This is our third blog post!',
        ],
        4 => [
            'id'    => 4,
            'title' => 'Hello World #4',
            'text'  => 'This is our fourth blog post!',
        ],
        5 => [
            'id'    => 5,
            'title' => 'Hello World #5',
            'text'  => 'This is our fifth blog post!',
        ],
    ];

    /**
     * {@inheritDoc}
     */
    public function findAllPosts()
    {
        return array_map(function ($post) {
            return new Post(
                $post['title'],
                $post['text'],
                $post['id']
            );
        }, $this->data);
    }

    /**
     * {@inheritDoc}
     */
    public function findPost($id)
    {
        if (! isset($this->data[$id])) {
            throw new DomainException(sprintf('Post by id "%s" not found', $id));
        }

        return new Post(
            $this->data[$id]['title'],
            $this->data[$id]['text'],
            $this->data[$id]['id']
        );
    }
}
```

现在两个方法都可以返回合适的内容了。注意，从技术的角度来说这点实现是远远不够的。
我们将会在后面一步步的优化，但是现在我们已经可以通过 `PostRepositoryInterface`
中定义的方法来获取我们想要的数据了。

## 在控制器（Controller）中使用服务（Service）

现在我们已经拥有了一个编写好的 `PostRepository`，我们希望在控制器中使用当前库。
此时，我们需要了解一个新的东西：依赖注入（DI）（Dependency Injection）。

在我们谈论依赖注入之前，我们先讨论下注入我们类所需依赖的方式。
通常我们都使用构造注入（Constructor Injection）的方式，
这中方式会一次性注入所有的依赖。

在这个例子中，我们希望 `ListController` 可以随时调用 `PostRepository`。
这意味着 `PostRepository` 类将作为 `ListController` 类的一个依赖（dependency）存在；
没有 `PostRepository` 的话我们的 `ListController` 将不具备正常的功能。
为了保证 `ListController` 能够随时调用这个依赖（dependency），我们将使用构造函数注入这个依赖（dependency）。
修改 `ListController` 的内容如下：

```php
namespace Blog\Controller;

use Blog\Model\PostRepositoryInterface;
use Laminas\Mvc\Controller\AbstractActionController;

class ListController extends AbstractActionController
{
    /**
     * @var PostRepositoryInterface
     */
    private $postRepository;

    public function __construct(PostRepositoryInterface $postRepository)
    {
        $this->postRepository = $postRepository;
    }
}
```

这个构造函数有一个必填参数；我们提供任何非继承自 `PostRepositoryInterface`
的类够无法创建这个实例。如果你进入浏览器通过 `localhost:8080/blog` 链接访问你的引用，
你将会看见如下错误信息：

```text
Catchable fatal error: Argument 1 passed to Blog\Controller\ListController::__construct()
must be an instance of Blog\Model\PostRepositoryInterface, none given,
called in {projectPath}/vendor/laminas/src/Factory/InvokableFactory.php on line {lineNumber}
and defined in {projectPath}/module/Blog/src/Controller/ListController.php on line {lineNumber}
```

这个错误是预料中的，他准确的告诉了我们 `ListController` 需要通过一个继承自
`PostRepositoryInterface` 的类来实现。那么我们如何才能保证 `ListController`
能够接收这个接口的实现呢？为了实现这个需求，我们需要告诉应用怎么去实例化 `Blog\Controller\ListController`。
如果你还记得我们创建我们的控制器的方式，我们在模块配置文件中将其映射到 `InvokableFactory` 中：


```php
// In module/Blog/config/module.config.php:
namespace Blog;

use Laminas\ServiceManager\Factory\InvokableFactory;

return [
    'controllers'  => [
        'factories' => [
            Controller\ListController::class => InvokableFactory::class,
        ],
    ],
    'router' => [ /** Router Config */ ]
    'view_manager' => [ /** ViewManager Config */ ],
);
```

`InvokableFactory` 使用不带参数的方式来实例化我们的控制器类。
但是 `ListController` 现在需要传入参数了，我们需要来修稿这个方法。
我们将为 `ListController` 创建一个自定义的工厂方法，
首先按照如下的内容更新我们的配置文件：

```php
// In module/Blog/config/module.config.php:
namespace Blog;

// Remove the InvokableFactory import statement

return [
    'controllers' => [
        'factories' => [
            // Update the following line:
            Controller\ListController::class => Factory\ListControllerFactory::class,
        ],
    ],
    'router' => [ /** Router Config */ ]
    'view_manager' => [ /** ViewManager Config */ ],
);
```

上面修改 `ListController` 映射到一个我们即将创建的新的工厂类 `Blog\Factory\ListControllerFactory`。
如果现在你刷新你的浏览器你看会看见如下错误信息：

```html
An error occurred

An error occurred during execution; please try again later.

Additional information:

Laminas\ServiceManager\Exception\ServiceNotFoundException

File:
{projectPath}/laminas/laminas-servicemanager/src/ServiceManager.php:{lineNumber}

Message:

Unable to resolve service "Blog\Controller\ListController" to a factory; are you
certain you provided it during configuration?
```

此异常消息表示服务容器无法解析这个工厂服务，并询问我们创建了配置中的服务。
我们没有，所以这个工厂服务必须不存在。接下来让我们来编写这个工厂类。

## 编写一个工厂类

为 zend-servicemanager 编写工厂类同样需要继承 `Laminas\ServiceManager\Factory\FactoryInterface`，
或者其他可被调用的类（实现了 `__invoke()` 方法的类）；
`FactoryInterface` 接口定义了一个 `__invoke()` 方法。
这个方法的第一个参数是这个应用的容器（container）类，同时也是必须的；如果你继承了 `FactoryInterface`
你同时也必须定义第二个参数 `$requestedName`，它是工厂对服务名的一个映射，
第三个参数 `$options`，他可以在控制管理器实例化的时候提供一些其他的参数，
大多数情况下最后一个参数是可以省略的；然而通过第二个参数你可以创建可复用的工厂，
因此在你编写工厂类的时候这是一个很好的途径！目前我们这是一个一次性的工厂，
所以我们只用第一个参数。接下来让我们实现我们的工厂类：

```php
// In /module/Blog/src/Factory/ListControllerFactory.php:
namespace Blog\Factory;

use Blog\Controller\ListController;
use Blog\Model\PostRepositoryInterface;
use Interop\Container\ContainerInterface;
use Laminas\ServiceManager\Factory\FactoryInterface;

class ListControllerFactory implements FactoryInterface
{
    /**
     * @param ContainerInterface $container
     * @param string $requestedName
     * @param null|array $options
     * @return ListController
     */
    public function __invoke(ContainerInterface $container, $requestedName, array $options = null)
    {
        return new ListController($container->get(PostRepositoryInterface::class));
    }
}
```

工厂类收到一个应用的容器（container）实例，这里代表的是 `Laminas\ServiceManager\ServiceManager` 实例；
他同样继承自 `Interop\Container\ContainerInterface`，这样使得其可以在其他系统中重用。
我们获取一个`PostRepositoryInterface` 的完全限定类名的服务，并将其传递给控制器的构造方法。

这里仅仅进行了一些编码，不会有什么申请的事情发生。

刷新你的浏览器，你将会看见如下错误信息：

```text
An error occurred

An error occurred during execution; please try again later.

Additional information:

Laminas\ServiceManager\Exception\ServiceNotFoundException

File:
{projectPath}/vendor/laminas/laminas-servicemanager/src/ServiceManager.php:{lineNumber}

Message:

Unable to resolve service "Blog\Model\PostRepositoryInterface" to a factory; are
you certain you provided it during configuration?
```

错误信息显示，在我们的工厂内，需要 `Blog\Model\PostRepositoryInterface` 服务，
但是 `ServiceManager` 并不知道其存在。因此他不能为请求的名称创建一个实例。

## 注册服务（Services）

注册控制器采用和注册其他服务采用相同的模式。
我们将会修稿 `module.config.php` 并添加一个名为 `service_manager` 的新键；
这个键内的配置信息和 `controllers` 键中的配置类似。我们将会添加两个条目，
一个名为 `aliases` 另一个为 `factories` 内容如下：

```php
// In module/Blog/config/module.config.php
namespace Blog;

// Re-add the following import:
use Laminas\ServiceManager\Factory\InvokableFactory;

return [
    // Add this section:
    'service_manager' => [
        'aliases' => [
            Model\PostRepositoryInterface::class => Model\PostRepository::class,
        ],
        'factories' => [
            Model\PostRepository::class => InvokableFactory::class,
        ],
    ],
    'controllers'  => [ /** Controller Config */ ],
    'router'       => [ /** Router Config */ ],
    'view_manager' => [ /** View Manager Config */ ],
];
```

别名 `PostRepositoryInterface` 对应 `PostRepository` 的实现，
接着我们创建 `PostRepository` 类对 `InvokableFactory` 的映射
（就像我们对 `ListController` 做的那样）；
 我们可以对 `PostRepository` 这样操作，是因为其本身没有依赖关系。

> #### 服务别名
>
> 在 laminas-servicemanager 中, 你可以通过通过别名来请求其映射的服务。
> 当你请求 `Model\PostRepositoryInterface::class` 时，
> 你将用其完整类名(FQCN)来获取 PostRepository 类。
> 我们通常将接口的名称设为其别名，这样做方便用户指定某个接口的实现，
> 而不用去关心某个具体的实现。
> 更多使用方法可以查看
> [the laminas-servicemanager Aliases documentation](https://docs.laminas.dev/laminas-servicemanager/configuring-the-service-manager/#aliases).

刷新你的浏览器，你应该就不会看见那么多的错误信息了，但是确切的说就是我们前面一章中实现的页面。

## 在控制器中使用库（repository）

接下来我们就可以在 `ListController` 中使用 `PostRepository` 了。
首先我们需要重写默认的 `indexAction()` 方法，并返回一个具有从 `PostRepository`
获取的结果的视图。修改 `ListController` 内容如下：

```php
// In module/Blog/src/Controller/ListController.php:
namespace Blog\Controller;

use Blog\Model\PostRepositoryInterface;
use Laminas\Mvc\Controller\AbstractActionController;
// Add the following import statement:
use Laminas\View\Model\ViewModel;

class ListController extends AbstractActionController
{
    /**
     * @var PostRepositoryInterface
     */
    private $postRepository;

    public function __construct(PostRepositoryInterface $postRepository)
    {
        $this->postRepository = $postRepository;
    }

    // Add the following method:
    public function indexAction()
    {
        return new ViewModel([
            'posts' => $this->postRepository->findAllPosts(),
        ]);
    }
}
```

首先，注意我们在控制器会引用里哪一个类 `Laminas\View\Model\ViewModel`;
这个类在 zend-mvc 应用中通常被用来作为返回值使用。
`ViewModel` 实例允许我们提供一些值用来选择视图以及指定视图。
在这个例子中我们分配了一个从 `findAllPosts()` 方法返回的 `$posts` 值
（`Post` 实例组成的数组）。刷新你的浏览器你不会看见任何的改变，
那是因为我们还没有跟新我们视图模板中的数据。

> ###  ViewModels 也不是非必须的
>
> 你实际上可以不用返回一个 `ViewModel` 的实例； 你可以直接返回一个普通的数组，
> laminas-mvc 会在内部将其转换为 `ViewModel`。下面的方式都是可行的：
>
> ```php
> // Explicit ViewModel:
> return new ViewModel(['foo' => 'bar']);
>
> // Implicit ViewModel:
> return ['foo' => 'bar'];
> ```

## 访问视图变量

让我们修改视图脚本用一个表格来显示从库中返回所有的帖子

```php
<!-- Filename: module/Blog/view/blog/list/index.phtml -->
<h1>Blog</h1>

<?php foreach ($this->posts as $post): ?>
<article>
  <h1 id="post<?= $post->getId() ?>"><?= $post->getTitle() ?></h1>

  <p><?= $post->getText() ?></p>
</article>
<?php endforeach ?>
```

在这个视图脚本中，我们队传递的帖子模型进行了迭代。
这个数组的每个实例都是一个独立的 `Blog\Model\Post` 类型的实例，
我们可以使用实例中的 getter 方法来获取结果。

#### 实例变量 和 脚本变量

> 默认情况下，所有的变量都通过视图模型的渲染器直接传递给视图脚本，
> 因此既可以使用实例变量又可以使用脚本变量的方式来引用
>（例如：`$this->posts` 同 `$posts` 一样使用）。
> 然而，我们推荐在视图脚本中使用 _实例变量_ (`$this->posts`)
> 的方式来引用原始视图中的数据。
> 一遍我们弄清楚其来源，并且只对在脚本中定义的变量使用 _脚本变量_(`$posts`)。

保存文件后刷新你的浏览器，你将会看见过一个完整的博客帖子列表！

## 总结

在这个章节中，我们学习到了：

- 一种在为应用创建模型的方法。
- 一些依赖注入的知识。
- 在 laminas-mvc 应用中怎么使用 laminas-servicemanager 实现注入。
- 怎么在视图脚本中访问控制器中传入的变量。

在下一个章节中，我们将处理一些在我们从数据库中获取数据的一些准备工作。
