# C++ Concurrency In Action

## 第1章 线程管理

### 1.1 启动线程

```cpp
void hello() {
    cout << "Hello World";
}

int main() {
    thread t(hello);
    t.join();
    return 0;
}
```

Callable对象

```cpp
struct Callable {
    int operator()(const int &l, const int &r) { return l + r; }
};

int main() {
    Callable f;
    cout << f(1, 1);
    return 0;
}
```

```cpp
struct Func {
    int &v;
    explicit Func(int &v_) : v(v_) { cout << "Construct\n"; }
    void operator()() const {
        for (auto i = 0; i < 100; ++i) cout << v;
        cout << "\nf return\n";
    }
};


int main() {
    int v = 0;
    Func f{v};
    thread t(f);
    try {
        throw(logic_error{"A logic error"});
    } catch (logic_error &e) {
        cout << "Error caught";
        t.join();
        throw;
    }
    t.join();
    return 0;
}
```

### 1.2 thread::join()

t线程加入main线程，main线程等待t线程结束

```cpp
class ThreadGuard {
    thread &t;

public:
    explicit ThreadGuard(thread &t_) : t(t_) {}          // 构造函数
    ThreadGuard(ThreadGuard const &) = delete;           // 禁止拷贝构造
    ThreadGuard &operator=(ThreadGuard const &) = delete;// 禁止拷贝赋值
    ~ThreadGuard() {                                     // 析构函数
        if (t.joinable())                                // 线程t是否可加入
            t.join();
        cout << "\nDestructing";
    }
};

struct Func {
    int &v;
    explicit Func(int &v_) : v(v_) { cout << "Constructing\n"; }
    void operator()() const {
        for (auto i = 0; i < 10000; ++i) cout << v;
    }
};

int main() {
    int v = 0;
    Func f{v};
    thread t{f};
    ThreadGuard tg{t};
    return 0; // 析构tg时, t线程加入主线程
}
```

### 1.3 thread::detach()

t线程分离main线程，main线程不等待t线程结束（分离后t线程不可再join）

```cpp
int main() {
    thread t{[]() { cout << "Hello World"; }};
    if (t.joinable()) t.detach();
    assert(!t.joinable());
    return 0;
}
```

### 1.4 向线程函数传递参数

```cpp
void f(int n, string const &s) {
    for (auto i = 0; i < n; ++i) cout << s << endl;
}

void oops(int n) {
    char buffer[1024];
    sprintf(buffer, "%i", n);
    thread t{f, 3, buffer}; // buffer is a pointer that's passed through to the t
    t.detach(); // oops will exit before the *buffer has been convert to a string, thus leading to undefined behaviour.
}

// line 8
thread t{f, 3, string{buffer}}; // using string constructor avoids dangling pointer悬空指针
```

| std::move          | 移动元素       |
| ------------------ | -------------- |
| std::ref/std::cref | 显式取引用     |
| std::bind          | 绑定函数与参数 |

```cpp
double divide(double x, double y) { return x / y; }

int main() {
    auto half = std::bind(divide, placeholders::_1, 2);
    cout << half(6); // 3
    return 0;
}
```

可以传递一个成员函数指针、一个对象指针（作为this指针）、若干个函数参数，以创建线程

```cpp
struct X {
    void echo(int v) { cout << v; }
};

int main() {
    X x;
    thread t(&X::echo, &x, 666); // function = X::echo, this = &x, v = 666
    t.join(); // 666
    return 0;
} 
```

简单的线程组

```cpp
void echo(unsigned v) { cout << v; }

int main() {
    vector<thread> threadGroup;
    for (unsigned i = 0; i < 10; ++i) {
        threadGroup.emplace_back(echo, 6);
    }
    for_each(threadGroup.begin(), threadGroup.end(),
             std::mem_fn(&thread::join));
    // func mem_fn(指向类的成员对象的指针) 指向类的成员对象的指针的包装对象
    return 0;
}
```

### 1.5 其他

```cpp
int main() {
    cout << this_thread::get_id(); // main thread id = 1
    cout << stdthread::hardware_concurrency(); // 24 core
    return 0;
}
```

## 第2章 线程间共享数据

### 2.1 mutex信号量、lock_guard、unique_lock、shared_lock

RAII **R**esource **A**cquisition **I**s **I**nitialization构造时加锁、析构时解锁

