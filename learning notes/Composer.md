
1. composer  install 和 composer update 有什么区别？
	 composer install 会按照composer.lock 文件中的依赖包版本下载软件包。
	 而 composer update 则会根据composer.json 文件中的包的版本约束解析依赖然后下载包，并更新composer.lock.
	 因此在线上环境部署通常使用composer install，而不是随意执行composer update
	 
2. Composer的自动加载机制
	Composer利用PHP的SPL自动加载机制，为项目中所有的类、接口、trait以及部分函数、常量、生成一套 类名 -> 具体文件路径 的规则。这样只需要在系统的入口文件引入 `vendor/autoload.php` ，之后new 某一个类的时候PHP会自动找到并加载对应的文件，不用手写一堆 `require/include`。
3. 