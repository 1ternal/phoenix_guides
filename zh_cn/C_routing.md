路由器是 Phoenix 应用的中枢。它们将HTTP请求匹配到控制器动作、连接实时通道的处理器
、给一组路由定义一系列流式转换中间件。

Phoenix 生成的路由文件 `web/router.ex` 如下所示：

```elixir
defmodule HelloPhoenix.Router do
  use HelloPhoenix.Web, :router

  pipeline :browser do
    plug :accepts, ["html"]
    plug :fetch_session
    plug :fetch_flash
    plug :protect_from_forgery
    plug :put_secure_browser_headers
  end

  pipeline :api do
    plug :accepts, ["json"]
  end

  scope "/", HelloPhoenix do
    pipe_through :browser # Use the default browser stack

    get "/", PageController, :index
  end

  # Other scopes may use custom stacks.
  # scope "/api", HelloPhoenix do
  #   pipe_through :api
  # end
end
```

你命名的应用名称将会取代 `HelloPhoenix` 作为路由模块和控制器名称。

第一行 `use HelloPhoenix.Web, :router` 使我们可以在这个特定的路由器中使用 Phoenix 路由器的函数。

Scope 在本节会单独讲，我们就先略过这里的 `scope "/", HelloPhoenix do` 代码。`pipe_through :browser` 稍后也会拿出来讲。现在，你只需要知道 pipeline 允许不同的路由允许执行一组特定的中间件变换(middleware transformations)。

在 scope 代码块内，第一条真正的路由出现啦。

```elixir
  get "/", PageController, :index
```

`get` 是个 Phoenix 宏，展开后是一个 `match/3` 函数的子句。响应 HTTP 的 GET 方法。类似的 HTTP 动词 宏包括了 POST, PUT, PATCH, DELETE, OPTIONS, CONNECT, TRACE 和 HEAD。

这些宏的第一个参数是一个路径(path)。这里是应用的根目录 `/`。后面俩参数是处理请求的模块和函数。这些宏也接受其他选项，它们会贯穿本文其余部分。

如果这是我们路由模块里唯一的路由条目，宏展开后，`match/3` 函数的子句会如下所示：

```elixir
  def match(conn, "GET", ["/"])
```

match 函数的主体部分会设定连接(connection)并调用匹配的控制器动作。

随着路由条目的增加，更多的 match 函数子句将会添加进路由器模块。它们与 Elixir 中多重子句函数的行为并无二致。从上到下尝试匹配，与 match 参数匹配成功的第一条子句将被执行。匹配成功后，搜索将停止，也不会尝试其他子句。

也就是说，无论路由器和动作，基于 HTTP 动作和路径，可能会创建一个永远匹配不到的路由条目。

如果我们新建了有歧义的条目，路由仍然会编译，只是我们将得到一个警告。让我们看看这个动作。

在 `scope "/", HelloPhoenix do` 代码块底部定义下面这样一个条目：

```elixir
get "/", RootController, :index
```

然后在项目根目录运行 `$ mix compile`。你会得到编译器给你像下面这样的警告：

```text
web/router.ex:1: warning: this clause cannot match because a previous clause at line 1 always matches
Compiled web/router.ex
```

### 查看路由

Phoenix 提供了一个很棒的工具用于查看路由条目，mix task `phoenix.routes`。

看看如何使用。在新生成的 Phoenix 应用的根目录运行 `$ mix phoenix.routes`（你需要事先运行 `$mix do deps.get, compile` 才能运行这个任务）。你应该可以看到类似下面这样由当前路由生成的条目：

```console
$ mix phoenix.routes
page_path  GET  /  HelloPhoenix.PageController :index
```

输出告诉我们任何根目录的 HTTP GET 请求将会被 `HelloPhoenix.PageController` 的 `index` 函数处理。

`page_path` 是一个所谓 path helper 的样例，稍后会讲到。

### 资源(Resources)

除了像 `get`、`post` 和 `put` 这样的HTTP 动词以外，路由还支持其他的宏。其中最重要的是 展开八条 match 函数子句的 `resources`。

如下所示，在`web/router.ex` 文件内添加一个资源(resource)。

```elixir
scope "/", HelloPhoenix do
  pipe_through :browser # Use the default browser stack

  get "/", PageController, :index
  resources "/users", UserController
end
```

就此目的，有木有 `HelloPhoenix.UserController` 都没关系。

返回根目录，运行 `$ mix phoenix.routes`。

应该看到如下所示。当然，`HelloPhoenix` 替换成应用名称。

```elixir
user_path  GET     /users           HelloPhoenix.UserController :index
user_path  GET     /users/:id/edit  HelloPhoenix.UserController :edit
user_path  GET     /users/new       HelloPhoenix.UserController :new
user_path  GET     /users/:id       HelloPhoenix.UserController :show
user_path  POST    /users           HelloPhoenix.UserController :create
user_path  PATCH   /users/:id       HelloPhoenix.UserController :update
           PUT     /users/:id       HelloPhoenix.UserController :update
user_path  DELETE  /users/:id       HelloPhoenix.UserController :delete
```

