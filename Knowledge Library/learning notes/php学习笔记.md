# PHP学习笔记

## 一、类型

### 类型系统

- 基本类型

  - 内置类型

    - null类型
    - 标量类型
      - int
      - float
      - string
      - bool

    - array类型

    - resource类型
    - object类型
    - never类型
    - void类型
    - 相对类类型（self，parent，static）

  - 字面量类型
    - false
    - true
  - 用户自定义类型
    - 类
    - 接口
    - 枚举
  - callable类型

- 复合类型

  - 交集类型
  - 联合类型

#### 1. NULL类型

未定义和unset（）的变量都将解析为null

#### 2. Boolean类型

1. bool类型使用true和false两个值，不区分大小写
2. 被认为是false的值有
   - false本身
   - 整数0
   - 浮点数0.0和-0.0
   - 空字符串“”和字符串“0”
   - 不包括任何元素的数组
   - NULL类型
   - 内部对象被强制转换重载为bool。（例如：由不带属性的空元素创建的SimpleXML对象。）

#### 4. int类型

1. 要使用八进制表达，数字前必须加上 `0`（零）。 PHP 8.1.0 起，八进制表达也可以在前面加上 `0o` 或者 `0O` 。 要使用十六进制表达，数字前必须加上 `0x`。要使用二进制表达，数字前必须加上 `0b`。

   从 PHP 7.4.0 开始，整型数值可能会包含下划线 (`_`)，为了更好的阅读体验，这些下划线在展示的时候，会被 PHP 过滤掉。

   ```php
   <?php
   $a = 1234; // 十进制数
   $a = 0123; // 八进制数 (等于十进制 83)
   $a = 0o123; // 八进制数 (PHP 8.1.0 起)
   $a = 0x1A; // 十六进制数 (等于十进制 26)
   $a = 0b11111111; // 二进制数字 (等于十进制 255)
   $a = 1_234_567; // 整型数值 (PHP 7.4.0 以后)
   ?>
   ```

2. 整数溢出。整数溢出的结果就是将溢出的类型自动转化为float类型。

3. 转换为int。

   - bool类型转化为int时，false为0，true为1；
   - float类型转化为int时，会向下取整；（注意<font color=red>：如果转化后也超过了int的取值范围，会将此变量标记为未定义，且不会有任何提示</font>）
   - string转化为int时，如果string是数字字符串或者前导数字，则直接输出对应的int
   - NULL类型转化为int时，值为0
   - resource类型转化为int时，会将其转化为php为其分配的唯一资源号

#### 5. float类型

1. 浮点数的定义

```php
<?php
$a = 1.234; 
$b = 1.2e3; 
$c = 7E-10;
$d = 1_234.567; // 从 PHP 7.4.0 开始支持
?>
```

​	2.浮点数的比较

​	浮点数的精度是不可信的，要对浮点数进行比较需要采用迂回的方式

```php
//方式1.使用精度值
<?php
$a = 1.23456789;
$b = 1.23456780;
$epsilon = 0.00001;

if(abs($a-$b) < $epsilon) {
    echo "true";
}
?>
//方式2.使用round（）
```

#### 6. string类型

1. 字符串的表达方式

   - 单引号‘
   - 双引号“
   - heredoc 语法结构
   - nowdoc语法结构

2. 单引号：

   定义一个字符串最简单的方式就是用‘将其包裹住。如果需要表达单引号（’）需要在之前添加转义字符\，如果要表达\则需要写为\\\\（<font color=red>**注意**: 不像[双引号]和 [heredoc] 语法结构，在单引号字符串中的[变量]和特殊字符的转义序列将*不会*被替换。</font>）

3. 双引号

   在双引号中的转义字符如下

   ![image-20230720114149941](C:\Users\admin\AppData\Roaming\Typora\typora-user-images\image-20230720114149941.png)

   在双引号中，变量会被解析。

