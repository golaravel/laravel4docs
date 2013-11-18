# 错误 & 日志

- [错误详情](#error-detail)
- [处理错误](#handling-errors)
- [HTTP异常](#http-exceptions)
- [处理404错误](#handling-404-errors)
- [日志](#logging)

<a name="error-detail"></a>
## 错误详情

默认情况下，应用中会默认输出错误详情。这就是说当有错误发生时，你将看到一个详细的堆栈轨迹（stack trace）和错误信息页面。你可以在`app/config/app.php`文件中将`debug`配置选项设为`false`来关闭它。**强烈建议在生产环境中关闭该选项。**

<a name="handling-errors"></a>
## 处理错误

默认情况下，`app/start/global.php`文件中包含了一个用于处理所有异常的处理器：

	App::error(function(Exception $exception)
	{
		Log::error($exception);
	});

这是一个最基本错误处理器。当然，你也可以自己定义更多的错误处理器。框架会根据你在定义错误处理器时指定的异常类型（type-hint）来分别调用。例如，你定义一个仅处理`RuntimeException`异常的处理器：

	App::error(function(RuntimeException $exception)
	{
		// Handle the exception...
	});

如果异常处理器返回一个response，此response将会发送给浏览器，其他异常处理器将不会再被调用：

	App::error(function(InvalidUserException $exception)
	{
		Log::error($exception);

		return 'Sorry! Something is wrong with this account!';
	});

若要监听PHP中的致命错误（fatal error），可以使用`App::fatal`方法：

	App::fatal(function($exception)
	{
		//
	});

如果同时有多个异常处理器，应该先定义最通用的，然后定义最具体的异常处理器。例如，处理所有 `Exception` 类型的异常处理器应当在处理 `Illuminate\Encryption\DecryptException` 类型的异常处理器之前。

<a name="http-exceptions"></a>
## HTTP 异常

HTTP异常一般发生在用户请求服务器的过程中。可能是页面未找到错误（404)、未授权错误（401)或者系统产生的500错误。为返回这些response，可以使用如下方法：

	App::abort(404, 'Page not found');

第一个参数是HTTP状态码，第二个参数是你自定义的针对此状态吗的错误信息。

如果要产生一个401未授权异常，只需如下所示：

	App::abort(401, 'You are not authorized.');

这些异常可以在request生命周期内的任何时间产生。

<a name="handling-404-errors"></a>
## 处理404错误

你可以注册一个错误处理器来处理应用中的所有"404 Not Found"错误，这样就可以返回一个自定义的404错误页面：

	App::missing(function($exception)
	{
		return Response::view('errors.missing', array(), 404);
	});

<a name="logging"></a>
## 日志

Laravel日志工具为强大的[Monolog](http://github.com/seldaek/monolog)提供了一层简单的抽象层。默认情况下，Laravel被配置为每天为应用创建单独的日志文件，这些文件存放在`app/storage/logs`目录下。你可以通过如下方法输出log：

	Log::info('This is some useful information.');

	Log::warning('Something could be going wrong.');

	Log::error('Something is really going wrong.');

日志工具提供了7个日志级别，参见[RFC 5424](http://tools.ietf.org/html/rfc5424)：**debug**、**info**、**notice**、**warning**、**error**、**critical** 和 **alert**。

还可以像Log方法传递一个包含额外数据的数组：

	Log::info('Log message', array('context' => 'Other helpful information'));

Monolog提供了多种处理日志的功能。如果需要，你可以通过如下方法获取Laravel内部所使用的Monolog实例：

	$monolog = Log::getMonolog();

你还可以注册一个事件，用以获取所有传输到日志的信息：

**注册一个日志监听器**

	Log::listen(function($level, $message, $context)
	{
		//
	});

译者：王赛  [（Bootstrap中文网）](http://www.bootcss.com)
