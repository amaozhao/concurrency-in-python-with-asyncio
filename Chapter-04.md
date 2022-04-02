# 构建并发网络爬虫

本章涵盖

- 异步上下文管理器
- 使用 aiohttp 进行异步友好的 Web 请求
- 与收集同时运行 Web 请求
- 处理完成后的结果
- 通过等待跟踪进行中的请求
- 为请求组和取消请求设置和处理超时

在第 3 章中，我们更多地了解了套接字的内部工作原理并构建了一个基本的回显服务器。现在我们已经了解了如何设计一个基本的应用程序，我们将把这些知识应用到并发的、非阻塞的 Web 请求中。将 asyncio 用于 Web 请求允许我们同时发出数百个请求，与同步方法相比，减少了应用程序的运行时间。当我们必须向一组 REST API 发出多个请求时，这很有用，这可能发生在微服务架构中或当我们有网络爬虫任务时。这种方法还允许在我们等待可能很长的 Web 请求完成时运行其他代码，从而使我们能够构建更具响应性的应用程序。

在本章中，我们将学习一个名为 aiohttp 的异步库来实现这一点。该库使用非阻塞套接字发出 Web 请求并为这些请求返回协程，然后我们可以等待结果。具体来说，我们将学习如何获取我们想要获取内容的数百个 URL 的列表，并同时运行所有这些请求。在此过程中，我们将检查 asyncio 提供的用于一次性运行协程的各种 API 方法，允许我们选择等待所有内容完成后再继续，或者尽快处理结果。此外，我们'将看看如何为这些请求设置超时，无论是在单个请求级别还是在一组请求中。我们还将了解如何根据其他请求的执行情况来取消一组正在进行的请求。这些 API 方法不仅对发出 Web 请求很有用，而且在我们需要同时运行一组协程或任务时也很有用。事实上，我们将在本书的其余部分使用我们在此处使用的函数，作为异步开发人员，您将广泛使用它们。

## 4.1 介绍aiohttp
在第 2 章中，我们提到了新手在第一次使用 asyncio 时面临的一个问题是尝试使用他们现有的代码并在其上添加 async 和 await，以期获得性能提升。在大多数情况下，这不起作用，在处理 Web 请求时尤其如此，因为大多数现有库都处于阻塞状态。

一个流行的用于发出 Web 请求的库是 requests 库。这个库在 asyncio 上表现不佳，因为它使用阻塞套接字。这意味着如果我们发出请求，它将阻塞它运行的线程，并且由于 asyncio 是单线程的，我们的整个事件循环将停止，直到该请求完成。

为了解决这个问题并获得并发性，我们需要使用一个一直到套接字层的非阻塞库。 aiohttp（用于 asyncio 和 Python 的异步 HTTP 客户端/服务器）是一个使用非阻塞套接字解决此问题的库。

aiohttp 是一个开源库，是 aio-libs 项目的一部分，该项目自称为“高质量构建的基于异步的库集”（参见 https://github.com/aio-libs）。这个库是一个功能齐全的 Web 客户端和 Web 服务器，这意味着它可以发出 Web 请求，开发人员可以使用它创建异步 Web 服务器。 （该库的文档可在 https://docs.aiohttp.org/ 获得。）在本章中，我们将关注 aiohttp 的客户端，但我们还将在后面的章节中了解如何使用它构建 Web 服务器书。

那么我们如何开始使用 aiohttp 呢？首先要学习的是发出 HTTP 请求。我们首先需要学习一些异步上下文管理器的新语法。使用这种语法将允许我们干净地获取和关闭 HTTP 会话。作为 asyncio 开发人员，您将经常使用此语法来异步获取资源，例如数据库连接。

## 4.2 异步上下文管理器
在任何编程语言中，处理必须打开然后关闭的资源（例如文件）是很常见的。在处理这些资源时，我们需要小心可能引发的任何异常。这是因为如果我们打开一个资源并抛出异常，我们可能永远不会执行任何代码来清理，从而使我们处于资源泄漏的状态。使用 finally 块在 Python 中处理这个问题很简单。虽然这个例子并不完全是 Pythonic，但即使抛出异常，我们也可以随时关闭文件：

```python
file = open('example.txt')
 
try:
    lines = file.readlines()
finally:
    file.close()
```

如果在 file.readlines 期间出现异常，这解决了文件句柄保持打开状态的问题。缺点是我们必须记住将所有内容都包装在 try finally 中，并且我们还需要记住要调用的方法以正确关闭我们的资源。这对文件来说并不难，因为我们只需要记住关闭它们，但我们仍然想要更可重用的东西，特别是因为我们的清理可能比调用一个方法更复杂。 Python 有一个语言特性来处理这个问题，称为上下文管理器。使用它，我们可以将关闭逻辑与 try/finally 块一起抽象：

```python
with open(‘example.txt’) as file:
    lines = file.readlines()
```

这种管理文件的 Pythonic 方式要干净得多。如果 with 块中抛出异常，我们的文件将自动关闭。这适用于同步资源，但是如果我们想异步使用具有这种语法的资源怎么办？在这种情况下，上下文管理器语法将不起作用，因为它仅适用于同步 Python 代码，而不适用于协程和任务。 Python 引入了一种新的语言特性来支持这个用例，称为异步上下文管理器。语法与同步上下文管理器的语法几乎相同，不同之处在于我们说 async with 而不是 with。

异步上下文管理器是实现两个特殊协程方法的类，\_\_aenter\_\_ 异步获取资源和 \_\_aexit\_\_ 关闭该资源。 \_\_aexit\_\_ 协程采用几个参数来处理发生的任何异常，我们不会在本章中进行讨论。

为了充分理解异步上下文管理器，让我们使用我们在第 3 章中介绍的套接字来实现一个简单的上下文管理器。我们可以将客户端套接字连接视为我们想要管理的资源。当客户端连接时，我们获取客户端连接。完成后，我们清理并关闭连接。在第 3 章中，我们将所有内容都包装在一个 try/finally 块中，但我们本可以实现一个异步上下文管理器来代替这样做。

清单 4.1 等待客户端连接的异步上下文管理器

```python
import asyncio
import socket
from types import TracebackType
from typing import Optional, Type
 
 
class ConnectedSocket:
 
    def __init__(self, server_socket):
        self._connection = None
        self._server_socket = server_socket
 
    async def __aenter__(self):                                      ❶
        print('Entering context manager, waiting for connection')
        loop = asyncio.get_event_loop()
        connection, address = await loop.sock_accept(self._server_socket)
        self._connection = connection
        print('Accepted a connection')
        return self._connection
 
    async def __aexit__(self,
                        exc_type: Optional[Type[BaseException]],
                        exc_val: Optional[BaseException],
                        exc_tb: Optional[TracebackType]):            ❷
        print('Exiting context manager')
        self._connection.close()
        print('Closed connection')
 
 
async def main():
    loop = asyncio.get_event_loop()
 
    server_socket = socket.socket()
    server_socket.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    server_address = ('127.0.0.1', 8000)
    server_socket.setblocking(False)
    server_socket.bind(server_address)
    server_socket.listen()
 
    async with ConnectedSocket(server_socket) as connection:         ❸
        data = await loop.sock_recv(connection, 1024)
        print(data)                                                  ❹
 
 
asyncio.run(main())
```