4. Heredoc结构

   语法的结构为<<<EOY...EOY,其结构类似双引号。在该运算符之后要提供一个标识符，然后换行。接下来是字符串 string 本身，最后要用前面定义的标识符作为结束标志。结束标识符可以使用空格或制表符（tab）缩进，此时文档字符串会删除所有缩进。 在 PHP 7.3.0 之前的版本中，结束时所引用的标识符*必须*在该行的第一列。

   见如下代码：

   ```php
   <?php
   // 无缩进
   echo <<<END
         a
        b
       c
   \n
   END;
   // 4 空格缩进
   echo <<<END
         a
        b
       c
       END;
   ```

   输出为

   ```
         a
        b
       c
     a
    b
   c
   ```

   也即是说，结束的标识符位置为文档的最左边界。因此<font color=red>标识符的缩进不得超过字符串的内容的任何一行</font>。除此之外，<font color=red>标识符和内容的缩进不能混合制表符和空格</font>。另外还需要遵循以下规则

   - 如果在某一行的开头找到了结束标识符，那么不管它是否是另外一个单词的一部分， 它都可能看作结束标识符并引起 [ParseError]。*不要选择正文内容中出现的词作为结束标识符*。
   - Heredocs结构可以能用来初始化类的属性。
   - 带有结束标识符的行不能包含除 (`;`)外的任何其他字符。 
   - 可以作为函数的参数

5. nowdoc语法结构。

   nowdoc的结构类似于单引号的结构，不会对内容中的变量进行解析。一个 nowdoc 结构也用和 heredocs 结构一样的标记 `<<<`， 但是跟在后面的标识符要用单引号括起来，即 `<<<'EOT'`。Heredoc 结构的所有规则也同样适用于 nowdoc 结构，尤其是结束标识符的规则。

   

6. 变量解析

   - 简单语法

     可以使用 `$变量名`或者 `${变量名}` 直接插入文本中

   - 复杂语法

     可以使用 `{复杂表达式}`将变量插入到文本中

7. 存储或修改字符串中的字符

   - $str[i].超出字符串长度的下标写入将会拉长该字符串并以空格填充。非整数类型下标会被转换成整数
   - 用 `[]` 或 `{}` 访问任何其它类型（不包括数组或具有相应接口的对象实现）的变量只会无声地返回 **`null`**。

8. 字符串的运算

   - 字符串的链接使用”.“来进行连接
   - 字符串操作函数。见[php源码笔记](E:\Code\Parctice\index.php)

   
   
