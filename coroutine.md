# C++20 Coroutine

新关键字`co_yield`, `co_return`, `co_await`。其中，`co_await`最常用，作为库的使用者基本只需要`co_await`等待值。使用以上三个中任意一个关键字的函数即为协程函数，协程函数必须声明为协程返回类型，例如：

```C++
struct Task{
    struct promise_type {
        Task get_return_object(){
            return Task();
        }
        std::suspend_never initial_suspend(){
            return {};
        }
        std::suspend_never final_suspend()noexcept{
            return {};
        }
        void return_void(){}
        void unhandled_exception(){std::terminate();}
    };
};
```

`promise_type`控制协程的初始化和结束行为。然后以`Task`为返回值类型声明函数，函数中可以使用`co_await`异步等待：

```C++
Task AwaitRun(int init)
{
    int r = co_await AwaitableTask(init);
    std::cout << "result is " << r <<std::endl;
}
```

`co_await`运算符必须接受一个Awaitable类的对象。Awaitable类必须实现`await_ready`、`await_resume`以及`await_suspend`三个方法，例如：

```C++
struct AwaitableTask {
    int init,res;
    AwaitableTask(int init):init(init) {}
    bool await_ready() const noexcept { return false; }
    int await_resume() noexcept { return res; }
    void await_suspend(std::coroutine_handle<> handle)  {
		// do sonething...
    }
};
```

在`co_await AwaitableTask(init);`发生时，程序首先调用`await_ready`方法，如果该方法返回真，那么协程不挂起立即结束；如果返回假，那么协程挂起。挂起就会调用`await_suspend`方法，该方法接收协程句柄`std::coroutine_handle<> handle`，一般需要保存该对象，在任务完成后调用`handle.resume();`以恢复协程。协程恢复时自动调用`await_resume`方法，`co_await AwaitableTask(init)`语句结束，并把`await_resume`的返回值返回出来。

在其他函数中调用`AwaitRun()`以启动协程。协程的启动与`Task`息息相关，`Task` 类作为协程的返回类型，通过其内嵌的 `promise_type` 控制协程的行为：

- **管理协程生命周期**：通过 `initial_suspend()` 和 `final_suspend()` 决定协程启动和结束时的挂起行为。
- **返回值处理**：通过 `return_void()` 处理协程无返回值的情况。
- **异常处理**：通过 `unhandled_exception()` 终止程序以避免未处理异常。
- **协程实例生成**：通过 `get_return_object()` 创建协程的返回对象。

也就是说，当调用协程函数，首先创建协程句柄（包含`promise_type`的对象promise），然后通过 `get_return_object()` 创建协程的返回对象`Task`，然后根据`initial_suspend()`的返回值确定是否立即执行协程（本质上是执行`co_await promise.initial_suspend()`）。`initial_suspend()`返回值可以是`std::suspend_never`或者`std::suspend_always`，前者表示总是不挂起（即立即执行后续代码），后者表示总是挂起。这里使用`std::suspend_never`，表示协程立即执行，这是比较简单的做法。如果选用了`std::suspend_always`，那么需要手动获取协程句柄，然后手动执行`handle.resume()`启动协程。

如果不立即执行，一般需要在`promise_type`的`get_return_object()`函数中获取协程句柄，因为该函数位于`promise_type`内部，可以通过`std::coroutine_handle<promise_type>::from_promise(*this)`获得协程句柄。然后，把该句柄保存在别的什么地方，例如新构造的`Task`类里面（比如成员变量）。然后，在有需要时取用。例如，

```C++
Task create_coro(){
    co_return;
}

int main()
{
    Task t = create_coro();
    t.handle.resume(); // 假设成员变量handle保存了句柄
    // do something...
    if(t.handle)
        t.handle.destroy(); // 销毁也可以在Task析构函数中完成
    return 0;
}
```

