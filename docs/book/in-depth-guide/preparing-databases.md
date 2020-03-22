# Preparing for Different Databases

在上一个章节中，我们创建了一个 `PostRepository` 用来返回一些博客帖子的数据。
这是以学习为目的，在现实中这是个不切实际的应用；
没有人会希望在每次添加新帖子的时候去修改源代码！
幸运的是我们可以将数据库作为帖子储存的实际位置；
我们需要学习的就是在应用中如何与数据库进行交互。
但是有个小问题：存在着许多的数据库系统，包括关系数据库、文档数据库、键/值存储以及图形数据库。
你可能比较倾向于直接去解决适合你当前应用程序的需求的方案，
但是最佳的实践方案是在你真实数据库交互方案上另加一层抽象的访问方案。
在我们上一个章节中使用的 *repository* 方法就是这样一种实践，其主要针对的是 *queries*。
在本节中，我们将会扩展它添加 *command* 功能来实现创建、更新和删除操作。

## 什么是数据库抽象化？

“数据库抽象化（Database abstraction）” 是为所有的数据库交互提供一个通用接口的行为。
对于 SQL 和 NoSQ 数据库；他们都有增删改查（Create, Read, Update, Delete）操作。
例如，从 Mysql 中请求并获取一个给定的行我们你可能使用下面的方式

```php
$results = mysqli_query('SELECT foo FROM bar')`;
```

然而，对于 MongoDB， 使用类似的方式：

```php
$results = $mongoDbClient->app->bar->find([], ['foo' => 1, '_id' => 0])`;
```

两个引擎都将给你同样的结果，但是执行方式却是不同的。

所以如果最开始我们使用 SQL 数据库并直接将代码写进 `PostRepository`
一年后我们决定切换到 NoSQL 数据去，那么以前的实现方式就变得没有用处了。
几年后有出现一个新的引擎，我们又不得不又重新开始。

如果我们最开始没有创建接口，我们就可能需要不断的修改我们的代码！

最重要的是，我们我们可能会使用某种分布式缓存层来实现 *read* 操作（获取项目），
当 *write* 操作的时候我们会直接将其写入真实的数据库。最理想的情况，
我们不希望控制器更多的关心这些细节，但我们需要在架构的时候考虑到这点。

在代码层，接口就是为我们处理不同实现的抽象层，我们现在只处理查询。
让我们来仔细讨论下。

## 添加抽象操作

我们先想下能有哪些可能与数据库的交互，我们可能需要这些：

- 查询一个独立的帖子列表
- 查询所有的帖子
- 插入一个新的帖子
- 更新一个存在的帖子
- 删除一个存在的帖子

这里，我们的 `PostRepositoryInterface` 先处理前面两个需求。
考虑到这是最有可能采用不同后端来实现的部分，我们可能希望其与更改才做层分开。

接下来我们在 `module/Blog/src/Model/PostCommandInterface.php`
中创建一个新的接口 `Blog\Model\PostCommandInterface` 内容如下：

```php
namespace Blog\Model;

interface PostCommandInterface
{
    /**
     * 在系统中保存一个新的帖子.
     *
     * @param Post $post The post to insert; may or may not have an identifier.
     * @return Post The inserted post, with identifier.
     */
    public function insertPost(Post $post);

    /**
     * 更新一个系统中存在的帖子.
     *
     * @param Post $post The post to update; must have an identifier.
     * @return Post The updated post.
     */
    public function updatePost(Post $post);

    /**
     * 从系统中删除一个帖子.
     *
     * @param Post $post The post to delete.
     * @return bool
     */
    public function deletePost(Post $post);
}
```

这个接口为我们在模型中为每个 *command* 都定义了方法。
每个方法都接收一个 `Post` 实例，并由其发出命令来对实例进行何种操作。
对于插入操作，我们的 `Post` 不需要一个 ID（在构造函数中这个值会被定义为 null），
但是我们会返回一个具有 ID 的新实例，同样的更新操作也将会返回一个更新后的实例
（也可能是同一个实例！），删除操作会返回操作是否成功的标记。

## 总结

这里我们不准备使用新的接口；在接下来的几个章节中我们会对其做阶段性的设置，
我们将准备使用 zend-db 来实现持久化，并在后面创建一个的新的控制器来处理博客帖子的操作。
