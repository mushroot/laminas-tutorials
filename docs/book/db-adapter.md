# 配置数据库适配器

laminas-db 提供了一个通用的数据库抽象层。其核心就是 `Adapter`，
他对我们常用的数据库的常用操作都做了抽象处理。

在本教程中，我们将学习如何配置单个的数据库适配器，以及多个适配器
（这在集群环境下的读写分离结果可能会比较有用）。

## 安装 laminas-db

首先, 使用 Composer 安装 laminas-db:

```bash
$ composer require laminas/laminas-db
```

### 安装并自动配置

如果使用 [laminas-component-installer](https://docs.laminas.dev/laminas-component-installer/)
（默认情况下会随着 skeleton application以及 Mezzio 一起安装），
并选择需要安装配置的文件。

- 对于 laminas-mvc 应用, 选择`application.config.php` 或者
  `modules.config.php`.
- 对于 Mezzio 应用, 选择 `config/config.php`.

### 安装并手动配置

如果捏没有使用安装脚本，你可能需要手动配置并将组件添加到应用中。

#### 配置一个 laminas-mvc-based 应用

对于 laminas-mvc 应用，需要更新模块列表配置，
在 `config/application.config.php` or `config/modules.config.php`
文件的顶部添加 `'Laminas\Db'`：
  
```php
// In config/modules.config.php
return [
    'Laminas\Db', // <-- This line
    'Laminas\Form', 
    /* ... */
];

// OR in config/application.config.php
return [
    /* ... */
    // Retrieve list of modules used in this application.
    'modules' => [
        'Laminas\Db', // <-- This line
        'Laminas\Form', 
        /* ... */
    ],
    /* ... */
];
```

#### 配置一个 mezzio-based 应用

对于 Mezzio 应用，创建一个新文件，
`config/autoload/laminas-db.global.php`
内容如下：

```php
use Laminas\Db\ConfigProvider;

return (new ConfigProvider())();
```

## 配置默认适配器

在工厂服务中，你可以使用类名 `Laminas\Db\Adapter\AdapterInterface`
在应用容器中检索默认的适配器：

```php
use Laminas\Db\Adapter\AdapterInterface;

function ($container) {
    return new SomeServiceObject($container->get(AdapterInterface::class));
}
```

当我们安装并配置好了，与工厂服务关联的 `AdapterInterface`
将在配置文件的顶层炒作 `db` key值，并试用其创建一个适配器。
例如，下面的配置将会使用 PDO 连接一个 MySQL 数据库，并关联其PDO DSN：

```php
// In config/autoload/global.php
return [
    'db' => [
        'driver' => 'Pdo',
        'dsn'    => 'mysql:dbname=laminastutorial;host=localhost;charset=utf8',
    ],
];
```

想要获取关于适配的更多信息，可以查看文档
[Laminas\\Db\\Adapter](http://docs.laminas.dev/laminas-db/adapter/#creating-an-adapter-using-dependency-injection).

## 配置命名适配器

某些时候，你可能需要多个适配器。
例如，如果你使用集群数据库，并开启了读写分离。

laminas-db 为这种情况提供了一个 [abstract factory](https://docs.laminas.dev/laminas-servicemanager/configuring-the-service-manager/#abstract-factories),
`Laminas\Db\Adapter\AdapterAbstractServiceFactory`。
为了使用他，需要在 `db.adapters` 下为每个适配器创建命名配置：

```php
// In config/autoload/global.php
return [
    'db' => [
        'adapters' => [
            'Application\Db\WriteAdapter' => [
                'driver' => 'Pdo',
                'dsn'    => 'mysql:dbname=application;host=canonical.example.com;charset=utf8',
            ],
            'Application\Db\ReadOnlyAdapter' => [
                'driver' => 'Pdo',
                'dsn'    => 'mysql:dbname=application;host=replica.example.com;charset=utf8',
            ],
        ],
    ],
];
```

你需要使用适配器的名称来调用他们，所以，需要确保他们再应用中是唯一的，
并且需要描述其作用！

### 使用命名适配器

在工厂中使用命名适配和是用别的服务一样：


```php
function ($container) {
    return new SomeServiceObject($container->get('Application\Db\ReadOnlyAdapter'));
}
```

### 使用 `AdapterAbstractServiceFactory` 作为工厂

依照使用应用程序容器的不同，抽象方法可能也会不一样。
另外，从容器中检索适配器的时候，你可能想减少检索的时间
（抽象工厂是最后查询的！）
laminas-servicemanager 抽象工厂以其自身作为一个抽象工厂，
并且将服务名称作为自变量传递，从而允许它们根据请求的服务名称更改其返回值。 
这样，您还可以添加以下服务配置：

```php
use Laminas\Db\Adapter\AdapterAbstractServiceFactory;

// If using laminas-mvc:
// In module/YourModule/config/module.config.php
'service_manager' => [
    'factories' => [
        'Application\Db\WriteAdapter' => AdapterAbstractServiceFactory::class,
    ],
],

// If using Mezzio
'dependencies' => [
    'factories' => [
        'Application\Db\WriteAdapter' => AdapterAbstractServiceFactory::class,
    ],
],
```