#### 7. array数组类型

   

   1. array数组的定义

      - 使用array（）方法构建
      - 使用短数组语法
       
      ```php
      $array1 = array("key"=>"value","key"=>"value","key"=>"value",...);
      
      $array2 = ["key"=>"value","key"=>"value","key"=>"value",...];
      ```
   
      
   
   2. key和value的类型限制及其规则
   
      
   
      - key的类型可以包括string、int、float、bool类型。value的类型可以是任意类型
       
      - key会有额外的类型转换
       
        - String中包含有效的int数字，除非数字前有➕号，否则会直接转化为对应的int数字
        - float会被舍弃小数部分转化为int
        - bool中true会被转化为1，false会被转化为0
        - null会被转化为空字符串
       
      - 当key键中出现重复的值，新的key=>value会覆盖掉旧的语句
       
      - key为可选项，如果未指定，php将会指定上一个使用过的最大的int＋1做为下一个key值。如下：
       
        数组中4个key的值依次为【0】【1】【9】【10】
       
        ```php
        $array1 = array(
        	1,
        	4,
        	9=>6,
        	8)
        ```
   
   3. 通过[]方括号访问数值元素
   
      - 通过array[key]可以访问对应key的value。如果key不存在，则会抛出异常
      - 可以通过$array[key]或者$array[]修改或增加数组元素（指定键名或者不指定）<font color=red>需要注意的是，如果array不存在将会新建它，这也是一种新建数组的方式，但斌不建议</font>
      - 可以通过unset（）来删除数组元素或者数组，例如 `unset($array[1])`删除key为1的键值对；`unset($array)`删除整个数组
   
   4. 数组解包
   
      在php中数组解包是指将数组中的元素解包成为独立的变量。可以使用list（）或者[]来为其中的变量进行赋值为数组中的对应元素。
   
      
   
      例如 `[$a,$b,$c] = [1,2,3]`,abc会被分别赋值为123，而如果指定key，如 `[2=>$a,0=>$b,1=>$c] = [1,2,3]`,运行的结果abc对应的值为312.<font color=red>注意，不能混合这两种方式例如[2=>$a,$b,$c]=[1,2,3]是不可行的</font>。如果变量未指定例如 `[, , $a]=[1,2,3]` 则会默认从0索引开始复制，$a对应的索引为2，因此$a=3
   
      
   
      此外数组解包还可以用来快捷的交换变量
   
      ```php
      <?php
      $a = 1;
      $b = 2;
      
      [$b, $a] = [$a, $b];
      
      echo $a;    // 打印 2
      echo $b;    // 打印 1
      ?>
      ```
   
      
   
   5. 实用函数
   
      PHP中有许多与数组相关的函数和方法。以下是一些常用的函数和方法及其用法：
   
      1. count() - 返回数组中元素的数量
      
      ```php
      $array = array(1, 2, 3, 4, 5);
      $count = count($array);
      echo $count; // 输出 5
      ```
   
      2. array_push() - 在数组末尾添加一个或多个元素
      
      ```php
      $array = array(1, 2, 3);
      array_push($array, 4, 5);
      print_r($array); // 输出 Array ( [0] => 1 [1] => 2 [2] => 3 [3] => 4 [4] => 5 )
      ```
   
      3. array_pop() - 删除并返回数组中的最后一个元素
      
      ```php
      $array = array(1, 2, 3);
      $last_element = array_pop($array);
      print_r($array); // 输出 Array ( [0] => 1 [1] => 2 )
      echo $last_element; // 输出 3
      ```
   
      4. array_shift() - 删除并返回数组中的第一个元素
      
      ```php
      $array = array(1, 2, 3);
      $first_element = array_shift($array);
      print_r($array); // 输出 Array ( [0] => 2 [1] => 3 )
      echo $first_element; // 输出 1
      ```
   
      5. array_unshift() - 在数组开头插入一个或多个元素
      
      ```php
      $array = array(1, 2, 3);
      array_unshift($array, 0);
      print_r($array); // 输出 Array ( [0] => 0 [1] => 1 [2] => 2 [3] => 3 )
      ```
   
      6. array_slice() - 返回一个数组的一部分
      
      ```php
      $array = array(1, 2, 3, 4, 5);
      $slice = array_slice($array, 2, 2);
      print_r($slice); // 输出 Array ( [0] => 3 [1] => 4 )
      ```
   
      7. array_merge() - 合并一个或多个数组
      
      ```php
      $array1 = array(1, 2, 3);
      $array2 = array(4, 5, 6);
      $merged_array = array_merge($array1, $array2);
      print_r($merged_array); // 输出 Array ( [0] => 1 [1] => 2 [2] => 3 [3] => 4 [4] => 5 [5] => 6 )
      ```
   
      8. array_search() - 在数组中搜索给定的值，如果找到则返回相应的键
      
      ```php
      $array = array('apple', 'banana', 'orange');
      $key = array_search('banana', $array);
      echo $key; // 输出 1
      ```
   
      9. in_array() - 检查给定的值是否在数组中存在
      
      ```php
      $array = array('apple', 'banana', 'orange');
      if (in_array('banana', $array)) {
          echo '找到了';
      } else {
          echo '没找到';
      }
      ```
   
      10. array_key_exists() - 检查给定的键是否在数组中存在
   
      ```php
      $array = array('name' => 'Tom', 'age' => 25, 'gender' => 'male');
      if (array_key_exists('name', $array)) {
          echo '找到了';
      } else {
          echo '没找到';
      }
      ```
   
      11. array_values() - 返回一个数组中所有的值，不包括键
   
      ```php
      $array = array('name' => 'Tom', 'age' => 25, 'gender' => 'male');
      $values = array_values($array);
      print_r($values); // 输出 Array ( [0] => Tom [1] => 25 [2] => male )
      ```
   
      12. array_keys() - 返回一个数组中所有的键，不包括值
   
      ```php
      $array = array('name' => 'Tom', 'age' => 25, 'gender' => 'male');
      $keys = array_keys($array);
      print_r($keys); // 输出 Array ( [0] => name [1] => age [2] => gender )
      ```
   
      13. array_reverse() - 返回一个数组的逆序数组
   
      ```php
      $array = array(1, 2, 3, 4, 5);
      $reversed_array = array_reverse($array);
      print_r($reversed_array); // 输出 Array ( [0] => 5 [1] => 4 [2] => 3 [3] => 2 [4] => 1 )
      ```
   
      14. sort() - 对数组进行升序排序
   
      ```php
      $array = array(3, 1, 4, 2, 5);
      sort($array);
      print_r($array); // 输出 Array ( [0] => 1 [1] => 2 [2] => 3 [3] => 4 [4] => 5 )
      ```
   
      15. rsort() - 对数组进行降序排序
   
      ```php
      $array = array(3, 1, 4, 2, 5);
      rsort($array);
      print_r($array); // 输出 Array ( [0] => 5 [1] => 4 [2] => 3 [3] => 2 [4] => 1 )
      ```
   
      16. array_unique() - 返回一个数组中去重后的值
   
      ```php
      $array = array(1, 2, 2, 3, 3, 3, 4, 4, 4, 4);
      $unique_array = array_unique($array);
      print_r($unique_array); // 输出 Array ( [0] => 1 [1] => 2 [3] => 3 [6] => 4 )
      ```
   
      17. array_map() - 对数组中的每个元素应用回调函数
   
      ```php
      $array = array(1, 2, 3, 4, 5);
      function square($n) {
          return $n * $n;
      }
      $squared_array = array_map('square', $array);
      print_r($squared_array); // 输出 Array ( [0] => 1 [1] => 4 [2] => 9 [3] => 16 [4] => 25 )
      ```
   
      18. array_filter() - 使用回调函数过滤数组中的元素
   
      ```php
      $array = array(1, 2, 3, 4, 5);
      function is_odd($n) {
          return $n % 2 == 1;
      }
      $odd_numbers = array_filter($array, 'is_odd');
      print_r($odd_numbers); // 输出 Array ( [0] => 1 [2] => 3 [4] => 5 )
      ```
   
      19. array_reduce() - 使用回调函数迭代地将数组简化为单个值
   
      ```php
      $array = array(1, 2, 3, 4, 5);
      function sum($carry, $item) {
          return $carry + $item;
      }
      $total = array_reduce($array, 'sum');
      echo $total; // 输出 15
      ```
   
      20. array_intersect() - 返回两个或多个数组的交集
   
      ```php
      $array1 = array(1, 2, 3, 4, 5);
      $array2 = array(3, 4, 5, 6, 7);
      $intersect_array = array_intersect($array1, $array2);
      print_r($intersect_array); // 输出 Array ( [2] => 3 [3] => 4 [4] => 5 )
      ```
   
      这些是PHP中更多常用的数组函数和方法，它们可以帮助您更好地操作和处理数组。请注意，这些函数和方法只是PHP提供的众多数组操作函数和方法的一部分。如果您想了解更多，请查看PHP官方文档。
   
   6. 转换为数组
   
      对于任意 int，float， string，bool 和 resource 类型，如果将一个值转换为 array，将得到一个仅有一个元素的数组，其下标为 0，该元素即为此标量的值。
   
      如果将 object 类型转换为 array，则结果为一个数组，其单元为该对象的属性。键名将为成员变量名，不过有几点例外：整数属性不可访问； 私有变量前会加上类名作前缀；保护变量前会加上一个 '*' 做前缀。这些前缀的前后都各有一个 `NUL` 字节。 未初始化的[类型属性](https://www.php.net/manual/zh/language.oop5.properties.php#language.oop5.properties.typed-properties)将会被丢弃。
   
   7. 解包数组
   
      ```php
      $arr1 = [1, 2, 3];
      $arr2 = [...$arr1]; //[1, 2, 3]
      $arr3 = [0, ...$arr1]; //[0, 1, 2, 3]
      $arr4 = [...$arr1, ...$arr2, 111]; //[1, 2, 3, 1, 2, 3, 111]
      $arr5 = [...$arr1, ...$arr1]; //[1, 2, 3, 1, 2, 3]
      
      ```

#### 8. 对象（Object）

   1. 转化为对象
      
      对象转化为对象，其值并不会改变。
      
      若是array数组转化为对象，则其key转化为对象的属性，value转化为属性值。
      
      对于其他值，都会转化为一个stdClass的内置类，并且可以通过scalar属性访问原本的值。

#### 9. resource类型

​	resource是一种特殊的类型，保存了到外部资源的引用。

#### 10. never类型，void类型

​	这两种仅用于返回类型。

​	never表示函数永不结束，除非调用exit（）或者出现bug

​	void表示函数不返回值。

#### 11. callable类型callback类型



#### 12. Enum枚举

1. 以下是枚举类型的定义与使用

```php
enum Suit
{
    case Hearts;
    case Diamonds;
    case Clubs;
    case Spades;
}

function do_stuff(Suit $s)
{
    // ...
}

do_stuff(Suit::Spades);
```

## 二、变量

### 基础

- 变量的命名方式。

​		变量的命名方式如下，以字符或者下划线开头，后面跟上任意数量的字母数字下划线。尤其注意变量不能以数字开头。

- 变量的值传递和引用传递。

​		值传递是将变量的值传递给另一个变量，但是两个变量的存储地址是不同的，即一个变量改变并不影响另一个变量。

​		引用传递是将变量的存储地址传递给两一个变量名，相当于多加了一个指向改地址的指针。此时对两个变量的操做会相互影响。

### 预定义变量

php中所有的预定义变量

- 超全局变量 — 在全部作用域中始终可用的内置变量
- [$GLOBALS]— 引用全局作用域中可用的全部变量
- [$_SERVER] — 服务器和执行环境信息
- [$_GET]— HTTP GET 变量
- [$_POST]— HTTP POST 变量
- [$_FILES] — HTTP 文件上传变量
- [$_REQUEST]— HTTP Request 变量
- [$_SESSION] — Session 变量
- [$_ENV] — 环境变量
- [$_COOKIE]— HTTP Cookies
- [$php_errormsg])— 前一个错误信息
- [$http_response_header] — HTTP 响应头
- [$argc] — 传递给脚本的参数数目
- [$argv] — 传递给脚本的参数数组

