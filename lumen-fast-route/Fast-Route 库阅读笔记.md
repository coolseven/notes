## Fast-Route 路由库阅读笔记

## 介绍

fast-route 是一个基于正则表达式的 php 路由库, 用于

- 基于**正则表达式**的路由配置
- **高效**匹配当前请求对应的路由配置

```php
composer require nikic/FastRoute
```

## 路由规则

一条路由规则一般包含三个部分

- HTTP Method
- Uri Pattern
- Handler

例如:

- lumen (fast-route) 中定义路由

  ```php
  /** @var Laravel\Lumen\Routing\Router $router */
  $router->get('1/{routeId:\d+}', ['uses' => 'RegexController@routeId']);
  ```

  

- yii2 中定义路由

  ```php
  'rules' => [
      'PUT,POST post/<id:\d+>' => 'post/update',
  ]
  ```

  

- laravel (symfony3) 中定义路由

  ```php
  use Route;
  
  Route::get('1/{routeId}','RegexController@routeId')->where([
      'routeId'=>'\d+' ,
  ]);
  ```

## 高效

- lumen  , symfony 4 , thinkphp 5.1.7  等框架均基于 fast-route 的路由匹配优化原理来封装自己的路由库

- 本机 lumen 和 laravel 对比实测:  

  - 360 条相同的正则路由 ( http method , uri pattern , handler 均相同)

  - 均只启用一个相同的中间件

  - 不使用 route:cache 

  - 不使用 opcache 

  - 不使用数据库或缓存等服务

    |                     | 匹配第一条路由 | 匹配最后一条路由 | 无匹配的路由 |
    | ------------------- | -------------- | ---------------- | ------------ |
    | lumen / fast-route  | 50 ms          | 50 ms            | 50 ms        |
    | laravel / symfony 3 | 101 ms         | 130 ms           | 160 ms       |
    | yii2                | 待测试         | 待测试           | 待测试       |

- 可以发现, fast-route 的特点是路由解析时间相对比较稳定, 几乎不受是否匹配得到路由的影响, 而 laravel 的路由解析则受当前请求的路由顺序影响较大

## 路由组件在 laravel 和 lumen 框架中实现的区别

#### 先对比 laravel 和 lumen 的框架整体流程

##### laravel app 整体流程

- 初始化 app ( container )
- handle 请求, 获取响应
  - 解析 $_GLOBALS 生成 request 对象
  - bootstrap
    - 收集 routes/web.php 中注册的路由规则 [http method , uri , handler]   **( collect )**
    - 注册 middlewares 中间件
  - router 组件进行匹配和分发  ( **dispatch** )
    - router 组件遍历路由规则, 与  request 对象进行对比, 得到匹配的路由规则和路由参数,  **( match )**
    - 前置中间件处理请求
    - 调用路由对象中的 handler 处理请求  
    - 后置中间件处理请求
- 输出响应
- 结束


##### lumen app 的整体流程

- 初始化 app ( container )
  - 调用 app 中的 router 对象, 收集 routes/web.php 中注册的路由规则 [http method , uri , handler]   **( collect )**
  - 注册 middlewares 中间件
- handle 请求, 获取响应
  - 解析 $_GLOBALS 生成 request 对象
  - RoutesRequests 组件进行匹配和分发 ( RoutesRequests  类似于 laravel 中的 HttpKernel )   ( **dispatch** )
    - 创建 FastRouteDispatcher 对象获取匹配的路由  **( match )**
    - 前置中间件处理请求
    - 调用路由对象中的 handler 处理请求  
    - 后置中间件处理请求
- 输出响应
- 结束

------

可以看到 laravel 和 lumen 中, 路由部分的流程是相似的,都是 collect -> match -> dispatch 这三步, 



那为什么 fast-route 中的路由比 laravel 还要快呢?

#### collect 阶段对比

- laravel 收集路由: 