❶ 这个协程在我们进入 with 块时被调用。它一直等到客户端连接并返回连接。
❷ 这个协程在我们退出 with 块时被调用。在其中，我们清理我们使用的任何资源。在这种情况下，我们关闭连接。
❸ 这会调用 \_\_aenter\_\_ 并等待客户端连接。
❹ 在此语句之后，\_\_aenter\_\_ 将执行，我们将关闭我们的连接。
在前面的清单中，我们创建了一个 ConnectedSocket 异步上下文管理器。这个类接受一个服务器套接字，并在我们的 \_\_aenter\_\_ 协程中等待客户端连接。一旦客户端连接，我们返回该客户端的连接。这让我们可以在 async with 语句的 as 部分访问该连接。然后，在我们的 async with 块中，我们使用该连接等待客户端向我们发送数据。一旦这个块完成执行， \_\_aexit\_\_ 协程就会运行并关闭连接。假设客户端连接 Telnet 并发送一些测试数据，运行该程序时我们应该看到如下输出：

```
Entering context manager, waiting for connection
Accepted a connection
b'test\r\n'
Exiting context manager
Closed connection
```

aiohttp 广泛使用异步上下文管理器来获取 HTTP 会话和连接，我们将在第 5 章稍后处理异步数据库连接和事务时使用它。通常，您不需要编写自己的异步上下文管理器，但了解它们的工作原理以及与普通上下文管理器的不同之处会很有帮助。现在我们已经介绍了上下文管理器及其工作原理，让我们将它们与 aiohttp 一起使用，看看如何发出异步 Web 请求。

### 4.2.1 使用 aiohttp 发起 web 请求
我们首先需要安装 aiohttp 库。我们可以通过运行以下命令使用 pip 来做到这一点：

```
pip install -Iv aiohttp==3.8.1
```

这将安装最新版本的 aiohttp（撰写本文时为 3.8.1）。完成后，您就可以开始提出请求了。

aiohttp 和一般的 Web 请求都使用会话的概念。将会话视为打开一个新的浏览器窗口。在新的浏览器窗口中，您将连接到任意数量的网页，这些网页可能会向您发送浏览器为您保存的 cookie。使用会话，您将保持许多连接打开，然后可以回收。这称为连接池。连接池是一个重要的概念，它有助于我们基于 aiohttp 的应用程序的性能。由于创建连接是资源密集型的，因此创建可重用的连接池可以降低资源分配成本。会话还将在内部保存我们收到的任何 cookie，但如果需要，可以关闭此功能。

通常，我们希望利用连接池，因此大多数基于 aiohttp 的应用程序为整个应用程序运行一个会话。然后将此会话对象传递给需要的方法。会话对象上有用于发出任意数量的 Web 请求的方法，例如 GET、PUT 和 POST。我们可以使用 async with 语法和 aiohttp.ClientSession 异步上下文管理器来创建会话。

清单 4.2 发出一个 aiohttp 网络请求

```python
import asyncio
import aiohttp
from aiohttp import ClientSession
from util import async_timed
 
 
@async_timed()
async def fetch_status(session: ClientSession, url: str) -> int:
    async with session.get(url) as result:
        return result.status
 
 
@async_timed()
async def main():
    async with aiohttp.ClientSession() as session:
        url = 'https:/ / www .example .com'
        status = await fetch_status(session, url)
        print(f'Status for {url} was {status}')
 
 
asyncio.run(main())
```

当我们运行它时，我们应该看到 http://www .example .com 的输出状态为 200。在前面的清单中，我们首先使用 aiohttp.ClientSession() 在 async with 块中创建了一个客户端会话。一旦我们有了客户端会话，我们就可以自由地发出任何所需的网络请求。在这种情况下，我们定义了一个方便的方法 fetch_status_code，它将接收一个会话和一个 URL，并返回给定 URL 的状态码。在这个函数中，我们有另一个 async with 块，并使用会话对 URL 运行 GET HTTP 请求。这会给我们一个结果，然后我们可以在 with 块中处理它。在这种情况下，我们只需获取状态码并返回。

请注意，默认情况下，ClientSession 将创建默认最多 100 个连接，为我们可以发出的并发请求数提供隐式上限。要更改此限制，我们可以创建一个 aiohttp TCPConnector 实例，指定最大连接数并将其传递给 ClientSession。要了解更多信息，请查看 https://docs.aiohttp.org/en/stable/client_advanced.html#connectors 上的 aiohttp 文档。

我们将在本章中重用 fetch_status，所以让我们让这个函数可重用。我们将创建一个名为 chapter_04 的 Python 模块，其 __init__.py 包含此函数。然后，我们将在本章后面的示例中将其导入为 from chapter_04 import fetch_status。

> 给 Windows 用户的注意事项
>
> 目前，Windows 上的 aiohttp 存在问题，您可能会看到类似 RuntimeError: Event loop is closed 即使您的应用程序运行良好的错误。在 https://github.com/aio-libs/aiohttp/issues/4324 和 https://bugs.python.org/issue39232 阅读有关此问题的更多信息。要解决此问题，您可以使用 asyncio.get_event_loop().run_until_complete(main()) 手动管理事件循环，如第 2 章所示，或者您可以通过以下方式将事件循环策略更改为 Windows 选择器事件循环策略在 asyncio.run(main()) 之前调用 asyncio.set_event_loop_policy (asyncio.WindowsSelectorEventLoopPolicy())。

### 4.2.2 使用 aiohttp 设置超时
之前我们看到了如何使用 asyncio.wait_ for 为可等待对象指定超时。这也适用于为 aiohttp 请求设置超时，但是设置超时的更简洁的方法是使用 aiohttp 提供的开箱即用的功能。

默认情况下，aiohttp 的超时时间为 5 分钟，这意味着任何单个操作都不应超过此时间。这是一个较长的超时时间，许多应用程序开发人员可能希望将其设置得较低。我们可以在会话级别指定超时，这将为每个操作应用该超时，或者在请求级别，提供更精细的控制。

我们可以使用 aiohttp 特定的 ClientTimeout 数据结构来指定超时。这种结构不仅允许我们为整个请求指定以秒为单位的总超时时间，还允许我们设置建立连接或读取数据的超时时间。让我们通过为我们的会话和一个单独的请求指定一个超时来检查如何使用它。

清单 4.3 使用 aiohttp 设置超时

```python
import asyncio
import aiohttp
from aiohttp import ClientSession
 
async def fetch_status(session: ClientSession,
                       url: str) -> int:
    ten_millis = aiohttp.ClientTimeout(total=.01)
    async with session.get(url, timeout=ten_millis) as result:
        return result.status
 
 
async def main():
    session_timeout = aiohttp.ClientTimeout(total=1, connect=.1)
    async with aiohttp.ClientSession(timeout=session_timeout) as session:
        await fetch_status(session, 'https:/ / example .com')
 
asyncio.run(main())
```

