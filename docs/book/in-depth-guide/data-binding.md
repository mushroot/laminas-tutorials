# 编辑以及删除数据

在之前的章节中，我们学习了如何使用 laminas-form 和 laminas-db 组件来创建一个新的数据集。
本章节我们将通过学记编辑和删除数据来系统性的学习增删改查功能。

## 将对象绑定到表单上

添加文章和编辑文章表单唯一不同的点就在于是够拥有数据。
这就意味着我们需要找到一种方式将库中的数据添加到表单中。
幸运的是，laminas-form 提供给了一个**数据绑定** 的特性。

为了使用这个特性，你需要获取一个 `Post` 实例，并将其绑定到表单中。
为了达到这个目的我们需要进行如下几步操作。

- 在 `WriteController` 中添加一个 `PostRepositoryInterface` 的依赖，
  以便我们获取 `Post`。
- 在 `WriteController` 中添加一个新方法 `editAction()`，
  他将获取一个 `Post` 实例并将其绑定到表单中，并展示或者处理数据。
- 更新 `WriteControllerFactory` 以注入 `PostRepositoryInterface`。

现在我们开始更新 `WriteController`:

- 引入 `PostRepositoryInterface`。
- 添加一个变量用来承接 `PostRepositoryInterface`。
- 更新 constructor 添加 `PostRepositoryInterface`。
- 添加 `editAction()` 方法。

最终效果如下：

```php
<?php
// In module/Blog/src/Controller/WriteController.php:

namespace Blog\Controller;

use Blog\Form\PostForm;
use Blog\Model\Post;
use Blog\Model\PostCommandInterface;
use Blog\Model\PostRepositoryInterface;
use InvalidArgumentException;
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
     * @var PostRepositoryInterface
     */
    private $repository;

    /**
     * @param PostCommandInterface $command
     * @param PostForm $form
     * @param PostRepositoryInterface $repository
     */
    public function __construct(
        PostCommandInterface $command,
        PostForm $form,
        PostRepositoryInterface $repository
    ) {
        $this->command = $command;
        $this->form = $form;
        $this->repository = $repository;
    }

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

        $post = $this->form->getData();

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

    public function editAction()
    {
        $id = $this->params()->fromRoute('id');
        if (! $id) {
            return $this->redirect()->toRoute('blog');
        }

        try {
            $post = $this->repository->findPost($id);
        } catch (InvalidArgumentException $ex) {
            return $this->redirect()->toRoute('blog');
        }

        $this->form->bind($post);
        $viewModel = new ViewModel(['form' => $this->form]);

        $request = $this->getRequest();
        if (! $request->isPost()) {
            return $viewModel;
        }

        $this->form->setData($request->getPost());

        if (! $this->form->isValid()) {
            return $viewModel;
        }

        $post = $this->command->updatePost($post);
        return $this->redirect()->toRoute(
            'blog/detail',
            ['id' => $post->getId()]
        );
    }
}
```

`addAction()` 和 `editAction()` 最大的不同就在于后者首先需要查询一个 `Post`，
并将其与表单绑定。通过绑定操作以确保数据通过表单展示出来，
并且在数据验证通过后可以直接更新实例。
这就意味着我们在验证表单后可以不用调用 `getData()`。

现在我们开始更新 `WriteControllerFactory`。
首先，引入一个新的类：

```php
// In module/Blog/src/Factory/WriteControllerFactory.php:
use Blog\Model\PostRepositoryInterface;
```

接下来按照如下内容更新内容：

```php
// In module/Blog/src/Factory/WriteControllerFactory.php:
public function __invoke(ContainerInterface $container, $requestedName, array $options = null)
{
    $formManager = $container->get('FormElementManager');

    return new WriteController(
        $container->get(PostCommandInterface::class),
        $formManager->get(PostForm::class),
        $container->get(PostRepositoryInterface::class)
    );
}
```

控制器和模型都进行了更新，是时候来更新路由了。

## 添加编辑时使用的路由

编辑时的路由和我们之前定义的 `blog/detail` 路由类似，但是有两个不同的地方：

- 需要拥有一个路径前缀 `/edit`。
- 需要导航到 `WriteController`

