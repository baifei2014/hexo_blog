---
layout: post
title: 从一个路由请求讲laravel的依赖注入
tags: [PHP, Laravel, 依赖注入]
date: 2019-06-04 12:52:18
---
框架不仅是一个工具，也是一种学习资料，通过学习成熟的开发框架，开发者可以更加深入的了解一些设计模式的应用。不管是以后的开发还是自己实现一些工具时，都有很好的促进作用。
这篇文章就以最基本的请求开始，以点带面，讲讲laravel中依赖注入的实现。
在介绍内容前，首先我们先来了解下几个概念：
依赖倒置原则：这是在软件开发中经常提到的几大设计原则之一，核心思想就是上层定义接口，下层实现这个接口，让下层依赖上层，降低耦合度，总的来说就是针对接口编程，而不是针对实现编程。
控制反转：控制反转也是一个设计原则，传统的依赖对象都是程序主动实例化去获取，而控制反转就是将依赖的对象交给容器控制，程序被动的接口依赖对象，依赖对象的获取反转了。
依赖注入：依赖注入是控制反转的一种实现，将类内的实例化转移到类外容器实现，不用在类内直接new依赖单元，由容器注入依赖单元。

在laravel中，框架已经帮助我们屏蔽掉了很多底层的实现，我们只需要按照文档开发就行了，今天我们讨论的是底层实现，那就要深入底层源码探讨实现了，首先从路由讲起，我们知道，网络交互的逻辑就是输入输出，接受标准输入，返回标准输出，访问laravel框架时，laravel是如何解析请求的呢？
这里我们以一个例子来说，首先配置一个接口路由，配置如下：
```php
$router->group(['prefix' => 'goods'], function () use ($router) {
    $router->get(
        'test',
        'GoodsController@test'
    );
});
```
然后请求`goods/test`接口，看一系列的请求操作。
引入路由模块
框架没办法直接识别请求的路由信息，必然会包括一个路由模块，根据路由及请求参数执行对应的PHP文件，下面是框架启动代码：
```php
$app = new Laravel\Lumen\Application(
    realpath(__DIR__ . '/../')
);
$app->router->group([
    'namespace' => 'App\Http\Controllers',
], function ($router) {
    require __DIR__ . '/../routes/web.php';
});
```
在这里对应用进行实例化，标志着应用开始一步步加载配置文件，但在应用实例化后，直接就可以使用router属性了，那秘密肯定就隐藏在应用实例化时的操作，接着看Application应用代码：
```php
public function __construct($basePath = null)
{
    $this->bootstrapRouter();
}
public function bootstrapRouter()
{
    $this->router = new Router($this);
}
```
代码很简单，在构造函数里，调用路由启动函数，实例化路由对象后赋给router属性，这样应用实例成功绑定路由对象，所以就解释通了，实例绑定路由对象后，操作引入了路由文件，通过group函数及后面的调用，路由被处理成了一定规则的数据结构，比如说我们配置的路由信息，就被处理成了：
```php
$this->routes['GET/goods/test'] = [
    'method' => 'get',
    'uri' => '/goods/test',
    'action' => ['uses' => 'App\Http\Controllers\GoodsController@test']
]
```
这里路由信息加载初始化已经完成了，
请求处理
在上一步讲了应用启动及加载配置及注册路由信息，那么请求是如何处理的呢，这里我们以一个例子说明，这里我们使用的例子信息如下：
```php
$router->group(['prefix' => 'goods'], function () use ($router) {
    $router->get(
        'test',
        'GoodsController@test'
    );
}
```

这里我们定义了一个接口goods/test，通过这个接口看看请求处理。
根据请求路由获取注册路由信息
应用启动，标志着应用已经开始处理请求，我们就从run方法开始了解请求处理，可以注意到应用类里并没有run方法，run方法是Laravel\Lumen\Concerns\RoutesRequests里实现的，RoutesRequests是一个trait，Application通过use将它引入，这样，就可以像调用类方法一样调用RoutesRequests里的方法了，下面是RoutesRequests里处理请求相关代码：
```php
public function run($request = null)
{
    // 分发请求
    $response = $this->dispatch($request);

    if ($response instanceof SymfonyResponse) {
        $response->send();
    } else {
        echo (string) $response;
    }

    if (count($this->middleware) > 0) {
        $this->callTerminableMiddleware($response);
    }
}
public function dispatch($request = null)
{
    list($method, $pathInfo) = $this->parseIncomingRequest($request);

    try {
        return $this->sendThroughPipeline($this->middleware, function () use ($method, $pathInfo) {
            if (isset($this->router->getRoutes()[$method.$pathInfo])) {
                // 根据请求路由获取对应的控制器映射
                return $this->handleFoundRoute([true, $this->router->getRoutes()[$method.$pathInfo]['action'], []]);
            }

            return $this->handleDispatcherResponse(
                $this->createDispatcher()->dispatch($method, $pathInfo)
            );
        });
    } catch (Exception $e) {
        return $this->prepareResponse($this->sendExceptionToHandler($e));
    } catch (Throwable $e) {
        return $this->prepareResponse($this->sendExceptionToHandler($e));
    }
}
```

