一旦应用能正常运行，我们就做好了部署的准备。别担心你的应用没有完成。按照[快速入门](http://www.phoenixframework.org/docs/up-and-running)创建的应用就行。

部署准备有以下三个主要步骤：

  * 处理应用密钥
  * 编译 assets 资源文件
  * 生产环境启动服务器

这些步骤如何处理取决于你的部署设施。特别的，我们有针对 [heroku](http://www.phoenixframework.org/docs/heroku) 的教程和使用 Erlang 版本方式的[高级部署向导](http://www.phoenixframework.org/docs/advanced-deployment)。任何情形下，不论你的部署设施如何，或者想在本地生产环境运行 ，本章部署步骤的总览都会对你有所帮助。

让我们一步步的探索这些步骤。

> 注意：本教程假定你的 Elixir 版本不低于 v1.0.4，这个版本提供了编译改进和生产环境优化。

## 处理应用密钥

任何 Phoenix 应用都有要保证安全的数据，例如，生产环境数据库的用户名和密码，Phoenix 用于签名和加密重要数据的密钥。这些数据保存在 `config/prod.secret.exs`，且默认是不会迁入进版本控制系统。

因此，第一步是在生产机器上获取这些数据。这些是跟新应用一起的模板：

```elixir
use Mix.Config

# In this file, we keep production configuration that
# you likely want to automate and keep it away from
# your version control system.

# You can generate a new secret by running:
#
#     mix phoenix.gen.secret
config :foo, Foo.Endpoint,
  secret_key_base: "A LONG SECRET"

# Configure your database
config :foo, Foo.Repo,
  adapter: Ecto.Adapters.Postgres,
  username: "postgres",
  password: "postgres",
  database: "foo_prod",
  size: 20 # The amount of database connections in the pool
```

生产环境下，有很多不同的方法获取这些数据。其中一个是使用环境变量替代这些数据，并在生产机器上设置这些变量。这是我们遵循 [Heroku 教程](http://www.phoenixframework.org/docs/heroku)的步骤。

另一个方式是使用上面的配置文件，除了你的代码外再把配置文件放进生产机器，如 "/var/config.prod.exs"。这样，你必须从 `config/prod.exs` 中导入它。使用 `import_config` 搜索并替换为合适的路径：

```elixir
import_config "/var/config.prod.exs"
```

配置完密钥信息，是配置 assets 资源的时候了！

## 编译 assets 资源文件

假如你的 Phoenix 应用有图片、javascript、stylesheets 和其他一些资源的时候，这一步是必须的。默认，Phoenix 使用 brunch，这也是我们要探索的。

编译静态文件分两步：

```console
$ brunch build --production
$ MIX_ENV=prod mix phoenix.digest
Check your digested files at "priv/static".
```

就这么多！第一条命令编译资源，第二条生成摘要和清单文件，这样在生产环境就可以快速的取到资源。

记住，如果忘记运行上面的步骤，Phoenix 将会显示一条错误信息：

```console
$ PORT=4001 MIX_ENV=prod mix phoenix.server
10:50:18.732 [info] Running MyApp.Endpoint with Cowboy on http://example.com
10:50:18.735 [error] Could not find static manifest at "my_app/_build/prod/lib/foo/priv/static/manifest.json". Run "mix phoenix.digest" after building your static files or remove the configuration from "config/prod.exs".
```

错误信息很清晰，它说 Phoenix 无法找到静态清单。运行上面的文件即可修正。或者，如果压根就不在乎 assets 资源，可以在 `config/prod.exs` 移除 `cache_static_manifest` 配置。

## 生产环境启动服务器

为了在生产环境运行 Phoenix，调用 `mix phoenix.server` 时，`PORT` 和 `MIX_ENV` 环境变量需要设置：

```console
$ PORT=4001 MIX_ENV=prod mix phoenix.server
10:59:19.136 [info] Running MyApp.Endpoint with Cowboy on http://example.com
```

如果需要错误信息，请仔细阅读，如果还不清楚如何解决请提交 bug 报告.

你可以再交互 shell 环境下运行应用：

```console
$ PORT=4001 MIX_ENV=prod iex -S mix phoenix.server
10:59:19.136 [info] Running MyApp.Endpoint with Cowboy on http://example.com
```

或者从 iex 控制台分离运行。这样有效的守护进程，它就可以在后台独立运行。

```elixir
MIX_ENV=prod PORT=4001 elixir --detached -S mix do compile, phoenix.server
```

分离模式运行下，就算我们关闭与服务器的 shell 连接，应用也会保持在后台运行。

## 总览

上面几节对部署 Phoenix 应用的主要步骤做了一个总览。实际中，你还需要增加其他步骤。例如，若使用数据库，启动服务前，为保证数据库最近同步，你还需要运行 `mix ecto.migrate`。

总的来说，你可以使用下面的脚本作为起点：

```elixir
# Initial setup
$ mix deps.get --only prod
$ MIX_ENV=prod mix compile

# Compile assets
$ brunch build --production
$ MIX_ENV=prod mix phoenix.digest

# Custom tasks (like DB migrations)
$ MIX_ENV=prod mix ecto.migrate

# Finally run the server
$ PORT=4001 MIX_ENV=prod mix phoenix.server
```
