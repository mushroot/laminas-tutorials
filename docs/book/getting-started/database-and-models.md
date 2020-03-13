# 数据库及模型

## 数据库

现在我们已经为 `Album` 模块设置好了，控制器操作方法以及视图脚本，现在就需要为应用设置
好数据模型。模型是应用中的核心部分（也称“业务规则”），本例中用来数据连接并处理数据库中
的数据。我们将使用laminas-db模块中的 `Laminas\Db\TableGateway\TableGateway` 
来对数据表中的数据进行增删改查等操作。

我们使用PHP的 PDO 驱动来操作 Sqlite。创建一个如下的文本文件 `data/schema.sql`：

```sql
CREATE TABLE album (id INTEGER PRIMARY KEY AUTOINCREMENT, artist varchar(100) NOT NULL, title varchar(100) NOT NULL);
INSERT INTO album (artist, title) VALUES ('The Military Wives', 'In My Dreams');
INSERT INTO album (artist, title) VALUES ('Adele', '21');
INSERT INTO album (artist, title) VALUES ('Bruce Springsteen', 'Wrecking Ball (Deluxe)');
INSERT INTO album (artist, title) VALUES ('Lana Del Rey', 'Born To Die');
INSERT INTO album (artist, title) VALUES ('Gotye', 'Making Mirrors');
```

(测试数据选自Amazon UK畅销榜！)

现在我们使用如下的命令来创建数据库：

```bash
$ sqlite data/laminastutorial.db < data/schema.sql
```

在一些系统中，包括Ubuntu，需要使用命令 `sqlite3`；使用适合您自己的系统命令即可。

> ### 使用PHP来创建数据库
>
> 如果您系统中没有安装 Sqlite，您可以用PHP使用SQL构造文件来加载数据库。创建一个如下文的
> `data/load_db.php` 文件：
>
> ```php
> <?php
> $db = new PDO('sqlite:' . realpath(__DIR__) . '/laminastutorial.db');
> $fh = fopen(__DIR__ . '/schema.sql', 'r');
> while ($line = fread($fh, 4096)) {
>     $db->exec($line);
> }
> fclose($fh);
> ```
>
> Once created, execute it:
>
> ```bash
> $ php data/load_db.php
> ```

现在我们的数据库中已经存在了一些数据，并且将为它写一个简单的模型文件。

## 模型文件

Laminas 中没有提供 laminas-model 组件，因为模型就是您的业务逻辑取决于您的需求。
这里任然有很多组件供您使用，一种方法是，在应用中创建与实体一一对应的类，然后使用对象
映射来进行记载并且将数据保存到数据库中。另一种是使用对象 - 关系映射(ORM)技术，例如
Doctrine 或 Propel。