```php
protected function createRoute($methods, $uri, $action)
    {
        // If the route is routing to a controller we will parse the route action into
        // an acceptable array format before registering it and creating this route
        // instance itself. We need to build the Closure that will call this out.
        if ($this->actionReferencesController($action)) {
            $action = $this->convertToControllerAction($action);
        }

        $route = $this->newRoute(
            $methods, $this->prefix($uri), $action
        );

        // If we have groups that need to be merged, we will merge them now after this
        // route has already been created and is ready to go. After we're done with
        // the merge we will be ready to return the route back out to the caller.
        if ($this->hasGroupStack()) {
            $this->mergeGroupAttributesIntoRoute($route);
        }

        $this->addWhereClausesToRoute($route);

        return $route;
    }
```



- lumen 收集路由数组

```php
public function addRoute($method, $uri, $action)
    {
        $action = $this->parseAction($action);

        $attributes = null;

        if ($this->hasGroupStack()) {
            $attributes = $this->mergeWithLastGroup([]);
        }

        if (isset($attributes) && is_array($attributes)) {
            if (isset($attributes['prefix'])) {
                $uri = trim($attributes['prefix'], '/').'/'.trim($uri, '/');
            }

            if (isset($attributes['suffix'])) {
                $uri = trim($uri, '/').rtrim($attributes['suffix'], '/');
            }

            $action = $this->mergeGroupAttributes($action, $attributes);
        }

        $uri = '/'.trim($uri, '/');

        if (isset($action['as'])) {
            $this->namedRoutes[$action['as']] = $uri;
        }

        if (is_array($method)) {
            foreach ($method as $verb) {
                $this->routes[$verb.$uri] = ['method' => $verb, 'uri' => $uri, 'action' => $action];
            }
        } else {
            $this->routes[$method.$uri] = ['method' => $method, 'uri' => $uri, 'action' => $action];
        }
    }
```

在 collect 阶段, laravel 收集的 Route 格式是对象, lumen 收集的 Route 格式是数组,  

此阶段虽然没有涉及正则解析 , 但是 laravel 在收集路由时进行了**大量的对象创建工作**

当存在 400 条路由时,

- laravel  collect 阶段耗时:   45 ms

- lumen  collect 阶段耗时:   19 ms

  

#### dispatch 阶段对比 

laravel 和 lumen 都是调用 Route 中的 action 相同的 handler , 都经过了相同的中间件,  因此此阶段对耗时的差异影响也较小.



#### match 阶段对比

