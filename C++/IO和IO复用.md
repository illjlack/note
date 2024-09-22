# I/O复用和select,poll,epoll

在 I/O 复用的应用场景下，I/O 描述符通常指的是**套接字**（也可以是管道和文件）。

**套接字本质上是一个内核中的缓冲区**。套接字在数据传输过程中会使用内核中的发送和接收缓冲区，暂存数据。

在内存中，只能内核态调用（操作要系统调用）。

## 直接操作多个I/O

如果直接用多个I/O，阻塞模式下肯定同时只能监听一个（等待内核唤醒进程）。

非阻塞要不断轮询每一个，而每次询问都要转到内核态，转回用户态（系统调用的代价）。

### 相关系统调用

```cpp
int bind(int sockfd, const struct sockaddr *addr, socklen_t addrlen);
/**
 * bind() 系统调用将套接字与指定的 IP 地址和端口号绑定。
 * - 操作流程：
 *   1. 系统检查是否可以将指定的 IP 和端口绑定到 sockfd。
 *   2. 如果成功，套接字将与该 IP 地址和端口相关联。
 *   3. 任何发往该 IP 和端口的数据都会通过该套接字进行处理。
 */
 
int listen(int sockfd, int backlog);
/**
 * listen() 将套接字转换为监听模式，准备接受传入的连接。
 * - 操作流程：
 *   1. 将套接字从主动模式转为被动模式，允许它监听客户端的连接请求。
 *   2. backlog 参数定义了内核连接队列的最大长度，超过此长度的连接将被拒绝。
 *   3. 在此阶段，套接字并不接受连接，只是将其置为可监听状态。
 */

int accept(int sockfd, struct sockaddr *addr, socklen_t *addrlen);
/**
 * accept() 接受来自客户端的连接请求，并返回一个新的套接字用于与客户端通信。
 * - 操作流程：
 *   1. accept 会从连接队列中获取一个已经完成三次握手的连接。
 *   2. 如果队列中没有已完成的连接，accept 会默认进入阻塞状态（可以设置立即返回），等待新的连接到来。
 *   3. 一旦有客户端连接，accept 将返回一个新的文件描述符（套接字），与该客户端通信。
 *   4. 原来的套接字继续用于监听其他连接。
 */

ssize_t recv(int sockfd, void *buf, size_t len, int flags);
/**
 * recv() 从套接字接收数据，并将其存储到缓冲区 `buf` 中。
 * - 操作流程：
 *   1. 如果套接字中有数据可读取，`recv` 会将数据复制到用户空间的缓冲区（就是buf）中。
 *   2. 如果套接字中没有数据且是阻塞模式（可以设置立即返回），`recv` 将阻塞，直到有数据可读取。
 *   3. 返回读取的字节数，返回 0 表示连接已关闭。
 */

ssize_t send(int sockfd, const void *buf, size_t len, int flags);
/**
 * send() 将缓冲区 `buf` 中的数据发送到套接字对应的目标地址。
 * - 操作流程：
 *   1. 系统将数据从用户空间的缓冲区复制到内核发送缓冲区中。
 *   2. 在 TCP 连接中，系统会通过底层网络协议将数据发送到对端。
 *   3. 成功时，返回实际发送的字节数。
 */

ssize_t read(int fd, void *buf, size_t count);
/**
 * read() 从文件描述符 `fd` 中读取数据，并将其存储到缓冲区 `buf` 中。
 * - 操作流程：
 *   1. 如果文件描述符 `fd`（可以是文件、管道、套接字等）中有数据可读，`read` 会将数据从内核缓冲区复制到用户空间的缓冲区。
 *   2. 如果文件描述符是套接字，且套接字中没有数据并且是阻塞模式，`read` 会阻塞，直到有数据可读。
 		非阻塞模式下，`read` 会立即返回。
 *   3. `read` 返回实际读取的字节数，返回 0 表示文件已到达结尾（EOF），对于套接字，表示对端关闭了连接。
 *   4. 如果出现错误，`read` 返回 -1，并将 `errno` 设置为合适的错误代码。
 */

ssize_t write(int fd, const void *buf, size_t count);
/**
 * write() 将缓冲区 `buf` 中的数据写入文件描述符 `fd` 中。
 * - 操作流程：
 *   1. `write` 会将用户空间缓冲区 `buf` 中的数据复制到内核缓冲区（如文件或套接字的发送缓冲区）。
 *   2. 对于套接字，数据会从内核缓冲区异步发送到对端（如果是 TCP 套接字）。
 *   3. `write` 返回实际写入的字节数。成功写入的字节数可能小于 `count`，这取决于内核缓冲区的可用空间。
 *   4. 如果内核缓冲区已满，并且套接字是阻塞模式，`write` 会阻塞，直到有空间可以写入数据。如果是非阻塞模式，`write` 会立即返回。
 *   5. 当出现错误时，`write` 返回 -1，并设置 `errno`。
 */
```





