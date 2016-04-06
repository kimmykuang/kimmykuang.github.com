---
layout: post
title: Laravel学习笔记 - 启动过程分析
category: blog
tag: laravel, 源码解析, 学习笔记
description: 公司的后端技术栈中有用到Laravel (5.0.14)，通过分析框架的源代码，可以了解这个框架的设计思想和实现技巧，这样更有助于写出“最佳laravel实践”的代码。
---

## laravel框架学习

### 一. laravel官方文档系统架构篇阅读

#### serviceprovider 服务提供者

serviceprovider是一份“配置”，所有laravel的核心服务或自定义服务都在serviceprovider中被注册，框架启动一开始，就会遍历provider中注册的服务，挨个调用```resgiter```方法将服务注册到容器中，接着```boot```方法被调用，相当于服务初始化，这样每个核心组件在这里经过了一系列的初始化后就可以供后续调用了；当然这些服务是可以延迟加载的。
providers一般默认被定义在config/app.php的providers数组中：

    ```
        /*
         * Laravel Framework Service Providers...
         */
        'Illuminate\Foundation\Providers\ArtisanServiceProvider',
        'Illuminate\Auth\AuthServiceProvider',
        'Illuminate\Bus\BusServiceProvider',
        'Illuminate\Cache\CacheServiceProvider',
        'Illuminate\Foundation\Providers\ConsoleSupportServiceProvider',
        'Illuminate\Routing\ControllerServiceProvider',
        'Illuminate\Cookie\CookieServiceProvider',
        'Illuminate\Database\DatabaseServiceProvider',
        'Illuminate\Encryption\EncryptionServiceProvider',
        'Illuminate\Filesystem\FilesystemServiceProvider',
        'Illuminate\Foundation\Providers\FoundationServiceProvider',
        'Illuminate\Hashing\HashServiceProvider',
        'Illuminate\Mail\MailServiceProvider',
        'Illuminate\Pagination\PaginationServiceProvider',
        'Illuminate\Pipeline\PipelineServiceProvider',
        'Illuminate\Queue\QueueServiceProvider',
        'Illuminate\Redis\RedisServiceProvider',
        'Illuminate\Auth\Passwords\PasswordResetServiceProvider',
        'Illuminate\Session\SessionServiceProvider',
        'Illuminate\Translation\TranslationServiceProvider',
        'Illuminate\Validation\ValidationServiceProvider',
        'Illuminate\View\ViewServiceProvider',
    ```
在serviceprovider里，总是通过```$this->app```这个实例变量来访问容器

Q: 这个容器是不是```bootstrap/app.php```初始化的那个application实例？
A：是的

#### Ioc容器 服务容器

容器是laravel的核心，很多用到的各种功能模块比如路由，数据库orm，请求，相应和缓存等模块都是各自独立的，通过服务绑定（serviceprovider的功能在这里体现），注入到了容器当中。
容器在哪儿被初始化？在```public/index.php```整个框架的唯一入口文件中引用了```bootstrap/app.php```，这里初始化了app这个容器：

    $app = new Illuminate\Foundation\Application(
        realpath(__DIR__.'/../')
    );

其实粗略的来看，可以把容器想象成一个数组，数组里有很多item，key是服务的名字，value就是服务的实例，这些实例可以在全局范围内使用，因为容器本身就是一个全局范围的变量。

    $this->app = [
        'cache' => '\path\to\cache',
        .....
    ];

Q: 容器和serviceprovder是什么关系？
A: 服务容器包含了所有核心或自定义的service，而serviceprovider的作用就是1：提供服务（如初始化一个单例），2：绑定至容器中（将单例注册到容器数组里）