php中常用的预定义变量

1. `$_SERVER`：包含关于当前脚本运行的服务器和执行环境的信息，如当前脚本的文件路径、请求的 URI、HTTP 请求方法等。
2. `$_GET`、`$_POST`、`$_COOKIE`、`$_REQUEST`：分别包含通过 HTTP GET、POST、Cookie 和任何其他方式发送到当前脚本的参数和数据。
3. `$_SESSION`：包含当前会话中的所有变量和它们的值。在使用 `session_start()` 函数启动会话后，可以使用该变量来存储和访问会话数据。
4. `$_FILES`：包含通过 HTTP POST 方法上传到当前脚本的所有文件的信息。该变量是一个关联数组，其中包含了上传文件的名称、类型、大小和临时文件名等信息。
5. `$_ENV`：包含当前脚本所在环境中的所有环境变量。通常在服务器上设置和管理环境变量，例如数据库连接信息、API 密钥等。
6. `$_SERVER['REMOTE_ADDR']`：包含当前请求的客户端 IP 地址。该变量通常用于跟踪或记录访问者的 IP 地址。
7. `$GLOBALS`：包含了当前脚本中定义的所有全局变量。可以使用该变量在函数内部访问和修改全局变量的值。



### 变量范围

变量的生效范围即他的上下文环境中都会生效。

