# C++ 多线程并发
由于之前一段时间都在用lua写游戏逻辑，并没有真正地涉及到多线程，故在今天遇到了动态加载和保存地图问题的时候，找了几篇来补充这部分的知识。

# C++11 Thread
C++11 以前，实现并发程序是需要借助操作系统的 API 的，而 C++11 后则在语言层面支持了多线程，这方便了编写跨平台的代码。

```C++
#include <iostream>
#include <thread>
#include <chrono>   // 时间
using namespace std;

void threadfunc()
{
    cout << "子线程开跑!" << endl;
    for (int i = 0; i <= 100; i++)
    {
        this_thread::sleep_for(chrono::milliseconds(50));
        cout << "子线程进行进度: " << i << " %" << endl;
    }
    cout << "子线程: 我搞定了!" << endl;
}

int main()
{
    thread t1(threadfunc);
    cout << "主线程开跑!" << endl;
    for (int i = 0; i <= 100; i++)
    {
        this_thread::sleep_for(chrono::milliseconds(80));
        cout << "主线程进行进度: " << i << " %" << endl;
    }
    cout << "主线程: 我搞定了!" << endl;
    t1.join();
    return 0;
}
```

这便是主线程开启了一个子线程执行 `threadfunc()` 的过程。

有参数的 thread 构造函数会在对象创建的那一刻立马开跑，而无参数的构造函数会构造一个空的线程对象。

注意到 t1.join()，他的作用是让主线程在这里等待 t1 线程是否完成，如果 t1 线程在此之前已经完成则立即返回，如果没有则等待其执行完毕再返回。

对应的有 t1.detach()，他会将这个子线程从主线程分离出去，而主线程就不会再被其阻塞了。

由于线程可以通过 this_thread 来访问自己，于是乎你可以这样获取主线程的线程 ID。

```C++
#include <iostream>
#include <thread>
using namespace std;

int main()
{
    cout << this_thread::get_id() << endl;
    cin.get();
    return 0;
}
```

由于 get_id() 方法返回的是 thread::id，不能直接转换为整形等类型，只能尝试使用 ostreamstring 流来转换。

```C++
#include <iostream>
#include <thread>
#include <sstream>
using namespace std;

void threadfunc()
{
    cout << "子线程开跑!" << endl;
    for (int i = 0; i <= 100; i++)
    {
        this_thread::sleep_for(chrono::milliseconds(30));
        cout << "子线程进行进度: " << i << " %" << endl;
    }
    cout << "子线程: 我搞定了!" << endl;
}

int main()
{
    thread t1(threadfunc);

	ostringstream os1;
    os1 << t1.get_id() <<endl;
    int id = (int)os1.str().c_str();

    t1.join();
    cout << id << endl;

    cin.get();
	return 0;
}
```

当然，提到多线程的程序，就不得不提到线程安全的问题。

# 互斥量(mutex)及锁(lock)

为了解决多个线程访问共享数据中的冲突问题，我们使用互斥量来保证一方的操作不被影响。

C++11 为我们提供了 4 种互斥量：

+ std::mutex

    + 非递归的互斥量

    + 同一时间内,只可以被一个函数获取并上锁

    + 使用较简单，一般配合 lock_guard 使用

+ std::shared_mutex

    + 读写互斥量，将访问资源的线程分为读者(共享)和写者(独享)

    + 可以有多个读者，但只能有一个写者

+ std::timed_mutex

    + 带超时的非递归互斥量

    + 使用 try_lock_for(timeout) 尝试获取锁，超时则返回 false 继续执行

+ std::recursive_mutex

    + 递归互斥量
    
    + 可以锁定两次以上,但解锁也需解锁两次以上
    
    + 常用于一个有锁的函数调用另一个有锁的函数

+ std::recursive_timed_mutex

    + 带超时的递归互斥量

接下来让我们看看普通的 mutex，一般也是用的最多的互斥量

```C++
#include<iostream>
#include<thread>
#include<mutex>
#include<chrono>
using namespace std;

mutex g_mutex;
void func()
{
	g_mutex.lock();
	cout << "entry func test thread ID is : " << this_thread::get_id() << endl;
	this_thread::sleep_for(chrono::microseconds(1000));
	cout << "leave func test thread ID is : " << this_thread::get_id() << endl;
	g_mutex.unlock();
}

int main()
{
	thread t1(func);
	thread t2(func);
	thread t3(func);
	thread t4(func);
	thread t5(func);

	t1.join();
	t2.join();
	t3.join();
	t4.join();
	t5.join();

	cin.get();
	return 0;
}
```
因为我们在函数逻辑执行的前后加了互斥量，所以这五个子线程可以完整地完成他们的各自的逻辑，

