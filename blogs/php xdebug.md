# PHP Xdebug 培训文档

## 一、为什么我们需要 Xdebug？

### 传统调试方式的痛点
在平时的 PHP 开发中，我们最常用的调试方式是什么？
- `var_dump()` / `print_r()`
- `echo`
- `die()` / `exit()`

这些"原始调试法"存在严重问题：
1. **效率低下**：需要反复修改代码、刷新页面
2. **破坏代码结构**：容易忘记删除调试代码，导致线上事故
3. **信息有限**：复杂数据结构输出不直观，无法查看执行流程
4. **无法交互**：不能实时观察变量变化，不能控制执行流程

### Xdebug 带来的变革
Xdebug 让我们能够：
- 像调试 JavaScript 一样调试 PHP
- 设置断点，暂停代码执行
- 单步执行，观察程序流程
- 实时查看和修改变量值
- 分析调用栈和性能瓶颈

## 二、Xdebug 是什么？

### 基本定义
Xdebug 是一个 PHP 扩展，提供完整的调试和分析工具套件。它不是独立软件，而是 PHP 的"外挂"或"插件"。

### 核心功能
1. **堆栈跟踪**：提供完整的函数调用栈信息
2. **断点调试**：在代码任意位置暂停执行
3. **代码覆盖率**：分析测试覆盖的代码行
4. **性能分析**：发现代码性能瓶颈
5. **远程调试**：在开发环境中调试远程代码

## 三、安装 Xdebug

