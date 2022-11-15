# ASYNCIO 的高级用法

本章涵盖

- 为协程和函数设计 API
- 协程上下文局部变量
- 屈服于事件循环
- 使用不同的事件循环实现
- 协程和生成器的关系
- 使用自定义等待对象创建自己的事件循环

我们已经了解了 asyncio 提供的大部分功能。使用前面章节中介绍的 asyncio 模块，你应该能够完成几乎所有你需要的任务。也就是说，你可能还需要使用一些鲜为人知的技术，尤其是在你设计自己的 asyncio API 时。

在本章中，我们将学习 asyncio 中可用的更高级技术的大杂烩。我们将学习如何设计可以同时处理协程和常规 Python 函数的 API，如何强制事件循环的迭代，以及如何在不传递参数的情况下在任务之间传递状态。我们还将深入了解 asyncio 究竟如何使用生成器来充分了解幕后发生的事情。我们将通过实现我们自己的自定义等待对象并使用它们来构建我们自己的简单实现可以同时运行多个协程的事件循环来做到这一点。

除非你正在构建依赖异步编程内部的新 API 或框架，否则你在日常开发任务中不太可能需要本章介绍的大量内容。这些技术主要适用于这些应用程序以及希望更深入了解异步 Python 内部的好奇者。

## 14.1 带有协程和函数的 API
如果我们自己构建 API，我们可能不想假设我们的用户在他们自己的异步应用程序中使用我们的库。他们可能尚未迁移，或者他们可能无法从异步堆栈中获得任何好处并且永远不会迁移。我们如何设计一个同时接受协程和普通 Python 函数的 API 来适应这些类型的用户？

asyncio 提供了几个方便的函数来帮助我们做到这一点：asyncio .iscoroutine 和 asyncio.iscoroutinefunction。这些函数让我们测试可调用对象是否是协程，让我们基于此应用不同的逻辑。正如我们在第 9 章中看到的，这些函数是 Django 如何无缝处理同步和异步视图的基础。

为了看到这一点，让我们构建一个接受函数和协程的示例任务运行器类。此类将允许用户将函数添加到内部列表中，然后当用户在我们的任务运行器上调用 start 方法时，我们将同时运行（如果它们是协程）或串行（如果它们是普通函数）。

清单 14.1 一个任务运行器类

```python
import asyncio
 
 
class TaskRunner:
 
    def __init__(self):
        self.loop = asyncio.new_event_loop()
        self.tasks = []
 
    def add_task(self, func):
        self.tasks.append(func)
 
    async def _run_all(self):
        awaitable_tasks = []
 
        for task in self.tasks:
            if asyncio.iscoroutinefunction(task):
                awaitable_tasks.append(asyncio.create_task(task()))
            elif asyncio.iscoroutine(task):
                awaitable_tasks.append(asyncio.create_task(task))
            else:
                self.loop.call_soon(task)
 
        await asyncio.gather(*awaitable_tasks)
 
    def run(self):
        self.loop.run_until_complete(self._run_all())
 
 
if __name__ == "__main__":
 
    def regular_function():
        print('Hello from a regular function!')
 
 
    async def coroutine_function():
        print('Running coroutine, sleeping!')
        await asyncio.sleep(1)
        print('Finished sleeping!')
 
 
    runner = TaskRunner()
    runner.add_task(coroutine_function)
    runner.add_task(coroutine_function())
    runner.add_task(regular_function)
 
    runner.run()
```

在前面的清单中，我们的任务运行器创建了一个新的事件循环和一个空任务列表。然后我们定义一个 add 方法，它只是将一个函数（或协程）添加到待处理的任务列表中。然后，一旦用户调用 run()，我们就会在事件循环中启动 _run_all 方法。我们的 _run_all 方法遍历我们的任务列表并检查所讨论的函数是否是协程。如果是协程，我们创建一个任务；否则，我们使用事件循环 call_soon 方法来安排我们的普通函数在事件循环的下一次迭代中运行。然后，一旦我们创建了我们需要的所有任务，我们就调用收集它们并等待它们全部完成。

然后我们定义了两个函数：一个普通的 Python 函数，恰当地命名为 regular_function 和一个名为 coroutine_function 的协程。我们创建一个 TaskRunner 实例并添加三个任务，调用 coroutine_function 两次来演示我们可以在 API 中引用协程的两种不同方式。这给了我们以下输出：

```sh
Running coroutine, sleeping!
Running coroutine, sleeping!
Hello from a regular function!
Finished sleeping!
Finished sleeping!
```

这表明我们已经成功地运行了协程和普通的 Python 函数。我们现在已经构建了一个 API，它可以处理协程以及普通的 Python 函数，增加了最终用户使用我们 API 的方式。接下来，我们将查看上下文变量，它使我们能够存储任务本地的状态，而无需将其作为函数参数显式传递。

## 14.2 上下文变量
想象一下，我们正在使用一个 REST API，该 API 是基于每个请求线程的 Web 服务器构建的。当对 Web 服务器的请求进入时，我们可能有关于发出请求的用户的公共数据，我们需要跟踪这些数据，例如用户 ID、访问令牌或其他信息。我们可能很想在 Web 服务器的所有线程中全局存储这些数据，但这有缺点。主要缺点是我们需要处理从线程到其数据的映射以及任何锁定以防止竞争条件。我们可以通过使用称为线程局部变量的概念来解决这个问题。线程局部变量是特定于一个线程的全局状态。我们在本地线程中设置的这些数据将由设置它的线程看到，并且只能由该线程看到，避免任何线程到数据的映射以及竞争条件。虽然我们不会详细介绍线程局部变量，但你可以在 https://docs.python.org/3/library/threading.html#thread-local- 上的线程模块文档中阅读更多关于它们的信息数据。

