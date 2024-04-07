---
title: concurrency
date: 2024-04-07 18:48:34
categories:
- OS
---

### 1.1 lock

#### 1、怎么实现？

##### 1.1 仅凭硬件实现

- lock() 关闭中断，unlock() 恢复中断

  - 需要对应用程序的过度信任
  - 不适用于多核处理器
  - 长时间关闭中断导致中断丢失

  适用于有限上下文中做互斥原语，例如OS内部的并发控制（OS总是信任自己的

- old Test-And-Set (&lock, new)

  - 返回旧值，并原子地设置新值
  - 是自旋锁

- old Compare-And-Swap(&lock, expected, new)

  - 返回旧值，如果预期值和旧值相等则写入新值
  - 比TAS更加强大

- Load-Linked and Store-Conditional

  - 两个指令成对工作，但这两个指令作为整体并不具有原子性
  - 获取到0时说明锁未被锁定，尝试取锁，如果没有其他人更新flag，则此次写入成功，否则失败重新循环。
  - 和TAS语义上很像

  ```c
  void lock(lock_t *lock) {
      while (1) {
          // spin until it’s zero
          while (LoadLinked(&lock->flag) == 1); 
          if (StoreConditional(&lock->flag, 1) == 1)
              return; // if set-to-1 was success: done
          // otherwise: try again
      }
  }
  
  void lock(lock_t *lock) {
      while (LoadLinked(&lock->flag) || !StoreConditional(&lock->flag, 1));
  }
  ```

- Fetch-And-Add

  - 保证了一定的公平性，取到票后该线程一定会在未来某时刻进入临界区

  ```c
  typedef struct __lock_t {
       int ticket;
       int turn;
  } lock_t;
  
  void lock_init(lock_t *lock) {
      lock->ticket = 0;
      lock->turn = 0;
  }
  
  void lock(lock_t *lock) {
      // 每人领一张带有时间戳的票
      int myturn = FetchAndAdd(&lock->ticket);
     	// 还没轮到就自旋
      while (lock->turn != myturn);
  }
  
  void unlock(lock_t *lock) {
  	lock->turn = lock->turn + 1;
  }
  ```

##### 1.2 OS参与

目的是为了避免自旋，提高性能，还能保证特定情况下的正确性（优先级翻转问题，比如说高优先级IO阻塞，低优先级获得了锁，IO完成后重新取得运行权但无法获得锁，一直自旋。。。

- yield() 代替 spin()
  - 当前线程不能获取锁时，running -> ready
  - 线程数量较多时，就算让步之后，继续执行的也不一定是锁的持有者，而且线程切换开销大
  - 无法保证一个需要进入临界区的线程何时能够进入（饥饿）
- sleep() 代替 spin()
  - 获取不到锁时进入阻塞队列，锁的持有者唤醒队首，需要注意的是，锁的持有者并未释放锁（除非没人阻塞在上面），相当于直接传递给了队首，被唤醒的线程直接进入临界区，而不需要再去竞争（类似于上面的FAA已经买到了票，进而不会出现饥饿。
  - 关锁和开锁都需要竞争一个自旋锁（自旋的时间是很少的，因为并不涉及用户定义的临界区代码执行）

##### 1.3 纯软件实现（Dekker、Peterson）

- 在现代硬件上无法工作，因为宽松(Relaxed)的内存一致性模型

#### 2、如何评估

- 正确性
- 公平性
- 性能
  - 单核
    - 自旋锁效率低
  - 多核
    - 自旋锁，当线程数约等于CPU数时还可以



### 1.2 Lock-based Concurrent Data Structures

一把大锁保平安，但是损失了单核到多核的扩展性（不能发挥多核优势），理应每个核和单核保持相近性能。

关键在于多核之间尽量少的争用锁。

小心控制流改变时附近的锁的获取和释放。

#### 并发计数器

各核利用局部锁更新本地计数器，达到阈值后尝试更新全局计数器，这个阈值会影响性能和准确性。（为什么不直接在读的时候再更新，也就是懒加载，这样就能保证准确性？

#### 并发链表和队列

#### 并发哈希表

给每个桶上一把锁，伸缩性良好。

### 1.3 Condition variable

#### 为什么要引入条件变量？

实现同步，即多个线程到达一个相同的状态。

#### join() 的朴素实现

```C
volatile int done = 0;

void *child(void *arg) {
    printf("child\n");
    done = 1;
    return NULL;
}

int main(int argc, char *argv[]) {
     printf("parent: begin\n");
     pthread_t c;
     Pthread_create(&c, NULL, child, NULL); // child
     // 父子线程在同一个CPU上时，自旋检查一个不会被改变的值显然是浪费
     while (done == 0);
     printf("parent: end\n");
     return 0;
}
```

#### join() 的 CV 实现

为什么 `while (done == 0)`而不是`if (done == 0)`？

> 详见生产者、消费者问题

为什么调用 `signal()` / `wait()` 时总要先获得锁？

> 对于 signal() ：考虑父线程准备睡眠前被打断，切换到子线程执行 `thr_exit()`，再回到父线程后，父线程将无法被唤醒。当然也有不需要上锁的情况，但是总上锁是不会错的。
>
> 对于 wait() ：这是语义决定的
>
> 1. 调用时已经获得锁 
> 2. 睡眠后释放锁
> 3. 返回前**尝试**获取锁（注意是尝试！！）

