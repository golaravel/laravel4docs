# Eloquent ORM

- [介绍](#introduction)
- [Basic Usage](#basic-usage)
- [Mass Assignment](#mass-assignment)
- [Insert, Update, Delete](#insert-update-delete)
- [Soft Deleting](#soft-deleting)
- [Timestamps](#timestamps)
- [Query Scopes](#query-scopes)
- [Relationships](#relationships)
- [Querying Relations](#querying-relations)
- [Eager Loading](#eager-loading)
- [Inserting Related Models](#inserting-related-models)
- [Touching Parent Timestamps](#touching-parent-timestamps)
- [Working With Pivot Tables](#working-with-pivot-tables)
- [Collections](#collections)
- [Accessors & Mutators](#accessors-and-mutators)
- [Date Mutators](#date-mutators)
- [Model Events](#model-events)
- [Model Observers](#model-observers)
- [Converting To Arrays / JSON](#converting-to-arrays-or-json)

<a name="introduction"></a>
## 介绍

Eloquent ORM 是 Laravel 提供的一个优雅的、简便的 ActiveRecord 实现方式来操作你的数据库. 每一个数据库表对应一个 "Model" 用来操作这个表.

在开始之前, 先确认你已经设置好了数据库连接在文件 `app/config/database.php` 中.

<a name="basic-usage"></a>
## 基本使用

首先, 创建一个 Eloquent 模型. 模型文件通常存放在 `app/models` 文件夹中, 当然你也可以存放在任何地方， 它们会根据你的 `composer.json` 文件中的设置自动加载.

**定义一个 Eloquent 模型**

	class User extends Eloquent {}

注意，我们没有告诉 Eloquent 哪一个数据表被用到 `User` 模型中. 小写的、复数形式的类名将作为表名， 除非你明确声明了表名. 因此, 在这个实例中, Eloquent 将假定 `User` 模型存储记录的表是 `users`. 你可以通过 `table` 属性， 在你的模型中自定义一个表名:

	class User extends Eloquent {

		protected $table = 'my_users';

	}

> **注意:** Eloquent 也将默认每一个表的主键字段是 `id`. 你可以定义一个 `primaryKey` 属性覆默认主键字段. 同样的, 当你在模型中需要更换数据库时， 你可以定义一个 `connection` 属性覆盖数据库名.

一旦模型被定义, 你准备检索并创建记录到你的表中的时候， 请注意在默认情况下，你需要在你的表中添加 `updated_at` 和 `created_at` 字段. 如果你不希望有这些自动维护的字段, 可以在你的模型中添加 `$timestamps` 属性，并将其设置为 `false`.

**检索模型中的所有记录**

	$users = User::all();

**用主键检索一条记录**

	$user = User::find(1);

	var_dump($user->name);

> **注意:** [query builder](/docs/queries) 中的所有方法在 Eloquent 模型中也是一样有效的.

**用主键检索模型或抛出异常**

有时候当模型检索不到的时候，你可能希望抛出一条异常, 以便你用 `App::error` 来捕获异常并显示 404 页面.

	$model = User::findOrFail(1);

注册 error handler, 监听 `ModelNotFoundException`

	use Illuminate\Database\Eloquent\ModelNotFoundException;

	App::error(function(ModelNotFoundException $e)
	{
		return Response::make('Not Found', 404);
	});

**使用 Eloquent 查询模型**

	$users = User::where('votes', '>', 100)->take(10)->get();

	foreach ($users as $user)
	{
		var_dump($user->name);
	}

当然, 你也可以使用 query builder 的函数.

**Eloquent 统计**

	$count = User::where('votes', '>', 100)->count();

如果你不能生成查询，你需要通过 fluent 的接口, 使用更随意的 `whereRaw`:

	$users = User::whereRaw('age > ? and votes = 100', array(25))->get();

<a name="mass-assignment"></a>
## 批量任务

当创建一条新记录的时候, 你通过一个属性数组去构造记录. 这些属性就会通过批量任务分配给模型. 这很方便; 但是, 当用户随意提交数据到模型的时候，这会是一个很**严重**的问题。如果用户随意的提交数据到模型, 那么用户就可以修改模型的**任何**属性. 为此, 所有的 Eloquent 模型在默认情况下是不容许批量任务的。

要想批量任务, 需要在模型中设置 `fillable` 或 `guarded` 属性.

`fillable` 属性指定哪些属性可以批量任务. 它可以在类或者实例这一级别中设定.

**在模型中定义 Fillable 属性**

	class User extends Eloquent {

		protected $fillable = array('first_name', 'last_name', 'email');

	}

在这个例子中，只有列出的这三个属性可以批量任务。

与 `fillable` 相反的是 `guarded`, 类似服务器的 "黑名单" 相对于 "白名单":

**在模型中定义 Guarded 属性**

	class User extends Eloquent {

		protected $guarded = array('id', 'password');

	}

在上面的例子中, `id` 和 `password` 属性 **不可以** 批量任务. 其他属性可以批量任务. 你也可以通过保护禁止 **所有** 批量任务:

**禁止所有属性批量任务**

	protected $guarded = array('*');

<a name="insert-update-delete"></a>
## 插入, 修改, 删除

通过模型创建一条新纪录到数据库中, 可以简单的创建一个模型实例，并调用 `save` 方法.

**保存一条新纪录**

	$user = new User;

	$user->name = 'John';

	$user->save();

> **注意:** 通常, 你的 Eloquent 模型会自增 keys. 但是, 如果你想指定自己的 keys, 可以在你的模型中设置 `incrementing` 属性为 `false`.

你也可以使用 `create` 属性来保存一条新纪录到模型. 插入的模型实例将从模型中返回给你. 但是, 在这之前, 你需要在模型中指定 `fillable` 或 `guarded` 属性, 因为所有的 Eloquent 模型默认禁止批量任务.

**在模型中设定 Guarded 属性**

	class User extends Eloquent {

		protected $guarded = array('id', 'account_id');

	}

**使用模型的 Create 属性**

	$user = User::create(array('name' => 'John'));

修改一条记录, 你需要先检索出它, 修改它的属性, 然后使用 `save` 方法:

**在模型中修改一条记录**

	$user = User::find(1);

	$user->email = 'john@foo.com';

	$user->save();

有时你可能想修改的不止一个模型, 而是所有的其他关联模型. 想这样做, 你只需要使用 `push` 属性:

**修改一个模型以及和其关联的其他模型**

	$user->push();

你也可以通过一组查询来修改数据:

	$affectedRows = User::where('votes', '>', 100)->update(array('status' => 2));

删除一条记录, 可以在实例中简单的使用 `delete` 方法:

**删除一条现有数据**

	$user = User::find(1);

	$user->delete();

**在模型中通过主键删除数据**

	User::destroy(1);

	User::destroy(1, 2, 3);

当然，你也可以通过查询删除一组数据:

	$affectedRows = User::where('votes', '>', 100)->delete();

如果你想更新模型中的时间戳, 你可以使用 `touch` 方法:

**只是更新模型中的时间戳**

	$user->touch();

<a name="timestamps"></a>
## 时间戳

默认情况下, Eloquent 将会自动维护你数据库表中的 `created_at` 和 `updated_at` 字段. 简单的添加 `timestamp` 字段到你的表中， Eloquent 将会处理剩下的工作. 如果你不希望 Eloquent 维护这些字段, 添加下面的属性到你的模型中:

**禁止自动时间戳**

	class User extends Eloquent {

		protected $table = 'users';

		public $timestamps = false;

	}

如果你想自定义时间戳的格式, 你可以在你的模型中覆盖 `freshTimestamp` 属性:

**Providing A Custom Timestamp Format**

	class User extends Eloquent {

		public function freshTimestamp()
		{
			return time();
		}

	}

<a name="soft-deleting"></a>
## 软删除

当你软删除一条记录的时候, 事实上它并没有从数据库中删除掉. 取而代之的是一个 `deleted_at` 的时间戳在记录中. 要想在模型中使用软删除, 需要在模型中定义 `softDelete` 属性:

	class User extends Eloquent {

		protected $softDelete = true;

	}

然后在你的表中添加 `deleted_at` 字段, 你也可以从 migration 中定义 `softDeletes`属性:

	$table->softDeletes();

现在, 当你在模型中使用 `delete` 属性的时候, `deleted_at` 字段将设置最近的时间为时间戳. 当查询使用软删除的模型的时候, "已删除" 的数据将不在查询结果中. 要迫使软删除的数据出现在结果集中, 需要在查询中使用 `withTrashed` 方法:

**Forcing Soft Deleted Models Into Results**

	$users = User::withTrashed()->where('account_id', 1)->get();

If you wish to **only** receive soft deleted models in your results, you may use the `onlyTrashed` method:

	$users = User::onlyTrashed()->where('account_id', 1)->get();

To restore a soft deleted model into an active state, use the `restore` method:

	$user->restore();

You may also use the `restore` method on a query:

	User::withTrashed()->where('account_id', 1)->restore();

The `restore` method may also be used on relationships:

	$user->posts()->restore();

If you wish to truly remove a model from the database, you may use the `forceDelete` method:

	$user->forceDelete();

The `forceDelete` method also works on relationships:

	$user->posts()->forceDelete();

To determine if a given model instance has been soft deleted, you may use the `trashed` method:

	if ($user->trashed())
	{
		//
	}

<a name="query-scopes"></a>
## Query Scopes

Scopes allow you to easily re-use query logic in your models. To define a scope, simply prefix a model method with `scope`:

**Defining A Query Scope**

	class User extends Eloquent {

		public function scopePopular($query)
		{
			return $query->where('votes', '>', 100);
		}

	}

**Utilizing A Query Scope**

	$users = User::popular()->orderBy('created_at')->get();

<a name="relationships"></a>
## Relationships

Of course, your database tables are probably related to one another. For example, a blog post may have many comments, or an order could be related to the user who placed it. Eloquent makes managing and working with these relationships easy. Laravel supports four types of relationships:

- [One To One](#one-to-one)
- [One To Many](#one-to-many)
- [Many To Many](#many-to-many)
- [Polymorphic Relations](#polymorphic-relations)

<a name="one-to-one"></a>
### One To One

A one-to-one relationship is a very basic relation. For example, a `User` model might have one `Phone`. We can define this relation in Eloquent:

**Defining A One To One Relation**

	class User extends Eloquent {

		public function phone()
		{
			return $this->hasOne('Phone');
		}

	}

The first argument passed to the `hasOne` method is the name of the related model. Once the relationship is defined, we may retrieve it using Eloquent's [dynamic properties](#dynamic-properties):

	$phone = User::find(1)->phone;

The SQL performed by this statement will be as follows:

	select * from users where id = 1

	select * from phones where user_id = 1

Take note that Eloquent assumes the foreign key of the relationship based on the model name. In this case, `Phone` model is assumed to use a `user_id` foreign key. If you wish to override this convention, you may pass a second argument to the `hasOne` method:

	return $this->hasOne('Phone', 'custom_key');

To define the inverse of the relationship on the `Phone` model, we use the `belongsTo` method:

**Defining The Inverse Of A Relation**

	class Phone extends Eloquent {

		public function user()
		{
			return $this->belongsTo('User');
		}

	}

<a name="one-to-many"></a>
### One To Many

An example of a one-to-many relation is a blog post that "has many" comments. We can model this relation like so:

	class Post extends Eloquent {

		public function comments()
		{
			return $this->hasMany('Comment');
		}

	}

Now we can access the post's comments through the [dynamic property](#dynamic-properties):

	$comments = Post::find(1)->comments;

If you need to add further constraints to which comments are retrieved, you may call the `comments` method and continue chaining conditions:

	$comments = Post::find(1)->comments()->where('title', '=', 'foo')->first();

Again, you may override the conventional foreign key by passing a second argument to the `hasMany` method:

	return $this->hasMany('Comment', 'custom_key');

To define the inverse of the relationship on the `Comment` model, we use the `belongsTo` method:

**Defining The Inverse Of A Relation**

	class Comment extends Eloquent {

		public function post()
		{
			return $this->belongsTo('Post');
		}

	}

<a name="many-to-many"></a>
### Many To Many

Many-to-many relations are a more complicated relationship type. An example of such a relationship is a user with many roles, where the roles are also shared by other users. For example, many users may have the role of "Admin". Three database tables are needed for this relationship: `users`, `roles`, and `role_user`. The `role_user` table is derived from the alphabetical order of the related model names, and should have `user_id` and `role_id` columns.

We can define a many-to-many relation using the `belongsToMany` method:

	class User extends Eloquent {

		public function roles()
		{
			return $this->belongsToMany('Role');
		}

	}

Now, we can retrieve the roles through the `User` model:

	$roles = User::find(1)->roles;

If you would like to use an unconventional table name for your pivot table, you may pass it as the second argument to the `belongsToMany` method:

	return $this->belongsToMany('Role', 'user_roles');

You may also override the conventional associated keys:

	return $this->belongsToMany('Role', 'user_roles', 'user_id', 'foo_id');

Of course, you may also define the inverse of the relationship on the `Role` model:

	class Role extends Eloquent {

		public function users()
		{
			return $this->belongsToMany('User');
		}

	}

<a name="polymorphic-relations"></a>
### Polymorphic Relations

Polymorphic relations allow a model to belong to more than one other model, on a single association. For example, you might have a photo model that belongs to either a staff model or an order model. We would define this relation like so:

	class Photo extends Eloquent {

		public function imageable()
		{
			return $this->morphTo();
		}

	}

	class Staff extends Eloquent {

		public function photos()
		{
			return $this->morphMany('Photo', 'imageable');
		}

	}

	class Order extends Eloquent {

		public function photos()
		{
			return $this->morphMany('Photo', 'imageable');
		}

	}

Now, we can retrieve the photos for either a staff member or an order:

**Retrieving A Polymorphic Relation**

	$staff = Staff::find(1);

	foreach ($staff->photos as $photo)
	{
		//
	}

However, the true "polymorphic" magic is when you access the staff or order from the `Photo` model:

**Retrieving The Owner Of A Polymorphic Relation**

	$photo = Photo::find(1);

	$imageable = $photo->imageable;

The `imageable` relation on the `Photo` model will return either a `Staff` or `Order` instance, depending on which type of model owns the photo.

To help understand how this works, let's explore the database structure for a polymorphic relation:

**Polymorphic Relation Table Structure**

	staff
		id - integer
		name - string

	orders
		id - integer
		price - integer

	photos
		id - integer
		path - string
		imageable_id - integer
		imageable_type - string

The key fields to notice here are the `imageable_id` and `imageable_type` on the `photos` table. The ID will contain the ID value of, in this example, the owning staff or order, while the type will contain the class name of the owning model. This is what allows the ORM to determine which type of owning model to return when accessing the `imageable` relation.

<a name="querying-relations"></a>
## Querying Relations

When accessing the records for a model, you may wish to limit your results based on the existence of a relationship. For example, you wish to pull all blog posts that have at least one comment. To do so, you may use the `has` method:

**Checking Relations When Selecting**

	$posts = Post::has('comments')->get();

You may also specify an operator and a count:

	$posts = Post::has('comments', '>=', 3)->get();

<a name="dynamic-properties"></a>
### Dynamic Properties

Eloquent allows you to access your relations via dynamic properties. Eloquent will automatically load the relationship for you, and is even smart enough to know whether to call the `get` (for one-to-many relationships) or `first` (for one-to-one relationships) method.  It will then be accessible via a dynamic property by the same name as the relation. For example, with the following model `$phone`:

	class Phone extends Eloquent {

		public function user()
		{
			return $this->belongsTo('User');
		}

	}

	$phone = Phone::find(1);
	
Instead of echoing the user's email like this:

	echo $phone->user()->first()->email;

It may be shortened to simply:

	echo $phone->user->email;

<a name="eager-loading"></a>
## Eager Loading

Eager loading exists to alleviate the N + 1 query problem. For example, consider a `Book` model that is related to `Author`. The relationship is defined like so:

	class Book extends Eloquent {

		public function author()
		{
			return $this->belongsTo('Author');
		}

	}

Now, consider the following code:

	foreach (Book::all() as $book)
	{
		echo $book->author->name;
	}

This loop will execute 1 query to retrieve all of the books on the table, then another query for each book to retrieve the author. So, if we have 25 books, this loop would run 26 queries.

Thankfully, we can use eager loading to drastically reduce the number of queries. The relationships that should be eager loaded may be specified via the `with` method:

	foreach (Book::with('author')->get() as $book)
	{
		echo $book->author->name;
	}

In the loop above, only two queries will be executed:

	select * from books

	select * from authors where id in (1, 2, 3, 4, 5, ...)

Wise use of eager loading can drastically increase the performance of your application.

Of course, you may eager load multiple relationships at one time:

	$books = Book::with('author', 'publisher')->get();

You may even eager load nested relationships:

	$books = Book::with('author.contacts')->get();

In the example above, the `author` relationship will be eager loaded, and the author's `contacts` relation will also be loaded.

### Eager Load Constraints

Sometimes you may wish to eager load a relationship, but also specify a condition for the eager load. Here's an example:

	$users = User::with(array('posts' => function($query)
	{
		$query->where('title', 'like', '%first%');
	}))->get();

In this example, we're eager loading the user's posts, but only if the post's title column contains the word "first".

### Lazy Eager Loading

It is also possible to eagerly load related models directly from an already existing model collection. This may be useful when dynamically deciding whether to load related models or not, or in combination with caching.

	$books = Book::all();

	$books->load('author', 'publisher');

<a name="inserting-related-models"></a>
## Inserting Related Models

You will often need to insert new related models. For example, you may wish to insert a new comment for a post. Instead of manually setting the `post_id` foreign key on the model, you may insert the new comment from its parent `Post` model directly:

**Attaching A Related Model**

	$comment = new Comment(array('message' => 'A new comment.'));

	$post = Post::find(1);

	$comment = $post->comments()->save($comment);

In this example, the `post_id` field will automatically be set on the inserted comment.

### Inserting Related Models (Many To Many)

You may also insert related models when working with many-to-many relations. Let's continue using our `User` and `Role` models as examples. We can easily attach new roles to a user using the `attach` method:

**Attaching Many To Many Models**

	$user = User::find(1);

	$user->roles()->attach(1);

You may also pass an array of attributes that should be stored on the pivot table for the relation:

	$user->roles()->attach(1, array('expires' => $expires));

Of course, the opposite of `attach` is `detach`:

	$user->roles()->detach(1);

You may also use the `sync` method to attach related models. The `sync` method accepts an array of IDs to place on the pivot table. After this operation is complete, only the IDs in the array will be on the intermediate table for the model:

**Using Sync To Attach Many To Many Models**

	$user->roles()->sync(array(1, 2, 3));

You may also associate other pivot table values with the given IDs:

**Adding Pivot Data When Syncing**

	$user->roles()->sync(array(1 => array('expires' => true)));

Sometimes you may wish to create a new related model and attach it in a single command. For this operation, you may use the `save` method:

	$role = new Role(array('name' => 'Editor'));

	User::find(1)->roles()->save($role);

In this example, the new `Role` model will be saved and attached to the user model. You may also pass an array of attributes to place on the joining table for this operation:

	User::find(1)->roles()->save($role, array('expires' => $expires));

<a name="touching-parent-timestamps"></a>
## Touching Parent Timestamps

When a model `belongsTo` another model, such as a `Comment` which belongs to a `Post`, it is often helpful to update the parent's timestamp when the child model is updated. For example, when a `Comment` model is updated, you may want to automatically touch the `updated_at` timestamp of the owning `Post`. Eloquent makes it easy. Just add a `touches` property containing the names of the relationships to the child model:

	class Comment extends Eloquent {

		protected $touches = array('post');

		public function post()
		{
			return $this->belongsTo('Post');
		}

	}

Now, when you update a `Comment`, the owning `Post` will have its `updated_at` column updated:

	$comment = Comment::find(1);

	$comment->text = 'Edit to this comment!';

	$comment->save();

<a name="working-with-pivot-tables"></a>
## Working With Pivot Tables

As you have already learned, working with many-to-many relations requires the presence of an intermediate table. Eloquent provides some very helpful ways of interacting with this table. For example, let's assume our `User` object has many `Role` objects that it is related to. After accessing this relationship, we may access the `pivot` table on the models:

	$user = User::find(1);

	foreach ($user->roles as $role)
	{
		echo $role->pivot->created_at;
	}

Notice that each `Role` model we retrieve is automatically assigned a `pivot` attribute. This attribute contains a model representing the intermediate table, and may be used as any other Eloquent model.

By default, only the keys will be present on the `pivot` object. If your pivot table contains extra attributes, you must specify them when defining the relationship:

	return $this->belongsToMany('Role')->withPivot('foo', 'bar');

Now the `foo` and `bar` attributes will be accessible on our `pivot` object for the `Role` model.

If you want your pivot table to have automatically maintained `created_at` and `updated_at` timestamps, use the `withTimestamps` method on the relationship definition:

	return $this->belongsToMany('Role')->withTimestamps();

To delete all records on the pivot table for a model, you may use the `detach` method:

**Deleting Records On A Pivot Table**

	User::find(1)->roles()->detach();

Note that this operation does not delete records from the `roles` table, but only from the pivot table.

<a name="collections"></a>
## Collections

All multi-result sets returned by Eloquent either via the `get` method or a relationship return an Eloquent `Collection` object. This object implements the `IteratorAggregate` PHP interface so it can be iterated over like an array. However, this object also has a variety of other helpful methods for working with result sets.

For example, we may determine if a result set contains a given primary key using the `contains` method:

**Checking If A Collection Contains A Key**

	$roles = User::find(1)->roles;

	if ($roles->contains(2))
	{
		//
	}

Collections may also be converted to an array or JSON:

	$roles = User::find(1)->roles->toArray();

	$roles = User::find(1)->roles->toJson();

If a collection is cast to a string, it will be returned as JSON:

	$roles = (string) User::find(1)->roles;

Eloquent collections also contain a few helpful methods for looping and filtering the items they contain:

**Iterating & Filtering Collections**

	$roles = $user->roles->each(function($role)
	{

	});

	$roles = $user->roles->filter(function($role)
	{

	});

**Applying A Callback To Each Collection Object**

	$roles = User::find(1)->roles;
	
	$roles->each(function($role)
	{
		//	
	});

**Sorting A Collection By A Value**

	$roles = $roles->sortBy(function($role)
	{
		return $role->created_at;
	});

Sometimes, you may wish to return a custom Collection object with your own added methods. You may specify this on your Eloquent model by overriding the `newCollection` method:

**Returning A Custom Collection Type**

	class User extends Eloquent {

		public function newCollection(array $models = array())
		{
			return new CustomCollection($models);
		}

	}

<a name="accessors-and-mutators"></a>
## Accessors & Mutators

Eloquent provides a convenient way to transform your model attributes when getting or setting them. Simply define a `getFooAttribute` method on your model to declare an accessor. Keep in mind that the methods should follow camel-casing, even though your database columns are snake-case:

**Defining An Accessor**

	class User extends Eloquent {

		public function getFirstNameAttribute($value)
		{
			return ucfirst($value);
		}

	}

In the example above, the `first_name` column has an accessor. Note that the value of the attribute is passed to the accessor.

Mutators are declared in a similar fashion:

**Defining A Mutator**

	class User extends Eloquent {

		public function setFirstNameAttribute($value)
		{
			$this->attributes['first_name'] = strtolower($value);
		}

	}

<a name="date-mutators"></a>
## Date Mutators

By default, Eloquent will convert the `created_at`, `updated_at`, and `deleted_at` columns to instances of [Carbon](https://github.com/briannesbitt/Carbon), which provides an assortment of helpful methods, and extends the native PHP `DateTime` class.

You may customize which fields are automatically mutated, and even completely disable this mutation, by overriding the `getDates` method of the model:

	public function getDates()
	{
		return array('created_at');
	}

When a column is considered a date, you may set its value to a UNIX timetamp, date string (`Y-m-d`), date-time string, and of course a `DateTime` / `Carbon` instance.

<a name="model-events"></a>
## Model Events

Eloquent models fire several events, allowing you to hook into various points in the model's lifecycle using the following methods: `creating`, `created`, `updating`, `updated`, `saving`, `saved`, `deleting`, `deleted`. If `false` is returned from the `creating`, `updating`, or `saving` events, the action will be cancelled:

**Cancelling Save Operations Via Events**

	User::creating(function($user)
	{
		if ( ! $user->isValid()) return false;
	});

Eloquent models also contain a static `boot` method, which may provide a convenient place to register your event bindings.

**Setting A Model Boot Method**

	class User extends Eloquent {

		public static function boot()
		{
			parent::boot();

			// Setup event bindings...
		}

	}

<a name="model-observers"></a>
## Model Observers

To consolidate the handling of model events, you may register a model observer. An observer class may have methods that correspond to the various model events. For example, `creating`, `updating`, `saving` methods may be on an observer, in addition to any other model event name.

So, for example, a model observer might look like this:

	class UserObserver {

		public function saving($model)
		{
			//
		}

		public function saved($model)
		{
			//
		}

	}

You may register an observer instance using the `observe` method:

	User::observe(new UserObserver);

<a name="converting-to-arrays-or-json"></a>
## Converting To Arrays / JSON

When building JSON APIs, you may often need to convert your models and relationships to arrays or JSON. So, Eloquent includes methods for doing so. To convert a model and its loaded relationship to an array, you may use the `toArray` method:

**Converting A Model To An Array**

	$user = User::with('roles')->first();

	return $user->toArray();

Note that entire collections of models may also be converted to arrays:

	return User::all()->toArray();

To convert a model to JSON, you may use the `toJson` method:

**Converting A Model To JSON**

	return User::find(1)->toJson();

Note that when a model or collection is cast to a string, it will be converted to JSON, meaning you can return Eloquent objects directly from your application's routes!

**Returning A Model From A Route**

	Route::get('users', function()
	{
		return User::all();
	});

Sometimes you may wish to limit the attributes that are included in your model's array or JSON form, such as passwords. To do so, add a `hidden` property definition to your model:

**Hiding Attributes From Array Or JSON Conversion**

	class User extends Eloquent {

		protected $hidden = array('password');

	}

Alternatively, you may use the `visible` property to define a white-list:

	protected $visible = array('first_name', 'last_name');
