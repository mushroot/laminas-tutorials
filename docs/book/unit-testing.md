# 在 Laminas MVC 应用中使用单元测试

对于一个大型的项目，在开发过程中进行扎实的单元测试以保证稳定性的非常有必要的，
特别的那些面向多用户的的系统。
每次修改之后都去手动测试是不现实的。
单元测试将会自动去测试应用程序的某些功能和组件，并在按照你编写的测试时用例运行失败的时候向你发出警告。

本教程将会写一个怎么去测试一个 laminas-mvc 应用的不同部分的单元测试。
因此，这个教程将使用[入门教程](getting-started/overview.md)。
总体来说它不是一个完整的单元测试指南，
而只是帮助我们为 laminas-mvc 应用编写单元测试做一个引导。

我们建议你需要对单元测试，断言以及 mocks 有一定的了解。

[laminas-test](https://docs.laminas.dev/laminas-test/), 为 laminas-mvc 集成了
[PHPUnit](http://phpunit.de/); 这个教程将会介绍如果用该库测试你的应用程序。

## 安装 laminas-test

[laminas-test](https://docs.laminas.dev/laminas-test/) 为 zend-mvc 集成了
PHPUnit, 包括脚手架以及自定义断言。您可以使用下面的方式安装：

```bash
$ composer require --dev laminas/laminas-test
```

上面的命令将会更新你的 `composer.json` 文件，并且执行一个更新，以及更新自动加载规则。

## 运行初始化测试

框架应用为 `Application\Controller\IndexController` 类提供了几个开箱即用的测试。
现在你已经安装了 laminas-test，你可以运行如下命令：

```bash
$ ./vendor/bin/phpunit
```

> ### PHPUnit invocation on Windows
>
> On Windows, you need to wrap the command in double quotes:
>
> ```bash
> $ "vendor/bin/phpunit"
> ```

你将看见如下信息：

```text
PHPUnit 5.4.6 by Sebastian Bergmann and contributors.

...                                                                 3 / 3 (100%)

Time: 116 ms, Memory: 11.00MB

OK (3 tests, 7 assertions)
```

如果你按照入门指南，可能会有2个失败的测试。那是因为 `AlbumController`
重写了 `Application\IndexController`。我们先忽略。

现在我们开始编写你自己的测试！

## 建立测试工作目录

和 laminas-mvc 应用一样，我们需要在模块中建立一个独立的模块，
我们不会对整个应用进行测试，而是一个模块一个模块的进行。

我们将展示在 `Album` 模块中用于测试一个模块的最基本的情况，使其可以作为测试任何
模块的基础部件。

开始在  `module/Album/` 目录下创建一个如下的 `test` 目录：

```text
module/
    Album/
        test/
            Controller/
```

接下来在 `composer.json` 中添加一个 `autoload-dev` 规则：

```json
"autoload-dev": {
    "psr-4": {
        "ApplicationTest\\": "module/Application/test/",
        "AlbumTest\\": "module/Album/test/"
    }
}
```

再运行:

```bash
$ composer dump-autoload
```

`test` 目录的结构与模块的源文件完全匹配，这将使得你测试有序并且易于找到。

## 启动您的测试

接下来编辑项目根目录的 `phpunit.xml.dist` 文件；我们将为他创建一个新的测试套件。
完成后，将会看见如下的样子：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<phpunit colors="true">
    <testsuites>
        <testsuite name="Laminas MVC Application Test Suite">
            <directory>./module/Application/test</directory>
        </testsuite>
        <testsuite name="Album">
            <directory>./module/Album/test</directory>
        </testsuite>
    </testsuites>
</phpunit>
```

现在在项目根目录运行 `phpunit -- testsuite Album`；您将会看见如下输出信息：

```bash
$ ./vendor/bin/phpunit --testsuite Album
```

> ### Windows and PHPUnit
>
> On Windows, don't forget to wrap the `phpunit` command in double quotes:
>
> ```bash
> $ "vendor/bin/phpunit" --testsuite Album
> ```

You should get similar output to the following:

```text
PHPUnit 5.4.6 by Sebastian Bergmann and contributors.

Time: 0 seconds, Memory: 1.75Mb

No tests executed!
```

开始编写我们的第一个测试！

## 第一个控制器的测试

测试控制器从来都不是一项简单的任务，但是 laminas-test 组件可以帮您省掉不少的麻烦。

首先在 `module/Album/test/Controller/` 目录下创建一个 `AlbumControllerTest.php`
文件内容如下：

```php
<?php
namespace AlbumTest\Controller;

use Album\Controller\AlbumController;
use Laminas\Stdlib\ArrayUtils;
use Laminas\Test\PHPUnit\Controller\AbstractHttpControllerTestCase;

class AlbumControllerTest extends AbstractHttpControllerTestCase
{
    protected $traceError = false;

    public function setUp()
    {
        // 模块的配置文件同样适用于测试。
        // 您可以在这里使用测试用例特殊值来重写配置信息。
        // 例如，视图模板，栈路径，模块监听选项等
        $configOverrides = [];

        $this->setApplicationConfig(ArrayUtils::merge(
            // 抓取完整的应用配置:
            include __DIR__ . '/../../../../config/application.config.php',
            $configOverrides
        ));
        parent::setUp();
    }
}
```

我们需要继承 `AbstractHttpControllerTestCase` 来对应用本身进行设置，帮助分发在
请求期间发生的其他任务，提供维护请求参数的方法，响应信息，跳转等。详见文档
 [laminas-test](https://docs.laminas.dev/laminas-test/phpunit/) 获取更多信息。

对任何 laminas-test 测试的主要条件都是在 `setApplicationConfig()` 方法中配置的。
现在我们假定默认应用中的配置都是合适的；然而我们可以使用 `$configOverrides` 变量
在测试中重写局部变量。

现在，我们为 `AlbumControllerTest` 类添加一个如下的方法：

```php
public function testIndexActionCanBeAccessed()
{
    $this->dispatch('/album');
    $this->assertResponseStatusCode(200);
    $this->assertModuleName('Album');
    $this->assertControllerName(AlbumController::class);
    $this->assertControllerClass('AlbumController');
    $this->assertMatchedRouteName('album');
}
```

整个测试代码将会调用 `/album` 链接，判断响应代码是否为 200，
并且判断控制器和方法是否匹配。

> ### 声明控制器的名称
>
> 对于判断 *控制器名称*，我们使用在 Album 模块路由配置中定义的控制器名称。
> 在我们的例子中，我们使用在 `module.config.php` 文件16行定义的 Album 模块。

如果你运行:

```bash
$ ./vendor/bin/phpunit --testsuite Album
```

这时，你讲会看见下面的输出信息：

```text
PHPUnit 5.4.6 by Sebastian Bergmann and contributors.

.                                                                   1 / 1 (100%)

Time: 124 ms, Memory: 11.50MB

OK (1 test, 5 assertions)
```

这就是第一个成功的测试!

## 一个失败的测试用例

我们可能不希望在测试过程中使用同样的数据库来作为我们站点的数据源。
接下来我们向测试用例中添加一些配置来删除数据库配置信息。
在 `AlbumControllerTest::setUp()` 方法中，
在调用 `parent::setUp();` 下添加如下代码：

```php
$services = $this->getApplicationServiceLocator();
$config = $services->get('config');
unset($config['db']);
$services->setAllowOverride(true);
$services->setService('config', $config);
$services->setAllowOverride(false);
```

以上的代码完全删除了 'db' 配置信息；
我们将在后面用别的信息来替代它。

接下来我们运行这个测试：

```bash
$ ./vendor/bin/phpunit --testsuite Album
PHPUnit 5.4.6 by Sebastian Bergmann and contributors.

F

Time: 0 seconds, Memory: 8.50Mb

这将会出现一个错误：

1) AlbumTest\Controller\AlbumControllerTest::testIndexActionCanBeAccessed
Failed asserting response code "200", actual status code is "500"

{projectPath}/vendor/laminas/laminas-test/src/PHPUnit/Controller/AbstractControllerTestCase.php:{lineNumber}
{projectPath}/module/Album/test/AlbumTest/Controller/AlbumControllerTest.php:{lineNumber}

FAILURES!
Tests: 1, Assertions: 0, Failures: 1.
```

这个错误信息除了预期状态码不是 200 而是 500 以外，并没有告诉我们更多信息。
为了能在这个测试实例出错的时候获取更多信息，我们设置 protected 成员 `$traceError`
为 `true` （默认被设置为 `false`）在 `AlbumControllerTest` 类的 `setUp`
方法上修改如下行：

```php
protected $traceError = true;
```

再次运行 `phpunit` 命令，我们将会在测试中看见更多的错误信息。
你将会得到一个抛出的异常列表，包含了信息，文件名，以及行号：

```text
1) AlbumTest\Controller\AlbumControllerTest::testIndexActionCanBeAccessed
Failed asserting response code "200", actual status code is "500"

Exceptions raised:
Exception 'Laminas\ServiceManager\Exception\ServiceNotCreatedException' with message 'Service with name "Laminas\Db\Adapter\AdapterInterface" could not be created. Reason: createDriver expects a "driver" key to be present inside the parameters' in {projectPath}/vendor/laminas/laminas-servicemanager/src/ServiceManager.php:{lineNumber}

Exception 'Laminas\Db\Adapter\Exception\InvalidArgumentException' with message 'createDriver expects a "driver" key to be present inside the parameters' in {projectPath}/vendor/laminas/laminas-db/src/Adapter/Adapter.php:{lineNumber}
```

根据这个错误消息可知，由于缺少配置文件，我们可能无法创建一个 laminas-db 适配器实例！

## 为测试配置 service manager

这个错误信息表明 service manager 不能为我们创建一个数据库适配器实例。
数据库是适配器间接的使用我们的 `Album\Model\AlbumTable` 来访问数据库中的专辑列表。

第一种方法是创建一个适配器实例，并将其传递给 service manager，并通过其使代码运行。
这样做的问题就是我们的测试用例最终会对数据库进行查询。为了保证测试的快速进行，
并且减少测试中的故障点，我们应该尽量避免这样做。

第二种方法是创建一个模拟的数据库驱动，通过模拟的数据库驱动阻止真实的数据库调用。
这是一个非常好的途径，但是创建一个模拟的驱动是很乏味的（但毫无疑问，我们又不得不这么做）。

最好的办法当然就是模拟我们的 `Album\Model\AlbumTable` 类来从数据库中检索我们的专辑列表。
记住，我们现在正在测试我们的控制器，因此我们可以模拟出实际调用的 `fetchAll`
并且使用虚拟的值来替代。在这个问题上，我们不必在意 `fetchAll()` 是如何检索数据的，
我们只需要知道他会返回一个专辑列表的数据；这样我们就能提供一个模拟的实例。
当我们测试 `AlbumTable` 的时候，就可以编写 `fetchAll` 方法的测试。

首先，让我们做一些设置。

在测试类的顶部引入 `AlbumTable` 和 `ServiceManager` 类：

```php
use Album\Model\AlbumTable;
use Laminas\ServiceManager\ServiceManager;
```

现在向测试类中添加如下属性：

```php
protected $albumTable;
```

接下来我们创建三个方法以便我们在启动中调用：

```php
protected function configureServiceManager(ServiceManager $services)
{
    $services->setAllowOverride(true);

    $services->setService('config', $this->updateConfig($services->get('config')));
    $services->setService(AlbumTable::class, $this->mockAlbumTable()->reveal());

    $services->setAllowOverride(false);
}

protected function updateConfig($config)
{
    $config['db'] = [];
    return $config;
}

protected function mockAlbumTable()
{
    $this->albumTable = $this->prophesize(AlbumTable::class);
    return $this->albumTable;
}
```

默认情况下，`ServiceManager` 不允许替换一个存在的服务。
`configureServiceManager()` 调用了实例中一个特殊的方法允许重写服务，
接下来我们就可以注入我们需要重写的部分了。最后我们禁止了重写，
这样做是为了确保在分发过程中其他服务调用重写而引发异常抛出。

上面最后一个方法使用 [Prophecy](https://github.com/phpspec/prophecy)
创建了一个 `AlbumTable` 的模拟实例绑定并集成到 PHPUnit 中。
`prophesize()` 返回的实例是一个脚手架实例；在上面的 `configureServiceManager()`
中调用其 `reveal()` 方法，将为会针对断言提供一个底层的模拟对象。

此刻，我们就按照如下可以更新我们的 `setUp()` 方法：

```php
public function setUp()
{
    // The module configuration should still be applicable for tests.
    // You can override configuration here with test case specific values,
    // such as sample view templates, path stacks, module_listener_options,
    // etc.
    $configOverrides = [];

    $this->setApplicationConfig(ArrayUtils::merge(
        include __DIR__ . '/../../../../config/application.config.php',
        $configOverrides
    ));

    parent::setUp();

    $this->configureServiceManager($this->getApplicationServiceLocator());
}
```

现在更新 `testIndexActionCanBeAccessed()` 方法添加一行 `AlbumTable` 将会调用的
`fetchAll()` 方法的断言，并返回一个数组：

```php
public function testIndexActionCanBeAccessed()
{
    $this->albumTable->fetchAll()->willReturn([]);

    $this->dispatch('/album');
    $this->assertResponseStatusCode(200);
    $this->assertModuleName('Album');
    $this->assertControllerName(AlbumController::class);
    $this->assertControllerClass('AlbumController');
    $this->assertMatchedRouteName('album');
}
```

这时运行 `phpunit`，我们将会看见如下的输出结果：

```bash
$ ./vendor/bin/phpunit --testsuite Album
PHPUnit 5.4.6 by Sebastian Bergmann and contributors.

.                                                                   1 / 1 (100%)

Time: 105 ms, Memory: 10.75MB

OK (1 test, 5 assertions)
```

## 使用 POST 方式测试操作

控制器比较常见的一个场景就是接收表单的 POST 数据并验证，就像我们在 `AlbumController::addAction()`
中做的那样。我们开始编写测试。

```php
public function testAddActionRedirectsAfterValidPost()
{
    $this->albumTable
        ->saveAlbum(Argument::type(Album::class))
        ->shouldBeCalled();

    $postData = [
        'title'  => 'Led Zeppelin III',
        'artist' => 'Led Zeppelin',
        'id'     => '',
    ];
    $this->dispatch('/album/add', 'POST', $postData);
    $this->assertResponseStatusCode(302);
    $this->assertRedirectTo('/album');
}
```

这个测试实例我们需要引入两个类；在类文件的顶部添加如下声明：

```php
use Album\Model\Album;
use Prophecy\Argument;
```

`Prophecy\Argument` 允许我们通过模拟对象的参数来执行这个断言。在这个实例中，
我们想要对 `Album` 实例进行断言。（我们可以做更深层次的断言，以确保 `Album` 实例包含了预期的数据。）

接下来在应用分发的时候，我们使用 POST 方式请求，并且向其传递数据。
这个测试实例断言了一个 302 的返回状态码，并且采用了一个新的针对回调跳转的断言。

运行 `phpunit` 将会给我们展示如下信息：

```bash
$ ./vendor/bin/phpunit --testsuite Album
PHPUnit 5.4.6 by Sebastian Bergmann and contributors.

..                                                                  2 / 2 (100%)

Time: 1.49 seconds, Memory: 13.25MB

OK (2 tests, 8 assertions)
```

测试 `editAction()` 和 `deleteAction()` 方法都是使用类似的方法；然后我们在测试
`editAction()` 方法的时候，需要对 `AlbumTable::getAlbum()` 方法进行断言：

```php
$this->albumTable->getAlbum($id)->willReturn(new Album());
```

理想的情况下， 你应该通过各种途径测试每个方法的变量。例如：

- 测试 `addAction()` 在非 POST 请求的时候是否返回一个空的表单。
- 测试当为 `addAction()` 提供一个无效的值得时候，重新展示表单，并且显示错误信息。
- 测试在 `editAction()` 或 `deleteAction()` 路由参数中提供一个无效的参数时是否跳转到适当的地址。
- 测试为 `editAction()` 提供一个无效的标识符的时候将会跳转到专辑页面。
- 测试 `editAction()` 和 `deleteAction()` 在非 POST 请求的时候显示表单。

等等。这样做将会帮助你理解进入你应用和控制器的途径，以确保不会因冒泡而造成的测试失败。

## 测试模型实体

现在我们已经知道如何测试我们测控制器，接下来我们移步去测试我们应用的其他部分：模型实体。

这里我们需要对实体的初试状态进行测试，是否是和我们期望的一样，我们可以从数据中转换模型的参数，
这样他就具备了我们需要过滤的数据。

再 `module/Album/test/Model` 目录中创建 `AlbumTest.php` 文件，并添加如下内容：

```php
<?php
namespace AlbumTest\Model;

use Album\Model\Album;
use PHPUnit\Framework\TestCase;

class AlbumTest extends TestCase
{
    public function testInitialAlbumValuesAreNull()
    {
        $album = new Album();

        $this->assertNull($album->artist, '"artist" should be null by default');
        $this->assertNull($album->id, '"id" should be null by default');
        $this->assertNull($album->title, '"title" should be null by default');
    }

    public function testExchangeArraySetsPropertiesCorrectly()
    {
        $album = new Album();
        $data  = [
            'artist' => 'some artist',
            'id'     => 123,
            'title'  => 'some title'
        ];

        $album->exchangeArray($data);

        $this->assertSame(
            $data['artist'],
            $album->artist,
            '"artist" was not set correctly'
        );

        $this->assertSame(
            $data['id'],
            $album->id,
            '"id" was not set correctly'
        );

        $this->assertSame(
            $data['title'],
            $album->title,
            '"title" was not set correctly'
        );
    }

    public function testExchangeArraySetsPropertiesToNullIfKeysAreNotPresent()
    {
        $album = new Album();

        $album->exchangeArray([
            'artist' => 'some artist',
            'id'     => 123,
            'title'  => 'some title',
        ]);
        $album->exchangeArray([]);

        $this->assertNull($album->artist, '"artist" should default to null');
        $this->assertNull($album->id, '"id" should default to null');
        $this->assertNull($album->title, '"title" should default to null');
    }

    public function testGetArrayCopyReturnsAnArrayWithPropertyValues()
    {
        $album = new Album();
        $data  = [
            'artist' => 'some artist',
            'id'     => 123,
            'title'  => 'some title'
        ];

        $album->exchangeArray($data);
        $copyArray = $album->getArrayCopy();

        $this->assertSame($data['artist'], $copyArray['artist'], '"artist" was not set correctly');
        $this->assertSame($data['id'], $copyArray['id'], '"id" was not set correctly');
        $this->assertSame($data['title'], $copyArray['title'], '"title" was not set correctly');
    }

    public function testInputFiltersAreSetCorrectly()
    {
        $album = new Album();

        $inputFilter = $album->getInputFilter();

        $this->assertSame(3, $inputFilter->count());
        $this->assertTrue($inputFilter->has('artist'));
        $this->assertTrue($inputFilter->has('id'));
        $this->assertTrue($inputFilter->has('title'));
    }
}
```

我们会测试这 5 项：

1. 是否 `Album` 所有的初始值都是 `NULL`?
2. 当我们调用 `exchangeArray()` 的时候，`Album` 的所有值都被正确的改变?
3. 当属性对应的键再 `$data` 数组中不存在的时候，属性是否会被定义为  `NULL`?
4. 是否可以获取整个模型的数组副本?
5. 所有的元素都是否被过滤了?

如果我们再次运行 `phpunit`, 我们将会获取如下信息，以确保我们的模型都被正确的配置：

```bash
$ ./vendor/bin/phpunit --testsuite Album
PHPUnit 5.4.6 by Sebastian Bergmann and contributors.

.......                                                             7 / 7 (100%)

Time: 186 ms, Memory: 13.75MB

OK (7 tests, 24 assertions)
```

## 测试表模型

我们 zend-mvc 应用的单元测试教程的最后一步就是为表模型建立单元测试。

这个测试保证我们可以获得专辑的列表，或者通过 ID 获取其中一个专辑，
并且我们可以从数据库中删除或者添加一个专辑。

为了避免不影响数据库本身的数据，我们将使用模拟对象替换一部分。

再 `module/Album/test/Model/` 目录下创建一个 `module/Album/test/Model/`
文件，添加如下数据：

```php
<?php
namespace AlbumTest\Model;

use Album\Model\AlbumTable;
use Album\Model\Album;
use PHPUnit\Framework\TestCase;
use RuntimeException;
use Laminas\Db\ResultSet\ResultSetInterface;
use Laminas\Db\TableGateway\TableGatewayInterface;

class AlbumTableTest extends TestCase
{
    protected function setUp()
    {
        $this->tableGateway = $this->prophesize(TableGatewayInterface::class);
        $this->albumTable = new AlbumTable($this->tableGateway->reveal());
    }

    public function testFetchAllReturnsAllAlbums()
    {
        $resultSet = $this->prophesize(ResultSetInterface::class)->reveal();
        $this->tableGateway->select()->willReturn($resultSet);

        $this->assertSame($resultSet, $this->albumTable->fetchAll());
    }
}
```

由于我们这里测试了 `AlbumTable` 而没有测试 `TableGateway` 类 （已经在 zend-db 中进行过测试了），
我们仅仅只想确保 `AlbumTable` 类以我们期望的方式和 `TableGateway` 类进行互动。
以上，我们测试了 `AlbumTable` 调用 `fetchAll()` 方法的时候是否调用了
`$tableGateway` 属性的 `select()` 不带参数方法。
如果有，应该返回一个 `ResultSet` 实例。最终，我们期望在调用方法的时候返回 `ResultSet` 对象。
这个测试应该是可以正常运行的，所以我们现在可以添加其余的测试方法：

```php
public function testCanDeleteAnAlbumByItsId()
{
    $this->tableGateway->delete(['id' => 123])->shouldBeCalled();
    $this->albumTable->deleteAlbum(123);
}

public function testSaveAlbumWillInsertNewAlbumsIfTheyDontAlreadyHaveAnId()
{
    $albumData = [
        'artist' => 'The Military Wives',
        'title'  => 'In My Dreams'
    ];
    $album = new Album();
    $album->exchangeArray($albumData);

    $this->tableGateway->insert($albumData)->shouldBeCalled();
    $this->albumTable->saveAlbum($album);
}

public function testSaveAlbumWillUpdateExistingAlbumsIfTheyAlreadyHaveAnId()
{
    $albumData = [
        'id'     => 123,
        'artist' => 'The Military Wives',
        'title'  => 'In My Dreams',
    ];
    $album = new Album();
    $album->exchangeArray($albumData);

    $resultSet = $this->prophesize(ResultSetInterface::class);
    $resultSet->current()->willReturn($album);

    $this->tableGateway
        ->select(['id' => 123])
        ->willReturn($resultSet->reveal());
    $this->tableGateway
        ->update(
            array_filter($albumData, function ($key) {
                return in_array($key, ['artist', 'title']);
            }, ARRAY_FILTER_USE_KEY),
            ['id' => 123]
        )->shouldBeCalled();

    $this->albumTable->saveAlbum($album);
}

public function testExceptionIsThrownWhenGettingNonExistentAlbum()
{
    $resultSet = $this->prophesize(ResultSetInterface::class);
    $resultSet->current()->willReturn(null);

    $this->tableGateway
        ->select(['id' => 123])
        ->willReturn($resultSet->reveal());

    $this->expectException(RuntimeException::class);
    $this->expectExceptionMessage('Could not find row with identifier 123');
    $this->albumTable->getAlbum(123);
}
```

这些测试没有什么复杂度应该比较容易理解。在每个测试中，我们为模拟的表单网关添加断言，
然后调用我们 `AlbumTable` 中的方法添加断言。

我们会做如下测试：

1. 我们可以通过 ID 取回专辑信息。
2. 我们可以删除专辑。
3. 我们可以保存一个新专辑。
4. 我们可以编辑一个存在的专辑。
5. 如果我们试图去取出一个不存在的数据将会存抛出一个异常。

最后运行 `phpunit` 将会获取如下信息：

```bash
$ ./vendor/bin/phpunit --testsuite Album
PHPUnit 5.4.6 by Sebastian Bergmann and contributors.

.............                                                     13 / 13 (100%)

Time: 151 ms, Memory: 14.00MB

OK (13 tests, 31 assertions)
```

## 结论

在这个简短的教程中，我们给出了一些怎么测试 zend-mvc 不同部分的示例。
我们讨论了测试环境，如何测试控制器和操作，如何对待失败的情况，
如何配置服务管理器，以及怎么去测试模型示例和表单模型。

本教程绝对不是一个指导单元测试的指导手册，只是一个帮助您开发更高级应用程序的垫脚石。
