# 对框架进行扩展

- [介绍](#introduction)
- [管理者 & 工厂](#managers-and-factories)
- [缓存](#cache)
- [Session会话](#session)
- [用户验证](#authentication)
- [基于IoC的扩展](#ioc-based-extension)
- [请求(Request)相关扩展](#request-extension)

<a name="introduction"></a>
## 介绍
Laravel 为你提供了很多可扩展的地方, 通过它们你可以定制框架中一些核心组件的行为,甚至对这些核心组件进行全部替换. 例如, 哈希组件是在 "HaserInterface" 接口中被定义的,你可以根据自己应用程序的需求来选择是否实现它. 你也可以扩展 "Request" 对象来添加专属你的"帮助"方法.你甚至可以添加全新的 用户验证,缓存以及会话(Session) 驱动!

Laravel 的组件通常以两种方式进行扩展:在IoC容器中绑定新的实现,或者为一个扩展注册一个 "Manager" 类,这种方式运用了设计模式中"工厂"的理念.在这一章中,我们会探索扩展框架的各种方法以及查看一些必需的实现代码.

> **注意:** 
请记住,Laravel 组件的扩展通常是以一下两种方法的其中一种实现的:IoC绑定和 "Manager" 类. Manager类充当实现工厂模式的作用,它负责缓存、会话(Session)等基本驱动的实例化工作.

<a name="managers-and-factories"></a>
## 管理者 & 工厂

Laravel 有几个 "Manager" 类 , 用来管理一些基本驱动组件的创建工作. 包括 缓存、会话(Session)、用户验证以及队列组件。"Manager"类负责根据应用程序中相关配置来创建一个驱动实现。例如，"CacheManager"类可以创建 <a href="http://www.php.net/apc">APC</a>、Memcached、Native 以及其他各种缓存驱动的实现。
每一个Manager类都包含有一个"extend"方法,通过它可以轻松的将新的驱动解决功能注入到Manager中. 下面,我们会通过列举向这些Manager中注入定制驱动的例子来依次对它们进行说明.


> **注意:**
花一些时间来探索Laravel中各种不同的"Manager"类,例如,"CacheManager"以及"SessionManager". 通过阅读这些类,可以让你对Laravel的底层实现有一个更加彻底的了解. 所有的"Manager"类都继承了"Illuminate\Support\Manager"基类, 这个基类为每一个"Manager"类提供了一些有用的,常见的功能.
 

<a name="cache"></a>
## 缓存

要扩展Laravel的缓存机制,我们需要使用"CacheManager"的"extend"方法来为"manager"绑定一个定制的驱动解析器,这个解析器在所有的"manager"中都是通用的.例如,注册一个新的名叫"mongo"的缓存驱动,我们需要做一下操作:


	Cache::extend('mongo', function($app)
	{
		// Return Illuminate\Cache\Repository instance...
	});
传入"extend"方法中的第一个参数是这个驱动的名字.这个会与你在"app/config/cache.php"文件中的"driver"选项相对应。第二个参数是一个返回"Illuminate\Cache\Repository"实例的闭包。 这闭包会传入参数"$app", 它是"Illuminate\Foundation\Application" 的一个实例而且是一个IoC容器.

为了创建我们定制的缓存驱动,我们首先应该实现"Illuminate\Cache\StoreInterface"接口. 因此,我们的 MongDB 缓存的实现应该是这样的:

	class MongoStore implements Illuminate\Cache\StoreInterface {

		public function get($key) {}
		public function put($key, $value, $minutes) {}
		public function increment($key, $value = 1) {}
		public function decrement($key, $value = 1) {}
		public function forever($key, $value) {}
		public function forget($key) {}
		public function flush() {}

	}

我们要用一个MongoDB连接来实现所有这些方法.一旦完成了这些实现,我们就完成了定制驱动的注册.

	use Illuminate\Cache\Repository;

	Cache::extend('mongo', function($app)
	{
		return new Repository(new MongoStore);
	});

正如你在上面的例子中所看到的,你会在创建定制缓存驱动时使用到"Illuminate\Cache\Repository"基类.通常情况下,不需要自己创建"Repository"类.

如果你不知道将定制的缓存驱动代码放在哪里,那么可以考虑将它们放在<a href="https://packagist.org/">Packagist</a>中,或者你可以在应用程序的主目录下创建一个"Extension"命名空间.例如,如果你的应用程序叫"Snappy",你可以将你的缓存扩展放在"app/Snappy/Extensions/MongoStore.php"中. 请记住,Laravel对于应用程序的结构没有严格的限制,你可以自由地根据自己的选择来组织你的应用程序结构.


> **注意:** 
当你不知道将代码放在哪里时,请回想一下"service provider" .  我们已经讨论过,利用"service provider"来组织你的框架扩展是组织代码的最好方式.


<a name="session"></a>
## 会话 Session

为Laravel扩展一个定制的Session驱动跟扩展一个缓存系统一样简单.同样,我们需要用"extend"方法来注册定制的驱动:

	Session::extend('mongo', function($app)
	{
		// Return implementation of SessionHandlerInterface
	});

请注意,定制的缓存驱动需要实现"SessionHandlerInterface"接口.这个接口在PHP5.4+core中.如果你正在使用PHP5.3,Laravel会帮你定义这个接口让你的系统可以向前兼容. 我们只需要实现这个接口中的一些简单的方法. 下面是一个简化的MongoDB实现:

	class MongoHandler implements SessionHandlerInterface {

		public function open($savePath, $sessionName) {}
		public function close() {}
		public function read($sessionId) {}
		public function write($sessionId, $data) {}
		public function destroy($sessionId) {}
		public function gc($lifetime) {}

	}	

上面这些方法不像在"StoreInterface"缓存接口中的方法一样让人容易理解.下面让我们快速的了解一下每一个方法的功能:

- "open"方法通常被用于基于文件的Session存储系统.因为Laravel自带了PHP原生文件存储session的session驱动.因此,你不需要在这个方法中添加任何代码.事实上 PHP强制要求我们实现这个不需要添加任何代码的方法是实在一个糟糕的接口设计(在后面的内容中会讨论这一点).
-"close"方法也像"open"方法一样,通常是可以被忽略的.大多数驱动不需要这个方法.
-"read"方法会返回一个与传入的"$sessionId"相关联的字符串形式的Session数据.将驱动中的session数据取出时,我们不需要做任何的序列化和转码的工作.因为Laravel会帮你处理这些.
-
- The `write` method should write the given `$data` string associated with the `$sessionId` to some persistent storage system, such as MongoDB, Dynamo, etc.
- The `destroy` method should remove the data associated with the `$sessionId` from persistent storage.
- The `gc` method should destroy all session data that is older than the given `$lifetime`, which is a UNIX timestamp. For self-expiring systems like Memcached and Redis, this method may be left empty.

Once the `SessionHandlerInterface` has been implemented, we are ready to register it with the Session manager:

	Session::extend('mongo', function($app)
	{
		return new MongoHandler;
	});

Once the session driver has been registered, we may use the `mongo` driver in our `app/config/session.php` configuration file.

> **Note:** Remember, if you write a custom session handler, share it on Packagist!

<a name="authentication"></a>
## Authentication

Authentication may be extended the same way as the cache and session facilities. Again, we will use the `extend` method we have become familiar with:

	Auth::extend('riak', function($app)
	{
		// Return implementation of Illuminate\Auth\UserProviderInterface
	});

The `UserProviderInterface` implementations are only responsible for fetching a `UserInterface` implementation out of a persistent storage system, such as MySQL, Riak, etc. These two interfaces allow the Laravel authentication mechanisms to continue functioning regardless of how the user data is stored or what type of class is used to represent it.

Let's take a look at the `UserProviderInterface`:

	interface UserProviderInterface {

		public function retrieveById($identifier);
		public function retrieveByCredentials(array $credentials);
		public function validateCredentials(UserInterface $user, array $credentials);

	}

The `retrieveById` function typically receives a numeric key representing the user, such as an auto-incrementing ID from a MySQL database. The `UserInterface` implementation matching the ID should be retrieved and returned by the method.

The `retrieveByCredentials` method receives the array of credentials passed to the `Auth::attempt` method when attempting to sign into an application. The method should then "query" the underlying persistent storage for the user matching those credentials. Typically, this method will run a query with a "where" condition on `$credentails['username']`. **This method should not attempt to do any password validation or authentication.**

The `validateCredentials` method should compare the given `$user` with the `$credentials` to authenticate the user. For example, this method might compare the `$user->getAuthPassword()` string to a `Hash::make` of `$credentials['password']`.

Now that we have explored each of the methods on the `UserProviderInterface`, let's take a look at the `UserInterface`. Remember, the provider should return implementations of this interface from the `retrieveById` and `retrieveByCredentials` methods:

	interface UserInterface {

		public function getAuthIdentifier();
		public function getAuthPassword();

	}

This interface is simple. The `getAuthIdentifier` method should return the "primary key" of the user. In a MySQL back-end, again, this would be the auto-incrementing primary key. The `getAuthPassword` should return the user's hashed password. This interface allows the authentication system to work with any User class, regardless of what ORM or storage abstraction layer you are using. By default, Laravel includes a `User` class in the `app/models` directory which implements this interface, so you may consult this class for an implementation example.

Finally, once we have implemented the `UserProviderInterface`, we are ready to register our extension with the `Auth` facade:

	Auth::extend('riak', function($app)
	{
		return new RiakUserProvider($app['riak.connection']);
	});

After you have registered the driver with the `extend` method, you switch to the new driver in your `app/config/auth.php` configuration file.

<a name="ioc-based-extension"></a>
## IoC Based Extension

Almost every service provider included with the Laravel framework binds objects into the IoC container. You can find a list of your application's service providers in the `app/config/app.php` configuration file. As you have time, you should skim through each of these provider's source code. By doing so, you will gain a much better understanding of what each provider adds to the framework, as well as what keys are used to bind various services into the IoC container.

For example, the `PaginationServiceProvider` binds a `paginator` key into the IoC container, which resolves into a `Illuminate\Pagination\Environment` instance. You can easily extend and override this class within your own application by overriding this IoC binding. For example, you could create a class that extend the base `Environment`:

	namespace Snappy\Extensions\Pagination;

	class Environment extends \Illuminate\Pagination\Environment {

		//

	}

Once you have created your class extension, you may create a new `SnappyPaginationProvider` service provider class which overrides the paginator in its `boot` method:

	class SnappyPaginationProvider extends PaginationServiceProvider {

		public function boot()
		{
			App::bind('paginator', function()
			{
				return new Snappy\Extensions\Pagination\Environment;
			});

			parent::boot();
		}

	}

Note that this class extends the `PaginationServiceProvider`, not the default `ServiceProvider` base class. Once you have extended the service provider, swap out the `PaginationServiceProvider` in your `app/config/app.php` configuration file with the name of your extended provider.

This is the general method of extending any core class that is bound in the container. Essentially every core class is bound in the container in this fashion, and can be overridden. Again, reading through the included framework service providers will familiarize you with where various classes are bound into the container, and what keys they are bound by. This is a great way to learn more about how Laravel is put together.

<a name="request-extension"></a>
## Request Extension

Because it is such a foundational piece of the framework and is instantiated very early in the request cycle, extending the `Request` class works a little differently than the previous examples.

First, extend the class like normal:

	<?php namespace QuickBill\Extensions;

	class Request extends \Illuminate\Http\Request {

		// Custom, helpful methods here...

	}

Once you have extended the class, open the `bootstrap/start.php` file. This file is one of the very first files to be included on each request to your application. Note that the first action performed is the creation of the Laravel `$app` instance:

	$app = new \Illuminate\Foundation\Application;

When a new application instance is created, it will create a new `Illuminate\Http\Request` instance and bind it to the IoC container using the `request` key. So, we need a way to specify a custom class that should be used as the "default" request type, right? And, thankfully, the `requestClass` method on the application instance does just this! So, we can add this line at the very top of our `bootstrap/start.php` file:

	use Illuminate\Foundation\Application;

	Application::requestClass('QuickBill\Extensions\Request');

Once you have specified the custom request class, Laravel will use this class anytime it creates a `Request` instance, conveniently allowing you to always have an instance of your custom request class available, even in unit tests!
