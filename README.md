# 轻量级MUDUO网络库

## 介绍
本库是参考《Linux多线程服务端编程》书中的代码，逐步重新实现了一个轻量级的Muduo网络库。实现的功能相对MUDUO而言较少。我引入了C++11标准的内容，替代了书中许多依赖boost的组件和自定义的基础库函数，如thread、mutex、lock_guard、condition_variable、bind、function、atomic等。


## 文件介绍

###
* ./Base: 基础代码，如CurrentThread Timestamp等
* ./Net: 网络相关代码，如重要的Reactor三剑客EventLoop/Channel/Poller
* ./TCP: 处理TCP连接的代码

##

## 个人理解
### 1.muduo的Reactor模型实现

EventLoop 是一个事件循环，其核心功能集中在 loop() 成员函数中。在 while 循环中，poller_->poll() 会等待事件的到来或超时，然后将所有活跃的文件描述符 (fd) 存储在 activeChannels_ 中。接下来，程序遍历这个数组，调用绑定的回调函数来处理相应的事件，最后处理 doPendingFunctors()，这是由 queueInLoop添加的任务。

在 Muduo 中，Poller 被设计为一个抽象类，它有两个派生类：PollPoller 和 EPollPoller，它们分别调用 poll 和 epoll 进行多路复用。后文中提到的 Poller 是 PollPoller 和 EPollPoller 的结合体。Poller 的核心功能函数是 poll()。在 poll() 函数中，程序会阻塞等待 epoll_wait，然后通过 fillActiveChannels() 将所有已经激活的 fd 保存到 activeChannels_ 中，并在 fillActiveChannels() 中设置 revents。需要注意的是，epoll_wait 可能因多种原因返回，例如监听超时、注册的 fd 上有事件发生、定时器超时或其他线程添加了任务并触发了 eventfd。

另一个重要的类是 Channel，它封装了一个 fd（Muduo 使用了统一的事件源，可以是 socket fd、timerfd、eventfd、signalfd），关注的事件是 events，实际发生的事件是 revents（对应 fillActiveChannels() 中的修改内容），以及每个事件对应的回调函数。Channel 中最重要的功能是 handleEvent()，它会根据发生的事件执行相应的回调函数。

EventLoop 持有两个关键组件：Channel 和 Poller。其中，Channel 是直接由 EventLoop 持有的，EventLoop 负责管理其生命周期；而 Poller 则是通过 unique_ptr 间接持有，它的生命周期与 EventLoop 相同。
在这个架构中，Channel 并不能直接调用 Poller。相反，Channel 通过 EventLoop 间接与 Poller 进行交互。具体来说，当 Channel::update() 被调用时，它会调用 loop_->updateChannel(this)，然后再由 EventLoop 调用 poller_->updateChannel(channel)。通过这一调用链，完成了在 epoll 上对文件描述符 (fd) 的注册和修改操作。

### 2.多线程思想
EventLoop 对象的生命周期和其所属的线程一样长，它不必是 heap 对象。muduo 中的部分成员函数只能在 IO 线程调用，所以用 isInLoopThread() 和 assertInLoopThread() 等函数进行检查和断言。
Channel 对象始终只会属于一个 EventLoop，属于一个 IO 线程。Channel 会持有一个 fd，但它并不拥有这个 fd，也就是在析构的的时候不会负责关闭该 fd。Channel 的生命周期由 TcpConnection 负责。Channel 的成员函数都只能在 IO 线程调用，因此更新数据成员不必加锁。
Poller 是 EventLoop 的间接成员，EventLoop 通过 unique_ptr 持有，只供其所属的 EventLoop 在 IO 线程调用，因此无需加锁，其生命周期与 EventLoop 相等。
TimerQueue 同样是只会在所属的 IO 线程调用，所以也是不用加锁。TimerQueue 的构造函数中会将 timerfdChannel_ 的读事件注册到其所属的 EventLoop 的 Poller 中。
runInLoop() 配合 queueInLoop() 实现了跨线程的函数转移调用，这涉及了函数参数的跨线程转移，最简单的实现就是将数据拷贝一份，绝对安全但是会有性能损失。queueInLoop() 有两种情况会唤醒 IO 线程，一是调用 queueInLoop() 的线程不是 IO 线程，二是调用 queueInLoop() 的线程是 IO 线程，但是此时位于 doPendingFunctors() 中。
Acceptor 接受新连接使用了最近但的一次接受一个连接的方式，还有循环接受连接直到没有新连接，或者每次接受 N 个连接，后两种策略是为短链接服务准备的，而 muduo 的目标是为长连接服务。
Buffer 底层以 std::vector 作为容器，是一块连续的可以自动扩容的内存，对外表现出 queue 的特性。Buffer 不是线程安全的，Buffer 是属于 TcpConnection 的，需要保证 TcpConnection 在自己所属的线程执行。
sendInLoop() 和 handleWrite() 都只调用了一次 write() 而不会反复调用直至它返回 EAGAIN，是因为如果第一次 write() 没有能够发送完数据的话，第二次调用几乎肯定会返回 EAGAIN。
TCP No Delay 和 TCP keepalive 都是常用的 TCP 选项，前者用来禁用 Nagle 算法，避免连续发包出现延迟，这对于编写低延迟的网络服务很重要。后者是定期探查 TCP 连接是否还存在。
epoll 在并发连接数较大而活动连接比不高的时候，比 poll 高效。

### 3.ET和LT的理解
LT 模式是在高电平时触发。连接建立后，只要读缓冲区有数据，就会一直触发可读事件，所以 LT 模式下，处理读事件时可以只读一次，后续读事件还会被触发，不用担心数据漏读。但是只要连接一建立，可写事件就会被触发（刚开始写缓冲区肯定为空），但是服务端又没有可以发送的数据，这会造成 CPU 资源的浪费（busy loop），所以 LT 模式下，只有需要写数据的时候才注册写事件，等到写完立刻将写事件注销掉。
ET 模式是低电平到高电平时触发。读事件只有在数据到来时被触发一次，所以 ET 模式下，读事件要一次性处理完，否则会造成数据漏读。写事件可以在一开始注册，此时不会被触发，需要发送数据时直接发送即可，但是要注意如果一次性写不完，缓冲区就会变成低电平，要重新注册写事件，等到内核将缓冲区数据发出，从低电平变为高电平，会再次触发写事件，发送剩余的数据。
muduo 为什么选择 LT 模式：一是 LT 模式编程更容易，并且不会出现漏掉事件的 bug，也不会漏读数据。二是对于 LT 模式，如果可以将数据一次性读取完，那么就和 ET 相同，也只会触发一次读事件，另外LT 模式下一次性读取完数据只会调用一次 read()，而 ET 模式至少需要两次，因为 ET 模式下 read() 必须返回 EAGAIN 才可以。写数据的情景也类似。三是在文件描述符较少的情况下，epoll 不一定比 poll 高效，使用 LT 可以于 poll 兼容，必要的时候可以切换为 PollPoller。