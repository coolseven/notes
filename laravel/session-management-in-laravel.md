# session in laravel vs session in other frameworks

这篇笔记从session blocking 问题出发，分析和对比了 laravel 和其他框架的 session 处理机制的优缺点，并分析了如何设计一个良好的 session ，从而提高系统的并发。

## 一. session blocking problem

现在的网站会大量使用 ajax 请求来优化页面体验。

有些 ajax 请求是需要顺序执行的，有些 ajax 请求则是可以并行执行的。

 对于可以并行执行的 ajax 请求，我们可以在 js 中同时触发这些请求，并期望这些 ajax 请求在后端也被并行执行。但是，如果我们打开浏览器开发工具中的 network 面板，我们会发现，这些 ajax 请求虽然是同时触发并同时到达后端的，但是只有某个一个 ajax 请求处理完成之后，另一个 ajax 请求才会真正开始被处理。

 你可以打开这个网站,在 F12 模式下观察 xhr 的时间顺序: https://demo.ma.ttias.be/demo-php-blocking-sessions/

### 1.2 如何重现这个问题

1.  php 编译时启用了 session 模块(默认启用), 
2.  php 代码或框架代码中使用了 `session_start()` 方法来启动 session
3.  后台使用了 session 来辨别每个请求所属的用户
4.  同时触发多个 ajax 请求,每个请求头部都携带相同的 sessionId
5.  后台配置的 `session.save_handler` 为 php 内置的 `file` 或 `memcache`

备注：ThinkPHP 框架和 Symfony 框架都满足 第 2 点。

### 1.3 问题产生的原因

这个问题产生的根本原因，是因为对于 php 内置的 `file` 或 `memcache` 类型的 sessionHandler  来说，`session_start()` 方法默认是阻塞的。

对于同一个 session_id 来说，前一个请求使用 `session_start()` 之后，会自动给这个 session_id 对应的资源加上读写锁。该锁直到调用  `session_commit() ` 或者 `session_write_close()` 或者脚本结束之后才会释放。

回顾一下 php内置的 sessionHandler 的处理机制：

-   代码或框架调用 `session_start()`方法来启动 session

    -   对应的 `sessionHandler  ` 使用 `open()` 方法打开对应的存储器
    -   对应的 `sessionHandler  ` 使用 `read()` 方法获取到反序列化之后的 session数据
    -   对应的 `sessionHandler  ` 将获取到的 session 数据挂载到 `$_SESSSION` 全局变量上

-   代码或框架读取/更新  `$_SESSSION` 变量

-   代码或框架调用 `session_commit() `或者`session_write_close()` 来提交 `$_SESSSION`，或者在脚本结束时，由php 自动提交 `$_SESSSION`

    -   对应的 `sessionHandler  ` 使用 `write()` 方法将`$_SESSSION` 全局变量上的数据写入到对应的资源中

-   session 的自动回收此处不分析

注意 `session_start()`的第一阶段，即对应的 `sessionHandler  ` 使用 `open()` 方法打开对应的资源时，如果 sessionHandler  类型为  `file` 或 `memcache` , 那么 `open()` 方法会 **自动给打开的资源上锁**。该锁为 LOCK_EX 类型(排他锁),意味着如果该锁被其他的进程获取了,那么当前进程会堵塞,直到该锁被其他所有进程释放.

```c
// 相关文件: https://github.com/php/php-src/blob/master/ext/session/mod_files.c#L156
// data->fd 即为保存当前 session 的文件

static void ps_files_open(ps_files *data, const char *key)
{
......

		if (data->fd != -1) {
......
			do {
				ret = flock(data->fd, LOCK_EX);
			} while (ret == -1 && errno == EINTR);

#ifdef F_SETFD
# ifndef FD_CLOEXEC
#  define FD_CLOEXEC 1
# endif
			if (fcntl(data->fd, F_SETFD, FD_CLOEXEC)) {
				php_error_docref(NULL, E_WARNING, "fcntl(%d, F_SETFD, FD_CLOEXEC) failed: %s (%d)", data->fd, strerror(errno), errno);
			}
#endif
		} else {
			php_error_docref(NULL, E_WARNING, "open(%s, O_RDWR) failed: %s (%d)", buf, strerror(errno), errno);
		}
	}
}
```



