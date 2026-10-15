1. PHP代码的strlen对中文字符串的方法。
	PHP代码的strlen方法获取的是字节数，其他的 strpos、 substr 都是针对字节数来计算的。
	 比如
	 ```
	 <?php

		$str = "你好世界";
		
		echo strlen($str);     // 12
		echo mb_strlen($str);  // 4
	 ```
	 所以需要引入 mbstring 扩展来解决这个问题。
	 记住下面这几个函数
	 |函数|作用|
|---|---|
|`mb_strlen()`|计算字符长度|
|`mb_substr()`|截取字符串|
|`mb_strpos()`|查找子字符串位置|
2. PHP 没有安装curl扩展的时候 也可以获取url返回的内容，使用fopen即可。但是在高并发的http请求的情况下 表现一般，一般更推荐使用curl。laravel中的 Guzzle Http Client 也是有限使用 curl。