## select

I/O复用：阻塞的时候由内核同时监听多个，非阻塞也一次完成询问。

### select的工作原理

内核为每个文件描述符维护一个等待队列（wait queue）。等待队列中包含了所有处于等待状态的进程/线程，当文件描述符状态变化时，这些进程/线程会被唤醒。

unix默认每个进程最多1024个文件描述符，文件描述符是进程本地的。



1. `select` 接受三个 `fd_set`（每个 `fd_set` 是 1024 位的位图，用来标记需要监听的文件描述符，位图中的某个位设置为 1 表示要监听该文件描述符）。分别表示读（可读事件）、写（可写事件）和异常事件要监听的文件描述符。

2. 当调用 `select()` 时，内核会遍历传入的文件描述符集合。如果某个文件描述符在初次遍历时已经就绪（或者在非阻塞模式下），`select()` 会立即返回。如果没有任何文件描述符就绪，进程会被挂起，并插入与相关文件描述符关联的等待队列。唤醒时，内核会从所有等待队列中移除该进程（等待队列通过链表实现，且记录有进程的插入位置），并让它重新处于运行状态。

   返回时会擦除`fd_set`中的没有就绪的文件描述符，

3. 在 `select()` 返回后，用户可以遍历传入的 `fd_set`，通过 `FD_ISSET()` 检查哪些文件描述符已经就绪。`FD_ISSET()` 用来判断某个文件描述符在 `select()` 返回后是否就绪。

4. 注意，`select()` 在一次调用中可能会返回多个就绪的文件描述符。某些文件描述符可能在第一次遍历时就已经就绪，或者多个文件描述符可能同时变得就绪（例如多个套接字同时收到数据）。



### select的系统调用

```cpp
int select(int nfds, fd_set *readfds, fd_set *writefds, fd_set *exceptfds, struct timeval *timeout);
/**
 * select() 监控多个文件描述符，等待其中任意一个或多个变为就绪（可读、可写或有异常）。
 * - 操作流程：
 *   1. 用户传入 3 个 `fd_set` 位图，分别用于监控可读、可写和异常事件的文件描述符。
 *   2. `nfds` 为需要监控的最大文件描述符值加 1（即 `fd + 1`）。
 *   3. 内核检查传入的文件描述符集合：
 *      - 如果任意一个文件描述符在调用时已经就绪，`select()` 立即返回。
 *      - 如果没有描述符就绪，进程被挂起，直到某个文件描述符变为就绪或超时。
 *   4. 内核将就绪的文件描述符保留在相应的 `fd_set` 中，其他未就绪的文件描述符会被清除。
 *   5. `select()` 返回就绪的文件描述符的数量。若超时则返回 0，出错返回 -1，并设置 `errno`。
 */
```



### select的demo

```cpp
#include <iostream>
#include <sys/select.h>
#include <unistd.h>

int main() {
    fd_set readfds;            // 定义文件描述符集合
    FD_ZERO(&readfds);         // 清空集合
    
    int fd = STDIN_FILENO;     // 获取标准输入的文件描述符
    FD_SET(fd, &readfds);      // 将标准输入文件描述符加入集合

    // 设置超时时间
    struct timeval timeout;
    timeout.tv_sec = 5;        // 设置超时时间为5秒
    timeout.tv_usec = 0;       // 微秒部分为0

    std::cout << "等待输入(5秒超时)...\n";

    // select函数监听多个文件描述符
    // 参数1：fd + 1，监听的最大文件描述符值加1 , 会监听fd从0~fd的所有被readfds设置的文件
    // 参数2：&readfds，监听可读事件的文件描述符集合
    // select 通过一个位图（bitmask）来表示文件描述符集。当你调用 FD_SET() 函数时，select 将在内部的位图中设置对应文件描述符的位置为1
    // fd_set 最大1024 常见的 UNIX 系统默认限制为一个进程最多可以打开 1024 个文件描述符（可通过修改系统参数提高此限制）
    // 参数3：nullptr，不监听可写事件，传入空指针
    // 参数4：nullptr，不监听异常事件，传入空指针
    // 参数5：&timeout，超时时间为5秒
    int result = select(fd + 1, &readfds, nullptr, nullptr, &timeout);
    
    if (result > 0) {
        // 如果有文件描述符就绪
        // FD_ISSET 通常与 select() 函数结合使用，用于检查在 select() 调用之后，哪个文件描述符触发了指定的事件（例如可读、可写或异常）。
        if (FD_ISSET(fd, &readfds)) {//每个fd都要检查，被唤醒后并不知道是因为哪些I/O激活的
            char buffer[100];
            read(fd, buffer, sizeof(buffer));
            std::cout << "输入内容: " << buffer << std::endl;
        }
    } else if (result == 0) {
        std::cout << "超时，没有输入。\n";
    } else {
        std::cerr << "select 错误。\n";
    }
    return 0;
}

```





