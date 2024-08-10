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