更新 'blog' 的 `child_routes`，以添加新的路由：

```php
// In module/Blog/config/module.config.php:

use Laminas\Router\Http\Segment;

return [
    'service_manager' => [ /* ... */ ],
    'controllers'     => [ /* ... */ ],
    'router'          => [
        'routes' => [
            'blog' => [
                /* ... */

                'child_routes' => [
                    /* ... */

                    'edit' => [
                        'type' => Segment::class,
                        'options' => [
                            'route'    => '/edit/:id',
                            'defaults' => [
                                'controller' => Controller\WriteController::class,
                                'action'     => 'edit',
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

## 创建编辑模板

`add` 和 `edit` 需要渲染的表单类似；唯一不同点就在于表单的 action 属性。
因此我们可以为表单创建一个新的 *partial* 脚本，更新 `add` 模板以使用他，
并创建一个新的 `edit` 模板。

创建一个新的文件 `module/Blog/view/blog/write/form.phtml`, 内容如下：

```php
<?php
$form = $this->form;
$fieldset = $form->get('post');

$title = $fieldset->get('title');
$title->setAttribute('class', 'form-control');
$title->setAttribute('placeholder', 'Post title');

$text = $fieldset->get('text');
$text->setAttribute('class', 'form-control');
$text->setAttribute('placeholder', 'Post content');

$submit = $form->get('submit');
$submit->setValue($this->submitLabel);
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

现在， 更新 `add` 模板 `module/Blog/view/write/add.phtml`，内容如下：

```php
<h1>Add a blog post</h1>

<?php
$form = $this->form;
$form->setAttribute('action', $this->url());
echo $this->partial('blog/write/form', [
    'form' => $form,
    'submitLabel' => 'Insert new post',
]);
```

以上获取了表单并设置了其 action 属性，为提交按钮提供了一个满足上下午的标签，
同时渲染了我们新建的 partial 视图脚本。


接下来创建新的模板, `blog/write/edit`:

```php
<h1>Edit blog post</h1>

<?php
$form = $this->form;
$form->setAttribute('action', $this->url('blog/edit', [], true));
echo $this->partial('blog/write/form', [
    'form' => $form,
    'submitLabel' => 'Update post',
]);
```

`add` 和 `edit` 模板有三处不同的地方：

- 标题。
- 表单 action 属性的链接。
- 提交按钮的文本。

因为 URI 需要一个ID，我们需要确保传了ID。
我们可以在控制器中将其作为参数直接传递：`$this->url('blog/edit/', ['id' => $id])`。
这样的话就需要我们将ID或者 `Post` 实例传递给视图，
然而 laminas-router 给了我们另一种选择，你可以告诉他直接使用当前匹配的参数。
我们只需要将视图脚本的最后一个变量设置为
`true`: `$this->url('blog/edit', [], true)`。

如果这时候更新文章，你将会得到如下错误信息：

```text
Call to member function getId() on null
```

那事因为我们尚未在类中实现更新功能并在成功后返回一个 Post 对象。
现在我们来实现这个功能。

编辑文件 `module/Blog/src/Model/LaminasDbSqlCommand.php`，
并按照如下方式更新 `updatePost()` 方法：

```php
public function updatePost(Post $post)
{
    if (! $post->getId()) {
        throw new RuntimeException('Cannot update post; missing identifier');
    }

    $update = new Update('posts');
    $update->set([
            'title' => $post->getTitle(),
            'text' => $post->getText(),
    ]);
    $update->where(['id = ?' => $post->getId()]);

    $sql = new Sql($this->db);
    $statement = $sql->prepareStatementForSqlObject($update);
    $result = $statement->execute();

    if (! $result instanceof ResultInterface) {
        throw new RuntimeException(
            'Database error occurred during blog post update operation'
        );
    }

    return $post;
}
```

这样看起来和我们之前编写的 `insertPost()` 比较类似。
二者主要的不同就在于后者使用了 `Update` 类；并且使用如下方法替换 `values()` 方法：

- `set()`，提供我们需要更新的值。
- `where()`，提供条件以确定更新哪些记录（本例中为单条记录）。

