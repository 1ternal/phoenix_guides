当前第一篇向导的目的是尽快的让 Phoenix 程序跑起来。

开始之前，花一分钟阅读[安装向导](http://www.phoenixframework.org/docs/installation)。为了能够顺畅的运行程序,我们必须先安装必要的依赖。

现在，我们应该安装好了 Elixir, Erlang, Hex 和 Phoenix存档。同时，为了建立默认的程序 PostgreSQL 和 node.js 也应该安装上。

Ok,我们可以开始了！

任一目录下运行 `mix phoenix.new` 就可以生成我们的 Phoenix 程序。Phoenix 允许使用绝对路径或相对路径作为新项目的路径。假设我们的应用名叫 `hello_phoenix`，下面两种都没问题。

```console
$ mix phoenix.new /Users/me/work/elixir-stuff/hello_phoenix
```

```console
$ mix phoenix.new hello_phoenix
```

> 开始前关于[Brunch.io]()：Phoenix 默认使用 Brunch.io 管理 assets 资源。Brunch.io 的依赖不是通过 mix,而是通过 node 包管理器安装。Phoenix 在 `mix phoenix.new` 后会提示我们执行这个任务。如果此时选择 “否”，此后也没有执行 `npm install` 安装依赖的话，当我们试图启动程序的时候会抛出错误，assets 资源也无法正确的载入。如果我们不想使用 Brunch.io，创建应用的时候应该传递 `--no-brunch` 参数给 `mix phoenix.new` 命令。

现在我们准备好了，运行 `phoenix.new` 加一个相对路径。

```console
$ mix phoenix.new hello_phoenix
* creating hello_phoenix/README.md
. . .
```

Phoenix 生成应用所需的所有的目录结构以及文件。完成后会询问我们是否安装依赖。选“是”。

```console
Fetch and install dependencies? [Yn] y
* running npm install && node node_modules/brunch/bin/brunch build
* running mix deps.get
```

依赖安装后，会提示我们进入程序目录，运行应用。

```console
We are all set! Run your Phoenix application:

$ cd hello_phoenix
$ mix ecto.create
$ mix phoenix.server

You can also run it inside IEx (Interactive Elixir) as:

    $ iex -S mix phoenix.server
```

Phoenix 假设我们的 PostgreSQL 数据库存在名为 `postgres`，密码为"postgres"的账户，这个账户必须有一定的权限。如果实际情况不符。请查看 [ecto.create](http://www.phoenixframework.org/docs/mix-tasks#section--ecto-create-) mix 任务说明。

Ok, 让我们试一试。

```console
$ cd hello_phoenix
$ mix ecto.create
$ mix phoenix.server
```

> 注意: 如果这是你第一次运行这个命令，Phoenix 会要求安装 Rebar。因为 Rebar 是用来编译 Erlang 包的，所以继续安装。

如果生成新应用的时候没有让 Phoenix 安装依赖，当我们想安装的时候 `phoenix.new` 会提示我们的。

```console
Fetch and install dependencies? [Yn] n

Phoenix uses an optional assets build tool called brunch.io
that requires node.js and npm. Installation instructions for
node.js, which includes npm, can be found at http://nodejs.org.

After npm is installed, install your brunch dependencies by
running inside your app:

    $ npm install

If you don't want brunch.io, you can re-run this generator
with the --no-brunch option.


We are all set! Run your Phoenix application:

    $ cd hello_phoenix
    $ mix deps.get
    $ mix phoenix.server

You can also run it inside IEx (Interactive Elixir) as:

    $ iex -S mix phoenix.server
```

Phoenix 默认在 4000 端口接受请求。使用浏览器打开 [http://localhost:4000](http://localhost:4000)，应该可以看到 Phoenix 框架的欢迎页面。

![Phoenix 欢迎页面](../images/welcome-to-phoenix.png)

如果显示页面类似上图，恭喜你！你现在有了一个正常工作的 Phoenix 应用。如果无法查看，请尝试使用 [http://127.0.0.1:4000](http://127.0.0.1:4000)， 并确保你的系统将 "127.0.0.1" 定义为 "localhost"。

我们的应用跑在本地的一个 iex 会话中。和正常停止 iex 一样，按俩次 ctrl-c 就可以停止我们的应用。

下一步，将稍微修改我们的应用，看看 Phoenix 如何运行的。
