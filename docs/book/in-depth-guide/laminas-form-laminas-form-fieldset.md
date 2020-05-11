# 使用表单和字段

目前，我们已经实现了从数据库中获取数据。在真实的环境中，这样是远远不够的，
我们需要支持完整的增删改查操作。通常，我们通过WEB表单的方式提交新的数据。

## 表单组件

[laminas-form](https://docs.laminas.dev/laminas-form/) 以及
[laminas-inputfilter](https://docs.laminas.dev/laminas-inputfilter/)
为我们提供了功能齐全的表单以及验证的组件。
laminas-form 内建 laminas-inputfilter，因此我们可以直接使用laminas-form。

### 表单元素集

`Laminas\Form\Fieldset` 是一组可重用的元素。
你可以使用`Fieldset`来创建和服务端实体对应的各种 HTML.
为你应用中的每个实体都创建一个  `Fieldset` 是一种比较好的习惯。

`Fieldset` 组件并非是一个表单，
这就意味着我们必须将 `Fieldset` 附加到  `Laminas\Form\Form` 之后才能使用。
这样做的好处就是你可以按照你的喜好对元素进行重用。

### 表单

`Laminas\Form\Form` 是一个和 HTML `<form>` 对用的容器。
你可以在容器内添加元素（`Laminas\Form\Fieldset` 的实体）。

## 创建你的第一个元素

介绍 laminas-form 如何工作的最好方法就是来一次真实的编码。
因此我们直接来创建 `Blog` 模块所需要的所欲表单元素。
我们先来创建一个包含处理输入博客数据的 `Fieldset`：

- 你可能需要为  `id` 属性创建一个隐藏的输入，以用来在编辑或者删除的时候使用。
- 需要为 `title` 属性创建一个输入框。
- 需要为 `text` 属性创建一个文本域。

创建一个内容如下的文件 `module/Blog/src/Form/PostFieldset.php`：

```php
<?php
namespace Blog\Form;

use Laminas\Form\Fieldset;

class PostFieldset extends Fieldset
{
    public function init()
    {
        $this->add([
            'type' => 'hidden',
            'name' => 'id',
        ]);

        $this->add([
            'type' => 'text',
            'name' => 'title',
            'options' => [
                'label' => 'Post Title',
            ],
        ]);

        $this->add([
            'type' => 'textarea',
            'name' => 'text',
            'options' => [
                'label' => 'Post Text',
            ],
        ]);
    }
}
```

这个方法继承自 `Laminas\Form\Fieldset`，并实现了 `init()` 方法（之后再详细介绍），
并为我们的博客添加了需要的元素。
我们可以在你需要的地方重用当前元素集。
接下来我们创建我们的第一个表单。

## 创建 PostForm

现在我们已经创建好了 `PostFieldset`。我们可以直接在 `Form` 中使用他。
表单将会使用 `PostFieldset`，以及一个提交按钮以便我们提交数据的时候使用。

创建文件 `module/Blog/src/Form/PostForm.php` 内容如下:

```php
<?php
namespace Blog\Form;

use Laminas\Form\Form;

class PostForm extends Form
{
    public function init()
    {
        $this->add([
            'name' => 'post',
            'type' => PostFieldset::class,
        ]);

        $this->add([
            'type' => 'submit',
            'name' => 'submit',
            'attributes' => [
                'value' => 'Insert new Post',
            ],
        ]);
    }
}
```

这就是我们的表单了。没有任何特别的地方，我们向表单中添加了  `PostFieldset`
和一个提交按钮，仅此而已。

## 创建一个新 Post

现在我们已经编写完了 `PostForm`，是时候使用他了。
在这之前我们还有一些事要做：

- 我们需要创建一个新的控制器 `WriteController` 并注入一些内容：
  - 一个 `PostCommandInterface` 实例
  - 一个 `PostForm` 实例
- 需要在新的 `WriteController` 中创建一个 `addAction()` 方法用来展示并处理表单。
- 创建一个新的路由 `blog/add`，路由指向 `WriteController` 的 `addAction()` 方法。
- 同时需要创建一个视图脚本来展示表单。

### 创建 WriteController

尽管我们可以重用已经存在的控制器，但是其却有不同的功能：用来撰写一篇新的文章。
因此，他需要发出请求，并使用之前定义的 `PostCommandInterface`。

为了实现这个目的，我们需要接收并处理用户输入的数据，
并使用本章节前面部分中定义的 `PostForm` 模块。

创建一个新的文件
`module/Blog/src/Controller/WriteController.php`,
内容如下：

```php
<?php
namespace Blog\Controller;

use Blog\Form\PostForm;
use Blog\Model\Post;
use Blog\Model\PostCommandInterface;
use Laminas\Mvc\Controller\AbstractActionController;
use Laminas\View\Model\ViewModel;

class WriteController extends AbstractActionController
{
    /**
     * @var PostCommandInterface
     */
    private $command;

    /**
     * @var PostForm
     */
    private $form;

    /**
     * @param PostCommandInterface $command
     * @param PostForm $form
     */
    public function __construct(PostCommandInterface $command, PostForm $form)
    {
        $this->command = $command;
        $this->form = $form;
    }

    public function addAction()
    {
    }
}
```

并为这个控制器创建一个新的工厂类
`module/Blog/src/Factory/WriteControllerFactory.php`,
内容如下

```php
<?php
namespace Blog\Factory;

use Blog\Controller\WriteController;
use Blog\Form\PostForm;
use Blog\Model\PostCommandInterface;
use Interop\Container\ContainerInterface;
use Laminas\ServiceManager\Factory\FactoryInterface;

class WriteControllerFactory implements FactoryInterface
{
    /**
     * @param ContainerInterface $container
     * @param string $requestedName
     * @param null|array $options
     * @return WriteController
     */
    public function __invoke(ContainerInterface $container, $requestedName, array $options = null)
    {
        $formManager = $container->get('FormElementManager');
        return new WriteController(
            $container->get(PostCommandInterface::class),
            $formManager->get(PostForm::class)
        );
    }
}
```

上面的工程类中我们引入了一个新的东西：`FormElementManager`。
这是一个专门针对表单的插件。
我们不必自己注册表单，因为当我们请求一个实体的时候，他会先去检测其是否为表单。
而且他还提供了一些不错的功能：

- 如果表单或者表单元素实现了 `init()` 方法，实例化之后将会调用该方法。
  这样做是很有用的，这样就可以在注入所有依赖之后进行初始化操作。
  我们的表单和表单元素都会定义在这个方法中。
- 他确保了与输入验证相关的插件管理器之前共享实例，我们将会在之后使用他。

最终，我们需要在 `module/Blog/config/module.config.php`
中配置新的工厂方法，并将其添加到 `controllers` 配置项中：

```php
'controllers' => [
    'factories' => [
        Controller\ListController::class => Factory\ListControllerFactory::class,
        // Add the following line:
        Controller\WriteController::class => Factory\WriteControllerFactory::class,
    ],
],
```

现在我们已经完成了控制器的基本配置，我们可以将其添加到路由中了：

```php
<?php
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
                                'id' => '\d+',
                            ],
                        ],
                    ],

                    // Add the following route:
                    'add' => [
                        'type' => Literal::class,
                        'options' => [
                            'route'    => '/add',
                            'defaults' => [
                                'controller' => Controller\WriteController::class,
                                'action'     => 'add',
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

然后，我们创建了一个临时模板：

```html
<!-- Filename: module/Blog/view/blog/write/add.phtml -->
<h1>WriteController::addAction()</h1>
```

### 检查

如果此时你访问路由 `localhost:8080/blog/add` 将会看见如下错误信息：

```text
An error occurred

An error occurred during execution; please try again later.

Additional information:

Laminas\ServiceManager\Exception\ServiceNotFoundException

File:
{projectPath}/vendor/laminas/laminas-servicemanager/src/ServiceManager.php:{lineNumber}

Message:
Unable to resolve service "Blog\Model\PostCommandInterface" to a factory; are you certain you provided it during configuration?
```

如果并非出现这种情况，请确保完全按照本教程操作，并检查所有的文件。

这个错误是指我们没有定义 `PostCommandInterface` 的实现，更不用说将其引入我们的应用中！

让我们先来创建一个临时的实现，就像是我们刚开始使用库那样。
创建文件 `module/Blog/src/Model/PostCommand.php` 内容如下：

```php
<?php
namespace Blog\Model;

class PostCommand implements PostCommandInterface
{
    /**
     * {@inheritDoc}
     */
    public function insertPost(Post $post)
    {
    }

    /**
     * {@inheritDoc}
     */
    public function updatePost(Post $post)
    {
    }

    /**
     * {@inheritDoc}
     */
    public function deletePost(Post $post)
    {
    }
}
```

并在 `module/Blog/config/module.config.php` 中添加服务配置：

```php
'service_manager' => [
    'aliases' => [
        /* ... */
        // Add the following line:
        Model\PostCommandInterface::class => Model\PostCommand::class,
    ],
    'factories' => [
        /* ... */
        // Add the following line:
        Model\PostCommand::class => InvokableFactory::class,
    ],
],
```

刷新你就可以看见想要的结果了。

## 展示表单

现在我们已经拥有了一个正常工作的控制器，是时候展示表单了。
修稿控制器将表单传递给视图：

```php
// In /module/Blog/src/Controller/WriteController.php:
public function addAction()
{
    return new ViewModel([
        'form' => $this->form,
    ]);
}
```

接下来我们需要修改视图来渲染表单：

```php
<!-- Filename: module/Blog/view/blog/write/add.phtml -->
<h1>Add a blog post</h1>

<?php
$form = $this->form;
$form->setAttribute('action', $this->url());
$form->prepare();

echo $this->form()->openTag($form);
echo $this->formCollection($form);
echo $this->form()->closeTag();
```

上面进行了如下操作：

- 我们设置了表单的 `action` 属性为当前路由。
- 初始化表单；以确保数据、错误信息或其他元素都注入表单以便显示。
- 渲染当前表单的开始标签。
- 使用 `formCollection()` 视图助手渲染表单内容；
  他具有一些通用的标签。我们暂时就使用他了。
- 渲染表单的结束标签。

> ### 表单方法
>
> HTML 表单可以使用 `POST` and `GET` 方法。 laminas-form 默认使用 `POST`.
> 如果你想切换为 `GET`:
>
> ```php
> $form->setAttribute('method', 'GET');
> ```

刷新浏览器你将会看见表单以及正常展示出来了。
不过他看起来并不好看，默认也不使用 Bootstrap 标签（框架默认使用）。
让我们稍微更新下，让其变得好看点；
在当前视图脚本中为视图加上相关标记：

```php
<!-- Filename: module/Blog/view/blog/write/add.phtml -->
<h1>Add a blog post</h1>

<?php
$form = $this->form;
$form->setAttribute('action', $this->url());

$fieldset = $form->get('post');

$title = $fieldset->get('title');
$title->setAttribute('class', 'form-control');
$title->setAttribute('placeholder', 'Post title');

$text = $fieldset->get('text');
$text->setAttribute('class', 'form-control');
$text->setAttribute('placeholder', 'Post content');

$submit = $form->get('submit');
$submit->setAttribute('class', 'btn btn-primary');

$form->prepare();

echo $this->form()->openTag($form);
?>

<fieldset>
<div class="form-group">
    <?= $this->formLabel($title) ?>
    <?= $this->formElement($title) ?>
    <?= $this->formElementErrors()->render($title, ['class' => 'help-block']) ?>
</div>

<div class="form-group">
    <?= $this->formLabel($text) ?>
    <?= $this->formElement($text) ?>
    <?= $this->formElementErrors()->render($text, ['class' => 'help-block']) ?>
</div>
</fieldset>

<?php
echo $this->formSubmit($submit);
echo $this->formHidden($fieldset->get('id'));
echo $this->form()->closeTag();
```

上面的代码中我们为每一个定义的元素都添加了 HTML 属性。
并使用更加明确的视图助手以允许我们来渲染需要的表单。

然后，当我们提交表单的时候，我们看见的只是一个重新渲染的表单。
当我们为向控制器中添加业务逻辑的时候这事一个默认的处理方式。

## 为控制器添加业务逻辑

无论表单或者实体的结构如果，编写一个处理表单工作流控制器的模式都差不多：

1. 你需要检查当前请求方式是否为 `POST`，确保表单提交了。
2. 如果接收到提交的表单，你需要：
  - 将提交的数据传递给 `Form` 实例
  - 验证表单实例
3. 如果验证通过，你需要：
  - 保存数据
  - 将用户重定向到数据数据的详情页或者预览页
4. 在其他情况下，你需要展示表单，并可能需要展示错误信息。

按照如下方式修改 `WriteController:addAction()`：

```php
public function addAction()
{
    $request   = $this->getRequest();
    $viewModel = new ViewModel(['form' => $this->form]);

    if (! $request->isPost()) {
        return $viewModel;
    }

    $this->form->setData($request->getPost());

    if (! $this->form->isValid()) {
        return $viewModel;
    }

    $data = $this->form->getData()['post'];
    $post = new Post($data['title'], $data['text']);

    try {
        $post = $this->command->insertPost($post);
    } catch (\Exception $ex) {
        // An exception occurred; we may want to log this later and/or
        // report it to the user. For now, we'll just re-throw.
        throw $ex;
    }

    return $this->redirect()->toRoute(
        'blog/detail',
        ['id' => $post->getId()]
    );
}
```

逐步执行代码：

- 接收请求。
- 创建一个默认的视图模型用来容纳表单。
- 如果我们收到非 `POST` 请求，直接返回视图。
- 将接收到的 post 数据填充入表单。
- 如果表单验证失败，直接返回默认的视图，这里的表单将会包含错误信息。
- 利用验证后的数据创建 `Post` 实例。
- 尝试插入数据。
- 成功后，重定向到详情页。

> ### 子路由名称
>
> 当我们使用 laminas-mvc 以及 laminas-view 提供的视图助手变量 `url()` 的时候，
> 你需要提供一个路由名称。当使用子路由的时候，
> 路由名称为  `<parent>/<child>` &mdash; i.e.,
> 父路由名称与子路由名称使用斜杠隔开。

现在我们提交表单将会看见如下的错误信息

```text
Fatal error: Call to a member function getId() on null in
{projectPath}/module/Blog/src/Controller/WriteController.php
on line {lineNumber}
```

那是因为我们的 `PostCommand` 类没有按照约定返回一个 `Post` 实例！

让我们使用 laminas-db 来创建一个新的实例的实现。
新建文件 `module/Blog/src/Model/LaminasDbSqlCommand.php` 内容如下：

```php
<?php
namespace Blog\Model;

use RuntimeException;
use Laminas\Db\Adapter\AdapterInterface;
use Laminas\Db\Adapter\Driver\ResultInterface;
use Laminas\Db\Sql\Delete;
use Laminas\Db\Sql\Insert;
use Laminas\Db\Sql\Sql;
use Laminas\Db\Sql\Update;

class LaminasDbSqlCommand implements PostCommandInterface
{
    /**
     * @var AdapterInterface
     */
    private $db;

    /**
     * @param AdapterInterface $db
     */
    public function __construct(AdapterInterface $db)
    {
        $this->db = $db;
    }

    /**
     * {@inheritDoc}
     */
    public function insertPost(Post $post)
    {
        $insert = new Insert('posts');
        $insert->values([
            'title' => $post->getTitle(),
            'text' => $post->getText(),
        ]);

        $sql = new Sql($this->db);
        $statement = $sql->prepareStatementForSqlObject($insert);
        $result = $statement->execute();

        if (! $result instanceof ResultInterface) {
            throw new RuntimeException(
                'Database error occurred during blog post insert operation'
            );
        }

        $id = $result->getGeneratedValue();

        return new Post(
            $post->getTitle(),
            $post->getText(),
            $id
        );
    }

    /**
     * {@inheritDoc}
     */
    public function updatePost(Post $post)
    {
    }

    /**
     * {@inheritDoc}
     */
    public function deletePost(Post $post)
    {
    }
}
```

在 `insertPost()` 方法中，我们做了如下操作：

- 依照表名创建了一个 `Laminas\Db\Sql\Insert` 实例。
- 向 `Insert` 实例中添加值。
- 使用数据库适配器创建了一个 `Laminas\Db\Sql\Sql` 实例，并将 `Insert` 实例作为语句传入。
- 执行语句并检查结果。
- 将返回结果封装并返回。

现在我们需要为其创建一个工厂；
新建文件 `module/Blog/src/Factory/LaminasDbSqlCommandFactory.php` 内容如下：

```php
<?php
namespace Blog\Factory;

use Interop\Container\ContainerInterface;
use Blog\Model\LaminasDbSqlCommand;
use Laminas\Db\Adapter\AdapterInterface;
use Laminas\ServiceManager\Factory\FactoryInterface;

class LaminasDbSqlCommandFactory implements FactoryInterface
{
    public function __invoke(ContainerInterface $container, $requestedName, array $options = null)
    {
        return new LaminasDbSqlCommand($container->get(AdapterInterface::class));
    }
}
```

最后，我们还需要更新配置；更新 `module/Blog/config/module.config.php`
的 `service_manager` 部分如下：

```php
'service_manager' => [
    'aliases' => [
        Model\PostRepositoryInterface::class => Model\LaminasDbSqlRepository::class,
        // Update the following alias:
        Model\PostCommandInterface::class => Model\LaminasDbSqlCommand::class,
    ],
    'factories' => [
        Model\PostRepository::class => InvokableFactory::class,
        Model\LaminasDbSqlRepository::class => Factory\LaminasDbSqlRepositoryFactory::class,
        Model\PostCommand::class => InvokableFactory::class,
        // Add the following line:
        Model\LaminasDbSqlCommand::class => Factory\LaminasDbSqlCommandFactory::class,
    ],
],
```

再次提交表单，他将会处理表单信息并将页面重定向到新添加条目的详情页：

让我们看看是否可以进一步优化。

## 在 laminas-form 中使用 laminas-hydrator

在我们当前的控制器中，我们的代码如下：

```php
$data = $this->form->getData()['post'];
$post = new Post($data['title'], $data['text']);
```

如果这里我们可以实现自动化操作，那么就不用担心会出现下面这种情况了：

- 我们是否使用了元素集
- 以及字段集合的名称是否出错

幸运的是，laminas-form 继承了 laminas-hydrator。
他将会在我们进行数据验证之后直接返回 `Post` 实例！

让我们更新我们的元素集合以继承 hydrator 和对象原型。

首先，在类文件的顶部引入两个类的声明：

```php
// In module/Blog/src/Form/PostFieldset.php:
use Blog\Model\Post;
use Laminas\Hydrator\Reflection as ReflectionHydrator;
```

接下来，更新 `init()` 方法，添加如下类：

```php
// In /module/Blog/src/Form/PostFieldset.php:

public function init()
{
    $this->setHydrator(new ReflectionHydrator());
    $this->setObject(new Post('', ''));

    /* ... */
}
```

当你从元素集中或者数据的时候，将会直接返回 `Post` 实例。

然而，当我们需要从 *嵌套* 元素中获取数据；我们应该如何简化这种操作呢？

由于我们仅仅就一个元素集，我们会将其设置为表单的默认元素集。
这样就会在我们获取数据的时候就会返回我们制定的那个元素集；
为了要让其返回 `Post` 实例，我们需要执行如下操作。

按照如下方式修改 `PostForm` 类：

```php
// In /module/Blog/src/Form/PostForm.php:

public function init()
{
    $this->add([
        'name' => 'post',
        'type' => PostFieldset::class,
        'options' => [
            'use_as_base_fieldset' => true,
        ],
    ]);

    /* ... */
```

接下来我们更新 `WriteController`；使用如下内容更新 `addAction()` 方法：

```php
$data = $this->form->getData()['post'];
$post = new Post($data['title'], $data['text']);
```

更新为：

```php
$post = $this->form->getData();
```

现在一切都正常运作了。做这么多的目的就是为了让控制器与表单结构解耦，
从而使我们可以直接与实体进行交互！

## 总结

在本章节中，我们学习了使用 laminas-form 的一些基本功能，
包括 fieldsets 和 elements，表单的渲染，验证输入信息，将元素集与实体类进行绑定。

在接下来的章节汇总，我们将会使用增删改查的功能来进一步完善博客模块的功能。
