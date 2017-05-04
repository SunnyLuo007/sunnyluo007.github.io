---
layout: post
title: (译)subprocess--生成额外进程
category: Python
tags: [python]
keywords: python,subprocess
---

> 本篇文章由Sunny翻译自：[subprocess -- Spawning Additional Processes](https://pymotw.com/3/subprocess/index.html)  
> 相关内容版权归原作者所有，转载请注明出处。   
> 本文原文地址：http://blog.opsdev.help/2017/05/02/subprocess-translate.html  
> 同时再次墙裂推荐https://pymotw.com/3/这个网站。再次感谢原作者。

**用途：** 启动进程和与其他进程通信
### 前言
*subprocess*模块提供三个API来和进程工作。 在Python 3.5中添加的run()函数是一个高阶API，用于运行进程并可选地收集其输出。  
函数call()，check_call() 和 check_output()是从Python 2继承的高级API。它们仍然被支持，并广泛地用于现有程序中。  
Popen 类是用于构建其他API的低级API，对于更复杂的进程间交互有用。Popen的构造函数使用参数来设置新进程，以便父进程可以通过管道与其进行通信。它提供了所有其他模块的功能以及它所替代的功能（译者注：这里其他模块指os和popen2等拥有类似功能的模块）。  
API对于所有用途是一致的，并将很多额外的步骤（如：关闭额外的文件描述符以确保管道关闭）都内部处理了，不需要在应用代码里单独处理。  

*subprocess*模块旨在替代在os与popen2模块中的osystem(), spawnv()和变体popen()，还有commands()。为了更容易与这两个模块进行比较，本章中的许多示例将重新创建使用os和popen2的示例。
> 注意：  
> API在Unix和Windows下基本相同，但底层的实现是不一样的，因为不同的操作系统处理模式不同。以下所有的示例是在Mac OS X环境下测试的。在非Unix操作系统下可能会有不同。

### 执行外部命令
无交互地执行外部命令可使用*run()*,等同于*os.system()*

```py
#subprocess_os_system.py
import subprocess

completed = subprocess.run(['ls', '-1'])
print('returncode:', completed.returncode)
```
命令行作为一个字符串列表形式传递给参数，避免了可能会由shell解释器转义引号或其他特殊字符的需要。（译者注：shell=True时，也可以直接将整个命令行以字符串形式传入。）  
run()返回一个CompletedProcess实例，其中包含相关的处理信息，如退出代码和输出。
```bash
$ python3 subprocess_os_system.py

index.rst
interaction.py
repeater.py
signal_child.py
signal_parent.py
subprocess_check_output_error_trap_output.py
subprocess_os_system.py
subprocess_pipes.py
subprocess_popen2.py
subprocess_popen3.py
subprocess_popen4.py
subprocess_popen_read.py
subprocess_popen_write.py
subprocess_run_check.py
subprocess_run_output.py
subprocess_run_output_error.py
subprocess_run_output_error_suppress.py
subprocess_run_output_error_trap.py
subprocess_shell_variables.py
subprocess_signal_parent_shell.py
subprocess_signal_setpgrp.py
returncode: 0
```
*shell*参数设置为true时,会导致*subprocess*产生一个中间shell进程，然后运行该命令。 默认是直接运行命令。（译者注：后续示例会看到，可以设置在执行命令前执行其他功能）

```py
#subprocess_shell_variables.py

import subprocess

completed = subprocess.run('echo $HOME', shell=True)
print('returncode:', completed.returncode)
```
使用中间shell意味着命令行中的变量，glob模式和其他shell特性在运行命令之前被处理。

```py
$ python3 subprocess_shell_variables.py

/Users/dhellmann
returncode: 0
```
> **备注：**  
> 使用*run()*时，如果没有设置参数*check=True*，就相当于*call()*。这将只会返回程序的退出代码。

### 错误处理
returncode是CompletedProcess的一个属性，保存程序退出代码。调用者负责解释它，用于检查错误。如果*run()*的*check*参数设置为True，退出代码会被（自动）检测，并且如果退出代码表示有错误发生，那么程序会抛出CalledProcessError异常。

```py
#subprocess_run_check.py
import subprocess

try:
    subprocess.run(['false'], check=True)
except subprocess.CalledProcessError as err:
    print('ERROR:', err)
```
执行失败的命令总是以非零的状态码退出，*run()*会解释为一个错误。

```bash
$ python3 subprocess_run_check.py

ERROR: Command '['false']' returned non-zero exit status 1
```
> **备注：**  
> *run()*设置参数*check=True*，等价于check_all()。

### 捕获输出

由run()启动的进程的标准输入、输出通道会被绑定到父进程的输入、输出上。这意味着调用程序不能捕获命令行的输出。为stdout和stderr参数传递PIPE来捕获输出用于后续处理。

```py
#subprocess_run_output.py
import subprocess

completed = subprocess.run(
    ['ls', '-1'],
    stdout=subprocess.PIPE,
)
print('returncode:', completed.returncode)
print('Have {} bytes in stdout:\n{}'.format(
    len(completed.stdout),
    completed.stdout.decode('utf-8'))
)
```
ls -l命令运行成功,所有输出到标准输出的文本被捕获并返回。

```bash
$ python3 subprocess_run_output.py

returncode: 0
Have 522 bytes in stdout:
index.rst
interaction.py
repeater.py
signal_child.py
signal_parent.py
subprocess_check_output_error_trap_output.py
subprocess_os_system.py
subprocess_pipes.py
subprocess_popen2.py
subprocess_popen3.py
subprocess_popen4.py
subprocess_popen_read.py
subprocess_popen_write.py
subprocess_run_check.py
subprocess_run_output.py
subprocess_run_output_error.py
subprocess_run_output_error_suppress.py
subprocess_run_output_error_trap.py
subprocess_shell_variables.py
subprocess_signal_parent_shell.py
subprocess_signal_setpgrp.py

```
> **备注：**  
> 设置check=True和stdout=PIPE等价于使用check_output()。

下一个示例在子shell中运行一系列命令。 在命令退出并发出错误代码之前，消息将发送到标准输出和标准错误。(译者注:请通过命令行执行脚本，IDE环境下可能会有不同结果)

```py
#subprocess_run_output_error.py
import subprocess

try:
    completed = subprocess.run(
        'echo to stdout; echo to stderr 1>&2; exit 1',
        check=True,
        shell=True,
        stdout=subprocess.PIPE,
    )
except subprocess.CalledProcessError as err:
    print('ERROR:', err)
else:
    print('returncode:', completed.returncode)
    print('Have {} bytes in stdout: {!r}'.format(
        len(completed.stdout),
        completed.stdout.decode('utf-8'))
    )
```
标准错误消息将打印到终端上，但标准输出消息被隐藏。
```bash
$ python3 subprocess_run_output_error.py

to stderr
ERROR: Command 'echo to stdout; echo to stderr 1>&2; exit 1'
returned non-zero exit status 1
```
为了防止通过run()运行的命令的错误消息写入控制台，请将stderr参数设置为常量PIPE。

```py
#subprocess_run_output_error_trap.py
import subprocess

try:
    completed = subprocess.run(
        'echo to stdout; echo to stderr 1>&2; exit 1',
        shell=True,
        stdout=subprocess.PIPE,
        stderr=subprocess.PIPE,
    )
except subprocess.CalledProcessError as err:
    print('ERROR:', err)
else:
    print('returncode:', completed.returncode)
    print('Have {} bytes in stdout: {!r}'.format(
        len(completed.stdout),
        completed.stdout.decode('utf-8'))
    )
    print('Have {} bytes in stderr: {!r}'.format(
        len(completed.stderr),
        completed.stderr.decode('utf-8'))
    )
```
该示例没有设置check=True，所以命令的输出被捕获并打印。
```bash
$ python3 subprocess_run_output_error_trap.py

returncode: 1
Have 10 bytes in stdout: 'to stdout\n'
Have 10 bytes in stderr: 'to stderr\n'
```
为了在使用check_output()时捕获错误消息，设置stderr=STDOUT，(错误)消息将会与命令其他的消息合并输出。（译者注：相当于2>&1）

```py
# subprocess_check_output_error_trap_output.py
import subprocess

try:
    output = subprocess.check_output(  #注意这里使用的是check_output--译者注
        'echo to stdout; echo to stderr 1>&2',
        shell=True,
        stderr=subprocess.STDOUT,
    )
except subprocess.CalledProcessError as err:
    print('ERROR:', err)
else:
    print('Have {} bytes in output: {!r}'.format(
        len(output),
        output.decode('utf-8'))
    )
```
输出顺序可能会变化，这取决于缓冲如何应用于标准输出流和数据量。
```bash
$ python3 subprocess_check_output_error_trap_output.py

Have 20 bytes in output: 'to stdout\nto stderr\n'
```
### 压制输出
对于不显示或捕获输出的情况，使用DEVNULL来压制输出流。 此示例抑制标准输出和错误流。
```py
# subprocess_run_output_error_suppress.py
import subprocess

try:
    completed = subprocess.run(
        'echo to stdout; echo to stderr 1>&2; exit 1',
        shell=True,
        stdout=subprocess.DEVNULL,
        stderr=subprocess.DEVNULL,
    )
except subprocess.CalledProcessError as err:
    print('ERROR:', err)
else:
    print('returncode:', completed.returncode)
    print('stdout is {!r}'.format(completed.stdout))
    print('stderr is {!r}'.format(completed.stderr))
```
DEVNULL名称来源于Unix特殊设备文件：/dev/null。当它被打开用于读取和接收时，会响应end-of-file（译者注：即读取它则会立即得到一个EOF）。而当写入时，会忽略所有的输入。
```bash
$ python3 subprocess_run_output_error_suppress.py

returncode: 1
stdout is None
stderr is None
```
### 直接使用管道
函数run(), call(), check_call()和check_output() 包装了Popen类。  直接使用Popen可以更好地控制命令的运行方式，以及如何处理其输入和输出流。 例如，通过设置stdin，stdout和stderr的不同参数，可以模拟os.popen()。
#### 与程序单向通信

```
#subprocess_popen_read.py
import subprocess

print('read:')
proc = subprocess.Popen(
    ['echo', '"to stdout"'],
    stdout=subprocess.PIPE,
)
stdout_value = proc.communicate()[0].decode('utf-8')
print('stdout:', repr(stdout_value))
```
这个类似于popen()的工作方式，只是读取由Popen实例内部管理。

```bash
$ python3 subprocess_popen_read.py

read:
stdout: '"to stdout"\n'
```
要设置一个管道以允许调用程序向其写入数据，可将stdin设置为PIPE。
```py
#subprocess_popen_write.py
import subprocess

print('write:')
proc = subprocess.Popen(
    ['cat', '-'],
    stdin=subprocess.PIPE,
)
proc.communicate('stdin: to stdin\n'.encode('utf-8'))
```
要将数据一次性发送到进程的标准输入通道，将数据传递给communication()。 这与使用'w'模式的popen()类似。
```bash
$ python3 -u subprocess_popen_write.py

write:
stdin: to stdin
```
#### 与程序双向通信
要设置同时读和写的Popen实例，将前面的技术组合就行。

```py
#subprocess_popen2.py
import subprocess

print('popen2:')

proc = subprocess.Popen(
    ['cat', '-'],
    stdin=subprocess.PIPE,
    stdout=subprocess.PIPE,
)
msg = 'through stdin to stdout'.encode('utf-8')
stdout_value = proc.communicate(msg)[0].decode('utf-8')
print('pass through:', repr(stdout_value))
```
这与popen2()类似。

```bash
$ python3 -u subprocess_popen2.py

popen2:
pass through: 'through stdin to stdout'
```
#### 捕获错误输出
也可以像popen3()一样，同时查看stdout和stderr。

```py
#subprocess_popen3.py
import subprocess

print('popen3:')
proc = subprocess.Popen(
    'cat -; echo "to stderr" 1>&2',
    shell=True,
    stdin=subprocess.PIPE,
    stdout=subprocess.PIPE,
    stderr=subprocess.PIPE,
)
msg = 'through stdin to stdout'.encode('utf-8')
stdout_value, stderr_value = proc.communicate(msg)
print('pass through:', repr(stdout_value.decode('utf-8')))
print('stderr      :', repr(stderr_value.decode('utf-8')))
```
从stderr读取与stdout相同。 传递PIPE告诉Popen附加到管道，communicate()读取完数据后返回。

```bash
$ python3 -u subprocess_popen3.py

popen3:
pass through: 'through stdin to stdout'
stderr      : 'to stderr\n'
```
#### 组合规则和错误输出
让错误信息从标准输出通道输出，可以将STDOUT代替PIPE传递给stderr参数。

```py
#subprocess_popen4.py
import subprocess

print('popen4:')
proc = subprocess.Popen(
    'cat -; echo "to stderr" 1>&2',
    shell=True,
    stdin=subprocess.PIPE,
    stdout=subprocess.PIPE,
    stderr=subprocess.STDOUT,
)
msg = 'through stdin to stdout\n'.encode('utf-8')
stdout_value, stderr_value = proc.communicate(msg)
print('combined output:', repr(stdout_value.decode('utf-8')))
print('stderr value   :', repr(stderr_value))
```
通过这种方式组合输出，工作方式类似于popen4()。
```bash
$ python3 -u subprocess_popen4.py

popen4:
combined output: 'through stdin to stdout\nto stderr\n'
stderr value   : None
```
#### 连接管道分段
（译者注：级联管道或者串联可能更好理解一些）  
通过创建单独的Popen实例并将其输入和输出链接在一起，可以将多个命令连接到管道线(pipeline)，类似于Unix shell的工作方式。 一个Popen实例的stdout属性用作管道中下一个的stdin参数，而不是常量PIPE。从最后命令的stdout句柄中读取输出结果。
#
```py
subprocess_pipes.py
import subprocess

cat = subprocess.Popen(
    ['cat', 'index.rst'],
    stdout=subprocess.PIPE,
)

grep = subprocess.Popen(
    ['grep', '.. literalinclude::'],
    stdin=cat.stdout,
    stdout=subprocess.PIPE,
)

cut = subprocess.Popen(
    ['cut', '-f', '3', '-d:'],
    stdin=grep.stdout,
    stdout=subprocess.PIPE,
)

end_of_pipe = cut.stdout

print('Included files:')
for line in end_of_pipe:
    print(line.decode('utf-8').strip())
```
该示例与以下命令行等效：

```bash
$ cat index.rst | grep ".. literalinclude" | cut -f 3 -d:
```
pipeline读取reStructuredText源文件，并查找包含其他文件的所有行，然后打印所包含的文件名称。
```bash
$ python3 -u subprocess_pipes.py

Included files:
subprocess_os_system.py
subprocess_shell_variables.py
subprocess_run_check.py
subprocess_run_output.py
subprocess_run_output_error.py
subprocess_run_output_error_trap.py
subprocess_check_output_error_trap_output.py
subprocess_run_output_error_suppress.py
subprocess_popen_read.py
subprocess_popen_write.py
subprocess_popen2.py
subprocess_popen3.py
subprocess_popen4.py
subprocess_pipes.py
repeater.py
interaction.py
signal_child.py
signal_parent.py
subprocess_signal_parent_shell.py
subprocess_signal_setpgrp.py
```
### 与其他命令交互
之前的例子都假定交互次数是有限的。 communication()方法读取所有输出，并在返回之前等待子进程退出。  
当程序运行时，也可以增量写入和读取Popen实例使用的各个管道句柄。  
一个简单的echo程序可以说明这一技术，即从stdin读取然后再输出到stdout。

脚本repeater.py用作下一个示例中的子进程。 它从stdin读取并将值写入stdout，一次一行，直到没有更多的输入。 它还在stderr启动和停止时写入一条消息，显示子进程的生命周期。
```py
#repeater.py
import sys

sys.stderr.write('repeater.py: starting\n')
sys.stderr.flush()

while True:
    next_line = sys.stdin.readline()
    sys.stderr.flush()
    if not next_line:
        break
    sys.stdout.write(next_line)
    sys.stdout.flush()

sys.stderr.write('repeater.py: exiting\n')
sys.stderr.flush()
```
下一个交互示例以不同的方式使用Popen实例拥有的stdin和stdout文件句柄。 在第一个例子中，将五个数字的序列写入进程的stdin，每次写入后，下一行输出被读回。 在第二个例子中，写入相同的五个数字，但是使用communic()一次读取所有输出。
```
#interaction.py
import io
import subprocess

print('One line at a time:')
proc = subprocess.Popen(
    'python3 repeater.py',
    shell=True,
    stdin=subprocess.PIPE,
    stdout=subprocess.PIPE,
)
stdin = io.TextIOWrapper(
    proc.stdin,
    encoding='utf-8',
    line_buffering=True,  # send data on newline
)
stdout = io.TextIOWrapper(
    proc.stdout,
    encoding='utf-8',
)
for i in range(5):
    line = '{}\n'.format(i)
    stdin.write(line)
    output = stdout.readline()
    print(output.rstrip())
remainder = proc.communicate()[0].decode('utf-8')
print(remainder)

print()
print('All output at once:')
proc = subprocess.Popen(
    'python3 repeater.py',
    shell=True,
    stdin=subprocess.PIPE,
    stdout=subprocess.PIPE,
)
stdin = io.TextIOWrapper(
    proc.stdin,
    encoding='utf-8',
)
for i in range(5):
    line = '{}\n'.format(i)
    stdin.write(line)
stdin.flush()

output = proc.communicate()[0].decode('utf-8')
print(output)
```
"repeater.py: exiting"在不同循环样式中出现时间点不同。

```py
$ python3 -u interaction.py

One line at a time:
repeater.py: starting
0
1
2
3
4
repeater.py: exiting


All output at once:
repeater.py: starting
repeater.py: exiting
0
1
2
3
4
```
### 进程间信令
[os](https://pymotw.com/3/os/index.html#module-os)模块的进程管理示例包括使用os.fork()和os.kill()的进程之间的信令演示。 由于每个Popen实例都提供了子进程的pid属性，因此可以对子进程执行类似的操作。 下一个示例组合了两个脚本。 下面这个（脚本）子进程用于USR信号的处理。

```py
#signal_child.py
import os
import signal
import time
import sys

pid = os.getpid()
received = False


def signal_usr1(signum, frame):
    "Callback invoked when a signal is received"
    global received
    received = True
    print('CHILD {:>6}: Received USR1'.format(pid))
    sys.stdout.flush()


print('CHILD {:>6}: Setting up signal handler'.format(pid))
sys.stdout.flush()
signal.signal(signal.SIGUSR1, signal_usr1)
print('CHILD {:>6}: Pausing to wait for signal'.format(pid))
sys.stdout.flush()
time.sleep(3)

if not received:
    print('CHILD {:>6}: Never received signal'.format(pid))
```
下面这个的脚本作为父进程运行。 它启动signal_child.py，然后发送USR1信号。

```py
#signal_parent.py
import os
import signal
import subprocess
import time
import sys

proc = subprocess.Popen(['python3', 'signal_child.py'])
print('PARENT      : Pausing before sending signal...')
sys.stdout.flush()
time.sleep(1)
print('PARENT      : Signaling child')
sys.stdout.flush()
os.kill(proc.pid, signal.SIGUSR1)
```
输出：
```bash
$ python3 signal_parent.py

PARENT      : Pausing before sending signal...
CHILD  26976: Setting up signal handler
CHILD  26976: Pausing to wait for signal
PARENT      : Signaling child
CHILD  26976: Received USR1
```
### 进程组和会话
如果由Popen创建的进程产生子进程，那么这些子进程将不会收到发送给父进程的任何信号。 这意味着当使用popen的shell参数时，通过发送SIGINT或SIGTERM很难使shell中启动的命令终止。

```py
#subprocess_signal_parent_shell.py
import os
import signal
import subprocess
import tempfile
import time
import sys

script = '''#!/bin/sh
echo "Shell script in process $$"
set -x
python3 signal_child.py
'''
script_file = tempfile.NamedTemporaryFile('wt')
script_file.write(script)
script_file.flush()

proc = subprocess.Popen(['sh', script_file.name])
print('PARENT      : Pausing before signaling {}...'.format(
    proc.pid))
sys.stdout.flush()
time.sleep(1)
print('PARENT      : Signaling child {}'.format(proc.pid))
sys.stdout.flush()
os.kill(proc.pid, signal.SIGUSR1)
time.sleep(3)
```
发送信号的目标pid与等待信号的shell脚本的子进程的pid不匹配，因为在此示例中有三个单独的进程进行交互：
1. subprocess_signal_parent_shell.py进程
2. 运行由主python程序创建的脚本的shell进程
3. signal_child.py进程

```bash
$ python3 subprocess_signal_parent_shell.py

PARENT      : Pausing before signaling 26984...
Shell script in process 26984
+ python3 signal_child.py
CHILD  26985: Setting up signal handler
CHILD  26985: Pausing to wait for signal
PARENT      : Signaling child 26984
CHILD  26985: Never received signal
```
要发送信号到后代却不知道他们的进程id，可以使用一个进程组关联这些子进程，使他们可以一起接收信号。使用os.setpgrp()创建进程组，它会将当前进程id设置为进程组id。 所有子进程从其父进程继承进程组，并且由于它只应在由Popen及其后代创建的shell中进行设置，所以在创建Popen的同一进程中不应调用os.setpgrp()。 正确做法是，一个函数作为preexec_fn参数传递给Popen，该函数在fork()出子进程后，exec()运行shell之前执行。 要发信号通知整个进程组，请使用os.killpg()和Popen实例中的pid值。
```py
#subprocess_signal_setpgrp.py
import os
import signal
import subprocess
import tempfile
import time
import sys


def show_setting_prgrp():
    print('Calling os.setpgrp() from {}'.format(os.getpid()))
    os.setpgrp()
    print('Process group is now {}'.format(
        os.getpid(), os.getpgrp()))
    sys.stdout.flush()


script = '''#!/bin/sh
echo "Shell script in process $$"
set -x
python3 signal_child.py
'''
script_file = tempfile.NamedTemporaryFile('wt')
script_file.write(script)
script_file.flush()

proc = subprocess.Popen(
    ['sh', script_file.name],
    preexec_fn=show_setting_prgrp,
)
print('PARENT      : Pausing before signaling {}...'.format(
    proc.pid))
sys.stdout.flush()
time.sleep(1)
print('PARENT      : Signaling process group {}'.format(
    proc.pid))
sys.stdout.flush()
os.killpg(proc.pid, signal.SIGUSR1)
time.sleep(3)
```
执行顺序如下：
1. 父进程实例化Popen
2. Popen实例fork一个新进程
3. 新进程运行os.setpgrp()
4. 新进程运行exec()启动shell
5. shell运行shell脚本
6. shell脚本fork新进程，该进程执行Python
7. Python运行signal_child.py
8. 父进程使用shell的pid发送信号给进程组
9. shell忽略信号
10. 运行signal_child.py的Python进程调用信号处理函数

```bash
$ python3 subprocess_signal_setpgrp.py

Calling os.setpgrp() from 26992
Process group is now 26992
PARENT      : Pausing before signaling 26992...
Shell script in process 26992
+ python3 signal_child.py
CHILD  26993: Setting up signal handler
CHILD  26993: Pausing to wait for signal
PARENT      : Signaling process group 26992
CHILD  26993: Received USR1
```

---
### 参考资料
[subprocess的标准库文档](https://docs.python.org/3.5/library/subprocess.html)  
[os](https://pymotw.com/3/os/index.html#module-os) - 尽管subprocess取代了它很多，但在os模块中使用的进程的功能仍然被广泛地应用于现有的代码中。  
[UNIX Signals and Process Groups ](http://www.cs.ucsb.edu/~almeroth/classes/W99.276/assignment1/signals.html)– 很好的描述了关于Unix信令和进程组如何工作。  
[signal](https://pymotw.com/3/signal/index.html#module-signal) – 使用signal模块的更多细节。  
unix环境高级编程 - 覆盖多进程，如：处理信号，关闭重复文件描述符，等等（经典好书）  
[pipes](https://docs.python.org/3/library/pipes.html) - 标准库中的Unix shell命令pipeline模板
---
