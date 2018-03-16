---
title: Laravel Pipeline
date: 2017-09-27 22:49:00
tags:
categories: PHP
---

## Laravel Pipeline源码分析

Laravel是一个设计很优雅的框架，同时他也是一个“调用栈”很深的框架，日常开发的时候我们去看laravel.log里面的报错，就经常会发现调用栈有几十层，单纯看这个调用栈的话会一脸懵逼，不知道那么深的调用栈都是干什么的。下面是一个报错的例子：

```
[2017-09-27 01:17:33] local.ERROR: exception 'PDOException' with message 'SQLSTATE[HY000] [2002] Operation timed out' in /project/vendor/laravel/framework/src/Illuminate/Database/Connectors/Connector.php:55
Stack trace:
#0 /project/vendor/laravel/framework/src/Illuminate/Database/Connectors/Connector.php(55): PDO->__construct('mysql:host=192....', 'user', 'passwd', Array)
#1 /project/vendor/laravel/framework/src/Illuminate/Database/Connectors/MySqlConnector.php(22): Illuminate\Database\Connectors\Connector->createConnection('mysql:host=192....', Array, Array)
...
#20 [internal function]: Illuminate\Pipeline\Pipeline->Illuminate\Pipeline\{closure}(Object(Illuminate\Http\Request))
#21 /project/vendor/laravel/framework/src/Illuminate/Pipeline/Pipeline.php(103): call_user_func(Object(Closure), Object(Illuminate\Http\Request))
#22 /project/vendor/laravel/framework/src/Illuminate/Routing/ControllerDispatcher.php(114): Illuminate\Pipeline\Pipeline->then(Object(Closure))
...
#56 /project/vendor/laravel/framework/src/Illuminate/Pipeline/Pipeline.php(103): call_user_func(Object(Closure), Object(Illuminate\Http\Request))
#57 /project/vendor/laravel/framework/src/Illuminate/Foundation/Http/Kernel.php(122): Illuminate\Pipeline\Pipeline->then(Object(Closure))
#58 /project/vendor/laravel/framework/src/Illuminate/Foundation/Http/Kernel.php(87): Illuminate\Foundation\Http\Kernel->sendRequestThroughRouter(Object(Illuminate\Http\Request))
#59 /project/public/index.php(54): Illuminate\Foundation\Http\Kernel->handle(Object(Illuminate\Http\Request))
#60 {main}
```
大家发现没有，这些调用栈重复出现了很多关键词，比如：Closure，call_user_func，Pipeline，没错，这就是我们今天要讨论的重点。

### Middleware

开始讨论前，我们先了解下laravel框架的Middleware组件，在做一个网站的时候，我们通常要对用户登陆进行校验，对输入参数进行安全注入检测，对api返回结果进行json格式打包，我们都可以用中间件来实现。

* Authenticate Middleware 实现用户登陆检验

```
class Authenticate
{
    ...

    public function handle($request, Closure $next)
    {
        if ($this->auth->guest()) {
            if ($request->ajax()) {
                return response('Unauthorized.', 401);
            } else {
                return redirect()->guest('auth/login');
            }
        }

        return $next($request);
    }
}
```
* FormatJsonOutput Middleware 实现api接口数据格式化输出

```
class FormatJsonOutput
{
	...

    public function handle($request, Closure $next)
    {
        $response = $next($request);

        return Response::JSON([
            'code' => config('custom.success.code'),
            'data' => $response->getOriginalContent()
        ]);
    }
}
```
以上是两个颇具代表性的中间件，Authenticate是前置中间件，FormatJsonOutput是后置中间件，怎么理解前置后置呢？中间件是在什么时候分别怎么被调用的呢？下面你理解了Pipeline的调用流程的话，那么你再回来这个问题就一目了然了。

### 快速了解框架处理请求流程

我们来看Laravel是怎么处理一个请求的，从index.php开始

```
...
//构造http请求处理核心类
$kernel = $app->make(Illuminate\Contracts\Http\Kernel::class);
//处理request
$response = $kernel->handle(
	//捕捉request
    $request = Illuminate\Http\Request::capture()
);
//发送response
$response->send();

$kernel->terminate($request, $response);

```
很简单是不是，再看$kernel（Illuminate\Foundation\Http\Kernel）的handle方法是怎么实现的:

