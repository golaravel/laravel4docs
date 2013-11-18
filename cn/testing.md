# 单元测试

- [介绍](#introduction)
- [定义以及运行测试](#defining-and-running-tests)
- [测试环境](#test-environment)
- [测试中执行路由](#calling-routes-from-tests)
- [Mocking Facades](#mocking-facades)
- [断言](#framework-assertions)
- [辅助方法](#helper-methods)

<a name="introduction"></a>
## 简介

Laravel 自建了单元测试。实际上，也较好的支持 PHPUnit 测试，并且应用程序已经配置好了 `phpunit.xml` 文件。除了 PHPUnit，Laravel 也使用了 Symfony HttpKernel，DomCrawler 和 BrowserKit 组件，允许你在测试中检查和操作视图和模拟浏览器行为。

一个示例测试文件存放在 `app/tests` 目录。在安装了一个新的 Laravel 应用程序后，简单的执行 `phpunit` 在命令行中来运行测试。


<a name="defining-and-running-tests"></a>
## 定义以及运行测试

要穿件一个测试用例，只需要简单的在 `app/tests` 目录创建测试文件。测试类需要继承 `TestCase` 。然后就如 PHPUnit 基本操作来定义测试方法。

**一个测试类示例**

	class FooTest extends TestCase {

		public function testSomethingIsTrue()
		{
			$this->assertTrue(true);
		}

	}

你可以在终端执行 `phpunit` 命令来运行应用程序中的所有测试。

> **注意:** 如果你定义了自己的 `setUp` 方法，请确保调用了 `parent::setUp `。

<a name="test-environment"></a>
## 测试环境

在运行单元测试时，Laravel 自动将配置环境设置为 `testing` 。并且，Laravel 在测试环境中包含了 `session` 和 `cache` 配置文件。测试环境中，这两个驱动都被设置为空 `array`，意味着在测试过程中，不会将 session 或者 cache 持久化。必要时，你也可以创建其他的测试环境。

<a name="calling-routes-from-tests"></a>
## 测试中执行路由

在测试时可以很简单的使用 `call` 方法来执行一个路由：

**测试中执行路由**

	$response = $this->call('GET', 'user/profile');

	$response = $this->call($method, $uri, $parameters, $files, $server, $content);

你可以检查 `Illuminate\Http\Response` 对象:

	$this->assertEquals('Hello World', $response->getContent());

你也可以在测试中执行控制器：

**测试中执行控制器**

	$response = $this->action('GET', 'HomeController@index');

	$response = $this->action('GET', 'UserController@profile', array('user' => 1));

`getContent` 方法将会返回和响应相同字符串内容。如果路由返回 `视图` ， 你可以使用 `original` 属性来访问：

	$view = $response->original;

	$this->assertEquals('John', $view['name']);

要执行个 HTTPS 路由，可以使用 `callSecure` 方法：

	$response = $this->callSecure('GET', 'foo/bar');

### DOM Crawler

你也可以执行一个路由并且接受一个 DOM Crawler 实例来检查内容：

	$crawler = $this->client->request('GET', '/');

	$this->assertTrue($this->client->getResponse()->isOk());

	$this->assertCount(1, $crawler->filter('h1:contains("Hello World!")'));

For more information on how to use the crawler, refer to its [official documentation](http://symfony.com/doc/master/components/dom_crawler.html).

<a name="mocking-facades"></a>
## Mocking Facades

在测试时， 你可能经常会想要模拟一次调用 Laravel 静态外观。例如，考虑如下控制器行为：

	public function getIndex()
	{
		Event::fire('foo', array('name' => 'Dayle'));

		return 'All done!';
	}

我可以通过在 facade 使用 `shouldReceive` 方法在模拟执行 `Event` 类, 它将返回一个 [Mockery](https://github.com/padraic/mockery) 实例。



**Mocking A Facade**

	public function testGetIndex()
	{
		Event::shouldReceive('fire')->once()->with('foo', array('name' => 'Dayle'));

		$this->call('GET', '/');
	}

> **Note:** You should not mock the `Request` facade. Instead, pass the input you desire into the `call` method when running your test.

<a name="framework-assertions"></a>
## 断言

Laravel 提供了一些 `断言` 方法来让测试更加容易：

**断言响应为OK**

	public function testMethod()
	{
		$this->call('GET', '/');

		$this->assertResponseOk();
	}

**断言响应状态码**

	$this->assertResponseStatus(403);

**断言响应为重定向**

	$this->assertRedirectedTo('foo');

	$this->assertRedirectedToRoute('route.name');

	$this->assertRedirectedToAction('Controller@method');

**断言视图含有的一些数据**

	public function testMethod()
	{
		$this->call('GET', '/');

		$this->assertViewHas('name');
		$this->assertViewHas('age', $value);
	}

**断言 Session 中含有的一些数据**

	public function testMethod()
	{
		$this->call('GET', '/');

		$this->assertSessionHas('name');
		$this->assertSessionHas('age', $value);
	}

<a name="helper-methods"></a>
## 辅助方法

`TestCase` 类包含一些辅助方法来让应用程序测试更加容易。

你可以使用 `be` 方法来设置当前验证用户：

**设置当前验证用户**

	$user = new User(array('name' => 'John'));

	$this->be($user);

你可以在测试中使用 `seed` 方法来重播数据库：

**测试中重播数据库**

	$this->seed();

	$this->seed($connection);

更多关于创建播种信息请查看文档 [迁移和播种](/docs/migrations#database-seeding)。
