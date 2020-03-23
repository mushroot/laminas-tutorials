# SQL 抽象和对象 Hydration

在上一个章节中，我们介绍了数据库的抽象并且为我们存储帖子的操作添加了一个新的接口。
我们现在开始创建支持数据库操作的 `PostRepositoryInterface` 和 `PostCommandInterface`
示例中我们将使用 `Laminas\Db\Sql` 类。

## 准备数据库

这个教程默认你已经看过 [快速开始](../getting-started/overview.md) 部分的教程了，
并且你已经向 `data/zftutorial.db` 这个 SQLite 数据库中填充了数据，
我们将会继续使用它，并添加一些其他的表。

创建 `data/posts.schema.sql` 文件内容如下：

```sql
CREATE TABLE posts (id INTEGER PRIMARY KEY AUTOINCREMENT, title varchar(100) NOT NULL, text TEXT NOT NULL);

INSERT INTO posts (title, text) VALUES ('Blog #1', 'Welcome to my first blog post');
INSERT INTO posts (title, text) VALUES ('Blog #2', 'Welcome to my second blog post');
INSERT INTO posts (title, text) VALUES ('Blog #3', 'Welcome to my third blog post');
INSERT INTO posts (title, text) VALUES ('Blog #4', 'Welcome to my fourth blog post');
INSERT INTO posts (title, text) VALUES ('Blog #5', 'Welcome to my fifth blog post');
```

我们再次执行 `sqlite` 命令（或 `sqlite3`，以你系统版本为准）
向  `data/zftutorial.db` 这个 SQLite 数据库中导入数据：

```bash
$ sqlite data/zftutorial.db < data/posts.schema.sql
```

如果你没有 `sqlite` 命令，你同样可以使用PHP。创建如下内容的脚本文件 `data/load_posts.php`：


```php
<?php
$db = new PDO('sqlite:' . __DIR__ . '/laminastutorial.db');
$fh = fopen(__DIR__ . '/posts.schema.sql', 'r');
while ($line = fread($fh, 4096)) {
    $line = trim($line);
    $db->exec($line);
}
fclose($fh);
```

并执行:

```bash
$ php data/load_posts.php
```

## 快速预览 Laminas\\Db\\Sql

