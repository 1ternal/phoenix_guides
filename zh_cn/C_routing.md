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

再次运行 `$ mix phoenix.routes`，控制器单独写得到的结果是一样的。

更棒的是，给 scope 添加别名选项也可以嵌套，控制器名称也就不会重复了。

Phoenix 新生成的应用生成的路由就已经这么做了(参见本节开始部分)。留意定义 `defmodule` 时的 `HelloPhoenix.Router` 的使用。

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

`$ mix phoenix.routes` 再次显示现在所有的控制器都是完整名称。

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

和资源(resource)一样，scopes 也可以嵌套。假设我们有个版本 API 定义了 图片、测评和用户资源。可以这样设置：

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

`$ mix phoenix.routes` 显示了我们想要的路由。

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

有趣的是，同一个路径下可以使用多个 scope，小心路由别重合就行。如果有重复路由，会得到这个熟悉的警告：

```console
warning: this clause cannot match because a previous clause at line 16 always matches
```

同一路径下定义俩 scope 的路由非常完美。

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

运行 `$ mix phoenix.routes` 就可以看到这样的结果：

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

### 管道(Pipelines)

教程到这里已经很长了，还没提到路由器看到的第一行 `pipe_through :browser`。时候解决这个问题了。

记得在[总览向导](http://www.phoenixframework.org/docs/overview)说到 plug 堆叠一起，以定好的顺序执行，像流水线一样？

管道就是有名称的多个 plug 中间件以特定的顺序堆叠在一起的栈。可自定义行为，也可以对相关的请求进行变换。Phoenix 给我们提供了一定数量的默认管道。可根据需求自定义或创建新的管道。

新生成的 Phoenix 应用定义了 `:browser` 和 `:api` 这两个管道。稍后会讲到它们，但是现在我们需要讲一讲 Endpoint 的 plug 栈。

##### Endpoint Plugs

Endpoint 将所有通用的 plug 应用于每个请求，分发至路由器之前，再把 `:browser`，`:api` 和 自定义的管道应用到请求上。默认的 Endpoint plug 中间件做了很多工作，下面依次介绍：

- [Plug.Static](http://hexdocs.pm/plug/Plug.Static.html) - 提供静态资源。因为此中间件在 logger 之前，所以它的信息不会被日志记录。

- [Plug.Logger](http://hexdocs.pm/plug/Plug.Logger.html) - 记录连入请求日志。

- [Phoenix.CodeReloader](http://hexdocs.pm/phoenix/Phoenix.CodeReloader.html) - 为 web 目录下所有文件提供代码重载功能。由 Phoenix 应用直接配置。

- [Plug.Parsers](http://hexdocs.pm/plug/Plug.Parsers.html) - 解析已知、可用的请求内容(request body)。默认解析器为 urlencoded，multipart 和 json(使用 poison)。当请求的内容类型(content-type)不能被解析，会原封不动的传递给下一个中间件。

- [Plug.MethodOverride](http://hexdocs.pm/plug/Plug.MethodOverride.html) - 将 POST 的 PUT, PATCH 或 DELETE 请求方法转换为合法的 `_method` 参数

- [Plug.Head](http://hexdocs.pm/plug/Plug.Head.html) - 转换 HEAD 请求到 GET 请求，并移除响应内容(response body)。

- [Plug.Session](http://hexdocs.pm/plug/Plug.Session.html) - 设置 session 管理。
  注意在使用 session 管理中间件设定如何获取 session 之前， `fetch_session/2` 必须明确调用。

- [Plug.Router](http://hexdocs.pm/plug/Plug.Router.html) - 将路由器加入请求循环(request cycle)。

##### `:browser` 和 `:api` 管道

Phoenix 定义了其他两个默认的管道，`:browser` 和 `:api`。如果在 scope 区间内使用 `pipe_through` 定义了它们， 路由匹配后才会调用。

和名称说明的一样，`:browser` 管道是为渲染到浏览器准备的。`:api` 管道是为生产 api 数据准备的。

`:browser` 管道有五个中间件: `plug :accepts, ["html"]` 定义了请求格式 或者说是接受哪些格式，`:fetch_session` 很自然就是获取 session 数据在连接中使用，`:fetch_flash` 取出所有可能被设定的 flash 信息，`:protect_from_forgery` 和 `:put_secure_browser_headers` 则用于保护跨扎伪造。

目前，`:api` 管道只定义了 `plug :accepts, ["json"]`

路由器调用的管道定义在 scope 内。如果没有定义 scope,路由器将会在所有的路由条目上应用管道。如果在嵌套的 scope 内调用 `pipe_through/1` 管道则只应用于内部 scope。

已经讲了很多。看些例子有助于我们理解。

这是一个新的由 Phoenix 应用生成的路由，取消了 api scope 注释并添加了一条路由。

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

服务器接受到一个请求时，请求将先通过 Endpoint 的中间件，再尝试匹配路径和 HTTP 动词。

假设这个请求匹配了我们第一条路由：GET 请求 `\`。路由器将请求先送到 `:browser` 管道，进行执行获取 session 数据，取 flash 和执行伪造保护，然后再分发到 `PageController` 的 `index` 动作。

相应的，如果请求匹配任何 `resource/2` 宏定义的路由，将会送到目前还没什么作用的`:api` 管道，然后再分发至 `HelloPhoenix.ReviewController` 控制器。

如果我们只需要为浏览器渲染，那路由可以简化一些，移除 `api` 和相关的 scope 路由即可。

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

移除所有的 scope 会强制调用 `:browser` 这个管道。

再扩展一点，如果需要经过 `:browser` 和另外一个或更多的自定义管道怎么办？`pipe_through` 管道列表即可，Phoenix 将会按顺序调用。

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

这儿还有一个嵌套 scope 有不同 通道的例子：

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

总之，通道的 scope 规则和你期待的一致。这个例子，所有路由都会通过 `:browser` 通道，因为 `/` scope 包含了所有路由。因为在 `/reviews` scope，也是 `review` 资源所在的 scope 宣告了 `pipe_through :review_checks`，所以 `review` 资源会通过 `:review_checks` 通道。

##### 创建新通道

Phoenix 允许我们在路由任何地方创建自己的通道。调用 `pipeline/2` 宏和这俩参数：新通道的原子名称和一个我们想要调用的 plug 中间件 block。


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

### Channel 路由

Channels 是 Phoenix 框架的一个令人兴奋的实时组件。Channel 通过 socket 广播给定的话题(topic)的出入站消息。为了分发至正确的 channel 频道，Channel 路由需要匹配socket请求和话题(请查看[Channel 向导](http://www.phoenixframework.org/docs/channels)获得更详细的信息)。

在 `lib/hello_phoenix\endpoint.ex` 挂载 socket 处理器到 endpoint。Socket 处理器负责认证回调和 channel 路由。

```elixir
defmodule HelloPhoenix.Endpoint do
  use Phoenix.Endpoint

  socket "/socket", HelloPhoenix.UserSocket
  ...
end
```

接着，我们需要打开 `web/channels/user_socket.ex` 文件，使用 `channel/3` 宏定义 channel 路由。路由匹配一个话题模式(topic pattern)到一个 channel 来处理事件。如果我们的 channel 模块叫 `RoomChannel`，话题叫 `"rooms:*"`，那要写的代码很直接：

```elixir
defmodule HelloPhoenix.UserSocket do
  use Phoenix.Socket

  channel "rooms:*", HelloPhoenix.RoomChannel
  ...
end
```

话题(topic)就是字符串标示。这里惯用法，定义话题和子话题在同一个字符串 - "topic:subtopic"。`*` 号是通配符，允许匹配任何子话题，所以 `"rooms:lobby"` 和 `"rooms:kitchen"` 都会匹配到这个路由。

Phoenix 抽象了 socket 传输层，包含了两个传输机制 - WebSocket 和 Long-Polling。如果我们确认只使用一种传输机制，可以使用 `via` 选项来指定，像这样：

```elixir
channel "rooms:*", HelloPhoenix.RoomChannel, via: [Phoenix.Transports.WebSocket]
```

每个 socket 可以处理多个 channel 请求。

```elixir
channel "rooms:*", HelloPhoenix.RoomChannel, via: [Phoenix.Transports.WebSocket]
channel "foods:*", HelloPhoenix.FoodChannel
```

我们可以在 endpoint 挂载多个 socket 处理器:

```elixir
socket "/socket", HelloPhoenix.UserSocket
socket "/admin-socket", HelloPhoenix.AdminSocket
```

### 总结

路由是个大话题，我们这里覆盖了不少内容。本篇提到的重点是：
- 以 HTTP 动词开始的路由展开为单条 match 函数。
- 以 `resource` 开始的路由展开为 8 条 match 函数子句。
- 资源可以使用 `only:` 或 `except:` 选项限制 match 函数子句的数量。
- 任何路由都可以被嵌套。
- 任何路由都可以限定在给定的路径下。
- scope 使用 `as:` 选项消除重复。
- scope 使用帮助选项消除不可及路径。
- 被限定的路由也可以嵌套。