在前面的清单中，我们设置了两个超时。第一次超时是在客户端会话级别。这里我们将总超时时间设置为 1 秒，并明确设置连接超时时间为 100 毫秒。然后，在 fetch_status 中，我们为我们的 get 请求覆盖它，将总超时设置为 10 毫秒。在这种情况下，如果我们对 example.com 的请求超过 10 毫秒，则在等待 fetch_status 时将引发 asyncio.TimeoutError。在此示例中，10 毫秒应该足以完成对 example.com 的请求，因此我们不太可能看到异常。如果您想查看此异常，请将 URL 更改为下载时间超过 10 毫秒的页面。

这些示例向我们展示了 aiohttp 的基础知识。但是，我们的应用程序的性能不会受益于仅使用 asyncio 运行单个请求。当我们同时运行多个 Web 请求时，我们将开始看到真正的好处。

## 4.3 并发运行任务，重温
在本书的前几章中，我们学习了如何创建多个任务来同时运行协同程序。为此，我们使用了 asyncio.create_task，然后等待如下任务：

```python
import asyncio
 
async def main() -> None:
    task_one = asyncio.create_task(delay(1))
    task_two = asyncio.create_task(delay(2))
 
    await task_one
    await task_two
```

这适用于像前一个这样的简单情况，在这种情况下，我们想要同时启动一两个协程。但是，在我们可能同时发出数百、数千甚至更多 Web 请求的世界中，这种风格会变得冗长和混乱。

我们可能很想利用 for 循环或列表推导来使这更平滑一点，如下面的清单所示。但是，如果编写不正确，这种方法可能会导致问题。

清单 4.4 错误地使用带有列表推导的任务

```python
import asyncio
from util import async_timed, delay
 
 
@async_timed()
async def main() -> None:
    delay_times = [3, 3, 3]
    [await asyncio.create_task(delay(seconds)) for seconds in delay_times]
 
asyncio.run(main())
```

鉴于我们理想地希望延迟任务同时运行，我们希望 main 方法在大约 3 秒内完成。但是，在这种情况下，运行需要 9 秒，因为一切都是按顺序完成的：

```
tarting <function main at 0x10f14a550> with args () {}
starting <function delay at 0x10f7684c0> with args (3,) {}
sleeping for 3 second(s)
finished sleeping for 3 second(s)
finished <function delay at 0x10f7684c0> in 3.0008 second(s)
starting <function delay at 0x10f7684c0> with args (3,) {}
sleeping for 3 second(s)
finished sleeping for 3 second(s)
finished <function delay at 0x10f7684c0> in 3.0009 second(s)
starting <function delay at 0x10f7684c0> with args (3,) {}
sleeping for 3 second(s)
finished sleeping for 3 second(s)
finished <function delay at 0x10f7684c0> in 3.0020 second(s)
finished <function main at 0x10f14a550> in 9.0044 second(s)
```

这里的问题很微妙。发生这种情况是因为我们一创建任务就使用 await。这意味着我们为我们创建的每个延迟任务暂停列表理解和主协程，直到该延迟任务完成。在这种情况下，我们将在任何给定时间只运行一个任务，而不是同时运行多个任务。修复很简单，虽然有点冗长。我们可以在一个列表理解中创建任务并在一秒钟内等待。这让一切都可以同时运行。

清单 4.5 同时使用任务和列表推导

```python
import asyncio
from util import async_timed, delay
 
@async_timed()
async def main() -> None:
    delay_times = [3, 3, 3]
    tasks = [asyncio.create_task(delay(seconds)) for seconds in delay_times]
    [await task for task in tasks]
 
asyncio.run(main())
```

此代码在任务列表中一次创建多个任务。一旦我们创建了所有任务，我们就在一个单独的列表理解中等待它们完成。这是有效的，因为 create_task 会立即返回，并且在创建所有任务之前我们不会进行任何等待。这确保了它最多只需要 delay_times 中的最大暂停，运行时间约为 3 秒：

```
starting <function main at 0x10d4e1550> with args () {}
starting <function delay at 0x10daff4c0> with args (3,) {}
sleeping for 3 second(s)
starting <function delay at 0x10daff4c0> with args (3,) {}
sleeping for 3 second(s)
starting <function delay at 0x10daff4c0> with args (3,) {}
sleeping for 3 second(s)
finished sleeping for 3 second(s)
finished <function delay at 0x10daff4c0> in 3.0029 second(s)
finished sleeping for 3 second(s)
finished <function delay at 0x10daff4c0> in 3.0029 second(s)
finished sleeping for 3 second(s)
finished <function delay at 0x10daff4c0> in 3.0029 second(s)
finished <function main at 0x10d4e1550> in 3.0031 second(s)
```

虽然这可以满足我们的要求，但缺点仍然存在。首先是它由多行代码组成，我们必须明确记住将我们的任务创建与我们的等待分开。第二个是它不灵活，如果我们的一个协程比其他协程完成得早，我们将陷入第二个列表推导中，等待所有其他协程完成。虽然在某些情况下这可能是可以接受的，但我们可能希望响应更快，在结果到达后立即处理。第三个也是可能最大的问题是异常处理。如果我们的协程中的一个有异常，它会在我们等待失败的任务时抛出。这意味着我们将无法处理任何成功完成的任务，因为一个异常将停止我们的执行。

asyncio 具有处理所有这些情况以及更多情况的便利功能。同时运行多个任务时建议使用这些功能。在接下来的部分中，我们将研究其中的一些，并研究如何在同时发出多个 Web 请求的上下文中使用它们。

## 4.4 与gather同时运行请求
一个广泛使用的用于并发运行等待的 asyncio API 函数是 asyncio .gather。这个函数接收一系列等待对象，让我们在一行代码中同时运行它们。如果我们传入的任何 awaitables 是协程，gather 将自动将其包装在任务中以确保它同时运行。这意味着我们不必像上面使用的那样用 asyncio.create_task 单独包装所有内容。

asyncio.gather 返回一个可等待的。当我们在 await 表达式中使用它时，它会暂停，直到我们传递给它的所有可等待对象都完成为止。一旦我们传入的所有内容都完成，asyncio.gather 将返回完成结果的列表。

我们可以使用这个函数同时运行尽可能多的 Web 请求。为了说明这一点，让我们看一个例子，我们同时发出 1000 个请求并获取每个响应的状态码。我们将用 @async_ 定时装饰我们的主协程，这样我们就知道事情需要多长时间。

清单 4.6 与收集同时运行请求

```python
import asyncio
import aiohttp
from aiohttp import ClientSession
from chapter_04 import fetch_status
from util import async_timed
 
 
@async_timed()
async def main():
    async with aiohttp.ClientSession() as session:
        urls = ['https:/ / example .com' for _ in range(1000)]
        requests = [fetch_status(session, url) for url in urls]   ❶
        status_codes = await asyncio.gather(*requests)            ❷
        print(status_codes)
 
 
asyncio.run(main())
```