使用 `Laminas\Db\Sql` 来创建数据库查询，你需要一个可用的数据库支配器。
我们可以重用在快速入门教程中 [covered this in the database chapter](../getting-started/database-and-models.md#using-servicemanager-to-configure-the-table-gateway-and-inject-into-the-albumtable),
配置的适配器。

有了适配器并填充了新的表后，我们就可以运行数据库查询了。
构建查询最好是通过 `Laminas\Db\Sql` 的 "QueryBuilder" 来进行，
比如 `Laminas\Db\Sql\Sql` 用来查询， `Laminas\Db\Sql\Insert` 用来插入，
`Laminas\Db\Sql\Update` 用来更新，`Laminas\Db\Sql\Delete` 用来删除。
这些组件的基本工作就是：

1. 使用相关的类 `Sql`, `Insert`, `Update` 或者 `Delete` 来构建查询。
2. 通过 `Sql` 来构建 SQL 语句。
3. 执行这个查询。
4. 对结果进行处理

让我们现在开始为接口编写数据库驱动的实现。

## 编写库（repository）的实现

在 `Blog\Model` 命名空间下创建一个继承自 `PostRepositoryInterface` 的名为
`ZendDbSqlRepository` 的类；目前保持所有方法都为空：

```php
// In module/Blog/src/Model/LaminasDbSqlRepository.php:
namespace Blog\Model;

use InvalidArgumentException;
use RuntimeException;

class LaminasDbSqlRepository implements PostRepositoryInterface
{
    /**
     * {@inheritDoc}
     */
    public function findAllPosts()
    {
    }

    /**
     * {@inheritDoc}
     * @throws InvalidArgumentException
     * @throws RuntimeException
     */
    public function findPost($id)
    {
    }
}
```

现在我们回到之前学的知识：为了使用 `Laminas\Db\Sql` 函数，我们需要实现 `AdapterInterface`。
这是一个*必要条件*，在这之前我们将使用*构造注入*。创建接收 `AdapterInterface`
作为参数的 `__construct()` 方法，并且将其存为一个实例属性：

```php
// In module/Blog/src/Model/LaminasDbSqlRepository.php:
namespace Blog\Model;

use InvalidArgumentException;
use RuntimeException;
use Laminas\Db\Adapter\AdapterInterface;

class LaminasDbSqlRepository implements PostRepositoryInterface
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
    public function findAllPosts()
    {
    }

    /**
     * {@inheritDoc}
     * @throws InvalidArgumentException
     * @throws RuntimeException
     */
    public function findPost($id)
    {
    }
}
```

我们现在有了一个必须的参数，我们需要编写一个我们自己的工厂方法。
为我们新库的实现编写一个工厂方法：

```php
// In module/Blog/src/Factory/LaminasDbSqlRepositoryFactory.php
namespace Blog\Factory;

use Interop\Container\ContainerInterface;
use Blog\Model\LaminasDbSqlRepository;
use Laminas\Db\Adapter\AdapterInterface;
use Laminas\ServiceManager\Factory\FactoryInterface;

class LaminasDbSqlRepositoryFactory implements FactoryInterface
{
    /**
     * @param ContainerInterface $container
     * @param string $requestedName
     * @param null|array $options
     * @return LaminasDbSqlRepository
     */
    public function __invoke(ContainerInterface $container, $requestedName, array $options = null)
    {
        return new LaminasDbSqlRepository($container->get(AdapterInterface::class));
    }
}
```

我们现在可以将我们的库的实现作为一个服务来注入了。为了实现这个需求，我们需要做两个修改：

- 为我们新的库注册一个工厂实例。
- 更新存在的别名 `PostRepositoryInterface` 指向新的库。

更新 `module/Blog/config/module.config.php` 内容如下：

```php
return [
    'service_manager' => [
        'aliases' => [
            // Update this line:
            Model\PostRepositoryInterface::class => Model\LaminasDbSqlRepository::class,
        ],
        'factories' => [
            Model\PostRepository::class => InvokableFactory::class,
            // Add this line:
            Model\LaminasDbSqlRepository::class => Factory\LaminasDbSqlRepositoryFactory::class,
        ],
    ],
    'controllers'  => [ /* ... */ ],
    'router'       => [ /* ... */ ],
    'view_manager' => [ /* ... */ ],
];
```


更新适配器后我们就可以刷新我们的博客页面 `localhost:8080/blog`
你将会看见抛出一个 `ServiceNotFoundException` 异常，并伴有如下错误信息

```text
Warning: Invalid argument supplied for foreach() in {projectPath}/module/Blog/view/blog/list/index.phtml on line {lineNumber}
```

实际上是我们的映射库没有返回任何信息。让我们修改 `findAllPosts()`
函数从数据表中返回所有的博客帖子：

```php
// In /module/Blog/src/Model/LaminasDbSqlRepository.php:
namespace Blog\Model;

use InvalidArgumentException;
use RuntimeException;
use Laminas\Db\Adapter\AdapterInterface;
use Laminas\Db\Sql\Sql;

class LaminasDbSqlRepository implements PostRepositoryInterface
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
    public function findAllPosts()
    {
        $sql    = new Sql($this->db);
        $select = $sql->select('posts');
        $stmt   = $sql->prepareStatementForSqlObject($select);
        $result = $stmt->execute();
        return $result;
    }

    /**
     * {@inheritDoc}
     * @throws InvalidArgumentException
     * @throw RuntimeException
     */
    public function findPost($id)
    {
    }
}
```

不幸的是，当我们再次刷新应用的时候又会看见如下错误信息：

```text
PHP Fatal error:  Call to a member function getId() on array in {projectPath}/module/Blog/view/blog/list/index.phtml on line {lineNumber}
```

让我们在返回 `$result` 变量之前打印出来看看获取的是何种结果。
修改 `findAllPosts()` 方法来打印结果：

```php
public function findAllPosts()
{
    $sql    = new Sql($this->db);
    $select = $sql->select('posts');
    $stmt   = $sql->prepareStatementForSqlObject($select);
    $result = $stmt->execute();

    var_export($result);
    die();

    return $result;
}
```

刷新应用，你将会看见如下的简短输出：

```text
Laminas\Db\Adapter\Driver\Pdo\Result::__set_state(array(
    'statementMode'   => 'forward',
    'fetchMode'       => 2,
    'resource'        => PDOStatement::__set_state(array(
        'queryString' => 'SELECT "posts".* FROM "posts"',
    )),
    'options'         => null,
    'currentComplete' => false,
    'currentData'     => null,
    'position'        => -1,
    'generatedValue'  => '0',
    'rowCount'        => Closure::__set_state(array()),
))
```

正如你看见的那样，我们不直接返回任何的数据。而是会提供一个似乎是没有任何数据的 `Result` 对象。
但这只是表面现象。这个 `Result` 对象会在你访问的时候返回数据。
如果你确定查询结果是正确的，最好的方法就是通过 `ResultSet` 对象来使用 `Result` 对象中的数据。

首先，我们在类文件中添加两个引入：

```php
use Laminas\Db\Adapter\Driver\ResultInterface;
use Laminas\Db\ResultSet\ResultSet;
```

Now update the `findAllPosts()` method as follows:

```php
public function findAllPosts()
{
    $sql    = new Sql($this->db);
    $select = $sql->select('posts');
    $stmt   = $sql->prepareStatementForSqlObject($select);
    $result = $stmt->execute();

    if ($result instanceof ResultInterface && $result->isQueryResult()) {
        $resultSet = new ResultSet();
        $resultSet->initialize($result);
        var_export($resultSet);
        die();
    }

    die('no data');
}
```

刷新页面，你将会看见如下的一个 `ResultSet` 输出信息：

```text
Laminas\Db\ResultSet\ResultSet::__set_state(array(
    'allowedReturnTypes'   =>
        array(
            0 => 'arrayobject',
            1 => 'array',
        ),
    'arrayObjectPrototype' =>
        ArrayObject::__set_state(array(
        )),
    'returnType'           => 'arrayobject',
    'buffer'               => null,
    'count'                => null,
    'dataSource'           =>
        Laminas\Db\Adapter\Driver\Pdo\Result::__set_state(array(
            'statementMode'   => 'forward',
            'fetchMode'       => 2,
            'resource'        =>
                PDOStatement::__set_state(array(
                    'queryString' => 'SELECT "album".* FROM "album"',
                )),
            'options'         => null,
            'currentComplete' => false,
            'currentData'     => null,
            'position'        => -1,
            'generatedValue'  => '0',
            'rowCount'        =>
                Closure::__set_state(array(
                )),
        )),
    'fieldCount'           => 3,
    'position'             => 0,
))
```

我们比较关注的是 `returnType` 属性，它的值为 `arrayobject`。
这表明所有的数据库实例都将返回一个 `ArrayObject` 实例。
但是对于我们来说这里有个小问题，我们的 `PostRepositoryInterface`  需要我们返回的是一个
`Post` 实例组成的数组。幸运的是 `Laminas\Db\ResultSet` 的子组件 `HydratingResultSet`
完美的为我们解决了这个问题；这个结果集将会使用返回的数据来填充我们指定的对象。

让我们来修改代码。首先，从类文件中删除如下的引入信息：

```php
use Laminas\Db\ResultSet\ResultSet;
```

接下来，我们将在类文件中添加如下的引入信息：

```php
use Laminas\Hydrator\Reflection as ReflectionHydrator;
use Laminas\Db\ResultSet\HydratingResultSet;
```

现在，按照如下方式更新 `findAllPosts()` 方法：

```php
public function findAllPosts()
{
    $sql       = new Sql($this->db);
    $select    = $sql->select('posts');
    $statement = $sql->prepareStatementForSqlObject($select);
    $result    = $statement->execute();

    if (! $result instanceof ResultInterface || ! $result->isQueryResult()) {
        return [];
    }

    $resultSet = new HydratingResultSet(
        new ReflectionHydrator(),
        new Post('', '')
    );
    $resultSet->initialize($result);
    return $resultSet;
}
```

这类我们做了一些修改，首先，使用 `HydratingResultSet` 代替了常规的 `ResultSet`。
这个特殊的结果集需要两个参数，第二个参数是我们需要用数据来填充的对象，
第一个参数为一个 `hydrator`（`hydrator`  是一个用来将数据填充到对象中的对象，反之亦然）。
我们这里使用 `Laminas\Hydrator\Reflection` 它能够注入一个实例的私有属性。
我们提供了一个空的 `Post` 实例，hydrator 将会对其 clone 并按照数据行建立新的实例。

替换了打印出的 `$result` 变量，我们直接返回了 初始化的 `HydratingResultSet`
便于我们可以通过其返回储存的数据。对于返回的非 `ResultInterface` 的实例，
我们将返回一个空的数组。

刷新你的页面，你将会在页面上看见所有的帖子列表。 Great!

## 对隐性依赖进行重构

这里还有一些小的地方我们做的不是那么的标准。我们在 `ZendDbSqlRepository`
中直接使用了 hydrator 和 `Post` 属性。我们将修改这个注入信息，
以便我们能在库及实现中或者在基于其的环境下。更新 `ZendDbSqlRepository` 如下：

```php
// In module/Blog/src/Model/LaminasDbSqlRepository.php:
namespace Blog\Model;

use InvalidArgumentException;
use RuntimeException;
// Replace the import of the Reflection hydrator with this:
use Laminas\Hydrator\HydratorInterface;
use Laminas\Db\Adapter\AdapterInterface;
use Laminas\Db\Adapter\Driver\ResultInterface;
use Laminas\Db\ResultSet\HydratingResultSet;
use Laminas\Db\Sql\Sql;

class LaminasDbSqlRepository implements PostRepositoryInterface
{
    /**
     * @var AdapterInterface
     */
    private $db;

    /**
     * @var HydratorInterface
     */
    private $hydrator;

    /**
     * @var Post
     */
    private $postPrototype;

    public function __construct(
        AdapterInterface $db,
        HydratorInterface $hydrator,
        Post $postPrototype
    ) {
        $this->db            = $db;
        $this->hydrator      = $hydrator;
        $this->postPrototype = $postPrototype;
    }

    /**
     * Return a set of all blog posts that we can iterate over.
     *
     * Each entry should be a Post instance.
     *
     * @return Post[]
     */
    public function findAllPosts()
    {
        $sql       = new Sql($this->db);
        $select    = $sql->select('posts');
        $statement = $sql->prepareStatementForSqlObject($select);
        $result    = $statement->execute();

        if (! $result instanceof ResultInterface || ! $result->isQueryResult()) {
            return [];
        }

        $resultSet = new HydratingResultSet($this->hydrator, $this->postPrototype);
        $resultSet->initialize($result);
        return $resultSet;
    }

    /**
     * Return a single blog post.
     *
     * @param  int $id Identifier of the post to return.
     * @return Post
     */
    public function findPost($id)
    {
    }
}
```

现在我们的库又拥有了几个参数，需要更新 `ZendDbSqlRepositoryFactory` 来注入这些参数：

```php
// In /module/Blog/src/Factory/LaminasDbSqlRepositoryFactory.php
namespace Blog\Factory;

use Interop\Container\ContainerInterface;
use Blog\Model\Post;
use Blog\Model\LaminasDbSqlRepository;
use Laminas\Db\Adapter\AdapterInterface;
use Laminas\Hydrator\Reflection as ReflectionHydrator;
use Laminas\ServiceManager\Factory\FactoryInterface;

class LaminasDbSqlRepositoryFactory implements FactoryInterface
{
    public function __invoke(ContainerInterface $container, $requestedName, array $options = null)
    {
        return new LaminasDbSqlRepository(
            $container->get(AdapterInterface::class),
            new ReflectionHydrator(),
            new Post('', '')
        );
    }
}
```

做完这些修改后，你可以再次刷新你的应用你将会再次看见你的帖子列表信息。
我们的库也不会再有隐藏的依赖了，并且能和数据库正常工作了！

## 最后整理库

在进入下一个章节前，让我们快速的完善 `findPost()` 方法库的实现：

```php
public function findPost($id)
{
    $sql       = new Sql($this->db);
    $select    = $sql->select('posts');
    $select->where(['id = ?' => $id]);

    $statement = $sql->prepareStatementForSqlObject($select);
    $result    = $statement->execute();

    if (! $result instanceof ResultInterface || ! $result->isQueryResult()) {
        throw new RuntimeException(sprintf(
            'Failed retrieving blog post with identifier "%s"; unknown database error.',
            $id
        ));
    }

    $resultSet = new HydratingResultSet($this->hydrator, $this->postPrototype);
    $resultSet->initialize($result);
    $post = $resultSet->current();

    if (! $post) {
        throw new InvalidArgumentException(sprintf(
            'Blog post with identifier "%s" not found.',
            $id
        ));
    }

    return $post;
}
```


`findPost()` 函数看起来比 `findAllPosts()` 方法简单，且又下面几个不同。

- 我们需要添加一个条件查询来根据给出的 ID 获取指定行的数据；这里我们使用 `Sql` 对象的 `where()` 方法。
- 我们使用 `isQueryResult()` 来验证 `$result`；如果验证失败，
  将抛出一个 `RuntimeException` 异常。
- 我们通过 `current()` 获取我们创建的结果，并确保能获取到数据，如若无，将抛出 `InvalidArgumentException` 异常。

## 总结

本章完结，你现在已经知道了如何使用 `Laminas\Db\Sql` 来*查询*数据，
同时你也学习到了 zend-hydrator 组件的部分知识，并与 zend-db 集成。
此外我们继续展示了我们应用各个部分的依赖注入情况。

下一个章节我们将完善路由，以便我们能够显示指定 ID 的帖子。