参考文章[Laravel 服务容器实例教程 —— 深入理解控制反转（IoC）和依赖注入（DI）](http://laravelacademy.org/post/769.html)

#### facedes 门面

facedes我更愿意称为绑定至容器中类的快捷方式。
举个cache的例子，当使用缓存时可以很方便地直接```use Cache```就可以用了：```cache::get('key')```，这里的cache就是一个facede（门面），其实是在```config/app.php```中被定义的。

    'aliases' => [
        ...
        'Cache'     => 'Illuminate\Support\Facades\Cache',
        ...
    ],

打开```Illuminate\Support\Facades\Cache```，里面并没有什么静态方法get/set/put，而是继承了```Facade```这个类，实现了```getFacadeAccessor```这个方法：

    /**
     * Get the registered name of the component.
     *
     * @return string
     */
    protected static function getFacadeAccessor()
    {
        return 'cache';
    }

这里简单地return了一个字符串cache，其实这个cache就是```CacheServiceProvider```这个缓存服务提供者注册到容器里的名字，简单来说就是容器里数组的key；那么根据根据这个key，就能找到真实地注册的类，```CacheServiceProvider```在providers数组里被定义：```Illuminate\Cache\CacheServiceProvider```，打开这个文件查看里面的```register```方法：

    /**
     * Register the service provider.
     *
     * @return void
     */
    public function register()
    {
        $this->app->singleton('cache', function($app)
        {
            return new CacheManager($app);
        });
        $this->app->singleton('cache.store', function($app)
        {
            return $app['cache']->driver();
        });
        $this->app->singleton('memcached.connector', function()
        {
            return new MemcachedConnector;
        });
        $this->registerCommands();
    }

就能看到这里将一个单例cache注册到了application容器中，所以使用cache这个facede其实就是调用了```CacheManager```这个类的实例。

###二. laravel框架启动流程

####入口文件

框架入口文件```public/index.php```：

    //引入自动加载，主要实现依靠composer
    require __DIR__.'/../bootstrap/autoload.php';
    
    //初始化服务容器
    $app = require_once __DIR__.'/../bootstrap/app.php';
    
    $kernel = $app->make('Illuminate\Contracts\Http\Kernel');
    $response = $kernel->handle(
        $request = Illuminate\Http\Request::capture()
    );
    $response->send();
    $kernel->terminate($request, $response);

####自动加载

看下```bootstrap/autoload.php```做了些什么事情：

    //定义框架开始时间
    define('LARAVEL_START', microtime(true));
    
    //加载vendor中的autoload
    require __DIR__.'/../vendor/autoload.php';
    
    //vendor/compiled.php通过命令php artisan optimize生成，将部分（全部？）vendor中的类文件提前加载到了一个文件中
    //php artisan optimize —force，debug模式下不会生成
    $compiledPath = __DIR__.'/../vendor/compiled.php';
    
    if (file_exists($compiledPath))
    {
        require $compiledPath;
    }
    
```vendor/autoload.php```

    //composer autoload
    require_once __DIR__ . '/composer' . '/autoload_real.php';
    //名字看上去很厉害，每次vendor重新拉取都会变化，应该是为了避免命名冲突的一种机制
    return ComposerAutoloaderInit7572fca069df827dd9dcf43b2f592e28::getLoader();
    
看下```vendor/composer/autoload_real.php```里做的事情，最主要的函数是```ComposerAutoloaderInit7572fca069df827dd9dcf43b2f592e28::getLoader```这个方法：

    public static function getLoader()
    {
        if (null !== self::$loader) {
            return self::$loader;
        }
        
        //spl_autoload_register是php spl系列函数提供的一种注册自动加载函数的机制，一般框架要么用这个函数要么使用__autoload来实现自动加载
        //\Composer\Autoload\ClassLoader()应该是自动加载的具体实现，根据下面提供的文件注册一些自动加载路径，具体代码还未看
        //这里先注册了\Composer\Autoload\ClassLoader()后又unregister掉，具体还没看懂
        spl_autoload_register(array('ComposerAutoloaderInit7572fca069df827dd9dcf43b2f592e28', 'loadClassLoader'), true, true);
        self::$loader = $loader = new \Composer\Autoload\ClassLoader();
        spl_autoload_unregister(array('ComposerAutoloaderInit7572fca069df827dd9dcf43b2f592e28', 'loadClassLoader'));
        //这里应该是注册了符合psr-0规范的目录
        $map = require __DIR__ . '/autoload_namespaces.php';
        foreach ($map as $namespace => $path) {
            $loader->set($namespace, $path);
        }
        //这里注册符合psr-4规范的目录
        $map = require __DIR__ . '/autoload_psr4.php';
        foreach ($map as $namespace => $path) {
            $loader->setPsr4($namespace, $path);
        }
        //这里是classmap，打开来就是一个大数组里面都是类名和具体文件的对应，既不符合psr-0也不符合psr-4规范的类通过这里声明后加载进来
        $classMap = require __DIR__ . '/autoload_classmap.php';
        if ($classMap) {
            $loader->addClassMap($classMap);
        }
        //根据上面定义的三种加载类型，重新注册自动加载
        $loader->register(true);
        //手动加载别的文件，例如helpers文件等
        $includeFiles = require __DIR__ . '/autoload_files.php';
        foreach ($includeFiles as $file) {
            composerRequire7572fca069df827dd9dcf43b2f592e28($file);
        }
        return $loader;
    }
    
####自定义加载
在```composer.json```文件中声明：

    "autoload": {
        //class map方式
        "classmap": [
            "database"
        ],
        //psr-4
        "psr-4": {
            "App\\": "app/"
        },
        //psr-0
        "psr-0": {
            "Carbon": "path/to/carbon"
        },
        //include files
        "files": [
            "app/Helpers"
        ]
    },
    
参考文档：
[PHP-FIG/PSR-4-autoloader-cn.md](https://github.com/PizzaLiu/PHP-FIG/blob/master/PSR-4-autoloader-cn.md)
[composer基本用法](http://docs.phpcomposer.com/01-basic-usage.html)
    
####初始化服务容器
```bootstrap/app.php```文件初始化了服务容器：

    //初始化服务容器application
    $app = new Illuminate\Foundation\Application(
        //设置base路径
        realpath(__DIR__.'/../')
    );
    
    //设置单例http kernel，一般用来响应http request
    $app->singleton(
        'Illuminate\Contracts\Http\Kernel',
        'App\Http\Kernel'
    );
    //设置单例console kernel，一般用来响应cli request，如queue等
    $app->singleton(
        'Illuminate\Contracts\Console\Kernel',
        'App\Console\Kernel'
    );
    //设置exception handler单例
    $app->singleton(
        'Illuminate\Contracts\Debug\ExceptionHandler',
        'App\Exceptions\Handler'
    );
    
    //返回初始化的容器
    return $app;
    
```Illuminate\Foundation\Application.php```代码挺长，继承自```Illuminate\Container\Container.php```

    /**
     * Create a new Illuminate application instance.
     *
     * @param  string|null  $basePath
     * @return void
     */
    public function __construct($basePath = null)
    {
        //初始化空的容器
        $this->registerBaseBindings();
        //注册最基本的service provider
        $this->registerBaseServiceProviders();
        $this->registerCoreContainerAliases();
        if ($basePath) $this->setBasePath($basePath);
    }
    
    /**
     * Register the basic bindings into the container.
     *
     * @return void
     */
    protected function registerBaseBindings()
    {
        //初始化对象：一个空的容器
        static::setInstance($this);
        //顾名思义就是产生一个实例在当前这个空的容器中，实例的名字叫app，值就是容器本身
        $this->instance('app', $this);
        //将key 'Illuminate\Container\Container'也指向当前容器，作用还未研究到
        $this->instance('Illuminate\Container\Container', $this);
    }
    
    /**
     * Register all of the base service providers.
     *
     * @return void
     */
    protected function registerBaseServiceProviders()
    {
        //向服务容器注册一个key为events的单例，具体请见Illuminate\Events\EventServiceProvider
        $this->register(new EventServiceProvider($this));
        //见Illuminate\Routing\RoutingServiceProvider，向服务容器注册了四个东西：
        //分别key是router，url，redirect和Illuminate\Contracts\Routing\ResponseFactory（这个是供自定义response宏的吗？）
        $this->register(new RoutingServiceProvider($this));
    }
    
    /**
     * Register the core class aliases in the container.
     *
     * @return void
     */
    public function registerCoreContainerAliases()
    {
        $aliases = array(
            'app'                  => ['Illuminate\Foundation\Application', 'Illuminate\Contracts\Container\Container', 'Illuminate\Contracts\Foundation\Application'],
            'artisan'              => ['Illuminate\Console\Application', 'Illuminate\Contracts\Console\Application'],
            'auth'                 => 'Illuminate\Auth\AuthManager',
            'auth.driver'          => ['Illuminate\Auth\Guard', 'Illuminate\Contracts\Auth\Guard'],
            'auth.password.tokens' => 'Illuminate\Auth\Passwords\TokenRepositoryInterface',
            'blade.compiler'       => 'Illuminate\View\Compilers\BladeCompiler',
            'cache'                => ['Illuminate\Cache\CacheManager', 'Illuminate\Contracts\Cache\Factory'],
            'cache.store'          => ['Illuminate\Cache\Repository', 'Illuminate\Contracts\Cache\Repository'],
            'config'               => ['Illuminate\Config\Repository', 'Illuminate\Contracts\Config\Repository'],
            'cookie'               => ['Illuminate\Cookie\CookieJar', 'Illuminate\Contracts\Cookie\Factory', 'Illuminate\Contracts\Cookie\QueueingFactory'],
            'encrypter'            => ['Illuminate\Encryption\Encrypter', 'Illuminate\Contracts\Encryption\Encrypter'],
            'db'                   => 'Illuminate\Database\DatabaseManager',
            'events'               => ['Illuminate\Events\Dispatcher', 'Illuminate\Contracts\Events\Dispatcher'],
            'files'                => 'Illuminate\Filesystem\Filesystem',
            'filesystem'           => ['Illuminate\Filesystem\FilesystemManager', 'Illuminate\Contracts\Filesystem\Factory'],
            'filesystem.disk'      => 'Illuminate\Contracts\Filesystem\Filesystem',
            'filesystem.cloud'     => 'Illuminate\Contracts\Filesystem\Cloud',
            'hash'                 => 'Illuminate\Contracts\Hashing\Hasher',
            'translator'           => ['Illuminate\Translation\Translator', 'Symfony\Component\Translation\TranslatorInterface'],
            'log'                  => ['Illuminate\Log\Writer', 'Illuminate\Contracts\Logging\Log', 'Psr\Log\LoggerInterface'],
            'mailer'               => ['Illuminate\Mail\Mailer', 'Illuminate\Contracts\Mail\Mailer', 'Illuminate\Contracts\Mail\MailQueue'],
            'paginator'            => 'Illuminate\Pagination\Factory',
            'auth.password'        => ['Illuminate\Auth\Passwords\PasswordBroker', 'Illuminate\Contracts\Auth\PasswordBroker'],
            'queue'                => ['Illuminate\Queue\QueueManager', 'Illuminate\Contracts\Queue\Factory', 'Illuminate\Contracts\Queue\Monitor'],
            'queue.connection'     => 'Illuminate\Contracts\Queue\Queue',
            'redirect'             => 'Illuminate\Routing\Redirector',
            'redis'                => ['Illuminate\Redis\Database', 'Illuminate\Contracts\Redis\Database'],
            'request'              => 'Illuminate\Http\Request',
            'router'               => ['Illuminate\Routing\Router', 'Illuminate\Contracts\Routing\Registrar'],
            'session'              => 'Illuminate\Session\SessionManager',
            'session.store'        => ['Illuminate\Session\Store', 'Symfony\Component\HttpFoundation\Session\SessionInterface'],
            'url'                  => ['Illuminate\Routing\UrlGenerator', 'Illuminate\Contracts\Routing\UrlGenerator'],
            'validator'            => ['Illuminate\Validation\Factory', 'Illuminate\Contracts\Validation\Factory'],
            'view'                 => ['Illuminate\View\Factory', 'Illuminate\Contracts\View\Factory'],
        );
        foreach ($aliases as $key => $aliases)
        {
            foreach ((array) $aliases as $alias)
            {
                $this->alias($key, $alias);
            }
        }
    }

```Illuminate\Routing\RoutingServiceProvider```路由ServiceProvider干的事情：

    public function register()
    {
        $this->registerRouter();          //注册路由
        $this->registerUrlGenerator();    //url生成器
        $this->registerRedirector();      //重定向服务
        $this->registerResponseFactory(); //响应宏工厂
    }

####初始化kernel

容器初始化后干的事情，需要初始化kelnel，http kernel的文件在```App\Http\Kernel```：

    /**
     * The application's global HTTP middleware stack.
     *
     * @var array
     */
    //全局http中间件，启动路由前被一一加载
    protected $middleware = [
        'Illuminate\Foundation\Http\Middleware\CheckForMaintenanceMode',
        'Illuminate\Cookie\Middleware\EncryptCookies',
        'Illuminate\Cookie\Middleware\AddQueuedCookiesToResponse',
        'Illuminate\Session\Middleware\StartSession',
        'Illuminate\View\Middleware\ShareErrorsFromSession',
        'App\Http\Middleware\VerifyCsrfToken',
    ];
    /**
     * The application's route middleware.
     *
     * @var array
     */
    //路由中间件，可以在route.php中使用
    protected $routeMiddleware = [
        'auth' => 'App\Http\Middleware\Authenticate',
        'auth.basic' => 'Illuminate\Auth\Middleware\AuthenticateWithBasicAuth',
        'guest' => 'App\Http\Middleware\RedirectIfAuthenticated',
    ];

这里就申明了几个中间件的名字（命名空间），初始化kernel需要调用到handle方法，这个方法在父类```Illuminate/Foundation/Http/Kernel.php```中：

    /**
     * Handle an incoming HTTP request.
     *
     * @param  \Illuminate\Http\Request  $request
     * @return \Illuminate\Http\Response
     */
    public function handle($request)
    {
        try
        {
            //这里是将http请求与router关联的地方
            $response = $this->sendRequestThroughRouter($request);
        }
        catch (Exception $e)
        {
            $this->reportException($e);
            $response = $this->renderException($request, $e);
        }
        $this->app['events']->fire('kernel.handled', [$request, $response]);
        return $response;
    }
    
    /**
     * Send the given request through the middleware / router.
     *
     * @param  \Illuminate\Http\Request  $request
     * @return \Illuminate\Http\Response
     */
    protected function sendRequestThroughRouter($request)
    {
        //容器中注入request实例
        $this->app->instance('request', $request);
        Facade::clearResolvedInstance('request');
        //运行定义的bootstrappers，进行一些诸如环境检测，加载配置之类的操作
        $this->bootstrap();
        //这里没仔细看，应该是把请求分发给了router的dispatcher了
        //这里应该是用了pipeline模式
        return (new Pipeline($this->app))
                    ->send($request)
                    ->through($this->middleware)
                    ->then($this->dispatchToRouter());
    }
    
    /**
     * Get the route dispatcher callback.
     *
     * @return \Closure
     */
    protected function dispatchToRouter()
    {
        return function($request)
        {
            $this->app->instance('request', $request);
            return $this->router->dispatch($request);
        };
    }

```Illuminate/Routing/Router.php```里就是路由讲请求分发的地方：

    public function dispatch(Request $request)
    {
        $this->currentRequest = $request;
        // If no response was returned from the before filter, we will call the proper
        // route instance to get the response. If no route is found a response will
        // still get returned based on why no routes were found for this request.
        $response = $this->callFilter('before', $request);
        if (is_null($response))
        {
            $response = $this->dispatchToRoute($request);
        }
        // Once this route has run and the response has been prepared, we will run the
        // after filter to do any last work on the response or for this application
        // before we will return the response back to the consuming code for use.
        $response = $this->prepareResponse($request, $response);
        $this->callFilter('after', $request, $response);
        return $response;
    }
    
这里也有个before/after操作，类似yii的beforeaction&afteraction(疑似已经不支持了，待确认)
