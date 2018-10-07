---
published: true
layout: post
categories: cheatsheet
---
在看 https://tell-k.github.io/pyconjp2018/ 的时候，看到有一个库挺有意思的，叫做 atexit，是一个内置的库，主要用来处理程序退出时的工作。

以下例子抄自：https://pymotw.com/3/atexit/ , 比如可以这样用：

```python
import atexit


def all_done():
    print('all_done()')


print('Registering')
atexit.register(all_done)
print('Registered')
```

```
$ python3 atexit_simple.py

Registering
Registered
all_done()
```

当然也可以预设好参数：

```python
import atexit


def my_cleanup(name):
    print('my_cleanup({})'.format(name))


atexit.register(my_cleanup, 'first')
atexit.register(my_cleanup, 'second')
atexit.register(my_cleanup, 'third')
```

或者是用适配器语法：

```python
@atexit.register
def all_done():
    print('all_done()')
```

在以下三种情况下，atexit 注册的函数不会运行：
- 通过 signal 发送结束指令
- 直接调用了 `os._exit()` 方法
- 出现 fatal error