❶ 为我们想要发出的每个请求生成一个协程列表。
❷ 等待所有请求完成。
在前面的清单中，我们首先生成一个我们想要从中检索状态代码的 URL 列表；为简单起见，我们将重复请求 example.com。然后，我们获取该 URL 列表并调用 fetch_status_code 以生成协程列表，然后将其传递给收集。这会将每个协程包装在一个任务中并开始同时运行它们。当我们执行这段代码时，我们会看到 1,000 条消息打印到标准输出，表示 fetch_status_code 协程按顺序启动，表示有 1,000 个请求同时启动。随着结果的出现，我们将在 0.5453 秒内看到完成 <function fetch_status_code at 0x10f3fe3a0> 之类的消息。一旦我们检索到我们请求的所有 URL 的内容，我们将看到状态代码开始打印出来。这个过程很快，取决于机器的互联网连接和速度，这个脚本可以在 500-600 毫秒内完成。

那么这与同步做事相比如何呢？当我们调用 fetch_status_code 时，可以很容易地调整 main 函数，以便通过使用 await 阻塞每个请求。这将暂停每个 URL 的主协程，有效地使事情同步：

```python
@async_timed()
async def main():
    async with aiohttp.ClientSession() as session:
        urls = ['https:/ / example .com' for _ in range(1000)]
        status_codes = [await fetch_status_code(session, url) for url in urls]
        print(status_codes)
```

如果我们运行它，请注意事情会花费更长的时间。我们还会注意到，不是在收到 1,000 条开始的函数 fetch_status_code 消息，然后是 1,000 条完成的函数 fetch_status_code 消息，而是每个请求的显示如下所示：

```
starting <function fetch_status_code at 0x10d95b310>
finished <function fetch_status_code at 0x10d95b310> in 0.01884 second(s)
```

这表明请求一个接一个地发生，等待对 fetch_status_code 的每次调用完成，然后再继续下一个请求。那么这比使用我们的异步版本慢多少呢？虽然这取决于您的互联网连接和运行它的机器，但按顺序运行可能需要大约 18 秒才能完成。与我们的异步版本（大约需要 600 毫秒）相比，后者的运行速度快了 33 倍，令人印象深刻。

值得注意的是，我们传入的每个 awaitable 的结果可能不会以确定的顺序完成。例如，如果我们按顺序传递协程 a 和 b 来收集，b 可能在 a 之前完成。 Gather 的一个很好的特性是，无论我们的 awaitables 何时完成，我们都可以保证结果将按照我们传入的顺序返回。让我们通过查看我们刚刚使用延迟函数描述的场景来演示这一点。

清单 4.7 Awaitables 乱序完成

```python
import asyncio
from util import delay
 
 
async def main():
    results = await asyncio.gather(delay(3), delay(1))
    print(results)
 
asyncio.run(main())
```

在前面的清单中，我们传递了两个协程来收集。第一个需要 3 秒才能完成，第二个需要 1 秒。我们可能期望这个结果是 [1, 3]，因为我们的 1 秒协程在我们的 3 秒协程之前完成，但结果实际上是 [3, 1]——我们传入的顺序。gather 函数尽管幕后存在固有的不确定性，但仍保持结果排序确定性。在后台，gather 使用一种特殊的feature实现来执行此操作。对于好奇的读者，查看gather 的源代码可能是了解有多少使用futures 构建的异步API 的有益方式。

在上面的示例中，假设所有请求都不会失败或抛出异常。这适用于“快乐路径”，但是当请求失败时会发生什么？

### 4.4.1 用gather处理异常
当然，当我们发出一个网络请求时，我们可能并不总是能得到一个值。我们可能会得到一个例外。由于网络可能不可靠，因此可能出现不同的故障情况。例如，我们可以传入一个无效的地址，或者由于该站点已被删除而变得无效。我们连接的服务器也可以关闭或拒绝我们的连接。

asyncio.gather 为我们提供了一个可选参数 return_exceptions，它允许我们指定我们希望如何处理来自等待对象的异常。 return_exceptions 是一个布尔值；因此，它有两种行为可供我们选择：

- return_exceptions=False - 这是收集的默认值。在这种情况下，如果我们的任何协程抛出异常，我们的收集调用也会在等待时抛出该异常。但是，即使我们的一个协程失败了，我们的其他协程也不会被取消，只要我们处理异常就会继续运行，或者异常不会导致事件循环停止并取消任务。
- return_exceptions=True——在这种情况下，gather 将返回任何异常作为我们等待它时返回的结果列表的一部分。对聚集的调用本身不会抛出任何异常，我们将能够按照我们的意愿处理所有异常。

为了说明这些选项是如何工作的，让我们更改我们的 URL 列表以包含一个无效的网址。这将导致 aiohttp 在我们尝试发出请求时引发异常。然后我们将其传递给 collect 并查看每个 return_ 异常的行为方式：

```python
@async_timed()
async def main():
    async with aiohttp.ClientSession() as session:
        urls = ['https://example .com', 'python://example .com']
        tasks = [fetch_status_code(session, url) for url in urls]
        status_codes = await asyncio.gather(*tasks)
        print(status_codes)
```

如果我们将 URL 列表更改为上述内容，则对“python://example.com”的请求将失败，因为该 URL 无效。因此，我们的 fetch_status_code 协程会抛出一个 AssertionError，这意味着 python:// 不会转换为端口。当我们等待收集协程时，将抛出此异常。如果我们运行它并查看输出，我们会看到我们的异常被抛出，但我们也会看到我们的其他请求继续运行（为简洁起见，我们删除了详细的回溯）：

```python
starting <function main at 0x107f4a4c0> with args () {}
starting <function fetch_status_code at 0x107f4a3a0>
starting <function fetch_status_code at 0x107f4a3a0>
finished <function fetch_status_code at 0x107f4a3a0> in 0.0004 second(s)
finished <function main at 0x107f4a4c0> in 0.0203 second(s)
finished <function fetch_status_code at 0x107f4a3a0> in 0.0198 second(s)
Traceback (most recent call last):
  File "gather_exception.py", line 22, in <module>
    asyncio.run(main())
AssertionError
 
Process finished with exit code 1
```

如果出现故障，asyncio.gather 不会取消任何其他正在运行的任务。这对于许多用例来说可能是可以接受的，但这是收集的缺点之一。我们将在本章后面看到如何取消我们同时运行的任务。

上述代码的另一个潜在问题是，如果发生多个异常，我们只会看到等待收集时发生的第一个异常。我们可以通过使用 return_exceptions=True 来解决这个问题，这将返回我们在运行协程时遇到的所有异常。然后我们可以过滤掉任何异常并根据需要处理它们。让我们检查一下我们之前使用无效 URL 的示例，以了解其工作原理：

