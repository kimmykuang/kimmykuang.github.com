---
layout: post
title: Laravel记录运行时sql的中间件
category: blog
tag: laravel, sql, 中间件
description: 一个可以将laravel框架中运行时sql记录获取然后记录日志的中间件，简单易用
---

###写一个Laravel框架的中间件来记录运行时产生的所有SQL语句

####1.新建SQLMiddleware

	php artisan make:middleware SQLMiddleware
	
####2.获取DB记录并记录日志

	<?php namespace App\Http\Middleware;

	use Closure;
	use DB;
	use Log;

	class SQLMiddleware {
	
		/**
		 * Handle an incoming request.
		 *
		 * @param  \Illuminate\Http\Request  $request
		 * @param  \Closure  $next
		 * @return mixed
		 */
		public function handle($request, Closure $next)
		{
			//before middleware, 开启DB Log，可以指定运行环境下开启
			if(env('APP_ENV', 'production') == 'dev') {
				DB::enableQueryLog();
			}
	
			//do the other middleware and logic process
			$response = $next($request);
	
			//after middleware, 将所有数据库sql显示到日志中
			if(env('APP_ENV', 'production') == 'dev') {
				$queries = DB::getQueryLog();
				foreach ($queries as $query) {
					$sql = $query['query'];       //查询语句sql
					$params = $query['bindings']; //查询参数
					$sql = str_replace('?', "'%s'", $sql);
					array_unshift($params, $sql);
					Log::debug(call_user_func_array('sprintf', $params));
				}
			}
			return $response;
		}
	
	}

####3.在```app/Http/Kernel.php```文件中注册中间件

	protected $middleware = [
		'App\Http\Middleware\SQLMiddleware',
	];

####4.请求接口后查询日志

![请求日志](images/laravel-sql-log-middleware.png)