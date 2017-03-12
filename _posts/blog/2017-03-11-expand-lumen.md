---
layout: post
title: 扩展Lumen框架
category: blog
tag: Lumen, Laravel, 框架, php
description: Lumen框架因为精简的关系某些功能是没有实现的，个人用的时候感觉不是很爽就做了点基本扩展。
---

>Lumen使用版本: Laravel Framework Lumen (5.4.5) (Laravel Components 5.4.*)


### 1.Lumen的自定义config目录

看了下`Laravel\Lumen\Application`的代码，`Application::registerConfigBindings`这个方法在容器中注册了config的单例实现，但是取config配置的方式却跟Laravel不太一样，Lumen中是通过以下方法获取配置文件的: 

```php

	/**
     * Get the path to the given configuration file.
     *
     * If no name is provided, then we'll return the path to the config folder.
     *
     * @param  string|null  $name
     * @return string
     */
    public function getConfigurationPath($name = null)
    {
        if (! $name) {
            $appConfigDir = $this->basePath('config').'/';

            if (file_exists($appConfigDir)) {
                return $appConfigDir;
            } elseif (file_exists($path = __DIR__.'/../config/')) {
                return $path;
            }
        } else {
            $appConfigPath = $this->basePath('config').'/'.$name.'.php';

            if (file_exists($appConfigPath)) {
                return $appConfigPath;
            } elseif (file_exists($path = __DIR__.'/../config/'.$name.'.php')) {
                return $path;
            }
        }
    }
```

如果传了参数则首先尝试从`[your_project_root]/config`目录下寻找{$name}.php，找不到再从`vendor/laravel/lumen-framework/config`目录下找，也就是说这里是不支持目录嵌套的。

