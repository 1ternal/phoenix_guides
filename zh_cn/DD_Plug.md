Plug 处于 Phoenix 的 HTTP 层的心脏部位，同时 Phoenix 把 Plug 也置于前端和中心。在连接的生命周期内，我们处处都在和 plug 打交道，Phoenix 核心组件如 Endpoint、路由和控制器内部均为 Plug 中间件。让我们一探究竟什么使得 Plug 如此特殊。

[Plug](https://github.com/elixir-lang/plug) 是 web 应用间组合模块的一个规范。也是对不同的 web 服务器的连接转换(connection adapters)的抽象。Plug 的基础思想是统一我们操作的"连接"这个概念。区别于像 Rack 这样的 HTTP 中间件层，在中间件栈中它们的请求和响应是分开的。

## Plug 规范

简单来说，Plug 规范有两个方面， *函数 plug* 和 *模块 plug*。

### 函数 Plugs

为了作为一个 Plug 中间件， 函数需要接受一个连接结构体(%Plug.Conn{}) 和 选项。同时也需要返回这个连接结构体。任何符合这俩条件的都是 函数 plug。下面是个例子：

```elixir
def put_headers(conn, key_values) do
  Enum.reduce key_values, conn, fn {k, v}, conn ->
    Plug.Conn.put_resp_header(conn, k, v)
  end
end
```

很简单，对么？

这就是我们如何使用它组合成一系列的变换，并将它应用于Phoenix连接：

```elixir
defmodule HelloPhoenix.MessageController do
  use HelloPhoenix.Web, :controller

  plug :put_headers, %{content_encoding: "gzip", cache_control: "max-age=3600"}
  plug :put_layout, "bare.html"

  ...
end
```

遵循 plug 的要求， `put_headeres/2`, `put_layout/2` 甚至 `action/2` 将一个应用请求进行一系列的指定变换。不仅如此。为了真正见识下 Plug 设计如何高效，假设这样一个情景，我们需要检查一些列的条件，对不符合条件的进行跳转或停止。如果没有 plug，我们可以需要这么做：

```elixir
defmodule HelloPhoenix.MessageController do
  use HelloPhoenix.Web, :controller

  def show(conn, params) do
    case authenticate(conn) do
      {:ok, user} ->
        case find_message(params["id"]) do
          nil ->
            conn |> put_flash(:info, "That message wasn't found") |> redirect(to: "/")
          message ->
            case authorize_message(conn, params["id"])
              :ok ->
                render conn, :show, page: find_message(params["id"])
              :error ->
                conn |> put_flash(:info, "You can't access that page") |> redirect(to: "/")
            end
        end
      :error ->
        conn |> put_flash(:info, "You must be logged in") |> redirect(to: "/")
    end
  end
end
```

看到仅仅只是验证和授权就需要这么复杂的嵌套和重复了吗？让我们使用一些 plug 改进：

```elixir
defmodule HelloPhoenix.MessageController do
  use HelloPhoenix.Web, :controller

  plug :authenticate
  plug :find_message
  plug :authorize_message

  def show(conn, params) do
    render conn, :show, page: find_message(params["id"])
  end

  defp authenticate(conn, _) do
    case Authenticator.find_user(conn) do
      {:ok, user} ->
        assign(conn, :user, user)
      :error ->
        conn |> put_flash(:info, "You must be logged in") |> redirect(to: "/") |> halt
    end
  end

  defp find_message(conn, _) do
    case find_message(params["id"]) do
      nil ->
        conn |> put_flash(:info, "That message wasn't found") |> redirect(to: "/") |> halt
      message ->
        assign(conn, :message, message)
    end
  end

  defp authorize_message(conn, _) do
    if Authorizer.can_access?(conn.assigns[:user], conn.assigns[:message]) do
      conn
    else
      conn |> put_flash(:info, "You can't access that page") |> redirect(to: "/") |> halt
    end
  end
end
```

通过一系列的 plug 变换将嵌套的代码块都代替拉平了，用更组件化、清晰和可复用的方式完成了同样的功能。

现在让我们看看另一种 plug，模块 plug。

### 模块 Plugs

模块 plug 是另一种 Plug，允许我们在模块内定义连接变换。模块只需要实现两个函数：

- `init/1` 初始化传递给 `call/2` 的任何参数或选项
- `call/2` 执行连接变换。`call/2` 是一个 我们之前看到的 plug 函数

看看实际代码，让我们写一个模块，为连接分配一个以`:locale`为键的值用于后续其他 plug, 控制器动作 和视图使用。

```elixir
defmodule HelloPhoenix.Plugs.Locale do
  import Plug.Conn

  @locales ["en", "fr", "de"]

  def init(default), do: default

  def call(%Plug.Conn{params: %{"locale" => loc}} = conn, _default) when loc in @locales do
    assign(conn, :locale, loc)
  end
  def call(conn, default), do: assign(conn, :locale, default)
end

defmodule HelloPhoenix.Router do
  use HelloPhoenix.Web, :router

  pipeline :browser do
    plug :accepts, ["html"]
    plug :fetch_session
    plug :fetch_flash
    plug :protect_from_forgery
    plug :put_secure_browser_headers
    plug HelloPhoenix.Plugs.Locale, "en"
  end
  ...
```

可以使用 `plug HelloPhoenix.Plugs.Locale, "en` 将这个模块添加到浏览器通道。在 `init/1` 回调，如果参数未指定，将传递默认的语言。同时使用了模式匹配来定义多重 `call/2` 函数来验证参数中的语言，如果不匹配则使用 "en"。

这就是 Plug 的全部内容。Phoenix 从头至尾都拥抱了 Plug 对于组合变换的设计。这只是第一次尝试。如果问自己，“可以把这个放入 plug 嘛？”，答案通常是“是的！”