- laravel 的路由匹配

  ```php
  // Illuminate\Routing\RouteCollection::match($request);
  public function match(Request $request)
      {
          $routes = $this->get($request->getMethod());
  
          // First, we will see if we can find a matching route for this current request
          // method. If we can, great, we can just return it so that it can be called
          // by the consumer. Otherwise we will check for routes with another verb.
          $route = $this->matchAgainstRoutes($routes, $request);
  
          if (! is_null($route)) {
              return $route->bind($request);
          }
  
          // If no route was found we will now check if a matching route is specified by
          // another HTTP verb. If it is we will need to throw a MethodNotAllowed and
          // inform the user agent of which HTTP verb it should use for this route.
          $others = $this->checkForAlternateVerbs($request);
  
          if (count($others) > 0) {
              return $this->getRouteForMethods($request, $others);
          }
  
          throw new NotFoundHttpException;
      }
  
  // Illuminate\Routing\RouteCollection::matchAgainstRoutes($request);
  	protected function matchAgainstRoutes(array $routes, $request, $includingMethod = true)
      {
          list($fallbacks, $routes) = collect($routes)->partition(function ($route) {
              return $route->isFallback;
          });
  
          return $routes->merge($fallbacks)->first(function ($value) use ($request, $includingMethod) {
              return $value->matches($request, $includingMethod);
          });
      }
  
  // Illuminate\Routing\Route::matches($request);
      public function matches(Request $request, $includingMethod = true)
      {
          $this->compileRoute();
  
          foreach ($this->getValidators() as $validator) {
              if (! $includingMethod && $validator instanceof MethodValidator) {
                  continue;
              }
  
              if (! $validator->matches($this, $request)) {
                  return false;
              }
          }
  
          return true;
      }
  ```

  可以看到, laravel 的路由 match 阶段的逻辑是

  1. 对 collect 阶段收集到的路由进行遍历 , 每一次遍历时:

  2. **compile**   将当前的路由对象中的 uri pattern 编译成正则表达式以及占位符列表

  3. **validate**   检查当前的 request 对象的 method / host / uri 等是否满足当前的正则表达式

     第 3 步成功时, 

  4. **bind**          将当前的路由对象的占位符与 request 对象中的实际值进行绑定, 

  5. 停止遍历, 不再遍历剩余的路由

     第 3 步失败时,

  6. 遍历下一个路由    

     

  其中, 

  **compile** 操作的作用是把 路由规则中的 uri pattern 和 where 中的变量的正则定义编译成完整的正则表达式, 如,对于 laravel 路由:

  ```php
  Route::get('4/{routeId}','RegexController@routeId')->where(['routeId'=>'\d+']);
  ```

  编译之后得到的结果是:

  ```php
  $regex = '#^/4/(?P<routeId>\d+)$#sD' ;
  
  $variables = [0 => 'routeId',1 => 'appId',2 => 'name',3 => 'postId'];
  
  $tokens = [
    0 => [0 => 'text',1 => '/one/11'],
    1 => [0 => 'variable',1 => '/',2 => '\d+',3 => 'routeId',4 => true],
  ];
  ```

  **validate** 操作的作用则是判断当前实际的 request uri 是否符合编译之后的路由对象的正则表达式

  **bind** 操作的作用则是将当前实际的 request uri 中的参数值按照对应的位置赋值给编译之后的路由对象

  

- lumen 的路由匹配

  ```php
  // \Laravel\Lumen\Concerns\RoutesRequests::dispatch($request)
  public function dispatch($request = null)
      {
          list($method, $pathInfo) = $this->parseIncomingRequest($request);
  
          try {
              return $this->sendThroughPipeline($this->middleware, function () use ($method, $pathInfo) {
                  if (isset($this->router->getRoutes()[$method.$pathInfo])) {
                      return $this->handleFoundRoute([true, $this->router->getRoutes()[$method.$pathInfo]['action'], []]);
                  }
  
                  return $this->handleDispatcherResponse(
                      $this->createDispatcher()->dispatch($method, $pathInfo)
                  );
              });
          } catch (Exception $e) {
              return $this->prepareResponse($this->sendExceptionToHandler($e));
          } catch (Throwable $e) {
              return $this->prepareResponse($this->sendExceptionToHandler($e));
          }
      }
  
  // \Laravel\Lumen\Concerns\RoutesRequests::createDispatcher()
  protected function createDispatcher()
      {
          return $this->dispatcher ?: \FastRoute\simpleDispatcher(function (RouteCollector $r) {
              foreach ($this->router->getRoutes() as $route) {
                  $r->addRoute($route['method'], $route['uri'], $route['action']);
              }
          });
      }
  
  // lumen/vendor/nikic/fast-route/src/functions.php
      function simpleDispatcher(callable $routeDefinitionCallback, array $options = [])
      {
          $options += [
              'routeParser' => 'FastRoute\\RouteParser\\Std',
              'dataGenerator' => 'FastRoute\\DataGenerator\\GroupCountBased',
              'dispatcher' => 'FastRoute\\Dispatcher\\GroupCountBased',
              'routeCollector' => 'FastRoute\\RouteCollector',
          ];
  
          /** @var RouteCollector $routeCollector */
          $routeCollector = new $options['routeCollector'](
              new $options['routeParser'], new $options['dataGenerator']
          );
  
          $routeDefinitionCallback($routeCollector);
  
          return new $options['dispatcher']($routeCollector->getData());
      }
  ```

  可以看到, lumen 的路由 match 阶段的逻辑是:

  1. 对 collect 阶段收集到的路由进行遍历 , 每次遍历时:
  2. **compile** 当前路由, 将当前的路由对象中的 uri pattern 编译成正则表达式以及占位符列表, 并将当前路由及其编译结果存储在 fastRoute\dataGenerator 对象的 $methodToRegexToRoutesMap 属性中
  3. **generate** 遍历完成之后, 使用 fastRoute\dataGenerator 对象的 getData() 方法, 将 $methodToRegexToRoutesMap 属性中存储的解析结果合并, 生成一个大的正则表达式  $oneBigRegex 及其占位符列表
  4. **match**      将当前的 request 对象与 第3步生成的大正则表达式和占位符列表进行 match , 得到满足条件的路由以及该路由的占位符的值



