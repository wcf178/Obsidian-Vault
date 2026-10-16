## 1. laravel的请求生命周期
当一个请求通过nginx转发到laravel的入口文件时, index文件首先引入autoload.php文件, 实现类的自动加载.
然后index引入bootstrap/app.php 创建应用程序实例 和 服务容器
然后根据请求类型, 判断是走http内核还是走console内核
假设请求为http请求, 那么请求过来之后http内核再执行请求之前会首先执行 bootstrappers 定义的一些引导程序, 这些引导程序会配置异常处理, 配置日志, 检测应用程序环境等一些执行请求之前需要完成的任务.
然后 http内核还定义了一个 HTTP请求 中间件列表,这些列表会读写 HTTP session 判断程序是否处于维护状态.,校验CSRF令牌.
在正式处理请求之前, 内核的引导操作会为laravel应用程序加载服务提供者. 这些服务提供者是引导框架的数据库 队列 验证器 路由组件用的. 然后将会在所有服务提供者上调用register方法.然后所有的服务提供者注册完成之后,就会对每个service provider 调用boot方法(执行初始化逻辑).
然后request被传递给路由器进行调度. 路由器将请求分发到路由绑定的方法或者控制器中. 如果路由设置有中间件,那么请求会首先经过中间件,进行一些权限校验,登录校验等一些过滤或其他处理.
如果请求通过了中间件 则执行路由方法或者控制器方法. 然后得到的响应会通过[[路由的中间件链路]]返回.
一旦响应通过中间件返回, HTTP的handle方法将响应对象返回给应用程序实例的handleRequest方法, 该方法调用响应的send方法,将响应内容返回给用户的web浏览器.
## 2. Laravel的服务容器
laravel的服务容器简单来说就是存放各类service类的实例. 让你可以在项目中无需到处写new 对象就可以获取到对应service类的实例.
### 2.1. 如何绑定容器? 
绑定容器的方法有很多种. 但是laravel中大部分场景下服务都是通过Service Provider 中的register方法中进行绑定的. 
- 使用bind方法 注册绑定, 将类名和返回实例闭包传递过来
``` php
$this->app->bind(OrderPaymentService::class, function () {
	return  new OrderPaymentService();
})
```
- 使用`singleton`绑定一个单例. 后续调用容器的这个服务只会返回同一个实例
```php
$this->app->singleton(OrderPaymentService::class, function () {
	return  new OrderPaymentService();
})
```
- 使用`scoped`绑定作用域单例,与上一个单例不同的是,在laravel执行一个新的生命周期的时候,实例会被重新刷新.
```php
$this->app->scoped(OrderPaymentService::class, function () {
	return  new OrderPaymentService();
})
```
- 使用instance方法将现有对象绑定到容器
```php
$paymentService = new OrderPaymentService();
$this->app->instance(OrderPaymentService::class, $paymentservice);
```
- 将接口绑定实例. 例如一个接口`ProductPublishInterface` 和 一个实例 `WalmartPublishService`. 可以直接将接口和walmart绑定,这样当类需要`ProductPublishInterface` 的时候,他就会注入 ``WalmartPublishService``
```php
$this->app->bind(ProductPublishInterface::class, WalmartPublishService::class);
```
注意: 调用 `bind()` 只是注册解析规则，并不意味着立即创建对象。 真正需要解析接口时，容器才会按照对应规则创建对象
### 2.2. 如何访问(解析)容器中的实例? 

1. 使用make方法 (可以访问 `ServersProvider` 类的 `$app` 变量时)
```php
	use App\Service\OrderPaymentService
	$paymentService = $this->app->make(OrderPaymentService:class);
```
不在ServiceProvider 类中 或 没有 $app 变量访问权限的时候
```php
	use App\Service\OrderPaymentService
	$paymentService = App::make(OrderPaymentService:class); //Facade
	$paymentService = app(OrderPaymentService:class); //app helper
```
2. 自动注入. 就是在一个类的构造函数中对一个依赖进行类型提示. 当laravel开始创建这个类的实例的时候, laravel注意到构造函数中出现依赖的类 就会先去解析这个依赖的类, 这就触发了容器解析依赖 然后自动创建依赖类对象. 比如订单控制器依赖`OrderPaymentService`.
```php
<?php

namespace App\Http\Controllers;

use App\Services\AppleMusic;

class OrderController extends Controller
{
    /**
     * 创建一个新的控制器实例。
     */
    public function __construct(
        protected OrderPaymentService $service,
    ) {}

    /**
     * 显示给定 ID 的用户。
     */
    public function show(string $id): Podcast
    {
        return $this->service->findPodcast($id);
    }
}
```
> 即便 OrderPaymentService 没有绑定到服务容器中, laravel也会自动解析. 只要OrderPaymentService的构造函数中没有其他的依赖 或者依赖的类可以自动解析,那么就会递归解析依赖 ,最后创建对象 自动注入到Controller. 最经典的无法解析的类就是依赖接口,因为如果没有通过 接口绑定实例 的话,laravel就不知道解析哪个具体的类 (为接口不能直接 `new`)
### 2.4 方法调用和注入 (App::call())
当你需要在其他地方调用某一个类的某一个方法的时候,可以直接使用
App::call(\[类对象,方法名\]) 如果方法名需要某些参数,并且这个参数支持自动解析,那么就可以不用传参. 例如
```php
class AppleMusic
{
    public function getPlays(): int
    {
        return 100;
    }
}

class PodcastStats
{
    public function generate(AppleMusic $apple): array
    {
        return [
            'plays' => $apple->getPlays()
        ];
    }
}
```

这个时候我调用 `App::call([new PodcastStats, 'generate'])`  服务容器会直接解析AppleMusic类并自动创建 对象.然后调用 generate($apple)

大致的流程如图 [[AppCall方法调用过程]]

### 2.3. 容器解析事件
可以通过 `resolving` 监听每一次服务容器解析对象的事件
```php
$this->app->resolving(OrderPaymentService::class,function(OrderPaymentService $orderpaymentservice,Application $app) {
// 当容器解析OrderPaymentService类的时候触发
})
$this->app->resolving(function (mixed $object, Application $app) { 
// 当容器解析任何类型的对象时调用... 
});
```



