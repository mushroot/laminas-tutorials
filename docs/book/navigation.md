# 在 Album 模块中使用 laminas-navigation

当前教程我们将使用 [zend-navigation 组件](https://docs.laminas.dev/laminas-navigation/intro/)
来添加一个导航菜单到屏幕顶端的黑色导航栏上，并且在站点内容的上面添加一个面包屑导航。

## 准备

在实际应用中，专辑页面将只是整个站点的一部分。通常，用户会访问首页，
接下来会通过使用一个标准的导航菜单来访问专辑页面。
因此我们的网站将来实现的不仅仅是只有专辑功能，让框架的欢迎页面作为主页，
并使用 `/album` 路由来访问我们的专辑模块。
我们实现这个功能，我们需要对之前的代码进行一些小的修改。
当前，应用的 (`/`) 路由默认指向 `AlbumController`。
我们需求撤销这个路由操作，这样我们的应用就拥有了两个入口，首页和专辑页面。

```php
// In module/Application/config/module.config.php:
'home' => [
   'type' => Literal::class,
    'options' => [
        'route'    => '/',
        'defaults' => [
            'controller' => Controller\IndexController::class, // <-- change back here
            'action'     => 'index',
        ],
    ],
],
```

(你现在还可以移除 `Album\Controller\AlbumController` 类的引入)

这个修改意味着当我们访问我们应用的首页
(`http://localhost:8080/` or `http://laminas-mvc-tutorial.localhost/`),
的时候，你将会看见框架的默认介绍页。你的专辑页面任然可以通过 `/album` 访问。

## 设置 laminas-navigation

首先，我们需要安装 laminas-navigation， 在你项目的根目录执行如下代码：

```bash
$ composer require laminas/laminas-navigation
```

假定你已经看过了 [入门教程](getting-started/overview.md),
你将按照 [laminas-component-installer](https://docs.laminas.dev/laminas-component-installer)
插件的提示来注入 `Zend\Navigation`; 确保选择了对
`config/application.config.php` 或 `config/modules.config.php` 操作的选项;
既然为安装单一的包，你可以选择 "y" 或 "n" 来记住“是否对相同的操作采用同样的选择” 选项。

> ### 手动配置
>
> 如果你没有使用 zend-component-installer，你讲需要手动来配置。
> 你只需要两步：
>
> - 在 `config/application.config.php` or `config/modules.config.php`
>   中注册  `Zend\Navigation` 模块.
>   确保将其放在了你自己定义的以及你正在使用的第三方模块列表的顶部。
> - 然后，添加一个新文件 `config/autoload/navigation.global.php`,
>   内容如下：
>
>   ```php
>   <?php
>   use Laminas\Navigation\ConfigProvider;
>   
>   return [
>       'service_manager' => (new ConfigProvider())->getDependencyConfig(),
>   ];
>   ```

一旦安装，我们的应用就可以使用 laminas-navigation 了，我们可以使用一些默认的工厂方法来实现。

## 配置我们的站点导航

接下来我们需要让 laminas-navigation 识别我们站点的结构级别。
我们需要在配置中添加一个 `navigation` 键，来呈现站点的结构信息。
我们将会在 `Application` 模块的配置文件中去配置：

```php
// in module/Application/config/module.config.php:
return [
    /* ... */

    'navigation' => [
        'default' => [
            [
                'label' => 'Home',
                'route' => 'home',
            ],
            [
                'label' => 'Album',
                'route' => 'album',
                'pages' => [
                    [
                        'label'  => 'Add',
                        'route'  => 'album',
                        'action' => 'add',
                    ],
                    [
                        'label'  => 'Edit',
                        'route'  => 'album',
                        'action' => 'edit',
                    ],
                    [
                        'label'  => 'Delete',
                        'route'  => 'album',
                        'action' => 'delete',
                    ],
                ],
            ],
        ],
    ],

    /* ... */
];
```

这个配置映射出了我们专辑模块的页面，包括了导航到操作的链接以及标签。
你可以通过页面以及子页面来定义层级更加复杂的针对路由，控制器/操作或者指向外部URI的链接。
更多操作详见[zend-navigation quick start](https://docs.laminas.dev/laminas-navigation/quick-start/).

## 在视图助手中添加目录

现在我们的配置通过服务管理器（service manager ）合并到了视图助手中，我们可以使用
[menu view helper](https://docs.laminas.dev/laminas-navigation/helpers/menu/)
想我们的布局层添加导航条了。

```php
<?php // in module/Application/view/layout/layout.phtml: ?>
<div class="collapse navbar-collapse">
    <?php // add this: ?>
    <?= $this->navigation('navigation')->menu() ?>
</div>
```

导航助手（navigation helper）默认由 laminas-view 提供，服务管理器会自动配置我们定义好的配置信息。
刷新你的应用，你将会看见一个正常运行的菜单；同时我们可以做一些调整，来让导航更加好看：

```php
<?php // in module/Application/view/layout/layout.phtml: ?>
<div class="collapse navbar-collapse">
    <?php // update to: ?>
    <?= $this->navigation('navigation')
        ->menu()
        ->setMinDepth(0)
        ->setMaxDepth(0)
        ->setUlClass('nav navbar-nav') ?>
</div>
```

这里我们为导航的根元素 `<ul>` 添加布局属性（class） `nav`
（这里使用的是 Bootstrap 框架），并且只展示一级导航页面。
如果你正在浏览器中浏览你的应用，你将会看见一个美丽的导航条。

zend-navigation 最实用的就是他会与路由集成，并且高亮正在访问的导航。
因为我们设置了当前活动页面导航的布局属性为 `active`；
Bootstrap 使用这个属性来高亮当前页面。

## 添加 Breadcrumbs 导航

添加 Breadcrumbs 的步骤类似。我们想要在你的 `layout.phtml` 中添加一个和上面类似的
Breadcrumbs 导航在页面信息的顶部，一遍我们的用户知道自己在我们站点当前访问的位置。
在前面添加容器 `<div>`，并通过
[breadcrumbs view helper](https://docs.laminas.dev/laminas-navigation/helpers/breadcrumbs/) 添加一个 breadcrumb 导航：

```php
<?php // module/Application/view/layout/layout.phtml: ?>
<div class="container">
    <?php // add the following line: ?>
    <?= $this->navigation('navigation')->breadcrumbs()->setMinDepth(0) ?>
    <?= $this->content ?>
</div>
```

这将为我们的页面添加一个简单而实用的 breadcrumb （我们设置其渲染深度为0，这样我们就可以看见所有页面的基表）
但是我们可以做的更好。因为 Bootstrap 框架拥有一个 breadcrumb 布局样式，让我们添加一个使用
Bootstrap 样式的partial。我们将会在 `Application` 模块的 `view` 目录来创建它
（partial 将会在应用这被广泛的调用，而不是仅仅在 album 中使用）:

```php
<?php // in module/Application/view/partial/breadcrumb.phtml: ?>
<ul class="breadcrumb">
    <?php
    // iterate through the pages
    foreach ($this->pages as $key => $page):
    ?>
        <li>
            <?php
            // if this isn't the last page, add a link and the separator:
            if ($key < count($this->pages) - 1):
            ?>
                <a href="<?= $page->getHref() ?>"><?= $page->getLabel() ?></a>
            <?php
            // otherwise, output the name only:
            else:
            ?>
                <?= $page->getLabel() ?>
            <?php endif; ?>
        </li>
    <?php endforeach; ?>
</ul>
```
注意 partial 是接收 `Laminas\View\Model\ViewModel` 中使用 `pages` 属性来设置的数组参数来进行渲染的。  

现在，我们需要告诉 breadcrumb 视图助手使用我们编写的 partial：

```php
<?php // in module/Application/view/layout/layout.phtml: ?>
<div class="container">
    <?php // Update to: ?>
    <?= $this->navigation('navigation')
            ->breadcrumbs()
            ->setMinDepth(0)
            ->setPartial('partial/breadcrumb') ?>
    <?= $this->content ?>
</div>
```

刷新页面，我们将会在每个页面中看见一个具有样式的 breadcrumbs。
