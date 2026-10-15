# Composer

## 1. `composer install` 和 `composer update` 有什么区别？

- `composer install` 会按照 `composer.lock` 文件中的依赖包版本下载软件包。
- 而 `composer update` 则会根据 `composer.json` 文件中的包的版本约束解析依赖，然后下载包，并更新 `composer.lock`。

> 因此，在线上环境部署通常使用 `composer install`，而不是随意执行 `composer update`。

## 2. Composer 的自动加载机制

Composer 利用 PHP 的 [[SPL自动加载机制]]，为项目中所有的类、接口、trait 以及部分函数、常量，生成一套“类名 → 具体文件路径”的规则。

这样只需要在系统的入口文件引入 `vendor/autoload.php`，之后 `new` 某一个类时，PHP 会自动找到并加载对应的文件，不用手写一堆 `require/include`。

### 2.1 Composer 做了什么优化？

它首先生成了一个高效的 `ClassLoader` 类，通过 `spl_autoload_register()` 将这个类放到 PHP 的自动加载栈里。当代码使用一个还没有加载的类时，PHP 触发自动加载。Composer 的 `ClassLoader` 根据配置找到文件，并 `include` 进来。

### 2.2 常见的自动加载规则

[[常见的自动加载规则]]：

- PSR-4
- classmap
- file
- PSR-0（罕见）

### 2.3 加载流程（简化版）

1. 入口文件 `require vendor/autoload.php`。
2. Compser 创建 `ClassLoader` 并使用 SPL 加载到自动加载栈。
3. 代码执行到未知的类：`$foo = new App\Bar\Foo();`。
4. 触发自动加载。
5. Composer 生成的 `ClassLoader` 先查 `classmap`，再查 PSR-4，最后查 PSR-0。
6. 找到文件后，`include`。
7. 程序继续。

### 2.4 生产环境中的自动加载优化

生产环境中一般会先优化：执行以下命令：

```bash
composer dump-autoload -o
```

Compseor 会扫描 PSR-4、PSR-0、classmap 指向的目录，解析 PHP 文件中声明的类、trait、接口、enum，然后把它们的完全限定名和文件路径的映射写入 `vendor/composer/autoload_classmap.php` 的静态数组。

## 3.

