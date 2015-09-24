模板跟它们的字面意思一样　－　接收传递来的数据形成完整的　HTTP　响应的文件。对 web 应用，响应通常是完整的 HTML　文档。对于　API，通常是　JSON　或 XML。模板文件的大部分代码常常是标记代码，但这儿也有部分　Phoenix 需要编译和执行的　Elixir 代码。实际上模板之所以非常快的原因是　Phoenix 的模板都进行了预编译。

Phoenix　的默认模板相同是　EEx，类似于　Ruby 中的 ERB。它实际是 Elixir　的组成部分，Phoenix 生成的新应用的时候也是使用 EEx　模板创建像路由和应用主视图这样的文件。 　

如 [View Guide](http://www.phoenixframework.org/docs/views)　所学，默认，模板在　`web/templates` 目录，以视图名称的方式进行组织。每个视图模块渲染相应目录内的模板。在应用的主视图内，我们可以通过指定一个新的 `root: "web/templates"`　值修改模板的根目录。

### 示例

我们已经见过一些不同的使用模板的方式，特别是在[添加页面向导](http://www.phoenixframework.org/docs/adding-pages) 和[视图向导](http://www.phoenixframework.org/docs/views)。这里或许一些重复，但我们肯定会增加一些新知识。

##### web.ex

Phoenix　生成的　`web/web.ex`　将常用的导入和别名进行了分组。所有定义于 `view`　代码内的申明会应用到所有模板。

为我们的应用添加点东西，这样我们进行小实验。

首先，让我们在 `web/router.ex` 定义一条新路由。

```elixir
scope "/", HelloPhoenix do
	pipe_through :browser # Use the default browser stack

	get "/", PageController, :index
	get "/test", PageController, :test
end
```

现在，让我们定义在路由中指定的控制器动作。在　`web/controller/page_controller.ex`　文件内新增一个　`test/2`　动作。

```elixir
def test(conn, _params) do
	render conn, "test.html"
end
```

再创建一个函数，告诉我们哪个控制器和动作处理我们的请求。

为此，我们需要从主视图的 `Phoenix.Controller` 导入　`action_name/１` 和 `controller_module/1` 函数。

```elixir
defmodule HelloPhoenix.Web do
	...
	def view do
		quote do
			use Phoenix.View, root: "web/templates"

			# Import convenience functions from controllers
			import Phoenix.Controller, only: [get_csrf_token: 0, get_flash: 2, view_module: 1,
																				action_name: 1, controller_module: 1] # Add these as imported functions
	...
```

接下来，在 `web/views/page_view.ex` 文件的最下面定义一个`handle_info/1`　函数使用 `controller_module/1` 和　`action_name`　函数。我们也会再定义一个一会儿用的　`connection_keys/1` 函数。

```elixir
. . .
defmodule HelloPhoenix.PageView do
	...
	def handler_info(conn) do
		"Request Handled By: #{controller_module conn}.#{action_name conn}"
	end

	def connection_keys(conn) do
		conn
		|> Map.from_struct()
		|> Map.keys()
	end
end
```

路由有了。刚刚也创建了一个新的控制器动作。应用的主视图也进行了修改。现在只需要一个新的模板来显示从 `handler_info/1`　得到的字符串。创建一个新的　`web/templates/page/test.html.eex`。

```elixir
<div class="jumbotron">
	<p><%= handler_info @conn %></p>
</div>
```

注意，通过　`assigns` map，`@conn`　可以直接在模板中使用。

访问 [localhost:4000/test](http://localhost:4000/test)，就会看到　`Elixir.HelloPhoenix.PageCotnroller.test` 带给我们的页面。

我们可以在　`web/views`　中任意一个单独的视图中定义函数。定义在单独的视图中的函数只在这个视图的模板中可用。比如，像上面的　`handler_info`　的函数，只能在　`web/templates/page`　下的模板使用。

##### 显示列表

目前为止，我们只是在模板中显示单个值　－　这里是字符串，其他向导里的整型。那我们如何显示列表里的所有元素呢？

答案是使用　Elixir 的列表推导式。

我们有一个模板可见的函数，它返回　`conn`　结构体里的一个列表，我们需要做的就是修改　`web/templates/page/test.html.eex`　模板显示它们。

我们可以增加一个页首和一个像这样列表推导式。

```elixir
<div class="jumbotron">
	<p><%= handler_info @conn %></p>

	<h3>Keys for the conn Struct</h3>

	<%= for key <- connection_keys @conn do %>
		<p><%= key %></p>
	<% end %>
</div>
```

我们使用 `connection_keys`　函数返回的列表作为要迭代的源列表。注意在列表推导的第一行和显示行都需要在 `<%=`　使用 `=`。没有这个，就什么都不会显示。　

当再次访问　[localhost:4000/test](http://localhost:4000/test)，　就可以看到所有的值显示了。

#####　模板中渲染模板

上面列表推导式例子中，显示值的部分实际非常简单。

```elixir
<p><%= key %></p>
```

就这么显示可能没什么问题。然而更常见的是，显示内容的代码会比较复杂，把这样的代码放在列表推导式的中间让我们的模板很难阅读。

简单的解决方法是使用另一个模板！模板只是函数调用，和正常代码一样，更干净的设计是使用小的，有目的性的函数来组合成较大的模板。这就是我们之前所见，一个简单的延续性。布局是被模板插入的已渲染的常规模板(Layouts are templates into which regular templates are rendered)。常规模板内可以包含其他模板。

让我们把这段显示代码变成自己的模板。创建一个如下所示的新模板 `web/templates/page/key.html.eex`

```elixir
<p><%= @key %></p>
```

我们这里需要修改　`key`　为　`@key`，因为它是一个新模板，而不是列表推导式的一部分。给模板传递数据的方式是通过　`assigns` map，从 `assigns`　map　取出值的方式使用 `@`　加键。

现在模板有了，我们可以在　`test.html.eex` 模板中的列表推导式中直接的渲染它。

```elixir
<%= for key <- connection_keys @conn do %>
	<%= render "key.html", key: key %>
<% end %>
```

再看看　[localhost:4000/test](http://localhost:4000/test)，页面应该和之前的一样。

##### 跨视图共享模板

常常，我们发现有一小部分数据在应用的不同部分需要以相同的方式渲染。最佳实践是将这些模板放到一个共享的目录，表示它们需要在应用的任何地方使用。

移动我们的模板到一个共享的视图。

`key.html.eex`　现在被　`HelloPhoenix.PageView`　模块渲染，但渲染调用假定当前视图模型就是我们想要渲染的。我们可以明确指定它，像这样重写：

```elixir
<%= for key <- connection_keys @conn do %>
	<%= render HelloPhoenix.PageView, "key.html", key: key %>
<% end %>
```

因为我们想把它放在新的 `web/templates/shared` 目录，我们需要一个新的单独的视图来渲染这个目录内的视图　`web/views/shared_view.ex`。

```elixir
defmodule HelloPhoenix.SharedView do
	use HelloPhoenix.Web, :view
end
```

现在我们把 `key.html.eex`　从 `web/templates/page`　目录移动到　`web/templates/shared` 目录。完成后，我们可以修改 render 调用使用新的　`HelloPhoenix.SharedView`。

```elixir
<%= for key <- connection_keys @conn do %>
	<%= render HelloPhoenix.SharedView, "key.html", key: key %>
<% end %>
```

再次返回　[localhost:4000/test](http://localhost:4000/test),　页面应该和之前的一样。
