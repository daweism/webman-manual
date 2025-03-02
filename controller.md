# 控制器


新建控制器文件 `app\controller\Foo.php`。

```php
<?php
namespace app\controller;

use support\Request;

class Foo
{
    public function index(Request $request)
    {
        return response('hello index');
    }
    
    public function hello(Request $request)
    {
        return response('hello webman');
    }
}
```

当访问 `http://127.0.0.1:8787/foo` 时，页面返回 `hello index`。

当访问 `http://127.0.0.1:8787/foo/hello` 时，页面返回 `hello webman`。

当然你可以通过路由配置来更改路由规则，参见[路由](route.md)。

## 说明
 - 框架会自动向控制器传递`support\Request` 对象，通过它可以获取用户输入数据(get post header cookie等数据)，参见[请求](request.md)
 - 控制器里可以返回数字、字符串或者`support\Response` 对象，但是不能返回其它类型的数据。
 - `support\Response` 对象可以通过`response()` `json()` `xml()` `jsonp()` `redirect()`等助手函数创建。
 
 
## 生命周期
 - 控制器仅在被需要的时候才会被实例化。
 - 控制器一旦实例化后遍会常驻内存直到进程销毁。
 - 由于控制器实例常驻内存，所以不会每个请求都会初始化一次控制器。
 
## 控制器钩子 `beforeAction()` `afterAction()`
在传统框架中，每个请求都会实例化一次控制器，所以很多开发者`__construct()`方法中做一些请求前的准备工作。

而webman由于控制器常驻内存，无法在`__construct()`里做这些工作，不过webman提供了更好的解决方案`beforeAction()` `afterAction()`，它不仅让开发者可以介入到请求前的流程中，而且还可以介入到请求后的处理流程中。

为了介入请求流程，我们需要使用[中间件](middleware.md)


1、创建文件 `app\middleware\ActionHook.php`(middleware目录不存在请自行创建)

```php
<?php
namespace app\middleware;

use support\Container;
use Webman\MiddlewareInterface;
use Webman\Http\Response;
use Webman\Http\Request;
class ActionHook implements MiddlewareInterface
{
    public function process(Request $request, callable $next) : Response
    {
        if ($request->controller) {
            // 禁止直接访问beforeAction afterAction
            if ($request->action === 'beforeAction' || $request->action === 'afterAction') {
                return response('<h1>404 Not Found</h1>', 404);
            }
            $controller = Container::get($request->controller);
            if (method_exists($controller, 'beforeAction')) {
                $before_response = call_user_func([$controller, 'beforeAction'], $request);
                if ($before_response instanceof Response) {
                    return $before_response;
                }
            }
            $response = $next($request);
            if (method_exists($controller, 'afterAction')) {
                $after_response = call_user_func([$controller, 'afterAction'], $request, $response);
                if ($after_response instanceof Response) {
                    return $after_response;
                }
            }
            return $response;
        }
        return $next($request);
    }
}
```

2、在 config/middleware.php 中添加如下配置
```php
return [
    '' => [
	    // .... 这里省略了其它配置 ....
        app\middleware\ActionHook::class,
    ]
];
```

3、这样如果 controller包含了 `beforeAction` 或者 `afterAction`方法会在请求发生时自动被调用。
例如：
```php
<?php
namespace app\controller;
use support\Request;
class Index
{
    /**
     * 该方法会在请求前调用 
     */
    public function beforeAction(Request $request)
    {
        echo 'beforeAction';
        // 若果想终止执行Action就直接返回Response对象，不想终止则无需return
        // return response('终止执行Action');
    }

    /**
     * 该方法会在请求后调用
     */
    public function afterAction(Request $request, $response)
    {
        echo 'afterAction';
        // 如果想串改请求结果，可以直接返回一个新的Response对象
        // return response('afterAction'); 
    }

    public function index(Request $request)
    {
        return response('index');
    }
}
```

**`beforeAction`说明：**
 - 在当前控制器被执行前调用
 - 框架会传递一个`Request`对象给`beforeAction`，开发者可以从中获得用户输入
 - 如需终止执行当前控制器，则只需要在`beforeAction`里返回一个`Response`对象，比如`return redirect('/user/login');`
 - 无需终止执行当前控制器时，不要返回任何数据
 
**`afterAction`说明：**
 - 在当前控制器被执行后调用
 - 框架会传递`Request`对象以及`Response`对象给`afterAction`，开发者可以从中获得用户输入以及控制器执行后返回的响应结果
 - 开发者可以通过`$response->rawBody()`获得响应内容
 - 开发者可以通过`$response->getHeader()`获得响应的header头
 - 开发者可以通过`$response->getStatusCode()`获得响应的http状态码
 - 开发者可利用`$response->withBody()` `$response->header()` `$response->withStatus()`串改响应，也可以创建并返回一个新的`Response`对象替代原响应
 
 
 

