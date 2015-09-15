这篇向导的目的是给 Phoenix 应用增加两个新页面。一个是纯静态页面，另一个则会将路径的一部分作为输入传递模板进行显示。通过这个方式，我们将会熟悉 Phoenix 应用的基本组件：路由(Router), 控制器(controller)，视图(views) 和 模板(templates)。

当 Phoenix 为我们生成新应用的时候，建立了像下面的根目录结构

```text
├── _build
├── config
├── deps
├── lib
├── priv
├── test
├── web
```

这篇向导的大部分工作将会在 `web` 目录下展开，这个目录展开后如下。

```text
├── channels
├── controllers
│   └── page_controller.ex
├── models
├── router.ex
├── templates
│   ├── layout
│   │   └── app.html.eex
│   └── page
│       └── index.html.eex
└── views
|   ├── error_view.ex
|   ├── layout_view.ex
|   └── page_view.ex
└── web.ex
```

如上一篇最后所见，所有 controllers, templates 和 views 目录下的文件帮我们建立了 "Welcome to Phoenix" 页面。稍后我们将会再使用这里的一些代码。

我们应用所有的静态assets资源，根据类型都存放 `priv/static` 下的相应 css, images 或 js 目录下。编译后的资源将会放入 `web/static` 目录，`priv/static` 目录下的源码将会编译为相应的 `app.js` 或 `app.css`。现在我们不做任何修改，但是现在知晓，以便日后查找。

```text
priv
└── static
    └── images
        └── phoenix.png
```

```text
web
└── static
    ├── css
    |   └── app.scss
    ├── js
    │   └── app.js
    └── vendor
        └── phoenix.js
```

我们也应该了解下 `lib` 目录包含的文件。我们的应用终结点是 `lib/hello_phoenix/endpoint.ex`，应用（就是启动应用和监控树）文件是 `lib/hello_phoenix.ex`。

```text
lib
├── hello_phoenix
|   ├── endpoint.ex
│   └── repo.ex
└── hello_phoenix.ex
```

废话不多说，开始我们第一个 Phoenix 页面！

### 一条新路由

路由映射唯一的 HTTP 动词/路径(verb/path)到处理它们的控制器/动作(controller/action)。Phoenix 为我们生成了一个路由文件 `web/router.ex`。这也是我们这一节要进行的地方。

之前的 "Welcome to Phoenix" 页面的路由是下面这样：
```elixir
get "/", PageController, :index
```

