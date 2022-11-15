# 管理子进程

本章涵盖

- 异步运行多个子进程
- 处理子进程的标准输出
- 使用标准输入与子进程通信
- 避免子进程的死锁和其他陷阱

许多应用程序永远不需要离开 Python 的世界。我们将从其他 Python 库和模块中调用代码，或者使用多处理或多线程同时运行 Python 代码。然而，并不是我们想要与之交互的所有东西都是用 Python 编写的。我们可能有一个已经构建的应用程序，它是用 C++、Go、Rust 或其他提供更好运行时特性的语言编写的，或者只是已经存在供我们使用而无需重新实现。我们可能还想使用操作系统提供的命令行实用程序，例如用于搜索大文件的 GREP、用于发出 HTTP 请求的 cURL，或者我们可以使用的任何数量的应用程序。

在标准 Python 中，我们可以使用 subprocess 模块在不同的进程中运行不同的应用程序。像大多数其他 Python 模块一样，标准的子进程 API 是阻塞的，这使得它在没有多线程或多处理的情况下与 asyncio 不兼容。 asyncio 提供了一个以子流程模块为模型的模块，用于与协程异步创建和管理子流程。

在本章中，我们将通过运行用不同语言编写的应用程序来学习使用 asyncio 创建和管理子进程的基础知识。我们还将学习如何处理输入和输出、读取标准输出以及将输入从我们的应用程序发送到我们的子进程。

## 13.1 创建子流程
假设你想扩展现有 Python Web API 的功能。你组织中的另一个团队已经为他们拥有的批处理机制在命令行应用程序中构建了你想要的功能，但存在一个主要问题是该应用程序是用 Rust 编写的。鉴于应用程序已经存在，你不希望通过在 Python 中重新实现它来重新发明轮子。有没有办法我们仍然可以在我们现有的 Python API 中使用这个应用程序的功能？

由于这个应用程序有一个命令行界面，我们可以使用子进程来重用这个应用程序。我们将通过其命令行界面调用应用程序并在单独的子进程中运行它。然后，我们可以读取应用程序的结果并根据需要在现有 API 中使用它，从而省去了重新实现应用程序的麻烦。

那么我们如何创建一个子流程并执行它呢？ asyncio 提供了两个开箱即用的协程函数来创建子进程：asyncio.create_subprocess_shell 和 asyncio.create_subprocess_exec。这些协程函数中的每一个都返回一个 Process 的实例，它具有让我们等待进程完成和终止进程以及其他一些进程的方法。为什么有两个协程来完成看似相同的任务？我们什么时候想使用其中一个？ create_subprocess_shell 协程函数在安装在它运行的系统上的 shell 中创建一个子进程，例如 zsh 或 bash。一般来说，除非你需要使用 shell 的功能，否则你会想要使用 create_subprocess_exec。使用 shell 可能会有一些陷阱，例如不同的机器有不同的 shell，或者相同的 shell 配置不同。这使得很难保证你的应用程序在不同的机器上表现相同。

要了解如何创建子进程的基础知识，让我们编写一个异步应用程序来运行一个简单的命令行程序。我们将从 ls 程序开始，它列出了当前目录的内容以进行测试，尽管我们不太可能在现实世界中这样做。如果你在 Windows 机器上运行，请将 ls -l 替换为 cmd /c dir。

清单 13.1 在子进程中运行一个简单的命令

```python
import asyncio
from asyncio.subprocess import Process
 
 
async def main():
    process: Process = await asyncio.create_subprocess_exec('ls', '-l')
    print(f'Process pid is: {process.pid}')
    status_code = await process.wait()
    print(f'Status code: {status_code}')
 
 
asyncio.run(main())
```

在前面的清单中，我们创建了一个 Process 实例来使用 create_subprocess_exec 运行 ls 命令。我们还可以通过在后面添加其他参数来指定传递给程序的参数。这里我们传入-l，它添加了一些关于谁在目录中创建文件的额外信息。创建进程后，我们打印出进程 ID，然后调用等待协程。这个协程会一直等到进程完成，一旦完成就会返回子进程的状态码；在这种情况下，它应该为零。默认情况下，我们的子进程的标准输出将通过管道传输到我们自己的应用程序的标准输出，因此当你运行它时，你应该会看到如下内容，具体取决于你目录中的内容：