这是标准的 HTTP 动词、路径和控制器动作组成的矩阵。以稍微不同的顺序单独看看它们。

- 到 `/users` 的 GET 请求会调用 `index` 动作来显示所有的用户。
- 到 `/users/:id` 的 GET 请求调用 id 和 `show` 动作来显示由 id 标示的单独的用户
- 到 `/users/new` 的 GET 请求会调用 `new` 动作来展现一个用于创建新用户的表单。
- 到 `/users` 的 POST 请求会调用 `create` 动作来保存一个新用户到数据存储。
- 到 `/users/:id/edit` 的 GET 请求调用 id 和 `edit` 动作，从数据存储中取出一个单独的用户，并将信息填入表单进行编辑。
- 到 `/users/:id` 的 PATCH 请求调用 id 和 `update` 动作会更新数据存储这个用户的信息。
- 到 `/users/:id` 的 PUT 请求调用 id 和 `update` 动作会更新数据存储这个用户的信息。
- 到 `/users/:id` 的 DELETE 请求调用 id 和 `delete` 动作会从数据存储中移除这个用户。

如果我们并不需要所有这些路由条目，可以选择性的使用 `:only` 和 `:except` 选项。

假设我们有一个只读的文章(posts)资源。我们这个像这样定义：

```elixir
resources "posts", PostController, only: [:index, :show]
```

`$ mix phoenix.routes` 显示我们只定义了 index 和 show 两条路由条目。

```elixir
post_path  GET     /posts HelloPhoenix.PostController :index
post_path  GET     /posts/:id HelloPhoenix.PostController :show
```

类似的，如果有一个评论(comment)资源，我们不想提供删除的路由，可以这样定义：

```elixir
resources "comments", CommentController, except: [:delete]
```

`$ mix phoenix.routes` 显示了除了 DELETE 请求外所有的路由条目。

```elixir
comment_path  GET     /comments HelloPhoenix.CommentController :index
comment_path  GET     /comments/:id/edit HelloPhoenix.CommentController :edit
comment_path  GET     /comments/new HelloPhoenix.CommentController :new
comment_path  GET     /comments/:id HelloPhoenix.CommentController :show
comment_path  POST    /comments HelloPhoenix.CommentController :create
comment_path  PATCH   /comments/:id HelloPhoenix.CommentController :update
              PUT     /comments/:id HelloPhoenix.CommentController :update
```

### 路径助手(Path Helper)

路径助手是在 `Router.Helpers` 模块中动态定义的函数。对我们来说，就是 `HelloPhoenix.Router.Helpers`。它们的名字派生自路由条目所定义的控制器名称。我们的控制器是 `HelloPhoenix.PageController`, 所以 `page_path` 就是这个返回应用根目录路径的函数。

有点绕口，我们看看实际情况。应用的根目录运行 `$ iex -S mix`。当我们以 `Endpoint` 或连接，和动作为参数，在路由助手调用 `page_path` 函数，会返回这个路径。

```elixir
iex> HelloPhoenix.Router.Helpers.page_path(HelloPhoenix.Endpoint, :index)
"/"
```

这非常有用，因为我们可以在模板中使用 `page_path` 函数链接到应用的根目录。注意：若果函数调用非常长，这儿有个解决方法。在我们主应用视图中，包含 `import HelloPhoenix.Router.Helpers` 就解决了。

```html
<a href="<%= page_path(@conn, :index) %>">To the Welcome Page!</a>
```