但这里仍然有一个不足，如果在函数逻辑执行过程中抛出异常导致程序退出了，就会导致死锁，

所以我们一般采用 `lock_guard` 来控制一个互斥量，确保互斥量解锁。

```C++
#include<iostream>
#include<thread>
#include<mutex>
#include<chrono>
using namespace std;

mutex g_mutex;
void func()
{
	lock_guard<mutex> lock(g_mutex);            // 用 lock_guard 代替手动开关
	cout << "entry func test thread ID is : " << this_thread::get_id() << endl;
	this_thread::sleep_for(chrono::microseconds(1000));
	cout << "leave func test thread ID is : " << this_thread::get_id() << endl;
}

int main()
{
	thread t1(func);
	thread t2(func);
	thread t3(func);
	thread t4(func);
	thread t5(func);

	t1.join();
	t2.join();
	t3.join();
	t4.join();
	t5.join();

	cin.get();
	return 0;
}
```

shared_mutex 通过 `lock()` 和 `unlock()` 进行写者的锁定和解锁，

通过 `lock_shared()` 和 `unlock_shared()` 进行读者的锁定与解锁。

```C++
#include<iostream>
#include<thread>
#include<shared_mutex>          // 和<mutex>不一样
using namespace std;

shared_mutex s_mutex;
void read()
{
	s_mutex.lock_shared();
	cout << "我是读者!" << endl;
	s_mutex.unlock_shared();
}
void write()
{
	s_mutex.lock();
	cout << "我是写者!" << endl;
	s_mutex.unlock();
}

int main()
{
	thread t1(read);
	thread t2(read);
	thread t3(read);
	thread t4(write);

	t1.join();
	t2.join();
	t3.join();
	t4.join();

	cin.get();
	return 0;
}
```

当然，你也可以用 `lock_guard` 来控制共享互斥量中写者的锁定(排他性锁定)，用 `shared_lock` 来控制共享互斥量中读者的锁定(共享锁定)，

这里请自己试一试，也就是在 read() 中用 `shared_lock`，在 write() 中用 `lock_guard`。

除了这两个锁以外，我们还有 `unique_lock`，他不仅拥有 `lock_guard` 的原有功能，还能手动上锁和解锁，

构造的时候我们还可以传入第二个参数，完成不同的功能，

但需要付出更多的时间和性能成本。

```C++
#include <iostream>
#include <thread>
#include <mutex>
using namespace std;

mutex m;
void proc1(int a)
{
	unique_lock<mutex> g1(m, defer_lock);					// 初始化一个没有上锁的锁
	cout << "proc1: 我可不是一般的锁!" << endl;
	g1.lock();												// 手动上锁
	cout << "proc1 函数正在改写 a" << endl;
	cout << "proc1: a 原来是 " << a << endl;
	cout << "proc1: a 现在是 " << a + 2 << endl;
	g1.unlock();											// 临时解锁
	cout << "proc1 暂时解锁啦!" << endl;
	g1.lock();
	cout << "proc1: 现在我又锁上啦! 嘿嘿嘿" << endl;		  // 但离开这个函数你就没了~!
}
void proc2(int a)
{
	unique_lock<mutex> g2(m, try_to_lock);
	cout << "proc2: 我看看这个东西能不能锁?" << endl;
	if (g2.owns_lock())										// 加锁成功
	{
		cout << "proc2: 我超，锁上了! 开始改写!" << endl;
		cout << "proc2: a 原来是 " << a << endl;
		cout << "proc2: a 现在是 " << a + 1 << endl;
	}
	else                                                    // 没锁上,别人锁了
	{
		cout << "proc2: 我超，锁不上，溜了溜了" << endl;
	}
}

int main()
{
	int a = 0;
	thread t1(proc1, a);
	thread t2(proc2, a);
    t1.join();
	t2.join();

	cin.get();
	return 0;
}
```

