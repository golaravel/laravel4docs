# 安全

- [配置](#configuration)
- [保存密码](#storing-passwords)
- [验证用户](#authenticating-users)
- [手动登陆用户](#manually)
- [保护路由](#protecting-routes)
- [HTTP基本验证](#http-basic-authentication)
- [密码提示和重置](#password-reminders-and-reset)
- [加密](#encryption)

<a name="configuration"></a>
## 配置

Laravel 旨在让验证实现简单。事实上，大部分都已经配置好了。验证配置文件位于 `app/config/auth.php`，该文件包含了一些文档注释良好的调整验证工具配置选项。

默认，Laravel 包含一个为默认 Eloquent 验证驱动所使用的 `User` 模型存放在 `app/models`目录。注意在构建该模型数据库结构时确保密码字段至少能容纳60个字符。

如果你的应用中不使用 Eloquent，你可以使用 Laravel 查询构造器来使用 `数据库` 验证驱动。

<a name="storing-passwords"></a>
## 保存密码

Laravel `Hash` 类提供了可靠的Bcrypt散列算法：

**使用Bcrypt散列密码**

	$password = Hash::make('secret');

**验证散列密码**

	if (Hash::check('secret', $hashedPassword))
	{
		// 密码匹配...
	}

**检查密码是否需要重新散列**

	if (Hash::needsRehash($hashed))
	{
		$hashed = Hash::make('secret');
	}

<a name="authenticating-users"></a>
## 验证用户

要在应用程序中登陆用户，使用 `Auth::attempt` 方法。

	if (Auth::attempt(array('email' => $email, 'password' => $password)))
	{
		return Redirect::intended('dashboard');
	}

注意 `email` 不是必需的，这里仅作为示例。你应该使用任意数据库字段名来作为"用户名"。 `Redirect::intended` 方法会将用户重定向用户到指定在验证过滤器捕获访问时URL。可以给该方法提供个回退URI，在重定向无效时，会重定向到该URI。

在调用 `attempt` 方法时，`auth.attempt` [事件](/docs/events) 将会触发。如果验证成功以及用户登陆了，`auth.login` 事件也会被触发。

要在应用程序中判断用户是否已经登陆，需要使用 `check` 方法：

**判断用户是否验证**

	if (Auth::check())
	{
		// 用户已经登陆...
	}
	
如果你想提供“记住我”功能，你可以传递true作为第二个参数传递给attempt方法，应用程序将会永久保持用户验证状态（除非手动退出）：	

**验证用户并且“记住”她们**

	if (Auth::attempt(array('email' => $email, 'password' => $password), true))
	{
		// 用户状态永久保存...
	}

**注意：** 如果 `attempt` 方法返回 `true`， 用户就已经成功登陆应用程序了。

你也可以在验证时指定其余的查询条件：

**指定其它条件验证用户**

    if (Auth::attempt(array('email' => $email, 'password' => $password, 'active' => 1)))
    {
        // 用户激动状态，没有暂停，并且存在。
    }

一旦用户验证通过了，就可以查看 User 模型/记录：

**查看登陆用户**

	$email = Auth::user()->email;

要简单使用用户ID来登陆应用程序，可以使用 `loginUsingId` 方法：

	Auth::loginUsingId(1);
	
`validate` 方法可以让你仅验证用户，而不实际登陆应用程序：

**验证用户但不登陆**

	if (Auth::validate($credentials))
	{
		//
	}

也可以使用 `once` 方法来在应用程序单次请求中登陆用户。而不放入 session或cookie中。

**单词请求中登陆用户**

	if (Auth::once($credentials))
	{
		//
	}

**应用程序中退出用户登陆**

	Auth::logout();

<a name="manually"></a>
## 手动登陆用户

如果想登陆一个已经存有的用户实例，只需要简单使用`login`方法并传入该实例即可：

	$user = User::find(1);

	Auth::login($user);

这与使用 `attempt` 方法验证用户登陆是一样的。


<a name="protecting-routes"></a>
## 保护路由

路由过滤器提供了只允许验证用户可访问路由。Laravel 默认提供了 `auth` 过滤器，它定义在 `app/filters.php`文件。

**保护路由**

	Route::get('profile', array('before' => 'auth', function()
	{
		// 只有验证用户可以访问……
	}));

### 跨站请求伪造（CSRF）保护

Laravel 提供很方便的方法来保护应用程序中跨站请求伪造。

**表单中插入 CSRF Token**

    <input type="hidden" name="_token" value="<?php echo csrf_token(); ?>">

**提供表单时验证 CSRF Token**

    Route::post('register', array('before' => 'csrf', function()
    {
        return 'You gave a valid CSRF token!';
    }));

<a name="http-basic-authentication"></a>
## HTTP 基本验证

HTTP 基本验证提供了快速的方法来在应用程序中验证用户，而不用操心于“登陆”页面。要使用该功能，在路由中附加 `auth.filter` 过滤器：

**HTTP 基本验证来保护路由**

	Route::get('profile', array('before' => 'auth.basic', function()
	{
		// 只有验证用户可以访问……
	}));

默认，`basic` 过滤器验证时会使用用户记录里的 `email` 字段。如果你想使用其他字段，你可以通过传递该字段名字给 `basic` 方法的第一个参数：

	return Auth::basic('username');

你也可以使用 HTTP 基本验证时不将用户信息写入 session，对 API 验证极其有用。要实现该功能，定义个过滤器并返回 `onceBasic` 方法：

**设置无状态的 HTTP 基本过滤器**

	Route::filter('basic.once', function()
	{
		return Auth::onceBasic();
	});

<a name="password-reminders-and-reset"></a>
## 密码提示和重置

### 发送密码提示

大多web应用程序提供了让用户重置密码功能。与其让你在每个应用程序中重复去实现该功能，Laravel 提供非常方便的方法来发送密码提示以及执行密码重置。在开始前，请确认你的 `User` 模型实现了 `Illuminate\Auth\Reminders\RemindableInterface` 接口。 当然，框架中的 `User` 模型已经实现了该接口。

**实现 RemindableInterface 接口**

	class User extends Eloquent implements RemindableInterface {

		public function getReminderEmail()
		{
			return $this->email;
		}

	}

接下来，要创建一个表来存储密码重置 token。要为该表生成迁移，简单执行 Artisan 命令 `auth::reminders`：

**为提示表建立迁移**

	php artisan auth:reminders

	php artisan migrate

要发送密码提示，我们可以使用 `Password::remind` 方法：

**发送密码提示**

	Route::post('password/remind', function()
	{
		$credentials = array('email' => Input::get('email'));

		return Password::remind($credentials);
	});

注意传递给 `remind` 方法的参数和 `Auth::attempt` 方法一致。该方法将会取得 `User` 并且通过邮箱发送密码重置链接。e-mail 视图将会传递一个用来构造重置密码表单链接的 `token` 变量。`user` 对象也会传递到该视图。

> **注意:** 你也可以改变邮件消息视图，通过改变 `auth.reminder.email` 配置项。当然，已经提供了个默认视图。.

你可以通过传递一个 Closure 作为 `remind` 方法的第二个参数来改变发送给用户的邮件实例：

	return Password::remind($credentials, function($message, $user)
	{
		$message->subject('Your Password Reminder');
	});

你也可能已经注意到我们我们直接从路由中返回 `remind` 方法的结果。默认，`remind` 方法将会返回 `Redirect` 重定向当前 URI. 当在尝试重置密码时发生错误， 一个 `error` 变量将会 flash 到session中，还有个 `reason` 变量，可以从 `reminders` 语言文件中取得文本。如果密码重置成功，`success` 变量会 flash 到session。所以，你的密码重置表单视图类似如下：

	@if (Session::has('error'))
		{{ trans(Session::get('reason')) }}
	@elseif (Session::has('success'))
		An e-mail with the password reset has been sent.
	@endif

	<input type="text" name="email">
	<input type="submit" value="Send Reminder">

### 重置密码

一旦用户从邮件提示点击了重置链接，她们将被重定向到一个包含 `token` 隐藏域的表单，还包含了一个 `password` 和 `password_confirmation` 域。以下是一个重置表单路由示例：

	Route::get('password/reset/{token}', function($token)
	{
		return View::make('auth.reset')->with('token', $token);
	});

重置表单类似：

	@if (Session::has('error'))
		{{ trans(Session::get('reason')) }}
	@endif

	<input type="hidden" name="token" value="{{ $token }}">
	<input type="text" name="email">
	<input type="password" name="password">
	<input type="password" name="password_confirmation">


再次强调，我们使用 `Session` 来显示任何重置密码时框架检测出的错误。接下来，我们可以定义一个 `POST` 路由来处理重置：

	Route::post('password/reset/{token}', function()
	{
		$credentials = array('email' => Input::get('email'));

		return Password::reset($credentials, function($user, $password)
		{
			$user->password = Hash::make($password);

			$user->save();

			return Redirect::to('home');
		});
	});

如果密码重置成功，`User` 实例以及密码将会传递到 Closure，允许你实际执行保存操作。然后，你可以在 `reset` 方法返回 `Redirect` 或是任何其他 Response 类型。注意 `reset` 方法会自动检查请求中 `token` 和 信息的有效性，并且匹配密码。

同 `remind` 方法， 如果在重置密码时发生错误， `reset` 方法将会 `Redirect` 到当前URI，并带有 `error` 和 `reason` 变量。

<a name="encryption"></a>
## 加密

Laravel 提供了强大的 AES-256 加密，通过使用 mcrypt PHP 扩展：

**加密**

	$encrypted = Crypt::encrypt('secret');

> **注意:** 确认在 `app/config/app.php` 文件设置了一个32随机字符给 `key` 项。否则，加密的值不安全。

**解密**

	$decrypted = Crypt::decrypt($encryptedValue);

你也可以使用 encrypter 设置 cipher 和 mode：


**设置 Cipher 和 Mode**

	Crypt::setMode('crt');

	Crypt::setCipher($cipher);