上面`std::coroutine_handle<promise_type>::from_promise()`可以通过promise获得句柄，其实也可以通过句柄获取promise（即`handle.promise()`方法返回promise），就可以直接使用`promise_type`的方法或者成员。这里配合`co_yield`，可以编写一个序列生成器。

`co_yield`在执行时，会触发`promise_type`的`yield_value()`方法，相当于`co_await promise.yield_value(value)`。因此，可以据此保存`co_yield`的值，并通过`Task`的方法返回出去。完整的例子如下：

```C++
#include <coroutine>
#include <exception>
#include <iostream>
#include <optional>

template<typename  T>
struct Generator{
    struct promise_type {
        std::optional<T> value;
        Generator get_return_object(){
            return Generator{std::coroutine_handle<promise_type>::from_promise(*this)};
        }
        std::suspend_always initial_suspend(){return {};}
        std::suspend_always final_suspend()noexcept{return {};}
        std::suspend_always yield_value(T v){
            value = v;
            return {};
        }
        void return_void(){}
        void unhandled_exception(){std::terminate();}
    };
    using handle_type = std::coroutine_handle<promise_type>;
    handle_type coro;
    Generator(handle_type h):coro(h){}
    ~Generator(){
        if(coro){
            coro.destroy();
        }
    }

    std::optional<T> next()
    {
        if(coro && !coro.done()){
            coro.resume();
            return coro.promise().value;
        }
        return std::nullopt;
    }
};

Generator<int> seq(int start = 0, int step = 1)
{
    int value = start;
    while (true) { 
        co_yield value;
        value += step;
    
    }
}

int main()
{
    auto gen = seq(2);
    for(int i = 0; i < 10; ++i) {
        std::cout << *gen.next() << " ";
    }
    return 0;
}
```

这里的`promise_type`只使用了`std::suspend_always`，表示只会被手动恢复，不会自动开始。在`gen.next()`执行时，会调用`coro.resume();`恢复协程，然后协程执行遇到`co_yield value`，转换为`co_await promise.yield_value(value)`，首先把值传递到`promise_type`的内部成员`value`，然后再次挂起（挂起后`coro.resume()`函数退出），最后返回`coro.promise().value`，即刚刚`co_yield value`的值。

最后是`co_return`。协程函数可以不返回值，即无显式的`co_return`或者直接`co_return;`，这样会自动调用promise的`return_void()`函数。也可以选择返回一个值，这样会调用`return_value(value)`函数（因此要求`promise_type`的`return_value(T v)`方法有定义）。一般也是在`return_value()`函数中把值保存起来，以后使用。

协程返回后，程序触发`promise_type`的`final_suspend()`方法，相当于`co_await promise.final_suspend()`。这里决定协程是否还要挂起还是直接彻底结束。

# Boost.Coroutines2

Boost.Coroutines2基本只是一个生产者消费者模型。`boost::coroutines2::coroutine<type>`为基本的类，`type`为生产或者消费的值的类型。通过`boost::coroutines2::coroutine<type>::pull_type`或者`boost::coroutines2::coroutine<type>::push_type`构造一个协程，`pull_type/push_type`的构造函数接收一个函数对象（即协程函数），该函数对象接收参数`push_type&/pull_type&`。区别在于`pull_type`会立即执行协程，`push_type`需要手动恢复执行。

正如其名，`pull_type`是消费者，`push_type`是生产者。如果使用`pull_type`构造协程，协程函数立即执行，并且持有一个`push_type&`类型的`yield`。协程函数通过`yield(value)`向外界产生一个值并挂起协程，外界可以使用`pull_type`对象的`get()`方法获取这个值，以及`operator()`恢复协程。

如果使用`push_type`构造协程，协程函数不会立即执行。外界调用`push_type`的`operator()(value)`方法才会执行或者恢复协程。协程函数持有`pull_type&`类型的`yield`，内部通过`yield`的`get()`方法获取外界输入的值，并通过`yield()`挂起协程。