```python
@async_timed()
async def main():
    async with aiohttp.ClientSession() as session:
        urls = ['https://example .com', 'python://example .com']
        tasks = [fetch_status_code(session, url) for url in urls]
        results = await asyncio.gather(*tasks, return_exceptions=True)
 
        exceptions = [res for res in results if isinstance(res, Exception)]
        successful_results = [res for res in results if not isinstance(res, Exception)]
 
        print(f'All results: {results}')
        print(f'Finished successfully: {successful_results}')
        print(f'Threw exceptions: {exceptions}')
```

运行此程序时，请注意没有抛出异常，并且我们在收集返回的列表中获得所有异常以及我们的成功结果。然后，我们过滤掉任何异常实例以检索成功响应列表，从而产生以下输出：

```
All results: [200, AssertionError()]
Finished successfully: [200]
Threw exceptions: [AssertionError()]
```

这解决了无法看到我们的协程抛出的所有异常的问题。现在我们不需要使用 try catch 块显式处理任何异常，这也很好，因为我们在等待时不再抛出异常。我们必须从成功的结果中过滤掉异常仍然有点笨拙，但是 API 并不完美。

收集有一些缺点。第一个已经提到过，如果抛出异常，取消我们的任务并不容易。想象一下，我们向同一台服务器发出请求，如果一个请求失败，所有其他请求也会失败，例如达到速率限制。在这种情况下，我们可能希望取消请求以释放资源，这并不容易做到，因为我们的协程被包装在后台的任务中。

第二个是我们必须等待所有协程完成才能处理我们的结果。如果我们想在结果完成后立即处理它们，这会带来问题。例如，如果我们有一个请求需要 100 毫秒，而另一个请求持续 20 秒，我们将等待 20 秒，然后才能处理仅在 100 毫秒内完成的请求。

asyncio 提供了允许我们解决这两个问题的 API。让我们先来看一下结果一进来就处理的问题。

## 4.5 在请求完成时处理请求
虽然 asyncio.gather 适用于许多情况，但它的缺点是它会等待所有可等待对象完成，然后才允许访问任何结果。如果我们想在结果一进来就对其进行处理，这是一个问题。如果我们有一些可以快速完成的等待对象和一些可能需要一些时间的等待对象，这也可能是一个问题，因为收集等待所有内容结束。这可能会导致我们的应用程序变得无响应；想象一个用户发出 100 个请求，其中两个很慢，但其余的很快完成。如果一旦请求开始完成，我们可以向用户输出一些信息，那就太好了。

为了处理这种情况，asyncio 公开了一个名为 as_completed 的 API 函数。这个方法接受一个等待列表并返回一个期货的迭代器。然后我们可以迭代这些期货，等待每一个。当 await 表达式完成时，我们将从所有可等待对象中检索首先完成的协程的结果。这意味着我们将能够在结果可用时立即处理它们，但现在没有确定的结果排序，因为我们无法保证哪些请求将首先完成。

为了说明这是如何工作的，让我们模拟一个请求快速完成而另一个请求需要更多时间的情况。我们将在我们的 fetch_status 函数中添加一个延迟参数，并调用 asyncio.sleep 来模拟一个长请求，如下所示：

```python
async def fetch_status(session: ClientSession, url: str, delay: int = 0) -> int:
    await asyncio.sleep(delay)
    async with session.get(url) as result:
        return result.status
```

然后，我们将使用 for 循环遍历从 as_completed 返回的迭代器。

清单 4.8 使用 as_completed

```python
import asyncio
import aiohttp
from aiohttp import ClientSession
from util import async_timed
from chapter_04 import fetch_status
 
 
@async_timed()
async def main():
    async with aiohttp.ClientSession() as session:
        fetchers = [fetch_status(session, 'https:/ / www.example .com', 1),
                    fetch_status(session, 'https:/ / www.example .com', 1),
                    fetch_status(session, 'https:/ / www.example .com', 10)]
 
        for finished_task in asyncio.as_completed(fetchers):
            print(await finished_task)
 
 
asyncio.run(main())
```

在前面的清单中，我们创建了三个协程——两个需要大约 1 秒才能完成，一个需要 10 秒。然后我们将它们传递给 as_completed。在幕后，每个协程都包装在一个任务中并开始并发运行。该例程立即返回一个开始循环的迭代器。当我们进入 for 循环时，我们点击 await finished_task。这里我们暂停执行并等待我们的第一个结果进来。在这种情况下，我们的第一个结果在 1 秒后进来，我们打印状态代码。然后我们再次到达 await 结果，由于我们的请求同时运行，我们应该几乎立即看到第二个结果。最后，我们的 10 秒请求将完成，我们的循环将完成。执行此操作将为我们提供如下输出：

```
starting <function fetch_status at 0x10dbed4c0>
starting <function fetch_status at 0x10dbed4c0>
starting <function fetch_status at 0x10dbed4c0>
finished <function fetch_status at 0x10dbed4c0> in 1.1269 second(s)
200
finished <function fetch_status at 0x10dbed4c0> in 1.1294 second(s)
200
finished <function fetch_status at 0x10dbed4c0> in 10.0345 second(s)
200
finished <function main at 0x10dbed5e0> in 10.0353 second(s)
```

总的来说，迭代 result_iterator 仍然需要大约 10 秒，就像我们使用 asynio.gather 一样；但是，我们能够执行代码以在我们的第一个请求完成后立即打印它的结果。这给了我们额外的时间来处理我们第一个成功完成的协程的结果，而其他人仍在等待完成，从而使我们的应用程序在我们的任务完成时更具响应性。

此功能还提供了对异常处理的更好控制。当一个任务抛出异常时，我们将能够在它发生时对其进行处理，因为当我们等待未来时会抛出异常。

### 4.5.1 as_completed 超时
任何基于 Web 的请求都存在花费很长时间的风险。服务器可能处于沉重的资源负载下，或者我们的网络连接可能很差。之前，我们看到了如何为特定请求添加超时，但是如果我们想为一组请求设置超时怎么办？ as_completed 函数通过提供一个可选的 timeout 参数来支持这个用例，它允许我们以秒为单位指定超时。这将跟踪 as_completed 调用花费了多长时间；如果花费的时间超过超时时间，迭代器中的每个可等待对象都会在我们等待时抛出 TimeoutException。

为了说明这一点，让我们以前面的示例为例，创建两个需要 10 秒才能完成的请求和一个需要 1 秒的请求。然后，我们将在 as_completed 上设置 2 秒的超时。完成循环后，我们将打印出当前正在运行的所有任务。

清单 4.9 在 as_completed 上设置超时

```python
import asyncio
import aiohttp
from aiohttp import ClientSession
from util import async_timed
from chapter_04 import fetch_status
 
 
@async_timed()
async def main():
    async with aiohttp.ClientSession() as session:
        fetchers = [fetch_status(session, 'https:/ / example .com', 1),
                    fetch_status(session, 'https:/ / example .com', 10),
                    fetch_status(session, 'https:/ / example .com', 10)]
 
        for done_task in asyncio.as_completed(fetchers, timeout=2):
            try:
                result = await done_task
                print(result)
            except asyncio.TimeoutError:
                print('We got a timeout error!')
 
        for task in asyncio.tasks.all_tasks():
            print(task)
 
 
asyncio.run(main())
```