## poll

`poll` 是 `select` 的替代方案，`poll` 使用了一个动态的结构来管理文件描述符，因此在处理大规模文件描述符时更加灵活，可以避免 `select` 的位图限制（默认1024个文件描述符）

 ### poll的工作原理

1. `poll` 通过一个 `pollfd` 数组来监控多个文件描述符。每个 `pollfd` 结构体包含：`fd`: 文件描述符。`events`: 要监控的事件，如 `POLLIN`（可读）、`POLLOUT`（可写）、`POLLERR`（异常）。`revents`: 内核用来返回实际发生的事件。
2. 如果在初次遍历文件描述符时（或者非阻塞模式），某个描述符已经就绪，`poll()` 会立即返回。如果没有描述符就绪，进程会被挂起，并插入相应的等待队列，等待文件描述符的状态发生变化。内核会使用等待队列机制将进程挂起，直到有 I/O 事件发生或超时。内核会将这些触发事件的信息放入 `pollfd` 结构的 `revents` 字段中。
3. 程序随后可以遍历 `pollfd` 数组，通过检查每个 `pollfd` 结构的 `revents` 字段，判断哪些文件描述符已经就绪，并进行相应的 I/O 操作。



### poll的系统调用

```cpp
int poll(struct pollfd *fds, nfds_t nfds, int timeout);
/**
 * poll() 监控多个文件描述符的 I/O 事件，类似于 `select()`，但没有文件描述符数量限制。
 * - 操作流程：
 *   1. 用户传入一个 `pollfd` 结构数组，每个 `pollfd` 结构体包含要监控的文件描述符和事件类型（可读、可写或异常）。
 *   2. `nfds` 表示数组中的文件描述符数量。
 *   3. 内核遍历传入的文件描述符：
 *      - 如果任意一个文件描述符已经就绪，`poll()` 立即返回。
 *      - 如果没有文件描述符就绪，进程会被挂起，直到某个文件描述符状态变化或超时。
 *   4. 内核在 `pollfd` 结构的 `revents` 字段中返回已发生的事件类型。
 *   5. `poll()` 返回就绪的文件描述符数量。超时返回 0，出错返回 -1，并设置 `errno`。
 */
```



### poll的demo

```cpp
#include <iostream>
#include <poll.h>
#include <unistd.h>

int main() {
    struct pollfd fds;              // 定义 pollfd 结构体
    fds.fd = STDIN_FILENO;          // 设置监听的文件描述符为标准输入
    fds.events = POLLIN;            // 设置监听的事件为“可读”

    int timeout = 5000;             // 超时设置为5秒（单位：毫秒）
    std::cout << "等待输入(5秒超时)...\n";

    // 调用 poll 函数，监听文件描述符
    // 参数1：&fds，文件描述符数组
    // 参数2：1，文件描述符数组的大小（监听一个文件描述符）
    // 参数3：timeout，超时时间，单位毫秒
    int result = poll(&fds, 1, timeout);

    if (result > 0) {
        // 如果有事件发生，检查是否是可读事件
        if (fds.revents & POLLIN) {
            char buffer[100];
            // 读取输入
            read(STDIN_FILENO, buffer, sizeof(buffer));
            std::cout << "输入内容: " << buffer << std::endl;
        }
    } else if (result == 0) {
        std::cout << "超时，没有输入。\n";
    } else {
        std::cerr << "poll 错误。\n";
    }

    return 0;
}
```





## epoll

### epoll的工作原理

可以发现，每次系统调用都要重新插入、删除一遍队列，遍历一遍监听的 fds 。

所以，可以把另建一个事件缓冲区，当某个文件描述符变为可读、可写或发生异常时，内核将其事件放入缓冲区。

当用户调用 `epoll_wait()` 时，只需从缓冲区中读取已就绪的事件，而不需要重新遍历所有文件描述符。这样大大减少了系统调用的开销。



1. `epoll` 的核心在于**事件驱动**机制和**事件回调机制**，通过在内核中维护一个红黑树（用于文件描述符管理）和一个就绪事件队列（用于事件通知）来高效监听和管理文件描述符。
2. 文件描述符的状态变化（如可读、可写等）会通过中断机制触发，在中断处理完毕之后，内核检查已经注册到 `epoll` 的文件描述符。如果该文件描述符的状态与注册时的事件（如 `EPOLLIN`、`EPOLLOUT`）匹配，内核会将此事件加入到 `epoll` 实例的**就绪队列**中。

**大致这样，内核具体做什么也不太清楚，哭。**



### epoll的系统调用

