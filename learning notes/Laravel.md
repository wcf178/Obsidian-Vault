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
$this->app->bind(ProductPublishInterface::class, WalmartPublishService::class)
```
### 2.2. 如何访问(解析)容器中的实例? 

1. 使用make方法
```php
	use App\Service\OrderPaymentService
	$paymentService = $this->app->make(OrderPaymentService:class)
```
1. 自动注入



