# 在 Album 模块中使用 laminas-paginator

在这个教程中，我们使用 [laminas-paginator component](https://docs.laminas.dev/laminas-paginator/intro/)
在专辑列表的底部添加一个便捷的分页控制器。


当前，我们只有屈指可数的几个专辑可以显示，所以所有的专辑都显示在一个页面上市没有问题的。
然而，如果我们数据中的专辑数超过了100或者更多呢？标准的解决方法就是将数据放置在不同的页码中，
并允许用户使用分页控制器切换不同页码。就像你在 Google 中搜索 "Laminas"，
你可以在页面的底部看见的分页控制器那样：

![Example pagination control](images/pagination.sample.png)

## 准备工作

开始之前，我们使用 PHP 的 PDO 驱动来使用 sqlite 数据库。
创建一个内容如下的 `data/album-fixtures.sql` 文件：

```sql
INSERT INTO "album" ("artist", "title")
VALUES
    ("David Bowie", "The Next Day (Deluxe Version)"),
    ("Bastille", "Bad Blood"),
    ("Bruno Mars", "Unorthodox Jukebox"),
    ("Emeli Sandé", "Our Version of Events (Special Edition)"),
    ("Bon Jovi", "What About Now (Deluxe Version)"),
    ("Justin Timberlake", "The 20/20 Experience (Deluxe Version)"),
    ("Bastille", "Bad Blood (The Extended Cut)"),
    ("P!nk", "The Truth About Love"),
    ("Sound City - Real to Reel", "Sound City - Real to Reel"),
    ("Jake Bugg", "Jake Bugg"),
    ("Various Artists", "The Trevor Nelson Collection"),
    ("David Bowie", "The Next Day"),
    ("Mumford & Sons", "Babel"),
    ("The Lumineers", "The Lumineers"),
    ("Various Artists", "Get Ur Freak On - R&B Anthems"),
    ("The 1975", "Music For Cars EP"),
    ("Various Artists", "Saturday Night Club Classics - Ministry of Sound"),
    ("Hurts", "Exile (Deluxe)"),
    ("Various Artists", "Mixmag - The Greatest Dance Tracks of All Time"),
    ("Ben Howard", "Every Kingdom"),
    ("Stereophonics", "Graffiti On the Train"),
    ("The Script", "#3"),
    ("Stornoway", "Tales from Terra Firma"),
    ("David Bowie", "Hunky Dory (Remastered)"),
    ("Worship Central", "Let It Be Known (Live)"),
    ("Ellie Goulding", "Halcyon"),
    ("Various Artists", "Dermot O'Leary Presents the Saturday Sessions 2013"),
    ("Stereophonics", "Graffiti On the Train (Deluxe Version)"),
    ("Dido", "Girl Who Got Away (Deluxe)"),
    ("Hurts", "Exile"),
    ("Bruno Mars", "Doo-Wops & Hooligans"),
    ("Calvin Harris", "18 Months"),
    ("Olly Murs", "Right Place Right Time"),
    ("Alt-J (?)", "An Awesome Wave"),
    ("One Direction", "Take Me Home"),
    ("Various Artists", "Pop Stars"),
    ("Various Artists", "Now That's What I Call Music! 83"),
    ("John Grant", "Pale Green Ghosts"),
    ("Paloma Faith", "Fall to Grace"),
    ("Laura Mvula", "Sing To the Moon (Deluxe)"),
    ("Duke Dumont", "Need U (100%) [feat. A*M*E] - EP"),
    ("Watsky", "Cardboard Castles"),
    ("Blondie", "Blondie: Greatest Hits"),
    ("Foals", "Holy Fire"),
    ("Maroon 5", "Overexposed"),
    ("Bastille", "Pompeii (Remixes) - EP"),
    ("Imagine Dragons", "Hear Me - EP"),
    ("Various Artists", "100 Hits: 80s Classics"),
    ("Various Artists", "Les Misérables (Highlights From the Motion Picture Soundtrack)"),
    ("Mumford & Sons", "Sigh No More"),
    ("Frank Ocean", "Channel ORANGE"),
    ("Bon Jovi", "What About Now"),
    ("Various Artists", "BRIT Awards 2013"),
    ("Taylor Swift", "Red"),
    ("Fleetwood Mac", "Fleetwood Mac: Greatest Hits"),
    ("David Guetta", "Nothing But the Beat Ultimate"),
    ("Various Artists", "Clubbers Guide 2013 (Mixed By Danny Howard) - Ministry of Sound"),
    ("David Bowie", "Best of Bowie"),
    ("Laura Mvula", "Sing To the Moon"),
    ("ADELE", "21"),
    ("Of Monsters and Men", "My Head Is an Animal"),
    ("Rihanna", "Unapologetic"),
    ("Various Artists", "BBC Radio 1's Live Lounge - 2012"),
    ("Avicii & Nicky Romero", "I Could Be the One (Avicii vs. Nicky Romero)"),
    ("The Streets", "A Grand Don't Come for Free"),
    ("Tim McGraw", "Two Lanes of Freedom"),
    ("Foo Fighters", "Foo Fighters: Greatest Hits"),
    ("Various Artists", "Now That's What I Call Running!"),
    ("Swedish House Mafia", "Until Now"),
    ("The xx", "Coexist"),
    ("Five", "Five: Greatest Hits"),
    ("Jimi Hendrix", "People, Hell & Angels"),
    ("Biffy Clyro", "Opposites (Deluxe)"),
    ("The Smiths", "The Sound of the Smiths"),
    ("The Saturdays", "What About Us - EP"),
    ("Fleetwood Mac", "Rumours"),
    ("Various Artists", "The Big Reunion"),
    ("Various Artists", "Anthems 90s - Ministry of Sound"),
    ("The Vaccines", "Come of Age"),
    ("Nicole Scherzinger", "Boomerang (Remixes) - EP"),
    ("Bob Marley", "Legend (Bonus Track Version)"),
    ("Josh Groban", "All That Echoes"),
    ("Blue", "Best of Blue"),
    ("Ed Sheeran", "+"),
    ("Olly Murs", "In Case You Didn't Know (Deluxe Edition)"),
    ("Macklemore & Ryan Lewis", "The Heist (Deluxe Edition)"),
    ("Various Artists", "Defected Presents Most Rated Miami 2013"),
    ("Gorgon City", "Real EP"),
    ("Mumford & Sons", "Babel (Deluxe Version)"),
    ("Various Artists", "The Music of Nashville: Season 1, Vol. 1 (Original Soundtrack)"),
    ("Various Artists", "The Twilight Saga: Breaking Dawn, Pt. 2 (Original Motion Picture Soundtrack)"),
    ("Various Artists", "Mum - The Ultimate Mothers Day Collection"),
    ("One Direction", "Up All Night"),
    ("Bon Jovi", "Bon Jovi Greatest Hits"),
    ("Agnetha Fältskog", "A"),
    ("Fun.", "Some Nights"),
    ("Justin Bieber", "Believe Acoustic"),
    ("Atoms for Peace", "Amok"),
    ("Justin Timberlake", "Justified"),
    ("Passenger", "All the Little Lights"),
    ("Kodaline", "The High Hopes EP"),
    ("Lana Del Rey", "Born to Die"),
    ("JAY Z & Kanye West", "Watch the Throne (Deluxe Version)"),
    ("Biffy Clyro", "Opposites"),
    ("Various Artists", "Return of the 90s"),
    ("Gabrielle Aplin", "Please Don't Say You Love Me - EP"),
    ("Various Artists", "100 Hits - Driving Rock"),
    ("Jimi Hendrix", "Experience Hendrix - The Best of Jimi Hendrix"),
    ("Various Artists", "The Workout Mix 2013"),
    ("The 1975", "Sex"),
    ("Chase & Status", "No More Idols"),
    ("Rihanna", "Unapologetic (Deluxe Version)"),
    ("The Killers", "Battle Born"),
    ("Olly Murs", "Right Place Right Time (Deluxe Edition)"),
    ("A$AP Rocky", "LONG.LIVE.A$AP (Deluxe Version)"),
    ("Various Artists", "Cooking Songs"),
    ("Haim", "Forever - EP"),
    ("Lianne La Havas", "Is Your Love Big Enough?"),
    ("Michael Bublé", "To Be Loved"),
    ("Daughter", "If You Leave"),
    ("The xx", "xx"),
    ("Eminem", "Curtain Call"),
    ("Kendrick Lamar", "good kid, m.A.A.d city (Deluxe)"),
    ("Disclosure", "The Face - EP"),
    ("Palma Violets", "180"),
    ("Cody Simpson", "Paradise"),
    ("Ed Sheeran", "+ (Deluxe Version)"),
    ("Michael Bublé", "Crazy Love (Hollywood Edition)"),
    ("Bon Jovi", "Bon Jovi Greatest Hits - The Ultimate Collection"),
    ("Rita Ora", "Ora"),
    ("g33k", "Spabby"),
    ("Various Artists", "Annie Mac Presents 2012"),
    ("David Bowie", "The Platinum Collection"),
    ("Bridgit Mendler", "Ready or Not (Remixes) - EP"),
    ("Dido", "Girl Who Got Away"),
    ("Various Artists", "Now That's What I Call Disney"),
    ("The 1975", "Facedown - EP"),
    ("Kodaline", "The Kodaline - EP"),
    ("Various Artists", "100 Hits: Super 70s"),
    ("Fred V & Grafix", "Goggles - EP"),
    ("Biffy Clyro", "Only Revolutions (Deluxe Version)"),
    ("Train", "California 37"),
    ("Ben Howard", "Every Kingdom (Deluxe Edition)"),
    ("Various Artists", "Motown Anthems"),
    ("Courteeners", "ANNA"),
    ("Johnny Marr", "The Messenger"),
    ("Rodriguez", "Searching for Sugar Man"),
    ("Jessie Ware", "Devotion"),
    ("Bruno Mars", "Unorthodox Jukebox"),
    ("Various Artists", "Call the Midwife (Music From the TV Series)"
);
```
（测试数据选择了编写文档时 iTunes 专辑榜的前 150 调数据！）

现在我们使用如下命令将数据导入数据库：

```bash
$ sqlite data/laminastutorial.db < data/album-fixtures.sql
```

一些系统，例如 Ubuntu，需要使用 `sqlite3` 命令；根据你当前系统选择哪条命令。

> ### 使用 PHP 创建数据库
>
> 如果你没有在系统上安装 Sqlite，你可以使用 PHP 很容易的将 SQL 源文件导入数据库。
> 创建一个内容如下的 `data/load_album_fixtures.php` 文件：
>
> ```php
> <?php
> $db = new PDO('sqlite:' . realpath(__DIR__) . '/laminastutorial.db');
> $fh = fopen(__DIR__ . '/album-fixtures.sql', 'r');
> while ($line = fread($fh, 4096)) {
>     $db->exec($line);
> }
> fclose($fh);
> ```
>
> 然后执行:
>
> ```bash
> $ php data/load_album_fixtures.php
> ```

这将使我们额外拥有了 150 行的专辑。如果你现在访问 `/album` 查看你的专辑列表，
你将会看见一个长达 150 行的专辑列表，这非常不友好。

## 安装 laminas-paginator

laminas-paginator 默认没有安装或配置的，因此我们可以通过如下命令安装:

```bash
$ composer require laminas/laminas-paginator
```

确保已经看过 [Getting Started tutorial](getting-started/overview.md),
你可以使用 [laminas-component-installer](https://docs.laminas.dev/laminas-component-installer)
脚本来注入 `Laminas\Paginator`; 并确保对
`config/application.config.php` 或 `config/modules.config.php` 进行了相应配置;
既然我们安装的是一个独立的包, 你可以在 "Remember this
option for other packages of the same type" 时候选择 "y" or "n".

> ### 手动配置
>
> 如果你没有使用 laminas-component-installer, 你将需要手动的来配置。
> 你只需要两步：
>
> - 在 `config/application.config.php` or `config/modules.config.php`
>   注册 `Laminas\Paginator` 模块。
>   确保将其放在了自己定义的以及正在使用的第三方模块列表的上方。
> - 然后，添加一个新文件, `config/autoload/paginator.global.php`,
>   内容如下:
>
>   ```php
>   <?php
>   use Laminas\Paginator\ConfigProvider;
>   
>   return [
>       'service_manager' => (new ConfigProvider())->getDependencyConfig(),
>   ];
>   ```

一旦安装，我们的应用就可以使用 laminas-paginator 了，我们可以使用一些默认的工厂方法来实现。

## 修改 AlbumTable

为了让 laminas-paginator 自动请求我们数据库中的数据，
我们将使用 [DbSelect pagination adapter](https://docs.laminas.dev/laminas-paginator/usage/#the-dbselect-adapter)
laminas-paginator 会自动调用并运行一个包含了正确的  `LIMIT` and `WHERE` 条件的
`Laminas\Db\Sql\Select` 对象，以确保只返回我们当前页面需要的数据。
现在我们修改 `AlbumTable` 模型中的 `fetchAll` 方法，以便其可以返回一个分页对象：

```php
// in module/Album/src/Model/AlbumTable.php:
namespace Album\Model;

use RuntimeException;
use Laminas\Db\ResultSet\ResultSet;
use Laminas\Db\Sql\Select;
use Laminas\Db\TableGateway\TableGatewayInterface;
use Laminas\Paginator\Adapter\DbSelect;
use Laminas\Paginator\Paginator;

class AlbumTable
{
    /* ... */

    public function fetchAll($paginated = false)
    {
        if ($paginated) {
            return $this->fetchPaginatedResults();
        }

        return $this->tableGateway->select();
    }

    private function fetchPaginatedResults()
    {
        // Create a new Select object for the table:
        $select = new Select($this->tableGateway->getTable());

        // Create a new result set based on the Album entity:
        $resultSetPrototype = new ResultSet();
        $resultSetPrototype->setArrayObjectPrototype(new Album());

        // Create a new pagination adapter object:
        $paginatorAdapter = new DbSelect(
            // our configured select object:
            $select,
            // the adapter to run it against:
            $this->tableGateway->getAdapter(),
            // the result set to hydrate:
            $resultSetPrototype
        );

        $paginator = new Paginator($paginatorAdapter);
        return $paginator;
    }

    /* ... */
}
```

这将返回一个配置好的 `Paginator` 实例。我们已经告诉了我们的 `Select` 适配器使用我们创建的
`Select` 对象，使用该适配器的 `TableGateway` 对象，以及使用 `TableGateway`
相同的方式将结果混入 `Album` 实体。
这将意味着我们的分页结果将会返回和没有使用分页结果类似的 `Album` 对象。

## 修改 AlbumController

接下来我们需要告诉专辑控制器我们提供了一个 `Pagination` 对象来代替 `ResultSet`。
这这个对象都可以遍历并返回转换后的 `Album`  对象，因此我们不需要对视图脚本做过多的修改：

```php
// in module/Album/src/Controller/AlbumController.php:

/* ... */

public function indexAction()
{
    // Grab the paginator from the AlbumTable:
    $paginator = $this->table->fetchAll(true);

    // Set the current page to what has been passed in query string,
    // or to 1 if none is set, or the page is invalid:
    $page = (int) $this->params()->fromQuery('page', 1);
    $page = ($page < 1) ? 1 : $page;
    $paginator->setCurrentPageNumber($page);

    // Set the number of items per page to 10:
    $paginator->setItemCountPerPage(10);

    return new ViewModel(['paginator' => $paginator]);
}

/* ... */
```

这时我们已经从 `AlbumTable` 中获取到了配置好了的 `Paginator` 对象，
然后将可选的 `page` 参数传递给查询的字符串（验证后的）。
同时我们也会告诉分页控制器希望每个页面显示10个专辑。

## 更新视图脚本

现在，我们需要在视图脚本中迭代 `pagination` 视图变量，而不是 `albums` 变量：

```php
<?php // in module/Album/view/album/album/index.phtml:
$title = 'My albums';
$this->headTitle($title);
?>
<h1><?= $this->escapeHtml($title); ?></h1>
<p>
    <a href="<?= $this->url('album', ['action' => 'add']) ?>">Add new album</a>
</p>

<table class="table">
    <tr>
        <th>Title</th>
        <th>Artist</th>
        <th>&nbsp;</th>
    </tr>
    <?php foreach ($this->paginator as $album) : // <-- change here! ?>
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

在你的站点中访问 `/album`你将会看见一个10个专辑的列表，但是没有方法进入到别的页面。
下面我们将来实现这个功能。

## 创建分页控制器局部视图

就像我们在 [导航教程](navigation.md#adding-breadcrumbs) 中自定义 breadcrumbs 导航的那样，
我们需要创建一个自定义的局部视图控制器来渲染我们的分页控制器。同时，因为我们使用 Bootstrap
这将影响我们输出正确 HTML 的格式。让我们在 `module/Application/view/partial/`
目录中创建 partial，以便我们能在所有的控制器中都能使用这个控制脚本：

```php
<?php // in module/Application/view/partial/paginator.phtml: ?>
<?php if ($this->pageCount): ?>
<div>
  <ul class="pagination">
  <!-- Previous page link -->
  <?php if (isset($this->previous)): ?>
    <li>
      <a href="<?= $this->url($this->route, [], ['query' => ['page' => $this->previous]]) ?>">
        &lt;&lt;
      </a>
    </li>
  <?php else: ?>
    <li class="disabled">
      <a href="#">
        &lt;&lt;
      </a>
    </li>
  <?php endif ?>

  <!-- Numbered page links -->
  <?php foreach ($this->pagesInRange as $page): ?>
    <?php if ($page !== $this->current): ?>
      <li>
        <a href="<?= $this->url($this->route, [], ['query' => ['page' => $page]]) ?>">
          <?= $page ?>
        </a>
      </li>
    <?php else: ?>
      <li class="active">
        <a href="#"><?= $page ?></a>
      </li>
    <?php endif ?>
  <?php endforeach ?>

  <!-- Next page link -->
  <?php if (isset($this->next)): ?>
    <li>
      <a href="<?= $this->url($this->route, [], ['query' => ['page' => $this->next]]) ?>">
        &gt;&gt;
      </a>
    </li>
  <?php else: ?>
    <li class="disabled">
      <a href="#">
        &gt;&gt;
      </a>
    </li>
  <?php endif ?>
  </ul>
</div>
<?php endif ?>
```
partial 将会从创建连接到正确页面的分页控制（如果在分页控制器存在一个以上的页面的话）。
这里将渲染一个上一页连接（并且在当前页面为首页的时候禁止这个连接），接下来渲染中间页的链接
（并渲染包含样式的 partial，我们将在下一步将其传递给视图助手），最后，
我们会创建一个下一页的链接（如果当前页面为最后一个则禁止）。注意我们是否通过其
告诉你的控制器将 `page` 变量传递给了查询字符串。

### 在视图脚本中使用分页控制器

要对专辑实现翻页，我们需要调用 [paginationControl view helper](https://docs.laminas.dev/laminas-paginator/usage/#rendering-pages-with-view-scripts)
来显示我们的分页控制器

```php
<?php
// In module/Album/view/album/album/index.phtml:
// Add at the end of the file after the table:
?>
<?= $this->paginationControl(
    // The paginator object:
    $this->paginator,
    // The scrolling style:
    'sliding',
    // The partial to use to render the control:
    'partial/paginator',
    // The route to link to when a user clicks a control link:
    ['route' => 'album']
) ?>
```

上面使用了 `paginationControl` 助手，并且将分页实例传递与其，
[sliding scrolling style](https://docs.laminas.dev/laminas-paginator/advanced/#custom-scrolling-styles),
用于告诉我们的分页部分使用哪种产生链接的方式。
现在刷新你的应用，你将会看见一个基于 Bootstrap-styled 的分页控制器！