```php
<?php
	$a = 12;
	include "b.inc";//b.inc中也有了$a的变量。
?>
```

- global关键字

​		该关键字定义的变量可以指向全局变量的引用例如

```php
<?php
$a = 1;
$b = 2;

function Sum()
{
    global $a, $b;

    $b = $a + $b;
}

Sum();
echo $b;
?>
```

- 使用$GLOBALS数组

  ```php
  <?php
  $a = 1;
  $b = 2;
  
  function Sum()
  {
     $GLOBAL['b'] = $GLOBAL['a']+$GLOBAL['b'];
  }
  
  Sum();
  echo $b;
  ?>
  ```

- 使用静态变量

​		静态变量当函数执行完毕之后，并不会将其销毁，而是会保存下来。

​		常量表达式可以赋值给静态变量，但是不能赋值为动态表达式如函数。

### 可变变量

可变变量是指变量名是可变的。例如 `$a = "hello";$$a = "world";//该语句等效为$hello=”world"` 

要将可变变量用于数组，必须解决一个模棱两可的问题。这就是当写下 $$a[1] 时，解析器需要知道是想要 $a[1] 作为一个变量呢，还是想要 $$a 作为一个变量并取出该变量中索引为 [1] 的值。解决此问题的语法是，对第一种情况用 ${$a[1]}，对第二种情况用 ${$a}[1]。

