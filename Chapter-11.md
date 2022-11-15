# 同步

本章涵盖

- 单线程并发问题
- 使用锁保护临界区
- 使用信号量限制并发
- 使用事件通知任务
- 使用条件通知任务并获取资源

当我们使用多线程和多进程编写应用程序时，我们需要担心使用非原子操作时的竞争条件。像同时增加一个整数这样简单的事情可能会导致微妙的、难以重现的错误。然而，当我们使用 asyncio 时，我们总是使用单线程（除非我们与多线程和多处理进行互操作），这是否意味着我们不需要担心竞争条件？事实证明，事情并不那么简单。

虽然 asyncio 的单线程特性消除了多线程或多处理应用程序中可能出现的某些并发错误，但它们并没有完全消除。虽然你可能不需要经常通过 asyncio 使用同步，但在某些情况下我们仍然需要这些构造。 asyncio 的同步原语可以帮助我们防止单线程并发模型特有的错误。

同步原语不仅限于防止并发错误，还有其他用途。例如，我们可能正在使用一个 API，它允许我们根据与供应商的合同同时发出几个请求，或者可能存在我们担心请求过载的 API。我们可能还有一个工作流程，其中有几个工作人员需要在新数据可用时得到通知。

在本章中，我们将看一些示例，在这些示例中，我们可以在 asyncio 代码中引入竞争条件，并学习如何使用锁和其他并发原语来解决它们。我们还将学习如何使用信号量来限制并发和控制对共享资源的访问，例如数据库连接池。最后，我们将研究可用于在发生某些事情时通知任务并在发生这种情况时获得对共享资源的访问权的事件和条件。

## 11.1 理解单线程并发错误
在前面关于多处理和多线程的章节中，回想一下，当我们处理在不同进程和线程之间共享的数据时，我们不得不担心竞争条件。这是因为一个线程或进程在被不同的线程或进程修改时可能会读取数据，从而导致状态不一致，从而导致数据损坏。

这种损坏部分是由于某些操作是非原子的，这意味着虽然它们看起来像一个操作，但它们在后台包含多个单独的操作。我们在第 6 章中给出的例子是处理整数变量的递增；首先，我们读取当前值，然后将其递增，然后将其重新分配回变量。这为其他线程和进程提供了充足的机会来获取处于不一致状态的数据。

在单线程并发模型中，我们避免了由非原子操作引起的竞态条件。在 asyncio 的单线程模型中，我们只有一个线程在任何给定时间执行一行 Python 代码。这意味着即使一个操作是非原子的，我们也总是会在没有其他协程读取不一致的状态信息的情况下运行它直到完成。

为了向我们自己证明这一点，让我们尝试重新创建我们在第 7 章中看到的竞争条件，其中多个线程试图实现一个共享计数器。我们将有多个任务，而不是让多个线程修改变量。我们将重复这 1000 次并断言我们得到了正确的值。

清单 11.1 尝试创建竞争条件

```python
import asyncio
 
counter: int = 0
 
 
async def increment():
    global counter
    await asyncio.sleep(0.01)
    counter = counter + 1
async def main():
    global counter
    for _ in range(1000):
        tasks = [asyncio.create_task(increment()) for _ in range(100)]
        await asyncio.gather(*tasks)
        print(f'Counter is {counter}')
        assert counter == 100
        counter = 0
 
 
asyncio.run(main())
```

在前面的清单中，我们创建了一个增量协程函数，它向全局计数器加一，添加 1 毫秒的延迟来模拟慢速操作。在我们的主协程中，我们创建了 100 个任务来递增计数器，然后将它们全部与gather 并发运行。然后我们断言我们的计数器是期望值，因为我们运行了 100 个增量任务，所以它应该总是 100。运行这个，你应该看到我们得到的值总是 100，即使递增一个整数是非原子的。如果我们运行多个线程而不是协程，我们应该会看到我们的断言在执行的某个时刻失败。

这是否意味着通过单线程并发模型我们已经找到了一种完全避免竞争条件的方法？不幸的是，情况并非如此。虽然我们避免了单个非原子操作可能导致错误的竞争条件，但我们仍然存在以错误顺序执行的多个操作可能导致问题的问题。为了看到这一点，让我们在 asyncio 非原子的眼中增加一个整数。

为此，我们将复制当我们增加一个全局计数器时发生的事情。我们读取全局值，将其递增，然后将其写回。基本思想是如果其他代码在我们的协程在等待时暂停时修改状态，一旦等待完成，我们可能会处于不一致的状态。

清单 11.2 单线程竞争条件

```python
import asyncio
 
counter: int = 0
 
 
async def increment():
    global counter
    temp_counter = counter
    temp_counter = temp_counter + 1
    await asyncio.sleep(0.01)
    counter = temp_counter
 
 
async def main():
    global counter
    for _ in range(1000):
        tasks = [asyncio.create_task(increment()) for _ in range(100)]
        await asyncio.gather(*tasks)
        print(f'Counter is {counter}')
        assert counter == 100
        counter = 0
 
 
asyncio.run(main())
```