```cpp
int epoll_create(int size);
/**
 * epoll_create() 创建一个新的 epoll 实例。
 * - 操作流程：
 *   1. 用户调用 `epoll_create()`，指定 `size` 用来提示内核分配的文件描述符数目大小。
 *   2. 内核分配相应的资源，并返回一个 epoll 文件描述符。
 *   3. 如果调用成功，返回 epoll 实例的文件描述符；若失败，返回 -1，并设置 `errno`。
 */
```

```cpp
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
/**
 * epoll_ctl() 用于向 epoll 实例中添加、修改或删除文件描述符及其相关事件。
 * - 操作流程：
 *   1. 用户通过 `epoll_create()` 创建 epoll 实例，并获取 epoll 文件描述符 `epfd`。
 *   2. 通过 `epoll_ctl()` 向 epoll 实例中添加 (`EPOLL_CTL_ADD`)、修改 (`EPOLL_CTL_MOD`)、或删除(`EPOLL_CTL_DEL`) 		文件描述符 `fd`。
 *   3. `event` 参数指定要监控的事件类型（可读、可写、异常等）。
 *   4. 如果操作成功，返回 0；若操作失败，返回 -1，并设置 `errno`。
 */
```

```cpp
int epoll_wait(int epfd, struct epoll_event *events, int maxevents, int timeout);
/**
 * epoll_wait() 用于等待在 epoll 实例中注册的文件描述符集合上的 I/O 事件，支持边缘触发模式（ET）。
 * - 操作流程：
 *   1. 用户首先通过 `epoll_create()` 创建 epoll 实例，并通过 `epoll_ctl()` 将文件描述符注册到 epoll 实例。
 *   2. `epoll_wait()` 等待已注册的文件描述符上的 I/O 事件，并将就绪的文件描述符返回给用户。
 *   3. 内核检查已注册的文件描述符：
 *      - 如果任意文件描述符有事件发生（可读、可写或异常），`epoll_wait()` 返回这些事件。
 *      - 如果没有事件发生，进程被挂起，直到某个文件描述符发生事件或超时。
 *   4. 就绪事件会存储在 `events` 数组中，最多返回 `maxevents` 个事件。
 *   5. `epoll_wait()` 返回就绪事件的数量。超时返回 0，出错返回 -1，并设置 `errno`。
 */
```

### epoll的demo

```cpp
#include <iostream>
#include <sys/epoll.h>
#include <unistd.h>

int main() {
    // 创建 epoll 实例，返回 epoll 文件描述符
    int epoll_fd = epoll_create1(0);
    if (epoll_fd == -1) {
        std::cerr << "epoll_create1 错误。\n";
        return 1;
    }

    struct epoll_event ev;        // 定义 epoll_event 结构体
    ev.events = EPOLLIN;          // 设置监听的事件为“可读”
    ev.data.fd = STDIN_FILENO;    // 监听标准输入

    // 将标准输入文件描述符加入 epoll 实例中
    if (epoll_ctl(epoll_fd, EPOLL_CTL_ADD, STDIN_FILENO, &ev) == -1) {
        std::cerr << "epoll_ctl 错误。\n";
        return 1;
    }

    struct epoll_event events[10];   // 存储事件的数组
    std::cout << "等待输入(5秒超时)...\n";
    // 调用 epoll_wait，等待事件发生
    int result = epoll_wait(epoll_fd, events, 10, 5000);

    if (result > 0) {
        // 如果有事件发生，遍历所有就绪的文件描述符
        for (int i = 0; i < result; ++i) {
            if (events[i].events & EPOLLIN) {
                char buffer[100];
                // 读取输入
                read(STDIN_FILENO, buffer, sizeof(buffer));
                std::cout << "输入内容: " << buffer << std::endl;
            }
        }
    } else if (result == 0) {
        std::cout << "超时，没有输入。\n";
    } else {
        std::cerr << "epoll_wait 错误。\n";
    }

    // 关闭 epoll 文件描述符
    close(epoll_fd);
    return 0;
}
```





## 总结

- select 和 poll 的原理基本相同，就是调用的方式不同。阻塞和非阻塞（超时时间设为0）都能一次系统调用完成监听多个文件描述符。
- select 、poll 每次系统调用都要重新插入、删除一遍队列，遍历一遍监听的 fds 。 
- 从原理可以看出 select 和 poll 都是水平触发的（不处理时，文件的就绪状态没有改变）。

- epoll 在内核里维护了树和队列，文件状态改变时主动回调查询树，插入队列。
- 所以，select 和 poll 每次都要遍历一遍所有监听的文件，复杂度$O(n)$, epoll_wait 直接复制到用户缓冲区，复杂度可以当$O(1)$ , 考虑内核树上的查询$ O(logn)$
- epoll 还是有消耗的，应该减少短连接，频繁的文件状态变化。合理使用边沿触发。