由上面的分析可以看到, lumen 和 laravel 在路由解析上最大的区别在于:

- laravel 是在遍历路由集合的过程中针对每个路由对象进行单独的编译和匹配 , 匹配通过后停止遍历

- lumen 是先把路由集合遍历完, 然后将遍历的结果生成一个大的正则表达式, 然后直接将请求与这个大正则表达式进行匹配

  

  因此:   如果有 400 条路由, 

  如果请求满足第一条路由, 那么:

  - laravel 只需要 compile 1 次, 匹配 1 次
  - lumen  需要 compile 400 次, 匹配 1 次

  如果请求满足最后一条路由, 那么:

  -  laravel 需要 compile 400 次,  匹配 400 次 
  - lumen 需要 compile 400 次, 匹配 1 次

 #### laravel 和 lumen 的 compile 逻辑

- lumen 的 compile 逻辑:

```php
//   \FastRoute\RouteParser\Std::parse($route)  此处的  $route  实际就就是 uri pattern
public function parse($route)
    {
        $routeWithoutClosingOptionals = rtrim($route, ']');
        $numOptionals = strlen($route) - strlen($routeWithoutClosingOptionals);

        // Split on [ while skipping placeholders
        $segments = preg_split('~' . self::VARIABLE_REGEX . '(*SKIP)(*F) | \[~x', $routeWithoutClosingOptionals);
        if ($numOptionals !== count($segments) - 1) {
            // If there are any ] in the middle of the route, throw a more specific error message
            if (preg_match('~' . self::VARIABLE_REGEX . '(*SKIP)(*F) | \]~x', $routeWithoutClosingOptionals)) {
                throw new BadRouteException('Optional segments can only occur at the end of a route');
            }
            throw new BadRouteException("Number of opening '[' and closing ']' does not match");
        }

        $currentRoute = '';
        $routeDatas = [];
        foreach ($segments as $n => $segment) {
            if ($segment === '' && $n !== 0) {
                throw new BadRouteException('Empty optional part');
            }

            $currentRoute .= $segment;
            $routeDatas[] = $this->parsePlaceholders($currentRoute);
        }
        return $routeDatas;
    }


// lumen/vendor/nikic/fast-route/src/DataGenerator/RegexBasedAbstract.php:125
private function buildRegexForRoute($routeData)
    {
        $regex = '';
        $variables = [];
        foreach ($routeData as $part) {
            if (is_string($part)) {
                $regex .= preg_quote($part, '~');
                continue;
            }

            list($varName, $regexPart) = $part;

            if (isset($variables[$varName])) {
                throw new BadRouteException(sprintf(
                    'Cannot use the same placeholder "%s" twice', $varName
                ));
            }

            if ($this->regexHasCapturingGroups($regexPart)) {
                throw new BadRouteException(sprintf(
                    'Regex "%s" for parameter "%s" contains a capturing group',
                    $regexPart, $varName
                ));
            }

            $variables[$varName] = $varName;
            $regex .= '(' . $regexPart . ')';
        }

        return [$regex, $variables];
    }
```

- laravel 的 compile 逻辑