`done` 的必要性？

> 子线程创建后立刻执行 `thr_exit()`，再回到父线程后，父线程将无法被唤醒。

```c
int done = 0;
pthread_mutex_t m = PTHREAD_MUTEX_INITIALIZER;
pthread_cond_t c = PTHREAD_COND_INITIALIZER;

void thr_exit() {
    Pthread_mutex_lock(&m);
    done = 1;
    Pthread_cond_signal(&c);
    Pthread_mutex_unlock(&m);
}

void *child(void *arg) {
    printf("child\n");
    thr_exit();
    return NULL;
}

void thr_join() {
    Pthread_mutex_lock(&m);
    while (done == 0)
        Pthread_cond_wait(&c, &m);
    Pthread_mutex_unlock(&m);
}

int main(int argc, char *argv[]) {
    printf("parent: begin\n");
    pthread_t p;
    Pthread_create(&p, NULL, child, NULL);
    thr_join();
    printf("parent: end\n");
    return 0;
}
```

#### 生产者、消费者问题

为什么是 `while` 而不是 `if` ？

>wait() 的语义的第三条只是说再返回前尝试获取锁，如果在获取到锁之前有其他线程先拿到锁并改变了条件后释放锁，比如说，有个消费者被唤醒，但是来了另外一个消费者并且先一步拿到互斥锁并把东西吃掉后释放锁，那么此时之前被唤醒的线程拿到锁后应当再判断一遍真有东西吗，否则就出错了。

为什么需要两个条件变量？

> signal() 只唤醒一个线程，防止消费者唤醒消费者而没有唤醒生产者导致都阻塞在同一个条件变量上，也就是 signal() 应当更准确。

只有一个条件变量真不行吗？

> broadcast() 而不是 signal() 就可以了，错误唤醒的或者没抢到锁的就又睡过去了。
>
> 其实这种写法的正确性会更加的显然，jyy 推荐。

### 1.4 Semaphores

用互斥锁不能实现同步吗？

>可以，用起来和大小为1的信号量一样，但是属于 `Undefined Behavior` ，POSIX 标准规定锁的获取和释放应当在同一个线程中完成。

信号量属于互斥锁的一个自然扩展，既可以做互斥，也可以来做同步。

在适合信号量的问题中使用会很优雅，且正确性相对容易保证，但信号量并不是万能的。

#### 两种典型应用

1. 实现一次临时的  `happens-before`，也就是上面提到的用互斥锁来实现同步，不过这对于信号量来说没有 UB 的限制。
2. 管理计数型资源

#### 生产者、消费者问题

当缓冲区大小为1时，一个信号量可以同时作为访问缓冲区的锁和缓冲区是否填满的条件变量。

如何避免死锁？

> 缩小互斥锁的范围

#### 信号量的实现

感觉很像锁 + 条件变量实现的生产者消费者。

```C
typedef struct __Zem_t {
    int value;
    pthread_cond_t cond;
    pthread_mutex_t lock;
} Zem_t;

// only one thread can call this
void Zem_init(Zem_t *s, int value) {
    s->value = value;
    Cond_init(&s->cond);
    Mutex_init(&s->lock);
}

void Zem_wait(Zem_t *s) {
    Mutex_lock(&s->lock);
    while (s->value <= 0)
        Cond_wait(&s->cond, &s->lock);
    s->value--;
    Mutex_unlock(&s->lock);
}

void Zem_post(Zem_t *s) {
    Mutex_lock(&s->lock);
    s->value++;
    Cond_signal(&s->cond);
    Mutex_unlock(&s->lock);
}

```

#### 缺陷

##### 哲学家吃饭问题

信号量可以实现（限制四人在桌上 / 先拿编号小的叉子），确保不会都拿到一只然后死锁，但是不如条件变量简单，而且随着条件的复杂性增加，条件变量的简单性会更明显。

##### 用信号量实现条件变量（失败的尝试）

谨记  `wait()` 的语义。

```c
void wait(struct condvar *cv, mutex_t *mutex) {
    mutex_lock(&cv->lock);
    cv->nwait++;
    mutex_unlock(&cv->lock);

    // 睡眠前释放锁
    mutex_unlock(mutex);
    
    /*
    	如果在这里被打断，并发生 broadcast()，由于 nwait 已经加过了，所以此时 cv->sleep 变为 1，原本应当留给当前线程的球被别的线程取走，
    	具体地，在生产者消费者问题中，如果使用一个信号量，即使使用 broadcast() 也会出现生产者因为错误的唤醒生产者而无法唤醒应当唤醒的消费者，当唤醒的生产者发现无法生产睡去后，出现死锁。
    	简而言之，如此实现的 wait() 和 broadcast() 配合使用时会有问题，所以需要实现成原子操作。
    */
    
    // 睡眠
    P(&cv->sleep);

    // 尝试重新获取锁
    mutex_lock(mutex);
}

void broadcast(struct condvar *cv) {
    mutex_lock(&cv->lock);

    for (int i = 0; i < cv->nwait; i++) {
        V(&cv->sleep);
    }
    cv->nwait = 0;

    mutex_unlock(&cv->lock);
}

```