```sh
Process pid is: 54438
total 8
drwxr-xr-x   4 matthewfowler  staff  128 Dec 23 15:20 .
drwxr-xr-x  25 matthewfowler  staff  800 Dec 23 14:52 ..
-rw-r--r--   1 matthewfowler  staff    0 Dec 23 14:52 __init__.py
-rw-r--r--   1 matthewfowler  staff  293 Dec 23 15:20 basics.py
Status code: 0
```

请注意，等待协程将阻塞，直到应用程序终止，并且无法保证进程需要多长时间才能终止，更不用说它是否会终止。如果你担心进程失控，则需要使用 asyncio.wait_for 引入超时。但是，有一个警告。回想一下，如果超时，wait_for 将终止正在运行的协程。你可能会认为这将终止进程，但事实并非如此。它只终止等待进程完成的任务，而不是底层进程。

我们需要一种更好的方法来在超时时关闭该进程。幸运的是，Process 有两种方法可以帮助我们解决这种情况：终止和终止。 terminate 方法将向子进程发送 SIGTERM 信号，而 kill 将发送 SIGKILL 信号。请注意，这两种方法都不是协程，也是非阻塞的。他们只是发送信号。如果你想在终止子进程后尝试获取返回码，或者你想等待任何清理，则需要再次调用 wait。

让我们测试一下使用 sleep 命令行应用程序终止长时间运行的应用程序（对于 Windows 用户，将 'sleep'、'3' 替换为更复杂的 'cmd'、'start'、'/wait'、'timeout'、' 3'）。我们将创建一个休眠几秒钟的子进程，并尝试在它有机会完成之前终止它。

清单 13.2 终止子流程

```python
import asyncio
from asyncio.subprocess import Process
 
async def main():
    process: Process = await asyncio.create_subprocess_exec('sleep', '3')
    print(f'Process pid is: {process.pid}')
    try:
        status_code = await asyncio.wait_for(process.wait(), timeout=1.0)
        print(status_code)
    except asyncio.TimeoutError:
        print('Timed out waiting to finish, terminating...')
        process.terminate()
        status_code = await process.wait()
        print(status_code)
 
 
asyncio.run(main())
```

在前面的清单中，我们创建了一个需要 3 秒才能完成的子进程，但将其包装在一个带有 1 秒超时的 wait_for 中。 1 秒后，wait_for 将抛出 TimeoutError，在 except 块中我们终止进程并等待它完成，打印出它的状态码。这应该给我们类似于以下的输出：

```sh
Process pid is: 54709
Timed out waiting to finish, terminating...
-15
```

编写自己的代码时要注意的一件事是 except 块内的等待仍然有可能需要很长时间，如果这是一个问题，你可能希望将其包装在 wait_for 中。

### 13.1.1 控制标准输出
在前面的示例中，我们的子进程的标准输出直接进入我们应用程序的标准输出。如果我们不想要这种行为怎么办？也许我们想对输出做额外的处理，或者输出是无关紧要的，我们可以放心地忽略它。 create_subprocess_exec 协程有一个 stdout 参数，可以让我们指定我们希望标准输出到哪里。这个参数接受一个枚举，让我们指定是否要将子进程的输出重定向到我们自己的标准输出，将其通过管道传输到 StreamReader，或者通过将其重定向到 /dev/null 来完全忽略它。

假设我们计划同时运行多个子进程并回显它们的输出。我们想知道哪个子进程生成了输出以避免混淆。为了使这个输出更容易阅读，我们将添加一些额外的数据，说明哪个子进程生成了输出，然后再将其写入应用程序的标准输出。我们将在打印输出之前添加生成输出的命令。

为此，我们需要做的第一件事是将 stdout 参数设置为 asyncio .subprocess.PIPE。这告诉子进程创建一个新的 StreamReader 实例，我们可以使用它来读取进程的输出。然后我们可以使用 Proccess.stdout 字段访问这个流读取器。让我们用我们的 ls -la 命令试试这个。