| RAII风格的锁 | 互斥量                 |                                                              |
| ------------ | ---------------------- | ------------------------------------------------------------ |
| lock_guard   | mutex                  | 不支持手动加锁解锁、条件变量                                 |
| scoped_lock  | mutex                  | 不支持手动加锁解锁、条件变量；支持同时对多个互斥量加锁、解锁 |
| unique_lock  | mutex                  | 支持手动加锁解锁、条件变量，提供独占式访问                   |
| shared_lock  | shared_mutex共享互斥量 | 支持手动加锁解锁、条件变量，提供共享式访问                   |

```cpp
mutex mut; // global object

void appendV(list<int> &spList, int v) {
    mut.lock();
    for (int i = 1; i <= v; ++i) {
        cout << "Appending " << i << endl;
        spList.emplace_back(i);
        this_thread::sleep_for(chrono::seconds(1));
    }
    mut.unlock();
}

void containsV(unique_ptr<list<int>> upMutList, int v) {
    lock_guard<mutex> guard{mut}; // 锁定一个互斥量
    for_each(upMutList->cbegin(), upMutList->cend(), [](const int i) -> void { cout << i; });
    cout << endl << (find(upMutList->cbegin(), upMutList->cend(), v) != upMutList->cend());
}

int main() {
    auto pMutList = new list<int>{};
    /**
     * Although `appendV` expects the 1st parameter to be passed by reference,
     * the thread constructor doesn't know that (use std::ref)
     */
    thread t1(appendV, std::ref(*pMutList), 6);
    /**
     * The following example shows the use of std::move 
     * to transfer ownership of a dynamic object into a thread
     */
    unique_ptr<list<int>> upMutList{pMutList};
    thread t2{containsV, std::move(upMutList), 6};
    t2.join();
    t1.join();
    return 0;
}
```

adopt_lock表示mutex已加锁、构造时不加锁
defer_lock表示mutex未加锁、构造时不加锁

```cpp
mutex mut1, mut2;
lock(mut1, mut2); // 对mut1 mut2加锁
lock_guard<mutex> guard1{mut1, std::adopt_lock}; 
lock_guard<mutex> guard2{mut2, std::adopt_lock};
```

```cpp
mutex mut1, mut2
unique_lock<mutex> lock1{mut1, std::defer_lock}
unique_lock<mutex> lock2{mut2, std::defer_lock}
lock(mut1, mut2); // 对mut1 mut2加锁
```

### 2.2 单例模式：延迟初始化

```cpp
struct User {
    string name;
    void echo() const { cout << name; }
    explicit User(string _name) : name(std::move(_name)) {}
};

shared_ptr<User> sp;
mutex mut{};

void lazyInitialization() {
    unique_lock<mutex> uniLock{mut};
    if (sp == nullptr) {
        // Clang-Tidy: Use std::make_shared instead. sp = std::make_shared<User>("yukino");
        sp.reset(new User{"yukino"});
    }
    uniLock.unlock();
    sp->echo();
}
```

### 2.3 单例模式：双重检查锁

```cpp
void doubleCheckedLocking() {
    if (userSp == nullptr) {
        lock_guard<mutex> guard{mut};
        // in case another thread has done the initialization between the first check and this thread acquiring the lock
        if (userSp == nullptr) { // potential data races (undefined behavior)
            userSp = make_shared<User>("Updated");
        }
        userSp->echo();
    }
}
```

### 2.4 单例模式 std::once_flag std::call_once

```cpp
void userInit() { userSp = make_shared<User>("yukino"); }

void getInstance() {
    std::call_once(flag, userInit);
    userSp->echo();
}

// User& getInstance() {
//     static User instance;
//     return instance;
// } 
```

### 2.5 使用std::shared_mutex保护数据结构

```cpp
#include <boost/thread/shared_mutex.hpp>
#include <map>
#include <mutex>
#include <string>

using namespace std;
using Dns = string;

class DnsCache {
    map<string, Dns> domain2Dns;
    mutable boost::shared_mutex mut;

public:
    /**
     * findDns() uses an instance of boost::shared_lock to protect it for shared, Read-Only access.共享读
     * multiple threads can therefore call findDns() simultaneously同时 without problems.
     */
    Dns findDns(string const &domain) const {
        boost::shared_lock<boost::shared_mutex> sharedLock{mut}; // Read-Only
        const auto iter = domain2Dns.find(domain); // map<string, Dns>::const_iterator
        return iter == domain2Dns.cend() ? Dns{} : iter->second;
    }

    /**
     * updateDns() uses an instance of std::lock_guard to provide exclusive access独占写
     * while the table is updated, not only are other threads prevented from doing updates in a call to updateDns()
     * but also threads that call findDns() are blocked too.
     */
    void updateDns(const string &domain, const Dns &dns) {
        lock_guard<boost::shared_mutex> sharedLock(mut);
        domain2Dns[domain] = dns; // UPDATE OR ADD
    }
};
```

