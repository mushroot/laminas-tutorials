# 使用事件管理器

本教程将会深入探讨 laminas-eventmanager。

## 属于

- **Event**：操作名。
- **Listener**：对 *事件* 做出任何反应的PHP回调。
- **EventManager**： 聚合了一个或多个事件的监听器，并触发事件。

通常，*事件* 会被包装成一个对象，包含了如何触发事件的元数据，事件名称，
触发对象（目标），以及需要提供的参数。
事件将会被 *命名*，并允许单个 *监听器* 基于事件处理逻辑分支。

## 开始

使用事件最基本的要求是：

- 一个 `EventManager` 实例
- 一个或者多个事件及其一个或者多个监听器
- 调用 `trigger()` 事件

一个基本的示例如下：

```php
use Laminas\EventManager\EventManager;

$events = new EventManager();
$events->attach('do', function ($e) {
    $event = $e->getName();
    $params = $e->getParams();
    printf(
        'Handled event "%s", with parameters %s',
        $event,
        json_encode($params)
    );
});

$params = ['foo' => 'bar', 'baz' => 'bat'];
$events->trigger('do', null, $params);
```

以上内容将会返回如下内容：

```text
Handled event "do", with parameters {"foo":"bar","baz":"bat"}
```

> ### 并非一定要使用闭包
>
> 本教程中，我们使用闭包函数作为监听器。但是，任何有效的PHP回调都可以作为监听器使用：
> PHP 函数名，静态类方法，对象实例，函数或者闭包。我们在本文中仅使用闭包。

### 事件实例

`trigger()` 是比较常用的，他会为你创建一个 `Laminas\EventManager\Event` 实例。
同时你可能想要首场创建实例；例如，希望复用同一实例来触发多个事件，或者你希望使用一些自定义的实例。

`Laminas\EventManager\Event`, 是附带的默认事件类型，并且 `EventManager` 使用的事件类型都
需要一个构造函数，该构造函数同 `trigger()` 一样接收三个参数：

```php
use Laminas\EventManager\Event;

$event = new Event('do', null, $params);
```

当你具有了一个可用的实例之后，你就需要使用 `EventManager` 的 `triggerEvent()` 来触发事件，例如：

```php
$events->triggerEvent($event);
```

### Event targets

如果你观察过上面第一个实例，你将会注意到在调用 `trigger()` 以及创建 `Event` 实例的时候
设置的第二个参数 `null`，为什么需要这么设置呢？

通常，你需要在类中构建一个 `EventManager`，以允许在方法内进行触发操作。
`trigger()` 中间一个参数就是 "target"，简单点说就是当前对象的实例，
这就使得事件监听器可以访问并调用对象，这是很实用的。

```php
use Laminas\EventManager\EventManager;
use Laminas\EventManager\EventManagerAwareInterface;
use Laminas\EventManager\EventManagerInterface;

class Example implements EventManagerAwareInterface
{
    protected $events;

    public function setEventManager(EventManagerInterface $events)
    {
        $events->setIdentifiers([
            __CLASS__,
            get_class($this),
        ]);
        $this->events = $events;
    }

    public function getEventManager()
    {
        if (! $this->events) {
            $this->setEventManager(new EventManager());
        }
        return $this->events;
    }

    public function doIt($foo, $baz)
    {
        $params = compact('foo', 'baz');
        $this->getEventManager()->trigger(__FUNCTION__, $this, $params);
    }

}

$example = new Example();

$example->getEventManager()->attach('doIt', function($e) {
    $event  = $e->getName();
    $target = get_class($e->getTarget()); // "Example"
    $params = $e->getParams();
    printf(
        'Handled event "%s" on target "%s", with parameters %s',
        $event,
        $target,
        json_encode($params)
    );
});

$example->doIt('bar', 'bat');
```

上面的内容与第一个示例基本相同。主要的区别在于我们使用了中间那个参数以传递 target，
即实例 `Example` 给监听器。监听器会查找 (`$e->getTarget()`) 并执行。

如果你足够仔细，可能会有一个新的问题：`setIdentifiers()` 又是用来干嘛的呢？