当我们运行它时，我们会注意到我们第一次获取的结果，并且在 2 秒后，我们会看到两个超时错误。我们还将看到两个 fetches 仍在运行，输出类似于以下内容：

```
starting <function main at 0x109c7c430> with args () {}
200
We got a timeout error!
We got a timeout error!
finished <function main at 0x109c7c430> in 2.0055 second(s)
<Task pending name='Task-2' coro=<fetch_status_code()>>
<Task pending name='Task-1' coro=<main>>
<Task pending name='Task-4' coro=<fetch_status_code()>>
```

as_completed 非常适合尽快获得结果，但也有缺点。首先是当我们得到结果时，没有任何方法可以轻松地看到我们正在等待哪个协程或任务，因为顺序是完全不确定的。如果我们不关心顺序，这可能没问题，但如果我们需要以某种方式将结果与请求相关联，我们就会面临挑战。

第二个是超时，虽然我们会正确地抛出异常并继续前进，但创建的任何任务仍将在后台运行。如果我们想取消它们，很难确定哪些任务仍在运行，我们面临另一个挑战。如果这些是我们需要处理的问题，那么我们将需要一些更细粒度的知识来了解哪些 awaitables 已完成，哪些未完成。为了处理这种情况，asyncio 提供了另一个 API 函数，称为 wait。

## 4.6 带等待的细粒度控制
collect 和 as_completed 的缺点之一是，当我们看到异常时，没有简单的方法可以取消已经在运行的任务。这在很多情况下可能没问题，但是想象一个用例，我们进行了几次协程调用，如果第一个调用失败，其余的也会失败。例如，将无效参数传递给 Web 请求或达到 API 速率限制。这有可能导致性能问题，因为我们将通过执行比我们需要的更多的任务来消耗更多的资源。我们注意到 as_completed 的另一个缺点是，由于迭代顺序是不确定的，因此很难准确跟踪已完成的任务。

asyncio 中的等待类似于收集等待，它提供更具体的控制来处理这些情况。这种方法有几个选项可供选择，具体取决于我们何时想要我们的结果。此外，此方法返回两组：一组以结果或异常完成的任务，以及一组仍在运行的任务。这个函数还允许我们指定一个与其他 API 方法操作方式不同的超时；它不会抛出异常。需要时，此函数可以解决我们迄今为止使用的其他 asyncio API 函数所注意到的一些问题。

wait 的基本签名是一个可等待对象的列表，后跟一个可选的 timeout 和一个可选的 return_when 字符串。这个字符串有一些我们将要检查的预定义值：ALL_COMPLETED、FIRST_EXCEPTION 和 FIRST_COMPLETED。它默认为 ALL_COMPLETED。虽然在撰写本文时，wait 需要一个可等待对象列表，但它会在 Python 的未来版本中更改为仅接受任务对象。我们将在本节末尾了解原因，但对于这些代码示例，由于这是最佳实践，我们将所有协程包装在任务中。

### 4.6.1 等待所有任务完成
如果未指定 return_when，则此选项是默认行为，并且它的行为与 asyncio.gather 最接近，尽管它有一些差异。正如暗示的那样，使用此选项将等待所有任务完成后再返回。让我们将其应用于我们同时发出多个 Web 请求的示例，以了解此功能的工作原理。

清单 4.10 检查等待的默认行为

```python
import asyncio
import aiohttp
from aiohttp import ClientSession
from util import async_timed
from chapter_04 import fetch_status
 
 
@async_timed()
async def main():
    async with aiohttp.ClientSession() as session:
        fetchers = \
            [asyncio.create_task(fetch_status(session, 'https:/ /example.com')),
             asyncio.create_task(fetch_status(session, 'https:/ /example.com'))]
        done, pending = await asyncio.wait(fetchers)
 
        print(f'Done task count: {len(done)}')
        print(f'Pending task count: {len(pending)}')
 
        for done_task in done:
            result = await done_task
            print(result)
 
 
asyncio.run(main())
```

在前面的清单中，我们通过传递要等待的协程列表同时运行两个 Web 请求。当我们等待时，一旦所有请求完成，它将返回两组：一组已完成的所有任务和一组仍在运行的任务。完成集包含所有成功完成或有异常完成的任务。待处理集包含所有尚未完成的任务。在这种情况下，由于我们使用的是 ALL_COMPLETED 选项，因此挂起的集合将始终为零，因为 asyncio.wait 直到一切都完成后才会返回。这将为我们提供以下输出：

```python
starting <function main at 0x10124b160> with args () {}
Done task count: 2
Pending task count: 0
200
200
finished <function main at 0x10124b160> in 0.4642 second(s)
```

如果我们的请求之一抛出异常，它不会像 asyncio.gather 那样在 asyncio.wait 调用中被抛出。在这种情况下，我们将像以前一样同时获得完成集和待处理集，但是在等待失败的完成任务之前，我们不会看到异常。

有了这个范式，我们有一些关于如何处理异常的选择。我们可以使用 await 并让异常抛出，我们可以使用 await 并将其包装在 try except 块中来处理异常，或者我们可以使用 task.result() 和 task.exception() 方法。我们可以安全地调用这些方法，因为完成集中的任务保证是已完成的任务；如果他们没有调用这些方法，则会产生异常。

假设我们不想抛出异常并让我们的应用程序崩溃。相反，如果我们有任务的结果，我们想打印它，如果有异常则记录一个错误。在这种情况下，使用 Task 对象上的方法是一个合适的解决方案。让我们看看如何使用这两个 Task 方法来处理异常。

清单 4.11 带有等待的异常

```python
import asyncio
import logging
 
@async_timed()
async def main():
    async with aiohttp.ClientSession() as session:
        good_request = fetch_status(session, 'https:/ / www .example .com')
        bad_request = fetch_status(session, 'python:/ /bad')
 
        fetchers = [asyncio.create_task(good_request),
                    asyncio.create_task(bad_request)]
 
        done, pending = await asyncio.wait(fetchers)
 
        print(f'Done task count: {len(done)}')
        print(f'Pending task count: {len(pending)}')
 
        for done_task in done:
            # result = await done_task will throw an exception
            if done_task.exception() is None:
                print(done_task.result())
            else:
                logging.error("Request got an exception",
                              exc_info=done_task.exception())
 
 
asyncio.run(main())
```

使用 done_task.exception() 将检查我们是否有异常。如果我们不这样做，那么我们可以继续使用 result 方法从 done_task 获取结果。在这里执行 result = await done_task 也是安全的，尽管它可能会抛出异常，这可能不是我们想要的。如果异常不是 None，那么我们知道 awaitable 有异常，我们可以根据需要进行处理。这里我们只是打印出异常的堆栈跟踪。运行此程序将产生类似于以下内容的输出（为简洁起见，我们删除了详细的回溯）：

