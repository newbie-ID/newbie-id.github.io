---
title: "Java 多线程手搓（一）"
date: 2026-05-05T02:00:00+08:00
draft: false
tags:
  - Java
  - 多线程
  - 并发编程
  - 学习笔记
categories:
  - 技术
summary: "学习记录：从零开始手搓 Java 多线程核心组件，包括定时任务调度器、线程池和轻量级锁（AQS 简化版），深入理解并发编程的底层原理。"
---

## 前言

在 Java 开发中，多线程和并发编程是绕不开的核心技能。我们日常开发中经常使用 `ScheduledExecutorService`、`ThreadPoolExecutor`、`ReentrantLock` 等高级 API，但它们底层到底是如何工作的？

本篇学习记录基于 B 站 [学Java的生生](https://space.bilibili.com/7968519) 的课程，从零开始手搓三个核心并发组件：**定时任务调度器**、**线程池**和**轻量级锁**。通过自己动手实现，深入理解并发编程的底层原理。

---

## 一、手搓定时任务

### 1.1 项目目的

实现一个定时任务调度器，能够每隔一段时间执行一次任务。Java 标准库中通常使用 `ScheduledExecutorService` 来完成这个需求：

```java
ScheduledExecutorService executor = Executors.newSingleThreadScheduledExecutor();
executor.scheduleAtFixedRate(() -> {
    System.out.println("任务执行，当前时间: " + LocalDateTime.now());
}, 0, 300, TimeUnit.MILLISECONDS);
```

我们的目标是逐步实现一个简化版的 `MyScheduleService`，最终达到类似的效果。

### 1.2 最初的尝试：简单线程池方案

最直观的想法是：每个任务创建一个线程，线程内部循环休眠后执行任务。

```java
public class MyScheduleService {
    void schedule(Runnable task, long initialDelay, long delay) {
        ExecutorService executorService = Executors.newFixedThreadPool(6);
        executorService.execute(() -> {
            while (true) {
                Thread.sleep(delay);
                task.run();
            }
        });
    }
}
```

这个方案虽然能跑，但缺陷非常明显：

- **线程池容量有限**：固定 6 个线程，第 7 个任务就会阻塞
- **资源浪费**：每个任务独占一个线程，即使大部分时间在休眠
- **没有实现初始延迟**：忽略了 `initialDelay` 参数

### 1.3 触发器思路

既然每个线程都在白白休眠，为什么不设计一个**触发器**来统一管理所有任务呢？核心思路是：

1. 定义任务实体类，包含任务内容、开始时间和延迟间隔
2. 使用一个触发器线程，不断从任务列表中取出到期任务执行
3. 执行完的任务重新计算下次执行时间，再次加入队列

首先定义任务实体类：

```java
public class Job implements Comparable<Job> {
    private Runnable task;
    private long startTime;
    private long delay;

    @Override
    public int compareTo(Job o) {
        return Long.compare(this.startTime, o.startTime);
    }
    // getter/setter 省略
}
```

触发器实现：

```java
public class MyScheduleService {
    ExecutorService executor = Executors.newSingleThreadExecutor();
    List<Job> jobList = new ArrayList<>();
    Trigger trigger = new Trigger();

    void schedule(Runnable task, long initialDelay, long delay) {
        Job job = new Job();
        job.setTask(task);
        job.setStartTime(System.currentTimeMillis() + initialDelay);
        job.setDelay(delay);
        synchronized (jobList) {
            jobList.add(job);
            jobList.notifyAll(); // 唤醒等待的 trigger 线程
        }
    }

    class Trigger {
        Thread thread = new Thread(() -> {
            while (true) {
                synchronized (jobList) {
                    while (jobList.isEmpty()) {
                        jobList.wait(); // 无任务时休眠
                    }
                    Collections.sort(jobList);
                    Job job = jobList.get(0);
                    long waitTime = job.getStartTime() - System.currentTimeMillis();
                    
                    if (waitTime > 0) {
                        Thread.sleep(waitTime); // 等待到执行时间
                    } else {
                        executor.execute(job.getTask());
                        jobList.remove(job);
                        // 重新计算下次执行时间
                        Job newJob = new Job();
                        newJob.setTask(job.getTask());
                        newJob.setStartTime(job.getStartTime() + job.getDelay());
                        newJob.setDelay(job.getDelay());
                        jobList.add(newJob);
                    }
                }
            }
        }, "Trigger");

        public Trigger() {
            thread.start();
        }
    }
}
```

**关键设计点：**

- **`wait()` vs `sleep()`**：`wait()` 会释放锁，允许新任务加入；`sleep()` 会保持锁，阻塞其他线程
- **排序保证公平**：每次取任务前排序，确保最先执行的是最早到期的任务
- **任务循环**：执行完后重新计算时间加入队列，实现周期性执行

### 1.4 优先队列优化

使用 `List` 每次都要 `O(n log n)` 排序，自然想到用 `PriorityQueue`。更进一步，Java 提供了 `PriorityBlockingQueue`，专门用于此类并发场景：

```java
public class MyScheduleService {
    ExecutorService executor = Executors.newSingleThreadExecutor();
    PriorityBlockingQueue<Job> jobList = new PriorityBlockingQueue<>();
    Trigger trigger = new Trigger();

    void schedule(Runnable task, long initialDelay, long delay) {
        Job job = new Job();
        job.setTask(task);
        job.setStartTime(System.currentTimeMillis() + initialDelay);
        job.setDelay(delay);
        jobList.add(job);
        trigger.wakeUp();
    }

    class Trigger {
        Thread thread = new Thread(() -> {
            while (true) {
                try {
                    Job job = jobList.take();
                    long waitTime = job.getStartTime() - System.currentTimeMillis();
                    
                    if (waitTime > 0) {
                        jobList.add(job); // 重新入队
                        LockSupport.parkUntil(waitTime); // 精确休眠
                    } else {
                        executor.execute(job.getTask());
                        job.setStartTime(job.getStartTime() + job.getDelay());
                        jobList.add(job);
                    }
                } catch (InterruptedException e) {
                    throw new RuntimeException(e);
                }
            }
        }, "Trigger");

        public Trigger() { thread.start(); }
        
        public void wakeUp() {
            LockSupport.unpark(thread);
        }
    }
}
```

**优化亮点：**

- `take()` 方法在队列为空时自动阻塞，无需手动 `wait/notify`
- `LockSupport.parkUntil()` 实现精确到绝对时间的休眠
- `wakeUp()` 方法暴露唤醒接口，新任务加入时可立即唤醒触发器重新判断

### 1.5 与标准库的对比

实际运行中发现，自定义的 `MyScheduleService` 输出比 `ScheduledExecutorService` 更加规律整齐。这是因为：

| 特性 | ScheduledExecutorService | MyScheduleService |
|------|-------------------------|-------------------|
| 执行顺序 | 共享线程，串行执行 | 独立触发，可能并发 |
| 定时精度 | 尽力而为，会动态调整 | 严格遵循设定时间 |
| 实现复杂度 | 高度优化，考虑各种边界情况 | 相对简单的自定义逻辑 |

### 1.6 拓展：Quartz 框架

对于企业级应用，推荐使用成熟的 Quartz 框架，它提供了更灵活的定时任务实现：

```java
// 定义任务
JobDetail job = JobBuilder.newJob(TaskJob.class)
    .withIdentity("job1", "group1")
    .usingJobData("taskName", "任务1")
    .build();

// 定义触发器
Trigger trigger = TriggerBuilder.newTrigger()
    .startAt(new Date(System.currentTimeMillis() + 100))
    .withSchedule(SimpleScheduleBuilder.simpleSchedule()
        .withIntervalInMilliseconds(200)
        .repeatForever())
    .build();

// 调度执行
scheduler.scheduleJob(job, trigger);
```

---

## 二、手搓线程池

### 2.1 项目目的

实现一个简化版的 `ThreadPoolExecutor`，支持核心线程、辅助线程、任务队列和拒绝策略。

```java
// 目标：用 MyThreadPool 替换 ThreadPoolExecutor
MyThreadPool executor = new MyThreadPool(
    corePoolSize, maximumPoolSize, keepAliveTime, unit, 
    workQueue, threadFactory, handler
);
```

### 2.2 核心设计思路

线程池的核心思想是：**线程复用**。避免频繁创建和销毁线程的开销。

- 每个工作线程持续从任务队列中获取任务并执行
- 执行完后不退出，继续获取下一个任务
- 核心线程永久存活，辅助线程空闲超时后自动销毁

### 2.3 最简实现

```java
public class MyThreadPool {
    private int corePoolSize;
    private BlockingQueue<Runnable> workQueue;
    private ThreadFactory threadFactory;
    List<Thread> corePool = new ArrayList<>();

    private final Runnable threadTask = () -> {
        while (true) {
            Runnable task = workQueue.take(); // 阻塞获取
            task.run();
        }
    };

    void execute(Runnable task) {
        if (corePool.size() < corePoolSize) {
            Thread thread = threadFactory.newThread(threadTask);
            corePool.add(thread);
            thread.start();
        }
        workQueue.offer(task);
    }
}
```

**存在的问题：**

- **并发安全**：多个线程同时调用 `execute` 可能创建超过核心数量的线程
- **队列满时**：`offer()` 可能失败，任务被丢弃

### 2.4 完整实现

使用 `AtomicInteger` 安全地跟踪工作线程数量，区分核心线程和辅助线程：

```java
public class MyThreadPool {
    private final AtomicInteger workerCount = new AtomicInteger(0);

    // 核心线程：永久运行
    private final Runnable coreTask = () -> {
        while (true) {
            Runnable task = workQueue.take();
            task.run();
        }
    };

    // 辅助线程：超时自动退出
    private final Runnable supportTask = () -> {
        while (true) {
            Runnable task = workQueue.poll(keepAliveTime, unit);
            if (task == null) break; // 超时退出
            task.run();
        }
    };

    void execute(Runnable task) {
        int currentWorkers = workerCount.get();

        // 1. 尝试创建核心线程
        if (currentWorkers < corePoolSize) {
            if (workerCount.compareAndSet(currentWorkers, currentWorkers + 1)) {
                threadFactory.newThread(coreTask).start();
            }
        }

        // 2. 尝试入队
        if (workQueue.offer(task)) {
            return;
        }

        // 3. 队列已满，创建辅助线程
        currentWorkers = workerCount.get();
        if (currentWorkers < maximumPoolSize) {
            if (workerCount.compareAndSet(currentWorkers, currentWorkers + 1)) {
                threadFactory.newThread(supportTask).start();
            }
        }

        // 4. 线程池已满，执行拒绝策略
        if (!workQueue.offer(task)) {
            handler.rejectedExecution(task, this);
        }
    }
}
```

**关键设计：**

- **CAS 原子操作**：`compareAndSet` 确保线程计数的线程安全
- **核心 vs 辅助**：核心线程用 `take()` 永久阻塞，辅助线程用 `poll(timeout)` 超时退出
- **执行流程**：核心线程 → 任务队列 → 辅助线程 → 拒绝策略，符合 JDK 线程池的标准行为

### 2.5 与 JDK 源码的差距

JDK 的 `ThreadPoolExecutor` 实现更加精细：

- 使用 `ctl` 原子变量同时存储线程池状态和线程数量
- 使用 `ReentrantLock` 保护关键区域
- 支持 `shutdown()` 和 `awaitTermination()` 优雅关闭
- 更复杂的线程创建和回收逻辑

---

## 三、手搓锁（AQS 简化版）

### 3.1 项目目的

实现一个可重入锁，替换 `ReentrantLock` 的效果。测试场景：10 个线程各执行 100 次递减操作，最终结果应为 0。

```java
MyLock lock = new MyLock();
// 替代 ReentrantLock
lock.lock();
cnt[0]--;
lock.unlock();
```

### 3.2 最简实现：CAS 自旋锁

利用 `AtomicBoolean` 和 CAS 操作实现最简单的锁：

```java
public class MyLock {
    private AtomicBoolean state = new AtomicBoolean(false);

    void lock() {
        while (!state.compareAndSet(false, true)) {
            // 空转等待
        }
    }

    void unlock() {
        state.compareAndSet(true, false);
    }
}
```

**问题：**

- **CPU 空转**：得不到锁的线程持续消耗 CPU 资源
- **无归属判断**：任意线程都能解锁，缺乏锁持有者验证

### 3.3 阻塞队列实现

引入等待队列，得不到锁的线程进入队列休眠，避免空转：

```java
public class MyLock {
    private AtomicBoolean state = new AtomicBoolean(false);
    private BlockingQueue<Thread> waiters = new LinkedBlockingQueue<>();

    void lock() throws InterruptedException {
        waiters.offer(Thread.currentThread());
        while (true) {
            // 唤醒条件：队头是自己 且 成功获取锁
            if (waiters.peek() == Thread.currentThread() 
                && state.compareAndSet(false, true)) {
                break;
            }
            LockSupport.park(); // 阻塞当前线程
        }
    }

    void unlock() throws InterruptedException {
        if (Thread.currentThread() != waiters.peek()) {
            throw new RuntimeException("当前线程不是锁的拥有者");
        }
        state.compareAndSet(true, false);
        waiters.poll(); // 自己出队
        Thread waiter = waiters.peek();
        if (waiter != null) {
            LockSupport.unpark(waiter); // 唤醒下一个
        }
    }
}
```

**设计要点：**

- **公平性**：先入队再竞争，保证 FIFO 顺序
- **LockSupport**：比 `wait/notify` 更灵活，支持从外部精确唤醒
- **队头即持有者**：巧妙利用阻塞队列的队头作为锁持有者标识

### 3.4 可重入改造

真正的 `ReentrantLock` 支持重入，即同一个线程可以多次获取同一把锁：

```java
public class MyLock {
    private AtomicBoolean state = new AtomicBoolean(false);
    private BlockingQueue<Thread> waiters = new LinkedBlockingQueue<>();
    private Thread owner = null;
    private int sync = 0; // 重入计数

    void lock() throws InterruptedException {
        // 重入判断
        if (owner == Thread.currentThread()) {
            sync++;
            return;
        }

        waiters.offer(Thread.currentThread());
        while (true) {
            if (waiters.peek() == Thread.currentThread() 
                && state.compareAndSet(false, true)) {
                owner = Thread.currentThread();
                sync = 1;
                break;
            }
            LockSupport.park();
        }
    }

    void unlock() throws InterruptedException {
        if (Thread.currentThread() != waiters.peek()) {
            throw new RuntimeException("当前线程不是锁的拥有者");
        }
        
        sync--;
        if (sync == 0) {
            state.compareAndSet(true, false);
            waiters.poll();
            owner = null;
            Thread waiter = waiters.peek();
            if (waiter != null) {
                LockSupport.unpark(waiter);
            }
        }
    }
}
```

**重入机制：**

- `sync` 计数器记录重入次数
- 同一线程再次获取锁时直接递增计数，无需竞争
- 解锁时递减计数，只有计数归零时才真正释放锁

### 3.5 无锁队列实现

使用阻塞队列本质上是"依赖循环"（队列内部也用了锁）。更彻底的实现是使用无锁链表队列：

```java
public class MyLinkedBlockingQueue<E> {
    static class Node<E> {
        volatile E item;
        AtomicReference<Node<E>> next = new AtomicReference<>();
        Node(E item) { this.item = item; }
    }

    private final AtomicReference<Node<E>> head;
    private final AtomicReference<Node<E>> tail;

    public MyLinkedBlockingQueue() {
        Node<E> dummy = new Node<>(null);
        head = new AtomicReference<>(dummy);
        tail = new AtomicReference<>(dummy);
    }

    public void offer(E e) {
        Node<E> newNode = new Node<>(e);
        while (true) {
            Node<E> curTail = tail.get();
            Node<E> next = curTail.next.get();
            if (curTail == tail.get()) {
                if (next == null) {
                    if (curTail.next.compareAndSet(null, newNode)) {
                        tail.compareAndSet(curTail, newNode);
                        return;
                    }
                } else {
                    tail.compareAndSet(curTail, next); // 帮助推进
                }
            }
        }
    }

    public E poll() {
        while (true) {
            Node<E> curHead = head.get();
            Node<E> curTail = tail.get();
            Node<E> next = curHead.next.get();
            if (curHead == head.get()) {
                if (curHead == curTail) {
                    if (next == null) return null;
                    tail.compareAndSet(curTail, next);
                } else {
                    E item = next.item;
                    if (head.compareAndSet(curHead, next)) {
                        next.item = null;
                        return item;
                    }
                }
            }
        }
    }

    public E peek() {
        Node<E> first = head.get().next.get();
        return first != null ? first.item : null;
    }
}
```

**关键技术：**

- **`volatile` 关键字**：保证变量的可见性，禁止编译器优化
- **CAS 无锁操作**：通过 `compareAndSet` 实现线程安全，无需加锁
- **帮助推进**：当发现 `tail` 指针滞后时，帮助其他线程推进指针

---

## 总结

通过从零实现这三个核心并发组件，深入理解了：

1. **定时任务**：触发器模式 + 优先队列是高效调度的关键
2. **线程池**：线程复用 + CAS 原子计数 + 分级拒绝策略
3. **锁机制**：CAS 自旋 → 阻塞队列 → 可重入 → 无锁队列的演进路径

这些组件虽然简化，但核心思想与 JDK 标准库一脉相承。理解它们的工作原理，对于日常开发中的并发问题排查和性能优化大有裨益。

> 学习来源：[B站 - 学Java的生生](https://space.bilibili.com/7968519)