```php
// \Symfony\Component\Routing\RouteCompiler::compilePattern(Route $route, $pattern, $isHost) 
private static function compilePattern(Route $route, $pattern, $isHost)
    {
        $tokens = array();
        $variables = array();
        $matches = array();
        $pos = 0;
        $defaultSeparator = $isHost ? '.' : '/';
        $useUtf8 = preg_match('//u', $pattern);
        $needsUtf8 = $route->getOption('utf8');

        if (!$needsUtf8 && $useUtf8 && preg_match('/[\x80-\xFF]/', $pattern)) {
            throw new \LogicException(sprintf('Cannot use UTF-8 route patterns without setting the "utf8" option for route "%s".', $route->getPath()));
        }
        if (!$useUtf8 && $needsUtf8) {
            throw new \LogicException(sprintf('Cannot mix UTF-8 requirements with non-UTF-8 pattern "%s".', $pattern));
        }

        // Match all variables enclosed in "{}" and iterate over them. But we only want to match the innermost variable
        // in case of nested "{}", e.g. {foo{bar}}. This in ensured because \w does not match "{" or "}" itself.
        preg_match_all('#\{\w+\}#', $pattern, $matches, PREG_OFFSET_CAPTURE | PREG_SET_ORDER);
        foreach ($matches as $match) {
            $varName = substr($match[0][0], 1, -1);
            // get all static text preceding the current variable
            $precedingText = substr($pattern, $pos, $match[0][1] - $pos);
            $pos = $match[0][1] + strlen($match[0][0]);

            if (!strlen($precedingText)) {
                $precedingChar = '';
            } elseif ($useUtf8) {
                preg_match('/.$/u', $precedingText, $precedingChar);
                $precedingChar = $precedingChar[0];
            } else {
                $precedingChar = substr($precedingText, -1);
            }
            $isSeparator = '' !== $precedingChar && false !== strpos(static::SEPARATORS, $precedingChar);

            // A PCRE subpattern name must start with a non-digit. Also a PHP variable cannot start with a digit so the
            // variable would not be usable as a Controller action argument.
            if (preg_match('/^\d/', $varName)) {
                throw new \DomainException(sprintf('Variable name "%s" cannot start with a digit in route pattern "%s". Please use a different name.', $varName, $pattern));
            }
            if (in_array($varName, $variables)) {
                throw new \LogicException(sprintf('Route pattern "%s" cannot reference variable name "%s" more than once.', $pattern, $varName));
            }

            if (strlen($varName) > self::VARIABLE_MAXIMUM_LENGTH) {
                throw new \DomainException(sprintf('Variable name "%s" cannot be longer than %s characters in route pattern "%s". Please use a shorter name.', $varName, self::VARIABLE_MAXIMUM_LENGTH, $pattern));
            }

            if ($isSeparator && $precedingText !== $precedingChar) {
                $tokens[] = array('text', substr($precedingText, 0, -strlen($precedingChar)));
            } elseif (!$isSeparator && strlen($precedingText) > 0) {
                $tokens[] = array('text', $precedingText);
            }

            $regexp = $route->getRequirement($varName);
            if (null === $regexp) {
                $followingPattern = (string) substr($pattern, $pos);
                // Find the next static character after the variable that functions as a separator. By default, this separator and '/'
                // are disallowed for the variable. This default requirement makes sure that optional variables can be matched at all
                // and that the generating-matching-combination of URLs unambiguous, i.e. the params used for generating the URL are
                // the same that will be matched. Example: new Route('/{page}.{_format}', array('_format' => 'html'))
                // If {page} would also match the separating dot, {_format} would never match as {page} will eagerly consume everything.
                // Also even if {_format} was not optional the requirement prevents that {page} matches something that was originally
                // part of {_format} when generating the URL, e.g. _format = 'mobile.html'.
                $nextSeparator = self::findNextSeparator($followingPattern, $useUtf8);
                $regexp = sprintf(
                    '[^%s%s]+',
                    preg_quote($defaultSeparator, self::REGEX_DELIMITER),
                    $defaultSeparator !== $nextSeparator && '' !== $nextSeparator ? preg_quote($nextSeparator, self::REGEX_DELIMITER) : ''
                );
                if (('' !== $nextSeparator && !preg_match('#^\{\w+\}#', $followingPattern)) || '' === $followingPattern) {
                    // When we have a separator, which is disallowed for the variable, we can optimize the regex with a possessive
                    // quantifier. This prevents useless backtracking of PCRE and improves performance by 20% for matching those patterns.
                    // Given the above example, there is no point in backtracking into {page} (that forbids the dot) when a dot must follow
                    // after it. This optimization cannot be applied when the next char is no real separator or when the next variable is
                    // directly adjacent, e.g. '/{x}{y}'.
                    $regexp .= '+';
                }
            } else {
                if (!preg_match('//u', $regexp)) {
                    $useUtf8 = false;
                } elseif (!$needsUtf8 && preg_match('/[\x80-\xFF]|(?<!\\\\)\\\\(?:\\\\\\\\)*+(?-i:X|[pP][\{CLMNPSZ]|x\{[A-Fa-f0-9]{3})/', $regexp)) {
                    throw new \LogicException(sprintf('Cannot use UTF-8 route requirements without setting the "utf8" option for variable "%s" in pattern "%s".', $varName, $pattern));
                }
                if (!$useUtf8 && $needsUtf8) {
                    throw new \LogicException(sprintf('Cannot mix UTF-8 requirement with non-UTF-8 charset for variable "%s" in pattern "%s".', $varName, $pattern));
                }
            }

            $tokens[] = array('variable', $isSeparator ? $precedingChar : '', $regexp, $varName);
            $variables[] = $varName;
        }

        if ($pos < strlen($pattern)) {
            $tokens[] = array('text', substr($pattern, $pos));
        }

        // find the first optional token
        $firstOptional = PHP_INT_MAX;
        if (!$isHost) {
            for ($i = count($tokens) - 1; $i >= 0; --$i) {
                $token = $tokens[$i];
                if ('variable' === $token[0] && $route->hasDefault($token[3])) {
                    $firstOptional = $i;
                } else {
                    break;
                }
            }
        }

        // compute the matching regexp
        $regexp = '';
        for ($i = 0, $nbToken = count($tokens); $i < $nbToken; ++$i) {
            $regexp .= self::computeRegexp($tokens, $i, $firstOptional);
        }
        $regexp = self::REGEX_DELIMITER.'^'.$regexp.'$'.self::REGEX_DELIMITER.'sD'.($isHost ? 'i' : '');

        // enable Utf8 matching if really required
        if ($needsUtf8) {
            $regexp .= 'u';
            for ($i = 0, $nbToken = count($tokens); $i < $nbToken; ++$i) {
                if ('variable' === $tokens[$i][0]) {
                    $tokens[$i][] = true;
                }
            }
        }

        return array(
            'staticPrefix' => self::determineStaticPrefix($route, $tokens),
            'regex' => $regexp,
            'tokens' => array_reverse($tokens),
            'variables' => $variables,
        );
    }
```

