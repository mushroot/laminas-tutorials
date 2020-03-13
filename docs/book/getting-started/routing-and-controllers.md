# 路由和控制器

我们将建立一个非常简单的库存系统来展示我们的专辑列表。主页将会列出我们的专辑并且允许
我们添加、编辑以及删除专辑，于是我们将需要下面的几个页面：

页面          | 描述
------------- | -----------
首页          | 本页将列出专辑并且提供编辑以及删除的链接。并且提供一个添加新专辑的的链接。
添加新专辑     | 本页将会提供一个添加新专辑的表单。
编辑专辑       | 本页将会提供一个编辑专辑的表单。
删除专辑       | 本页将会提供一个删除确认信息，并在确认后删除表单。

在建立文件之前，弄懂框架是如何组织页面是很必要的。框架的每个页面都是在 *模块* 的 *控制器* 
中的一个 *action*。 因此，您需要将操作组织到控制器中；例如，一个新的控制器可能会有`current`、
`archived` 以及 `view` 操作。

就像我们的专辑应用需要四个页面，我们将在 `Album` 模块中建立 `AlbumController` 控制器，并
且添加如下的四个操作方法：

页面          | 控制器             | 操作
------------- | ----------------- | ------
主页          | `AlbumController` | `index`
添加新专辑    | `AlbumController` | `add`
编辑专辑      | `AlbumController` | `edit`
删除专辑      | `AlbumController` | `delete`

我们需要使用模块的 `module.config.php` 文件的 router 项来将URL映射到每个方法上。
我们将为参照如下注释行的操作为专辑的四个操作添加一个路由。

```php
namespace Album;

use Laminas\Router\Http\Segment;
use Laminas\ServiceManager\Factory\InvokableFactory;

return [
    'controllers' => [
        'factories' => [
            Controller\AlbumController::class => InvokableFactory::class,
        ],
    ],

    // 下面行为新添加的，请更新您的文件
    'router' => [
        'routes' => [
            'album' => [
                'type'    => Segment::class,
                'options' => [
                    'route' => '/album[/:action[/:id]]',
                    'constraints' => [
                        'action' => '[a-zA-Z][a-zA-Z0-9_-]*',
                        'id'     => '[0-9]+',
                    ],
                    'defaults' => [
                        'controller' => Controller\AlbumController::class,
                        'action'     => 'index',
                    ],
                ],
            ],
        ],
    ],

    'view_manager' => [
        'template_path_stack' => [
            'album' => __DIR__ . '/../view',
        ],
    ],
];
```

路由‘album’的类型为‘segment’，segment类型的路由允许我们在路由中使用别名来映射。当前
路由为 **`/album[/:action[/:id]]`**，将会匹配以`/album`开头的URL，接下来是一个可选
的 action 名称，以及映射一个可选的id。其中方括号内的都是可选项。constraints部分允许
我们对可选项中的字符进行限制。因此我们限制 action 只能以字母开头，后面只能跟字母数字以
及下滑线和短线。id只能为数字。

当前路由允许我们使用下面的URL:

URL               | 页面                         | 操作
----------------- | ---------------------------- | ------
`/album`          | 主页 (list of albums)        | `index`
`/album/add`      | 添加新专辑                    | `add`
`/album/edit/2`   | 编辑编号为 2 的专辑           | `edit`
`/album/delete/4` | 删除编号为 4 的专辑           | `delete`

### 创建控制器

现在我们已经做好了创建控制器的准备，在 zend-mvc 中，控制器为一个名为  `{控制器名}Controller`
的类；注意，`{控制器名}Controller` 必须以大写字母打头。类位于模块中 `Controller` 目录下的
`{控制器名}Controller.php` 的文件中。当前项目中即 `module/Album/src/Controller/`目录。
每个操作为在控制器中名为 `{操作名}Action`的公开方法，注 `{操作名}`需以小写字母开头。

> #### 约定
>
> 按照惯例，laminas-mvc 除了必须继承自  `Laminas\Stdlib\Dispatchable` 外不设任何限制。
> 框架一共 `Laminas\Mvc\Controller\AbstractActionController` 和 
> `Laminas\Mvc\Controller\AbstractRestfulController` 两个抽象类。我们通常使用
> `AbstractActionController`，但如果您打算写 RESTful 服务，
> `AbstractRestfulController` 可能更适合您。

让我们开始在
`laminas-mvc-tutorials/module/Album/src/Controller/AlbumController.php`
文件中创建我们的控制器：

```php
namespace Album\Controller;

use Laminas\Mvc\Controller\AbstractActionController;
use Laminas\View\Model\ViewModel;

class AlbumController extends AbstractActionController
{
    public function indexAction()
    {
    }

    public function addAction()
    {
    }

    public function editAction()
    {
    }

    public function deleteAction()
    {
    }
}
```

现在我们已经建立了四个需要的操作。在建立视图之前他们任然不能正常运行。URL和操作
一一对应如下：

URL                                                  | Method called
---------------------------------------------------- | -------------
`http://laminas-mvc-tutorial.localhost/album`        | `Album\Controller\AlbumController::indexAction`
`http://laminas-mvc-tutorial.localhost/album/add`    | `Album\Controller\AlbumController::addAction`
`http://laminas-mvc-tutorial.localhost/album/edit`   | `Album\Controller\AlbumController::editAction`
`http://laminas-mvc-tutorial.localhost/album/delete` | `Album\Controller\AlbumController::deleteAction`

现在我们已经在应用中为每个页面设置好了路由和操作方法。

接下来让我们建立视图和模型层。

## 初始化视图脚本

为了将视图整合到我们的应用中，我们需要创建一个视图脚本文件。这些文件将会被 `DefaultViewStrategy` 
执行，并且传递一些变量并且通过控制器操作方法的返回值中获取变量。这些视图脚本保存在模块视图
目录的控制器同名文件夹下。下面我们来创建如下四个空文件：

- `module/Album/view/album/album/index.phtml`
- `module/Album/view/album/album/add.phtml`
- `module/Album/view/album/album/edit.phtml`
- `module/Album/view/album/album/delete.phtml`

现在我们可以对应用进行填充，并且开始添加数据库和模型。