我们的增量协程不是直接递增计数器，而是首先将其读入一个临时变量，然后将临时计数器加一。然后我们等待 asyncio.sleep 来模拟一个缓慢的操作，暂停我们的协程，然后我们才将它重新分配回全局计数器变量。运行它，你应该会立即看到此代码失败并出现断言错误，并且我们的计数器只会设置为 1！每个协程首先读取计数器值，即 0，将其存储到临时值，然后进入睡眠状态。由于我们是单线程的，每次对临时变量的读取都是按顺序运行的，这意味着每个协程都将 counter 的值存储为 0 并将其递增到 1。然后，一旦 sleep 完成，每个协程都会将 counter 的值设置为1，这意味着尽管运行了 100 个协程来增加我们的计数器，但我们的计数器永远只有 1。请注意，如果你删除 await 表达式，事情将以正确的顺序运行，因为在我们暂停时没有机会修改应用程序状态在等待点。

诚然，这是一个简单化且有些不切实际的例子。为了更好地了解何时会发生这种情况，让我们创建一个稍微复杂一点的竞态条件。想象一下，我们正在实现一个向连接的用户发送消息的服务器。在此服务器中，我们将用户名字典保存到可用于向这些用户发送消息的套接字。当用户断开连接时，将运行一个回调，将用户从字典中删除并关闭他们的套接字。由于我们在断开连接时关闭套接字，因此尝试发送任何其他消息都会失败并出现异常。如果在我们发送消息的过程中用户断开连接会发生什么？假设我们想要的行为是让所有用户在我们开始发送消息时都收到一条消息。

为了测试这一点，让我们实现一个模拟套接字。这个模拟套接字将有一个发送协程和一个关闭方法。我们的发送协程将模拟通过慢速网络发送的消息。这个协程还会检查一个标志，看看我们是否关闭了套接字，如果我们关闭了，就会抛出异常。

然后，我们将创建一个包含一些已连接用户的字典，并为每个用户创建模拟套接字。我们将向每个用户发送消息，并在发送消息时手动触发单个用户的断开连接以查看会发生什么。

清单 11.3 带有字典的竞争条件

```python
import asyncio
 
 
class MockSocket:
    def __init__(self):
        self.socket_closed = False
 
    async def send(self, msg: str):                ❶
        if self.socket_closed:
            raise Exception('Socket is closed!')
        print(f'Sending: {msg}')
        await asyncio.sleep(1)
        print(f'Sent: {msg}')
 
    def close(self):
        self.socket_closed = True
 
 
user_names_to_sockets = {'John': MockSocket(),
                         'Terry': MockSocket(),
                         'Graham': MockSocket(),
                         'Eric': MockSocket()}
 
 
async def user_disconnect(username: str):          ❷
    print(f'{username} disconnected!')
    socket = user_names_to_sockets.pop(username)
    socket.close()
 
 
async def message_all_users():                     ❸
    print('Creating message tasks')
    messages = [socket.send(f'Hello {user}')
                for user, socket in
                user_names_to_sockets.items()]
    await asyncio.gather(*messages)
 
 
async def main():
    await asyncio.gather(message_all_users(), user_disconnect('Eric'))
 
 
asyncio.run(main())
```

❶ 模拟向客户端缓慢发送消息。
❷ 断开用户连接并将其从应用程序内存中删除。
❸ 同时向所有用户发送消息。
如果你运行此代码，你将看到应用程序崩溃并显示以下输出：

```sh
Creating message tasks
Eric disconnected!
Sending: Hello John
Sending: Hello Terry
Sending: Hello Graham
Traceback (most recent call last):
  File 'chapter_11/listing_11_3.py', line 45, in <module>
    asyncio.run(main())
  File "asyncio/runners.py", line 44, in run
    return loop.run_until_complete(main)
  File "python3.9/asyncio/base_events.py", line 642, in run_until_complete
    return future.result()
  File 'chapter_11/listing_11_3.py', line 42, in main
    await asyncio.gather(message_all_users(), user_disconnect('Eric'))
  File 'chapter_11/listing_11_3.py', line 37, in message_all_users
    await asyncio.gather(*messages)
  File 'chapter_11/listing_11_3.py', line 11, in send
    raise Exception('Socket is closed!')
Exception: Socket is closed!
```

在这个例子中，我们首先创建消息任务，然后我们等待，暂停我们的 message_all_users 协程。这使 user_disconnect('Eric') 有机会运行，这将关闭 Eric 的套接字并将其从 user_names_to_sockets 字典中删除。完成后，message_all_users 恢复；我们开始发送消息。由于 Eric 的套接字已关闭，我们看到了一个异常，他不会收到我们打算发送的消息。请注意，我们还修改了 user_names_to_sockets 字典。如果我们以某种方式需要使用这本字典并依赖 Eric 仍在其中，我们可能会遇到异常或其他错误。

这些是你在单线程并发模型中容易看到的错误类型。你使用 await 达到了暂停点，另一个协程运行并修改了一些共享状态，一旦它以不希望的方式恢复，就将其更改为第一个协程。多线程并发错误和单线程并发错误之间的主要区别在于，在多线程应用程序中，在你修改可变状态的任何地方都可能出现竞争条件。在单线程并发模型中，你需要在等待点期间修改可变状态。现在我们了解了单线程模型中并发错误的类型，让我们看看如何通过使用异步锁来避免它们。

## 11.2 锁
异步锁的操作类似于多处理和多线程模块中的锁。我们获得一个锁，在临界区内部工作，当我们完成后，我们释放锁，让其他相关方获得它。主要区别在于异步锁是可等待对象，当它们被阻塞时会暂停协程执行。这意味着当协程被阻塞等待获取锁时，其他代码可以运行。此外，asyncio 锁也是异步上下文管理器，使用它们的首选方式是使用 async with 语法。

