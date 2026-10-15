SPL是 Standard PHP Libraray 的一部分，它提供了一套自动加载类的机制，主要提供了下面这组方法。
- spl_autoload_register(): 注册自动加载函数
- spl_autoload_unregister():移除自动加载函数
- spl_autoload_functions():查看已注册的自动加载函数
- spl_autoload_call():手动触发自动加载
- spl_autoload_extensions():设置默认自动加载的文件扩展名