直接看例子

```php
<?php
class foo {
    var $bar = 'I am bar.';
    var $arr = array('I am A.', 'I am B.', 'I am C.');
    var $r   = 'I am r.';
}

$foo = new foo();
$bar = 'bar';
$baz = array('foo', 'bar', 'baz', 'quux');
echo $foo->$bar . "\n";
echo $foo->{$baz[1]} . "\n";

$start = 'b';
$end   = 'ar';
echo $foo->{$start . $end} . "\n";

$arr = 'arr';
echo $foo->{$arr[1]} . "\n";

?>
```

## 三、常量

### 常量定义

常量的定义方式有两种。

第一种：`define(PI,3.1415936)`定义一个PI的常量。可以在任何地方定义

第二种：使用const关键字定义。`const PI = 3.1415926 ` 但是只能在类中定义。

### 魔术常量

1. `__LINE__`：表示当前代码行号的常量。
2. `__FILE__`：表示当前文件的完整路径和文件名的常量。
3. `__DIR__`：表示当前文件所在目录的常量。
4. `__FUNCTION__`：表示当前函数名称的常量。
5. `__CLASS__`：表示当前类名称的常量。
6. `__TRAIT__`：表示当前 trait 名称的常量。traits 是 PHP 5.4 中引入的一种代码复用机制。
7. `__METHOD__`：表示当前方法名称的常量。
8. `__NAMESPACE__`：表示当前命名空间名称的常量。
9. `ClassName::class`	完整的类名。

**require** 在出错时产生 **`E_COMPILE_ERROR`** 级别的错误。换句话说将导致脚本中止而 [include](https://www.php.net/manual/zh/function.include.php) 只产生警告（**`E_WARNING`**），脚本会继续运行。






​      

   

   

   

   

   

   

   