## 第3章 同步并发操作

### 3.1 条件变量std::condition_variable

```cpp
mutex mut;
queue<string> q;
condition_variable cond;

void dataPrepare() {
    lock_guard<mutex> guard{mut};
    q.emplace("data"); // pushes the data onto the queue.
    cond.notify_one();
    // calls the notify_one() on the std::condition_variable instance to notify the waiting thread
}

void dataProcess() {
    std::unique_lock<mutex> uniLock{mut};// 独占锁
    /**
     * a lambda function that expresses the condition being waiting for
     * if the `q` is empty, wait() unlocks the mutex and puts the thread in a blocked or waiting state; 释放锁（解锁）
     * if the `q` is not empty, wait() returns with the mutex still locked. 占有锁（加锁）
     */
    cond.wait(uniLock, [] { return !q.empty(); });
    cout << q.front();
    q.pop();
}
```

使用条件变量的线程安全的队列

```cpp
template<typename T>
class ThreadSafeQueue {
    mutable std::mutex mut; // The mutex bust be mutable
    queue<T> dataQueue;
    condition_variable cond;

public:
    void push(T value) {
        lock_guard<mutex> guard(mut);
        dataQueue.push(value);
        cond.notify_one();
    }

    void waitPop(T &value) {
        unique_lock<mutex> mutLock{mut}; // 加锁（占有锁）
        cond.wait(mutLock, [this] { return !dataQueue.empty(); });
        value = dataQueue.front();
        dataQueue.pop();
    }

    shared_ptr<T> waitPop() {
        unique_lock<mutex> mutLock{mut};
        cond.wait(mutLock, [this] { return !dataQueue.empty(); });
        shared_ptr<T> ret = make_shared<T>(dataQueue.front());
        dataQueue.pop();
        return ret;
    }

    bool tryPop(T &value) {
        lock_guard<mutex> guard{mut};
        if (dataQueue.empty()) return false;
        value = dataQueue.front();
        dataQueue.pop();
        return true;
    }

    shared_ptr<T> tryPop() {
        lock_guard<mutex> guard{mut};
        if (dataQueue.empty()) return shared_ptr<T>{};
        shared_ptr<T> ret = make_shared<T>(dataQueue.front());
        dataQueue.pop();
        return ret;
    }

    bool empty() const {
        lock_guard<mutex> guard{mut};
        return dataQueue.empty();
    }
};
```

### 3.2 期望std::future<T\> fut = std::async(callable, args);

异步调用std::async(callable, args)

唯一期望future（类似unique_ptr）future只可关联一个task

共享期望shared_future（类似shared_ptr）shared_future可以关联多个task

```cpp
template<typename T>
struct arrDeleter {
    void operator()(T const *p) {
        delete[] p;
    }
};

string getCurrTime(const string &prefix) {
    auto now = std::chrono::system_clock::now();
    std::time_t timeT = std::chrono::system_clock::to_time_t(now);
    std::tm *currTime = std::localtime(&timeT);
    shared_ptr<char> sp0{new char[80], arrDeleter<char>()};
    shared_ptr<char> sp1{new char[80], [](const char *p) -> void { delete[] p; }};
    shared_ptr<char> sp2{new char[80], std::default_delete<char[]>()};
    unique_ptr<char> up{new char[80]};
    std::strftime(up.get(), 80, "%Y-%m-%d %H:%M:%S", currTime);
    unique_ptr<string> ret{new string(up.get())};
    return prefix + " " + *ret;
}

// 使用std::future从异步任务中获取返回值
int main() {
    // use std::async to start an asynchronous task for which you don't need the result right away.
    std::future<string> ans0 = std::async(std::launch::async, &getCurrTime, "在新线程上执行");
    future<string> ans1 = async(launch::deferred, &getCurrTime, "在wait()或get()调用时执行");
    future<string> ans2 = async(launch::async | launch::deferred, &getCurrTime,
                                "选择执行方式，默认launch::deferred");
    cout << ans0.get();
}
```

### 3.3 任务std::package_task

std::package_task对象为一个函数或可调用对象绑定一个期望std::future，当std::package_task对象被调用时，调用绑定的函数或可调用对象，得到返回值，将期望状态置为就绪 