请查看[视图向导](http://www.phoenixframework.org/docs/views)获取更多信息。

这使得我们在修改路由器的路由条目时获益匪浅。因为路径助手是根据路由动态建立的,模板中任何调用 `page_path` 函数仍然能工作正常。

### 更多关于路径助手

为用户资源(user resource)执行 `phoenix.routes` 任务时，它会将 `user_path` 作为每行的路径助手输出出来。下面就是每个动作转换过来的路径：

```elixir
iex> import HelloPhoenix.Router.Helpers
iex> alias HelloPhoenix.Endpoint
iex> user_path(Endpoint, :index)
"/users"

iex> user_path(Endpoint, :show, 17)
"/users/17"

iex> user_path(Endpoint, :new)
"/users/new"

iex> user_path(Endpoint, :create)
"/users"

iex> user_path(Endpoint, :edit, 37)
"/users/37/edit"

iex> user_path(Endpoint, :update, 37)
"/users/37"

iex> user_path(Endpoint, :delete, 17)
"/users/17"
```

搜索路径怎么办？在第四个可选参数添加一个键值对，路径助手就返回以键值对为查询字符的路径了。

```elixir
iex> user_path(Endpoint, :show, 17, admin: true, active: false)
"/users/17?admin=true&active=false"
```

如果需要 url 而不是 路径咋整？将 `_path` 替换成 `_url` 即可：

```elixir
iex(3)> user_url(Endpoint, :index)
"http://localhost:4000/users"
```

后面会有单独的章节将应用终点(endpoints)。现在，把它当作路由接管之前处理请求的实体。包括了启动应用/服务器，设定配置和对所有请求设定常用 plug 等工作。

`_url` 函数将会得到由环境变量设置的，包含了主机、端口、代理端口和 ssl 信息组成完整 url 地址。这些配置会在单独一节探讨。目前，你可以自己项目的 `/config/dev.exs` 看到这些值。

### 嵌套资源

Phoenix 允许路由里嵌套资源。假设用户和文章资源之间存在一对多的关系。也就是说，一个用户可以写很多篇文章，一篇文章只属于某一个用户。我们可以像下面这样，在 `web/router.ex` 中增加一个嵌套路由。

```elixir
resources "users", UserController do
  resources "posts", PostController
end
```

现在运行 `$ mix phoenix.routes` 时，包含之前看到的用户路由器，我们得到以下这些路由条目：

```elixir
. . .
user_post_path  GET     users/:user_id/posts HelloPhoenix.PostController :index
user_post_path  GET     users/:user_id/posts/:id/edit HelloPhoenix.PostController :edit
user_post_path  GET     users/:user_id/posts/new HelloPhoenix.PostController :new
user_post_path  GET     users/:user_id/posts/:id HelloPhoenix.PostController :show
user_post_path  POST    users/:user_id/posts HelloPhoenix.PostController :create
user_post_path  PATCH   users/:user_id/posts/:id HelloPhoenix.PostController :update
                PUT     users/:user_id/posts/:id HelloPhoenix.PostController :update
user_post_path  DELETE  users/:user_id/posts/:id HelloPhoenix.PostController :delete
```

可以看到每一条文章资源都被限定在某个用户 id 之下。拿第一条来说，调用 `PostController` 的 `index` 动作时，还需要传入 `user_id`。意味着我们将只展示这个用户的文章。这个限制适用于其他所有路由。

调用嵌套路由助手时，我们需要按照路由定义的顺序传递这些 id。拿 `show` 条目举例，`42` 是 `user_id`， `17` 是 `post_id`。开始前先给 `HelloPHoenix.Endpoint` 别名下：

```elixir
iex> alias HelloPhoenix.Endpoint
iex> HelloPhoenix.Router.Helpers.user_post_path(Endpoint, :show, 42, 17)
"/users/42/posts/17"
```

同样，如果在函数调用的最后增加一个键值对，同样会加入到查询字符。

```elixir
iex> HelloPhoenix.Router.Helpers.user_post_path(Endpoint, :index, 42, active: true)
"/users/42/posts?active=true"
```

### 限定的路由(Scoped Routes)

限定是将一组路由置于同一个路径前缀、或一组 plug 中间件的方法。管理员功能、API接口，尤其是对版本API，都需要这样的功能。假如网站有用户产生的测评，而这些测评需要先通过管理员的批准。这两个资源的语义显然是不同的，也可能不会使用同一个控制器。Scope 允许我们生成这样的路由。

面向用户的测评看起来像标准资源。

```text
/reviews
/reviews/1234
/reviews/1234/edit
. . .
```

管理员评测路径以 `/admin` 为前缀。

```text
/admin/reviews
/admin/reviews/1234
/admin/reviews/1234/edit
. . .
```

可以像这样，将一组路由条目限定在 `/admin` 路径之下。现在，先别急嵌套资源(跟新生成的 `scope "/", HelloPhoenix do` 保持一致)。

```elixir
scope "/admin" do
  pipe_through :browser

  resources "/reviews", HelloPhoenix.Admin.ReviewController
end
```

注意，Phoenix 将会假定我们设定的路径以斜杠开头，所以 `scope "/admin" do` 和 `scope "admin do` 结果是一样的。

再次注意，这种方式定义 scope，我们需要控制器的完整名称，`HelloPhoenix.Admin.ReviewController`。我们马上就会解决这个问题。

再次执行 `$ mix phoenix.routes，在上次的路有条目基础上，我们得到如下：

```elixir
. . .
review_path  GET     /admin/reviews HelloPhoenix.Admin.ReviewController :index
review_path  GET     /admin/reviews/:id/edit HelloPhoenix.Admin.ReviewController :edit
review_path  GET     /admin/reviews/new HelloPhoenix.Admin.ReviewController :new
review_path  GET     /admin/reviews/:id HelloPhoenix.Admin.ReviewController :show
review_path  POST    /admin/reviews HelloPhoenix.Admin.ReviewController :create
review_path  PATCH   /admin/reviews/:id HelloPhoenix.Admin.ReviewController :update
             PUT     /admin/reviews/:id HelloPhoenix.Admin.ReviewController :update
review_path  DELETE  /admin/reviews/:id HelloPhoenix.Admin.ReviewController :delete
```

看上去不错，但是还有个问题。我们想要的是面向用户的 `/reviews` 和面向管理员的 `/admin/reviews`。如果现在加入如下面向用户的路由：

```elixir
scope "/", HelloPhoenix do
  pipe_through :browser
  . . .
  resources "/reviews", ReviewController
  . . .
end

scope "/admin" do
  resources "/reviews", HelloPhoenix.Admin.ReviewController
end
```

执行 `$ mix phoenix.routes`, 得到如下：

```elixir
. . .
review_path  GET     /reviews HelloPhoenix.ReviewController :index
review_path  GET     /reviews/:id/edit HelloPhoenix.ReviewController :edit
review_path  GET     /reviews/new HelloPhoenix.ReviewController :new
review_path  GET     /reviews/:id HelloPhoenix.ReviewController :show
review_path  POST    /reviews HelloPhoenix.ReviewController :create
review_path  PATCH   /reviews/:id HelloPhoenix.ReviewController :update
             PUT     /reviews/:id HelloPhoenix.ReviewController :update
review_path  DELETE  /reviews/:id HelloPhoenix.ReviewController :delete
. . .
review_path  GET     /admin/reviews HelloPhoenix.Admin.ReviewController :index
review_path  GET     /admin/reviews/:id/edit HelloPhoenix.Admin.ReviewController :edit
review_path  GET     /admin/reviews/new HelloPhoenix.Admin.ReviewController :new
review_path  GET     /admin/reviews/:id HelloPhoenix.Admin.ReviewController :show
review_path  POST    /admin/reviews HelloPhoenix.Admin.ReviewController :create
review_path  PATCH   /admin/reviews/:id HelloPhoenix.Admin.ReviewController :update
             PUT     /admin/reviews/:id HelloPhoenix.Admin.ReviewController :update
review_path  DELETE  /admin/reviews/:id HelloPhoenix.Admin.ReviewController :delete
```

除了每行开头的 `review_path` 路径助手外，路由条目看上去没问题。面向用户和面向管理员的助手是一个，这肯定不对。在管理员 scope 下加入 `as: :admin` 选项就可以解决这个问题了。

```elixir
scope "/", HelloPhoenix do
  pipe_through :browser
  . . .
  resources "/reviews", ReviewController
  . . .
end

scope "/admin", as: :admin do
  resources "/reviews", HelloPhoenix.Admin.ReviewController
end
```

`$ mix phoenix.routes` 输出了我们想要的结果。

```elixir
. . .
      review_path  GET     /reviews HelloPhoenix.ReviewController :index
      review_path  GET     /reviews/:id/edit HelloPhoenix.ReviewController :edit
      review_path  GET     /reviews/new HelloPhoenix.ReviewController :new
      review_path  GET     /reviews/:id HelloPhoenix.ReviewController :show
      review_path  POST    /reviews HelloPhoenix.ReviewController :create
      review_path  PATCH   /reviews/:id HelloPhoenix.ReviewController :update
                   PUT     /reviews/:id HelloPhoenix.ReviewController :update
      review_path  DELETE  /reviews/:id HelloPhoenix.ReviewController :delete
. . .
admin_review_path  GET     /admin/reviews HelloPhoenix.Admin.ReviewController :index
admin_review_path  GET     /admin/reviews/:id/edit HelloPhoenix.Admin.ReviewController :edit
admin_review_path  GET     /admin/reviews/new HelloPhoenix.Admin.ReviewController :new
admin_review_path  GET     /admin/reviews/:id HelloPhoenix.Admin.ReviewController :show
admin_review_path  POST    /admin/reviews HelloPhoenix.Admin.ReviewController :create
admin_review_path  PATCH   /admin/reviews/:id HelloPhoenix.Admin.ReviewController :update
                   PUT     /admin/reviews/:id HelloPhoenix.Admin.ReviewController :update
admin_review_path  DELETE  /admin/reviews/:id HelloPhoenix.Admin.ReviewController :delete
```

路径助手现在也返回了我们想要的结果。自己运行 `$ iex -S mix` 来试一试：

```elixir
iex(1)> HelloPhoenix.Router.Helpers.review_path(Endpoint, :index)
"/reviews"

iex(2)> HelloPhoenix.Router.Helpers.admin_review_path(Endpoint, :show, 1234)
"/admin/reviews/1234"
```

管理员有一堆资源需要处理怎么办？我们可以如下所示，把他们都放在一个 scope：

```elixir
scope "/admin", as: :admin do
  pipe_through :browser

  resources "/images", HelloPhoenix.Admin.ImageController
  resources "/reviews", HelloPhoenix.Admin.ReviewController
  resources "/users", HelloPhoenix.Admin.UserController
end
```

下面是 `$ mix phoenix.routes` 输出的完整信息：

```elixir
. . .
 admin_image_path  GET     /admin/images HelloPhoenix.Admin.ImageController :index
 admin_image_path  GET     /admin/images/:id/edit HelloPhoenix.Admin.ImageController :edit
 admin_image_path  GET     /admin/images/new HelloPhoenix.Admin.ImageController :new
 admin_image_path  GET     /admin/images/:id HelloPhoenix.Admin.ImageController :show
 admin_image_path  POST    /admin/images HelloPhoenix.Admin.ImageController :create
 admin_image_path  PATCH   /admin/images/:id HelloPhoenix.Admin.ImageController :update
                   PUT     /admin/images/:id HelloPhoenix.Admin.ImageController :update
 admin_image_path  DELETE  /admin/images/:id HelloPhoenix.Admin.ImageController :delete
admin_review_path  GET     /admin/reviews HelloPhoenix.Admin.ReviewController :index
admin_review_path  GET     /admin/reviews/:id/edit HelloPhoenix.Admin.ReviewController :edit
admin_review_path  GET     /admin/reviews/new HelloPhoenix.Admin.ReviewController :new
admin_review_path  GET     /admin/reviews/:id HelloPhoenix.Admin.ReviewController :show
admin_review_path  POST    /admin/reviews HelloPhoenix.Admin.ReviewController :create
admin_review_path  PATCH   /admin/reviews/:id HelloPhoenix.Admin.ReviewController :update
                   PUT     /admin/reviews/:id HelloPhoenix.Admin.ReviewController :update
admin_review_path  DELETE  /admin/reviews/:id HelloPhoenix.Admin.ReviewController :delete
  admin_user_path  GET     /admin/users HelloPhoenix.Admin.UserController :index
  admin_user_path  GET     /admin/users/:id/edit HelloPhoenix.Admin.UserController :edit
  admin_user_path  GET     /admin/users/new HelloPhoenix.Admin.UserController :new
  admin_user_path  GET     /admin/users/:id HelloPhoenix.Admin.UserController :show
  admin_user_path  POST    /admin/users HelloPhoenix.Admin.UserController :create
  admin_user_path  PATCH   /admin/users/:id HelloPhoenix.Admin.UserController :update
                   PUT     /admin/users/:id HelloPhoenix.Admin.UserController :update
  admin_user_path  DELETE  /admin/users/:id HelloPhoenix.Admin.UserController :delete
```

太好了，这真是我们想要的结果，还可以做的更好。可以看到每个资源我们都必须输入控制器 `HelloPhoenix.Admin` 全称。太啰嗦,且容易出错。假定每个控制器都以 `HelloPhoenix.Admin` 开头，我们可以在 scope 声明的路径后添加 `HelloPhoenix.Admin` 选项，这样所有的路由就对了，也是全称。

```elixir
scope "/admin", HelloPhoenix.Admin, as: :admin do
  pipe_through :browser

  resources "/images",  ImageController
  resources "/reviews", ReviewController
  resources "/users",   UserController
end
```

Now run `$ mix phoenix.routes` again and you can see that we get the same result as above when we qualified each controller name individually.

As an extra bonus, we could nest all of the routes for our application inside a scope that simply has an alias for the name of our Phoenix app, and eliminate the duplication in our controller names.

Phoenix already does this now for us in the generated router for a new application (see beginning of this section). Notice here the use of `HelloPhoenix.Router` in the `defmodule` declaration:

```elixir
defmodule HelloPhoenix.Router do
  use HelloPhoenix.Web, :router

  scope "/", HelloPhoenix do
    pipe_through :browser

    get "/images", ImageController, :index
    resources "/reviews", ReviewController
    resources "/users",   UserController
  end
end
```

Again `$ mix phoenix.routes` tells us that all of our controllers now have the correct, fully-qualified names.

```elixir
image_path   GET     /images            HelloPhoenix.ImageController :index
review_path  GET     /reviews           HelloPhoenix.ReviewController :index
review_path  GET     /reviews/:id/edit  HelloPhoenix.ReviewController :edit
review_path  GET     /reviews/new       HelloPhoenix.ReviewController :new
review_path  GET     /reviews/:id       HelloPhoenix.ReviewController :show
review_path  POST    /reviews           HelloPhoenix.ReviewController :create
review_path  PATCH   /reviews/:id       HelloPhoenix.ReviewController :update
             PUT     /reviews/:id       HelloPhoenix.ReviewController :update
review_path  DELETE  /reviews/:id       HelloPhoenix.ReviewController :delete
  user_path  GET     /users             HelloPhoenix.UserController :index
  user_path  GET     /users/:id/edit    HelloPhoenix.UserController :edit
  user_path  GET     /users/new         HelloPhoenix.UserController :new
  user_path  GET     /users/:id         HelloPhoenix.UserController :show
  user_path  POST    /users             HelloPhoenix.UserController :create
  user_path  PATCH   /users/:id         HelloPhoenix.UserController :update
             PUT     /users/:id         HelloPhoenix.UserController :update
  user_path  DELETE  /users/:id         HelloPhoenix.UserController :delete
```

Scopes can also be nested, just like resources. Suppose that we had a versioned API with resources defined for images, reviews and users. We could then setup routes for the versioned API like this:

```elixir
scope "/api", HelloPhoenix.Api, as: :api do
  pipe_through :api

  scope "/v1", V1, as: :v1 do
    resources "/images",  ImageController
    resources "/reviews", ReviewController
    resources "/users",   UserController
  end
end
```

`$ mix phoenix.routes` tells us that we have the routes we're looking for.

```elixir
 api_v1_image_path  GET     /api/v1/images HelloPhoenix.Api.V1.ImageController :index
 api_v1_image_path  GET     /api/v1/images/:id/edit HelloPhoenix.Api.V1.ImageController :edit
 api_v1_image_path  GET     /api/v1/images/new HelloPhoenix.Api.V1.ImageController :new
 api_v1_image_path  GET     /api/v1/images/:id HelloPhoenix.Api.V1.ImageController :show
 api_v1_image_path  POST    /api/v1/images HelloPhoenix.Api.V1.ImageController :create
 api_v1_image_path  PATCH   /api/v1/images/:id HelloPhoenix.Api.V1.ImageController :update
                    PUT     /api/v1/images/:id HelloPhoenix.Api.V1.ImageController :update
 api_v1_image_path  DELETE  /api/v1/images/:id HelloPhoenix.Api.V1.ImageController :delete
api_v1_review_path  GET     /api/v1/reviews HelloPhoenix.Api.V1.ReviewController :index
api_v1_review_path  GET     /api/v1/reviews/:id/edit HelloPhoenix.Api.V1.ReviewController :edit
api_v1_review_path  GET     /api/v1/reviews/new HelloPhoenix.Api.V1.ReviewController :new
api_v1_review_path  GET     /api/v1/reviews/:id HelloPhoenix.Api.V1.ReviewController :show
api_v1_review_path  POST    /api/v1/reviews HelloPhoenix.Api.V1.ReviewController :create
api_v1_review_path  PATCH   /api/v1/reviews/:id HelloPhoenix.Api.V1.ReviewController :update
                    PUT     /api/v1/reviews/:id HelloPhoenix.Api.V1.ReviewController :update
api_v1_review_path  DELETE  /api/v1/reviews/:id HelloPhoenix.Api.V1.ReviewController :delete
  api_v1_user_path  GET     /api/v1/users HelloPhoenix.Api.V1.UserController :index
  api_v1_user_path  GET     /api/v1/users/:id/edit HelloPhoenix.Api.V1.UserController :edit
  api_v1_user_path  GET     /api/v1/users/new HelloPhoenix.Api.V1.UserController :new
  api_v1_user_path  GET     /api/v1/users/:id HelloPhoenix.Api.V1.UserController :show
  api_v1_user_path  POST    /api/v1/users HelloPhoenix.Api.V1.UserController :create
  api_v1_user_path  PATCH   /api/v1/users/:id HelloPhoenix.Api.V1.UserController :update
                    PUT     /api/v1/users/:id HelloPhoenix.Api.V1.UserController :update
  api_v1_user_path  DELETE  /api/v1/users/:id HelloPhoenix.Api.V1.UserController :delete
```
Interestingly, we can use multiple scopes with the same path as long as we are careful not to duplicate routes. If we do duplicate a route, we'll get this familiar warning.

```console
warning: this clause cannot match because a previous clause at line 16 always matches
```
This router is perfectly fine with two scopes defined for the same path.

```elixir
defmodule HelloPhoenix.Router do
  use Phoenix.Router
  . . .
  scope "/", HelloPhoenix do
    pipe_through :browser

    resources "users", UserController
  end

  scope "/", AnotherApp do
    pipe_through :browser

    resources "posts", PostController
  end
  . . .
end
```
And when we run `$ mix phoenix.routes`, we see the following output.

```elixir
user_path  GET     /users           HelloPhoenix.UserController :index
user_path  GET     /users/:id/edit  HelloPhoenix.UserController :edit
user_path  GET     /users/new       HelloPhoenix.UserController :new
user_path  GET     /users/:id       HelloPhoenix.UserController :show
user_path  POST    /users           HelloPhoenix.UserController :create
user_path  PATCH   /users/:id       HelloPhoenix.UserController :update
           PUT     /users/:id       HelloPhoenix.UserController :update
user_path  DELETE  /users/:id       HelloPhoenix.UserController :delete
post_path  GET     /posts           AnotherApp.PostController :index
post_path  GET     /posts/:id/edit  AnotherApp.PostController :edit
post_path  GET     /posts/new       AnotherApp.PostController :new
post_path  GET     /posts/:id       AnotherApp.PostController :show
post_path  POST    /posts           AnotherApp.PostController :create
post_path  PATCH   /posts/:id       AnotherApp.PostController :update
           PUT     /posts/:id       AnotherApp.PostController :update
post_path  DELETE  /posts/:id       AnotherApp.PostController :delete
```

### Pipelines

We have come quite a long way in this guide without talking about one of the first lines we saw in the router - `pipe_through :browser`. It's time to fix that.

Remember in the [Overview Guide](http://www.phoenixframework.org/docs/overview) when we described plugs as being stacked and executable in a pre-determined order, like a pipeline? Now we're going to take a closer look at how these plug stacks work in the router.

Pipelines are simply plugs stacked up together in a specific order and given a name. They allow us to customize behaviors and transformations related to the handling of requests. Phoenix provides us with some default pipelines for a number of common tasks. In turn we can customize them as well as create new pipelines to meet our needs.

A newly generated Phoenix application defines two pipelines called `:browser` and `:api`. We'll get to those in a minute, but first we need to talk about the plug stack in the Endpoint plugs.

##### The Endpoint Plugs

Endpoints organize all the plugs common to every request, and apply them before dispatching into the router(s) with their underlying `:browser`, `:api`, and custom pipelines. The default Endpoint plugs do quite a lot of work. Here they are in order.

- [Plug.Static](http://hexdocs.pm/plug/Plug.Static.html) - serves static assets. Since this plug comes before the logger, serving of static assets is not logged

- [Plug.Logger](http://hexdocs.pm/plug/Plug.Logger.html) - logs incoming requests

- [Phoenix.CodeReloader](http://hexdocs.pm/phoenix/Phoenix.CodeReloader.html) - a plug that enables code reloading for all entries in the web directory. It is configured directly in the Phoenix application

- [Plug.Parsers](http://hexdocs.pm/plug/Plug.Parsers.html) - parses the request body when a known parser is available. By default parsers urlencoded, multipart and json (with poison). The request body is left untouched when the request content-type cannot be parsed

- [Plug.MethodOverride](http://hexdocs.pm/plug/Plug.MethodOverride.html) - converts the request method to
  PUT, PATCH or DELETE for POST requests with a valid `_method` parameter

- [Plug.Head](http://hexdocs.pm/plug/Plug.Head.html) - converts HEAD requests to GET requests and strips the response body

- [Plug.Session](http://hexdocs.pm/plug/Plug.Session.html) - a plug that sets up session management.
  Note that `fetch_session/2` must still be explicitly called before using the session as this plug just sets up how the session is fetched

- [Plug.Router](http://hexdocs.pm/plug/Plug.Router.html) - plugs a router into the request cycle

##### The `:browser` and `:api` Pipelines

Phoenix defines two other pipelines by default, `:browser` and `:api`. The router will invoke these after it matches a route, assuming we have called `pipe_through/1` with them in the enclosing scope.

As their names suggest, the `:browser` pipeline prepares for routes which render requests for a browser. The `:api` pipeline prepares for routes which produce data for an api.

The `:browser` pipeline has five plugs: `plug :accepts, ["html"]` which defines the request format or formats which will be accepted, `:fetch_session`, which, naturally, fetches the session data and makes it available in the connection, `:fetch_flash` which retrieves any flash messages which may have been set, as well as `:protect_from_forgery` and `:put_secure_browser_headers`, which protects form posts from cross site forgery.

Currently, the `:api` pipeline only defines `plug :accepts, ["json"]`.

The router invokes a pipeline on a route defined within a scope. If no scope is defined, the router will invoke the pipeline on all the routes in the router. If we call `pipe_through/1` from within a nested scope, the router will invoke it on the inner scope only.

Those are a lot of words bunched up together. Let's take a look at some examples to untangle their meaning.

Here's another look at the router from a newly generated Phoenix application, this time with the api scope uncommented back in and a route added.

```elixir
defmodule HelloPhoenix.Router do
  use HelloPhoenix.Web, :router

  pipeline :browser do
    plug :accepts, ["html"]
    plug :fetch_session
    plug :fetch_flash
    plug :protect_from_forgery
    plug :put_secure_browser_headers
  end

  pipeline :api do
    plug :accepts, ["json"]
  end

  scope "/", HelloPhoenix do
    pipe_through :browser

    get "/", PageController, :index
  end

  # Other scopes may use custom stacks.
  scope "/api", HelloPhoenix do
    pipe_through :api

    resources "reviews", ReviewController
  end
end
```

When the server accepts a request, the request will always first pass through the plugs in our Endpoint, after which it will attempt to match on the path and HTTP verb.

Let's say that the request matches our first route: a GET to `/`. The router will first pipe that request through the `:browser` pipeline - which will fetch the session data, fetch the flash, and execute forgery protection - before it dispatches the request to the `PageController` `index` action.

Conversely, if the request matches any of the routes defined by the `resources/2` macro, the router will pipe it through the `:api` pipeline - which currently does nothing - before it dispatches further to the correct action of the `HelloPhoenix.ReviewController`.

If we know that our application only renders views for the browser, we can simplify our router quite a bit by removing the `api` stuff as well as the scopes:

```elixir
defmodule HelloPhoenix.Router do
  use HelloPhoenix.Web, :router

  pipeline :browser do
    plug :accepts, ["html"]
    plug :fetch_session
    plug :fetch_flash
    plug :protect_from_forgery
    plug :put_secure_browser_headers
  end

  pipe_through :browser

  get "/", HelloPhoenix.PageController, :index

  resources "reviews", HelloPhoenix.ReviewController
end
```
Removing all scopes forces the router to invoke the `:browser` pipeline on all routes.

Let's stretch these ideas out a little bit more. What if we need to pipe requests through both `:browser` and one or more custom pipelines? We simply `pipe_through` a list of pipelines, and Phoenix will invoke them in order.

```elixir
defmodule HelloPhoenix.Router do
  use HelloPhoenix.Web, :router

  pipeline :browser do
    plug :accepts, ["html"]
    plug :fetch_session
    plug :fetch_flash
    plug :protect_from_forgery
    plug :put_secure_browser_headers
  end
  ...

  scope "/reviews" do
    # Use the default browser stack.
    pipe_through [:browser, :review_checks, :other_great_stuff]

    resources "reviews", HelloPhoenix.ReviewController
  end
end
```

Here's another example where nested scopes have different pipelines:

```elixir
defmodule HelloPhoenix.Router do
  use HelloPhoenix.Web, :router

  pipeline :browser do
    plug :accepts, ["html"]
    plug :fetch_session
    plug :fetch_flash
    plug :protect_from_forgery
    plug :put_secure_browser_headers
  end
  ...

  scope "/", HelloPhoenix do
    pipe_through :browser

    resources "posts", PostController

    scope "/reviews" do
      pipe_through :review_checks

      resources "reviews", ReviewController
    end
  end
end
```

In general, the scoping rules for pipelines behave as you might expect. In this example, all routes will pipe through the `:browser` pipeline, because the `/` scope encloses all the routes. Only the `reviews` resources routes will pipe through the `:review_checks` pipeline, because we declare `pipe_through :review_checks` within the `/reviews` scope, where the `reviews` resources routes are located.

##### Creating New Pipelines

Phoenix allows us to create our own custom pipelines anywhere in the router. To do so, we call the `pipeline/2` macro with these arguments: an atom for the name of our new pipeline and a block with all the plugs we want in it.

```elixir
defmodule HelloPhoenix.Router do
  use HelloPhoenix.Web, :router

  pipeline :browser do
    plug :accepts, ["html"]
    plug :fetch_session
    plug :fetch_flash
    plug :protect_from_forgery
    plug :put_secure_browser_headers
  end

  pipeline :review_checks do
    plug :ensure_authenticated_user
    plug :ensure_user_owns_review
  end

  scope "/reviews", HelloPhoenix do
    pipe_through :review_checks

    resources "reviews", ReviewController
  end
end
```

### Channel Routes

Channels are a very exciting, real-time component of the Phoenix framework. Channels handle incoming and outgoing messages broadcast over a socket for a given topic. Channel routes, then, need to match requests by socket and topic in order to dispatch to the correct channel. (For a more detailed description of channels and their behavior, please see the [Channel Guide](http://www.phoenixframework.org/docs/channels).)

We mount socket handlers in our endpoint at `lib/hello_phoenix/endpoint.ex`. Socket handlers take care of authentication callbacks and channel routes.

```elixir
defmodule HelloPhoenix.Endpoint do
  use Phoenix.Endpoint

  socket "/socket", HelloPhoenix.UserSocket
  ...
end
```

Next, we need to open our `web/channels/user_socket.ex` file and use the `channel/3` macro to define our channel routes. The routes will match a topic pattern to a channel to handle events. If we have a channel module called `RoomChannel` and a topic called `"rooms:*"`, the code to do this is straightforward.

```elixir
defmodule HelloPhoenix.UserSocket do
  use Phoenix.Socket

  channel "rooms:*", HelloPhoenix.RoomChannel
  ...
end
```

Topics are just string identifiers. The form we are using here is a convention which allows us to define topics and subtopics in the same string - "topic:subtopic". The `*` is a wildcard character which allows us to match on any subtopic, so `"rooms:lobby"` and `"rooms:kitchen"` would both match this route.

Phoenix abstracts the socket transport layer and includes two transport mechanisms out of the box - WebSockets and Long-Polling. If we wanted to make sure that our channel is handled by only one type of transport, we could specify that using the `via` option, like this.

```elixir
channel "rooms:*", HelloPhoenix.RoomChannel, via: [Phoenix.Transports.WebSocket]
```

Each socket can handle requests for multiple channels.

```elixir
channel "rooms:*", HelloPhoenix.RoomChannel, via: [Phoenix.Transports.WebSocket]
channel "foods:*", HelloPhoenix.FoodChannel
```

We can mount multiple socket handlers in our endpoint:

```elixir
socket "/socket", HelloPhoenix.UserSocket
socket "/admin-socket", HelloPhoenix.AdminSocket
```


### Summary

Routing is a big topic, and we have covered a lot of ground here. The important points to take away from this guide are:
- Routes which begin with an HTTP verb name expand to a single clause of the match function.
- Routes which begin with 'resources' expand to 8 clauses of the match function.
- Resources may restrict the number of match function clauses by using the `only:` or `except:` options.
- Any of these routes may be nested.
- Any of these routes may be scoped to a given path.
- Using the `as:` option in a scope can reduce duplication.
- Using the helper option for scoped routes eliminates unreachable paths.
- Scoped routes may also be nested.