为了熟悉锁的工作原理，让我们看一个简单的例子，一个锁在两个协程之间共享。我们将获得锁，这将阻止其他协程在关键部分运行代码，直到有人释放它。

清单 11.4 使用异步锁

```python
import asyncio
from asyncio import Lock
from util import delay
 
 
async def a(lock: Lock):
    print('Coroutine a waiting to acquire the lock')
    async with lock:
        print('Coroutine a is in the critical section')
        await delay(2)
    print('Coroutine a released the lock')
async def b(lock: Lock):
    print('Coroutine b waiting to acquire the lock')
    async with lock:
        print('Coroutine b is in the critical section')
        await delay(2)
    print('Coroutine b released the lock')
 
 
async def main():
    lock = Lock()
    await asyncio.gather(a(lock), b(lock))
 
 
asyncio.run(main())
```

当我们运行上面的清单时，我们会看到协程 a 先获取锁，让协程 b 等待直到 a 释放锁。一旦 a 释放了锁， b 就可以在临界区执行它的工作，给我们以下输出：

```sh
Coroutine a waiting to acquire the lock
Coroutine a is in the critical section
sleeping for 2 second(s)
Coroutine b waiting to acquire the lock
finished sleeping for 2 second(s)
Coroutine a released the lock
Coroutine b is in the critical section
sleeping for 2 second(s)
finished sleeping for 2 second(s)
Coroutine b released the lock
```

这里我们使用了 async with 语法。如果我们愿意，我们可以像这样在锁上使用获取和释放方法：

```python
await lock.acquire()
try:
    print('In critical section')
finally:
    lock.release()
```

也就是说，最好的做法是尽可能使用带语法的异步。

需要注意的一件重要事情是我们在主协程内部创建了锁。由于锁在我们创建的协程之间全局共享，我们可能会尝试将其设为全局变量以避免每次都像这样传递它：

```python
lock = Lock()
 
# coroutine definitions
 
async def main():
    await asyncio.gather(a(), b())
```

如果我们这样做，我们会很快看到一个崩溃，并报告多个事件循环的错误：

```python
Task <Task pending name='Task-3' coro=<b()> got Future <Future pending> attached to a different loop
```

当我们所做的只是移动我们的锁定义时，为什么会发生这种情况？这是 asyncio 库的一个令人困惑的怪癖，并且不仅仅是锁所独有的。 asyncio 中的大多数对象都提供了一个可选的循环参数，可让你指定要在其中运行的特定事件循环。当未提供此参数时，asyncio 会尝试获取当前正在运行的事件循环，但如果没有，它会创建一个新的。在上面的例子中，创建一个 Lock 会创建一个新的事件循环，因为当我们的脚本第一次运行时，我们还没有创建一个。然后， asyncio.run(main()) 创建了第二个事件循环，当我们尝试使用我们的锁时，我们将这两个独立的事件循环混合在一起，这会导致崩溃。

这种行为非常棘手，以至于在 Python 3.10 中，事件循环参数将被删除，这种令人困惑的行为将消失，但在此之前，在使用全局 asyncio 变量时需要仔细考虑这些情况。

现在我们了解了基础知识，让我们看看如何使用锁来解决清单 11.3 中的错误，在该错误中，我们试图向我们过早关闭其套接字的用户发送消息。解决这个问题的想法是在两个地方使用锁：首先，当用户断开连接时，其次，当我们向用户发送消息时。这样，如果在我们发送消息时发生断开连接，我们将等到它们都完成后再关闭任何套接字。

清单 11.5 使用锁来避免竞争条件

```python
import asyncio
from asyncio import Lock
 
 
class MockSocket:
    def __init__(self):
        self.socket_closed = False
 
    async def send(self, msg: str):
        if self.socket_closed:
            raise Exception('Socket is closed!')
        print(f'Sending: {msg}')
        await asyncio.sleep(1)
        print(f'Sent: {msg}')
 
    def close(self):
        self.socket_closed = True
 
 
user_names_to_sockets = {'John': MockSocket(),
                         'Terry': MockSocket(),
                         'Graham': MockSocket(),
                         'Eric': MockSocket()}
 
 
async def user_disconnect(username: str, user_lock: Lock):
    print(f'{username} disconnected!')
    async with user_lock:                                ❶
        print(f'Removing {username} from dictionary')
        socket = user_names_to_sockets.pop(username)
        socket.close()
 
 
async def message_all_users(user_lock: Lock):
    print('Creating message tasks')
    async with user_lock:                                ❷
        messages = [socket.send(f'Hello {user}')
                    for user, socket in
                    user_names_to_sockets.items()]
        await asyncio.gather(*messages)
 
 
async def main():
    user_lock = Lock()
    await asyncio.gather(message_all_users(user_lock),
                         user_disconnect('Eric', user_lock))
 
 
asyncio.run(main())
```

❶ 在移除用户并关闭插座之前获取锁。
❷ 发送前获取锁。
当我们运行以下清单时，我们不会再看到任何崩溃，我们将获得以下输出：

```sh
Creating message tasks
Eric disconnected!
Sending: Hello John
Sending: Hello Terry
Sending: Hello Graham
Sending: Hello Eric
Sent: Hello John
Sent: Hello Terry
Sent: Hello Graham
Sent: Hello Eric
Removing Eric from dictionary
```