在`dispatch`函数里，首先获取请求类型和请求路由，比如一个GET类型接口goods/test，那method类型就是GET，pathInfo就是`/goods/test`，然后根据前面应用加载时注册的路由信息，获取对应的控制器及函数，比如说，goods/test接口对应的就是`App\Http\Controllers\GoodsController@test`，走到这里，已经清楚获取接口对应的是哪个控制器了，就可以解决相关依赖了，

解决依赖
在上一步根据请求路由信息获取注册路由绑定信息，信息如下：
```php
Array
(
    [0] => 1
    [1] => Array
        (
            [uses] => App\Http\Controllers\GoodsController@test
        )

    [2] => Array
        (
        )
)
```
下面代码是相关调用函数：
```php
protected function handleFoundRoute($routeInfo)
{
    return $this->prepareResponse(
        $this->callActionOnArrayBasedRoute($routeInfo)
    );
}
protected function callActionOnArrayBasedRoute($routeInfo)
{
    $action = $routeInfo[1];

    if (isset($action['uses'])) {
        return $this->prepareResponse($this->callControllerAction($routeInfo));
    }
}
protected function callControllerAction($routeInfo)
{
    $uses = $routeInfo[1]['uses'];

    if (is_string($uses) && ! Str::contains($uses, '@')) {
        $uses .= '@__invoke';
    }

    list($controller, $method) = explode('@', $uses);
    // 使用容器解决依赖
    if (! method_exists($instance = $this->make($controller), $method)) {
        throw new NotFoundHttpException;
    }

    if ($instance instanceof LumenController) {
        return $this->callLumenController($instance, $method, $routeInfo);
    } else {
        return $this->callControllerCallable(
            [$instance, $method], $routeInfo[2]
        );
    }
}
```
解决依赖
在callControllerAction函数里，可以看出，对参数信息处理后，controller变量赋值是App\Http\Controllers\GoodsController，method变量是test，$this->make($controller)这步是什么操作的，这里就涉及到了laravel的依赖注入，先来看下laravel应用类的继承关系，代码如下：
```php
use Illuminate\Container\Container;
class Application extends Container
{
}
```
laravel应用类继承自DI容器，整个应用就是基于依赖注入实现的，所以在make的时候，容器负责应用实例化，解决依赖关系，接下来看下依赖注入调用栈，代码如下：
```php
public function make($abstract, array $parameters = [])
{
    return $this->resolve($abstract, $parameters);
}
```


对象的实例化过程比较复杂，因为要一步步解决所有涉及到的依赖项，类的实例化通过调用Illuminate\Container\Container::build()函数创建，代码如下：
```php
public function build($concrete)
    {
        // 如果是一个匿名函数，直接执行并返回结果
        if ($concrete instanceof Closure) {
            return $concrete($this, $this->getLastParameterOverride());
        }

        $reflector = new ReflectionClass($concrete);

        // 如果不能实例化，调用函数处理错误提示，抛出错误
        if (! $reflector->isInstantiable()) {
            return $this->notInstantiable($concrete);
        }

        $this->buildStack[] = $concrete;

        $constructor = $reflector->getConstructor();

        // 如果没有构造函数，直接实例化返回
        if (is_null($constructor)) {
            array_pop($this->buildStack);

            return new $concrete;
        }

        $dependencies = $constructor->getParameters();

        // 构造函数不为空时，用反射创建这个实例，并注入创建的依赖项
        $instances = $this->resolveDependencies(
            $dependencies
        );

        array_pop($this->buildStack);

        return $reflector->newInstanceArgs($instances);
    }
```


这里我们看到了类的实例是如何创建的，那依赖是如何解决的？，解决依赖resolveDependencies函数代码如下：
```php
protected function resolveDependencies(array $dependencies)
{
    $results = [];

    foreach ($dependencies as $dependency) {
        // If this dependency has a override for this particular build we will use
        // that instead as the value. Otherwise, we will continue with this run
        // of resolutions and let reflection attempt to determine the result.
        if ($this->hasParameterOverride($dependency)) {
            $results[] = $this->getParameterOverride($dependency);

            continue;
        }
        
        // 如果类名是null，那就可能是一个字符串或其他类型，否则就获取对应的实例
        // If the class is null, it means the dependency is a string or some other
        // primitive type which we can not resolve since it is not a class and
        // we will just bomb out with an error since we have no-where to go.
        $results[] = is_null($dependency->getClass())
                        ? $this->resolvePrimitive($dependency)
                        : $this->resolveClass($dependency);
    }

    return $results;
}
protected function resolveClass(ReflectionParameter $parameter)
{
    try {
        return $this->make($parameter->getClass()->name);
    }
}
```

在解决依赖这里我们可以看出，如果依赖项是一个类，调用resolveClass解决时，又回到了调用make方法创建实例，整个就是一个递归过程，使用递归的方式解决依赖项，所以如果a依赖b，b依赖a，这样的代码写出来，程序肯定会崩溃的。整个流程是下图这样的。

{% asset_img ioc.png 依赖注入 %}
这样利用PHP反射机制，成功的解决了所有的依赖单元，创建实例并成功返回，这样就成功实现了依赖注入，不用在程序中一遍遍写new 一个对象，将对象的实例化通过容器去创建，并且自动解决了依赖项，实现了依赖注入。














