这个示例中我们看到了两个参数，`defer_lock` 和 `try_to_lock`，还有一个没出现的是 `adopt_lock`，没出现的这个就是 `lock_guard` 的状态。

而 `defer_lock` 会构造一个没有上锁的锁，`try_to_lock` 会尝试对一个互斥量进行上锁，不成功的话他也不会等，直接开溜，所以我们要判断他是否上锁成功 `owns_lock()` 来控制他的行为。

所有的锁都不能复制，但是可以通过 move() 函数来交换一个互斥量的所有权。

# 条件变量(condition_variable)

他并不是用来管理互斥量的，而是用来同步线程的，他通过一个判断条件决定是否阻塞线程，当条件满足时就会唤醒阻塞的线程。常与互斥量配合使用，请让我们看看示例。

```C++
#include <iostream>
#include <condition_variable>
#include <thread>
#include <list>
#include <mutex>
#include <chrono>
using namespace std;

class CTask
{
public:
	CTask(int taskId)
	{
		this->taskID = taskId;
	}
	void DoTask(int a)
	{
		cout << a << ": 第" << taskID << "个活！" << "艹，走，忽略！" << endl;
	}
private:
	int taskID;
};

list<shared_ptr<CTask>> g_task;
mutex g_mutex;
condition_variable g_conv;

void ProducerFunc()
{
	int n_taskId = 0;
	shared_ptr<CTask> ptask = nullptr;
	while (true)
	{
		ptask = make_shared<CTask>(n_taskId);					// 代替智能指针的构造函数，在该场景下有好处
		{
			lock_guard<mutex> lock(g_mutex);
			g_task.push_back(ptask);
			cout << "来，小亮，给他整第 " << n_taskId << " 个活！" << endl;
		}
		g_conv.notify_one();
		n_taskId++;
		this_thread::sleep_for(chrono::milliseconds(1000));
	}
}
void ConsumerFunc(int a)
{
	shared_ptr<CTask> ptask = nullptr;
	while (true)
	{
		unique_lock<mutex> lock(g_mutex);
		while (g_task.empty())
		{
			g_conv.wait(lock);
		}
		ptask = g_task.front();
		g_task.pop_front();
		if (ptask == nullptr)
		{
			continue;
		}
		ptask->DoTask(a);
	}
}

int main()
{
	thread t1(ConsumerFunc, 1);
	thread t2(ConsumerFunc, 2);
	thread t3(ConsumerFunc, 3);

	thread t4(ProducerFunc);

	t1.join();
	t2.join();
	t3.join();
	t4.join();
	
	return 0;
}
```

这个示例里面你可以看到4个线程，其中3个为"消费者线程"，还有一个 t4 为"生产者线程"，他们之间除了信号量，还有一个链表储存要处理的事件。

我们可以看到在 `ProducerFunc()` 中，它先上了锁，然后把想要整活的信息发送到了全局的链表中，接着如果想要提醒那些等着 `g_conv` 信号量的线程"有活整了!"，就得用 `notify_one()` 或 `notify_all()`，使那些函数的 `wait(lock)` 不再堵塞，但为什么我们还需要一个 `while(g_task.empty())` 呢?

在这里，`notify_one()` 可以随机唤醒一个线程，而 `notify_all()` 可以唤醒所有线程，但 `notify_one()` 仍然有唤醒多余一个线程 (即`虚假唤醒`) 的风险，保险起见，我们用到这个循环判断当前链表中是否有消费者线程想要的事件，以防止出现多个线程被唤醒去获取这个事件的情况。

# 原子类型 atomic<>

原子类型进行着"原子操作"，指的就是"不可分割的操作"，他针对一个变量的操作进行加锁以防止其它线程干扰，使用起来也极其方便。

```C++
#include <atomic>
std::atomic<bool> b(true);
b = false;
```

`atomic<>` 模板最少会给类型提供原子读操作 `load()`，原子写操作 `store()`，以及原子交换操作 `exchange()`，其中对于 `store()`，这个模板还重载了 `= 运算符` 以进行相同的操作。

而对于 int，float，string 等类型，他提供了更多原子操作，诸如加减乘除之类。

# 阅读文章

[C++多线程并发基础入门教程](https://zhuanlan.zhihu.com/p/194198073)

[C++11多线程](https://zhuanlan.zhihu.com/p/354316544)

> by hsz 2021.11.11