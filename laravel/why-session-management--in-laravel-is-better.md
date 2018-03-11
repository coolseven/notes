# 为什么 laravel 的 session 机制更好

这篇笔记从session blocking 问题出发，进而分析和对比了 laravel 和其他框架的 session 处理机制的优缺点，并分析了如何设计一个良好的session ，从而提高系统的并发。

## 一. session blocking problem

现在的网站会大量使用 ajax 请求来优化页面体验。

有些 ajax 请求是需要顺序执行的，有些 ajax 请求则是可以并行执行的。

 对于可以并行执行的 ajax 请求，我们可以在 js 中同时触发这些请求，并期望这些 ajax 请求在后端也被并行执行。但是，如果我们打开浏览器开发工具中的 network 面板，我们会发现，这些 ajax 请求虽然是同时触发并同时到达后端的，但是只有某个一个 ajax 请求处理完成之后，另一个 ajax 请求才会真正开始被处理。

https://demo.ma.ttias.be/demo-php-blocking-sessions/

### 1.2 如何重现这个问题

1.  ajax 请求的后台接口是 php 
2.  ajax 请求头部携带了SessionCookie
3.  后台使用了 session 来辨别 ajax 请求所属的用户
4.  后台配置的 `session.save_handler` 为 php 内置的 `file` 或 `memcache`
5.  后台代码中直接或间接使用了 `session_start()` 方法来启动 session

备注：ThinkPHP 框架和 Symfony 框架都满足 第 5 点。

### 1.3 问题产生的原因

这个问题产生的根本原因，是因为对于 php 内置的 `file` 或 `memcache` 类型的 sessionHandler  来说，`session_start()` 方法是 阻塞的。

对于同一个 session_id 来说，前一个请求使用 `session_start()` 之后，会自动给这个 session_id 对应的资源加上读写锁。该锁直到 `session_write_close()` 或者脚本结束之后才会释放。

回顾一下 php内置的 sessionHandler 的处理机制：

-   代码或框架调用 `session_start()`方法来启动 session

    -   对应的 `sessionHandler  ` 使用 `open()` 方法打开对应的资源
    -   对应的 `sessionHandler  ` 使用 `read()` 方法获取到反序列化之后的 session数据
    -   对应的 `sessionHandler  ` 将获取到的 session 数据挂载到 `$_SESSSION` 全局变量上

-   代码或框架读取/更新  `$_SESSSION` 变量中的 session 数据

-   代码或框架在最终阶段调用 `session_commit() `或者`session_write_close()` 来提交 `$_SESSSION`，或者在脚本结束时，由php 自动提交 `$_SESSSION`

    -   对应的 `sessionHandler  ` 使用 `write()` 方法将`$_SESSSION` 全局变量上的数据写入到对应的资源中

-   session 的自动回收此处不分析

注意 `session_start()`的第一阶段，即对应的 `sessionHandler  ` 使用 `open()` 方法打开对应的资源时，如果 sessionHandler  类型为  `file` 或 `memcache` , 那么 `open()` 方法会**自动给打开的资源上锁(类似于悲观锁，锁住整个事务(当前的会话))**。

因此，在某个请求[打开session之后，提交session之前]的这段时间里，如果有其它的请求执行到了  `session_start() ` ，这些请求都会进入阻塞状态，直到这个 session 资源上的锁被释放。

而当锁被释放之后，这些请求中只能有一个请求能占用session资源，于是剩余的请求又只能进入阻塞状态中了。



### 1.4 解决办法

从上面的机制中可以看到，解决这个问题的办法下面几种：

-   尽量减少每个请求内的 session 锁的占用时间。亦即尽可能早地使用  `session_commit() `来结束session。
-   设置 session.save_handler 为 `redis` ,  因为 redis 的 session 操作不支持锁
-   如果你的 session.save_handler 为 `memcached` ，那么将 `memcached.sess_locking = 'off'`
-   通过 `ini.session_set_save_handler()` 来使用自定义的 sessionHandler ，在自定义的 sessionHandler 的 `open()` 方法中优化session的加锁逻辑
-   不再依赖于 php 的 session wrapper 去管理 session，而是独立实现一套 session 管理机制



### 1.5 拓展

你可能会问，如果我在 `session_commit()` 结束 session 之后，又需要更新session怎么办？比如在初始化时把用户的信息加载到session中，然后关闭session，在表单校验后需要把表单错误信息写入到session中。

