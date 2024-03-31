### asyncio 的重要组成部分

- 多路复用(select/poll/epoll)
- 回调函数
- Eventloop(核心组件)
- Future
- Task
- Handle(TimerHandle)

### 异步编程模型

- 事件循环 + 回调

### Eventloop(事件循环)是什么

- 一句话: 事件循环就一个循环，一个无限循环

### Eventloop(事件循环)的作用是什么

- 事件循环是 asyncio的心脏，负责处理 IO ，安排协程和回调函数的执行计划，即保存上下文、切换运行协程、恢复上下文、重新运行协程

- 事件循环有三个重要组成部分

  - IO 处理机制，多路复用
  - 任务队列
  - 待安排队列

  ```python
  # base_event.py
  class BaseEventLoop:
    def __init__(self):
      self._ready = deque() # List[Handle]
      self._scheduled = []  # List[TimerHandl]，是一个最小二叉堆
      self._selector = Selector()
      
    def run_until_complete(self, future):
      pass
    
    def create_task(self, coro):
      pass
    
    def _run_once(self):
      pass
    
    def call_soon(self, callback, *args):
      pass
  ```

  

- asyncio 中的协程是如何串联起来的

  `Task._step 和 Task._wakeup`

- 基于生成器的协程, `yield`

  - 可以保存自己的上下文，如运行状态，运行的值

  - 让出控制权
  - 产出一个值
  - 接收一个值
  - 可以多次重新进入从上次让出控制权的地方

- 原生协程, `__await__`或者`__iter__`

- async 是一个关键字，async def 定义的类型还是一个function类型，只有当它被调用时才返回一个协程对象
   `async def` 跟`def`定义的方法在没被调用时没有任何区别，不必看得很神秘，它也可以有`return`语句

- asyncio底层运行都是普通函数，每轮循环只会执行当前循环的就绪队列中的任务(函数)
   
- Async 和 await 只是语法糖，源码底下也是还是用了生成器, 是遇到`yield`关键字才会挂起

  ```python
  def __iter__(self):
    if not self.done():
      self._asyncio_future_blocking = True
      yield self  # This tells Task to wait for completion.
      assert self.done(), "yield from wasn't used with future"
      return self.result()  # May raise too.
    
  # PY35 = sys.version_info >= (3, 5)
  if compat.PY35:
  		__await__ = __iter__ # make compatible with 'await' expression
  ```

  

- 生成器的方法

  - send
  - throw
  - close
  - \_\_next\_\_(next)

- 生成器完结异常:

  `StopIteration`

- Event_loop

  ```python
  # base_event.py
  class BaseEventLoop:
    def __init__(self):
      self._ready = deque() # List[Handle]
      self._scheduled = []  # List[TimerHandl]，是一个最小二叉堆
      self._selector = Selector()
      
    def run_until_complete(self, future):
      future = tasks.ensure_future(future, loop=self)
      future.add_done_callback(_run_until_complete_cb)
      self.run_forever()
      
      
    def run_forever():
      self._run_once()
    
    def create_task(self, coro):
       task = tasks.Task(coro, loop=self)
       return task
    
    def _run_once(self):
      event_list = self._selector.select(timeout)
      self._process_events(event_list)
      ntodo = len(self._ready)
      for i in range(ntodo):
        handle = self._ready.popleft()
        handle._run()
    
    def call_soon(self, callback, *args):
      handle = events.Handle(callback, args, self)
      self._ready.append(handle)
      return handle
    
    def call_at(self, when, callback, *args):
      timer = events.TimerHandle(when, callback, args, self)
      heapq.heappush(self._scheduled, timer)
      timer._scheduled = True
      return timer
  ```

  

- Future

  ```python
  # futures.py
  class Future:
    def __init__(self, loop=None):
      self._loop = loop
      self._callback = [] # List[Function]
      
    def set_result(self, result):
       self._result = result
       self._state = _FINISHED
       for callback in self.callbacks:
          self._loop.call_soon(callback, self)
          
    def __iter__(self):
    	if not self.done():
      	self._asyncio_future_blocking = True
        yield self  # This tells Task to wait for completion.
      assert self.done(), "yield from wasn't used with future"
      return self.result()  # May raise too.
  ```

  

- Task

  ```python
  # tasks.py
  class Task(Future):
    def __init__(self, coro, *, loop=None):
      super().__init__(self, loop)
      self._coro = coro
      self._loop.call_soon(self._step)
      
  	def _step(self, exc=None):
      coro = self._coro
      try:
      	result = coro.send(None)
      except StopIteration as exc:
        self.set_result(exc.value)
      else:
        result.add_done_callback(self._wakeup)
      
    
    def _wakeup(self, future):
      self._step()
  ```

- Handle

  ```python
  # events.py
  class Handle:
      def __init__(self, callback, args, loop):
          self._loop = loop
          self._callback = callback
          self._args = args
          
  		def _run(self):
        self._callback(*self._args)
      
      
  class TimerHandle(Handle):
      def __init__(self, when, callback, args, loop):
          assert when is not None
          super().__init__(callback, args, loop)
          self._when = when
          self._scheduled = False
  ```

  

### 代码执行流程

```python
import asyncio


async def cor():
    print('enter cor ...')
    await asyncio.sleep(2)
    print('exit cor ...')
    
    return 'cor'

loop = asyncio.get_event_loop()
task = loop.create_task(cor())
rst = loop.run_until_complete(task)
print(rst)
```

