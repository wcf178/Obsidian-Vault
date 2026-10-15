
1. composer  install 和 composer update 有什么区别？
	 composer install 会按照composer.lock 文件中的依赖包版本下载软件包。
	 而 composer update 则会根据composer.json 文件中的包的版本约束解析依赖然后下载包，并更新composer.lock.
	 因此在线上环境部署通常使用composer install，而不是随意执行composer update
	 
2. PHP的自动加载机制