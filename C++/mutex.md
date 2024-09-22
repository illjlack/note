`std::mutex` 是通过操作系统的底层机制实现的。C++ 标准库并不直接实现锁，而是依赖操作系统提供的线程库（如 POSIX 或 Windows API）来管理锁的获取和释放。因此，`std::mutex` 的底层实现因平台不同而异。

在liunx上 , 它可能封装了 `pthread_mutex_t` 。它通过调用系统调用来实现锁定和解锁操作。

```cpp
class mutex {
private:
    pthread_mutex_t native_mutex;
public:
    mutex() {
        pthread_mutex_init(&native_mutex, nullptr);
    }
    ~mutex() {
        pthread_mutex_destroy(&native_mutex);
    }
    void lock() {
        pthread_mutex_lock(&native_mutex);
    }
    void unlock() {
        pthread_mutex_unlock(&native_mutex);
    }
    bool try_lock() {
        return pthread_mutex_trylock(&native_mutex) == 0;
    }
};
```

