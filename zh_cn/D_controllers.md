Phoenix 控制器作为协调模块，它的函数 - 称为动作(actions)被路由器调用，响应 HTTP 请求。所谓的动作，在调用视图层渲染模板或返回 JSON 响应之前，会收集所有必要的数据并执行所有必要的步骤。

Phoenix 的控制器也基于 Plug 包，本身也是 plug 中间件。控制器提供了动作中几乎所有我们需要的功能。如果我们发现有些功能 Phoenix 控制器没有提供，或许我们找的东西在 Plug 中。请查看 [Plug 向导](http://www.phoenixframework.org/docs/understanding-plug) or [Plug 文档](http://hexdocs.pm/plug/) 获取更多信息。

新生成的 Phoenix 应用有一个单独的控制器，`PageController`，可以在 `web/controllers/page_controller.ex` 看到它的内容。

```elixir
defmodule HelloPhoenix.PageController do
  use HelloPhoenix.Web, :controller

  def index(conn, _params) do
    render conn, "index.html"
  end
end
```

模块定义下的第一行调用 `HelloPhoenix` 模块的 `__using__/1` 宏，用于导入一些有用的模块。

`PageController` 的 `index` 与相关的定义在 Phoenix 路由的默认路由一起显示出了 Phoenix 欢迎页面。

### 动作

控制器动作就是函数。只要遵循 Elixir 的命名约束，我们将其命名为任何名字。唯一需要满足的就是动作名要和定义在路由里的名称相匹配。

例如，在 `web/router.ex` 我们可以更改 Phoenix 给我们的默认路由 index:

```elixir
get "/", HelloPhoenix.PageController, :index
```

到 test:

```elixir
get "/", HelloPhoenix.PageController, :test
```

只要修改 `PageController` 的动作名为 `test`，欢迎页面和之前一样。

```elixir
defmodule HelloPhoenix.PageController do
  . . .

  def test(conn, _params) do
    render conn, "index.html"
  end
end
```

虽然我们可以随意命名动作，有一些惯用名我们应该尽可能的遵循。我们在 [路由向导](http://www.phoenixframework.org/docs/routing)见过，这里再快速看一看.

- index   - 渲染给定资源类型所有元素的列表
- show    - 渲染单独的给定 id 的元素
- new     - 渲染一个用于创建新元素的表单
- create  - 为新元素接收参数并存入数据存储
- edit    - 检索给定 id 元素并显示在表单中用于编辑
- update  - 为编辑过的元素接收参数并存储数据存储
- delete  - 接收元素 id，并将其从数据存储中删除

每个动作接收由 Phoenix 提供的两个参数。

第一个参数总是 `conn`，它是一个存有像主机(host)，路径元素(path elements)，端口(port)、查询字符串(query string)等更多信息的结构体。`conn`，由 Elixir 的 Plug 中间件框架提供给 Phoenix。更多关于 `conn` 详细信息尽在[plug 文档](http://hexdocs.pm/plug/Plug.Conn.html)。

第二个参数是 `params`。没有悬念的，它是存有所有通过 HTTP 请求传递的参数。模式匹配函数签名的参数，简单解包后用于渲染是比较好的实践。我们在[添加新页面](http://www.phoenixframework.org/docs/adding-pages) `web/controllers/hello_controller.ex` 文件中的传递 messenger 参数到 `show` 路由已经见识过。

```elixir
defmodule HelloPhoenix.HelloController do
  . . .

  def show(conn, %{"messenger" => messenger}) do
    render conn, "show.html", messenger: messenger
  end
end
```

某些情况下 - 通常在 `index` 动作，例如 - 我们不关心参数，因为我们的行为并不取决参数。这种情况，我们不使用传来的参数，知识简单的在变量名前加个下划线, `_params`。解决了既有正确的参数量又有不使用的变量的情况下避免编译器抱怨。

### 收集数据

因为 Phoenix 没有自带数据接入层，Elixir 项目 [Ecto](http://hexdocs.pm/ecto) 使用关系型数据库 [Postgres](http://www.postgresql.org/) 为它提供了非常好的解决方案(未来 Ecto 计划提供其他的转换器)。Phoenix 项目中如何使用 Ecto 会在 [Ecto 模型向导](http://www.phoenixframework.org/docs/ecto-models) 讲到。

当然，还有很多其他的数据接入选项。[Ets](http://www.erlang.org/doc/man/ets.html) 和 [Dets](http://www.erlang.org/doc/man/ets.html) 是 [OTP](http://www.erlang.org/doc/) 内建的键值存储。OTP 也提供了一个关系型数据库 [mnesia](http://www.erlang.org/doc/man/mnesia.html) 和它自带的查询语言 QLC。Elixir 和 Erlang 也有很多用于流行的数据存储的库。

你自己可以掌控数据，教程将不会覆盖它们。

### Flash 信息

很多时候我们需要通过动作跟用户通讯。或许是一个模型更新错误信息，也可能我们只是想欢迎回到应用。我们可以使用 flash 信息来完成。

`Phoenix.Controller` 模块提供了 `put_flash/3` 和 `get_flash/2` 函数帮助我们设定和检索 flash 键值对信息。

为此我们需要修改 `index` 动作如下：

```elixir
defmodule HelloPhoenix.PageController do
  . . .
  def index(conn, _params) do
    conn
    |> put_flash(:info, "Welcome to Phoenix, from flash info!")
    |> put_flash(:error, "Let's pretend we have an error.")
    |> render("index.html")
  end
end
```

`Phoenix.Controller` 模块不是特别关心我们使用什么键。只要我们内部一致就好。`:info` 和 `:error` 都是比较常见的。

为了看到我们的 flash 信息，我们需要能够检索到它们并在模板/视图显示。完成第一部分要做的就是 `get_flash/2` 接收 `conn` 和我们关系的键。它就会返回这个键的值。

幸运的是，我们的应用布局文件 `web/templates/layout/app.html.eex` 已经有了显示 flash 信息的标记。

```html
<p class="alert alert-info" role="alert"><%= get_flash(@conn, :info) %></p>
<p class="alert alert-danger" role="alert"><%= get_flash(@conn, :error) %></p>
```

当我们刷新 [欢迎页面](http://localhost:4000/)，我们的信息就会出现在 "Welcome to Phoenix!" 之上。

除了 `put_flash/3` 和 `get_flash/2` 之外，`Phoenix.Controller` 模块还有另外一个值得知道的有用的函数。`clear_flash/1` 只接收 `conn` 参数，移除所有可能存储在 session 中的 falsh 信息。

### 渲染

控制器有多种渲染内容的方式。渲染文字最简单的方式是使用 Phoenix 提供的 `text/2` 函数。

如果我们有一个 `show` 动作从参数 map 收到一个 id，我们所要做的是用 id 返回一些文本，可以这么做。

```elixir
def show(conn, %{"id" => id}) do
  text conn, "Showing id #{id}"
end
```

假设我们有一个 `get "/our_path/:id` 路由映射到这个 `show` 动作，浏览器访问 `/our_path/15` 应该显示 `Showing id 15` 文本，无任何 HTML。

更近一步的是使用 `json/2` 渲染纯 JSON。我们需要传递 [Posion 库](https://github.com/devinus/poison) 可以解析的的东西，例如 map（Poison 是 Phoenix 其中一个依赖)。

```elixir
def show(conn, %{"id" => id}) do
  json conn, %{id: id}
end
```

如果我们再次在浏览器访问 `our_path/15` ，我们应该看到一个 键 `id` 映射为 `15` 值的 JSON。

```elixir
{
  id: "15"
}
```

Phoenix 控制器不用模板也可以渲染 HTML。跟你猜的一样， `html/2` 函数就是这个作用。这次，我们这样实现 `show` 动作。

```elixir
def show(conn, %{"id" => id}) do
  html conn, """
     <html>
       <head>
          <title>Passing an Id</title>
       </head>
       <body>
         <p>You sent in id #{id}</p>
       </body>
     </html>
    """
end
```

访问 `/our_path/15` 现在渲染的是我们定义在 `show` 动作内的 HTML 字符，`15` 字符也已被插入。注意我们写在动作内的不是 `eex` 模板。它是一个多行字符串，这样我们就可以像 `#{id}` 这样，而不是 `<%= id %>` 插入 `id` 变量。

It is worth noting that the `text/2`, `json/2`, and `html/2` functions require neither a Phoenix view, nor a template to render.
`text/2`, `json/2` 和 `html/2` 函数既不需要视图也不需要模板来渲染。

`json/2` 函数明显对编写 API 有用，其他两个非常便携，但是将一个模板和传递给它变量通过布局渲染是非常常见的事情。

为此，Phoenix 提供了 `render/3` 函数。

有趣的是,`render/3` 定义在 `Phoenix.View` 模块内，而不是 `Phoenix.Controller`内，只是方便起见别名在了 `Phoenix.Controller`。

在[添加新页面](http://www.phoenixframework.org/docs/adding-pages)，我们已经见过渲染函数。`web/controlleres/hello_controller.ex` 的 `show` 动作如下：

```elixir
defmodule HelloPhoenix.HelloController do
  use HelloPhoenix.Web, :controller

  def show(conn, %{"messenger" => messenger}) do
    render conn, "show.html", messenger: messenger
  end
end
```

为了让 `render/3` 正常工作，控制器必须和单独视图有相同的根名称。单独的视图也必须和 `show.html.eex` 存在的模板文件夹有相同的根名称。换句话说，`HelloController` 需要 `HelloView`，`HelloView` 需要 `web/templates/hello` 文件夹，这个文件夹必须包含 `show.html.eex` 模板。

`render/3` will also pass the value which the `show` action received for `messenger` from the params hash into the template for interpolation.
`render/3` 也将传递 `show` 动作从 `messenger` 接收到的值，从参数哈希(params hash) 给模板进行插值。

If we need to pass values into the template when using `render`, that's easy. We can pass a dictionary like we've seen with `messenger: messenger`, or we can use `Plug.Conn.assign/3`, which conveniently returns `conn`.

```elixir
def index(conn, _params) do
  conn
  |> assign(:message, "Welcome Back!")
  |> render("index.html")
end
```
Note: The `Phoenix.Controller` module imports `Plug.Conn`, so shortening the call to `assign/3` works just fine.

We can access this message in our `index.html.eex` template, or in our layout, with this `<%= @message %>`.

Passing more than one value in to our template is as simple as connecting `assign/3` functions together in a pipeline.

```elixir
def index(conn, _params) do
  conn
  |> assign(:message, "Welcome Back!")
  |> assign(:name, "Dweezil")
  |> render("index.html")
end
```
With this, both `@message` and `@name` will be available in the `index.html.eex` template.

What if we want to have a default welcome message that some actions can override? That's easy, we just use `plug` and transform `conn` on its way towards the controller action.

```elixir
plug :assign_welcome_message, "Welcome Back"

def index(conn, _params) do
  conn
  |> assign(:name, "Dweezil")
  |> render("index.html")
end

defp assign_welcome_message(conn, msg) do
  assign(conn, :message, msg)
end
```

What if we want to plug `assign_welcome_message`, but only for some of our actions? Phoenix offers a solution to this by letting us specify which actions a plug should be applied to. If we only wanted `plug :assign_welcome_message` to work on the `index` and `show` actions, we could do this.

```elixir
defmodule HelloPhoenix.PageController do
  use HelloPhoenix.Web, :controller

  plug :assign_welcome_message, "Hi!" when action in [:index, :show]
. . .
```

### Sending responses directly

If none of the rendering options above quite fits our needs, we can compose our own using some of the functions that Plug gives us. Let's say we want to send a response with a status of "201" and no body whatsoever. We can easily do that with the `send_resp/3` function.

```elixir
def index(conn, _params) do
  conn
  |> send_resp(201, "")
end
```

Reloading [http://localhost:4000](http://localhost:4000) should show us a completely blank page. The network tab of our browser's developer tools should show a response status of "201".

If we would like to be really specific about the content type, we can use `put_resp_content_type/2` in conjunction with `send_resp/3`.

```elixir
def index(conn, _params) do
  conn
  |> put_resp_content_type("text/plain")
  |> send_resp(201, "")
end
```

Using Plug functions in this way, we can craft just the response we need.

Rendering does not end with the template, though. By default, the results of the template render will be inserted into a layout, which will also be rendered.

[Templates and layouts](http://www.phoenixframework.org/docs/templates) have their own guide, so we won't spend much time on them here. What we will look at is how to assign a different layout, or none at all, from inside a controller action.

### Assigning Layouts

Layouts are just a special subset of templates. They live in `/web/templates/layout`. Phoenix created one for us when we generated our app. It's called `app.html.eex`, and it is the layout into which all templates will be rendered by default.

Since layouts are really just templates, they need a view to render them. This is the `LayoutView` module defined in `/web/views/layout_view.ex`. Since Phoenix generated this view for us, we won't have to create a new one as long as we put the layouts we want to render inside the `/web/templates/layout` directory.

Before we create a new layout, though, let's do the simplest possible thing and render a template with no layout at all.

The `Phoenix.Controller` module provides the `put_layout/2` function for us to switch layouts. This takes `conn` as its first argument and a string for the basename of the layout we want to render. Another clause of the function will match on the boolean `false` for the second argument, and that's how we will render the Phoenix welcome page without a layout.

In a freshly generated Phoenix app, edit the `index` action of the `PageController` module `web/controllers/page_controller.ex` to look like this.

```elixir
def index(conn, params) do
  conn
  |> put_layout(false)
  |> render "index.html"
end
```
After reloading [http://localhost:4000/](http://localhost:4000/), we should see a very different page, one with no title, logo image, or css styling at all.

Very Important! For function calls in the middle of a pipeline, like `put_layout/2` here, it is critical to use parenthesis around the arguments because the pipe operator binds very tightly. This leads to parsing problems and very strange results.

If you ever get a stack trace that looks like this,

```text
**(FunctionClauseError) no function clause matching in Plug.Conn.get_resp_header/2

Stacktrace

    (plug) lib/plug/conn.ex:353: Plug.Conn.get_resp_header(false, "content-type")
```

where your argument replaces `conn` as the first argument, one of the first things to check is whether there are parentheses in the right places.

This is fine.

```elixir
def index(conn, params) do
  conn
  |> put_layout(false)
  |> render "index.html"
end
```

Whereas this won't work.

```elixir
def index(conn, params) do
  conn
  |> put_layout false
  |> render "index.html"
end
```

Now let's actually create another layout and render the index template into it. As an example, let's say we had a different layout for the admin section of our application which didn't have the logo image. To do this, let's copy the existing `app.html.eex` to a new file `admin.html.eex` in the same directory `web/templates/layout`. Then let's remove the line in `admin.html.eex` that displays the logo.

```html
<span class="logo"></span> <!-- remove this line -->
```

Then, pass the basename of the new layout into `put_layout/2` in our `index` action in `web/controllers/page_controller.ex`.

```elixir
def index(conn, params) do
  conn
  |> put_layout("admin.html")
  |> render "index.html"
end
```
When we load the page, and we should be rendering the admin layout without a logo.

### Overriding Rendering Formats

Rendering HTML through a template is fine, but what if we need to change the rendering format on the fly? Let's say that sometimes we need HTML, sometimes we need plain text, and sometimes we need JSON. Then what?

Phoenix allows us to change formats on the fly with the `_format` query string parameter. To make this happen, Phoenix requires an appropriately named view and an appropriately named template in the correct directory.

As an example, let's take the `PageController` index action from a newly generated app. Out of the box, this has the right view, `PageView`, the right templates directory, `/web/templates/page`, and the right template for rendering HTML, `index.html.eex`.

```elixir
def index(conn, _params) do
  render conn, "index.html"
end
```
What it doesn't have is an alternative template for rendering text. Let's add one at `/web/templates/page/index.text.eex`. Here is our example `index.text.eex` template.

```elixir
"OMG, this is actually some text."
```

There are just few more things we need to do to make this work. We need to tell our router that it should accept the `text` format. We do that by adding `text` to the list of accepted formats in the `:browser` pipeline. Let's open up `web/router.ex` and change the `plug :accepts` to include `text` as well as `html` like this.

```elixir
defmodule HelloPhoenix.Router do
  use HelloPhoenix.Web, :router

  pipeline :browser do
    plug :accepts, ["html", "text"]
    plug :fetch_session
    plug :protect_from_forgery
    plug :put_secure_browser_headers
  end
. . .
```

We also need to tell the controller to render a template with the same format as the one returned by `Phoenix.Controller.get_format/1`. We do that by substituting the atom version of the template `:index` for the string version `"index.html"`.

```elixir
def index(conn, _params) do
  render conn, :index
end
```

If we go to [http://localhost:4000/?_format=text](http://localhost:4000/?_format=text), we will see `OMG, this is actually some text.`

Of course, we can pass data into our template as well. Let's change our action to take in a message parameter by removing the `_` in front of `params` in the function definition. This time, we'll use the somewhat less-flexible string version of our text template, just to see that it works as well.

```elixir
def index(conn, params) do
  render conn, "index.text", message: params["message"]
end
```

And let's add a bit to our text template.

```elixir
"OMG, this is actually some text." <%= @message %>
```

Now if we go to `http://localhost:4000/?_format=text&message=CrazyTown`, we will see "OMG, this is actually some text. CrazyTown"

### Setting the Content Type

Analogous to the `_format` query string param, we can render any sort of format we want by modifying the HTTP Accepts Header and providing the appropriate template.

If we wanted to render an xml version of our `index` action, we might implement the action like this in `web/page_controller.ex`.

```elixir
def index(conn, _params) do
  conn
  |> put_resp_content_type("text/xml")
  |> render "index.xml", content: some_xml_content
end
```

We would then need to provide an `index.xml.eex` template which created valid xml, and we would be done.

For a list of valid content mime-types, please see the [mime.types](https://github.com/elixir-lang/plug/blob/master/lib/plug/mime.types) documentation from the Plug middleware framework.

### Setting the HTTP Status

We can also set the HTTP status code of a response similarly to the way we set the content type. The `Plug.Conn` module, imported into all controllers, has a `put_status/2` function to do this.

`put_status/2` takes `conn` as the first parameter and as the second parameter either an integer or a "friendly name" used as an atom for the status code we want to set. Here is the list of supported [friendly names](https://github.com/elixir-lang/plug/blob/master/lib/plug/conn/status.ex#L7-L63).

Let's change the status in our `PageController` `index` action.

```elixir
def index(conn, _params) do
  conn
  |> put_status(202)
  |> render("index.html")
end
```

The status code we provide must be valid - [Cowboy](https://github.com/ninenines/cowboy), the web server Phoenix runs on, will throw an error on invalid codes. If we look at our development logs (which is to say, the iex session), or use our browser's web inspection network tool, we will see the status code being set as we reload the page.

If the action sends a response - either renders or redirects - changing the code will not change the behavior of the response. If, for example, we set the status to 404 or 500 and then `render "index.html"`, we do not get an error page. Similarly, no 300 level code will actually redirect. (It wouldn't know where to redirect to, even if the code did affect behavior.)

The following implementation of the `HelloPhoenix.PageController` `index` action, for example, will _not_ render the default `not_found` behavior as expected.

```elixir
def index(conn, _params) do
  conn
  |> put_status(:not_found)
  |> render("index.html")
end
```

The correct way to render the 404 page from `HelloPhoenix.PageController` is:

```elixir
def index(conn, _params) do
  conn
  |> put_status(:not_found)
  |> render(HelloPhoenix.ErrorView, "404.html")
end
```

### Redirection

Often, we need to redirect to a new url in the middle of a request. A successful `create` action, for instance, will usually redirect to the `show` action for the model we just created. Alternately, it could redirect to the `index` action to show all the things of that same type. There are plenty of other cases where redirection is useful as well.

Whatever the circumstance, Phoenix controllers provide the handy `redirect/2` function to make redirection easy. Phoenix differentiates between redirecting to a path within the application and redirecting to a url - either within our application or external to it.

In order to try out `redirect/2`, let's create a new route in `web/router.ex`.

```elixir
defmodule HelloPhoenix.Router do
  use HelloPhoenix.Web, :router
  . . .

  scope "/", HelloPhoenix do
    . . .
    get "/", PageController, :index
  end

  # New route for redirects
  scope "/", HelloPhoenix do
    get "/redirect_test", PageController, :redirect_test, as: :redirect_test
  end
  . . .
end
```

Then we'll change the `index` action to do nothing but redirect to our new route.

```elixir
def index(conn, _params) do
  redirect conn, to: "/redirect_test"
end
```

Finally, let's define in the same file the action we redirect to, which simply renders the text `Redirect!`.

```elixir
def redirect_test(conn, _params) do
  text conn, "Redirect!"
end
```

When we reload our [Welcome Page](http://localhost:4000), we see that we've been redirected to `/redirect_test` which has rendered the text `Redirect!`. It works!

If we care to, we can open up our developer tools, click on the network tab, and visit our root route again. We see two main requests for this page - a get to `/` with a status of `302`, and a get to `/redirect_test` with a status of `200`.

Notice that the redirect function takes `conn` as well as a string representing a relative path within our application. It can also take `conn` and a string representing a fully-qualified url.

```elixir
def index(conn, _params) do
  redirect conn, external: "http://elixir-lang.org/"
end
```

We can also make use of the path helpers we learned about in the [Routing Guide](http://www.phoenixframework.org/docs/routing). It's useful to `alias` the helpers in `web/router.ex` in order to shorten the expression.

```elixir
defmodule HelloPhoenix.PageController do
  use HelloPhoenix.Web, :controller

  def index(conn, _params) do
    redirect conn, to: redirect_test_path(conn, :redirect_test)
  end
end
```

Note that we can't use the url helper here because `redirect/2` using the atom `:to`, expects a path. For example, the following will fail.

```elixir
def index(conn, _params) do
  redirect conn, to: redirect_test_url(conn, :redirect_test)
end
```

If we want to use the url helper to pass a full url to `redirect/2`, we must use the atom `:external`. Note that the url does not have to be truly external to our application to use `:external`, as we see in this example.

```elixir
def index(conn, _params) do
  redirect conn, external: redirect_test_url(conn, :redirect_test)
end
```