```python
class Task(futures.Future):
    
    ...
    
    def _step(self, exc=None):
        """
        _step方法可以看做是task包装的coroutine对象中的代码的直到yield的前半部分逻辑
        """
        ...
        try:
            if exc is None:
                
                # 1.关键代码
                result = coro.send(None)
            else:
                result = coro.throw(exc)
        # 2. coro执行完毕会抛出StopIteration异常
        except StopIteration as exc:
            if self._must_cancel:
                # Task is cancelled right before coro stops.
                self._must_cancel = False
                self.set_exception(futures.CancelledError())
            else:
                # result为None时，调用task的callbasks列表中的回调方法，在调用loop.run_until_complite，结束loop循环
                self.set_result(exc.value)
        except futures.CancelledError:
            super().cancel()  # I.e., Future.cancel(self).
        except Exception as exc:
            self.set_exception(exc)
        except BaseException as exc:
            self.set_exception(exc)
            raise
        # 3. result = coro.send(None)不抛出异常
        else:
            # 4. 查看result是否含有_asyncio_future_blocking属性
            blocking = getattr(result, '_asyncio_future_blocking', None)
            if blocking is not None:
                # Yielded Future must come from Future.__iter__().
                if result._loop is not self._loop:
                    self._loop.call_soon(
                        self._step,
                        RuntimeError(
                            'Task {!r} got Future {!r} attached to a '
                            'different loop'.format(self, result)))
                
                elif blocking:
                    if result is self:
                        self._loop.call_soon(
                            self._step,
                            RuntimeError(
                                'Task cannot await on itself: {!r}'.format(
                                    self)))
                    # 4.1. 如果result是一个future对象时，blocking会被设置成true
                    else:
                        result._asyncio_future_blocking = False
                        # 把_wakeup回调函数设置到此future对象中，当此future对象调用set_result()方法时，就会调用_wakeup方法
                        result.add_done_callback(self._wakeup)
                        self._fut_waiter = result
                        if self._must_cancel:
                            if self._fut_waiter.cancel():
                                self._must_cancel = False
                else:
                    self._loop.call_soon(
                        self._step,
                        RuntimeError(
                            'yield was used instead of yield from '
                            'in task {!r} with {!r}'.format(self, result)))
            # 5. 如果result是None，则注册task._step到loop对象中去，在下一轮_run_once中被回调
            elif result is None:
                # Bare yield relinquishes control for one event loop iteration.
                self._loop.call_soon(self._step)

            # --------下面的代码可以暂时不关注了--------
            elif inspect.isgenerator(result):
                # Yielding a generator is just wrong.
                self._loop.call_soon(
                    self._step,
                    RuntimeError(
                        'yield was used instead of yield from for '
                        'generator in task {!r} with {}'.format(
                            self, result)))
            else:
                # Yielding something else is an error.
                self._loop.call_soon(
                    self._step,
                    RuntimeError(
                        'Task got bad yield: {!r}'.format(result)))
        finally:
            self.__class__._current_tasks.pop(self._loop)
            self = None  # Needed to break cycles when an exception occurs.

    def _wakeup(self, future):
        try:
            future.result()
        except Exception as exc:
            # This may also be a cancellation.
            self._step(exc)
        else:
            
            # 这里是关键代码，上次的_step()执行到第一次碰到yield的地方挂住了，此时再次执行_step(),
            # 也就是再次执行 result = coro.send(None) 这句代码，也就是从上次yield的地方继续执行yield后面的逻辑
            self._step()
        self = None  # Needed to break cycles when an exception occurs.
```





`run_until_complete()` --> `run_forever()` --> `_run_once()`，重点看`_run_once`这个方法的执行

此时:

- `cor`协程还未开始执行
- `loop._ready = [handle(task._step)]`，`loop._scheduled = [], task._callbacks=[_run_until_complete_cb]`

#### 第一轮`_run_once()`的调用执行开始

此时:

- `cor`协程的执行流程挂起在`sleep`协程的中产生的新`Future`对象的`__iter__`方法的`yield`处
- 新`Future`对象的`_callbacks = [task._wakeup,]`
- `loop._scheduled=[handle(delay_2s__set_result_unless_cancelled)]`，`loop._ready=[]`

#### 第二轮`_run_once()`的调用执行开始

此时:

- `cor`协程的执行流程挂起在`sleep`协程的中产生的新`Future`对象的`__iter__`方法的`yield`处。

- 新`Future`对象的`_callbacks = []`
- `loop._ready = [handle(task._wakeup)]`， `loop._scheduled=[]`

#### 第三轮`_run_once()`的调用执行开始

此时:

- `cor`协程的执行完毕。

-  新`Future`对象的`_callbacks = []`
- `loop._ready = [handle(_run_until_complete_cb)]`， `loop._scheduled=[]`

#### 第四轮`_run_once()`开始执行

跳出`while`循环， `run_forever()`执行结束，`run_until_complete()`也就执行完毕了，最后把`cor`协程的返回值'cor'返回出来赋值给`rst`变量。

到此为止所有整个`task`任务执行完毕，`loop`循环关闭



### 总结

Asyncio 就是利用操作系统 的多路复用机制处理 IO 阻塞，通过就绪队列`loop._ready` 以及延迟最小堆`loop._scheduled`存放回调函数，通过生成器来让分散的回调代码变成逻辑同步代码, 然后在时间循环中通过`Task._step`和`Task._wakeup`进行完成一整个调用逻辑链路