```
public function handle($request)
{
    try {
        $request->enableHttpMethodParameterOverride();

        $response = $this->sendRequestThroughRouter($request);
    } catch (Exception $e) {
        ...
    }
	...

    return $response;
}

protected function sendRequestThroughRouter($request)
{
    $this->app->instance('request', $request);

    Facade::clearResolvedInstance('request');

    $this->bootstrap();
	//敲黑板，看这里，在把请求发送给router处理前，会经历这么一个Pipeline
    return (new Pipeline($this->app))
                ->send($request)
                ->through($this->app->shouldSkipMiddleware() ? [] : $this->middleware)
                ->then($this->dispatchToRouter());
}
```
所以，它是干什么用的？

### Pipeline

顾名思义，pipeline通俗解释就是管道流水线，它的功能是构建一条流水线，依次把需要处理的对象放到流水线上，依次处理后返回。你去翻Pipeline的源代码你会发现，前面三个函数_construct,send,through实现都很简单，最令人费解的是then函数，他接受了一个Closure，然后返回值是一个call_user_func：


```
/**
 * Run the pipeline with a final destination callback.
 *
 * @param  \Closure  $destination
 * @return mixed
 */
public function then(Closure $destination)
{
	//把$destination打包成一个闭包
    $firstSlice = $this->getInitialSlice($destination);

	//调用栈是后进先出，所以这里要翻转数组，保证调用顺序和原始$pipes的顺序一致
    $pipes = array_reverse($this->pipes);

	//调用array_reduce返回的最后一个函数，这个调用会形成文章开头的函数调用栈
    return call_user_func(
        array_reduce($pipes, $this->getSlice(), $firstSlice), $this->passable
    );
}

```
[array_reduce](http://php.net/manual/zh/function.array-reduce.php) 函数是我们理解这段代码的关键，它的作用是把$firstSlice和数组$pipes里的元素当做参数传给$this->getSlice()这个函数依次作用，代码展开如下：

```
//array_reduce($pipes, $this->getSlice(), $firstSlice)等价于调用：
function array_reduce($pipes, $func, $initial)
    foreach($pipes as $v) {
        $initial = $func($initial, $v);
    }
    return $initial;
}

```

要理解调用栈的形成，我们还要继续看$this->getSlice()这个闭包函数到底是什么，简化等价后的函数如下：

```
/**
 * Get a Closure that represents a slice of the application onion.
 *
 * @return \Closure
 */
protected function getSlice()
{
    return function ($stack, $pipe) {
        return function ($passable) use ($stack, $pipe) {
            return call_user_func($pipe, $passable, $stack);
        };
    };
}
```
看到这里可能有的小伙伴懵逼到了极点，这个函数嵌套函数到底是什么鬼？要理解它，我们要学会屏蔽掉细节，getSlice()可以理解为就返回了一个函数，他接收两个参数$stack, $pipe，仅此而已，先记住这个结论。

好，我们再从头捋一遍这个调用流程。new Pipeline()接收了$request参数，然后获取了我们要执行的中间件列表，进入到then处理；then接收了一个目标处理函数A，然后对中间件列表做了个reverse，对于每一个中间件，执行getSlice()函数也就是执行function ($stack, $pipe)，得到的是还是一个闭包函数，array_reduce这个流程我们分解下：

* 第一次执行function ($stack, $pipe)，stack是我们传入的A函数，pipe是第一个中间件middle1（实际上就是函数），得到的是一个闭包B，B就是function ($passable) use (A, middle1)
* 第二次执行function ($stack, $pipe)，stack是第一步的返回结果B，pipe是第二个中间件middle2，得到的是一个闭包C，C就是function ($passable) use (B, middle2)
* 第三次执行function ($stack, $pipe)，stack是第二步的返回结果C，pipe是第三个中间件middle3，得到的是一个闭包D，D就是function ($passable) use (C, middle3)
* ... ...

上述过程实际上就是在形成一个调用栈，array_reduce结束以后得到一个最终函数X，然后被调用call_user_func(X, $this->passable)，实际上就是调用 getSlice（）里面的function ($passable) use ($stack, $pipe)函数，这个函数执行call_user_func($pipe, $passable, $stack)就是调用中间件函数，回到我们开头写的中间件，就是Authenticate->handle($passable, $stack)，中间件里面$stack就是$next，所以Authenticate执行完了会执行$stack，而$stack是啥？看上面的步骤分析知道$stack是上一个步骤的function ($passable) use ($stack, $pipe)。所以，这个调用栈是一层一层回调一直到调用then传入的闭包函数。

前置后置怎么做的？如果你看懂了他们的调用关系，那么前置后置是不是就是自然而然的事情。




