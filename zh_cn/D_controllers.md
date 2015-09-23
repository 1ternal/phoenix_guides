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

如果需要使用 `render` 传递数据给模板，很简单，我们可以传递一个如之前所见 `messenger: messenger` 类似的字典，或者使用方便返回 `conn` 的 `Plug.Conn.assign/3` 函数。

```elixir
def index(conn, _params) do
  conn
  |> assign(:message, "Welcome Back!")
  |> render("index.html")
end
```

注意：`Phoenix.Controller` 模块导入了 `Plug.Conn`，所以直接调用 `assign/3` 可以正常工作。

我们可以在 `index.html.eex` 模板中获取这个信息，也可以在布局中使用 `<%= @message %>` 获取。

向模板中传递多个值与使用管道将 `assign/3` 连接起来一样简单。

```elixir
def index(conn, _params) do
  conn
  |> assign(:message, "Welcome Back!")
  |> assign(:name, "Dweezil")
  |> render("index.html")
end
```

这样，`@message` 和 `@name` 在模板 `index.html.eex` 中就都可用了。

我们需要一个某些动作可以覆盖的默认欢迎信息该怎么办呢？很简单，使用 `plug` 并在控制器动作之前转换 `conn` 即可。

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

如果我们只想在一些动作中使用 plug `assign_welcome_message` 呢？Phoenix 为此提供了一个解决方法，可以让我们指定应用于哪些动作。假如我们只想 `plug :assign_welcome_message` 到 `index` 和 `show` 动作，可以这么做。

```elixir
defmodule HelloPhoenix.PageController do
  use HelloPhoenix.Web, :controller

  plug :assign_welcome_message, "Hi!" when action in [:index, :show]
. . .
```

### 直接发送响应

如果上面的渲染方式都不符合需求，我们可以使用 Plug 提供给我们的一些函数进行组合。假如我们想发送无响应内容 “201” 状态。使用 `send_resp/3` 函数就非常简单。

```elixir
def index(conn, _params) do
  conn
  |> send_resp(201, "")
end
```