我们首先获取锁并创建消息任务。发生这种情况时，Eric 断开连接，并且断开连接中的代码尝试获取锁。由于 message_all_users 仍然持有锁，我们需要等待它完成，然后再以断开连接的方式运行代码。这可以让所有消息在关闭套接字之前完成发送，从而防止我们的错误。

你可能不需要经常在 asyncio 代码中使用锁，因为它的单线程特性避免了许多并发问题。即使发生竞争条件，有时你也可以重构代码，以便在协程挂起时不修改状态（例如，通过使用不可变对象）。当你不能以这种方式重构时，锁可以强制修改以所需的同步顺序发生。现在我们了解了使用锁避免并发错误的概念，让我们看看如何使用同步在我们的 asyncio 应用程序中实现新功能。

## 11.3 用信号量限制并发
我们的应用程序需要使用的资源通常是有限的。我们可以与数据库同时使用的连接数量可能有限；我们可能不想过载的 CPU 数量有限；或者，根据我们当前的订阅定价，我们可能正在使用仅允许少数并发请求的 API。我们也可以使用我们自己的内部 API，并且可能会担心负载过重，从而有效地对我们自己发起分布式拒绝服务攻击。

信号量是一种可以在这些情况下帮助我们的结构。信号量的作用很像锁，我们可以获取它也可以释放它，主要区别在于我们可以多次获取它，直到我们指定的限制。在内部，信号量跟踪这个限制；每次我们获取信号量时，我们都会减少限制，每次我们释放信号量时，我们都会增加它。如果计数达到零，任何进一步获取信号量的尝试都将阻塞，直到其他人调用 release 并增加计数。为了与我们刚刚学习的锁相提并论，你可以将锁视为限制为 1 的信号量的特殊情况。

为了查看信号量的作用，让我们构建一个简单的示例，我们只希望同时运行两个任务，但我们总共有四个任务要运行。为此，我们将创建一个限制为 2 的信号量，并在我们的协程中获取它。

清单 11.6 使用信号量

```python
import asyncio
from asyncio import Semaphore
 
 
async def operation(semaphore: Semaphore):
    print('Waiting to acquire semaphore...')
    async with semaphore:
        print('Semaphore acquired!')
        await asyncio.sleep(2)
    print('Semaphore released!')
 
 
async def main():
    semaphore = Semaphore(2)
    await asyncio.gather(*[operation(semaphore) for _ in range(4)])
 
asyncio.run(main())
```

在我们的主协程中，我们创建了一个限制为 2 的信号量，表明我们可以在额外的获取尝试开始阻塞之前获取它两次。然后，我们创建了四个并发的操作调用——这个协程使用异步块获取信号量，并模拟一些带有睡眠的阻塞工作。当我们运行它时，我们将看到以下输出：

```sh
Waiting to acquire semaphore...
Semaphore acquired!
Waiting to acquire semaphore...
Semaphore acquired!
Waiting to acquire semaphore...
Waiting to acquire semaphore...
Semaphore released!
Semaphore released!
Semaphore acquired!
Semaphore acquired!
Semaphore released!
Semaphore released!
```

由于我们的信号量在阻塞之前只允许两次获取，所以我们的前两个任务成功获取锁，而我们的其他两个任务等待前两个任务释放信号量。一旦前两个任务中的工作完成并且我们释放信号量，我们的其他两个任务就可以获取信号量并开始工作。

让我们采用这种模式并将其应用于现实世界的用例。假设你正在为一家斗志旺盛、资金拮据的初创公司工作，而你刚刚与第三方 REST API 供应商合作。他们的合同对于无限制的查询来说特别昂贵，但他们提供了一个只允许 10 个并发请求的计划，这对预算更友好。如果你同时发出超过 10 个请求，他们的 API 将返回状态码 429（请求过多）。如果收到 429，你可以发送一组请求并重试，但这效率低下，并且会给供应商的服务器带来额外的负载，这可能不会让他们的站点可靠性工程师满意。更好的方法是创建一个限制为 10 的信号量，然后在你发出 API 请求时获取它。在发出请求时使用信号量将确保你在任何给定时间都只有 10 个正在运行的请求。

让我们看看如何使用 aiohttp 库来做到这一点。我们将向示例 API 发出 1,000 个请求，但使用信号量将并发请求总数限制为 10 个。请注意，aiohttp 也有我们可以调整的连接限制，默认情况下它一次只允许 100 个连接。通过调整此限制可以实现与以下相同的效果。

清单 11.7 使用信号量限制 API 请求

```python
import asyncio
from asyncio import Semaphore
from aiohttp import ClientSession
 
 
async def get_url(url: str,
                  session: ClientSession,
                  semaphore: Semaphore):
    print('Waiting to acquire semaphore...')
    async with semaphore:
        print('Acquired semaphore, requesting...')
        response = await session.get(url)
        print('Finished requesting')
        return response.status
 
async def main():
    semaphore = Semaphore(10)
    async with ClientSession() as session:
        tasks = [get_url('https:/ / www .example .com', session, semaphore)
                 for _ in range(1000)]
        await asyncio.gather(*tasks)
 
 
asyncio.run(main())
```

虽然取决于外部延迟因素，输出将是不确定的，但你应该会看到类似于以下内容的输出：

```
Acquired semaphore, requesting...
Acquired semaphore, requesting...
Acquired semaphore, requesting...
Acquired semaphore, requesting...
Acquired semaphore, requesting...
Finished requesting
Finished requesting
Acquired semaphore, requesting...
Acquired semaphore, requesting...
```