```
starting <function main at 0x10401f1f0> with args () {}
Done task count: 2
Pending task count: 0
200
finished <function main at 0x10401f1f0> in 0.12386679649353027 second(s)
ERROR:root:Request got an exception
Traceback (most recent call last):
AssertionError
```

### 4.6.2 观察异常
ALL_COMPLETED 的缺点就像我们在 collect 中看到的缺点一样。在等待其他协程完成时，我们可能会遇到任意数量的异常，直到所有任务完成后我们才会看到。如果由于一个异常，我们想取消其他正在运行的请求，这可能是一个问题。我们可能还想立即处理任何错误以确保响应并继续等待其他协程完成。

为了支持这些用例，wait 支持 FIRST_EXCEPTION 选项。当我们使用这个选项时，我们会得到两种不同的行为，这取决于我们的任何任务是否抛出异常。

> 任何可等待对象都没有例外
>
> 如果我们的任何任务都没有异常，则此选项等效于 ALL_COMPLETED。我们将等待所有任务完成，然后完成集将包含所有已完成的任务，待处理集将为空。

> 一项或多项任务异常
>
> 如果任何任务抛出异常，一旦抛出异常，wait 将立即返回。完成集将包含任何成功完成的协程以及任何有异常的协程。在这种情况下，完成集至少保证有一个失败的任务，但可能已经成功完成了任务。挂起的集合可能是空的，但它也可能有仍在运行的任务。然后，我们可以根据需要使用这个待处理的集合来管理当前正在运行的任务。

为了说明等待在这些场景中的行为方式，看看当我们有几个长时间运行的 Web 请求时，当一个协程立即失败并出现异常时我们想取消时会发生什么。

清单 4.12 取消正在运行的异常请求

```python
import aiohttp
import asyncio
import logging
from chapter_04 import  fetch_status
from util import async_timed
 
 
@async_timed()
async def main():
    async with aiohttp.ClientSession() as session:
        fetchers = \
            [asyncio.create_task(fetch_status(session, 'python:/ / bad.com')),
             asyncio.create_task(fetch_status(session, 'https:/ / www.example
                                              .com', delay=3)),
             asyncio.create_task(fetch_status(session, 'https:/ / www.example
                                              .com', delay=3))]
 
        done, pending = await asyncio.wait(fetchers, return_when=asyncio.FIRST_EXCEPTION)
 
        print(f'Done task count: {len(done)}')
        print(f'Pending task count: {len(pending)}')
        for done_task in done:
            if done_task.exception() is None:
                print(done_task.result())
            else:
                logging.error("Request got an exception",
                              exc_info=done_task.exception())
 
        for pending_task in pending:
            pending_task.cancel()
 
 
asyncio.run(main())
```

在前面的清单中，我们提出了一个坏请求和两个好请求；每个持续 3 秒。当我们等待我们的等待语句时，我们几乎立即返回，因为我们的错误请求立即出错。然后我们遍历完成的任务。在这种情况下，自从我们的第一个请求立即以异常结束后，我们将只有一个在完成集中。为此，我们将执行打印异常的分支。

待处理的集合将有两个元素，因为我们有两个请求，每个请求大约需要 3 秒才能运行，而我们的第一个请求几乎立即失败。由于我们想阻止它们运行，我们可以调用它们的取消方法。这将为我们提供以下输出：

```
starting <function main at 0x105cfd280> with args () {}
Done task count: 1
Pending task count: 2
finished <function main at 0x105cfd280> in 0.0044 second(s)
ERROR:root:Request got an exception
```

注意我们的应用程序几乎没有时间运行，因为我们很快就对我们的一个请求引发了异常做出了反应。使用此选项的强大之处在于我们实现了快速失败的行为，对出现的任何问题做出快速反应。

### 4.6.3 完成后处理结果
ALL_COMPLETED 和 FIRST_EXCEPTION 都有一个缺点，在协程成功并且不抛出异常的情况下，我们必须等待所有协程完成。根据用例，这可能是可以接受的，但如果我们想要在协程成功完成后立即响应协程，那么我们就不走运了。

在我们想要在结果完成后立即对其做出反应的情况下，我们可以使用 as_completed;但是， as_completed 的问题是没有简单的方法可以查看哪些任务还剩下，哪些任务已经完成。我们通过迭代器一次只得到一个。

好消息是 return_when 参数接受 FIRST_COMPLETED 选项。此选项将在等待协程至少有一个结果时立即返回。这可以是失败的协程，也可以是成功运行的协程。然后，我们可以取消其他正在运行的协程或调整哪些协程继续运行，具体取决于我们的用例。让我们使用此选项发出一些 Web 请求并处理先完成的请求。

清单 4.13 完成时的处理

```python
import asyncio
import aiohttp
from util import async_timed
from chapter_04 import fetch_status
 
 
@async_timed()
async def main():
    async with aiohttp.ClientSession() as session:
        url = 'https:/ / www .example .com'
        fetchers = [asyncio.create_task(fetch_status(session, url)),
                    asyncio.create_task(fetch_status(session, url)),
                    asyncio.create_task(fetch_status(session, url))]
 
        done, pending = await asyncio.wait(fetchers, return_when=asyncio.FIRST_COMPLETED)
 
        print(f'Done task count: {len(done)}')
        print(f'Pending task count: {len(pending)}')
 
        for done_task in done:
            print(await done_task)
 
 
asyncio.run(main())
```

在前面的清单中，我们同时启动了三个请求。只要这些请求中的任何一个完成，我们的等待协程就会返回。这意味着 done 将有一个完整的请求，而 pending 将包含仍在运行的任何内容，为我们提供以下输出：

```
starting <function main at 0x10222f1f0> with args () {}
Done task count: 1
Pending task count: 2
200
finished <function main at 0x10222f1f0> in 0.1138 second(s)
```

这些请求几乎可以同时完成，因此我们还可以看到显示两三个任务已完成的输出。尝试运行此列表几次，看看结果如何变化。

这种方法可以让我们在第一个任务完成时立即做出响应。如果我们想像 as_completed 那样处理其余的结果怎么办？可以很容易地采用上面的示例来循环挂起的任务，直到它们为空。这将为我们提供类似于 as_completed 的行为，其好处是在每一步我们都可以准确地知道哪些任务已经完成以及哪些仍在运行。

清单 4.14 处理所有进来的结果

```python
import asyncio
import aiohttp
from chapter_04 import fetch_status
from util import async_timed
 
 
@async_timed()
async def main():
    async with aiohttp.ClientSession() as session:
        url = 'https://www.example.com'
        pending = [asyncio.create_task(fetch_status(session, url)),
                   asyncio.create_task(fetch_status(session, url)),
                   asyncio.create_task(fetch_status(session, url))]
 
        while pending:
            done, pending = await asyncio.wait(pending, return_when=asyncio.FIRST_COMPLETED)
 
            print(f'Done task count: {len(done)}')
            print(f'Pending task count: {len(pending)}')
 
            for done_task in done:
                print(await done_task)
 
asyncio.run(main())
```