另外，在执行操作之前我们需要通过检测ID是够存在以确保内容是否存在。
然后将我们对帖子的编辑情况提交给数据库，成功后返回当前帖子。

## 实现删除操作

最后但是同样重要的是，我们需要去删除一些数据。
我们首先在 `LaminasDbSqlCommand` 中实现 `deletePost()` 方法：

```php
// In module/Blog/src/Model/LaminasDbSqlCommand.php:

public function deletePost(Post $post)
{
    if (! $post->getId()) {
        throw new RuntimeException('Cannot update post; missing identifier');
    }

    $delete = new Delete('posts');
    $delete->where(['id = ?' => $post->getId()]);

    $sql = new Sql($this->db);
    $statement = $sql->prepareStatementForSqlObject($delete);
    $result = $statement->execute();

    if (! $result instanceof ResultInterface) {
        return false;
    }

    return true;
}
```

上面的代码中我们使用了 `Laminas\Db\Sql\Delete` 来创建 SQL 语句已删除我们制定的文章。

接下来，我们在文件 `module/Blog/src/Controller/DeleteController.php`
中创建一个新的控制器 `Blog\Controller\DeleteController` 内容如下：

```php
<?php
namespace Blog\Controller;

use Blog\Model\Post;
use Blog\Model\PostCommandInterface;
use Blog\Model\PostRepositoryInterface;
use InvalidArgumentException;
use Laminas\Mvc\Controller\AbstractActionController;
use Laminas\View\Model\ViewModel;

class DeleteController extends AbstractActionController
{
    /**
     * @var PostCommandInterface
     */
    private $command;

    /**
     * @var PostRepositoryInterface
     */
    private $repository;

    /**
     * @param PostCommandInterface $command
     * @param PostRepositoryInterface $repository
     */
    public function __construct(
        PostCommandInterface $command,
        PostRepositoryInterface $repository
    ) {
        $this->command = $command;
        $this->repository = $repository;
    }

    public function deleteAction()
    {
        $id = $this->params()->fromRoute('id');
        if (! $id) {
            return $this->redirect()->toRoute('blog');
        }

        try {
            $post = $this->repository->findPost($id);
        } catch (InvalidArgumentException $ex) {
            return $this->redirect()->toRoute('blog');
        }

        $request = $this->getRequest();
        if (! $request->isPost()) {
            return new ViewModel(['post' => $post]);
        }

        if ($id != $request->getPost('id')
            || 'Delete' !== $request->getPost('confirm', 'no')
        ) {
            return $this->redirect()->toRoute('blog');
        }

        $post = $this->command->deletePost($post);
        return $this->redirect()->toRoute('blog');
    }
}
```

和 `WriteController` 类似，
同样需要依赖 `PostRepositoryInterface` 和 `PostCommandInterface`。
前者确保我们可以获取执行的 post 实例，后者用于执行实际的删除操作。

当我们使用 `GET` 方式访问页面的时候，我们将会展示一个包含文章详情的确认表单。
以便在我们提交删除操作之前确认文章，当发生错误或者成功，我们都将重定向到文章列表页。

和其他控制器类似，我们需要一个工厂类。
新建文件 `module/Blog/src/Factory/DeleteControllerFactory.php`
内容如下：

```php
<?php
namespace Blog\Factory;

use Blog\Controller\DeleteController;
use Blog\Model\PostCommandInterface;
use Blog\Model\PostRepositoryInterface;
use Interop\Container\ContainerInterface;
use Laminas\ServiceManager\Factory\FactoryInterface;

class DeleteControllerFactory implements FactoryInterface
{
    /**
     * @param ContainerInterface $container
     * @param string $requestedName
     * @param null|array $options
     * @return DeleteController
     */
    public function __invoke(ContainerInterface $container, $requestedName, array $options = null)
    {
        return new DeleteController(
            $container->get(PostCommandInterface::class),
            $container->get(PostRepositoryInterface::class)
        );
    }
}
```

现在我们需要将其和应用结合，将控制器映射到工厂，并提供一个新的路由。
打开文件 `module/Blog/config/module.config.php` 按如下内容修改：

首先将控制器映射到工厂：

