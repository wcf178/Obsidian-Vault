## 1. laravel的请求生命周期
当一个请求通过nginx转发到laravel的入口文件时, index文件首先引入autoload.php文件, 实现类的自动加载.
然后index引入bootstrap/app.php 创建应用程序实例 和 服务容器
然后根据请求类型, 判断是走http内核还是走console内核
假设请求为http请求, 那么请求过来之后http内核再执行请求之前会首先执行 bootstrappers 定义的一些引导程序, 这些引导程序会配置异常处理, 配置日志, 检测应用程序环境等一些执行请求之前需要完成的任务.
然后 http内核还定义了一个 HTTP请求 中间件列表,这些列表会读写 HTTP session 判断程序是否处于维护状态.,校验CSRF令牌.
在正式处理请求之前, 内核的引导操作会为laravel应用程序加载服务提供者. 这些服务提供者是引导框架的数据库 队列 验证器 路由组件用的.
然后请求正式进入 HTTP内核的 handle方法. 接收request,返回response