## Shared managers

一方面 `EventManager` 提供了 `SharedEventManagerInterface` 的实现方式。

`Laminas\EventManager\SharedEventManagerInterface` 定义了一个对象，
该对象聚合监听器以监听附加到具有特定 *标识符* 的对象。他本身并不能主动触发。
相反的是，构成 `SharedEventManager` 的 `EventManager` 实例会在
`SharedEventManager` 上查询具有某种标识符的监听器并触发他。

那么他们是如何工作的呢？

见下面示例：

```php
use Laminas\EventManager\SharedEventManager;

$sharedEvents = new SharedEventManager();
$sharedEvents->attach('Example', 'do', function ($e) {
    $event  = $e->getName();
    $target = get_class($e->getTarget()); // "Example"
    $params = $e->getParams();
    printf(
        'Handled event "%s" on target "%s", with parameters %s',
        $event,
        $target,
        json_encode($params)
    );
});
```

这看起来和之前的示例几乎相同，不同点在于参数列表 *之前* 多了一个参数 `'Example'`。
这部分代码表示 “监听目标事件 'Example' 的 'do' 事件，并在收到通知的时候执行回调。”

这时就该 `EventManager` 的 `setIdentifiers()` 参数起作用了。
此方法接受一个字符串或者一个字符串数组，以定义给定实例相关的上下文名称或者目标名称。
如果给定一个数组，将会通知给定目标事件的所有监听器。

因此，回到我们的示例，假定前面的共享监听器都被注册了，并且 `Example` 类也如上定义了。
我们就可以执行如下操作了：

```php
$example = new Example();
$example->getEventManager()->setSharedManager($sharedEvents);
$example->do('bar', 'bat');
```

正常会打印如下结果：

```text
Handled event "do" on target "Example", with parameters {"foo":"bar","baz":"bat"}
```

现在我们按照如下方式扩展 `Example`：

```php
class SubExample extends Example
{
}
```

我们定义的 `setEventManager()` 方法有个有趣的点就是同时定义了 `__CLASS__` 和 `get_class($this)`。
这就意味着我们在 `SubExample` 类中调用 `do()` 也将出发共享监听器！同时，
One interesting aspect of our `setEventManager()` method is that we defined it
to listen both on `__CLASS__` and `get_class($this)`. This means that calling
`do()` on our `SubExample` class would also trigger the shared listener! It also
means that, if desired, we could attach to specifically `SubExample`, and
listeners attached to only the `Example` target would not be triggered.

Finally, the names used as contexts or targets need not be class names; they can
be some name that only has meaning in your application if desired. As an
example, you could have a set of classes that respond to "log" or "cache"
&mdash; and listeners on these would be notified by any of them.

> ### Use class names as identifiers
>
> We recommend using class names, interface names, and/or abstract class names
> for identifiers. This makes determining what events are available easier, as
> well as finding which listeners might be attaching to those events. Interfaces
> make a particularly good use case, as they allow attaching to a group of
> related classes a single operation.

At any point, if you do not want to notify shared listeners, pass a `null` value
to `setSharedManager()`:

```php
$events->setSharedManager(null);
```

and they will be ignored. If at any point, you want to enable them again, pass
the `SharedEventManager` instance:

```php
$events->setSharedManager($sharedEvents);
```

### Wildcards

So far, with both a normal `EventManager` instance and with the
`SharedEventManager` instance, we've seen the usage of string event and string
target names to which we want to attach. What if you want to attach a listener
to multiple events or targets?

The answer is to supply an array of events or targets, or a wildcard, `*`.

Consider the following examples:

