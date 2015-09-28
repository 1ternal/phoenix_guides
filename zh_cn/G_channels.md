Channel 是 Phoenix 令人兴奋和强力的部分，可以让我们很简单的给应用增加软实时功能。Channels 基于一个简单的想法 - 发送和接收信息。发送者就主题进行信息广播。接收者订阅这些主题，这样就可以得到这些信息。任何时候，就同一个主题，发送者和接收者可以互换角色。

因为 Elixir 基于消息传递，你可能想知道为什么需要这样多余的机制来收发信息。用 Channels，发送者或接收者可以不必是 Elixir 进程。通过 Channel 它可以是任务我们想让它成为的东西 - 一个 JavaScript 客户端、一个 iOS 应用、另一个 Phoenix 应用，我们的手表。同样，消息通过 Channel 可以有多个接收者。Elixir 进程是一对一通讯。

“Channel” 这个词其实是一些组件组成的一层系统。先快速浏览一下，以便更好的从宏观上理解。

#### 移动的部分

- Socket 处理器

Phoenix 持有到服务器的单连接和通过单连接的多路 channel socket。Socket 处理器，例如 `web/channels/user_socket.ex`，是一个验证和区分socket 连接的模块，并允许你给所有的 Channel 设置默认socket 数据。

- Channel 路由

它们定义于 Socket 处理器，像 `web/channels/user_socket.ex` 文件，区别于其他的路由。它们匹配主题字符串，将后将匹配的请求派发到给定的 Channel 模块。星号 “*” 作为通配符，所以后面的路由例子，`sample_topic:pizza` 和 `sample_topic:oranges` 都会派发到 `SampleTopicChannel`。

```elixir
channel "sample_topic:*", HelloPhoenix.SampleTopicChannel
```

- Channels

Channels 处理来自客户端的事件，所以它们跟控制器（controllers）类似，但有两个关键的不同。Channel 事件有两个方向 - 出和入。Channel 连接在多个请求/响应之间进行持久化。Channel 是对 Phoenix 实时通讯组件的最高层级的抽象。

每个 Channel 将会实现一个或多个子句像这样四个回调函数 - `join/3`, `terminate/2`, `handle_in/3` 和 `handle_out/3`。

- PubSub

Phoenix PubSub 层包含 `Phoenix PubSub` 模块和不同转换器和它们的 GenServers 模块。这些模块包含 Channel 通讯的基本函数 - 订阅主题、取消订阅主题和对某个主题广播等。