可以看下[Laravel的config加载实现](https://github.com/laravel/framework/blob/5.0/src/Illuminate/Foundation/Bootstrap/LoadConfiguration.php), Laravel是在Application boot的时候执行了`LoadConfiguration::bootstrap`，我看了下Lumen好像都把几个component booter都去掉了。

我觉得不能嵌套config目录不太好，就写了一个`app/Providers/ConfigServiceProvider.php`，将其在`bootstrap/app.php`中注册一下: `$app->register(App\Providers\ConfigServiceProvider::class);`，下面是这个ServiceProvider的实现: 

```php
<?php
namespace App\Providers;

use Illuminate\Support\ServiceProvider;
use Symfony\Component\Finder\Finder;
use Symfony\Component\Finder\SplFileInfo;
use Illuminate\Contracts\Container\Container;

class ConfigServiceProvider extends ServiceProvider
{
    /**
     * 在容器中注册ConfigServiceProvider，提供配置的度、取、判断是否存在等方法
     *
     * @return void
     */
    public function register()
    {
        //Application@registerConfigBindings has already registered `config` as Illuminate\Config\Repository instance
    }

    /**
     * 在ServiceProviders都注册完成后加载config文件中的配置项(数组)至内存
     * @return [type] [description]
     */
    public function boot()
    {
        foreach ($this->getConfigFiles($this->app) as $key => $path) {
            $this->app->make('config')->set($key, require $path);
        }
    }

    /**
     * 获取配置文件
     * @param  Container $app [description]
     * @return [type]           [description]
     */
    protected function getConfigFiles(Container $app)
    {
        $files = [];

        foreach (Finder::create()->files()->name('*.php')->in($app->getConfigurationPath()) as $file)
        {
            //获取配置文件对应的key
            $key = $this->getConfigNestingKey($file, $app);
            $files[$key] = $file->getRealPath();
        }

        return $files;
    }

    /**
     * 获取配置文件的嵌套目录,将config/aa/bb/cc.php的嵌套目录返回形如aa.bb.cc作为key
     * @param  SplFileInfo $file [description]
     * @param  Container $app  [description]
     * @return [type]            [description]
     */
    protected function getConfigNestingKey(SplFileInfo $file, Container $app)
    {
        //应用的配置目录,加trim函数是为了确保$appConfigPath<=$directory
        $appConfigPath = trim($app->getConfigurationPath(), DIRECTORY_SEPARATOR);
        //当前配置文件的目录
        $directory = dirname($file->getRealPath());
        //补充key,配置文件的key一般是: config/app.php 则 app => require(config/app.php),配置文件会有目录嵌套
        $key = basename($file->getRealPath(), '.php');
        //判断是否有目录嵌套
        if($nesting = trim(str_replace($appConfigPath, '', $directory), DIRECTORY_SEPARATOR)) {
            $nesting = str_replace(DIRECTORY_SEPARATOR, '.', $nesting);
            $key = $nesting . '.' . $key;
        }
        return $key;
    }
}
```

其实这段代码跟Laravel的LoadConfiguration实现类似，都是使用了`Symfony\Component\Finder\Finder`这个很棒的组件，可以按文件名正则来遍历指定目录下的所有文件，支持嵌套目录；再将遍历出来的文件路径与config_path做比较，比如: 

```
config/test/test/aa.php与config/目录做比较

得到test/test/aa.php，

再做一些字符串的替换等操作，最终形成test.test.aa => config/test/test/aa.php的数组

这样后面就可以通过 app()->make('config')->get('test.test.aa') 的方式来取值了

```


### 2.我的Facade去哪儿了

Lumen在`bootstrap/app.php`中注释了两个地方:


```php
$app->withEloquent();

$app->withFacades();

```

可以把注释去掉开启Eloquent和Facade的使用，但是很明显压根没有`config/app.php`中的`alias`来自定义Facade，我一开始以为Lumen把这个功能去掉了，看了下代码才发现是可以自定义的，Lumen不会做这么没有扩展性的事情，只不过在开启Facade的时候需要额外多传一个参数，将自定义的`alias`传过去。

下面是Lumen的withFacades实现，也是在Application中:

```php

	/**
     * Register the facades for the application.
     *
     * @param  bool  $aliases
     * @param  array $userAliases
     * @return void
     */
    public function withFacades($aliases = true, $userAliases = [])
    {
        Facade::setFacadeApplication($this);

        if ($aliases) {
            $this->withAliases($userAliases);
        }
    }

    /**
     * Register the aliases for the application.
     *
     * @param  array  $userAliases
     * @return void
     */
    public function withAliases($userAliases = [])
    {
        $defaults = [
            'Illuminate\Support\Facades\Auth' => 'Auth',
            'Illuminate\Support\Facades\Cache' => 'Cache',
            'Illuminate\Support\Facades\DB' => 'DB',
            'Illuminate\Support\Facades\Event' => 'Event',
            'Illuminate\Support\Facades\Gate' => 'Gate',
            'Illuminate\Support\Facades\Log' => 'Log',
            'Illuminate\Support\Facades\Queue' => 'Queue',
            'Illuminate\Support\Facades\Schema' => 'Schema',
            'Illuminate\Support\Facades\URL' => 'URL',
            'Illuminate\Support\Facades\Validator' => 'Validator',
        ];

        if (! static::$aliasesRegistered) {
            static::$aliasesRegistered = true;

            $merged = array_merge($defaults, $userAliases);

            foreach ($merged as $original => $alias) {
                class_alias($original, $alias);
            }
        }
    }
```

可以看到需要传一个`$userAliases`过去，就会合并alias了，当然自定义的Facade也要继承好`Illuminate\Support\Facades\Facade`这个抽象类并且实现`getFacadeAccessor`方法，否则会找不到需要class_alias的类的。

以刚才的ConfigServiceProvider为例，实现`Config`这个Facade吧。

首先，修改`bootstrap/app.php`文件，把withFacade()移到register serviceprovider的代码后面，保证在那些ServiceProvider该注册和boot都完成了。

然后新建`config/app.php`配置:

```php
return [
	/**
     * Facades的别名配置
     */
    'alias' => [
        App\Support\Facades\Config::class => 'Config',
    ],
];
```

然后启动Facade代码改成: `$app->withFacades(true, $app->make('config')->get('app.alias'));`。

接着新建`app/Support/Facades/Config.php`:

```php
<?php
namespace App\Support\Facades;

use Illuminate\Support\Facades\Facade;

class Config extends Facade {

    public static function getFacadeAccessor()
    {
        return 'config';
    }
}
```

这样就行了，可以直接使用`Config::get('app.key')`这种方式来取配置了。

后面需要自定义Facade也跟上面类似就行了，十分简单。