每次请求完成时，都会释放信号量，这意味着可以开始等待信号量的阻塞任务。这意味着我们在给定时间最多只能运行 10 个请求，当一个请求完成时，我们可以启动一个新请求。

这解决了并发运行的请求过多的问题，但上面的代码是突发的，这意味着它有可能同时突发 10 个请求，从而产生潜在的流量峰值。如果我们担心我们正在调用的 API 的负载峰值，这可能是不可取的。如果你只需要在某个单位时间内爆发出一定数量的请求，则需要将其与流量整形算法的实现一起使用，例如“漏桶”或“令牌桶”。

### 11.3.1 有界信号量
信号量的一个方面是调用 release 的次数多于我们调用的 acquire 是有效的。如果我们总是使用带有 async with 块的信号量，这是不可能的，因为每次获取都会自动与发布配对。但是，如果我们需要对发布和获取机制进行更细粒度的控制（例如，也许我们有一些分支代码，其中一个分支让我们比另一个更早发布），我们可能会遇到问题。作为一个例子，让我们看看当我们有一个正常的协程获取和释放一个带有 async with 块的信号量时会发生什么，而该协程正在执行另一个协程调用 release。

清单 11.8 释放的比我们获得的多

```python
import asyncio
from asyncio import Semaphore
 
async def acquire(semaphore: Semaphore):
    print('Waiting to acquire')
    async with semaphore:
        print('Acquired')
        await asyncio.sleep(5)
    print('Releasing')
 
 
async def release(semaphore: Semaphore):
    print('Releasing as a one off!')
    semaphore.release()
    print('Released as a one off!')
 
 
async def main():
    semaphore = Semaphore(2)
 
    print("Acquiring twice, releasing three times...")
    await asyncio.gather(acquire(semaphore),
                         acquire(semaphore),
                         release(semaphore))
 
    print("Acquiring three times...")
    await asyncio.gather(acquire(semaphore),
                         acquire(semaphore),
                         acquire(semaphore))
 
 
asyncio.run(main())
```

在前面的清单中，我们创建了一个带有两个许可的信号量。然后我们运行两次获取调用和一次释放调用，这意味着我们将调用释放三次。我们第一次调用 gather 似乎运行良好，给了我们以下输出：

```python
Acquiring twice, releasing three times...
Waiting to acquire
Acquired
Waiting to acquire
Acquired
Releasing as a one off!
Released as a one off!
Releasing
Releasing
```

但是，我们第二次获取信号量三次的调用遇到了问题，我们一次获取了三次锁！我们无意中增加了信号量可用的许可数量：

```python
Acquiring three times...
Waiting to acquire
Acquired
Waiting to acquire
Acquired
Waiting to acquire
Acquired
Releasing
Releasing
Releasing
```

为了处理这些类型的情况，asyncio 提供了一个 BoundedSemaphore。这个信号量的行为与我们一直在使用的信号量完全一样，关键区别在于如果我们调用 release 会抛出一个 ValueError: BoundedSemaphore release too many times 异常，这样它会改变可用的许可。让我们看一下以下清单中的一个非常简单的示例。

清单 11.9 有界信号量

```python
import asyncio
from asyncio import BoundedSemaphore
 
 
async def main():
    semaphore = BoundedSemaphore(1)
 
    await semaphore.acquire()
    semaphore.release()
    semaphore.release()
 
 
asyncio.run(main())
```

当我们运行上面的清单时，我们对 release 的第二次调用将抛出一个 ValueError ，表明我们已经释放了太多次信号量。如果你将清单 11.8 中的代码更改为使用 BoundedSemaphore 而不是 Semaphore，你将看到类似的结果。如果你手动调用acquire 和release 以动态增加信号量可用的许可数量将是一个错误，那么使用BoundedSemaphore 是明智的，因此你会看到一个异常来警告你该错误。

我们现在已经了解了如何使用信号量来限制并发性，这在我们需要在应用程序中限制并发性的情况下很有用。 asyncio 同步原语不仅允许我们限制并发性，还允许我们在发生某些事情时通知任务。接下来，让我们看看如何使用 Event 同步原语来做到这一点。

## 11.4 用事件通知任务
有时，我们可能需要等待一些外部事件发生才能继续。我们可能需要等待缓冲区填满才能开始处理它，我们可能需要等待设备连接到我们的应用程序，或者我们可能需要等待一些初始化发生。我们也可能有多个任务等待处理可能尚不可用的数据。事件对象提供了一种机制来帮助我们在我们想要空闲而等待特定事件发生的情况下帮助我们。

在内部，Event 类跟踪指示事件是否已经发生的标志。我们可以通过两种方法来控制这个标志，设置和清除。 set 方法将此内部标志设置为 True 并通知等待事件发生的任何人。 clear 方法将此内部标志设置为 False，任何等待事件的人现在都将阻塞。

使用这两种方法，我们可以管理内部状态，但是我们如何阻塞直到事件发生呢？ Event 类有一个名为 wait 的协程方法。当我们等待这个协程时，它会阻塞，直到有人调用事件对象上的 set。一旦发生这种情况，任何额外的等待调用都不会阻塞并且会立即返回。如果我们在调用 set 后调用 clear，则调用 wait 将再次开始阻塞，直到我们再次调用 set。

让我们创建一个虚拟示例来查看正在发生的事件。我们会假装我们有两个任务依赖于正在发生的事情。在触发事件之前，我们会让这些任务等待并处于空闲状态。

