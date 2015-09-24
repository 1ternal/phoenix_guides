Phoenix 视图有两项主要工作.首先也是最重要的，渲染模板(包含布局).涉及渲染的关键函数 `render/3` 是定义在 Phoenix 的 `Phoenix.View` 模块.视图还提供接收原始数据并让其更容易的被模板使用的函数.如果你熟悉装饰器或包装模式，它们很相近。

从控制器到视图再到要渲染的模板，Phoenix会假定有很强的命名惯例。 `PageController` 需要有一个 `PageView` 渲染 `web/templates/page` 目录。

如果需要，我们可以修改 Phoenix 模板的根目录。Phoenix 提供了一个定义在 `web/web。ex` 内 `HelloPhoenix。Web` 模块的 `view/0` 函数。`view/0` 的第一行允许我们通过修改 `:root` 键的值来更改根目录。

A newly generated Phoenix application has three view modules - `ErrorView`， `LayoutView`， and `PageView` -  which are all in the， `web/views` directory。
新生成的 Phoenix 应用有三个视图模块 - `ErrorView`、`LayoutView` 和 `PageView`，它们三个都在 `web/views` 目录内。

我们快速的看看 `LayoutView`。

```elixir
defmodule HelloPhoenix.LayoutView do
  use HelloPhoenix.Web， :view
end
```

足够简单。只有一行，`use HelloPhoenix.Web, :view`。这行调用我们刚刚看到的 `view/0` 函数。除了允许我们修改模板根目录以外， `view/0` 在 `Phoenix.View` 使用了 `__using__/1` 宏，处理任何我们需要导入或可能需要的视图模块的别名。

本文开头提到，视图还是放置模板中需要使用的函数的地方。让我们尝试一下。

打开应用的布局模板 `templates/layout/app.html.eex`，修改这一行

```html
<title>Hello Phoenix!</title>
```

像这样调用 `title/0` 函数。

```elixir
<title><%= title %></title>
```

现在在 `LayoutView` 中添加一个 `title/0` 函数。

```elixir
defmodule HelloPhoenix.LayoutView do
  use HelloPhoenix.Web, :view

  def title do
    "Awesome New Title!"
  end
end
```

当重新载入欢迎页面的时，应该可以看到我们的新标题。