- #### windows 下安装PHP xDebug

  > 如果使用phpStudy管理环境，可以直接勾选xdebug扩展

  1. **检查 PHP 版本**：
     确保你的 PHP 已经安装，并知道其版本号和线程安全（TS/NTS）情况。

     - 使用以下命令查看 PHP 信息：

       ```bash
       php -v
       ```

     - 记录 PHP 的版本号和线程模式（如 `TS` 或 `NTS`）。

  2. **下载 Xdebug 扩展**：

     - 前往 [Xdebug 下载页面](https://xdebug.org/download)。
     - 根据你的 PHP 版本、线程模式（TS/NTS）和架构（x86/x64）下载对应的 Xdebug DLL 文件。

  3. **复制 DLL 文件**：

     - 将下载的 Xdebug DLL 文件复制到你的 PHP 扩展目录（通常位于 `php/ext` 文件夹）。

### 验证安装
```bash
php -v
```
在输出信息中应该能看到 "with Xdebug" 字样。

或者创建 PHP 信息页面：
```php
<?php phpinfo();
```
在页面中搜索 "Xdebug"，确认扩展已加载。

或者在终端输入

```
php -m
```

输出所有的扩展中看到xdebug扩展

## 四、配置 Xdebug

### php.ini主要配置项

> 注意：xdebug版本不同，配置项也不同

修改 `php.ini` 文件，添加以下配置：

- xdebug 3.x

```ini
[xdebug]
; 加载 Xdebug 扩展
zend_extension="path/to/your/xdebug.so"  ; Windows 是 xdebug.dll
; 注意：这里必须是 zend_extension，不是 extension

; 设置调试模式
xdebug.mode = debug

; 设置客户端配置
xdebug.client_host = 127.0.0.1
xdebug.client_port = 9003

; 自动启动调试会话（方便网页调试）


; IDE 密钥
xdebug.idekey = VSCODE

; 其他有用的模式
; 开发模式（增强错误显示）
; xdebug.mode = develop

; 性能分析模式
; xdebug.mode = profile

; 代码覆盖率模式
; xdebug.mode = coverage
```

- xdebug 2.x

```ini
[Xdebug]
; 指定 Xdebug 扩展的 DLL 文件路径，启用 Xdebug 的关键一步
zend_extension=D:/tools/phpstudy_pro/Extensions/php/php7.3.4nts/ext/php_xdebug.dll

; 是否启用远程调试（必须为 1 才能被 IDE 如 VSCode 连接）
xdebug.remote_enable=1

; 是否自动启动调试（不用每次 GET/POST 加 XDEBUG_SESSION_START 参数）
; 开启后，只要 PHP 执行就会尝试连接调试客户端
xdebug.remote_autostart=1

; 调试协议类型（固定用 dbgp，调试客户端与 Xdebug 通信的协议）
xdebug.remote_handler=dbgp

; 远程调试连接到哪个 IP（通常是 localhost）
; 若你调 docker/虚拟机，要设置为宿主机 IP
xdebug.remote_host=127.0.0.1

; 远程调试端口，默认是 9000，但后期被改成 9003（避免与 php-fpm 冲突）
xdebug.remote_port=9003

; 是否让 Xdebug 连接请求的 IP（即 "自动回连"）
; 0 = 不自动，根据 remote_host
; 1 = 自动根据客户端 IP 回连（本地调试一般不用）
xdebug.remote_connect_back=0

; IDE 标识，用来识别连接工具，比如 VSCODE、PHPSTORM
; 这个可有可无，但有些 IDE 会用来过滤连接会话
xdebug.idekey=VSCODE

; 调试连接日志输出，用来排查连不上 IDE 的原因（非常有用！）
xdebug.remote_log="D:/tools/phpstudy_pro/Extensions/php/php7.3.4nts/xdebug_remote.log"

```



### 浏览器配置

#### **方法1：使用浏览器扩展**

- **Chrome**: 安装 "Xdebug helper" 扩展
- 设置IDE key 为 `VSCODE`

#### **方法2：URL参数**

在URL后添加：`?XDEBUG_SESSION=VSCODE`

#### **方法3：Cookie方式**

在浏览器控制台执行：

```javascript
document.cookie = "XDEBUG_SESSION=VSCODE";
```

### 重要提示

1. **端口注意**：php.ini设置的端口需要和vscode中设置的保持一致
2. **模式配置**：`xdebug.mode` 可以同时设置多个模式，用逗号分隔
3. **重启服务**：修改配置后必须重启 Web 服务器或 PHP-FPM

## 五、核心功能：断点调试

### 工作原理
```
浏览器/HTTP请求 <---> 带有Xdebug的PHP <---> 你的IDE
```

当浏览器访问页面时，Xdebug 尝试与配置的 IDE 建立连接，开启调试会话。

![image-20251125205111728](./../../../Code/FBA-DOC/images/php xdebug/image-20251125205111728-1764075099701-1.png)

### VS Code 配置示例

1. **安装 PHP Debug 扩展**

2. **创建调试配置**（.vscode/launch.json）：

   > 如果使用工作区，则需要在工作区配置文件中添加launch项 ，并设置为下面的内容
```json
{
		"version": "0.2.0",
		"configurations": [
			{
				"name": "Listen for Xdebug",
				"type": "php",
				"request": "launch",
				"port": 9003,
				"pathMappings": {
					"/": "${workspaceFolder}"
				},
				"log": true,
				"externalConsole": false,
				"ignore": [
					"**/vendor/**/*.php"
				]
			},
			{
				"name": "Launch currently open script",
				"type": "php",
				"request": "launch",
				"program": "${file}",
				"cwd": "${workspaceFolder}",
				"port": 9003,
				"runtimeArgs": [
					"-dxdebug.remote_enable=1",
					"-dxdebug.remote_autostart=1",
					"-dxdebug.remote_port=9003"
				],
				"env": {
					"XDEBUG_CONFIG": "remote_enable=1 remote_autostart=1 remote_port=9003 idekey=VSCODE"
				}
			}
		]
	}
```

> #### pathMapping参数详解
>
> ##### 1. **核心作用：路径映射**
>
> `pathMappings` 建立了**服务器文件路径**和**本地开发环境文件路径**之间的映射关系。
>
> ##### **为什么需要这个映射？**
>
> **服务器端**（PHPStudy环境）：
>
> ```
> 文件路径：E:\Code\wm-listing.valsun.cn\app\Http\Controllers\Common\AutoAdjustPrice\BaseAutoAdjustPriceController.php
> ```
>
> **调试器端**（VSCode）：
>
> ```
> 文件路径：E:\Code\wm-listing.valsun.cn\app\Http\Controllers\Common\AutoAdjustPrice\BaseAutoAdjustPriceController.php
> ```
>
> 虽然看起来路径相同，但可能存在：
>
> - 大小写差异
> - 斜杠方向差异（/ vs \）
> - 驱动器号表示方式不同
> - 虚拟路径与实际路径差异
>
> ##### 2. **实际工作流程**
>
> 当Xdebug触发断点时：
>
> ```
> 1. 服务器执行到断点 → Xdebug激活
> 2. Xdebug发送文件路径给VSCode："E:/Code/wm-listing.valsun.cn/app/Http/Controllers/..."
> 3. VSCode查找 pathMappings 规则：
>    "E:/Code/wm-listing.valsun.cn" → "${workspaceFolder}"
> 4. VSCode将服务器路径转换为本地路径：
>    "E:/Code/wm-listing.valsun.cn/app/Http/..." → "当前工作空间/app/Http/..."
> 5. VSCode打开本地文件并在断点处暂停
> ```
>
> `pathMappings` 就是**翻译官**，它告诉VSCode：
> "当Xdebug说它在 `服务器路径` 遇到断点时，实际上你应该在 `本地路径` 的对应文件中找到这个位置"
>
> ##### 3.如何确定服务器路径（xdebug传给IED的路径）
>
> 打开xdebugConsole，在launch中配置log：true，找到stackTraceResponse请求，截取path中的对应的文件夹
>
> ```json
> <- stackTraceResponse
> kT {
>   seq: 0,
>   type: 'response',
>   request_seq: 246,
>   command: 'stackTrace',
>   success: true,
>   body: {
>     stackFrames: [
>       {
>         id: 2357,
>         name: 'App\\Http\\Middleware\\PerformanceMiddleware->handle',
>         source: {
>           name: 'PerformanceMiddleware.php',
>           path: 'file:///e%3A/Code/wm-listing.valsun.cn/app/Http/Middleware/PerformanceMiddleware.php'
>         },
>         line: 33,
>         column: 1
>       }
> 	]
>   }
> }
> ```
>
> 

3. **开始调试**：
   - 在代码行号左侧点击设置断点（红色圆点）
   - 按 F5 或点击"运行和调试"启动监听
   - 状态栏变橙色，表示等待连接
   - 在浏览器中访问对应页面

### 调试面板功能详解

当代码在断点处暂停时，你可以使用以下功能：

#### 1. 变量查看 (Variables)
- 查看当前作用域所有变量
- 展开数组和对象查看内部结构
- 实时显示变量值的变化

#### 2. 监视表达式 (Watch)
- 添加复杂表达式进行持续监视
- 例如：`$user->getProfile()->getName()`

#### 3. 调用堆栈 (Call Stack)
- 显示代码执行路径
- 点击堆栈帧跳转到对应上下文
- 理解函数调用关系

#### 4. 调试控制台 (Debug Console)
- 执行任意 PHP 表达式
- 实时测试代码片段
- 修改变量值进行实验

#### 5. 调试控制按钮
- **继续 (F5)**：执行到下一个断点
- **单步跳过 (F10)**：执行当前行，不进入函数
- **单步进入 (F11)**：执行当前行，进入函数内部
- **单步跳出 (Shift+F11)**：跳出当前函数
- **重启/停止**：重新开始或结束会话

### 实际调试示例

现场演示

### xdebug如何知道断点位置

Xdebug知道断点位置的机制是：

1. **IDE设置断点** → 本地记录
2. **Xdebug连接** → 建立调试会话  
3. **IDE发送断点信息** → Xdebug注册到内部
4. **PHP执行监控** → 每行检查是否匹配断点
5. **命中断点** → 暂停执行，通知IDE

这就是为什么：

- 需要先启动调试监听，再触发请求
- 断点信息是实时同步的
- 可以在调试过程中动态修改断点



## 六、Xdebug 的其他强大功能

### 1. 增强的堆栈跟踪
配置：`xdebug.mode = develop`

当发生错误时，Xdebug 提供：
- 完整的函数调用栈
- 每个函数的参数值
- 语法高亮的错误信息
- 可折叠的详细信息

### 2. 代码覆盖率分析
配置：`xdebug.mode = coverage`

用于测试环境，生成代码覆盖率报告：
```bash
phpunit --coverage-html ./coverage-report
```

可以查看：
- 哪些代码行被测试覆盖
- 哪些代码缺少测试
- 测试覆盖率百分比

### 3. 性能分析
配置：`xdebug.mode = profile`

生成缓存grind文件，使用工具分析：
- 函数调用次数
- 执行时间分布
- 内存使用情况
- 性能瓶颈定位

分析工具：
- QCacheGrind (Linux)
- KCacheGrind (Linux)
- WinCacheGrind (Windows)

## 七、常见问题与解决方案

### 问题1：断点不生效，IDE 收不到调试请求

**排查步骤**：
1. 确认 Xdebug 安装成功
   ```bash
   php -m | grep xdebug
   ```

2. 检查 php.ini 配置
   - 确认 `xdebug.mode=debug`
   - 确认 `xdebug.client_port=9003`
   - 确认 `zend_extension` 路径正确

3. 检查 IDE 配置
   - 确认调试监听已启动
   - 确认端口号匹配
   - 检查路径映射（远程调试时）

4. 检查浏览器触发
   - 使用浏览器扩展（Xdebug Helper）
   - 或配置 `xdebug.start_with_request=yes`

### 问题2：调试会话无法启动

**解决方案**：
1. 检查防火墙设置
2. 确认端口未被占用
3. 重启 IDE 和 Web 服务器
4. 查看 IDE 调试日志

### 问题3：性能明显下降

**原因**：Xdebug 会显著影响性能，这是正常现象

**建议**：
- 仅在开发环境启用 Xdebug
- 生产环境务必禁用 Xdebug
- 需要性能分析时再开启 profile 模式

## 八、实用技巧与最佳实践

### 1. 远程调试配置
对于 Docker 或远程服务器：

> php.ini配置

```ini
[XDebug]
zend_extension=xdebug.so
xdebug.remote_enable = 1
xdebug.remote_autostart=1
xdebug.remote_handler = "dbgp"
; Set to host.docker.internal on Mac and Windows, otherwise, set to host real ip
xdebug.remote_host = host.docker.internal
; xdebug.remote_host = 192.168.100.1 real_ip
xdebug.remote_port = 9003
xdebug.idekey=VSCODE
xdebug.remote_log = /var/log/php/xdebug.log
```

> 修改IDE中launch配置

在 IDE 中配置服务器路径与本地路径的映射：
```json
"pathMappings": {
    "/www/wm-listing.valsun.test": "${workspaceFolder}" // 左边替换为项目实际所在的服务器的位置
}
```

### 2.脚本调试

对于laravel项目或者远程/Docker部署项目，必须使用监听模式开启调试
对于本地部署的直接使用php命令执行的脚本，可以选择启动模式或者监听模式

> 两种模式有何不同：
>
> | 配置名称                         | 作用                                        | 什么时候用                                 |
> | -------------------------------- | ------------------------------------------- | ------------------------------------------ |
> | **Listen for Xdebug**            | 等待 PHP（FPM、Nginx、Apache、CLI）主动连接 | 调试网页、接口、artisan 命令               |
> | **Launch currently open script** | VSCode 主动执行一个 PHP 文件                | 调试单个纯脚本（不走浏览器、不走 artisan） |

### 3.高级断点

1. ##### Expression（条件断点）

   只有当表达式为真时，才会停下，适合循环内断点

   例如填写 `$a > 0` 只有在 变量 `$a`大于0的时候才会停止

2. ##### Hit Count（命中次数断点）

   代码被执行第 N 次时才会停。

   填入数字或者比较表达式 例如 `2`、 `>=2`

3. ##### Log Message（日志断点 / Logpoint）

   不会暂停程序，只在 Debug Console 输出你写的日志。

   输入示例：`a= {$a}, b = {$b}` 变量值通过`{}`符号注入 输出 `a= 2, b = 1`

4. ##### Wait for Breakpoint（排队等待）

   该断点会等待其他断点条件满足后才停。比较少用，适合复杂逻辑

##### 总结

| 类型                    | 是否暂停程序 | 触发条件           | 常用场景                 |
| ----------------------- | ------------ | ------------------ | ------------------------ |
| **Expression**          | ✔ 会停       | 表达式为 true      | 筛选变量、定位异常       |
| **Hit Count**           | ✔ 会停       | 第 N 次执行        | 循环、重复调用、定点触发 |
| **Log Message**         | ✖ 不停       | 每次执行都打印日志 | 查看变量、不影响流程     |
| **Wait for Breakpoint** | ✔ 会停       | 等别的断点先触发   | 链式调试                 |

### 

## 九、总结

### 核心价值
1. **提升调试效率**：告别重复修改代码的原始方式
2. **深入理解代码**：通过单步执行真正理解程序流程
3. **快速定位问题**：实时观察变量状态，快速找到 Bug 根源
4. **改善代码质量**：通过覆盖率和性能分析优化代码

### 行动建议
1. **立即实践**：在你的开发环境中安装配置 Xdebug
2. **替代习惯**：用断点调试替代 `var_dump` 和 `die`
3. **掌握快捷键**：熟练使用调试控制快捷键
4. **探索高级功能**：尝试代码覆盖率和性能分析

### 最后提醒
- 生产环境务必禁用 Xdebug
- 定期更新 Xdebug 版本
- 结合单元测试获得最佳效果
- 与团队成员分享调试技巧

掌握 Xdebug 是现代 PHP 开发者的必备技能，它将彻底改变你的调试方式，显著提升开发效率和代码质量。