如果你打算尝试使用 `session_start() `来重新启动 session，那么，你可能会遇到一系列的问题：

```php
<?php
 // open session for the first time 
  session_start(); 
 // read session
  $user_id = $_SESSSION['user_id'];  
 // update session
  $_SESSION['user_permission']  = ['canDoSomething' => true]; 
 // release session to prevent session blocking
  session_commit();

...
 // some business logic 
...

 // open session for the second time
 session_start(); // you will probably get "headers already sent" warning
```

此时，你需要使用 ob_start() 系列函数来防止第二次输出header之前有任何body的输出

## 二. laravel 的 session 管理机制

在上面 1.4 中，我们看到，解决 session blocking problem 的其中一个办法就是不再依赖php的session wrapper，而是独立实现一套session管理机制。laravel 也是选择的的这一种方式。

Facade\Session::start() 

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

以 file 类型的 driver 为例，其对应的 session 处理类位于 `vendor\laravel\framework\src\Illuminate\Session\FileSessionHandler.php`,其代码如下：

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

第三个是 该类的 write() 并不存在版本检查(乐观锁)机制。

第二点和第三点结合起来看，完全避免了 session blocking problem 。



## 三. laravel 和 其它框架的 session 管理机制的异同点

相同点：

-   模式相同，都使用了中间变量： 打开存储器 -> 挂载到中间变量 -> 读写中间变量 -> 将中间变量写入存储器

不同点：

-   传统方式只要读写$_SESSION_ 变量即可，剩下的session过期，session回收，session 的读取，session 的持久化，以及响应的Header中设置 session相关的cookie 都由php自动完成
-   larave 每一步都是独立实现的。特别是 session 的读取和持久化，跟响应Header 是独立的，意味着即使输出了header，如 terminate() 阶段,你也依然可以读写session，当然，此时你不能再操作Header了。
-   传统方式，当你输出了 header ，或/及 body 之后，再尝试读写session 时，会产生一个 Warning
-   laravel 方式，当你输出了 Header ，或/及 body 之后，再尝试读写session 时，不产生任何 Warning


-   laravel 的打开存储器不加锁，
-   传统方式打开存储器时默认会加锁，并且是事务锁，其他进程不可读，更不可写
-   laravel 在Session::save() 之前如果发生过多次 Session::start() ，那么每次都会真的打开存储器导致中间变量的修改会被重置
-   传统方式在 session_commit() 之前多次调用 session_start() 方法，只有第一次是有效的。
-   laravel 的 session 存取 跟 Header 中的 setCookie('laravel-session') 是独立的，


-   laravel 的 session 更方便测试
-   laravel 的 sessionHandler 选项更多，包含 database / apc / array / cookie
-   laravel 的 session 会保存两次 (handle() 一次， terminate() 又一次)
-   laravel 的 session 没有 lock() 方法



## 四. 拓展

### 4.1 重新认识session

session 这个词翻译成中文叫做“会话”，现实中，两个人之间进行会话时，正常的形式都是你问我答，再问再答，如果你使用了分身术，一下子同时问好几个问题，我可以选择也选择使用分身术，然后每个我同时各自回答你的问题，也可以选择只有我一个人，挨个回答问题。

### 4.2 session racing problem ?



### 4.3 session racing problem is a fake problem

session 的核心目的是区分当前http请求的访问者是哪一个用户/哪一个游客。

系统识别出访问者之后，通常需要获取访问者的其他信息。由于sessionHandler 解析session资源获取用户id 这一步是不可避免的，那么，干脆让这一步顺便把其他信息也解析出来。这样就不用单独去数据库或缓存中查询其他信息了。

session 中可以存什么样的数据？

-   用户标识 ，如 user_id (必须)
-   没有必要专门设计缓存 key 的一次性数据 ，如表单错误提示 (常用)，用户最后一次登陆时间等。
-   用户登录之后不会发生变更且其他请求会频繁使用到的信息 (可选，视实际业务而定，用于节省查询次数)

session 中不应该存什么样的数据？

-   会频繁变动的数据，如用户最后一次请求的接口地址。如果存在session中，很明显会出现多个ajax并发请求时互相覆盖的问题

### 4.4 access_token or session_cookie ?

## 参考资料

https://wiki.php.net/rfc/session-lock-ini