在前面的清单中，我们创建了一个名为 pending 的集合，我们将其初始化为我们想要运行的协程。当我们在待处理集合中有项目时，我们循环并在每次迭代时使用该集合调用等待。一旦我们得到等待的结果，我们更新完成和待处理的集合，然后打印出任何完成的任务。这将为我们提供类似于 as_completed 的行为，不同之处在于我们可以更好地了解哪些任务已完成以及哪些任务仍在运行。运行它，我们将看到以下输出：

```
starting <function main at 0x10d1671f0> with args () {}
Done task count: 1
Pending task count: 2
200
Done task count: 1
Pending task count: 1
200
Done task count: 1
Pending task count: 0
200
finished <function main at 0x10d1671f0> in 0.1153 second(s)
```

由于请求函数可能很快完成，以至于所有请求同时完成，所以我们也看到类似这样的输出并非不可能：

```
starting <function main at 0x1100f11f0> with args () {}
Done task count: 3
Pending task count: 0
200
200
200
finished <function main at 0x1100f11f0> in 0.1304 second(s)
```

### 4.6.4 处理超时
除了允许我们对如何等待协程完成进行更细粒度的控制外，wait 还允许我们设置超时以指定我们希望所有等待完成的时间。要启用此功能，我们可以将 timeout 参数设置为所需的最大秒数。如果我们超过了这个超时时间，wait 将返回已完成和待处理的任务集。与我们目前所看到的 wait_for 和 as_completed 相比，超时在等待中的行为方式存在一些差异。

> 协程不会被取消
>
> 当我们使用 wait_for 时，如果我们的协程超时，它会自动为我们请求取消。等待的情况并非如此；它的行为更接近我们在收集和 as_completed 中看到的情况。如果我们想因为超时而取消协程，我们必须显式地遍历任务并取消它们。

> 不会引发超时错误
>
> wait 与 wait_for 和 as_ completed 一样，在超时事件中不依赖异常。相反，如果发生超时，则等待返回所有已完成的任务以及在发生超时时仍处于挂起状态的所有任务。

例如，让我们来看看两个请求快速完成而一个需要几秒钟的情况。我们将使用 1 秒的超时等待来了解当我们的任务花费的时间超过超时时会发生什么。对于 return_when 参数，我们将使用默认值 ALL_COMPLETED。

清单 4.15 使用带有等待的超时

```python
@async_timed()
async def main():
    async with aiohttp.ClientSession() as session:
        url = 'https://example.com'
        fetchers = [asyncio.create_task(fetch_status(session, url),
                    asyncio.create_task(fetch_status(session, url),
                    asyncio.create_task(fetch_status(session, url, delay=3))]
 
        done, pending = await asyncio.wait(fetchers, timeout=1)
 
        print(f'Done task count: {len(done)}')
        print(f'Pending task count: {len(pending)}')
        for done_task in done:
            result = await done_task
            print(result)
 
 
asyncio.run(main())
```

运行前面的清单，我们的等待调用将在 1 秒后返回我们的完成集和待处理集。在完成的集合中，我们将看到两个快速请求，因为它们在 1 秒内完成。我们的慢请求仍在运行，因此处于待处理集中。然后我们等待完成的任务以提取它们的返回值。如果我们愿意，我们也可以取消挂起的任务。运行此代码，我们将看到以下输出：

```
starting <function main at 0x11c68dd30> with args () {}
Done task count: 2
Pending task count: 1
200
200
finished <function main at 0x11c68dd30> in 1.0022 second(s)
```

请注意，和以前一样，我们在待处理集中的任务不会被取消，并且会继续运行，尽管超时。如果我们有一个要终止待处理任务的用例，我们需要显式地遍历待处理集并在每个任务上调用取消。

### 4.6.5 为什么将所有内容都包装在一个任务中？
在本节的开头，我们提到最好将我们传入的协程包装到任务中的等待中。为什么是这样？让我们回到我们之前的超时示例并稍微改变一下。假设我们有两个不同的 Web API 请求，我们将调用 API A 和 API B。两者都可能很慢，但我们的应用程序可以在没有来自 API B 的结果的情况下运行，所以它只是“很高兴拥有”。由于我们想要一个响应式应用程序，我们设置了 1 秒的超时时间来完成请求。如果在该超时之后对 API B 的请求仍处于未决状态，我们将取消它并继续。让我们看看如果我们在不将请求包装在任务中的情况下实现它会发生什么。

清单 4.16 取消慢速请求

```python
import asyncio
import aiohttp
from chapter_04 import fetch_status
 
 
async def main():
    async with aiohttp.ClientSession() as session:
        api_a = fetch_status(session, 'https://www.example.com')
        api_b = fetch_status(session, 'https:/ /www.example.com', delay=2)
 
        done, pending = await asyncio.wait([api_a, api_b], timeout=1)
 
        for task in pending:
            if task is api_b:
                print('API B too slow, cancelling')
                task.cancel()
 
asyncio.run(main())
```

我们希望这段代码打印出 API B 太慢并且取消，但是如果我们根本看不到这条消息会发生什么？这可能会发生，因为当我们只使用协程调用 wait 时，它们会自动包装在任务中，并且返回的完成集和待处理集是为我们创建的等待任务。这意味着我们无法进行任何比较来查看待处理集中的特定任务，例如如果任务是 api_b，因为我们将比较一个任务对象，我们无法访问协程。但是，如果我们将 fetch_status 包装在一个任务中，wait 不会创建任何新对象，并且如果 task 是 api_b 的比较将按预期进行。在这种情况下，我们正确地比较了两个任务对象。

## 概括

- 我们已经学会了如何使用和创建我们自己的异步上下文管理器。这些是允许我们异步获取资源然后释放它们的特殊类，即使发生异常也是如此。这些让我们可以清理我们可能以非详细方式获取的任何资源，并且在处理 HTTP 会话和数据库连接时很有用。我们可以将它们与特殊的 async with 语法一起使用。
- 我们可以使用 aiohttp 库来发出异步 Web 请求。 aiohttp 是一个使用非阻塞套接字的 Web 客户端和服务器。使用 Web 客户端，我们可以以不阻塞事件循环的方式同时执行多个 Web 请求。
- asyncio.gather 函数让我们可以同时运行多个协程并等待它们完成。一旦我们传递给它的所有等待完成，这个函数就会返回。如果我们想跟踪发生的任何错误，我们可以将 return_exeptions 设置为 True。这将返回成功完成的可等待对象的结果以及我们收到的任何异常。
- 我们可以使用 as_completed 函数来处理等待列表的结果，一旦它们完成。这将为我们提供一个可以循环的期货迭代器。一旦协程或任务完成，我们就可以访问结果并进行处理。
- 如果我们想同时运行多个任务，但又希望能够了解哪些任务已完成，哪些仍在运行，我们可以使用等待。这个函数还允许我们更好地控制它何时返回结果。当它返回时，我们得到一组已完成的任务和一组仍在运行的任务。然后我们可以取消任何我们希望的任务或做任何其他我们需要的等待。