在当前教程中，每个专辑就是一个`Album`对象（或者说是*entity*），我们将会使用
`Laminas\Db\TableGateway\TableGateway` 来创建一个 `AlbumTable` 类来构建一个模型。
这是一个使用 [Table Data Gateway](http://martinfowler.com/eaaCatalog/tableDataGateway.html)
设计模式实现的数据库接口。值得注意的是 Table Data Gateway 模式在大型系统中会有一定的
限制。同样还有一个坑，就是您将数据库访问代码放入控制器的操作方法中，这些代码将会被
`Laminas\Db\TableGateway\AbstractTableGateway` 暴露，所以*千万不要这么做*！

让我们在 `module/Album/src/Model` 下创建一个 `Album.php` 文件：

```php
namespace Album\Model;

class Album
{
    public $id;
    public $artist;
    public $title;

    public function exchangeArray(array $data)
    {
        $this->id     = !empty($data['id']) ? $data['id'] : null;
        $this->artist = !empty($data['artist']) ? $data['artist'] : null;
        $this->title  = !empty($data['title']) ? $data['title'] : null;
    }
}
```

我们的 `Album` 实体对象是一个PHP类。为了使 laminas-db 的`TableGateway`正常运行，我们需要
实现 `exchangeArray()` 方法；当前方法用来从我们提供的数组中复制数据到实体属性中，我们
将添加一个输入验证，确保注入的值都是在允许范围内的。

接下来，我们在 `module/Album/src/Model` 目录中创建一个 `AlbumTable.php` 文件：

```php
namespace Album\Model;

use RuntimeException;
use Laminas\Db\TableGateway\TableGatewayInterface;

class AlbumTable
{
    private $tableGateway;

    public function __construct(TableGatewayInterface $tableGateway)
    {
        $this->tableGateway = $tableGateway;
    }

    public function fetchAll()
    {
        return $this->tableGateway->select();
    }

    public function getAlbum($id)
    {
        $id = (int) $id;
        $rowset = $this->tableGateway->select(['id' => $id]);
        $row = $rowset->current();
        if (! $row) {
            throw new RuntimeException(sprintf(
                'Could not find row with identifier %d',
                $id
            ));
        }

        return $row;
    }

    public function saveAlbum(Album $album)
    {
        $data = [
            'artist' => $album->artist,
            'title'  => $album->title,
        ];

        $id = (int) $album->id;

        if ($id === 0) {
            $this->tableGateway->insert($data);
            return;
        }

        try {
            $this->getAlbum($id);
        } catch (RuntimeException $e) {
            throw new RuntimeException(sprintf(
                'Cannot update album with identifier %d; does not exist',
                $id
            ));
        }

        $this->tableGateway->update($data, ['id' => $id]);
    }

    public function deleteAlbum($id)
    {
        $this->tableGateway->delete(['id' => (int) $id]);
    }
}

```

接下来还有许多问题需要我们去完善。首先，我们设置一个 protected 属性的 `$tableGateway`
用来在构造函数中接收实现自 `TableGatewayInterface` 的 `TableGateway` 的实例（这将
很容易被替代的方法接管，包括在测试中模拟实例）。我们将使用其为我们的专辑来操作数据库。

我们将在应用中创建一些辅助的方法来实现table gateway，`fetchAll()` 检索出数据库中所有
专辑的记录并返回一个 `ResultSet`，`getAlbum()` 检索出一个单独的记录并返回一个  `Album`
对象，`saveAlbum()` 在数据库中添加一个新记录或者更新一个存在的行，`deleteAlbum()` 完全
删除一条记录，每个方法的代码都是比较清晰明了的，不再做过多的说明。

## 使用 ServiceManager 配置 table gateway 并且将其注入 AlbumTable

为了总是能在应用中使用 `AlbumTable` 的同一个实例，我们将使用 `ServiceManager` 来进行
定义和实现。通常使用 `Module` 类中名为 `getServiceConfig()` 的方法来定义，他可以被
`ModuleManager` 自动调用，并且添加到 `ServiceManager` 中去。当我们需要的时候就可以
去访问。

为了配置 `ServiceManager`， 我们可以根据 `ServiceManager` 的需求提供一个类名或
者一个工厂（闭包, 回掉及工厂类的类名）来生成实例化对象，首先我们在 
`module/Album/src/Module.php` 中添加方法 `getServiceConfig()` 来提供一个创建 
`AlbumTable` 的工厂。

```php
namespace Album;

// Add these import statements:
use Laminas\Db\Adapter\AdapterInterface;
use Laminas\Db\ResultSet\ResultSet;
use Laminas\Db\TableGateway\TableGateway;
use Laminas\ModuleManager\Feature\ConfigProviderInterface;

class Module implements ConfigProviderInterface
{
    // getConfig() method is here

    // Add this method:
    public function getServiceConfig()
    {
        return [
            'factories' => [
                Model\AlbumTable::class => function($container) {
                    $tableGateway = $container->get(Model\AlbumTableGateway::class);
                    return new Model\AlbumTable($tableGateway);
                },
                Model\AlbumTableGateway::class => function ($container) {
                    $dbAdapter = $container->get(AdapterInterface::class);
                    $resultSetPrototype = new ResultSet();
                    $resultSetPrototype->setArrayObjectPrototype(new Model\Album());
                    return new TableGateway('album', $dbAdapter, null, $resultSetPrototype);
                },
            ],
        ];
    }
}
```

当前方法返回一个包含 `factories` 的数组，并在传递给 `ServiceManager` 之前，通过
`ModuleManager` 对其进行了合并操作。`Album\Model\AlbumTable` 的工厂方法使用
`ServiceManager` 来创建了一个通过构造函数传入了`TableGateway`的 
`Album\Model\AlbumTableGateway` 服务。我们同样需要告知 `ServiceManager` 
`AlbumTableGateway` 服务获取了一个 `Zend\Db\Adapter\AdapterInterface` 的实现
（同样获取自 `ServiceManager`），`TableGateway` 同样也需要一个 `Album` 对象用来
创建一个新的结果行。`TableGateway` 使用原型模式来创造结果集和实体。这将意味着对象
并不是在需要的时候被实例化，系统将会克隆一个预先实例化的对象。详情参见
[PHP Constructor Best Practices and the Prototype Pattern](http://ralphschindler.com/2012/03/09/php-constructor-best-practices-and-the-prototype-pattern)
获取更多信息。

> ### 工厂
>
> 上面的例子中是在在module类中建立了一个工厂的闭包。另一种选择是，先建立一个工厂*类*，
> 然后将其在模块配置文件中做映射。这种方式有很多有点：
>
> - 除非工厂调用，代码不会被执行和解析。
> - 您可以很容易的对工厂类进行单元测试以确保其正常工作。
> - 如果需要的话，您可以很容易的对工厂进行扩展。
> - 您可以横跨具有类似构造的实例对工厂进行复用。
> 
> 创建工厂详见 [laminas-servicemanager 文档](https://laminas.org.cn/laminas-servicemanager/configuring-the-service-manager/#factories).

`Laminas\Db\Adapter\AdapterInterface` 服务属于 laminas-db 组件。您可能已经注意到前面的
 `config/modules.config.php` 做了如下更改：

```php
return [
    'Laminas\Form',
    'Laminas\Db',
    'Laminas\Router',
    'Laminas\Validator',
    /* ... */
],
```

All Laminas components that provide laminas-servicemanager configuration are
also exposed as modules themselves; the prompts as to where to register the
components during our initial installation occurred to ensure that the above
entries are created for you.

最终我们我们可以使用具有 `Laminas\Db\Adapter\AdapterInterface` 服务的工厂；现在我们需要
一个配置信息以确保能够创建一个适配器。

Laminas 的 `ModuleManager` 组件将会合并每个模块中的 `module.config.php` 文件，
接下来合并 `config/autoload/` 文件夹下的文件（首先合并 `*.global.php` ，其次合并
`*.local.php` 文件）。我们将会添加数据库配置信息到 `global.php` 文件中，同时可以提交到
版本控制系统中。你可以使用 `local.php` （除开VCS）来储存你的数据库访问证书。修改
`config/autoload/global.php` 文件（在项目根目录，不在 `Album` 目录中）添加如下代码：

```php
return [
    'db' => [
        'driver' => 'Pdo',
        'dsn'    => sprintf('sqlite:%s/data/laminastutorial.db', realpath(getcwd())),
    ],
];
```

如果您的数据库需要验证，你可以将公用信息放入`config/autoload/global.php`文件中，接下来
将当前环境配置信息，包括DSN和证书，放入 `config/autoload/local.php` 文件中。这些配置
信息都将在应用运行的时候合并，确保配置信息的完整性，但是也允许您将证书文件排除在版本控制
系统之外。

## 回到控制器


现在我们已经拥有了一个模型，我们需要将其注入到我们的控制器中以便使用。

首先，我们在控制器中添加一个构造函数，打开文件 
`module/Album/src/Controller/AlbumController.php` 然后添加如下的构造函数：

```php
namespace Album\Controller;

// Add the following import:
use Album\Model\AlbumTable;
use Laminas\Mvc\Controller\AbstractActionController;
use Laminas\View\Model\ViewModel;

class AlbumController extends AbstractActionController
{
    // Add this property:
    private $table;

    // Add this constructor:
    public function __construct(AlbumTable $table)
    {
        $this->table = $table;
    }

    /* ... */
}
```

我们的控制器现在依赖 `AlbumTable`，所以我们将为控制器创建一个工厂。类似我们为模型创建
工厂，我们将在  `Module` 类中添加一个新方法 `Album\Module::getControllerConfig()`:

```php
namespace Album;

use Laminas\Db\Adapter\AdapterInterface;
use Laminas\Db\ResultSet\ResultSet;
use Laminas\Db\TableGateway\TableGateway;
use Laminas\ModuleManager\Feature\ConfigProviderInterface;

class Module implements ConfigProviderInterface
{
    // getConfig() and getServiceConfig() methods are here

    // Add this method:
    public function getControllerConfig()
    {
        return [
            'factories' => [
                Controller\AlbumController::class => function($container) {
                    return new Controller\AlbumController(
                        $container->get(Model\AlbumTable::class)
                    );
                },
            ],
        ];
    }
}
```

因为我们已经定义了我们的工厂，我们可以修改 `module.config.php` 文件移除以前的
配置信息，打开 `module/Album/config/module.config.php` 删除如下所示行：

```php
<?php
namespace Album;

// Remove this:
use Laminas\ServiceManager\Factory\InvokableFactory;

return [
    // And remove the entire "controllers" section here:
    'controllers' => [
        'factories' => [
            Controller\AlbumController::class => InvokableFactory::class,
        ],
    ],

    /* ... */
];
```


我们可以在控制器中顶定义一个 property `$table` 变量以确保我们能在任何时候都能调用
我们的模型。

## 列出专辑信息

为了列出专辑信息，我们需要从模型中取出这些信息并且传送给视图。因此我们需要完善 `AlbumController`
中的 `indexAction()` 方法，按照如下代码更新 `AlbumController::indexAction()`：

```php
// module/Album/src/Controller/AlbumController.php:
// ...
    public function indexAction()
    {
        return new ViewModel([
            'albums' => $this->table->fetchAll(),
        ]);
    }
// ...
```

在Laminas中，为了在视图中设置变量。我们将返回一个`ViewModel`实例，其构造的第
一个参数就是我们希望返回数据的数组。这些数据都将自动的被传递给视图脚本。`ViewModel`
对象也将允许我们制定视图脚本的位置，但是默认是使用 `{模块名}/{控制器名}/{操作名}`。
接下来我们将在  `index.phtml` 视图脚本中填写如下内容：

```php
<?php
// module/Album/view/album/album/index.phtml:

$title = 'My albums';
$this->headTitle($title);
?>
<h1><?= $this->escapeHtml($title) ?></h1>
<p>
    <a href="<?= $this->url('album', ['action' => 'add']) ?>">Add new album</a>
</p>

<table class="table">
<tr>
    <th>Title</th>
    <th>Artist</th>
    <th>&nbsp;</th>
</tr>
<?php foreach ($albums as $album) : ?>
    <tr>
        <td><?= $this->escapeHtml($album->title) ?></td>
        <td><?= $this->escapeHtml($album->artist) ?></td>
        <td>
            <a href="<?= $this->url('album', ['action' => 'edit', 'id' => $album->id]) ?>">Edit</a>
            <a href="<?= $this->url('album', ['action' => 'delete', 'id' => $album->id]) ?>">Delete</a>
        </td>
    </tr>
<?php endforeach; ?>
</table>
```

首先我们来设置页面的标题（使用layout），并且使用 `headTitle()` 方法来设置 `<head>` 项中
的title，使其能够在浏览器的标题栏中显示。接下来创建一个添加新专辑的链接。

`url()` 视图助手是由 laminas-mvc 以 laminas-view 提供，用来创建我们需要的链接。`url()` 的第一个
参数是我们的路由名称，第二个参数是用来替换路由中占位符的数组。当前例子中，我们使用 `album`
路由并设置 `action` 及 `id` 这两个占位符的变量。

我们遍历从控制器方法中分发的 `$albums` laminas-view 将自动确保这边变量分配到视图脚本中。
我们可以使用`$this->{变量名}` 的方式来访问，以区分在视图中创建的变量。

我们接下来创建一个表格用来展示专辑的标题和内容，并且提供一个允许我们删除编辑专辑的链接，
我们使用一个标准的的 `foreach:` 及 `endforeach;` 来替代大括号形式的遍历，这样在大块的
区间中更容易识别。同样，使用`url()` 实体助手来创建编辑以及删除的链接。

> ### Escaping
>
> 我们使用 `escapeHtml()` 视图脚本有助于保护我们自己免受
> [跨站脚本 (XSS) 攻击](http://en.wikipedia.org/wiki/Cross-site_scripting).

如果打开 `http://localhost:8080/album` (或者使用Apache的时候打开 `http://zf2-tutorial.localhost/album` 
) 您将会看见如下页面:

![Initial album listing](../images/user-guide.database-and-models.album-list.png)