如果需要我们也可以定义自己的 PubSub 转换器。查看 [Phoenix.PubSub 文档](http://hexdocs.pm/phoenix/Phoenix.PubSub.html) 获取更多信息。

值得注意的是，这些模块都是在 Phoenix 内部使用。Channel 底层使用它们完成工作。作为使用者，我们没有任何必要在应用中直接使用。

- 消息（Messages）

`Phoenix.Socket.Message` 模块定义一个使用下列的键作为合法消息的结构体。由 [Phoenix.Socket.Message 文档](http://hexdocs.pm/phoenix/Phoenix.Socket.Message.html)
  - `topic` - 字符串主题或 `topic:subtopic`, 例如 "messages", "messages:123"
  - `event` - 事件名称字符串, 如 “join”
  - `payload` - 承载的 JSON 消息字符串
  - `ref` - 用于回复接收到的事件的独一无二的字符串

主题就是辨识符字符串 - 为让消息准确送达该去的地方而准备的不同层的名称。如之前所见，主题可以使用通配符。这就允许使用 “topic:subtopic” 惯用法。通常，你会使用模型层的 RecordID，像 `"users:123"` 组成主题。

- 传输（Transports）

传输层是关键的模块。Channel 中所有的消息进出 都是由 `Phoenix.Channel.Transport` 模块派送。

- 传输转换器

默认的传输机制是通过 WebSockets,如果 WebSockets 不可用/工作将会降为 LongPolling。也可以用其他传输转换器，只要遵循转换器约定，可以自己写。查看 `Phoenix.Transports.WebSocket` 示例。

- 客户端库

Phoenix 目前自带 JavaScript 客户端。iOS 和 Android 客户端已随 Phoenix 1.0 发布。

## 一起尝试

让我们通过构建一个简单的聊天应用把这些点串起来。
先从 endpoint 挂载我们的 socket 和连接 channel 路由做起。

```elixir
# lib/hello_phoenix/endpoint.ex
defmodule HelloPhoenix.Endpoint do
  use Phoenix.Endpoint

  socket "/socket", HelloPhoenix.UserSocket
  ...
end

# web/channels/user_socket.ex
defmodule HelloPhoenix.UserSocket do
  use Phoenix.Socket

  channel "rooms:*", HelloPhoenix.RoomChannel

  transport :websocket, Phoenix.Transports.WebSocket

  def connect(_params, socket), do: {:ok, socket}

  def id(_socket), do: nil
end
```

现在，当一个客户端发送以 `"rooms:"` 开头的消息，将会被路由到我们的 `RoomChannel`。接下来，我们将定义一个 `RoomChannel` 模块管理我们的聊天室消息。

### Joining Channels

The first priority of your channels is to authorize clients to join a given topic. For authorization, we must implement `join/3` in `web/channels/room_channel.ex`.
Channel 首要任务是验证加入给定主题的客户端。为此，我们必须在 `web/channels/room_channel.ex` 实现 `join/3`。

```elixir
defmodule HelloPhoenix.RoomChannel do
  use Phoenix.Channel

  def join("rooms:lobby", auth_msg, socket) do
    {:ok, socket}
  end
  def join("rooms:" <> _private_room_id, _auth_msg, socket) do
    {:error, %{reason: "unauthorized"}}
  end

end
```

对我们的聊天应用，将允许任何人加入 `"rooms:lobby"` 主题，但其他聊天室被认为是私密和需要特殊的授权，假设从数据库。现在这个练习不考虑私密聊天室，我们完成后你可以自行探索。验证这个 socket 加入主题，我们返回 `{:ok, socket}` 或 `{:ok, reply, socket}`。拒绝接入，返回 `{:error, reply}`。更多信息关于使用令牌验证的信息，请查看[`Phoenix.Token` 文档](http://hexdocs.pm/phoenix/Phoenix.Token.html)。

我们的 Channel 就位了，接下来是 `web/static/js/app.js`，让客户端和服务器通信。

```javascript
import {Socket} from "deps/phoenix/web/static/js/phoenix"

let socket = new Socket("/socket")
socket.connect()
let chan = socket.channel("rooms:lobby", {})
chan.join().receive("ok", chan => {
  console.log("Welcome to Phoenix Chat!")
})
```

保存文件，浏览器应该自动刷新，多谢 Phoenix 实时刷新器。如果一切工作，我们应该在浏览器的 JavaScript 控制台看到 "Welcome to Phoenix Chat!" 我们的客户端和服务器现在通过一个持久连接进行交谈。现在启用聊天让它有用点。

在 `web/templates/page/index.html.eex`，增加一个容器放置聊天信息和用于发送信息的输入框。

```html
<div id="messages"></div>
<input id="chat-input" type="text"></input>
```

再 添加 jQuery 到我们的引用布局 `web/templates/layout/app`。

```html
  ...
    <%= @inner %>

  </div> <!-- /container -->
  <script src="//code.jquery.com/jquery-1.11.2.min.js"></script>
  <script src="<%= static_path(@conn, "/js/app.js") %>"></script>
</body>
```

添加一些事件监听器到 `app.js`：

```javascript
let chatInput         = $("#chat-input")
let messagesContainer = $("#messages")

let socket = new Socket("/socket")
socket.connect()
let chan = socket.channel("rooms:lobby", {})

chatInput.on("keypress", event => {
  if(event.keyCode === 13){
    chan.push("new_msg", {body: chatInput.val()})
    chatInput.val("")
  }
})

chan.join().receive("ok", chan => {
  console.log("Welcome to Phoenix Chat!")
})
```

所有我们做的是监视回车键按下，然后通过 channel `push` 一个消息。我们称这个事件为 "new_msg"。先把这个放一边，让我们处理聊天引用的其他部分，监听新信息然后把添加到消息容器内。

```javascript
let chatInput         = $("#chat-input")
let messagesContainer = $("#messages")

let socket = new Socket("/socket")
socket.connect()
let chan = socket.channel("rooms:lobby", {})

chatInput.on("keypress", event => {
  if(event.keyCode === 13){
    chan.push("new_msg", {body: chatInput.val()})
    chatInput.val("")
  }
})

chan.on("new_msg", payload => {
  messagesContainer.append(`<br/>[${Date()}] ${payload.body}`)
})

chan.join().receive("ok", chan => {
  console.log("Welcome to Phoenix Chat!")
})
```

我们使用 `chan.on` 监听 `"new_msg"` 事件, 然后将信息追加到 DOM。现在，让我们在服务器处理接收和输出事件来完成应用。

### 接收事件

我们通过 `handle_in/3` 处理收到的事件。我们可以模式匹配事件名称，像 `"new_msg"`，然后拉出通过 channel 从客户端传来的信息。对我们的聊天应用，我们需要简单的使用 `broadcast!/3` 通知所有其他 `rooms:lobby` 订阅者新信息。

```elixir
defmodule HelloPhoenix.RoomChannel do
  use Phoenix.Channel

  def join("rooms:lobby", auth_msg, socket) do
    {:ok, socket}
  end
  def join("rooms:" <> _private_room_id, _auth_msg, socket) do
    {:error, %{reason: "unauthorized"}}
  end

  def handle_in("new_msg", %{"body" => body}, socket) do
    broadcast! socket, "new_msg", %{body: body}
    {:noreply, socket}
  end

  def handle_out("new_msg", payload, socket) do
    push socket, "new_msg", payload
    {:noreply, socket}
  end
end
```

`broadcasts!/3` 将会通知所有加入这个 `socket` 主题的客户端 并调用 `handle_out/3` 回调。`handle_out/3` 并不是必须的回调，但它允许广播到达每个客户端之前，进行自定义和过滤。默认，`handle_out/3` 已经实现了且简单的把消息推给客户端，跟我们定义的一样。我们这里自己实现了，因为能够处理出站事件允许我们更强大的消息自定义和过滤。让我们看看怎么做。

#### 拦截出站事件

我们不会为这个应用实现它，但是想象下聊天程序允许用户忽略新用户加入聊天室的信息。我们可以明确告诉 Phoenix 想要拦截的出站事件行为来实现这一点，为这些事件定义 `handle_out/3` 回调（当然，这个假定我们有一个 `User` 模型和 `ignoring?/2` 函数，我们通过 `assigns` map 传递一个用户）。

```elixir
intercept ["user_joined"]

def handle_out("user_joined", msg, socket) do
  if User.ignoring?(socket.assigns[:user], msg.user_id) do
    {:noreply, socket}
  else
    push socket, "user_joined", msg
    {:noreply, socket}
  end
end
```

这就是我们基本的聊天应用的全部。打开多个浏览器标签，就应该看到信息就会广播到所有窗口！

#### Socket 赋值

与连接结构体 `%Plug.Conn{}` 类似，也可以给 channel socket 赋值。`Phoenix.Socket.assign/3` 是作为 `assign/3`方便起见导入到 channel 模块的：

```elixir
  socket = assign(socket, :user, msg["user"])
```

Sockets 存储的值在 `socket.assigns` map 中。

#### 示例应用

查看我们刚建好的应用，可以看这个项目(https://github.com/chrismccord/phoenix_chat_example)。

你也可以在 (http://phoenixchat.herokuapp.com/) 看到一个在线的 Demo。