可以看到, laravel ( symfony ) 的 compiler 相比 lumen ( fast-route ) 的 compiler, 功能上和逻辑上更加复杂, 提供了诸如  

- 子域名功能 ,
-  utf8 模式, 
- 自定义路由分隔符  
- 嵌套的 {} 表达式

等功能, 因此,当 laravel 的 compile 次数太多时, 会显著影响路由模块的整体耗时.



### 总结

lumen 的路由参数示例

- 路由定义

  ```php
  $router->get('1/routeId/{routeId:\d+}','RegexController@one');
  ```

- 单个路由的 compile 结果

  ```php
    array (
      '/1/routeId/(\\d+)' => 
      FastRoute\Route::__set_state(array(
         'httpMethod' => 'GET',
         'regex' => '/1/routeId/(\\d+)',
         'variables' => 
        array (
          'routeId' => 'routeId',
        ),
         'handler' => 
        array (
          'uses' => 'App\\Http\\Controllers\\RegexController@one',
        ),
      )),
    ),   
  ```

- 多个路由定义

  ```php
  $router->get('1/routeId/{routeId:\d+}','RegexController@one');
  $router->get('1/routeId/{routeId:\d+}/user/{name}','RegexController@two');
  ```

- 多个路由时最终的 compile 结果 (存储在 fastRoute\dataGenerator 对象的 $methodToRegexToRoutesMap 属性上)

  ```php
  array (
    'GET' => 
    array (
      '/1/routeId/(\\d+)' => 
      FastRoute\Route::__set_state(array(
         'httpMethod' => 'GET',
         'regex' => '/1/routeId/(\\d+)',
         'variables' => 
        array (
          'routeId' => 'routeId',
        ),
         'handler' => 
        array (
          'uses' => 'App\\Http\\Controllers\\RegexController@one',
        ),
      )),
        
      '/1/routeId/(\\d+)/user/([^/]+)' => 
      FastRoute\Route::__set_state(array(
         'httpMethod' => 'GET',
         'regex' => '/1/routeId/(\\d+)/user/([^/]+)',
         'variables' => 
        array (
          'routeId' => 'routeId',
          'name' => 'name',
        ),
         'handler' => 
        array (
          'uses' => 'App\\Http\\Controllers\\RegexController@two',
        ),
      )),
    ),
  )
  ```

  