当然，在异步应用程序中，我们通常只有一个线程，因此我们存储为本地线程的任何内容都可以在应用程序的任何位置使用。在 PEP-567 (https://www.python.org/dev/peps/pep-0567) 中，引入了上下文变量的概念来处理单线程并发模型中本地线程的概念。上下文变量类似于线程本地变量，不同之处在于它们是特定任务的本地变量，而不是线程的本地变量。这意味着如果一个任务创建了一个上下文变量，那么该初始任务中的任何内部协程或任务都可以访问该变量。该链之外的任何其他任务都无法查看或修改该变量。这使我们可以跟踪特定于一项任务的状态，而无需将其作为显式参数传递。

为了看一个例子，我们将创建一个简单的服务器来监听来自连接客户端的数据。我们将创建一个上下文变量来跟踪连接用户的地址，当用户发送消息时，我们将打印出他们的地址以及他们发送的消息。

清单 14.2 带有上下文变量的服务器

```python
import asyncio
from asyncio import StreamReader, StreamWriter
from contextvars import ContextVar
 
 
class Server:
    user_address = ContextVar('user_address')                             ❶
 
    def __init__(self, host: str, port: int):
        self.host = host
        self.port = port
 
    async def start_server(self):
        server = await asyncio.start_server(self._client_connected,
                                            self.host, self.port)
        await server.serve_forever()
 
    def _client_connected(self, reader: StreamReader, writer: StreamWriter):
        self.user_address.set(writer.get_extra_info('peername'))          ❷
        asyncio.create_task(self.listen_for_messages(reader))
 
    async def listen_for_messages(self, reader: StreamReader):
        while data := await reader.readline():
            print(f'Got message {data} from {self.user_address.get()}')   ❸
 
 
async def main():
    server = Server('127.0.0.1', 9000)
    await server.start_server()
 
 
asyncio.run(main())
```

❶ 创建一个名为“user_address”的上下文变量。
❷ 当客户端连接时，将客户端的地址存储在上下文变量中。
❸ 在上下文变量的地址旁边显示用户的消息。
在前面的清单中，我们首先创建了一个 ContextVar 的实例来保存我们用户的地址信息。上下文变量要求我们提供一个字符串名称，所以这里我们给它一个描述性名称 user_address，主要用于调试目的。然后在我们的 _client_connected 回调中，我们将上下文变量的数据设置为客户端的地址。这将允许从该父任务产生的任何任务都可以访问我们设置的信息；在这种情况下，这将是侦听来自客户端的消息的任务。

在我们的 listen_for_messages 协程方法中，我们侦听来自客户端的数据，当我们得到它时，我们将它与我们存储在上下文变量中的地址一起打印出来。当你运行此应用程序并连接多个客户端并发送一些消息时，你应该会看到如下输出：

```sh
Got message b'Hello!\r\n' from ('127.0.0.1', 50036)
Got message b'Okay!\r\n' from ('127.0.0.1', 50038)
```

请注意，地址的端口号不同，表明我们从 localhost 上的两个不同客户端获取了消息。即使我们只创建了一个上下文变量，我们仍然能够访问特定于每个客户端的唯一数据。这为我们提供了一种在任务之间传递数据的简洁方式，而无需显式地将数据传递给任务或该任务中的其他方法调用。

## 14.3 强制事件循环迭代
事件循环如何在内部运行大部分是我们无法控制的。它决定何时以及如何执行协程和任务。也就是说，如果我们需要，有一种方法可以触发事件循环迭代。这对于长时间运行的任务可以派上用场，以避免阻塞事件循环（尽管在这种情况下你也应该考虑线程）或确保任务立即启动。

回想一下，如果我们正在创建多个任务，它们都不会开始运行，直到我们点击等待，这将触发事件循环来安排并开始运行它们。如果我们希望每个任务立即开始运行怎么办？

asyncio 提供了一个优化的习惯用法来暂停当前协程并通过将零传递给 asyncio.sleep 来强制事件循环的迭代。让我们看看如何在创建任务后立即使用它来开始运行任务。我们将创建两个函数：一个不使用睡眠，另一个用于比较事物运行的顺序。

清单 14.3 强制事件循环迭代

```python
import asyncio
from util import delay
 
 
async def create_tasks_no_sleep():
    task1 = asyncio.create_task(delay(1))
    task2 = asyncio.create_task(delay(2))
    print('Gathering tasks:')
    await asyncio.gather(task1, task2)
 
 
async def create_tasks_sleep():
    task1 = asyncio.create_task(delay(1))
    await asyncio.sleep(0)
    task2 = asyncio.create_task(delay(2))
    await asyncio.sleep(0)
    print('Gathering tasks:')
    await asyncio.gather(task1, task2)
 
 
async def main():
    print('--- Testing without asyncio.sleep(0) ---')
    await create_tasks_no_sleep()
    print('--- Testing with asyncio.sleep(0) ---')
    await create_tasks_sleep()
 
asyncio.run(main())
```

当我们运行上述清单时，我们将看到以下输出：

```sh
--- Testing without asyncio.sleep(0) ---
Gathering tasks:
sleeping for 1 second(s)
sleeping for 2 second(s)
finished sleeping for 1 second(s)
finished sleeping for 2 second(s)
--- Testing with asyncio.sleep(0) ---
sleeping for 1 second(s)
sleeping for 2 second(s)
Gathering tasks:
finished sleeping for 1 second(s)
finished sleeping for 2 second(s)
```

我们首先创建两个任务，然后在不使用 asyncio.sleep(0) 的情况下收集它们，这会按照我们通常预期的方式运行，两个延迟协程直到我们的收集语句才会运行。接下来，我们在创建每个任务后插入一个 asyncio.sleep(0)。在输出中，你会注意到延迟协程的消息在我们调用任务之前立即打印。使用 sleep 会强制进行事件循环迭代，这会导致我们任务中的代码立即执行。

我们几乎一直在使用事件循环的 asyncio 实现。但是，如果需要，我们可以交换其他实现。接下来，让我们看看如何使用不同的事件循环。

## 14.4 使用不同的事件循环实现
asyncio 提供了一个事件循环的默认实现，到目前为止我们一直在使用它，但完全可以使用可能具有不同特征的不同实现。有几种方法可以使用不同的实现。一种是继承 AbstractEventLoop 类并实现其方法，创建 this 的实例，然后使用 asyncio.set_event_loop 函数将其设置为事件循环。如果我们正在构建自己的自定义实现，这是有道理的，但是我们可以使用现成的事件循环。让我们看一个这样的实现，称为 uvloop。

那么，什么是 uvloop，为什么要使用它？ uvloop 是一个事件循环的实现，它严重依赖于 libuv 库 (https://libuv.org)，它是 node.js 运行时的主干。由于 libuv 是用 C 实现的，因此它比纯解释型 Python 代码具有更好的性能。因此，uvloop 可以比默认的 asyncio 事件循环更快。在编写基于套接字和流的应用程序时，它往往表现得特别好。你可以在项目的 github 站点 https://github.com/magicstack/uvloop 上阅读有关基准测试的更多信息。请注意，在撰写本文时，uvloop 仅在 *Nix 平台上可用。

首先，让我们先使用以下命令安装最新版本的 uvloop：

```sh
pip -Iv uvloop==0.16.0
```

一旦我们安装了 libuv，我们就可以使用它了。我们将制作一个简单的回显服务器，并将使用事件循环的 uvloop 实现。

清单 14.4 使用 uvloop 作为事件循环

```python
import asyncio
from asyncio import StreamReader, StreamWriter
import uvloop
 
 
async def connected(reader: StreamReader, writer: StreamWriter):
    line = await reader.readline()
    writer.write(line)
    await writer.drain()
    writer.close()
    await writer.wait_closed()
 
 
async def main():
    server = await asyncio.start_server(connected, port=9000)
    await server.serve_forever()
 
 
uvloop.install()      ❶
asyncio.run(main())
```

❶ 安装 uvloop 事件循环。
在前面的清单中，我们调用了 uvloop.install()，它将为我们切换出事件循环。如果我们愿意，我们可以使用以下代码手动执行此操作，而不是调用 install：

```python
loop = uvloop.new_event_loop()
asyncio.set_event_loop(loop)
```

重要的部分是在调用 asyncio.run(main()) 之前调用它。在内部，asyncio.run 调用 get_event_loop，如果不存在，它将创建一个事件循环。如果我们在正确安装 uvloop 之前这样做，我们将得到一个典型的 asyncio 事件循环，而事后安装将没有任何效果。

如果诸如 uvloop 之类的事件循环有助于应用程序的性能特征，你可能需要完成一个基准测试。 Github 上的 uvloop 项目的代码可以在吞吐量和每秒请求数方面运行基准测试。

我们现在已经看到了如何使用现有的事件循环实现而不是默认的事件循环实现。接下来，我们将看到如何完全在 asyncio 的范围之外创建我们自己的事件循环。这将使我们更深入地了解异步事件循环以及协程、任务和期货如何在后台工作。

## 14.5 创建自定义事件循环
asyncio 的一个可能不明显的方面是它在概念上不同于 async/await 语法和协程。协程类定义甚至不在 asyncio 库模块中！

协程和 async/await 语法是独立于执行它们的能力的概念。 Python 带有一个默认的事件循环实现，asyncio，这是我们迄今为止一直用来运行它们的，但我们可以使用任何事件循环实现，甚至是我们自己的。在上一节中，我们看到了如何将 asyncio 事件循环替换为具有可能更好（或至少不同）运行时性能的不同实现。现在，让我们看看如何构建我们自己的可以处理非阻塞套接字的简单事件循环实现。

### 14.5.1 协程和生成器
在 Python 3.5 引入 async 和 await 语法之前，协程和生成器之间的关系是显而易见的。让我们使用装饰器和生成器构建一个简单的协程，它使用旧语法休眠 1 秒来理解这一点。

清单 14.5 基于生成器的协程

```python
import asyncio
 
 
@asyncio.coroutine
def coroutine():
    print('Sleeping!')
    yield from asyncio.sleep(1)
    print('Finished!')
 
 
asyncio.run(coroutine())
```

代替 async 关键字，我们应用 @asyncio.coroutine 装饰器来指定函数是协程函数，而不是 await 关键字，我们使用我们在生成器中熟悉的语法 yield from 。目前， async 和 await 关键字只是围绕这个结构的语法糖。

### 14.5.2 不推荐使用基于生成器的协程
请注意，基于生成器的协程目前计划在 Python 3.10 版中完全删除。你可能会在遗留代码库中遇到它们，但你不应该再以这种风格编写新的异步代码。

那么为什么生成器对单线程并发模型有意义呢？回想一下，协程在遇到阻塞操作时需要暂停执行以允许其他协程运行。生成器在达到屈服点时会暂停执行，从而有效地在中途暂停它们。这意味着如果我们有两个生成器，我们可以交错执行它们。我们让第一个生成器运行直到它到达一个屈服点（或者，在协程语言中，一个等待点），然后我们让第二个生成器运行到它的屈服点，我们重复这个直到两个生成器都耗尽。为了看到这一点，让我们构建一个非常简单的示例，该示例将两个生成器交错，使用一些我们需要用来构建事件循环的后台方法。

清单 14.6 交错生成器执行

```python
from typing import Generator
 
 
def generator(start: int, end: int):
    for i in range(start, end):
        yield i
 
 
one_to_five = generator(1, 5)
five_to_ten = generator(5, 10)
 
 
def run_generator_step(gen: Generator[int, None, None]):   ❶
    try:
        return gen.send(None)
    except StopIteration as si:
        return si.value
 
 
while True:                                                ❷
    one_to_five_result = run_generator_step(one_to_five)
    five_to_ten_result = run_generator_step(five_to_ten)
    print(one_to_five_result)
    print(five_to_ten_result)
 
    if one_to_five_result is None and five_to_ten_result is None:
        break
```

❶ 运行发电机的一个步骤。
❷ 两个生成器的交错执行。
在前面的清单中，我们创建了一个简单的生成器，它从开始整数计数到结束整数，并在此过程中产生值。然后我们创建该生成器的两个实例：一个从 1 计数到 4，另一个从 5 计数到 9。

我们还创建了一个方便的方法 run_generator_step 来处理生成器的运行一个步骤。生成器类有一个 send 方法，它将生成器推进到下一个 yield 语句，运行所有代码到该点。在我们调用 send 之后，我们可以认为生成器暂停，直到我们再次调用 send，让我们在其他生成器中运行代码。 send 方法接受我们想要作为参数发送给生成器的任何值的参数。这里我们什么都没有，所以我们只传入 None 。一旦生成器到达终点，它就会引发 StopIteration 异常。这个异常包含来自生成器的任何返回值，在这里我们返回它。最后，我们创建一个循环并逐步运行每个生成器。这具有同时交错两个生成器的效果，为我们提供以下输出：

```sh
1
5
2
6
3
7
4
8
None
9
None
None
```

想象一下，我们没有屈服于数字，而是屈服于一些缓慢的操作。一旦缓慢的操作完成，我们可以恢复生成器，从我们中断的地方继续，而其他没有暂停的生成器可以运行其他代码。这是事件循环如何工作的核心。我们跟踪在缓慢操作中暂停执行的生成器。然后，任何其他生成器都可以在其他生成器暂停时运行。一旦慢操作完成，我们可以通过再次调用它来唤醒前一个生成器，前进到它的下一个屈服点。

如前所述， async 和 await 只是生成器周围的语法糖。我们可以通过创建一个协程实例并在其上调用 send 来证明这一点。让我们举一个例子，两个协程只打印简单的消息，第三个协程用 await 语句调用另外两个协程。然后我们将使用生成器的 send 方法来查看如何调用我们的协程。

清单 14.7 使用带有发送的协程

```python
async def say_hello():
    print('Hello!')
 
 
async def say_goodbye():
    print('Goodbye!')
 
 
async def meet_and_greet():
    await say_hello()
    await say_goodbye()
 
 
coro = meet_and_greet()
 
coro.send(None)
```

当我们运行上述清单中的代码时，我们将看到以下输出：

```sh
Hello!
Goodbye!
Traceback (most recent call last):
  File "chapter_14/listing_14_7.py", line 16, in <module>
    coro.send(None)
StopIteration
```

在我们的协程上调用 send 会在 meet_and_greet 中运行我们所有的协程。因为在等待结果时我们实际上没有“暂停”，因为所有代码都会立即运行，即使在我们的等待语句中也是如此。

那么我们如何让协程在缓慢的操作中暂停和唤醒呢？为此，让我们定义如何自定义可等待，因此我们可以使用等待语法而不是生成器样式的语法。

### 14.5.3 自定义等待对象
我们如何定义可等待对象，以及它们如何在幕后工作？我们可以通过在一个类上实现 \_\_await\_\_ 方法来定义一个可等待对象，但是我们如何实现这个方法呢？它甚至应该返回什么？

\_\_await\_\_ 方法的唯一要求是它返回一个迭代器，而这个要求本身并不是很有帮助。我们能否让迭代器的概念在事件循环的上下文中有意义？为了理解它是如何工作的，我们将实现我们自己的 asyncio Future 版本，我们将调用 CustomFuture，然后我们将在我们自己的事件循环实现中使用它。

回想一下，Future 是对未来某个时间点可能存在的值的包装，具有两种状态：完整和不完整。想象一下，我们处于一个无限事件循环中，我们想检查一个未来是否用迭代器完成。如果操作完成，我们可以只返回结果，迭代器就完成了。如果没有完成，我们需要某种方式说“我还没有完成；稍后再检查我，”在这种情况下，迭代器可以直接让出！

这就是我们为 CustomFuture 类实现 \_\_await\_\_ 方法的方式。如果结果还没有，我们的迭代器只返回 CustomFuture 本身；如果结果在那里，我们返回结果，迭代器完成。如果没有完成，我们就屈服于自我。如果结果不存在，下次我们尝试推进迭代器时，我们再次运行 \_\_await\_\_ 内的代码。在这个实现中，我们还将实现一个方法来向我们的未来添加一个回调，该回调在设置值时运行。我们稍后在实现事件循环时会用到它。

清单 14.8 一个自定义的未来实现

```python
class CustomFuture:
 
    def __init__(self):
        self._result = None
        self._is_finished = False
        self._done_callback = None
 
    def result(self):
        return self._result
 
    def is_finished(self):
        return self._is_finished
 
    def set_result(self, result):
        self._result = result
        self._is_finished = True
        if self._done_callback:
            self._done_callback(result)
 
    def add_done_callback(self, fn):
        self._done_callback = fn
 
    def __await__(self):
        if not self._is_finished:
            yield self
        return self.result()
```

在前面的清单中，我们定义了 CustomFuture 类，其中定义了 \_\_await\_\_ 以及设置结果、获取结果和添加回调的方法。我们的 \_\_await\_\_ 方法检查未来是否完成。如果是，我们只返回结果，迭代器就完成了。如果它没有完成，我们返回 self，这意味着我们的迭代器将继续无限返回自身，直到值被设置。就生成器而言，这意味着我们可以一直调用 \_\_await\_\_ 直到有人为我们设置值。

让我们看一个小例子，以了解流程在事件循环中的外观。我们将创建一个自定义的 future 并在几次迭代后设置它的值，在每次迭代时调用 \_\_await\_\_。

清单 14.9 循环中的自定义未来

```python
from listing_14_8 import CustomFuture
 
future = CustomFuture()
 
i = 0
 
while True:
    try:
        print('Checking future...')
        gen = future.__await__()
        gen.send(None)
        print('Future is not done...')
        if i == 1:
            print('Setting future value...')
            future.set_result('Finished!')
        i = i + 1
    except StopIteration as si:
        print(f'Value is: {si.value}')
        break
```

在前面的清单中，我们创建了一个自定义的 future 和一个调用 await 方法然后尝试推进迭代器的循环。如果 future 完成，将抛出一个 StopIteration 异常和 future 的结果。否则，我们的迭代器将只返回未来，然后我们继续循环的下一次迭代。在我们的示例中，我们在几次迭代后设置值，为我们提供以下输出：

```sh
Checking future...
Future is not done...
Checking future...
Future is not done...
Setting future value...
Checking future...
Value is: Finished!
```

这个例子只是为了加强思考等待的方式，我们不会在现实生活中编写这样的代码，因为我们通常想要其他东西来设置我们未来的结果。接下来，让我们扩展它以对套接字和选择器模块做一些更有用的事情。

### 14.5.4 使用带有期货的套接字
在第 3 章中，我们了解了一些选择器模块，它允许我们注册回调以在套接字事件（例如新连接或准备读取的数据）发生时运行。现在我们将通过使用我们自定义的 future 类与选择器进行交互来扩展这方面的知识，当套接字事件发生时在 future 上设置结果。

回想一下，选择器让我们注册回调以在套接字上发生事件（例如读取或写入）时运行。这个概念非常适合我们构建的未来。当在套接字上发生读取时，我们可以将 set_result 方法注册为回调。当我们想要异步等待来自套接字的结果时，我们创建一个新的未来，向该套接字的选择器模块注册该未来的 set_result 方法，然后返回未来。 We can then await it, and we’ll get the result when the selector calls the callback for us.

为了看到这一点，让我们构建一个应用程序来监听来自非阻塞套接字的连接。一旦我们获得连接，我们将返回它并让应用程序终止。

清单 14.10 带有自定义期货的套接字

```python
import functools
import selectors
import socket
from listing_14_8 import CustomFuture
from selectors import BaseSelector
 
 
def accept_connection(future: CustomFuture, connection: socket):   ❶
    print(f'We got a connection from {connection}!')
    future.set_result(connection)
 
 
async def sock_accept(sel: BaseSelector, sock) -> socket:          ❷
    print('Registering socket to listen for connections')
    future = CustomFuture()
    sel.register(sock, selectors.EVENT_READ, functools.partial(accept_connection, future))
    print('Pausing to listen for connections...')
    connection: socket = await future
    return connection
 
 
async def main(sel: BaseSelector):
    sock = socket.socket()
    sock.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
 
    sock.bind(('127.0.0.1', 8000))
    sock.listen()
    sock.setblocking(False)
 
    print('Waiting for socket connection!')
    connection = await sock_accept(sel, sock)                      ❸
    print(f'Got a connection {connection}!')
selector = selectors.DefaultSelector()
 
coro = main(selector)
 
while True:                                                        ❹
    try:
        state = coro.send(None)
 
        events = selector.select()
 
        for key, mask in events:
            print('Processing selector events...')
            callback = key.data
            callback(key.fileobj)
    except StopIteration as si:
        print('Application finished!')
        break
```

❶ 将连接套接字设置在将来客户端连接时。
❷ 向选择器注册accept_connection函数并暂停等待客户端连接。
❸ 等待客户端连接。
❹ 永远循环，在主协程上调用 send。每次发生选择器事件时，运行注册的回调。
在前面的清单中，我们首先定义了一个 accept_connection 函数。这个函数接受一个 CustomFuture 和一个客户端套接字。我们打印一条消息，表明我们有一个套接字，然后将该套接字设置为未来的结果。然后我们定义 sock_accept；这个函数接受一个服务器套接字和一个选择器，并将accept_connection（绑定到一个CustomFuture）注册为来自服务器套接字的读取事件的回调。然后我们等待未来，暂停直到我们建立连接，然后返回它。

然后我们定义一个主协程函数。在这个函数中，我们创建一个服务器套接字，然后等待 sock_accept 协程，直到我们收到一个连接，记录一条消息并在我们这样做后终止。有了这个，我们可以构建一个最小可行的事件循环。我们创建一个主协程函数的实例，传入一个选择器，然后永远循环。在我们的循环中，我们首先调用 send 将我们的主协程推进到它的第一个 await 语句，然后我们调用 selector.select，它将阻塞直到客户端连接。然后我们调用任何注册的回调；在我们的例子中，这将始终是 accept_connection。一旦有人连接，我们将再次调用 send ，这将再次推进所有协程并让我们的应用程序完成。如果你运行以下代码并通过 Telnet 连接，你应该会看到类似于以下内容的输出：

```python
Waiting for socket connection!
Registering socket to listen for connections
Pausing to listen for connections...
Processing selector events...
We got a connection from <socket.socket fd=4, family=AddressFamily.AF_INET, type=SocketKind.SOCK_STREAM, proto=0, laddr=('127.0.0.1', 8000)>!
Got a connection <socket.socket fd=4, family=AddressFamily.AF_INET, type=SocketKind.SOCK_STREAM, proto=0, laddr=('127.0.0.1', 8000)>!
Application finished!
```

我们现在已经构建了一个基本的异步应用程序，只使用 async 和 await 关键字，没有任何 asyncio！最后的 while 循环是一个简单的事件循环，并演示了 asyncio 事件循环如何工作的关键概念。当然，如果没有创建任务的能力，我们不能同时做太多事情。

### 14.5.5 任务实现
任务是未来和协程的组合。当它包装的协程完成时，任务的未来就完成了。通过继承，我们可以通过继承我们的 CustomFuture 类并编写一个接受协程的构造函数来包装未来的协程，但我们仍然需要一种方法来运行该协程。我们可以通过构建一个我们将调用 step 的方法来做到这一点，该方法将调用协程的 send 方法并跟踪结果，每次调用有效地运行协程的一个 step。

在实现此方法时，我们需要记住的一件事是 send 也可能返回其他期货。为了处理这个问题，我们需要使用任何发送返回的期货的 add_done_callback 方法。我们将注册一个回调，该回调将在任务的协程上调用 send，并在未来完成时使用结果值。

清单 14.11 一个任务实现

```python
from chapter_14.listing_14_8 import CustomFuture
 
 
class CustomTask(CustomFuture):
 
    def __init__(self, coro, loop):
        super(CustomTask, self).__init__()
        self._coro = coro
        self._loop = loop
        self._current_result = None
        self._task_state = None
        loop.register_task(self)                              ❶
 
    def step(self):                                           ❷
        try:
            if self._task_state is None:
                self._task_state = self._coro.send(None)
            if isinstance(self._task_state, CustomFuture):    ❸
                self._task_state.add_done_callback(self._future_done)
        except StopIteration as si:
            self.set_result(si.value)
 
    def _future_done(self, result):                           ❹
        self._current_result = result
        try:
            self._task_state = self._coro.send(self._current_result)
        except StopIteration as si:
            self.set_result(si.value)
```

❶ 向事件循环注册任务。
❷ 运行协程的一步。
❸ 如果协程产生一个未来，添加一个 done 回调。
❹ 一旦future完成，将结果发送给协程。
在上面的清单中，我们继承了 CustomFuture 并创建了一个接受协程和事件循环的构造函数，通过调用 loop.register_task 将任务注册到循环中。然后，在我们的 step 方法中，我们在协程上调用 send，如果协程产生一个 CustomFuture，我们添加一个 done 回调。在这种情况下，我们的 done 回调将获取未来的结果并将其发送到我们包装的协程，并在未来完成时推进它。

### 14.5.6 实现事件循环
我们现在知道如何运行协程，并创建了期货和任务的实现，为我们提供了构建事件循环所需的所有构建块。我们的事件 API 需要什么样的外观才能构建异步套接字应用程序？我们需要一些具有不同目的的方法：

我们需要一个方法来接受一个主入口协程，就像 asyncio.run 一样。
我们需要接受连接、接收数据和关闭套接字的方法。这些方法将使用选择器注册和取消注册套接字。
我们需要一个方法来注册一个 CustomTask；这只是我们之前在 CustomTask 构造函数中使用的方法的一个实现。
首先说一下我们的主要切入点；我们将调用此方法运行。这是我们事件循环的动力源。此方法将获取一个主入口点协程并在其上调用 send，在无限循环中跟踪生成器的结果。如果主协程产生了一个future，我们将添加一个done回调来跟踪future完成后的结果。完成此操作后，我们将运行任何已注册任务的 step 方法，然后调用选择器等待任何套接字事件触发。一旦它们运行，我们将运行相关的回调并触发循环的另一次迭代。如果在任何时候我们的主协程抛出一个 StopIteration 异常，我们就知道我们的应用程序已经完成，我们可以退出并返回异常中的值。

接下来，我们需要协程方法来接受套接字连接并从客户端套接字接收数据。我们这里的策略是创建一个回调将设置结果的 CustomFuture 实例，将此回调注册到选择器以触发读取事件。然后，我们将等待这个未来。

最后，我们需要一个方法来向事件循环注册任务。此方法将简单地接受一个任务并将其添加到列表中。然后，在事件循环的每次迭代中，我们将对我们在事件循环中注册的任何任务调用 step，如果它们准备好则推进它们。实现所有这些将产生一个最小的可行事件循环。

清单 14.12 一个事件循环实现

```python
import functools
import selectors
from typing import List
from chapter_14.listing_14_11 import CustomTask
from chapter_14.listing_14_8 import CustomFuture
 
 
class EventLoop:
    _tasks_to_run: List[CustomTask] = []
    def __init__(self):
        self.selector = selectors.DefaultSelector()
        self.current_result = None
 
    def _register_socket_to_read(self, sock, callback):        ❶
        future = CustomFuture()
        try:
            self.selector.get_key(sock)
        except KeyError:
            sock.setblocking(False)
            self.selector.register(sock, selectors.EVENT_READ, functools.partial(callback, future))
        else:
            self.selector.modify(sock, selectors.EVENT_READ, functools.partial(callback, future))
        return future
 
    def _set_current_result(self, result):
        self.current_result = result
 
    async def sock_recv(self, sock):                           ❷
        print('Registering socket to listen for data...')
        return await self._register_socket_to_read(sock, self.recieved_data)
 
    async def sock_accept(self, sock):                         ❸
        print('Registering socket to accept connections...')
        return await self._register_socket_to_read(sock, self.accept_connection)
 
    def sock_close(self, sock):
        self.selector.unregister(sock)
        sock.close()
 
    def register_task(self, task):                             ❹
        self._tasks_to_run.append(task)
 
    def recieved_data(self, future, sock):
        data = sock.recv(1024)
        future.set_result(data)
 
    def accept_connection(self, future, sock):
        result = sock.accept()
        future.set_result(result)
 
    def run(self, coro):                                       ❺
        self.current_result = coro.send(None)
 
        while True:
            try:
                if isinstance(self.current_result, CustomFuture):
                    self.current_result.add_done_callback(self._set_current_result)
                    if self.current_result.result() is not None:
                        self.current_result = coro.send(self.current_result.result())
                else:
                    self.current_result = coro.send(self.current_result)
            except StopIteration as si:
                return si.value
 
            for task in self._tasks_to_run:
                task.step()
 
            self._tasks_to_run = [task for task in self._tasks_to_run if not task.is_finished()]
 
            events = self.selector.select()
            print('Selector has an event, processing...')
            for key, mask in events:
                callback = key.data
                callback(key.fileobj)
```

❶ 使用选择器注册一个套接字以用于读取事件。
❷ 注册一个套接字来接收来自客户端的数据。
❸ 注册一个套接字以接受来自客户端的连接。
❹ 在事件循环中注册一个任务。
❺ 运行一个主协程直到它完成，在每次迭代中执行所有待处理的任务。
我们首先定义一个 _register_socket_to_read 便捷方法。此方法接收一个套接字和一个回调，如果套接字尚未注册，则将它们注册到选择器。如果套接字已注册，我们替换回调。我们回调的第一个参数需要是一个未来，在这个方法中我们创建一个新参数并将其部分应用于回调。最后，我们返回绑定到回调的未来，这意味着我们方法的调用者现在可以等待它并暂停执行，直到回调完成。

然后我们定义协程方法来接收套接字数据和接受新的客户端连接，分别是 sock_recv 和 sock_accept。这些方法调用我们刚刚定义的 _register_socket_to_read 便捷方法，传入处理数据和新连接可用的回调（这些方法只是将这些数据设置为未来）。

最后，我们构建我们的 run 方法。这个方法接受我们的主入口点协程并在它上面调用send，将它推进到它的第一个暂停点并存储来自send的结果。然后我们开始一个无限循环，首先检查主协程的当前结果是否是 CustomFuture；如果是，我们注册一个回调来存储结果，如果需要，我们可以将其发送回主协程。如果结果不是 CustomFuture，我们只需将其发送给协程。一旦我们控制了主协程的流程，我们就可以通过调用 step 来运行在事件循环中注册的任何任务。一旦我们运行了我们的任务，我们就会从我们的任务列表中删除任何已完成的任务。

最后，我们调用 selector.select，阻塞直到在我们注册的套接字上触发任何事件。一旦我们有一个套接字事件或一组事件，我们循环它们，调用我们在 _register_socket_to_read 中为该套接字注册的回调。在我们的实现中，任何套接字事件都会触发事件循环的迭代。我们现在已经实现了 EventLoop 类，并且准备好创建我们的第一个没有 asyncio 的异步应用程序！

### 14.5.7 使用自定义事件循环实现服务器
现在我们有了一个事件循环，我们将构建一个非常简单的服务器应用程序来记录我们从连接的客户端收到的消息。我们将创建一个服务器套接字并编写一个协程函数来在无限循环中监听连接。建立连接后，我们将创建一个任务来从该客户端读取数据，直到它们断开连接。这看起来与我们在第 3 章中构建的非常相似，主要区别在于这里我们使用自己的事件循环而不是 asyncio 的。

清单 14.13 实现一个服务器

```python
import socket
 
from chapter_14.listing_14_11 import CustomTask
from chapter_14.listing_14_12 import EventLoop
 
 
async def read_from_client(conn, loop: EventLoop):          ❶
    print(f'Reading data from client {conn}')
    try:
        while data := await loop.sock_recv(conn):
            print(f'Got {data} from client!')
    finally:
        loop.sock_close(conn)
 
 
async def listen_for_connections(sock, loop: EventLoop):    ❷
    while True:
        print('Waiting for connection...')
        conn, addr = await loop.sock_accept(sock)
        CustomTask(read_from_client(conn, loop), loop)
        print(f'I got a new connection from {sock}!')
 
 
async def main(loop: EventLoop):
    server_socket = socket.socket()
    server_socket.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
 
    server_socket.bind(('127.0.0.1', 8000))
    server_socket.listen()
    server_socket.setblocking(False)
 
    await listen_for_connections(server_socket, loop)
 
 
event_loop = EventLoop()                                    ❸
event_loop.run(main(event_loop))
```

❶ 从客户端读取数据，并记录下来。
❷ 监听客户端连接，创建一个任务在客户端连接时读取数据。
❸ 创建一个事件循环实例，并运行主协程。
在前面的清单中，我们首先定义了一个协程函数，以循环方式从客户端读取数据，并在获得结果时打印结果。我们还定义了一个协程函数来在无限循环中监听来自服务器套接字的客户端连接，创建一个 CustomTask 来同时监听来自该客户端的数据。在我们的主协程中，我们创建一个服务器套接字并调用我们的 listen_for_connections 协程函数。然后，我们创建一个事件循环实现的实例，将主协程传递给 run 方法。

运行此代码，你应该能够通过 Telnet 同时连接多个客户端并向服务器发送消息。例如，两个客户端连接并发送一些测试消息可能如下所示：

```python
Waiting for connection...
Registering socket to accept connections...
Selector has an event, processing...
I got a new connection from <socket.socket fd=4, family=AddressFamily.AF_INET, type=SocketKind.SOCK_STREAM, proto=0, laddr=('127.0.0.1', 8000)>!
Waiting for connection...
Registering socket to accept connections...
Reading data from client <socket.socket fd=7, family=AddressFamily.AF_INET, type=SocketKind.SOCK_STREAM, proto=0, laddr=('127.0.0.1', 8000), raddr=('127.0.0.1', 58641)>
Registering socket to listen for data...
Selector has an event, processing...
Got b'test from client one!\r\n' from client!
Registering socket to listen for data...
Selector has an event, processing...
I got a new connection from <socket.socket fd=4, family=AddressFamily.AF_INET, type=SocketKind.SOCK_STREAM, proto=0, laddr=('127.0.0.1', 8000)>!
Waiting for connection...
Registering socket to accept connections...
Reading data from client <socket.socket fd=8, family=AddressFamily.AF_INET, type=SocketKind.SOCK_STREAM, proto=0, laddr=('127.0.0.1', 8000), raddr=('127.0.0.1', 58645)>
Registering socket to listen for data...
Selector has an event, processing...
Got b'test from client two!\r\n' from client!
Registering socket to listen for data...
```

在上面的输出中，一个客户端连接，触发选择器从其在 loop.sock_accept 上的挂起点恢复 listen_for_connections。当我们为 read_from_client 创建任务时，这也会向选择器注册客户端连接。第一个客户端发送消息“来自客户端的测试！”，这再次触发选择器触发任何回调。在这种情况下，我们推进 read_from_client 任务，将客户端的消息输出到控制台。然后，第二个客户端连接，同样的过程再次发生。

虽然这不是一个值得生产的事件循环（我们并没有真正正确地处理异常，并且我们只允许套接字事件触发事件循环迭代，以及其他缺点），但这应该给你一个想法关于 Python 中事件循环和异步编程的内部工作原理。一个练习是采用这里的概念并构建一个生产就绪的事件循环。也许你可以创建下一代异步 Python 框架。

## 概括

- 我们可以检查可调用参数是否是协程，以创建处理协程和常规函数的 API。
- 当你有需要在协程之间传递的状态时使用上下文局部变量，但你希望此状态独立于你的参数。
- asyncio 的 sleep 协程可用于强制事件循环的迭代。当我们需要触发事件循环来做某事但没有自然等待点时，这很有用。
- asyncio 只是 Python 的事件循环的标准实现。存在其他实现，例如 uvloop，我们可以根据需要更改它们，并且仍然使用 async 和 await 语法。如果我们想设计具有不同特征的东西以更好地满足我们的需求，我们也可以创建自己的事件循环。