清单 11.10 事件基础

```python
import asyncio
import functools
from asyncio import Event
 
 
def trigger_event(event: Event):
    event.set()
 
 
async def do_work_on_event(event: Event):
    print('Waiting for event...')
    await event.wait()                             ❶
    print('Performing work!')
    await asyncio.sleep(1)                         ❷
    print('Finished work!')
    event.clear()                                  ❸
 
 
async def main():
    event = asyncio.Event()
    asyncio.get_running_loop().call_later(5.0, functools.partial(trigger_event, event))      ❹
    await asyncio.gather(do_work_on_event(event), do_work_on_event(event))
 
 
asyncio.run(main())
```

❶ 等到事件发生。
❷一旦事件发生，wait就不再阻塞，我们可以做工作了。
❸ 重置事件，因此未来的等待调用将被阻塞。
❹ 5 秒后触发事件。
在上面的清单中，我们创建了一个协程方法 do_work_on_event，这个协程接收一个事件并首先调用它的等待协程。这将一直阻塞，直到有人调用事件的 set 方法来指示事件已经发生。我们还创建了一个简单的方法 trigger_event，它设置一个给定的事件。在我们的主协程中，我们创建了一个事件对象，并使用 call_later 在未来 5 秒触发事件。然后我们用gather调用do_work_on_event两次，这将为我们创建两个并发任务。我们将看到两个 do_work_on_event 任务空闲 5 秒，直到我们触发事件，之后我们将看到它们执行它们的工作，并为我们提供以下输出：

```python
Waiting for event...
Waiting for event...
Triggering event!
Performing work!
Performing work!
Finished work!
Finished work!
```

这向我们展示了基础知识；等待一个事件会阻塞一个或多个协程，直到我们触发一个事件，之后它们才能继续工作。接下来，让我们看一个更真实的例子。想象一下，我们正在构建一个 API 来接受来自客户端的文件上传。由于网络延迟和缓冲，文件上传可能需要一些时间才能完成。有了这个约束，我们希望我们的 API 有一个协程来阻塞，直到文件完全上传。然后，这个协程的调用者可以等待所有数据进入并用它做任何他们想做的事情。

我们可以使用一个事件来实现这一点。我们将有一个协程来监听上传的数据并将其存储在内部缓冲区中。一旦我们到达文件的末尾，我们将触发一个指示上传完成的事件。然后我们将有一个协程方法来获取文件内容，它将等待事件被设置。一旦设置了事件，我们就可以返回完全形成的上传数据。让我们在一个名为 FileUpload: 的类中创建这个 API。

清单 11.11 文件上传 API

```python
import asyncio
from asyncio import StreamReader, StreamWriter
 
 
class FileUpload:
    def __init__(self,
                 reader: StreamReader,
                 writer: StreamWriter):
        self._reader = reader
        self._writer = writer
        self._finished_event = asyncio.Event()
        self._buffer = b''
        self._upload_task = None
 
    def listen_for_uploads(self):
        self._upload_task = asyncio.create_task(self._accept_upload())   ❶
 
    async def _accept_upload(self):
        while data := await self._reader.read(1024):
            self._buffer = self._buffer + data
        self._finished_event.set()
        self._writer.close()
        await self._writer.wait_closed()
 
    async def get_contents(self):                                        ❷
        await self._finished_event.wait()
        return self._buffer
```

❶ 创建一个任务来监听上传并将其附加到缓冲区。
❷ 阻塞直到完成事件被设置，然后返回缓冲区的内容。
现在让我们创建一个文件上传服务器来测试这个 API。假设在每次成功上传时，我们都希望将内容转储到标准输出。当客户端连接时，我们将创建一个 FileUpload 对象并调用 listen_for_uploads。然后，我们将创建一个单独的任务来等待 get_contents 的结果。

清单 11.12 在文件上传服务器中使用 API

```python
import asyncio
from asyncio import StreamReader, StreamWriter
from chapter_11.listing_11_11 import FileUpload
 
 
class FileServer:
 
    def __init__(self, host: str, port: int):
        self.host = host
        self.port = port
        self.upload_event = asyncio.Event()
 
    async def start_server(self):
        server = await asyncio.start_server(self._client_connected,
                                            self.host,
                                            self.port)
        await server.serve_forever()
 
    async def dump_contents_on_complete(self, upload: FileUpload):
        file_contents = await upload.get_contents()
        print(file_contents)
 
    def _client_connected(self, reader: StreamReader, writer: StreamWriter):
        upload = FileUpload(reader, writer)
        upload.listen_for_uploads()
        asyncio.create_task(self.dump_contents_on_complete(upload))
 
 
async def main():
    server = FileServer('127.0.0.1', 9000)
    await server.start_server()
 
 
asyncio.run(main())
```

在前面的清单中，我们创建了一个 FileServer 类。每次客户端连接到我们的服务器时，我们都会创建一个我们在前面的清单中创建的 FileUpload 类的实例，它开始侦听来自已连接客户端的上传。我们还同时为 dump_contents_on_complete 协程创建了一个任务。这会在文件上传时调用 get_contents 协程（仅在上传完成后返回）并将文件打印到标准输出。

我们可以使用 netcat 来测试这个服务器。在你的文件系统上选择一个文件，然后运行以下命令，将 file 替换为你选择的文件：

```python
cat file | nc localhost 9000
```

一旦所有内容完全上传，你应该会看到你上传的任何文件都打印到标准输出。