清单 13.3 使用标准输出流阅读器

```python
import asyncio
from asyncio import StreamReader
from asyncio.subprocess import Process
 
 
async def write_output(prefix: str, stdout: StreamReader):
    while line := await stdout.readline():
        print(f'[{prefix}]: {line.rstrip().decode()}')
 
 
async def main():
    program = ['ls', '-la']
    process: Process = await asyncio.create_subprocess_exec(*program,
                                                            stdout=asyncio
                                                            .subprocess.PIPE)
    print(f'Process pid is: {process.pid}')
    stdout_task = asyncio.create_task(write_output(' '.join(program), process.stdout))
 
    return_code, _ = await asyncio.gather(process.wait(), stdout_task)
    print(f'Process returned: {return_code}')
 
 
asyncio.run(main())
```

在前面的清单中，我们首先创建一个协程 write_output 来为流读取器逐行输出添加前缀。然后，在我们的主协程中，我们创建一个子进程，指定我们想要管道标准输出。我们还创建了一个运行 write_output 的任务，传入进程的标准输出流读取器，并与 wait 并发运行。运行此命令时，你将看到前面带有命令的输出：

```sh
Process pid is: 56925
[ls -la]: total 32
[ls -la]: drwxr-xr-x   7 matthewfowler  staff  224 Dec 23 09:07 .
[ls -la]: drwxr-xr-x  25 matthewfowler  staff  800 Dec 23 14:52 ..
[ls -la]: -rw-r--r--   1 matthewfowler  staff    0 Dec 23 14:52 __init__.py
Process returned: 0
```

使用管道以及处理子流程输入和输出的一个关键方面是它们容易出现死锁。如果我们的子进程生成大量输出，而我们没有正确使用它，那么等待协程特别容易受到这种影响。为了证明这一点，让我们看一个简单的示例，它生成一个 Python 应用程序，该应用程序将大量数据写入标准输出并一次全部刷新。

清单 13.4 生成大量输出

```python
import sys
 
[sys.stdout.buffer.write(b'Hello there!!\n') for _ in range(1000000)]
 
sys.stdout.flush()
```

前面的清单写着 Hello there!!到标准输出缓冲区 1,000,000 次并一次将其全部刷新。让我们看看如果我们在这个应用程序中使用管道但不使用数据会发生什么。

清单 13.5 管道死锁

```python
import asyncio
from asyncio.subprocess import Process
 
 
async def main():
    program = ['python3', 'listing_13_4.py']
    process: Process = await asyncio.create_subprocess_exec(*program,
                                                            stdout=asyncio
                                                            .subprocess.PIPE)
    print(f'Process pid is: {process.pid}')
 
    return_code = await process.wait()
    print(f'Process returned: {return_code}')
 
 
asyncio.run(main())
```

如果你运行前面的清单，你将看到进程 pid 打印出来，然后仅此而已。该应用程序将永远挂起，你需要强制终止它。如果这在你的系统上没有发生，只需增加我们在输出应用程序中输出数据的次数，你最终会遇到问题。

我们的应用程序看起来很简单，那么为什么会遇到这个死锁呢？问题在于流读取器的缓冲区如何工作。当流读取器的缓冲区填满时，任何更多写入它的调用都会阻塞，直到缓冲区中有更多空间可用。虽然我们的流读取器缓冲区因缓冲区已满而被阻塞，但我们的进程仍在尝试完成将其大输出写入流读取器。这使得我们的进程依赖于流读取器变得畅通，但流读取器永远不会畅通，因为我们永远不会释放缓冲区中的任何空间。这是一个循环依赖，因此是一个死锁。

以前，我们通过在等待进程完成时同时读取标准输出流读取器来完全避免这个问题。这意味着即使缓冲区已满，我们也会将其排空，这样进程就不会无限期地阻塞等待写入额外的数据。在处理管道时，你需要小心使用流数据，以免遇到死锁。