`<%=` 和 `%>` 来自于 Elixir [Eex](http://elixir-lang.org/docs/stable/eex/) 项目。在模板内包裹可执行的 Elixir 代码 。`=` 告诉 Eex 输出结果。如果没有 `=`，Eex 仍然会执行代码，但不会输出结果。在例子中，我们调用了 `LayoutView` 的 `title/0` 函数并将结果输出到 title 标签。

注意 `title/0` 不需要和 `HelloPhoenix.LayoutView` 一起这样的完整名称，因为进行渲染的就是 `LayoutView`。

使用 `use HelloPhoenix.Web, :view` 还会其他好处。因为 `view/0` 函数导入了 `HelloPhoenix.Router.Helpers`，模板中也不需要使用完整路径调用路径助手(path helpers)。修改欢迎页面的模板，看看它是如何工作的。

打开 `templates/page/index.html.eex` 找到如下内容

```html
<div class="jumbotron">
  <h2>Welcome to Phoenix!</h2>
  <p class="lead">Most frameworks make you choose between speed and a productive environment. <a href="http://phoenixframework.org">Phoenix</a> and <a href="http://elixir-lang.org">Elixir</a> give you both.</p>
</div>
```

添加链接同一页面的链接（主要看如何在模板中处理路径助手，而不是添加任何功能）。

```html
<div class="jumbotron">
  <h2>Welcome to Phoenix!</h2>
  <p class="lead">Most frameworks make you choose between speed and a productive environment. <a href="http://phoenixframework.org">Phoenix</a> and <a href="http://elixir-lang.org">Elixir</a> give you both.</p>
  <p><a href="<%= page_path @conn, :index %>">Link back to ourselves</a></p>
</div>
```

现在可以重新载入页面并查看我们得到了什么样的源码。

```html
<a href="/">Link back to ourselves</a>
```

太好了，`page_path/2` 如愿的指向了 `/` 目录，而且我们不需要 `HelloPhoenix.View` 完整调用。

### 更多关于视图

你可能想知道视图如何这么紧密的和模板在一起工作。

通过 `use Phoenix.Template` 内的 `__using__/1` 宏，`Phoenix.View` 模块获取了模板行为。`Phoenix.Template` 提供了很多方便的与模板一起工作的方法 - 查找，提取它们的名称和路径等。

让我们用 Phoenix 生成给我们的 `web/views/page_view.ex` 做点小实验。添加以如下所示的 `message/0` 函数。

```elixir
defmodule HelloPhoenix.PageView do
  use HelloPhoenix.Web, :view

  def message do
    "Hello from the view!"
  end
end
```

现在让我们创建一个实验模板 `web/templates/page/test.html.eex`。

```html
This is the message: <%= message %>
```

没有任何符合的控制器动作，但我们将在一个 `iex` 会话中实践。在项目根目录，可以运行 `iex -S mix` 然后指定我们要渲染的模板。

```console
iex(1)> Phoenix.View.render(HelloPhoenix.PageView, "test.html", %{})
  {:safe, [["" | "This is the message: "] | "Hello from the view!"]}
```

可以看到，我们使用了这几个参数调用 `render/3` 函数，test 模板相应的视图、test 模板的名称和一个我们可以传递任何数据的空 map。

返回值是一个元组，以 `:safe` 原子开始，然后是一个已完成插值的模板的字符串。

这里“Safe”的意思是 Phoenix 已经为我们渲染的模板进行了过滤。Phoenix定义了自己的 `Phoenix.HTML.safe` 协议实现，当需要渲染为字符串时，对原子、二进制字串、列表、整型、浮点型和元组进行过滤。

赋值给 `render/3` 的第三个参数一些键值对会发生什么呢？为此，我们需要稍稍修改模板。

```html
I came from assigns: <%= @message %>
This is the message: <%= message %>
```

注意最上面的 `@`。现在修改函数调用，重新编译 `PageView` 模块后将看到区别。

```console
iex(2)> r HelloPhoenix.PageView
web/views/page_view.ex:1: warning: redefining module HelloPhoenix.PageView
{:reloaded, HelloPhoenix.PageView, [HelloPhoenix.PageView]}

iex(3)> Phoenix.View.render(HelloPhoenix.PageView, "test.html", message: "Assigns has an @.")
{:safe,
  [[[["" | "I came from assigns: "] | "Assigns has an @."] |
  "\nThis is the message: "] | "Hello from the view!"]}
 ```

测试下 HTML 过滤

```console
iex(4)> Phoenix.View.render(HelloPhoenix.PageView, "test.html", message: "<script>badThings();</script>")
{:safe,
  [[[["" | "I came from assigns: "] |
     "&lt;script&gt;badThings();&lt;/script&gt;"] |
    "\nThis is the message: "] | "Hello from the view!"]}
```

如果我们只需要渲染字符串，而不用所有元组，可以使用 `render_to_iodata/3`。

 ```console
 iex(5)> Phoenix.View.render_to_iodata(HelloPhoenix.PageView, "test.html", message: "Assigns has an @.")
 [[[["" | "I came from assigns: "] | "Assigns has an @."] |
   "\nThis is the message: "] | "Hello from the view!"]
  ```

###　布局概述

布局就是模板。和其他模板一样，它们有一个视图。在新生成的应用中是 `web/views/layout_view.ex`　文件。你或许想知道来自视图渲染结果的字符串为何会在布局中？这是一个非常棒的问题！

当模板渲染后，布局视图将已渲染的模板内容赋值给　`@inner`。对 HTML　模板来说，`@inner`　总是标记为安全。

如果查看　`web/templates/layout/app.html.eex`　文件 `<body>`　的中间，会看到下面内容：

```html
<%= @inner %>
```

这就是渲染后的字符串填充的地方。

### 错误视图(ErrorView)

Phoenix 最近为所有生成的应用添加了一个新的视图，`web/views/error_view.ex`　中的　`ErrorView`。`ErrorView`　的目的是以通用、集中的方式处理最常见的两种错误　- `404　not found`　和 `500 internal error`。　让我们看看它们：

```elixir
defmodule HelloPhoenix.ErrorView do
  use HelloPhoenix.Web, :view

  def render("404.html", _assigns) do
    "Page not found"
  end

  def render("500.html", _assigns) do
    "Server internal error"
  end

  # In case no render clause matches or no
  # template is found, let's render it as 500
  def template_not_found(_template, assigns) do
    render "500.html", assigns
  end
end
```

详细查看之前，先在浏览器中看看　`404 not found`　页面长啥样。开发模式中，Phoenix 默认会　debug 错误，显示一个信息量巨大的排错页面。但我们现在是想看看生产模式下页面。为此我们需要在 `config/dev.ex`　中设置 `debug_errors: false`。

```elixir
use Mix.Config

config :hello_phoenix, HelloPhoenix.Endpoint,
  http: [port: 4000],
  debug_errors: false,
  code_reloader: true,
  . . .
```

现在访问　[http://localhost:4000/such/a/wrong/path](http://localhost:4000/such/a/wrong/path)　看看有什么结果。

好吧，没有什么好兴奋的。我们看到的只是字符“Page not found”，没有任何标记或样式。

看看我们能不能用现有的支持，让它错误页面有趣点。

第一个问题是，错误字符串从哪儿来？答案就在 `ErrorView`。

```elixir
def render("404.html", _assigns) do
  "Page not found"
end
```

太好了，我们有个　`render/2`　函数，它接受一个模板和忽略的　`assigns` map。`render/2`　从哪里调用呢？

答案就是定义在 `Phoenix.Endpoint.ErrorHandler` 的　`render/5`　函数。这个模块的目的就是捕捉错误并使用视图进行渲染，本例中，就是　`HelloPhoenix.ErrorView`　视图。

知道来龙去脉了，让我们做一个更好的错误页面。

Phoenix 会生成一个　`ErrorView`，但它没有给我们　`web/templates/error`　目录。我们现在就创建一个。在新目录内，新建一个　`not_found.eex`　文件并添加些样式　－　应用布局和给用户一些信息的新　`div`　的混合体。

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="utf-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <meta name="description" content="">
    <meta name="author" content="">

    <title>Welcome to Phoenix!</title>
    <link rel="stylesheet" href="/css/app.css">
  </head>

  <body>
    <div class="container">
      <div class="header">
        <ul class="nav nav-pills pull-right">
          <li><a href="http://www.phoenixframework.org/docs">Get Started</a></li>
        </ul>
        <span class="logo"></span>
      </div>

      <div class="jumbotron">
        <p>Sorry, the page you are looking for does not exist.</p>
      </div>

      <div class="footer">
        <p><a href="http://phoenixframework.org">phoenixframework.org</a></p>
      </div>

    </div> <!-- /container -->
    <script src="/js/app.js"></script>
  </body>
</html>
```

现在我们使用之前在　`iex`　会话中实验所见到的　`render/2`　函数。

修改后，我们的　`render/2`　函数应该像下面这样。

```elixir
def render("404.html", _assigns) do
  render("not_found.html", %{})
end
```

当返回　[http://localhost:4000/such/a/wrong/path](http://localhost:4000/such/a/wrong/path)　页面，我应该看到一个好多了的错误页面。

值得注意的是，尽管想让错误页面的风格和感觉与站点的一致，我们没有通过应用布局来渲染　`not_found.html.eex`　模板。重要原因是全局处理错误时非常容易遇到临界状况问题。

如果想将我们的应用布局和　`not_found.htmleex`　模板的重复代码减到最少，可以为页首和页尾使用共享模板。请查看　[模板向导](http://www.phoenixframework.org/docs/templates#section-shared-templates-across-views)获取更多信息。

当然，在 `ErrorView` 中以同样的步骤处理　`def render("500.html", _assigns) do`。

我们还可以使用 `assigns` map　传递给 `ErrorView`　中的任何`render/2`，来达到在模板中显示更多信息的目的，而不是抛弃它。