尝试解读下这个路由告诉了我们什么。 访问 [http://localhost:4000/](http://localhost:4000/) 产生一个到根目录的 http GET 请求。所有这样的请求将会在`web/controlleres/page_controller.ex` 内定义的 `HelloPhoenix.PageController` 模块的 `index`函数处理。

当浏览器访问 [http://localhost:4000/hello](http://localhost:4000/hello)，我们将要编写的页面将会简单的答复 "Hello World, from Phoenix!"。

创建页面的第一件事情就是为它定义一条路由。使用文本编辑器打开 `web/router.ex`，目前它应该是这样：

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
目前，先忽略 pipelines 和 `scope` 的用法，专注与添加一条路由(好奇的话，我们将会在[路由教程](http://www.phoenixframework.org/docs/routing)详细讲解)。

添加一条新路由，映射 `/hello` 的 `GET` 到即将创建的 `HelloPhoenix.HelloController` 的 `index` 动作。

```elixir
get "/hello", HelloController, :index
```

`router.ex` 文件的 `scope "/"` 代码块应该像下面这样。

```elixir
scope "/", HelloPhoenix do
  pipe_through :browser # Use the default browser stack

  get "/", PageController, :index
  get "/hello", HelloController, :index
end
```

### 一个新控制器

控制器是 Elixir 模块，动作是定义在此模块内的 Elixir 的函数。动作的目的是收集所有数据并执行渲染所需的任务。路由规定了我们需要一个 `HelloPhoenix.HelloController` 模块内的 `index/2` 动作。

因此，我们创建一个新的 `web/controllers/hello_controller.ex` 文件，它的内容如下所示：

```elixir
defmodule HelloPhoenix.HelloController do
  use HelloPhoenix.Web, :controller

  def index(conn, _params) do
    render conn, "index.html"
  end
end
```

关于 `use HelloPhoenix.Web, :controller` 讨论留在[控制器教程](http://www.phoenixframework.org/docs/controllers)。现在，让我们关注在 `index/2` 动作。

所有的控制器动作接受两个参数。第一个是 `conn`， 它持有关于请求的一大堆数据。另一个是 `params`，它是请求的参数。这里，我们用不到 `params`，为了避免编译器告警在头部添加了 `_`。

这个动作的核心是 `render  conn, "index.html`。它告诉 Phoenix 查找一个叫 `index.html.eex` 的模板并进行渲染。Phoenix 将会和控制器同名的文件夹下查找，也就是 `web/templates/hello`。

注意：也可以使用原子作为模板名。 `render conn, :index`，基于接受的 headers 来选择模板，比如 `"index.html"` 或 `"index.json"` 等。

负责渲染的是视图(views)，接下来就要新建一个新的视图。

### 一个新视图

Phoenix 视图有一些重要的工作。它们渲染模板；它们从控制端作为原始数据的展示层，为模板中使用做准备。执行这样操作的函数应该属于视图。

举例来说，我们有一个用户这样的数据结构，有一个 `first_name` 字段和 `last_name` 字段，在模板中我们想显示全名。我们可以在模板中完成拼接这些字段，更好的处理方式是在视图中写一个函数来处理字符拼接，然后在模板中调用。这样模板就清晰明了的多。

为了渲染 `HelloController` 任一模板,我们需要一个 `HelloView`。名字在这里很重要，视图和控制的前半部分必须匹配。让我们先创建一个空文件，稍后再详细描述。创建 `web/views/hello_view.ex` 如下所示：

```elixir
defmodule HelloPhoenix.HelloView do
  use HelloPhoenix.Web, :view
end
```

### 一个新模板

Phoenix 模板只做一件事，决定哪些数据可被渲染。Phoenix 使用的标准模板引擎是 eex, 意为 [Embedded Elixir](http://elixir-lang.org/docs/stable/eex/)。我们所有的模板的后缀名是 `.eex`。

模板限定与特定的视图，视图又限定于控制器。实际上，我们要在 `web/templates` 创建一个名为控制器的目录。对于 hello 页面，需要在 `web/templates` 目录下创建一个 `hello` 目录，然后创建一个名为 `index.html.eex` 的文件。

现在就开始把。创建 `web/templates/hello/index.html.eex` 如下所示：

```html
<div class="jumbotron">
  <h2>Hello World, from Phoenix!</h2>
</div>
```

现在我们有了一个路由、控制器、视图和模板，当访问[http://localhost:4000/hello](http://localhost:4000/hello) 页面的时候，应该可以看到来自 Phoenix 的问候啦！(如果你停止的服务器，重启的命令是 `mix phoenix.server`)。

![来自 Phoenix 的问候](../images/hello-from-phoenix.png)

我们刚才做的有一些有趣的事情要注意。我们修改的时候不需要停止然后重启服务器。是的，Phoenix 自带了代码热更新！同样，尽管我们的 `index.html.eex` 文件只有一个单行 div 标签，我们得到的是一个完整的 HTML 文档。我们的 index 模板加入进了应用的布局 - `web/templates/layout/app.html.eex`。如果打开它，会看到像这样的一行： `<%= @inner %>`，它的作用就是在发送给浏览器之前插入进我们要渲染的的模板。

## 另一个页面

给我们的应用增加点复杂度。我们将加入一个新的页面，它可以识别 url 的部分内容，标记为 "messenger" 然后再通过控制器传递给模板，这样我们的信使就可以说 "hello" 了。

和之前类似，我们首先要做的就是添加一条新路由。

### 一条新路由

就这个练习，我们将使用刚创建的 `HelloController`，增加一个新的 `show` 动作。像下面这样，在最后一条路由后增加一行。

```elixir
scope "/", HelloPhoenix do
  pipe_through :browser # Use the default browser stack.

  get "/", PageController, :index
  get "/hello", HelloController, :index
  get "/hello/:messenger", HelloController, :show
end
```

注意到我们加入了 `:messenger` 原子到路径。Phoenix 在此 url 位置会接受任意值, 然后传递一个包含 `messenger` 为键的[字典](http://elixir-lang.org/docs/stable/elixir/Dict.html)给控制器。

例如，如果我们浏览  [http://localhost:4000/hello/Frank](http://localhost:4000/hello/Frank) 页面，":messenger" 的值将会是 "Frank"。

### 一个新动作

这条新路由的请求将由 `HelloPhoenix.HelloController` 的 `show` 函数处理。已经有了控制器 `web/controllers/hello_controller.ex`，所以我们需要做的是编辑这个文件，添加一个 `show` 动作。这次，我们需要将参数内的一个数据传递给动作，这样才能传递（信使）给模板。为此，我们添加这样的 show 函数：

```elixir
def show(conn, %{"messenger" => messenger}) do
  render conn, "show.html", messenger: messenger
end
```

这里有一些要注意的地方。我们模式匹配传递给 show 函数的 params 参数，这样 `messenger` 变量将会绑定到我们给定的 url 中 `:messenger` 位置的值。例如，如果我们的 url 是 [http://localhost:4000/hello/Frank](http://localhost:4000/hello/Frank) 页面，":messenger" 的值将绑定为 "Frank"。

`show` 动作内，我们也传递了第三个参数给 render 函数， 一个 `:messenger` 为键，mesenger 变量作为值的键值对。

注意：如果需要获取额外所有的参数键值，我们可以这样定义 `show/2` 函数：

```elixir
def show(conn, %{"messenger" => messenger} = params) do
  ...
end
```

最好记住 params [字典](http://elixir-lang.org/docs/stable/elixir/Dict.html) 的键是字符串类型，等号不是赋值，而是一个[模式匹配](http://elixir-lang.org/getting-started/pattern-matching.html) 。

### 一个新模板

最后一块拼图，我们需要一个新的模板。因为是为 `HelloController` 的 `show` 动作，所以它会调用 `web/templates/hello` 目录下的 `show.html.eex` 文件。除了我们需要显示信使的名字外，它跟我们的 `index.html.eex` 模板非常像。

为此，我们使用了特殊的 eex 标签来执行 Elixir 表达式 - `<%= %>`。特别的，起始标签包含有一个等号 `<%=`。意思是当中的任何 Elixir 代码将会被执行，结果将会替代这个标签。如果没有等号，代码仍然将会执行，只是页面不显示。

下面是模板的模样：

```html
<div class="jumbotron">
  <h2>Hello World, from <%= @messenger %>!</h2>
</div>
```

我们的信使以 `@messenger` 出现。这个情形下，它不是模块属性。它是特殊的元编程语法，意思是 `Dict.get(assigns, :messenger)`。 这样看起来和用起来都比较简单。

完成了。如果你用浏览器打开 http://localhost:4000/hello/Frank](http://localhost:4000/hello/Frank)，会看到如下的页面：

![Frank Greets Us from Phoenix](../images/hello-world-from-frank.png)

玩会儿吧。不管在 /hello/ 后添加什么，它都会作为信使出现在页面上。