你还可以通过避免使用等待协程来解决此问题。此外，Process 类还有另一个协程方法，称为通信，可以完全避免死锁。这个协程一直阻塞，直到子进程完成并同时使用标准输出和标准错误，一旦应用程序完成就返回完整的输出。让我们修改之前的示例，使用通信来解决问题。

清单 13.6 使用通信

```python
import asyncio
from asyncio.subprocess import Process
 
 
async def main():
    program = ['python3', 'listing_13_4.py']
    process: Process = await asyncio.create_subprocess_exec(*program,
                                                            stdout=asyncio
                                                            .subprocess.PIPE)
    print(f'Process pid is: {process.pid}')
 
    stdout, stderr = await process.communicate()
    print(stdout)
    print(stderr)
    print(f'Process returned: {process.returncode}')
 
 
asyncio.run(main())
```

当你运行前面的清单时，你会看到应用程序的所有输出一次全部打印到控制台（并且 None 打印一次，因为我们没有向标准输出写入任何内容）。在内部，通信创建了一些任务，这些任务不断地将标准输出和标准错误的输出读取到内部缓冲区中，从而避免任何死锁问题。虽然我们避免了潜在的死锁，但我们有一个严重的缺点，即我们不能交互式地处理来自标准输出的输出。如果你需要对应用程序的输出做出反应（也许你需要在遇到特定消息或产生另一个任务时终止），你需要使用等待，但要小心阅读输出从你的流阅读器适当地避免死锁。

另一个缺点是通信将来自标准输出和标准输入的所有数据缓存在内存中。如果你正在使用可能产生大量数据的子进程，那么你将面临内存不足的风险。我们将在下一节中看到如何解决这些缺点。

### 13.1.2 并发运行子进程
现在我们已经了解了创建、终止和读取子流程输出的基础知识，现在我们将添加现有知识以同时运行多个应用程序。假设我们需要加密内存中的多段文本，出于安全目的，我们希望使用 Twofish 密码算法。 hashlib 模块不支持此算法，因此我们需要一个替代方案。我们可以使用 gpg（GNU Privacy Guard 的缩写，它是 PGP [pretty good privacy] 的免费软件替代品）命令行应用程序。你可以在 https://gnupg.org/download/ 下载 gpg。

首先，让我们定义要用于加密的命令。我们可以通过定义密码并使用命令行参数设置算法来使用 gpg。然后，就是将文本回显到应用程序的问题。例如，要加密文本“encrypt this!”，我们可以运行以下命令：

```sh
echo 'encrypt this!' | gpg -c --batch --passphrase 3ncryptm3 --cipher-algo TWOFISH
```

这应该会产生类似于以下内容的标准输出的加密输出：

```sh
?
Q+??/??*??C??H`??`)R??u??7þ_{f{R;n?FE .?b5??(?i??????o\k?b<????`%
```

这将在我们的命令行上运行，但如果我们使用 create_subprocess_exec，它将无法运行，因为我们不会有 |管道运算符可用（如果你确实需要管道，create_subprocess_shell 将起作用）。那么我们如何传入我们想要加密的文本呢？除了允许我们管道标准输出和标准错误之外，通信和等待也让我们管道输入标准输入。通信协程还允许我们在启动应用程序时指定输入字节。如果我们在创建进程时通过管道传输标准输入，这些字节将被发送到应用程序。这对我们很有效；我们将简单地通过通信协程传递我们想要加密的字符串。

让我们通过生成随机文本片段并同时加密它们来尝试一下。我们将创建一个包含 100 个随机文本字符串的列表，每个字符串包含 1,000 个字符，并同时在每个字符串上运行 gpg。

清单 13.7 并发加密文本

```python
import asyncio
import random
import string
import time
from asyncio.subprocess import Process
 
 
async def encrypt(text: str) -> bytes:
    program = ['gpg', '-c', '--batch', '--passphrase', '3ncryptm3',
        '--cipher-algo', 'TWOFISH']
 
    process: Process = await asyncio.create_subprocess_exec(*program,
                                                            stdout=asyncio
                                                            .subprocess.PIPE,
                                                            stdin=asyncio
                                                            .subprocess.PIPE)
    stdout, stderr = await process.communicate(text.encode())
    return stdout
 
 