```php
// Multiple named events:
$events->attach(
    ['foo', 'bar', 'baz'], // events
    $listener
);

// All events via wildcard:
$events->attach(
    '*', // all events
    $listener
);

// Multiple named targets:
$sharedEvents->attach(
    ['Foo', 'Bar', 'Baz'], // targets
    'doSomething', // named event
    $listener
);

// All targets via wildcard
$sharedEvents->attach(
    '*', // all targets
    'doSomething', // named event
    $listener
);

// Mix and match: multiple named events on multiple named targets:
$sharedEvents->attach(
    ['Foo', 'Bar', 'Baz'], // targets
    ['foo', 'bar', 'baz'], // events
    $listener
);

// Mix and match: all events on multiple named targets:
$sharedEvents->attach(
    ['Foo', 'Bar', 'Baz'], // targets
    '*', // events
    $listener
);

// Mix and match: multiple named events on all targets:
$sharedEvents->attach(
    '*', // targets
    ['foo', 'bar', 'baz'], // events
    $listener
);

// Mix and match: all events on all targets:
$sharedEvents->attach(
    '*', // targets
    '*', // events
    $listener
);
```

The ability to specify multiple targets and/or events when attaching can slim
down your code immensely.

> ### Wildcards can cause problems
>
> Wildcards, while they simplify listener attachment, can cause some problems.
> First, the listener must either be able to accept any incoming event, or it
> must have logic to branch based on the type of event, the target, or the
> event parameters. This can quickly become difficult to manage.
>
> Additionally, there are performance considerations. Each time an event is
> triggered, it loops through all attached listeners; if your listener cannot
> actually handle the event, but was attached as a wildcard listener, you're
> introducing needless cycles both in aggregating the listeners to trigger, and
> by handling the event itself.
>
> We recommend being specific about what you attach a listener to, in order
> to prevent these problems.

### Listener aggregates

Another approach to listening to multiple events is via a concept of listener
aggregates, represented by `Laminas\EventManager\ListenerAggregateInterface`. Via
this approach, a single class can listen to multiple events, attaching one or
more instance methods as listeners.

This interface defines two methods, `attach(EventManagerInterface $events)` and
`detach(EventManagerInterface $events)`. You pass an `EventManager` instance to
one and/or the other, and then it's up to the implementing class to determine
what to do.

The trait `Laminas\EventManager\ListenerAggregateTrait` defines a `$listeners`
property and common logic for detaching an aggregate's listeners. We'll use that
to demonstrate creating an aggregate logging listener:

```php
use Laminas\EventManager\EventInterface;
use Laminas\EventManager\EventManagerInterface;
use Laminas\EventManager\ListenerAggregateInterface;
use Laminas\EventManager\ListenerAggregateTrait;
use Laminas\Log\Logger;

class LogEvents implements ListenerAggregateInterface
{
    use ListenerAggregateTrait;

    private $log;

    public function __construct(Logger $log)
    {
        $this->log = $log;
    }

    public function attach(EventManagerInterface $events)
    {
        $this->listeners[] = $events->attach('do', [$this, 'log']);
        $this->listeners[] = $events->attach('doSomethingElse', [$this, 'log']);
    }

    public function log(EventInterface $e)
    {
        $event  = $e->getName();
        $params = $e->getParams();
        $this->log->info(sprintf('%s: %s', $event, json_encode($params)));
    }
}
```

Attach the aggregate by passing it an event manager instance:

```php
$logListener = new LogEvents($logger);
$logListener->attach($events);
```

Any events the aggregate attaches to will then be notified when triggered.

Why bother? For a couple of reasons:

- Aggregates allow you to have stateful listeners. The above example
  demonstrates this via the composition of the logger; another example would be
  tracking configuration options.
- Aggregates make detaching listeners easier, as you can detach all
  listeners a class defines at once.

## Introspecting results

Sometimes you'll want to know what your listeners returned. One thing to
remember is that you may have multiple listeners on the same event; the
interface for results must be consistent regardless of the number of listeners.

