---
title: Laravel中的错误处理的理解
tags: [PHP, Laravel, ErrorHandler]
date: 2019-04-25 15:32:45
---
# Laravel中的错误处理的理解

`Laravel`  `依赖注入`


## 前言
在PHP5中，错误处理使用起来非常不方便，遇到错误，便会终止脚本运行，整个服务变得不可用，可能在线上会暴露出非常low的页面给用户，在PHP7中，PHP很好的解决了这个问题，通过 `Error` 和 `Exception` 两个错误和异常基类，我们可以非常方便的处理大部分程序运行中出现的问题，并且可以通过扩展 `Exception` 类自定义异常处理。

**本文中所展示的代码均为文章讨论部分的关键代码。**

## Lumen中错误处理
```flow
st=>start: index.php
op=>operation: 实例化Application
registerHandler=>operation: 绑定异常处理器
run=>operation: 执行逻辑
cond=>condition: 抛出异常Yes or No?
running=>operation: 继续运行
response=>operation: 格式化输出
e=>end: End


st->op->registerHandler->run->cond->running->response->e
cond(yes)->running
cond(no)->response
```
### 入口文件 index.php
熟悉一个框架，除了阅读文档，另一个重要的一点就是阅读源码的实现，首先来看入口文件index.php
```php
// index.php
$app = new Laravel\Lumen\Application(
    realpath(__DIR__ . '/../')
);

$app->singleton(
    Illuminate\Contracts\Debug\ExceptionHandler::class,
    App\Exceptions\Handler::class
);
```
在入口文件里，这两处是非常重要的操作：

> * 实例化Application类，启动应用
> * 利用依赖注入容器绑定自定义错误处理器

这里声明了错误处理器的名字为 `Illuminate\Contracts\Debug\ExceptionHandler` ，现在还看不出来这样操作的好处，后面会看出这样设计的妙处。

### 启动应用 Application类的实例化操作

实例化的对象，就是应用运行的基类，贯穿Lumen应用的整个生命周期，这时，我们就该好奇了，到底实例化时进行了什么操作，才赋予了Lumen应用这么强大的功能呢，我们从代码看操作了啥。
```php
// Laravel\Lumen\Application
class Application extends Container
{
    use Concerns\RoutesRequests,
        Concerns\RegistersExceptionHandlers;
    public function __construct($basePath = null)
    {
        if (! empty(env('APP_TIMEZONE'))) {
            date_default_timezone_set(env('APP_TIMEZONE', 'UTC'));
        }

        $this->basePath = $basePath;

        $this->bootstrapContainer();
        $this->registerErrorHandling();
        $this->bootstrapRouter();
    }
}
```

通过看源码可以知道，Lumen应用继承了容器，这是它绑定错误处理程序的基础，然后使用PHP的代码复用机制trait引入了`RegistersExceptionHandlers`（trait的好处这里不展开描述了），这样构造函数里可直接使用`RegistersExceptionHandlers` 里的方法 `registerErrorHandling`
，具体调用这个函数进行了哪些操作，我们接着看代码。

### 设置自定义错误处理函数

```php
// Laravel\Lumen\Concerns\RegistersExceptionHandlers
protected function registerErrorHandling()
{
    error_reporting(-1);

    set_error_handler(function ($level, $message, $file = '', $line = 0) {
        if (error_reporting() & $level) {
            throw new ErrorException($message, 0, $level, $file, $line);
        }
    });

    set_exception_handler(function ($e) {
        $this->handleUncaughtException($e);
    });

    register_shutdown_function(function () {
        $this->handleShutdown();
    });
}
```
在registerErrorHandling函数里，首先通过 `error_reporting()` 函数，尽可能显示PHP所有错误，这是Laravel，也是Lumen的一个设计原则，解决完程序中所有的异常，尽可能的不让程序中出现一些可能会出现错误的地方，然后通过`set_error_handler()`函数，声明了一个自定义的错误处理匿名函数，当运行出现错误时，统一抛出`ErrorException`，即转换成异常。不过注意一点，自定义错误处理函数不能捕获以下级别的错误：E_ERROR、 E_PARSE、 E_CORE_ERROR、 E_CORE_WARNING、 E_COMPILE_ERROR、 E_COMPILE_WARNING。
接着往下看，代码中又用`set_exception_handler()`自定义了一个匿名的错误处理函数，这是怎么回事呢？我们看PHP手册上的介绍:

> PHP 7 改变了大多数错误的报告方式。不同于传统（PHP 5）的错误报告机制，现在大多数错误被作为 Error 异常抛出。这种 Error 异常可以像 Exception 异常一样被第一个匹配的 try / catch 块所捕获。如果没有匹配的 catch 块，则调用异常处理函数（事先通过 set_exception_handler() 注册）进行处理。 如果尚未注册异常处理函数，则按照传统方式处理：被报告为一个致命错误（Fatal Error）。 

> Error 类并非继承自 Exception 类，所以不能用 catch (Exception e) { ... } 来捕获 Error。你可以用 catch (Error $e) { ... }，或者通过注册异常处理函数（ set_exception_handler()）来捕获 Error。

所以对于异常类型的错误，我们通过`set_exception_handler()`函数来自定义异常处理函数处理，到这一步，又调用了`handleUncaughtException()`函数，我们接着往下看,
```php
// Laravel\Lumen\Concerns\RegistersExceptionHandlers
protected function handleUncaughtException($e)
{
    $handler = $this->resolveExceptionHandler();

    if ($e instanceof Error) {
        $e = new FatalThrowableError($e);
    }

    $handler->report($e);

    if ($this->runningInConsole()) {
        $handler->renderForConsole(new ConsoleOutput, $e);
    } else {
        $handler->render($this->make('request'), $e)->send();
    }
}
    
protected function resolveExceptionHandler()
{
    if ($this->bound('Illuminate\Contracts\Debug\ExceptionHandler')) {
        return $this->make('Illuminate\Contracts\Debug\ExceptionHandler');
    } else {
        return $this->make('Laravel\Lumen\Exceptions\Handler');
    }
}
```
我们看到，在`handleUncaughtException()`函数里，首先调用了 `resolveExceptionHandler()` 函数，在 `resolveExceptionHandler()` 函数里，通过容器功能判断是否有名为`Illuminate\Contracts\Debug\ExceptionHandler` 的实例，而在一开始，Lumen启动时，已经注册了名为`Illuminate\Contracts\Debug\ExceptionHandler`的`App\Exceptions\Handler`类，即我们自定义的错误处理类，返回后，最后执行的render()函数，也就是我们自定义的`render`函数，看到这里，最终拨云见雾，整个错误处理代码的执行流程一目了然。通过一开始时的操作容器实例化`App\Exceptions\Handler`，然后在最后通过容器获取到实例化后的对象，整个依赖注入，单例模式都在代码中得到了很好的使用。

### 小结
Lumen首先通过`set_error_handler()`函数自定义错误处理函数捕获到了错误，统一抛出`ErrorException`异常，然后又通过`set_exception_handler()`自定义异常处理函数，处理程序运行中出现的异常。这样，在PHP运行声明周期内的大部分错误都已经能够成功捕获到了。最后，在加上`register_shutdown_function()`函数的功能，在这三种函数的配合下，所有的错误及异常都交给了Lumen框架自定义或用户自定义的错误处理器来处理了。这里我们也可以看出，虽然继承`Error`和`Exception`都可以抛出异常错误，但是在程序中错误最后还是交给异常处理程序来执行，所以，一般我们实现`Exception`就行了。