```cpp
struct Kobe {
    int mamba(int v) { return v + v; }
    int operator()(int v) { return v * v; }
};

int main() {
    Kobe kobe;
    // thread{kobe, 6};                         // copy of kobe(6) in a new thread
    // thread{std::ref(kobe), 6};               // kobe(6) in a new thread
    // thread{&Kobe::mamba, kobe, 6};           // copy of kobe.mamba(6) in a new thread
    // thread(&Kobe::mamba, std::ref(kobe), 6); // kobe.mamba(6) in a new thread
    // thread([](int v) { return v; }, 6);
    // packaged_task can bind a callable object
    packaged_task<int(int)> pacTask0{kobe}; // kobe是一个可调用对象
    packaged_task<int()> pacTask1{std::bind(kobe, 6)}; // Clang-Tidy: Prefer a lambda to std::bind
    future<int> _future = pacTask0.get_future(); // 获取与packaged_task关联的future对象
    thread t{std::move(pacTask0), 6}; // 构造一个新线程执行kobe()
    int x0 = _future.get();
    t.join();
    pacTask1(); // std::package_task对象被调用
    int x1 = pacTask1.get_future().get();
    cout << x0 << x1;
}
```

```cpp
deque<packaged_task<int()>> taskQ;
mutex mut;

int main() {
    thread t{[]() {
        bool loop = true;
        while (loop) {
            packaged_task<int()> reciever;
            {
                lock_guard<mutex> guard{mut};
                if (taskQ.empty()) continue;
                loop = false;
                reciever = std::move(taskQ.front());
                taskQ.pop_front();
            }
            reciever();
        }
    }};
    auto f = [](int x) { return x; };
    packaged_task<int()> pacTask{bind(f, 6)};
    future<int> _future = pacTask.get_future();
    {
        lock_guard<mutex> guard{mut};
        taskQ.push_back(std::move(pacTask));
    }
    t.join();
    cout << _future.get(); // 6
    return 0;
}
```

改进：使用unique_lock

```cpp
deque<packaged_task<int()>> taskQ;
mutex mut;
condition_variable cond;

int main() {
    thread t{[]() {
        packaged_task<int()> reciever;
        {
            unique_lock<mutex> uniLock{mut};
            cond.wait(uniLock, []() { return !taskQ.empty(); });
            reciever = std::move(taskQ.front());
            taskQ.pop_front();
        }
        reciever();
    }};
    auto f = [](int x) { return x; };
    packaged_task<int()> pacTask{bind(f, 6)};
    future<int> _future = pacTask.get_future();
    {
        lock_guard<mutex> guard{mut};
        taskQ.push_back(std::move(pacTask));
    }
    cond.notify_one();
    t.join();
    cout << _future.get(); // 6
    return 0;
}
```

### 3.4 承诺std::promise

| 承诺promise            |                      |
| ---------------------- | -------------------- |
| promise::set_value     | 为期望future存储值   |
| promise::set_exception | 为期望future存储异常 |

```cpp
// std::promise类型的对象与std::future类型的对象关联
// 可以通过std::promise的成员函数get_future得到与std::promise关联的std::future对象
void acc(vector<int>::iterator first, vector<int>::iterator last, promise<int> accPromise) {
    int sum = accumulate(first, last, 0);
    accPromise.set_value(sum);// 为future存储值
}

int main() {
    vector<int> nums{1, 2, 3, 4, 5};
    promise<int> accPromise;
    future<int> accFuture = accPromise.get_future();
    thread accThread{acc, nums.begin(), nums.end(), std::move(accPromise)};
    // accFuture.wait(); // 等待future的值，调用get()前无需调用wait()
    cout << "result = " << accFuture.get() << endl; // result = 15
    accThread.join();
}
```

```cpp
void squareRoot(double x, promise<double> sqrtPromise) {
    if (x < 0) throw out_of_range("x < 0");
    double ans = sqrt(x);
    try {
        sqrtPromise.set_value(ans);
    } catch (out_of_range &err) {
        // sqrtPromise.set_exception(current_exception()); // 存储异常，抛出
        sqrtPromise.set_exception(make_exception_ptr(err)); // 存储异常，不抛出
    }
}

int main() {
    promise<double> sqrtPromise;
    future<double> sqrtFut = sqrtPromise.get_future();
    thread sqrtThread = thread{squareRoot, -1, std::move(sqrtPromise)};
    sqrtThread.join(); // terminate called after throwing an instance of 'std::out_of_range', what(): x < 0
    return 0;
}
```

-   std::future多个线程等待时，只有一个线程可以获取等待结果（一个线程等待一个事件）
-   std::shared_future多个线程等待时，多个线程都可以获取等待结果（多个线程等待一个事件）