async def main():
    text_list = [''.join(random.choice(string.ascii_letters) for _ in range(1000)) for _ in range(100)]
 
    s = time.time()
    tasks = [asyncio.create_task(encrypt(text)) for text in text_list]
    encrypted_text = await asyncio.gather(*tasks)
    e = time.time()
 
    print(f'Total time: {e - s}')
    print(encrypted_text)
 
 
asyncio.run(main())
```

在前面的清单中，我们定义了一个名为 encrypt 的协程，它创建一个 gpg 进程并发送我们想要通过通信加密的文本。为简单起见，我们只返回标准输出结果，不做任何错误处理；在现实世界的应用程序中，你可能希望在这里更加健壮。然后，在我们的主协程中，我们创建一个随机文本列表，并为每个文本创建一个加密任务。然后我们同时运行它们，收集并打印出总运行时间和加密的文本位。你可以通过将 await 放在 asyncio.create_task 前面并删除聚集来比较并发运行时和同步运行时，你应该会看到合理的加速。

在这个清单中，我们只有 100 条文本。如果我们有数千个或更多呢？我们当前的代码需要 100 条文本，并尝试同时加密它们；这意味着我们同时创建了 100 个进程。这带来了挑战，因为我们的机器资源有限，一个进程可能会占用大量内存。此外，启动成百上千个进程会产生重要的上下文切换开销。

在我们的例子中，gpg 造成了另一个问题，因为它依赖于共享状态来加密数据。如果你使用清单 13.7 中的代码并将文本的数量增加到数千，你可能会开始看到以下打印到标准错误：

```sh
gpg: waiting for lock on `/Users/matthewfowler/.gnupg/random_seed'...
```

因此，我们不仅创建了很多进程以及与之相关的所有开销，而且还创建了实际上在 gpg 需要的共享状态上被阻塞的进程。那么我们如何限制运行的进程数量来规避这个问题呢？这是信号量派上用场的完美示例。由于我们的工作是受 CPU 限制的，因此添加一个信号量来将进程数限制为我们可用的 CPU 内核数是有意义的。让我们通过使用限制在我们系统上的 CPU 内核数量的信号量并加密 1,000 条文本来试试这个，看看这是否可以提高我们的性能。

清单 13.8 带有信号量的子流程

```python
import asyncio
import random
import string
import time
import os
from asyncio import Semaphore
from asyncio.subprocess import Process
 
 
async def encrypt(sem: Semaphore, text: str) -> bytes:
    program = ['gpg', '-c', '--batch', '--passphrase', '3ncryptm3', '--cipher-algo', 'TWOFISH']
 
    async with sem:
        process: Process = await asyncio.create_subprocess_exec(*program,
                                                                stdout=asyncio
                                                                .subprocess.PIPE,
                                                                stdin=asyncio
                                                                .subprocess.PIPE)
        stdout, stderr = await process.communicate(text.encode())
        return stdout
 
 
async def main():
    text_list = [''.join(random.choice(string.ascii_letters) for _ in range(1000)) for _ in range(1000)]
    semaphore = Semaphore(os.cpu_count())
    s = time.time()
    tasks = [asyncio.create_task(encrypt(semaphore, text)) for text in text_list]
    encrypted_text = await asyncio.gather(*tasks)
    e = time.time()
 
    print(f'Total time: {e - s}')
 
 
asyncio.run(main())
```

将此与 1,000 条文本的运行时间与一组无限的子进程进行比较，你应该会看到一些性能改进，同时内存使用量减少。你可能会认为这类似于我们在第 6 章中看到的 ProcessPoolExecutor 的最大工作人员概念，你是对的。在内部，ProcessPoolExecutor 使用信号量来管理并发运行的进程数。

我们现在已经了解了有关同时创建、终止和运行多个子进程的基础知识。接下来，我们将看看如何以更具交互性的方式与子流程进行通信。