- 最终生成的大正则表达式及其占位符列表

  ```php
  array (
    'GET' => 
    array (
      0 => 
      array (
        'regex' => '~^(?|/1/routeId/(\\d+)|/1/routeId/(\\d+)/user/([^/]+))$~',  // 大正则表达式
        'routeMap' => 
        array (
          2 => 
          array (
            0 => 
            array (
              'uses' => 'App\\Http\\Controllers\\RegexController@one',
            ),
            1 => 
            array (
              'routeId' => 'routeId',
            ),
          ),
          3 => 
          array (
            0 => 
            array (
              'uses' => 'App\\Http\\Controllers\\RegexController@two',
            ),
            1 => 
            array (
              'routeId' => 'routeId',
              'name' => 'name',
            ),
          ),
        ),
      ),
    ),
  )
  ```

- fast-route 的核心思路:

  将多个正则表达式合并为一个正则表达式

  ```php
  '/1/routeId/(\d+)',
  '/1/routeId/(\d+)/user/([^/]+)',
  
  // 合并的结果
  '~^(?|/1/routeId/(\d+)|/1/routeId/(\d+)/user/([^/]+))$~',
  
  // 使用当前的请求去匹配路由
  preg_match(
      '~^(?|/1/routeId/(\d+)|/1/routeId/(\d+)/user/([^/]+))$~',
      '/1/routeId/1/user/hzy', 
      $matches
  );
  
  $matches = array(3) {
    [0] =>
    string(21) "/1/routeId/1/user/hzy"
    [1] =>
    string(1) "1"
    [2] =>
    string(3) "hzy"
  }
  ```

  

### 正则表达式表

```
\ Quote the next metacharacter
^ Match the beginning of the line
. Match any character (except newline)
$ Match the end of the line (or before newline at the end)
| Alternation
() Grouping
[] Character class
* Match 0 or more times
+ Match 1 or more times
? Match 1 or 0 times
{n} Match exactly n times
{n,} Match at least n times
{n,m} Match at least n but not more than m times
More Special Character Stuff
\t tab (HT, TAB)
\n newline (LF, NL)
\r return (CR)
\f form feed (FF)
Even More Special Characters
\w Match a "word" character (alphanumeric plus "_")
\W Match a non-word character
\s Match a whitespace character
\S Match a non-whitespace character
\d Match a digit character
\D Match a non-digit character
\b Match a word boundary
\B Match a non-(word boundary)
\A Match only at beginning of string
\Z Match only at end of string, or before newline at the end
\z Match only at end of string
\G Match only where previous m//g left off (works only with /g)
```



## 参考资料



  

  

  

  