因此，在某个请求[打开session之后，提交session之前]的这段时间里，如果有其它的相同 sessionId 的请求执行到了  `session_start() ` ，这些请求都会进入阻塞状态，直到这个 session 资源上的锁被释放。

而当锁被释放之后，这些请求中也只能有一个请求能抢到 session 资源，于是剩余的请求又只能进入阻塞状态中了。



### 1.4 解决办法

从上面的机制中可以看到，解决这个问题的办法下面几种：

-   尽量减少每个请求内的 session 锁的占用时间。亦即尽可能早地使用  `session_commit() `来结束session。
-   设置 session.save_handler 为 `redis` ,  因为 redis 的 session 操作不支持锁
-   如果你的 session.save_handler 为 `memcached` ，那么将 `memcached.sess_locking = 'off'`
-   通过 `ini.session_set_save_handler()` 来使用自定义的 sessionHandler ，在自定义的 sessionHandler 的 `open()` 方法中优化session的加锁逻辑
-   不再依赖于 php 的 session 模块 去管理 session，而是独立实现一套 session 管理机制



### 1.5 拓展

你可能会问，如果我在 `session_commit()` 结束 session 之后，又需要更新session怎么办？比如在初始化时把用户的信息加载到session中，然后关闭session，在表单校验后需要把表单错误信息写入到session中。

如果你打算尝试使用 `session_start() `来重新启动 session，那么，你可能会遇到 headers_already_sent 的问题：

```php
<?php
 // open session for the first time 
  session_start(); 
 // read session
  $user_id = $_SESSSION['user_id'];  
 // update session
  $_SESSION['user_permission']  = ['canDoSomething' => true]; 
 // release session lock early, to prevent session blocking
  session_commit();

...
 // some business logic 
...

 // open session for the second time
 session_start(); // you will probably get "headers already sent" warning
```

此时，你需要使用 ob_start() / ob_get_clean() 等方式来防止第二次输出header之前有任何body的输出

## 二. laravel 的 session 管理机制

在上面 1.4 中，我们看到，解决 session blocking problem 的其中一个办法就是不再依赖php的session wrapper，而是独立实现一套session管理机制。laravel 也是选择的的这一种方式。

注: laravel 的 session 管理组件并不依赖 php 的 session 模块,这意味着即使你在编译 php 时禁用了 session 模块(`./configure  --disable-session`),laravel 中涉及到 session 的部分也仍然可以正常运行.

### 2.1 laravel 支持的 session driver

根据 `config/session.php` 配置文件的说明，laravel 目前提供了如下的session driver：

```php
/*
    |--------------------------------------------------------------------------
    | Default Session Driver
    |--------------------------------------------------------------------------
    |
    | This option controls the default session "driver" that will be used on
    | requests. By default, we will use the lightweight native driver but
    | you may specify any of the other wonderful drivers provided here.
    |
    | Supported: "file", "cookie", "database", "apc",
    |            "memcached", "redis", "array"
    |
    */

    'driver' => env('SESSION_DRIVER', 'file'),
```

### 2.2 laravel 的 FileSessionHandler 分析

以 file 类型的 driver 为例，其对应的处理类位于 `vendor\laravel\framework\src\Illuminate\Session\FileSessionHandler.php`,其代码如下：

```php
<?php

namespace Illuminate\Session;
use .....

class FileSessionHandler implements SessionHandlerInterface
{
    /**
     * {@inheritdoc}
     */
    public function open($savePath, $sessionName)
    {
        return true;
    }

/**
     * {@inheritdoc}
     */
    public function close()
    {
        return true;
    }

    /**
     * {@inheritdoc}
     */
    public function read($sessionId)
    {
        if ($this->files->exists($path = $this->path.'/'.$sessionId)) {
            if (filemtime($path) >= Carbon::now()->subMinutes($this->minutes)->getTimestamp()) {
                return $this->files->get($path, true);
            }
        }

        return '';
    }

    /**
     * {@inheritdoc}
     */
    public function write($sessionId, $data)
    {
        $this->files->put($this->path.'/'.$sessionId, $data, true);

        return true;
    }
}
```