重新载入后，[http://localhost:4000](http://localhost:4000) 应该显示一个完全空白的页面。我们浏览器开发者工具的网络标签应该显示响应状态为 "201"。

如果我们想指定内容类型，可以使用 `put_resp_content_type` 和 `send_resp` 一起来完成。

```elixir
def index(conn, _params) do
  conn
  |> put_resp_content_type("text/plain")
  |> send_resp(201, "")
end
```

这样使用 Plug 函数，我们可以完成需要的响应。

渲染并不是以模板作为结束。默认，模板的结果将会插入到一个同样被渲染的布局(layout)中。

[模板和布局](http://www.phoenixframework.org/docs/templates) 有单独的章节，所以这里不会花太多的时间。我们要关注的是如何在控制器动作中分配不同的布局，或者不用布局。

### 分配布局

布局只是模板的特殊子集。它们在 `web/templates/layout` 文件夹内。当我们生成应用时，Phoenix 会为我们创建一个布局。叫 `app.html.eex`，默认，它是所有模板渲染的布局。

因为布局实际只是模板，需要视图来渲染。也就是定义在 `web/views/layout_view.ex` 内的 `LayoutView` 模块。Phoenix 已经为我们生成这个视图，所以只要将我们想要渲染的布局放入 `/web/templates/layout` 文件夹就不需要创建新的视图。

创建一个新布局之前，让我们先做一个可能最简单的事情，不用布局来渲染一个模板。

`Phoenix.Controller` 模块提供的 `put_layout/2` 函数用于切换布局。它接受一个 `conn` 参数和想要渲染布局名称的字符串。此函数另一个子句的第二个参数将匹配布尔值 `false`，这就是我们不使用布局渲染欢迎页面的方法。

在新生成的 Phoenix 应用内，修改 `web/controllers/page_controller.ex` 文件的 `PageController` 模块的 `index` 动作如下：

```elixir
def index(conn, params) do
  conn
  |> put_layout(false)
  |> render "index.html"
end
```

重新载入 [http://localhost:4000/](http://localhost:4000/)，我们应该看到一个非常不同的页面，没有标题，没有图片，也完全没有 css 样式。

非常重要！在管道中的函数调用，像这里的 `put_layout`，参数使用括号包裹非常关键，因为管道操作符优先级非常高。它会导致解析问题和非常奇怪的结果。

如果你曾看到类似这样的 stack trace,

```text
**(FunctionClauseError) no function clause matching in Plug.Conn.get_resp_header/2

Stacktrace

    (plug) lib/plug/conn.ex:353: Plug.Conn.get_resp_header(false, "content-type")
```

你的参数代替 `conn` 作为第一参数，首先要检查的就是括号是否在正确的位置。

这样就没问题。

```elixir
def index(conn, params) do
  conn
  |> put_layout(false)
  |> render "index.html"
end
```

这样不行。

```elixir
def index(conn, params) do
  conn
  |> put_layout false
  |> render "index.html"
end
```

现在让我们创建另一个布局然后将 index 模板渲染其中。作为一个例子，假设应用的管理员部分有不同的布局，并没有 logo 图片。
为此，在 `web/templates/layout` 相同目录下，拷贝 `app.html.eex` 为一个新文件 `admin.html.eex`。然后移除 `admin.html.eex` 文件中显示 logo 的代码。

```html
<span class="logo"></span> <!-- remove this line -->
```

然后传递新布局的文件名给 `web/controllers/page_controller.ex` 中 `index` 动作里的 `put_layout/2` 函数。

```elixir
def index(conn, params) do
  conn
  |> put_layout("admin.html")
  |> render "index.html"
end
```

页面载入后，我们应该渲染了一个没有 logo 的管理员布局。

### 覆盖渲染格式

通过模板可以很好的渲染 HTML, 当我们需要更改渲染格式该怎么办呢? 比如说有些时候我们需要 HTML,有些时候需要文本,还有可能需要渲染JSON,怎么做?

渲染的时候,Phoenix 允许使用查询字符串参数 `_format` 修改格式.为此,Phoenix 要求在正确的目录下有准确的视图和模板.

以新生成的 `PageController` 的 index 动作为例.生成后,它有正确的视图 `PageView`,正确的模板目录 `/web/templates/pages` 和正确的渲染 HTML 的模板 `index.html.eex`.

```elixir
def index(conn, _params) do
  render conn, "index.html"
end
```

它没有提供的是可选的渲染文本的模板.我们添加 `/web/templates/page/index.text.eex` 文件.示例 `index.text.eex` 如下所示.

```elixir
"OMG, this is actually some text."
```

为了渲染文本,还需要做一些其他事情.我们需要告诉路由器可以接受 `text` 格式.在 `:browser` 管道内添加 `text` 到接受格式的列表内.打开 `web/router.ex`,像下面这样修改 `plug :accepts` 包含 `text` 和 `html`.

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

还需要告诉控制器使用与 `Phoenix.Controller.get_format/1` 返回值相同格式的模板.使用原子格式 `:index` 模板代替字符串格式 `"index.html"` 就可以了.

```elixir
def index(conn, _params) do
  render conn, :index
end
```

如果我们浏览 [http://localhost:4000/?_format=text](http://localhost:4000/?_format=text) 页面,就会看到 `OMG, this is actually some text.`

当然,我们还可以传递数据给模板.修改动作,在函数定义处移除 `params` 前面的 `_` 来接受一个消息参数.这次,我们将使用兼容性稍差的字符串模板,确保它能正常工作.

```elixir
def index(conn, params) do
  render conn, "index.text", message: params["message"]
end
```

给我们的文本模板增加点东西.

```elixir
"OMG, this is actually some text." <%= @message %>
```

现在浏览 `http://localhost:4000/?_format=text&message=CrazyTown` 页面,将会看到 "OMG, this is actually some text. CrazyTown"

### 设置内容类型(Content Type)

和 `_format` 查询字符参数类似,通过修改 HTTP 所接受的 Header 和 合适的模板,我们可以渲染任何想要的格式.

如果我们想渲染一个 xml 版本的 `index` 动作,在 `web/page_controller.ex` 可以这样实现.

```elixir
def index(conn, _params) do
  conn
  |> put_resp_content_type("text/xml")
  |> render "index.xml", content: some_xml_content
end
```

我们还需要提供一个 `index.xml.eex` 模板来创建合法的 xml 文件.

所有合法的 mime-types 列表,请查看 plug 中间件框架的[mime.types](https://github.com/elixir-lang/plug/blob/master/lib/plug/mime.types) 文档.

### 设置 HTTP 状态码(Status)

与设置 Content-Type 类似,我们也可以设置响应的 HTTP 状态码.导入到所有控制器的 `Plug.Conn` 的 `put_status/2` 函数就是这个作用.

`put_status/2` 函数接受的第一个参数是 `conn`,第二个参数是可以是整型数值或我们想要设置的状态码的"友好"名字的原子.这是支持的 [友好名称](https://github.com/elixir-lang/plug/blob/master/lib/plug/conn/status.ex#L7-L63).

在 `PageController` 的 `index` 内修改状态码.

```elixir
def index(conn, _params) do
  conn
  |> put_status(202)
  |> render("index.html")
end
```

我们提供的状态码必须是合法的 - 跑在其上的 [Cowboy](https://github.com/ninenines/cowboy) web 服务器,将会对非法的状态码抛出错误.刷新页面后,如果我们查看开发日志(也就是 iex 会话),或者使用浏览器的 web 检查网络工具,会看到状态码已被设定.

如果动作发送一个响应 - 无论渲染或跳转 - 修改状态码不会更改响应的行为.假如我们设定状态码为 404 或 500,再 `render "index.html"`, 不会出现错误页面.类似的,没有 3xx 状态码也能进行跳转(就算状态码确实影响了行为,它也不知道跳转到哪里).

下面实现的 `HelloPhoenix.PageController` 的 `index` 动作, **不** 会渲染期望的默认 `not_found` 行为.


```elixir
def index(conn, _params) do
  conn
  |> put_status(:not_found)
  |> render("index.html")
end
```

从 `HelloPhoenix.PageController` 渲染 404 页面正确的方式:

```elixir
def index(conn, _params) do
  conn
  |> put_status(:not_found)
  |> render(HelloPhoenix.ErrorView, "404.html")
end
```

### 跳转

我们经常需要在请求中间跳转到一个新的 url 地址.拿执行成功的 `create` 来说,通常需要跳转到刚创建的模型页面.也可以选择跳转到 `index` 动作来显示所有东西.跳转在很多其他的情况也非常有用.

任何情况下,Phoenix 控制器提供的 `redirect/2` 函数,都能很简单进行跳转.Phoenix 会区分应用内跳转和跳转到一个 url 地址 - 不是应用内部就是应用的外部.

为了尝试 `redirect/2`,在 `web/router.ex` 我们创建一个新的路由.

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

然后我们将修改 `index` 动作成除跳转到新的路由上不做其他任何事.

```elixir
def index(conn, _params) do
  redirect conn, to: "/redirect_test"
end
```

最后,在相同的文件内定义我们跳转到的动作,让它简单的渲染文本 `Redirect!`.

```elixir
def redirect_test(conn, _params) do
  text conn, "Redirect!"
end
```

当重新载入 [欢迎页面](http://localhost:4000),可以看到已经跳转到了 `/redirect_test` 并渲染了文本 `Redirect!`.
成功了!

如果我们关心,可以打开我们的开发工具,打开网络标签并再次访问路由根目录.可以看到这个页面有两个主请求 - 一个状态码为 `302` 的 `\` get 请求,另一个是 状态码是 `200` 的 `/redirect_test` get 请求.

注意到跳转函数接受 `conn` 参数和一个应用内相对地址的的字符串.也接受 `conn` 和完全路径字符串.

```elixir
def index(conn, _params) do
  redirect conn, external: "http://elixir-lang.org/"
end
```

我们还可以使用 [路由向导](http://www.phoenixframework.org/docs/routing) 中学到的路径助手(path heler).为了简化表达式,在 `web/router.ex` 文件 `alias` 它们会很有用.

```elixir
defmodule HelloPhoenix.PageController do
  use HelloPhoenix.Web, :controller

  def index(conn, _params) do
    redirect conn, to: redirect_test_path(conn, :redirect_test)
  end
end
```

注意这里我们不能使用 url 助手,因为 `redirect/2` 使用的原子 `:to` 需要路径.例如,下面这样就会失败.

```elixir
def index(conn, _params) do
  redirect conn, to: redirect_test_url(conn, :redirect_test)
end
```

如果我们想使用 url 助手传递给 `redirect/2` 一个完整地址,必须使用原子 `:external`.注意使用 `:external`的这个 url 地址,不必为真正的外部地址,可以像这样使用.

```elixir
def index(conn, _params) do
  redirect conn, external: redirect_test_url(conn, :redirect_test)
end
```
