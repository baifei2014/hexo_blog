---
title: Laravel中外观模式的使用
tags: blog
date: 2019-06-05 15:32:45
---
Laravel是一款非常流行的PHP开发框架，简洁，优雅。Laravel使用到了很多的新特性，并且在框架爱中大量使用设计模式。今天介绍下Laravel中使用到的外观（门面）模式。

## 外观模式
外观模式是指外部与一个子系统的通信必须通过一个统一的外观对象进行，为子系统中的一组接口提供一个一致的界面。外观模式定义了一个高层接口，这个接口使得这一子系统更加容易使用。外观模式是一种对象结构型模式。

> Laravel中整个成员变量和函数都是静态属性，外观类没有实例化，不过为了描述更形象，我们还是统称为外观对象

## Laravel的外观对象
Laravel中内含了多个模块，为每个模块实现便捷的引用，这就是外观模式的作用，在Laravel中，所有的外观类都继承自`Illuminate\Support\Facades\Facade`，通过外观类，统一去进行各个子模块的实例化调用。

## 外观对象的初始化
外观对象会在Laravel实例化启动后进行装载，代码很简单：
``` php
$app = new Laravel\Lumen\Application(
    realpath(__DIR__ . '/../')
);
$app->withFacades();
```
Application类实例化之后，调用了`withFacades`方法，这步操作将外观对象正式引入了Laravel应用，我们看下`Laravel\Lumen\Application`中`withFacades`包含了哪些操作：
``` php
class Application extends Container
{
    public function withFacades($aliases = true, $userAliases = [])
    {
        Facade::setFacadeApplication($this);
    }
}
```
在函数内，将`$this`即Laravel应用实例做参数传给了`Illuminate\Support\Facades\Facade`的`setFacadeApplication`函数，要想知道究竟发生了什么，还是接着往下探索：
``` php
abstract class Facade
{
    /**
     * The application instance being facaded.
     *
     * @var \Illuminate\Contracts\Foundation\Application
     */
    protected static $app;

    /**
     * The resolved object instances.
     *
     * @var array
     */
    protected static $resolvedInstance;
    /**
     * Set the application instance.
     *
     * @param  \Illuminate\Contracts\Foundation\Application  $app
     * @return void
     */
    public static function setFacadeApplication($app)
    {
        static::$app = $app;
    }
}
```
在这里我们已经看出一些脉络了，Laravel启动后，引入外观对象，并将实例化对应传给外观对象的静态属性`$app`，自此，成功将应用实例与外观对象做了绑定。这里外观对象两个成员变量，`$app`保存的是应用实例化对象，`$resolvedInstance`保存的是各个子模块实例对象，以key/value形式存在array数组里，这个变量的功能会在下面说到。

## 外观对象的使用
外观对象的使用在Laravel中非常普遍，这里以Redis为例做下介绍，下面是Redis外观模式类：
``` php
class Redis extends Facade
{
    /**
     * Get the registered name of the component.
     *
     * @return string
     */
    protected static function getFacadeAccessor()
    {
        return 'redis';
    }
}
```
它继承于外观对象，重写了`getFacadeAccessor`方法，返回一个redis字符串，非常简单是不是，没错，外观模式就是这样，当我们使用时，直接引入这个命名空间，直接就可以使用Redis进行操作了，通常是这样使用的：
``` php
use Illuminate\Support\Facades\Redis;
class RedisTest
{
    
    public function getSomeThing()
    {
        Redis::get('key');
    }
}
```
在使用中我们不用了解Redis扩展源码，并且在调用时也不用手动实例化，优雅的使用Redis，接下来我们就讲讲Laravel框架是如何实现优雅的调用的。
我们知道PHP有一些魔术方法，当调用类中一个不存在的方法时，就会调用`__callStatic`魔术方法。在这里，Redis调用get方法，外观对象没有实现任何get方法，就会调用php魔术方法`__callStatic`，通过魔术方法获取实例，并调用方法，代码实现是这样的：
``` php
public static function __callStatic($method, $args)
{
    $instance = static::getFacadeRoot();

    if (! $instance) {
        throw new RuntimeException('A facade root has not been set.');
    }

    return $instance->$method(...$args);
}
public static function getFacadeRoot()
{
    return static::resolveFacadeInstance(static::getFacadeAccessor());
}
protected static function resolveFacadeInstance($name)
{
    if (is_object($name)) {
        return $name;
    }

    if (isset(static::$resolvedInstance[$name])) {
        return static::$resolvedInstance[$name];
    }

    return static::$resolvedInstance[$name] = static::$app[$name];
}
```
在魔术方法里，首先调用`getFacadeRoot`方法获取对应实例，在`getFacadeRoot`方法里，可以看出，这里调用的`getFacadeAccessor`方法，就是在Redis外观类里的成员方法，返回的是一个字符串`redis`，获取到返回值后，再接着调用`resolveFacadeInstance`方法获取实例，这里会判断，如果传进来的参数是个对象，就直接返回了，不是就接着往下走，会判断这个键值在`$resolvedInstance`里是否存在，如果存在，返回对象，如果不存在就基于容器获取实例，这里就涉及框架的DI容器的概念了，这里暂不做延伸，只讨论通过容器解决依赖获取对象后，将获取到的对象存入`$resolvedInstance`实例池中，方便下次取用，并且返回获取到的对象，再然后，就能在魔术方法里成功调用子模块对象的方法了，整个实现过程就是这样的。

## 小结
在外观模式中，外部与子系统的交互必须通过一个统一的对象进行，这样使得使用方只要了解外观对象，而不用深入了解子系统内部实现，优点是对外屏蔽子系统组件，实现了松耦合，缺点在于增加子系统需要修改外观类或调用方代码。