这段代码有三个地方值得注意：

一个是 该类继承了 标准的 `\SessionHandlerInterface`接口 ,意味着你可以在其他项目中通过 `ini.session_set_save_handler()` 来移植 laravel 的这一套 session 机制

另一个是 该类的open() 直接返回 true ， 因此读取session时不会加锁，可以并发读。

第三个是 该类的 write() 并不存在锁机制。

第二点和第三点结合起来看，完全避免了 session blocking problem 。



## 三. laravel 和 其它框架的 session 管理机制的异同点

相同点：

-   模式相同，都使用了内存变量： 打开存储器 -> 挂载到变量 -> 读写变量 -> 将变量持久化到存储器
-   对于传统模式来说,该变量是 $_SESSION, 
-   对于 laravel 来说,该变量是 Illuminicate\Session\Store 对象上的 attributes 属性

不同点：

-   传统方式只要读写 $_SESSION 变量即可，剩下的 session 过期判断，回收，读取，持久化，以及在响应的 Header 中设置相关的响应头等都是由 php 内置的 session 模块自动完成的
-   在 larave 中,上述的每一步都是在框架层由 StartSession 中间件实现的。

-   laravel 的打开存储器不加锁，
-   传统方式打开存储器时默认会加锁，其他进程不可读，更不可写

-   同一个请求中,传统方式下,在一对 `session_start()` -> `session_commit()`  之间,如果出现了重复的 `session_start()`，那么这些多余的 `session_start()` 并不会重新打开存储器,而是直接被忽略。
-   同一个请求中,laravel 机制下,在一对 `Session::start()` -> `Session::save()` 之间,如果出现了重复的 `Session::start()` ，那么每次 `Session::start()` 时,内存中的 session 数据都会与存储器中的最新数据进行合并. 因此,laravel 中的  `Session::start()` 方法可以理解成 `Session::mergeAttributesWithLatestStore()`

-   laravel 的 session 更方便测试
-   laravel 的 sessionHandler 选项更多，包含 database / apc / array / cookie
-   由于 laravel 是在 php 层实现的 session 管理,如果请求在到达 Kernel 的 terminate() 阶段前就被 exit 了,那么 session 中的数据将丢失,无法保存到存储器.



## 四. 拓展

### 4.1 重新认识session

session 这个词翻译成中文叫做“会话”，现实中，两个人之间进行会话时，正常的形式都是你问我答，再问再答，如果你使用了分身术，一下子同时问好几个问题，我可以选择也使用分身术，然后每个我同时各自回答你的问题，也可以选择只有我一个人，挨个回答问题。

### 4.2 session racing problem ?
如果对于同一个 sessionId 存在多个并发请求,那么这些并发的请求可能会并发地修改 session 中的数据,这样会可能导致最终的存储器中的数据被互相覆盖.


### 4.3 session racing problem should be a fake problem

session 的核心目的是区分当前http请求的访问者是哪一个用户。

session 存储方式从大的思路上可以分为两类: 
- 一类是将 session 数据存储在用户的浏览器端,
- 一类是将 session 数据存储在服务器端,如 file,redis,memcache 等
不管是哪一类,session 数据中至少会保存该访问者的用户 id   
对于前一类,session 数据的解析和存储都只涉及到请求头和响应头,服务器无需任何存储器,因此先不讨论.  
而对于后一类,由于连接并打开存储器,从而获取其中的用户id 这一步是不可避免的,因此,各种实践中,除了在存储器中保存用户id之外,还会保存一些其他的热点数据作为缓存,从而节省一次查询.  

因此,避免 session racing problem, 实际上也就是避免 cache racing problem.   
我们知道,缓存中不应该存储变更特别频繁,或者实时性很重要的数据,这个原则对于 session 来说也是一样的.   
相反,对于表单验证提示,菜单权限等数据就比较适合保存到 session 中. 这些场景并不会产生 session racing problem  

### 4.4 access_token or session_cookie ?

## 参考资料

https://wiki.php.net/rfc/session-lock-ini