事件需要注意的一个缺点是它们触发的频率可能比你的协程响应它们的频率高。假设我们在一种生产者-消费者工作流中使用单个事件来唤醒多个任务。如果我们所有的工作任务都忙了很长时间，事件可能会在我们工作的时候运行，而我们永远看不到它。让我们创建一个虚拟示例来演示这一点。我们将创建两个工作任务，每个任务执行 5 秒的工作。我们还将创建一个每秒触发一个事件的任务，超过消费者可以处理的速度。

清单 11.13 一个工人落后于一个事件

```python
import asyncio
from asyncio import Event
from contextlib import suppress
 
 
async def trigger_event_periodically(event: Event):
    while True:
        print('Triggering event!')
        event.set()
        await asyncio.sleep(1)
 
 
async def do_work_on_event(event: Event):
    while True:
        print('Waiting for event...')
        await event.wait()
        event.clear()
        print('Performing work!')
        await asyncio.sleep(5)
        print('Finished work!')
 
 
async def main():
    event = asyncio.Event()
    trigger = asyncio.wait_for(trigger_event_periodically(event), 5.0)
 
    with suppress(asyncio.TimeoutError):
        await asyncio.gather(do_work_on_event(event), do_work_on_event(event), trigger)
 
 
asyncio.run(main())
```

当我们运行前面的清单时，我们会看到我们的事件触发并且我们的两个工作人员同时开始他们的工作。与此同时，我们不断触发我们的事件。由于我们的工作人员很忙，他们不会看到我们的事件再次触发，直到他们完成工作并再次调用 event.wait()。如果你关心每次事件发生时的响应，则需要使用排队机制，我们将在下一章中学习。

当我们想要在特定事件发生时发出警报时，事件很有用，但是如果我们需要将等待事件与对共享资源（例如数据库连接）的独占访问相结合，会发生什么？条件可以帮助我们解决这些类型的工作流。

## 11.5 条件
当事情发生时，事件对于简单的通知很有用，但是更复杂的用例呢？想象一下，需要访问需要锁定事件的共享资源，等待一组更复杂的事实为真，然后再继续或只唤醒一定数量的任务而不是所有任务。条件在这些类型的情况下可能很有用。它们是迄今为止我们遇到的最复杂的同步原语，因此，你可能不需要经常使用它们。

条件将锁和事件的各个方面结合到一个同步原语中，有效地包装了两者的行为。我们首先获取条件锁，让我们的协程独占访问任何共享资源，让我们能够安全地更改我们需要的任何状态。然后，我们使用 wait 或 wait_for 协程等待特定事件发生。这些协程释放锁并阻塞，直到事件发生，一旦发生，它就会重新获得锁，从而为我们提供独占访问权限。

由于这有点令人困惑，让我们创建一个虚拟示例来了解如何使用条件。我们将创建两个工作任务，每个任务都尝试获取条件锁，然后等待事件通知。然后，几秒钟后，我们将触发条件，这将唤醒两个工作任务并允许它们工作。

清单 11.14 条件基础

```python
import asyncio
from asyncio import Condition
 
 
async def do_work(condition: Condition):
    while True:
        print('Waiting for condition lock...')
        async with condition:                                              ❶
            print('Acquired lock, releasing and waiting for condition...')
            await condition.wait()                                         ❷
            print('Condition event fired, re-acquiring lock and doing work...')
            await asyncio.sleep(1)
        print('Work finished, lock released.')                             ❸
 
async def fire_event(condition: Condition):
    while True:
        await asyncio.sleep(5)
        print('About to notify, acquiring condition lock...')
        async with condition:
            print('Lock acquired, notifying all workers.')
            condition.notify_all()                                         ❹
        print('Notification finished, releasing lock.')
 
 
async def main():
    condition = Condition()
 
    asyncio.create_task(fire_event(condition))
    await asyncio.gather(do_work(condition), do_work(condition))
 
 
asyncio.run(main())
```

❶ 等待获取条件锁；一旦获得，释放锁。
❷ 等待事件触发；完成后，重新获取条件锁。
❸ 一旦我们退出 async with 块，释放条件锁。
❹ 通知所有任务事件已经发生。
在前面的清单中，我们创建了两个协程方法：do_work 和 fire_event。 do_work 方法获取条件，类似于获取锁，然后调用条件的等待协程方法。等待协程方法将阻塞，直到有人调用条件的 notify_all 方法。

fire_event 协程方法会休眠一点，然后获取条件并调用 notify_all 方法，该方法将唤醒当前正在等待条件的所有任务。然后，在我们的主协程中，我们创建一个 fire_event 任务和两个 do_work 任务并同时运行它们。运行此程序时，如果应用程序运行，你将看到以下重复：

```python
Worker 1: waiting for condition lock...
Worker 1: acquired lock, releasing and waiting for condition...
Worker 2: waiting for condition lock...
Worker 2: acquired lock, releasing and waiting for condition...
fire_event: about to notify, acquiring condition lock...
fire_event: Lock acquired, notifying all workers.
fire_event: Notification finished, releasing lock.
Worker 1: condition event fired, re-acquiring lock and doing work...
Worker 1: Work finished, lock released.
Worker 1: waiting for condition lock...
Worker 2: condition event fired, re-acquiring lock and doing work...
Worker 2: Work finished, lock released.
Worker 2: waiting for condition lock...
Worker 1: acquired lock, releasing and waiting for condition...
Worker 2: acquired lock, releasing and waiting for condition...
```

