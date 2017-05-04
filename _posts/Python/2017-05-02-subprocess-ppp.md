---
layout: post
title: subprocess--生成额外进程
category: Python
tags: [python]
keywords: python,subprocess
---

---
本篇文章由Sunny翻译自：[subprocess -- Spawning Additional Processes](https://pymotw.com/3/subprocess/index.html)  
相关内容版权归原作者所有，转载请注明出处。   
本文原文地址：http://blog.opsdev.help/2017/05/02/subprocess.html  
同时再次墙裂推荐https://pymotw.com/3/这个网站。再次感谢原作者。
---
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