## 13.2 与子进程通信
到目前为止，我们一直在使用单向、非交互式的进程通信。但是，如果我们正在使用可能需要用户输入的应用程序怎么办？例如，我们可能会被要求输入密码、用户名或任何其他数量的输入。

在我们知道我们只有一个输入要处理的情况下，使用通信是理想的。我们之前看到过使用 gpg 发送文本进行加密，但是让我们在子进程明确要求输入时尝试一下。我们将首先创建一个简单的 Python 程序来询问用户名并将其回显到标准输出。

清单 13.9 回显用户输入

```python
username = input('Please enter a username: ')
print(f'Your username is {username}')
```

现在，我们可以使用通信来输入用户名。

清单 13.10 使用与标准输入通信

```python
import asyncio
from asyncio.subprocess import Process
 
 
async def main():
    program = ['python3', 'listing_13_9.py']
    process: Process = await asyncio.create_subprocess_exec(*program,
                                                            stdout=asyncio
                                                            .subprocess.PIPE,
                                                            stdin=asyncio
                                                            .subprocess.PIPE)
 
    stdout, stderr = await process.communicate(b'Zoot')
    print(stdout)
    print(stderr)
 
 
asyncio.run(main())
```

当我们运行此代码时，我们将看到 b'请输入用户名：你的用户名是 Zoot\n' 打印到控制台，因为我们的应用程序在我们第一次用户输入后立即终止。如果我们有一个更具交互性的应用程序，这将不起作用。例如，以这个应用程序为例，它反复询问用户输入并回显它，直到用户键入退出。

清单 13.11 一个回显应用程序

```python
user_input = ''
 
while user_input != 'quit':
    user_input = input('Enter text to echo: ')
    print(user_input)
```

由于通信会一直等到进程终止，所以我们需要使用等待并同时处理标准输出和标准输入。 Process 类在我们为 PIPE 设置标准输入时可以使用的标准输入字段中公开 StreamWriter。我们可以同时使用它和标准输出 StreamReader 来处理这些类型的应用程序。让我们看看如何使用以下清单来执行此操作，我们将在其中创建一个应用程序来将一些文本写入我们的子进程。

清单 13.12 使用带有子进程的 echo 应用程序

```python
import asyncio
from asyncio import StreamWriter, StreamReader
from asyncio.subprocess import Process
 
 
async def consume_and_send(text_list, stdout: StreamReader, stdin: StreamWriter):
    for text in text_list:
        line = await stdout.read(2048)
        print(line)
        stdin.write(text.encode())
        await stdin.drain()
 
 
async def main():
    program = ['python3', 'listing_13_11.py']
    process: Process = await asyncio.create_subprocess_exec(*program,
                                                            stdout=asyncio
                                                            .subprocess.PIPE,
                                                            stdin=asyncio
                                                            .subprocess.PIPE)
 
    text_input = ['one\n', 'two\n', 'three\n', 'four\n', 'quit\n']
 
    await asyncio.gather(consume_and_send(text_input, process.stdout, process.stdin), process.wait())
 
 
asyncio.run(main())
```

在前面的清单中，我们定义了一个 consume_and_send 协程，它读取标准输出，直到我们收到用户指定输入的预期消息。收到此消息后，我们将数据转储到我们自己的应用程序的标准输出，并将“text_list”中的字符串写入标准输入。我们重复这个，直到我们将所有数据发送到我们的子进程中。当我们运行它时，我们应该看到我们所有的输出都被发送到我们的子进程并正确地回显：

```python
b'Enter text to echo: '
b'one\nEnter text to echo: '
b'two\nEnter text to echo: '
b'three\nEnter text to echo: '
b'four\nEnter text to echo: '
```

我们目前正在运行的应用程序具有产生确定性输出并在确定性点停止以请求输入的奢侈。这使得管理标准输出和标准输入相对简单。如果我们在子进程中运行的应用程序有时只要求输入，或者在要求输入之前可以写入大量数据怎么办？让我们调整我们的示例 echo 程序，使其更复杂一些。我们将让它随机回显用户输入 1 到 10 次，并且我们将在每个回显之间休眠半秒。