```php
'controllers' => [
    'factories' => [
        Controller\ListController::class => Factory\ListControllerFactory::class,
        Controller\WriteController::class => Factory\WriteControllerFactory::class,
        // Add the following line:
        Controller\DeleteController::class => Factory\DeleteControllerFactory::class,
    ],
],
```

为 "blog" 新增一个子路由：

```php
'router' => [
    'routes' => [
        'blog' => [
            /* ... */

            'child_routes' => [
                /* ... */

                'delete' => [
                    'type' => Segment::class,
                    'options' => [
                        'route' => '/delete/:id',
                        'defaults' => [
                            'controller' => Controller\DeleteController::class,
                            'action'     => 'delete',
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
```

最后，创建视图脚本，
`module/Blog/view/blog/delete/delete.phtml`
内容如下：

```php
<h1>Delete post</h1>

<p>Are you sure you want to delete the following post?</p>

<ul class="list-group">
    <li class="list-group-item"><?= $this->escapeHtml($this->post->getTitle()) ?></li>
</ul>

<form action="<?php $this->url('blog/delete', [], true) ?>" method="post">
    <input type="hidden" name="id" value="<?= $this->escapeHtmlAttr($this->post->getId()) ?>" />
    <input class="btn btn-default" type="submit" name="confirm" value="Cancel" />
    <input class="btn btn-danger" type="submit" name="confirm" value="Delete" />
</form>
```

在这里，我们并没有使用 laminas-form，那是因为这里仅仅使用了一个因此的元素
以及取货这货同意按钮，没有必要再为其提供 OOP 模型。

到此，你可以访问已经存在的文章，例如
`http://localhost:8080/blog/delete/1` 来查看这个表单。
如果你选择 `Cancel` 你将会直接返回文章列表；
如果你选择 `Delete` 系统将会删除文章然后返回文章列表，
当前文章也将被永久删除。

## 让列表变得更加实用

现在的文章列表列出了文章的所有信息；
另外，也没有添加链接，这样的话我们就需要在浏览器中输入链接以达到操作文章的目的。
我们参照如下步骤来更新列表，使其更加实用：

- 仅仅在列表中列出文章的标题；
- 为每个文章的标题添加展示文章内容的链接；
- 提供删除和编辑文章的链接；
- 添加新增文章的链接。

在真实应用场景中，我们可能会实用某些权限控制来确定是否显示删除或者编辑链接；
但是我们会在另一个教程中来展示。

打开文件 `module/Blog/view/blog/list/index.phtml` 按照如下内容更新：

```php
<h1>Blog Posts</h1>

<div class="list-group">
<?php foreach ($this->posts as $post): ?>
  <div class="list-group-item">
    <h4 class="list-group-item-heading">
      <a href="<?= $this->url('blog/detail', ['id' => $post->getId()]) ?>">
        <?= $post->getTitle() ?>
      </a>
    </h4>

    <div class="btn-group" role="group" aria-label="Post actions">
      <a class="btn btn-xs btn-default" href="<?= $this->url('blog/edit', ['id' => $post->getId()]) ?>">Edit</a>
      <a class="btn btn-xs btn-danger" href="<?= $this->url('blog/delete', ['id' => $post->getId()]) ?>">Delete</a>
    </div>
  </div>    
<?php endforeach ?>
</div>

<div class="btn-group" role="group" aria-label="Post actions">
  <a class="btn btn-primary" href="<?= $this->url('blog/add') ?>">Write new post</a>
</div>
```

至此，我们有了一个功能更强大的博客，因为我们可以使用链接和按钮在页面之间移动。

## 总结

在本章节中，我们学习了如何使用 laminas-form 组件中的数据绑定，
并使用他更新了几个功能。
我们还了解了如何将控制器与表单解耦，从而将具体的实现排除在控制器之外。

我们还展示了局部（partials）视图组件的使用，这样使得我们可以将视图的中部分组件拆分重用。特别是我们将其使用在了表单中，以防止出现重复的标记。

最后我们还学习了 `Laminas\Db\Sql` 的两个子组件 `Update` 和 `Delete`，
并学习了如何去操作他们。

在接下来的章节中，我们将对之前的操作做一个总结，
并讨论下我们在其中使用的设计模式，
并涵盖了本教材中可能会出现的几个问题。