```cpp
promise<int> p1;
future<int> f1{p1.get_future()};
assert(f1.valid()); // 期望f1合法
shared_future<int> sf1{std::move(f1)};
assert(!f1.valid()); // 期望f1不合法
assert(sf1.valid()); // 期望sf1合法
promise<string> p2;
shared_future<string> sf2{p2.get_future()};  // 隐式转移所有权
```

### 3.5 限制等待时间

时延duration

````cpp
chrono::milliseconds ms(54802);
chrono::seconds s = chrono::duration_cast<chrono::seconds>(ms); // 显式转换

int main() {
    chrono::milliseconds ms(161043261);
    chrono::seconds s = chrono::duration_cast<chrono::seconds>(ms); // 时延变量的显式转换
    cout << s.count(); // 161043
    s -= chrono::seconds(1043);
    cout << s.count(); // 160000
    future<int> f = std::async(std::launch::async, [](int v) {
        this_thread::sleep_for(chrono::milliseconds(2999));
        return v;
    }, 666); // policy, fn, args
    if (f.wait_for(chrono::seconds(3)) == future_status::ready) {
        cout << f.get();
    }
}
````

时间戳time_point

```cpp
// using high_resolution_clock = system_lock
chrono::hight_resolution_clock::now() + chrono::seconds(3); // 3秒后

int main() {
    chrono::time_point begin = chrono::high_resolution_clock::now(); 
    for (int i = 0; i < INT32_MAX; ++i);
    chrono::time_point end = chrono::high_resolution_clock::now();
    cout << chrono::duration_cast<chrono::milliseconds>(end - begin).count(); // 850
}
```

具有超时功能的函数

| 类型                                                 | 函数                                    | 返回值                                                  |
| ---------------------------------------------------- | --------------------------------------- | ------------------------------------------------------- |
| std::this_thread                                     | sleep_for(duration)                     | N/A                                                     |
| std::this_thread                                     | sleep_until(time_point)                 | N/A                                                     |
| std::condition_variable, std::condition_variable_any | wait_for(lock, duration)                | std::cv_status::timeout, std::cv_status::no_timeout     |
| std::condition_variable, std::condition_variable_any | wait_until(lock, time_point)            | std::cv_status::timeout, std::cv_status::no_timeout     |
| std::condition_variable, std::condition_variable_any | wait_for(lock, duration, predicate)     | bool                                                    |
| std::condition_variable, std::condition_variable_any | wait_until(lock, time_point, predicate) | bool                                                    |
| std::timed_mutex, std::recursive_timed_mutex         | try_lock_for(duration)                  | N/A                                                     |
| std::timed_mutex, std::recursive_timed_mutex         | try_lock_until(time_point)              | bool                                                    |
| std::future<T\>, std::shared_future<T\>              | wait_for(duration)                      | std::future_status::timeout                             |
| std::future<T\>, std::shared_future<T\>              | wait_until(time_point)                  | std::future_status::ready, std::future_status::deferred |

快速排序：并行的期望

```cpp
template<typename T>
list<T> quickSort(list<T> _in) {
    if (_in.empty()) return _in;
    list<T> ret;
    /**
     * Transfers elements from one list to another.
     * _position Const_iterator referencing the element to insert BEFORE.
     * _x        Source list.
     * _i        Const_iterator referencing the element to move
     */
    //
    ret.splice(ret.cbegin(), _in, _in.cbegin()); // ret{1}, _in{6, 1, 0, 4, 3, 2, 6, 1}
    T const &pivot = *ret.cbegin();
    // all elements returns true precede all elements returns false
    auto div = std::partition(_in.begin(), _in.end(), [&](T const &t) { return t <= pivot; });

    list<T> leftIn;
    leftIn.splice(leftIn.cend(), _in, _in.cbegin(), div); // leftIn{1, 1, 0}
    list<T> rightIn = _in; // rightIn{4, 3, 2, 6, 6}

    // list<T> newLin = quickSort(std::move(leftIn));
    future<list<T>> newLeft{async(quickSort<T>, std::move(_in))};
    ret.splice(ret.cend(), rightIn);
    ret.splice(ret.cbegin(), newLeft.get());
    return ret;
}

int main() {
    list<int> in{1, 6, 1, 0, 4, 3, 2, 6, 1};
    list out = quickSort(in);
    copy(out.cbegin(), out.cend(), ostream_iterator<int>(cout, " "));
    return 0;
}
```

## 第4章 高级线程管理

### 4.1 线程池

```cpp

```