清单 13.13 一个更复杂的 echo 应用程序

```python
from random import randrange
import time
 
user_input = ''
 
while user_input != 'quit':
    user_input = input('Enter text to echo: ')
    for i in range(randrange(10)):
        time.sleep(.5)
        print(user_input)
```

如果我们以与清单 13.12 类似的方法将这个应用程序作为子进程运行，它将起作用，因为我们仍然是确定性的，因为我们最终会要求输入一段已知的文本。然而，使用这种方法的缺点是我们从标准输出读取和写入标准输入的代码是强耦合的。这与我们的输入/输出逻辑日益复杂的情况相结合，会使代码难以遵循和维护。

我们可以通过将读取标准输出与将数据写入标准输入分离来解决这个问题，从而分离读取标准输出和写入标准输入的关注点。我们将创建一个协程来读取标准输出和一个协程来将文本写入标准输入。我们读取标准输出的协程将在收到我们期望的输入提示后设置一个事件。我们写入标准输入的协程将等待该事件被设置，然后一旦设置，它将写入指定的文本。然后我们将使用这两个协程并与gather同时运行它们。

清单 13.14 将输出读取与输入写入解耦

```python
import asyncio
from asyncio import StreamWriter, StreamReader, Event
from asyncio.subprocess import Process
 
 
async def output_consumer(input_ready_event: Event, stdout: StreamReader):
    while (data := await stdout.read(1024)) != b'':
        print(data)
        if data.decode().endswith("Enter text to echo: "):
            input_ready_event.set()
async def input_writer(text_data, input_ready_event: Event, stdin: StreamWriter):
    for text in text_data:
        await input_ready_event.wait()
        stdin.write(text.encode())
        await stdin.drain()
        input_ready_event.clear()
 
async def main():
    program = ['python3', 'interactive_echo_random.py']
    process: Process = await asyncio.create_subprocess_exec(*program,
                                                            stdout=asyncio
                                                            .subprocess.PIPE,
                                                            stdin=asyncio
                                                            .subprocess.PIPE)
 
    input_ready_event = asyncio.Event()
 
    text_input = ['one\n', 'two\n', 'three\n', 'four\n', 'quit\n']
 
    await asyncio.gather(output_consumer(input_ready_event, process.stdout),
                         input_writer(text_input, input_ready_event,
                         process.stdin),
                         process.wait())
 
 
asyncio.run(main())
```

在前面的清单中，我们首先定义了一个 output_consumer 协程函数。这个函数接受一个 input_ready 事件以及一个 StreamReader，它将引用标准输出并从标准输出读取，直到我们遇到文本 Enter text to echo:。一旦我们看到这个文本，我们就知道我们的子流程的标准输入已经准备好接受输入，所以我们设置了 input_ready 事件。

我们的 input_writer 协程函数迭代我们的输入列表并等待我们的事件让标准输入准备好。一旦标准输入准备好，我们就写出输入并清除事件，以便在我们的 for 循环的下一次迭代中，我们将阻塞，直到标准输入再次准备好。有了这个实现，我们现在有两个协程函数，每个函数都有一个明确的职责：一个写入标准输入，一个读取标准输出，增加了代码的可读性和可维护性。

## 概括

- 我们可以使用 asyncio 的 subprocess 模块通过 create_subprocess_shell 和 create_subprocess_exec 异步启动子进程。只要有可能，首选 create_subprocess_exec，因为它可以确保跨机器的行为一致。
- 默认情况下，子进程的输出将转到我们自己的应用程序的标准输出。如果我们需要读取标准输入和标准输出并与之交互，我们需要将它们配置为通过管道连接到 StreamReader 和 StreamWriter 实例。
- 当我们管道标准输出或标准错误时，我们需要小心使用输出。如果我们不这样做，我们可能会死锁我们的应用程序。
- 当我们有大量子进程要同时运行时，信号量可以避免滥用系统资源和创建不必要的争用。
- 我们可以使用通信协程方法将输入发送到子进程的标准输入。