你会注意到两个工作人员立即启动并阻塞等待 fire_ 事件协程调用 notify_all。一旦 fire_event 调用 notify_all，工作任务就会唤醒，然后继续工作。

条件有一个额外的协程方法，称为 wait_for。 wait_for 不会阻塞直到有人通知条件，而是接受一个谓词（一个返回布尔值的无参数函数）并将阻塞直到该谓词返回 True。当有一个共享资源与一些依赖于某些状态的协程变为 true 时，这被证明是有用的。

例如，假设我们正在创建一个类来包装数据库连接并运行查询。我们首先有一个底层连接，它不能同时运行多个查询，并且在有人尝试运行查询之前数据库连接可能没有初始化。共享资源和我们需要阻止的事件的组合为我们提供了使用 Condition 的正确条件。让我们用一个模拟数据库连接类来模拟它。此类将运行查询，但只有在我们正确初始化连接后才会这样做。然后，在我们完成连接初始化之前，我们将使用这个模拟连接类尝试同时运行两个查询。

清单 11.15 使用条件等待特定状态

```python
import asyncio
from enum import Enum
 
 
class ConnectionState(Enum):
    WAIT_INIT = 0
    INITIALIZING = 1
    INITIALIZED = 2
 
 
class Connection:
 
    def __init__(self):
        self._state = ConnectionState.WAIT_INIT
        self._condition = asyncio.Condition()
 
    async def initialize(self):
        await self._change_state(ConnectionState.INITIALIZING)
        print('initialize: Initializing connection...')
        await asyncio.sleep(3)  # simulate connection startup time
        print('initialize: Finished initializing connection')
        await self._change_state(ConnectionState.INITIALIZED)
 
    async def execute(self, query: str):
        async with self._condition:
            print('execute: Waiting for connection to initialize')
            await self._condition.wait_for(self._is_initialized)
            print(f'execute: Running {query}!!!')
            await asyncio.sleep(3)  # simulate a long query
 
    async def _change_state(self, state: ConnectionState):
        async with self._condition:
            print(f'change_state: State changing from {self._state} to {state}')
            self._state = state
            self._condition.notify_all()
 
    def _is_initialized(self):
        if self._state is not ConnectionState.INITIALIZED:
            print(f'_is_initialized: Connection not finished initializing, state is {self._state}')
            return False
        print(f'_is_initialized: Connection is initialized!')
        return True
async def main():
    connection = Connection()
    query_one = asyncio.create_task(connection.execute('select * from table'))
    query_two = asyncio.create_task(connection.execute('select * from other_table'))
    asyncio.create_task(connection.initialize())
    await query_one
    await query_two
 
 
asyncio.run(main())
```

在前面的清单中，我们创建了一个包含条件对象的连接类，并跟踪我们初始化为 WAIT_INIT 的内部状态，表明我们正在等待初始化发生。我们还在 Connection 类上创建了一些方法。第一个是初始化，它模拟创建数据库连接。此方法在第一次调用时调用 _change_state 方法将状态设置为 INITIALIZING，然后在连接初始化后将状态设置为 INITIALIZED。在_change_state方法内部，我们设置内部状态，然后调用条件notify_all方法。这将唤醒任何等待条件的任务。

在我们的 execute 方法中，我们在 async with 块中获取条件对象，然后我们使用谓词调用 wait_for，以检查状态是否为 INITIALIZED。这将阻塞直到我们的数据库连接完全初始化，防止我们在连接存在之前意外发出查询。然后，在我们的主协程中，我们创建一个连接类并创建两个任务来运行查询，然后是一个任务来初始化连接。运行此代码，你将看到以下输出，表明我们的查询在运行查询之前正确等待初始化任务完成：

```python
execute: Waiting for connection to initialize
_is_initialized: Connection not finished initializing, state is ConnectionState.WAIT_INIT
execute: Waiting for connection to initialize
_is_initialized: Connection not finished initializing, state is ConnectionState.WAIT_INIT
change_state: State changing from ConnectionState.WAIT_INIT to ConnectionState.INITIALIZING
initialize: Initializing connection...
_is_initialized: Connection not finished initializing, state is ConnectionState.INITIALIZING
_is_initialized: Connection not finished initializing, state is ConnectionState.INITIALIZING
initialize: Finished initializing connection
change_state: State changing from ConnectionState.INITIALIZING to ConnectionState.INITIALIZED
_is_initialized: Connection is initialized!
execute: Running select * from table!!!
_is_initialized: Connection is initialized!
execute: Running select * from other_table!!!
```

条件在我们需要访问共享资源并且需要在工作之前通知我们的状态的情况下很有用。这是一个有点复杂的用例，因此，你不太可能在 asyncio 代码中遇到或需要条件。

## 概括

- 我们已经了解了单线程并发错误以及它们与多线程和多处理中的并发错误有何不同。
- 我们知道如何使用异步锁来防止并发错误和同步协程。由于 asyncio 的单线程特性，这种情况发生的频率较低，有时在等待期间共享状态可能发生变化时可能需要它们。
- 我们已经学习了如何使用信号量来控制对有限资源的访问和限制并发，这对于流量整形很有用。
- 我们知道如何在发生某些事情时使用事件来触发动作，例如初始化或唤醒工作任务。
- 我们知道如何使用条件来等待一个动作，并因为一个动作而获得对共享资源的访问权。