The `EventManager` implementation by default returns a
`Laminas\EventManager\ResponseCollection` instance. This class extends PHP's
`SplStack`, allowing you to loop through responses in reverse order (since the
last one executed is likely the one you're most interested in). It also
implements the following methods:

- `first()` will retrieve the first result received
- `last()` will retrieve the last result received
- `contains($value)` allows you to test all values to see if a given one was
  received, and returns a boolean `true` if found, and `false` if not.
- `stopped()` will return a boolean value indicating whether or not a
  short-circuit occured; more on this in the next section.

Typically, you should not worry about the return values from events, as the
object triggering the event shouldn't really have much insight into what
listeners are attached. However, sometimes you may want to short-circuit
execution if interesting results are obtained. (laminas-mvc uses this feature to
check for listeners returning responses, which are then returned immediately.)

### Short-circuiting listener execution

You may want to short-circuit execution if a particular result is obtained, or if
a listener determines that something is wrong, or that it can return something
quicker than the target.

As examples, one rationale for adding an `EventManager` is as a caching
mechanism. You can trigger one event early in the method, returning if a cache
is found, and trigger another event late in the method, seeding the cache.

The `EventManager` component offers two ways to handle this, depending on
whether you have an event instance already, or want the event manager to create
one for you.

- `triggerEventUntil(callable $callback, EventInterface $event)`
- `triggerUntil(callable $callback, $eventName, $target = null, $argv = [])`

In each case, `$callback` will be any PHP callable, and will be passed the
return value from the most recently executed listener. The `$callback` must then
return a boolean value indicating whether or not to halt execution; boolean
`true` indicates execution should halt.

Your consuming code can then check to see if execution was short-circuited by
using the `stopped()` method of the returned `ResponseCollection`.

Here's an example:

```php
public function someExpensiveCall($criteria1, $criteria2)
{
    $params  = compact('criteria1', 'criteria2');
    $results = $this->getEventManager()->triggerUntil(
        function ($r) {
            return ($r instanceof SomeResultClass);
        },
        __FUNCTION__, 
        $this, 
        $params
    );

    if ($results->stopped()) {
        return $results->last();
    }

    // ... do some work ...
}
```

With this paradigm, we know that the likely reason of execution halting is due
to the last result meeting the test callback criteria; as such, we return that
last result.

The other way to halt execution is within a listener, acting on the `Event`
object it receives. In this case, the listener calls `stopPropagation(true)`,
and the `EventManager` will then return without notifying any additional
listeners.

```php
$events->attach('do', function ($e) {
    $e->stopPropagation();
    return new SomeResultClass();
});
```

This, of course, raises some ambiguity when using the trigger paradigm, as you
can no longer be certain that the last result meets the criteria it's searching
on. As such, we recommend that you standardize on one approach or the other.

## Keeping it in order

On occasion, you may be concerned about the order in which listeners execute. As
an example, you may want to do any logging early, to ensure that if
short-circuiting occurs, you've logged; if implementing a cache, you may want
to return early if a cache hit is found, and execute late when saving to a
cache.

Each of `EventManager::attach()` and `SharedEventManager::attach()` accept one
additional argument, a *priority*. By default, if this is omitted, listeners get
a priority of 1, and are executed in the order in which they are attached.
However, if you provide a priority value, you can influence order of execution.

- *Higher* priority values execute *earlier*.
- *Lower* (negative) priority values execute *later*.

To borrow an example from earlier:

```php
$priority = 100;
$events->attach('Example', 'do', function($e) {
    $event  = $e->getName();
    $target = get_class($e->getTarget()); // "Example"
    $params = $e->getParams();
    printf(
        'Handled event "%s" on target "%s", with parameters %s',
        $event,
        $target,
        json_encode($params)
    );
}, $priority);
```

This would execute with high priority, meaning it would execute early. If we
changed `$priority` to `-100`, it would execute with low priority, executing
late.

While you can't necessarily know all the listeners attached, chances are you can
make adequate guesses when necessary in order to set appropriate priority
values.

We advise avoiding setting a priority value unless absolutely necessary.

## Custom event objects

As noted earlier, an `Event` instance is created when you call either
`trigger()` or `triggerUntil()`, using the arguments passed to each;
additionally, you can manually create an instance. Why would you do so, however?

One thing that looks like a code smell is when you have code like this:

```php
$routeMatch = $e->getParam('route-match', false);
if (! $routeMatch) {
    // Oh noes! we cannot do our work! whatever shall we do?!?!?!
}
```

The problems with this are several: 

- Relying on string keys for event parameters is going to very quickly run into
  problems &mdash; typos when setting or retrieving the argument can lead to
  hard to debug situations.
- Second, we now have a documentation issue; how do we document expected
  arguments? how do we document what we're shoving into the event?
- Third, as a side effect, we can't use IDE or editor hinting support &mdash;
  string keys give these tools nothing to work with.

Similarly, consider how you might represent a computational result of a method
when triggering an event. As an example:

```php
// in the method:
$params['__RESULT__'] = $computedResult;
$events->trigger(__FUNCTION__ . '.post', $this, $params);

// in the listener:
$result = $e->getParam('__RESULT__');
if (! $result) {
    // Oh noes! we cannot do our work! whatever shall we do?!?!?!
}
```

Sure, that key may be unique, but it suffers from a lot of the same issues.

The solution is to create *custom event types*. As an example, laminas-mvc defines
a custom `MvcEvent`; this event composes the application instance,
the router, the route match, the request and response instances, the view
model, and also a result. We end up with code like this in our listeners:

```php
$response = $e->getResponse();
$result   = $e->getResult();
if (is_string($result)) {
    $content = $view->render('layout.phtml', ['content' => $result]);
    $response->setContent($content);
}
```

As noted earlier, if using a custom event, you will need to use the
`triggerEvent()` and/or `triggerEventUntil()` methods instead of the normal
`trigger()` and `triggerUntil()`.


## Putting it together: Implementing a caching system

In previous sections, I indicated that short-circuiting is a way to potentially
implement a caching solution. Let's create a full example.

First, let's define a method that could use caching. You'll note that in most of
the examples, we use `__FUNCTION__` as the event name; this is a good practice,
as it makes code completion simpler, maps event names directly to the method
triggering the event, and typically keeps the event names unique.  However, in
the case of a caching example, this might lead to identical events being
triggered, as we will be triggering multiple events from the same method. In
such cases, we recommend adding a semantic suffix: `__FUNCTION__ . 'pre'`,
`__FUNCTION__ . 'post'`, `__FUNCTION__ . 'error'`, etc.  We will use this
convention in the upcoming example.

Additionally, you'll notice that the `$params` passed to the event are usually
the parameters passed to the method. This is because those are often not
stored in the object, and also to ensure the listeners have the exact same
context as the calling method. In the upcoming example, however, we will be
triggering an event using the *results of execution*, and will need a way of
representing that. We have two possibilities:

- Use a "magic" key, such as `__RESULT__`, and add that to our parameter list.
- Create a custom event that allows injecting the result.

The latter is a more correct approach, as it introduces type safety, and
prevents typographical errors. Let's create that event now:

```php
use Laminas\EventManager\Event;

class ExpensiveCallEvent extends Event
{
    private $criteria1;
    private $criteria2;
    private $result;

    public function __construct($target, $criteria1, $criteria2)
    {
        // Set the default event name:
        $this->setName('someExpensiveCall');
        $this->setTarget($target);
        $this->criteria1 = $criteria1;
        $this->criteria2 = $criteria2;
    }

    public function getCriteria1()
    {
        return $this->criteria1;
    }

    public function getCriteria2()
    {
        return $this->criteria2;
    }

    public function setResult(SomeResultClass $result)
    {
        $this->result = $result;
    }

    public function getResult()
    {
        return $this->result;
    }
}
```

We can now create an instance of this within our class method, and use it to
trigger listeners:

```php
public function someExpensiveCall($criteria1, $criteria2)
{
    $event = new ExpensiveCallEvent($this, $criteria1, $criteria2);
    $event->setName(__FUNCTION__ . '.pre');
    $results = $this->getEventManager()->triggerEventUntil(
        function ($r) {
            return ($r instanceof SomeResultClass);
        },
        $event
    );

    if ($results->stopped()) {
        return $results->last();
    }

    // ... do some work ...

    $event->setName(__FUNCTION__ . '.post');
    $event->setResult($calculatedResult);
    $this->events()->triggerEvent($event);
    return $calculatedResult;
}
```

Before triggering either event, we set the event name in the instance to ensure
the correct listeners are notified. The first trigger checks to see if we get a
result class returned, and, if so, we return it. The second trigger is a
fire-and-forget; we don't care what is returned, and only want to notify
listeners of the result.

To provide some caching listeners, we'll need to attach to each of the
`someExpensiveCall.pre` and `someExpensiveCall.post` events. In the former case,
if a cache hit is detected, we return it. In the latter, we store the value in
the cache.

The following listeners attach to the `.pre` and `.post` events triggered by the
above method. We'll assume `$cache` is defined, and is a
[laminas-cache](https://docs.laminas.dev/laminas-cache/) storage adapter. The
first listener will return a result when a cache hit occurs, and the second will
store a result in the cache if one is provided.

```php
$events->attach('someExpensiveCall.pre', function (ExpensiveCallEvent $e) use ($cache) {
    $key = md5(json_encode([
        'criteria1' => $e->getCriteria1(),
        'criteria2' => $e->getCriteria2(),
    ]));

    $result = $cache->getItem($key, $success);

    if (! $success) {
        return;
    }

    $result = new SomeResultClass($result);
    $e->setResult($result);
    return $result;
});

$events->attach('someExpensiveCall.post', function (ExpensiveCallEvent $e) use ($cache) {
    $result = $e->getResult();
    if (! $result instanceof SomeResultClass) {
        return;
    }

    $key = md5(json_encode([
        'criteria1' => $e->getCriteria1(),
        'criteria2' => $e->getCriteria2(),
    ]));

    $cache->setItem($key, $result);
});
```

> ### ListenerAggregates allow stateful listeners
>
> The above could have been done within a `ListenerAggregate`, which would have
> allowed keeping the `$cache` instance as a stateful property, instead of
> importing it into closures.

Another approach would be to move the body of the method to a listener as well,
which would allow using the priority system in order to implement caching.

If we did that, we'd modify the `ExpensiveCallEvent` to omit the `.pre` suffix
on the default event name, and then implement the class that triggers the event
as follows:

```php
public function setEventManager(EventManagerInterface $events)
{
    $this->events = $events;
    $events->setIdentifiers([__CLASS__, get_class($this)]);
    $events->attach('someExpensiveCall', [$this, 'doSomeExpensiveCall']);
}

public function someExpensiveCall($criteria1, $criteria2)
{
    $event = new ExpensiveCallEvent($this, $criteria1, $criteria2);
    $this->getEventManager()->triggerEventUntil(
        function ($r) {
            return $r instanceof SomeResultClass;
        },
        $event
    );
    return $event->getResult();
}

public function doSomeExpensiveCall(ExpensiveCallEvent $e)
{
    // ... do some work ...
    $e->setResult($calculatedResult);
}
```

Note that the `doSomeExpensiveCall` method does not return the result directly;
this allows what was originally our `.post` listener to trigger. You'll also
notice that we return the result from the `Event` instance; this is why the
first listener passes the result into the event, as we can then use it from the
calling method!

We will need to change how we attach the listeners; they will now attach
directly to the `someExpensiveCall` event, without any suffixes; they will also
now use priority in order to intercept before and after the default listener
registered by the class. The first listener will listen at priority `100` to
ensure it executes before the default listener, and the second will listen at
priority `-100` to ensure it triggers after we already have a result:

```php
$events->attach('someExpensiveCall', function (ExpensiveCallEvent $e) use ($cache) {
    // listener for checking against the cache
}, 100);

$events->attach('someExpensiveCall', function (ExpensiveCallEvent $e) use ($cache) {
    // listener for injecting into the cache
}, -100);
```

The workflow ends up being approximately the same, but eliminates the
conditional logic from the original version, and reduces the number of events to
one.

The alternative, of course, is to have the object compose a cache instance and
use it directly. However, the event-based approach
allows:

- Re-using the listeners with multiple events.
- Attaching multiple listeners to the event; as an example, to implement
  argument validation, or to add logging.

The point is that if you design your object with events in mind, you can add
flexibility and extension points without requiring decoration or class
extension.

## Conclusion

laminas-eventmanager is a powerful component. It drives the workflow of laminas-mvc,
and is used in many Laminas components to provide hook points for
developers to manipulate the workflow. It can be a powerful tool in your
development toolbox.
