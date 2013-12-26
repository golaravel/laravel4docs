# 队列

- [配置](#configuration)
- [基础用法](#basic-usage)
- [队列闭包](#queueing-closures)
- [运行队列监听器](#running-the-queue-listener)
- [推送队列](#push-queues)

<a name="configuration"></a>
## 配置

Laravel的队列组件为许多队列服务提供了统一的API接口。队列服务让你可以异步处理一个耗时任务，比如延迟发送一封邮件，从而大大加快了应用的Web请求处理速度。

队列的设置信息储存在 `app/config/queue.php` 文件中。在这个文件中你可以找到所有目前支持的队列驱动的连接设置，包括[Beanstalkd](http://kr.github.com/beanstalkd)、[IronMQ](http://iron.io)、[Amazon SQS](http://aws.amazon.com/sqs)和同步处理（本地环境使用）驱动。

下面是相应队列驱动所需的依赖性包：

- Beanstalkd: `pda/pheanstalk`
- Amazon SQS: `aws/aws-sdk-php`
- IronMQ: `iron-io/iron_mq`

<a name="basic-usage"></a>
## 基础用法

使用 `Queue::push` 方法推送一个新任务到队列中：

**推送一个任务到队列中**

	Queue::push('SendEmail', array('message' => $message));

`push` 方法的第一个参数是用来处理任务的类的名称。第二个参数是一个数组，包含了需要传递给处理器的数据。一个任务处理器应该像这样定义：

**定义一个任务处理器**

	class SendEmail {

		public function fire($job, $data)
		{
			//
		}

	}

注意，类唯一需要的方法是 `fire` 方法，它接受一个 `Job` 实例就像 `data` 数组一样发送到队列。

如果你希望任务调用 `fire` 以外的方法，你可以在推送任务时指定相应方法：

**指定一个自定义处理器方法**

	Queue::push('SendEmail@send', array('message' => $message));

一旦你处理完了一个任务，必须从队列中将它删除，可以通过 `Job` 实例中的 `delete` 方法完成这项工作：

**删除一个处理完的任务**

	public function fire($job, $data)
	{
		// Process the job...

		$job->delete();
	}

如果你想要将一个任务放回队列，你可以使用 `release` 方法：

**将一个任务放回队列**

	public function fire($job, $data)
	{
		// Process the job...

		$job->release();
	}

你也可以指定几秒后将任务放回队列：

	$job->release(5);

如果任务在处理时发生异常，它会被自动放回队列。你可以使用 `attempts` 方法获取尝试处理任务的次数：

**获取尝试处理任务的次数**

	if ($job->attempts() > 3)
	{
		//
	}

你也可以获取任务的ID：

**获取任务ID**

	$job->getJobId();

<a name="queueing-closures"></a>
## 队列闭包

你可以将一个闭包函数推送到队列中。这非常便于快速、简单地处理队列任务：

**推送一个闭包函数到队列中**

	Queue::push(function($job) use ($id)
	{
		Account::delete($id);

		$job->delete();
	});

> **注意：** 当推送闭包到队列时，不能使用 `__DIR__` 和 `__FILE__` 常量。

当使用 Iron.io [推送队列](#push-queues) 时，你需要特别谨慎地处理队列闭包。接受队列信息的结尾应该带有Iron.io的验证令牌。例如，推送队列的结尾应该类似： `https://yourapp.com/queue/receive?token=SecretToken`。你也许需要在封装队列请求前检查一下应用的秘密令牌的值。

<a name="running-the-queue-listener"></a>
## 运行队列监听器

Laravel包含了一个用于运行已推送到队列的任务的Artisan服务。可以使用 `queue:listen` 命令来运行这个功能：

**开启队列监听器**

	php artisan queue:listen

你也可以指定队列监听器需要使用的连接：

	php artisan queue:listen connection

注意一旦任务启动，它会一直运行除非你手动停止它。可以使用进程监视工具（例如 [Supervisor](http://supervisord.org/)）来确保队列监听器处于运行状态。

你也可以设置单个任务可以执行的最长时间（单位秒）：

**设置任务的超时参数**

	php artisan queue:listen --timeout=60

另外，你还可以指定新任务轮询之前所需要等待的秒数：

	php artisan queue:listen --sleep=5

如果只想处理队列的第一个任务，你可以使用 `queue:work` 命令：

**处理队列的第一个任务**

	php artisan queue:work

<a name="push-queues"></a>
## 推送队列

推送队列可以让你在没有守护进程和后台监听器的情况下使用 Laravel 4 强大的队列工具。当前，推送队列仅支持[Iron.io](http://iron.io)驱动。在开始前，创建一个 Iron.io 账户，然后将Iron的认证信息填入到 `app/config/queue.php` 配置文件中。

接下来，你可以使用 `queue:subscribe` Artisan命令注册一个用来接收新的推送队列任务的URL结尾。

**注册一个推送队列订阅**

	php artisan queue:subscribe queue_name http://foo.com/queue/receive

现在，当你登录到Iron后台，你可以看见新的推送队列和订阅的URL。你可以为一个指定的队列订阅多个URL。接下来，为 `queue/receive` 结尾的URL创建一个路由，并且返回来自 `Queue::marshal` 方法的响应：

	Route::post('queue/receive', function()
	{
		return Queue::marshal();
	});

`marshal` 方法会自动执行相应的任务处理器类。想要处理推送队列上的任务，可以像处理一般的队列一样使用 `Queue::push` 方法。
