# 设计线程池
## 关注哪些
- 线程数可以配置。要满足开发人员可以根据硬件特性来设定线程数
- 注意空闲时的CPU占有率，理论上在空闲时应当尽量低，有任务时再被唤醒触发
- 任务队列
- 带参函数和返回值

## 简单版本

```C++
#include <condition_variable>
#include <functional>
#include <mutex>
#include <queue>
#include <thread>
#include <utility>

class ThreadPool {
 public:
  explicit ThreadPool(std::size_t n) {
    for (std::size_t i = 0; i < n; ++i) {
      std::thread{[this] {
        std::unique_lock<std::mutex> l(m_);
        while (true) {
          if (!q_.empty()) {
            auto task = std::move(q_.front()); // 移动语义，避免拷贝
            q_.pop();
            l.unlock();
            task();
            l.lock();
          } else if (done_) {
            break;
          } else {
            cv_.wait(l); // 如果使用 std::this_thread::yield(), 会导致 cpu 占用率极高
          }
        }
      }}.detach(); // 由于使用了 cv_.wait，因此采用 detach
    }
  }

  ~ThreadPool() {
    {
      std::lock_guard<std::mutex> l(m_);
      done_ = true;  // cv_.wait 使用了 done_ 判断所以要加锁
    }
    cv_.notify_all();
  }

  template <typename F>
  void submit(F&& f) { // 不能带参数，和返回值
    {
      std::lock_guard<std::mutex> l(m_);
      q_.emplace(std::forward<F>(f));
    }
    cv_.notify_one();
  }

 private:
  std::mutex m_;
  std::condition_variable cv_;
  bool done_ = false;
  std::queue<std::function<void()>> q_;
};
```

- 使用条件变量，避免 while 循环导致cpu 占比极高，但是线程无法在析构中回收，需要 detach
- 线程任务函数不能传参，也不能返回结果

## 带参版本
只需要修改 submit
```C++
template <class F, class... Args>
auto ThreadPool::submit(F&& f, Args&&... args) {
  using RT = std::invoke_result_t<F, Args...>;
  // std::packaged_task 不允许拷贝构造，不能直接传入 lambda，
  // 因此要借助 std::shared_ptr
  auto task = std::make_shared<std::packaged_task<RT()>>(
      std::bind(std::forward<F>(f), std::forward<Args>(args)...));
  // 但 std::bind 会按值拷贝实参，因此这个实现不允许任务的实参是 move-only 类型
  {
    std::lock_guard<std::mutex> l(m_);
    q_.emplace([task]() { (*task)(); });  // 捕获指针以传入 std::packaged_task
                                          // 注意此处 std::packaged_task 是函数对象，可以与std::function<void()>匹配
  }
  cv_.notify_one();
  return task->get_future();
}
```
