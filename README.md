# 操作系统课程设计报告

------

## Pintos线程管理

### 一、实验内容

#### 1.1	实现优先级调度

​	当一个线程被添加到具有比当前运行的线程更高优先级的就绪列表时，当前线程应该立即将处理器分给新线程中。类似地，当线程正在等待锁，信号量或条件变量时，应首先唤醒优先级最高的等待线程。线程应可以随时提高或降低自己的优先级，但降低其优先级而不再具有最高优先级时必须放弃CPU。

#### 1.2	实现多级反馈调度

​	实现多级反馈队列调度程序，减少在系统上运行作业的平均响应时间。这里维持了64个队列，每个队列对应一个优先级，从PRI_MIN到PRI_MAX。.通过一些公式计算来计算出线程当前的优先级，系统调度的时候会从高优先级队列开始选择线程执行，这里线程的优先级随着操作系统的运转数据而动态改变。

### 二、主要功能

#### 2.1	重写timer_sleep()函数

##### 2.1.1	原方法

```c
 /* Sleeps for approximately TICKS timer ticks.  Interrupts must be turned on. */
void
timer_sleep (int64_t ticks)
{
  int64_t start = timer_ticks ();
  ASSERT (intr_get_level () == INTR_ON);
  while (timer_elapsed (start) < ticks)
  	thread_yield();
}
```

​	对原方法的解释说明：首先调用timer_ticks()获得ticks的当前值，然后在该ticks时间段内，调用thread_yield()，thread_yield()方法的作用是把当前线程放到就绪队列里， 然后重新schedule切换下一个线程运行，这样可以在ticks时间段内，该线程一直在ready队列，不被执行，从而达到alarm的目的。这样做的缺点是线程不断在cpu就绪队列和running队列之间来回， 占用了cpu资源。

##### 2.1.2	函数重新实现

###### （1）数据结构

​	在线程结构体中加上新的变量ticks_blocked并在线程被创建时初始化为0，该变量是用来记录线程阻塞的时间：

```c
int64_t ticks_blocked;/*记录线程阻塞的时间*/
```

###### （2）算法

​	首先进行分析我们的实现逻辑：调用timer_sleep的时候把线程阻塞掉，根据操作系统自身的时钟中断实现对线程状态的检测， 每次检测将ticks_blocked减1, 如果减到0就唤醒这个线程。这样就无需将线程一遍遍的从running队列放到ready队列，减少cpu资源的消耗。

```c
/* Sleeps for approximately TICKS timer ticks.  Interrupts must be turned on. */
void
timer_sleep (int64_t ticks)
{
  if (ticks <= 0)
  {
    return;
  }
  ASSERT (intr_get_level () == INTR_ON);
  enum intr_level old_level = intr_disable ();
  struct thread *current_thread = thread_current ();
  current_thread->ticks_blocked = ticks;
  thread_block ();
  intr_set_level (old_level);
}
```

​	在操作系统的时钟中断处理函数timer_interrupt中，添加线程检测函数 thread_foreach（），使每次系统中断时都会对每个线程检测ticks_blocked的状态,此处的检测必须同时满足两个条件，因为操作系统中存在进程因为等待锁而阻塞的状态，而不是主动调取timer_sleep函数的结果。添加的代码如下所示：

```c
 thread_foreach (blocked_thread_check, NULL);
```

​	在thread_foreach中对每个线程都执行blocked_thread_check这个函数判断ticks_blocked状态：

```c
void
blocked_thread_check(struct thread *t,void *aux UNUSED)
{
    if(t->status==THREAD_BLOCKED&&t->ticks_blocked>0)
    {
        t->ticks_blocked--;
        if(t->ticks_blocked==0)
        {
            thread_unblock(t);
        }
    }
}
```

​	函数说明：该函数用来检测该线程ticks_blocked是否为0，如果不为0就-1，为0就调用函数thread_unblock(t)把线程放入就绪队列。

thread_unblock就是在线程ticks_blocked==0时放到就绪队列里准备运行，实现如下所示：

```c
void
thread_unblock (struct thread *t) 
{
  enum intr_level old_level;

  ASSERT (is_thread (t));

  old_level = intr_disable ();
  ASSERT (t->status == THREAD_BLOCKED);
  list_push_back (&ready_list, &t->elem);
  t->status = THREAD_READY;
  intr_set_level (old_level);
}
```

#### 2.2	实现优先级调度

##### 2.2.1	维护优先级队列

###### 	（1）list_push_back函数的修改

​	由于pintos的基本实现是把线程直接放进就绪队列尾部，所以当前的就绪队列是先进先出队列，没有优先级的顺序关系，要想实现优先级调度，需要将就绪队列变为一个优先级队列。

​	我们可以借助list_insert_ordered()，插入队列的时候按优先级高低进行排序插入，使优先级高的线程排在优先级低的线程之前。因此可以将thread_init，thread_unblock和thread_yield函数里的list_push_back全都改成list_insert_ordered来维护优先级队列。修改的代码如下所示：

```c
list_insert_ordered (&ready_list, &t->elem, (list_less_func *) &thread_cmp_priority, NULL);
```

###### 	（2）实现比较函数

​	实现比较函数thread_cmp_priority如下所示：

```c
bool
thread_cmp_priority(const struct list_elem *a,const struct list_elem *b,void *aux UNUSED)
{
    return list_entry(a,struct thread,elem)->priority>list_entry(b,struct thread,elem)->priority;
}
```

​	函数说明：按照优先级的大小进行比较。

##### 2.2.2	抢占式调度的实现

​	我们需要实现在创建一个新线程的时候， 如果新线程优先级高于当前线程就先执行创建的线程。

​	首先查看基于抢占式调度的测试。

```c
void
test_priority_change (void) 
{
  /* This test does not work with the MLFQS. */
  ASSERT (!thread_mlfqs);

  msg ("Creating a high-priority thread 2.");
  thread_create ("thread 2", PRI_DEFAULT + 1, changing_thread, NULL);
  msg ("Thread 2 should have just lowered its priority.");
  thread_set_priority (PRI_DEFAULT - 2);
  msg ("Thread 2 should have just exited.");
}

static void
changing_thread (void *aux UNUSED) 
{
  msg ("Thread 2 now lowering priority.");
  thread_set_priority (PRI_DEFAULT - 1);
  msg ("Thread 2 exiting.");
}
```

​	函数说明： 测试线程创建了一个PRI_DEFAULT+1优先级的内核线程thread2，由于thread2优先级高，所以线程执行直接切换到thread2， thread1阻塞， 然后thread2执行的时候调用changing_thread， 又把自身优先级调为PRI_DEFAULT-1，此时测试线程的优先级大于thread2， 线程切换到测试线程， 然后测试线程又把自己优先级改成PRI_DEFAULT-2,这个时候thread2又高于测试线程了， 所以执行thread2。

​	通过上述分析，我们需要在设置一个线程优先级时重新考虑所有线程执行顺序， 重新安排执行顺序。所以在线程设置优先级的时候需要调用thread_yield把当前线程重新插入到优先级就绪队列中执行。

​	设置优先级的函数的实现如下：

```c
/* Sets the current thread's priority to NEW_PRIORITY. */
void
thread_set_priority (int new_priority) 
{
    current_thread->priority=new_priority;
    thread_yield();
}
```

​	并且更改thread_create线程创建函数，在thread_unblock (t)也就是将线程放入就绪队列中后，判断如果当前线程的优先级小于新创建的线程的优先级，需要重新将新创建的线程加入到优先队列中。修改的代码如下所示：

```c
thread_unblock (t);

if(thread_current()->priority<priority)
{
   thread_yield();
}
```

##### 2.2.3	优先级捐赠的实现

###### 	（1）测试代码的实现分析

- test_priority_donate_one

```c
void
test_priority_donate_one (void) 
{
  struct lock lock;

  /* This test does not work with the MLFQS. */
  ASSERT (!thread_mlfqs);

  /* Make sure our priority is the default. */
  ASSERT (thread_get_priority () == PRI_DEFAULT);

  lock_init (&lock);
  lock_acquire (&lock);
  thread_create ("acquire1", PRI_DEFAULT + 1, acquire1_thread_func, &lock);
  msg ("This thread should have priority %d.  Actual priority: %d.",PRI_DEFAULT + 1, thread_get_priority ());
  thread_create ("acquire2", PRI_DEFAULT + 2, acquire2_thread_func, &lock);
  msg ("This thread should have priority %d.  Actual priority: %d.",PRI_DEFAULT + 2, thread_get_priority ());
  lock_release (&lock);
  msg ("acquire2, acquire1 must already have finished, in that order.");
  msg ("This should be the last line before finishing this test.");
}
```

​	test_priority_donate_one分析说明：将当前线程称为original_thread，是优先级为PRI_DEFAULT的线程，在original_thread线程中创建并初始化了一个互斥锁，然后调用lock_acquire (&lock);占用互斥锁。接着创建了一个优先级为PRI_DEFAULT + 1的线程acquire1，由于优先级抢占，会立刻调acquire1，执行acquire1_thread_func，在acquire1_thread_func中acquire1也申请同一个互斥锁，由于互斥锁已经被占用，因此acquire1会被阻塞在互斥锁的等待队列。然后调度original_thread线程，original_thread线程接着创建优先级为PRI_DEFAULT + 2的线程acquire2，优先级为PRI_DEFAULT+2， 这里调用和acquire1一致， 然后original_thread继续输出msg。接着original_thread释放了这个锁（V操作）， 释放的过程会触发被锁着的线程acquire1, acquire2， 然后根据优先级调度， 先执行acquire2, 再acquire1, 最后再执行original_thread。那么这里应该是acquire2, acquire1分别释放锁然后输出msg， 最后original_thread再输出msg。

​	实现说明：在一个线程获取一个锁的时候， 如果拥有这个锁的线程优先级比自己低就提高它的优先级，然后在这个线程释放掉这个锁之后把原来拥有这个锁的线程改回原来的优先级。

- test_priority_donate_multiple

```c
void
test_priority_donate_multiple (void) 
{
  struct lock a, b;

  /* This test does not work with the MLFQS. */
  ASSERT (!thread_mlfqs);

  /* Make sure our priority is the default. */
  ASSERT (thread_get_priority () == PRI_DEFAULT);

  lock_init (&a);
  lock_init (&b);

  lock_acquire (&a);
  lock_acquire (&b);

  thread_create ("a", PRI_DEFAULT + 1, a_thread_func, &a);
  msg ("Main thread should have priority %d.  Actual priority: %d.",
       PRI_DEFAULT + 1, thread_get_priority ());

  thread_create ("b", PRI_DEFAULT + 2, b_thread_func, &b);
  msg ("Main thread should have priority %d.  Actual priority: %d.",PRI_DEFAULT + 2, thread_get_priority ());

  lock_release (&b);
  msg ("Thread b should have just finished.");
  msg ("Main thread should have priority %d.  Actual priority: %d.",PRI_DEFAULT + 1, thread_get_priority ());

  lock_release (&a);
  msg ("Thread a should have just finished.");
  msg ("Main thread should have priority %d.  Actual priority: %d.",PRI_DEFAULT, thread_get_priority ());
}
```

​	test_priority_donate_multiple分析说明：original_thread是优先级为PRI_DEFAULT的线程，然后创建2个锁，接着创建优先级为PRI_DEFAULT+1的线程a，把锁a丢给这个线程的执行函数。这时候线程a抢占式地调用a_thread_func，获取了a这个锁，阻塞。然后original_thread输出线程优先级的msg。然后再创建一个线程优先级为PRI_DEFAULT+2的线程b， 和a一样做同样的操作。 然后original_thread释放掉了锁b，此时线程b被唤醒，抢占式执行b_thread_func。然后original再输出msg，a同上。线程b结束后，original_thread线程的优先级并没有直接恢复为默认值，而是保持a线程捐赠的优先级。造成这个结果的原因是，main线程占用了多个锁。

​	实现说明：该测试函数测试的行为实际是多锁情况下优先级逻辑的正确性。 可以通过释放一个锁的时候， 将该锁的拥有者改为该线程被捐赠的第二优先级，若没有其余捐赠者， 则恢复原始优先级来实现，并且需要增加一个数据结构来记录所有对这个线程有捐赠行为的线程。	

- test_priority_donate_nest

```c
void
test_priority_donate_nest (void) 
{
  struct lock a, b;
  struct locks locks;

  /* This test does not work with the MLFQS. */
  ASSERT (!thread_mlfqs);

  /* Make sure our priority is the default. */
  ASSERT (thread_get_priority () == PRI_DEFAULT);

  lock_init (&a);
  lock_init (&b);

  lock_acquire (&a);

  locks.a = &a;
  locks.b = &b;
  thread_create ("medium", PRI_DEFAULT + 1, medium_thread_func, &locks);
  thread_yield ();
  msg ("Low thread should have priority %d.  Actual priority: %d.", PRI_DEFAULT + 1, thread_get_priority ());

  thread_create ("high", PRI_DEFAULT + 2, high_thread_func, &b);
  thread_yield ();
  msg ("Low thread should have priority %d.  Actual priority: %d.", PRI_DEFAULT + 2, thread_get_priority ());

  lock_release (&a);
  thread_yield ();
  msg ("Medium thread should just have finished.");
  msg ("Low thread should have priority %d.  Actual priority: %d.", PRI_DEFAULT, thread_get_priority ());
}
```

​	test_priority_donate_nest分析说明：首先original_thread线程中创建了两个锁a、b。original_thread线程占用了锁a，然后创建优先级为PRI_DEFAULT + 1的medium线程，medium抢占original_thread，medium中先申请锁b，再申请锁a，锁b申请成功，锁a由于已经被占用，medium阻塞在锁b上。然后回到original_thread线程，创建优先级为PRI_DEFAULT + 2的high线程，同样，high中申请锁b，但是被阻塞在锁b上。

​	实现说明：优先级嵌套问题，优先级提升具有连环效应， medium被提升了， 此时它被锁捆绑的low线程应该跟着一起提升，需要一个记录这个线程被锁于哪个线程的数据结构。

- test_priority_donate_chain

```c
void
test_priority_donate_chain (void) 
{
  int i;  
  struct lock locks[NESTING_DEPTH - 1];
  struct lock_pair lock_pairs[NESTING_DEPTH];

  /* This test does not work with the MLFQS. */
  ASSERT (!thread_mlfqs);

  thread_set_priority (PRI_MIN);

  for (i = 0; i < NESTING_DEPTH - 1; i++)
    lock_init (&locks[i]);

  lock_acquire (&locks[0]);
  msg ("%s got lock.", thread_name ());

  for (i = 1; i < NESTING_DEPTH; i++)
    {
      char name[16];
      int thread_priority;

      snprintf (name, sizeof name, "thread %d", i);
      thread_priority = PRI_MIN + i * 3;
      lock_pairs[i].first = i < NESTING_DEPTH - 1 ? locks + i: NULL;
      lock_pairs[i].second = locks + i - 1;

      thread_create (name, thread_priority, donor_thread_func, lock_pairs + i);
      msg ("%s should have priority %d.  Actual priority: %d.",
          thread_name (), thread_priority, thread_get_priority ());

      snprintf (name, sizeof name, "interloper %d", i);
      thread_create (name, thread_priority - 1, interloper_thread_func, NULL);
    }

  lock_release (&locks[0]);
  msg ("%s finishing with priority %d.", thread_name (), thread_get_priority ());
}
```

​	test_priority_donate_chain分析说明：这个测试是一个链式优先级捐赠， 本质测试的还是多层优先级捐赠逻辑的正确性，释放掉一个锁之后， 如果当前线程不被捐赠即马上改为原来的优先级， 抢占式调度。

- test_priority_donate_sema

```c
void
test_priority_donate_sema (void) 
{
  struct lock_and_sema ls;

  /* This test does not work with the MLFQS. */
  ASSERT (!thread_mlfqs);

  /* Make sure our priority is the default. */
  ASSERT (thread_get_priority () == PRI_DEFAULT);

  lock_init (&ls.lock);
  sema_init (&ls.sema, 0);
  thread_create ("low", PRI_DEFAULT + 1, l_thread_func, &ls);
  thread_create ("med", PRI_DEFAULT + 3, m_thread_func, &ls);
  thread_create ("high", PRI_DEFAULT + 5, h_thread_func, &ls);
  sema_up (&ls.sema);
  msg ("Main thread finished.");
}
```

​	test_priority_donate_sema分析说明：包含了信号量和锁的混合触发， 实际上还是信号量在起作用， 因为锁是由信号量实现的。

- test_priority_donate_lower

```c
void
test_priority_donate_lower (void) 
{
  struct lock lock;

  /* This test does not work with the MLFQS. */
  ASSERT (!thread_mlfqs);

  /* Make sure our priority is the default. */
  ASSERT (thread_get_priority () == PRI_DEFAULT);

  lock_init (&lock);
  lock_acquire (&lock);
  thread_create ("acquire", PRI_DEFAULT + 10, acquire_thread_func, &lock);
  msg ("Main thread should have priority %d.  Actual priority: %d.",
       PRI_DEFAULT + 10, thread_get_priority ());

  msg ("Lowering base priority...");
  thread_set_priority (PRI_DEFAULT - 10);
  msg ("Main thread should have priority %d.  Actual priority: %d.",
       PRI_DEFAULT + 10, thread_get_priority ());
  lock_release (&lock);
  msg ("acquire must already have finished.");
  msg ("Main thread should have priority %d.  Actual priority: %d.",
       PRI_DEFAULT - 10, thread_get_priority ());
}
```

​	test_priority_donate_lower分析说明：original_thread创建并占有了一个锁，此时创建线程acquire，优先级为PRI_DEFAULT + 10，抢占后申请同一个锁，被阻塞。回到original_thread，original_thread修改自己的优先级为PRI_DEFAULT - 10，然后释放锁，acquire被唤醒。

​	实现说明：该测试函数测试的是当修改一个被捐赠的线程优先级的时候的行为正确性。

- test_priority_sema

```c
void
test_priority_sema (void) 
{
  int i;
  
  /* This test does not work with the MLFQS. */
  ASSERT (!thread_mlfqs);

  sema_init (&sema, 0);
  thread_set_priority (PRI_MIN);
  for (i = 0; i < 10; i++) 
    {
      int priority = PRI_DEFAULT - (i + 3) % 10 - 1;
      char name[16];
      snprintf (name, sizeof name, "priority %d", priority);
      thread_create (name, priority, priority_sema_thread, NULL);
    }

  for (i = 0; i < 10; i++) 
    {
      sema_up (&sema);
      msg ("Back in main thread."); 
    }
}
```

​	test_priority_sema分析说明：主线程中将信号量sema的值初始化为0，将自己的优先级设置为最低，然后创建10个优先级不等的线程，并且每个线程调用sema_down函数，由于信号量初始化为0，sema_down操作会将子线程阻塞在信号量sema的等待队列。

​	实现说明：每次运行的线程释放信号量时必须确保优先级最高的线程继续执行，即线程唤醒的时候是优先级高的先唤醒，信号量的等待队列是优先级队列。

- test_priority_condvar

```c
void
test_priority_condvar (void) 
{
  int i;
  
  /* This test does not work with the MLFQS. */
  ASSERT (!thread_mlfqs);

  lock_init (&lock);
  cond_init (&condition);

  thread_set_priority (PRI_MIN);
  for (i = 0; i < 10; i++) 
    {
      int priority = PRI_DEFAULT - (i + 7) % 10 - 1;
      char name[16];
      snprintf (name, sizeof name, "priority %d", priority);
      thread_create (name, priority, priority_condvar_thread, NULL);
    }

  for (i = 0; i < 10; i++) 
    {
      lock_acquire (&lock);
      msg ("Signaling...");
      cond_signal (&condition, &lock);
      lock_release (&lock);
    }
}
```

​	test_priority_condvar分析说明：实现初始化了一个锁变量和一个条件变量，然后将主线程的优先级设置为最低，然后创建10个优先级在21~30之间的子线程，子线程会立刻抢占主线程。cond_wait与信号量的wait操作相似，将线程阻塞在条件变量的等待队列上。

​	实现说明：函数中的condition变量维护了一个waiters用于存储等待接受条件变量的线程，从结果来看， 本测试的要求是 condition的waiters队列是优先级队列。

###### （2）数据结构

​	根据上述分析，我们需要在thread结构体中加入下列成员：

```c
int base_priority;  //基础优先级
struct list locks;  //线程已持有的锁
struct lock *lock_waiting;  //线程等待的锁
```

​	并且给lock结构体加入下列成员：

```c
struct list_elem elem;  //线程中locks的链表元素
int max_priority;   //获取锁的线程的最大优先级
```

###### （3）算法

- ​	修改lock_acquire函数


```c
void
lock_acquire (struct lock *lock)
{
  struct thread *current_thread=thread_current();
  struct lock *l;
  enum intr_level old_level;

  ASSERT (lock != NULL);
  ASSERT (!intr_context ());
  ASSERT (!lock_held_by_current_thread (lock));

  if(lock->holder!=NULL&&!thread_mlfqs)
  {
      current_thread->lock_waiting=lock;
      l=lock;
      while(l&&current_thread->priority>l->max_priority)
      {
          l->max_priority=current_thread->priority;
          thread_donate_priority(l->holder);
          l=l->holder->lock_waiting;
      }
  }

  sema_down (&lock->semaphore);

  old_level=intr_disable();
  current_thread=thread_current();
  if(!thread_mlfqs)
  {
      current_thread->lock_waiting=NULL;
      lock->max_priority=current_thread->priority;
      thread_hold_the_lock(lock);
  }
  lock->holder = current_thread;

  intr_set_level(old_level);
}
```

​	函数说明：线程申请互斥锁，如果互斥锁已经被别的线程占用，那么将被阻塞在lock中的信号量的等待队列上。在P操作之前递归地实现优先级捐赠， 然后在被唤醒之后，成为这个锁的拥有者。

- 实现上述函数中的thread_hold_the_lock函数：


```c
void thread_hold_the_lock(struct lock *lock)
{
    enum intr_level old_level=intr_disable();
    list_insert_ordered(&thread_current()->locks,&lock->elem,lock_cmp_priority,NULL);

    if(lock->max_priority>thread_current()->priority)
    {
        thread_current()->priority=lock->max_priority;
        thread_yield();
    }

    intr_set_level(old_level);
}
```

​	函数说明：通过直接修改锁的最高优先级实现优先级捐赠。

- 实现上述函数中的thread_donate_priority函数：


```c
void thread_donate_priority(struct thread *t)
{
    enum intr_level old_level=intr_disable();
    thread_update_priority(t);

    if(t->status==THREAD_READY)
    {
        list_remove(&t->elem);
        list_insert_ordered(&ready_list,&t->elem, thread_cmp_priority,NULL);
    }
    intr_set_level(old_level);
}
```

​	函数说明：调用thread_update_priority的时候实现现成优先级更新。

- 实现锁队列排序函数lock_cmp_priority：


```c
bool lock_cmp_priority(const struct list_elem *a,const struct list_elem *b,void *aux UNUSED)
{
    return list_entry(a,struct lock,elem)->max_priority>list_entry(b,struct lock,elem)->max_priority;
}
```

​	函数说明：维护锁队列的优先级。

- 修改lock_release函数：


```c
 if (!thread_mlfqs)
    thread_remove_lock (lock);
```

​	函数说明：当不进行mlfqs测试时，释放锁。

- 实现thread_remove_lock函数：


```c
void thread_remove_lock(struct lock *lock)
{
    enum intr_level old_level=intr_disable();
    list_remove(&lock->elem);
    thread_update_priority(thread_current());
    intr_set_level(old_level);
}
```

​	函数说明：当释放掉一个锁的时候， 当前线程的优先级可能发生变化，用thread_update_priority来处理这个逻辑。

- 实现thread_update_priority函数：


```c
void thread_update_priority(struct thread *t)
{
    enum intr_level old_level=intr_disable();
    int max_priority=t->base_priority;
    int lock_priority;

    if(!list_empty(&t->locks))
    {
        list_sort(&t->locks,lock_cmp_priority,NULL);
        lock_priority=list_entry(list_front(&t->locks),struct lock,elem)->max_priority;
        if (lock_priority>max_priority)
        {
            max_priority=lock_priority;
        }
    }
    t->priority=max_priority;
    intr_set_level(old_level);
}
```

​	函数说明：如果这个线程还占有锁， 就先获取这个线程拥有锁的最大优先级，如果这个优先级比base_priority大的话更新的应该是被捐赠的优先级。

- 在init_thread中加入初始化：


```c
t->base_priority = priority;
list_init (&t->locks);
t->lock_waiting = NULL;
```

- 修改thread_set_priority函数：


```c
void
thread_set_priority (int new_priority) 
{
    if(thread_mlfqs)
    {
        return;
    }

    enum intr_level old_level=intr_disable();

    struct thread *current_thread=thread_current();
    int old_priority=current_thread->priority;
    current_thread->base_priority=new_priority;

    if (list_empty(&current_thread->locks)||new_priority>old_priority)
    {
        current_thread->priority=new_priority;
        thread_yield();
    }
    intr_set_level(old_level);
    thread_current()->priority=new_priority;
}
```

​	函数说明：设置当前线程的优先级。

- 把condition的队列改成优先级队列，修改cond_signal函数：


```c
void
cond_signal (struct condition *cond, struct lock *lock UNUSED) 
{
  ASSERT (cond != NULL);
  ASSERT (lock != NULL);
  ASSERT (!intr_context ());
  ASSERT (lock_held_by_current_thread (lock));

  if (!list_empty (&cond->waiters))
  {
      list_sort(&cond->waiters,cond_sema_cmp_priority,NULL);
      sema_up (&list_entry (list_pop_front (&cond->waiters),struct semaphore_elem, elem)->semaphore);
  }
}
```

​	函数说明：通过比较函数维护condition的队列为优先级队列。

- 实现cond_sema_cmp_priority比较函数：


```c
bool cond_sema_cmp_priority (const struct list_elem *a,const struct list_elem *b, void *aux UNUSED)
{
    struct semaphore_elem *sa=list_entry(a,struct semaphore_elem,elem);
    struct semaphore_elem *sb=list_entry(b,struct semaphore_elem,elem);
    return list_entry(list_front(&sa->semaphore.waiters),struct thread,elem)->priority
        >list_entry(list_front(&sb->semaphore.waiters),struct thread,elem)->priority;
}
```

​	函数说明：维护condition的队列为优先级队列的比较函数。

- 把信号量的等待队列实现为优先级队列，修改sema_up和sema_down函数：


```c
void
sema_up (struct semaphore *sema) 
{
  enum intr_level old_level;

  ASSERT (sema != NULL);

  old_level = intr_disable ();
  if (!list_empty (&sema->waiters))
  {
      list_sort(&sema->waiters,thread_cmp_priority,NULL);
      thread_unblock (list_entry (list_pop_front (&sema->waiters),struct thread, elem));
  }
  sema->value++;
  thread_yield();
  
  intr_set_level (old_level);
}
```

```c
void
sema_down (struct semaphore *sema) 
{
  enum intr_level old_level;

  ASSERT (sema != NULL);
  ASSERT (!intr_context ());

  old_level = intr_disable ();
  while (sema->value == 0) 
    {
      list_insert_ordered (&sema->waiters, &thread_current ()->elem,thread_cmp_priority,NULL);
      thread_block ();
    }
  sema->value--;
  intr_set_level (old_level);
}
```

​	函数说明：维护信号量的等待队列为优先级队列。

#### 2.3	实现多级反馈调度

##### 2.3.1	实现浮点运算逻辑

```c
#ifndef __THREAD_FIXED_POINT_H
#define __THREAD_FIXED_POINT_H

//定点的定义
typedef int fixed_t;

//16位用于小数部分
#define FP_SHIFT_AMOUNT 16

//将值转化为定点值
#define FP_CONST(A)((fixed_t)(A<<FP_SHIFT_AMOUNT))

//两个定点数相加
#define FP_ADD(A,B)(A+B)

//定点数A和整数B相加.
#define FP_ADD_MIX(A,B) (A + (B << FP_SHIFT_AMOUNT))

//定义两个数的减法
#define FP_SUB(A,B) (A - B)

//定点数A减去一个整型B
#define FP_SUB_MIX(A,B) (A - (B << FP_SHIFT_AMOUNT))

//定点数A乘整型B
#define FP_MULT_MIX(A,B) (A * B)

//定点数A除以整型B
#define FP_DIV_MIX(A,B) (A / B)

//两个定点数相乘
#define FP_MULT(A,B) ((fixed_t)(((int64_t) A) * B >> FP_SHIFT_AMOUNT))

//两个定点数相除
#define FP_DIV(A,B) ((fixed_t)((((int64_t) A) << FP_SHIFT_AMOUNT) / B))

//取定点数的整数部分
#define FP_INT_PART(A) (A >> FP_SHIFT_AMOUNT)

//定点数四舍五入
#define FP_ROUND(A) (A >= 0 ? ((A + (1 << (FP_SHIFT_AMOUNT - 1))) >> FP_SHIFT_AMOUNT) \
        : ((A - (1 << (FP_SHIFT_AMOUNT - 1))) >> FP_SHIFT_AMOUNT))

#endif /* thread/fixed_point.h */
```

​	函数说明：实现浮点运算的逻辑，包括以上运算逻辑，并在代码上已经加上了注释。

##### 2.3.2	具体实现

###### （1）数据结构

在thread结构体加入以下成员：

```c
int nice;                           
fixed_t recent_cpu;                 //用整数模拟的浮点数
```

###### （2）算法

- 首先需要在线程初始化的时候初始化这两个新的成员， 在init_thread中加入下列代码：


```c
t->nice = 0;
t->recent_cpu = FP_CONST (0);
```

- 要实现多级反馈调度，需要在timer_interrupt中固定一段时间计算更新线程的优先级。修改timer_interrupt函数如下所示：


```c
static void
timer_interrupt (struct intr_frame *args UNUSED)
{
  ticks++;
  if (thread_mlfqs)
  {
      thread_mlfqs_increase_recent_cpu_by_one();
      if (ticks%TIMER_FREQ==0)
      {
          thread_mlfqs_update_load_avg_and_recent_cpu();
      }
      else if (ticks%4==0)
      {
          thread_mlfqs_update_priority(thread_current());
      }
  }
  thread_foreach(blocked_thread_check,NULL);
  thread_tick ();
}
```

​	函数说明：每TIMER_FREQ时间更新一次系统load_avg和所有线程的recent_cpu， 每4个timer_ticks更新一次线程优先级， 每个timer_tick running线程的recent_cpu加一。

- 实现thread_mlfqs_increase_recent_cpu_by_one 函数：

```c
void thread_mlfqs_increase_recent_cpu_by_one(void)
{
    ASSERT(thread_mlfqs);
    ASSERT(intr_context());

    struct thread *current_thread=thread_current();
    if(current_thread==idle_thread)
    {
        return;
    }
    current_thread->recent_cpu=FP_ADD_MIX(current_thread->recent_cpu,1);
}
```

​	函数说明：更新当前线程的recent_cpu，使其增加1。

- 实现thread_mlfqs_update_load_avg_and_recent_cpu函数：

```c
void thread_mlfqs_update_load_avg_and_recent_cpu (void)
{
    ASSERT (thread_mlfqs);
    ASSERT (intr_context ());

    size_t ready_threads = list_size (&ready_list);
    if (thread_current () != idle_thread)
        ready_threads++;
    load_avg = FP_ADD (FP_DIV_MIX (FP_MULT_MIX (load_avg, 59), 60), FP_DIV_MIX (FP_CONST (ready_threads), 60));

    struct thread *t;
    struct list_elem *e = list_begin (&all_list);
    for (; e != list_end (&all_list); e = list_next (e))
    {
        t = list_entry(e, struct thread, allelem);
        if (t != idle_thread)
        {
            t->recent_cpu = FP_ADD_MIX (FP_MULT (FP_DIV (FP_MULT_MIX (load_avg, 2),
                                     FP_ADD_MIX (FP_MULT_MIX (load_avg, 2), 1)), t->recent_cpu), t->nice);
            thread_mlfqs_update_priority (t);
        }
    }
}
```

​	函数说明：每秒都要刷新所有线程的load_avg和recent_cpu。

- 实现thread_mlfqs_update_priority函数：

```c
void thread_mlfqs_update_priority (struct thread *t)
{
    if (t == idle_thread)
        return;

    ASSERT (thread_mlfqs);
    ASSERT (t != idle_thread);

    t->priority = FP_INT_PART (FP_SUB_MIX (FP_SUB (FP_CONST (PRI_MAX), FP_DIV_MIX (t->recent_cpu, 4)), 2 * t->nice));
    t->priority = t->priority < PRI_MIN ? PRI_MIN : t->priority;
    t->priority = t->priority > PRI_MAX ? PRI_MAX : t->priority;
}
```

​	函数说明：更新线程的优先级。

- 实现系统未实现的函数：

```c
/* Sets the current thread's nice value to NICE. */
void
thread_set_nice (int nice)
{
  thread_current ()->nice = nice;
  thread_mlfqs_update_priority (thread_current ());
  thread_yield ();
}

/* Returns the current thread's nice value. */
int
thread_get_nice (void)
{
  return thread_current ()->nice;
}

/* Returns 100 times the system load average. */
int
thread_get_load_avg (void)
{
  return FP_ROUND (FP_MULT_MIX (load_avg, 100));
}

/* Returns 100 times the current thread's recent_cpu value. */
int
thread_get_recent_cpu (void)
{
  return FP_ROUND (FP_MULT_MIX (thread_current ()->recent_cpu, 100));
}
```

- 在thread.c中加入全局变量：

```c
fixed_t load_avg;
```

- 并在thread_start中初始化：

```c
load_avg = FP_CONST (0);
```

### 三、实验测试结果

![image-20221227153328920](F:\md\typora-images\image-20221227153328920.png)

### 四、实验总结

​	通过此次实验，更加了解了线程调度的相关工作流程，线程的抢占，线程的优先级调度和多级反馈调度，通过动手做实验对课上所学的知识加深了理解，由于是第一次做操作系统有关的大型试验，所以做起来不是那么的一帆风顺，包括对c语言掌握的不够好，对Linux系统的了解不够充分，不过好在在坚持不懈的努力下成功的攻克了难题，成功跑通了所有的测试，还是很有成就感的。希望在未来的努力下可以对操作系统知识掌握的更加充分，为将来工作就业打下坚实的基础。

------

## <center>Pintos用户程序</center>

### 一、实验内容

#### 1.1	参数传递

#### 1.2	进程控制系统调用

#### 1.3	文件操作系统调用

### 二、主要功能

#### 2.1	参数传递

##### 	2.1.1	strtok_r函数

```c
char *
strtok_r (char *s, const char *delimiters, char **save_ptr) 
{
  char *token;
  
  ASSERT (delimiters != NULL);
  ASSERT (save_ptr != NULL);

  /* If S is nonnull, start from it.
     If S is null, start from saved position. */
  if (s == NULL)
    s = *save_ptr;
  ASSERT (s != NULL);

  /* Skip any DELIMITERS at our current position. */
  while (strchr (delimiters, *s) != NULL) 
    {
      /* strchr() will always return nonnull if we're searching
         for a null byte, because every string contains a null
         byte (at the end). */
      if (*s == '\0')
        {
          *save_ptr = s;
          return NULL;
        }

      s++;
    }

  /* Skip any non-DELIMITERS up to the end of the string. */
  token = s;
  while (strchr (delimiters, *s) == NULL)
    s++;
  if (*s != '\0') 
    {
      *s = '\0';
      *save_ptr = s + 1;
    }
  else 
    *save_ptr = s;
  return token;
}
```

​	函数说明：函数的返回值是 排在前面的被分割出的字串,或者为NULL，s是传入的字符串。需要注意的是 ：第一次使用strtok_r之后，要把str置为NULL，delim指向依据分割的字符串,常见的空格“ ” 逗号“,”等。saveptr保存剩下待分割的字符串。

##### 2.1.2	参数传递的实现

###### （1）数据结构

无需复杂的数据结构。

###### （2）算法

参数传递的任务是重写process_execute()以及相关函数，使得传入的filename分割成文件名、参数，并压入栈中。

1. **process_execute函数的实现如下所示：**

```c
tid_t
process_execute (const char *file_name)
{
    char *fn_copy0, *fn_copy1;
    tid_t tid;

    /* Make a copy of FILE_NAME.
       Otherwise strtok_r will modify the const char *file_name. */
    fn_copy0 = palloc_get_page(0);//palloc_get_page(0)动态分配了一个内存页
    if (fn_copy0 == NULL)//分配失败
        return TID_ERROR;

    /* Make a copy of FILE_NAME.
       Otherwise there's a race between the caller and load(). */
    fn_copy1 = palloc_get_page (0);
    if (fn_copy1 == NULL)
    {
        palloc_free_page(fn_copy0);
        return TID_ERROR;
    }
    //把file_name 复制2份，PGSIZE为页大小
    strlcpy (fn_copy0, file_name, PGSIZE);
    strlcpy (fn_copy1, file_name, PGSIZE);


    /* Create a new thread to execute FILE_NAME. */
    char *save_ptr;
    char *cmd = strtok_r(fn_copy0, " ", &save_ptr);

    tid = thread_create(cmd, PRI_DEFAULT, start_process, fn_copy1);
    palloc_free_page(fn_copy0);
    if (tid == TID_ERROR)
    {
        palloc_free_page (fn_copy1);
        return tid;
    }

    /* Sema down the parent process, waiting for child */
    sema_down(&thread_current()->sema);
    if (!thread_current()->success) return TID_ERROR;//can't create new process thread,return error

    return tid;
}
```

​	函数说明：首先，分割file_name以分隔命令和参数。我们将此命令作为新线程的名称，然后将参数传递给函数start_process、load和setup_stack。开始执行的时候要把可执行文件从硬盘用load函数加载到内存里，然后根据用户在命令行输入的参数初始化程序的栈，也就是把参数按照某种方式一个个压进栈里。然后才跳转到这个程序的start处开始运行。

2. **接下来需要完成start_process()函数：压栈过程在函数start_process()和push_argument中完成。**

首先查看一下pintos文档对于用户栈的描述：

![img](F:\md\typora-images\watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0Nhbl9fZXI=,size_16,color_FFFFFF,t_70.png)

​	所以我们可以通过如下步骤来完成压栈的操作：

1. 把命令按空格拆开，搞成一堆字符串（以\0结尾）
2. 从后往前循环，把esp往下压一个argv[i]的长度，然后把argv[i]给复制到那个地方

3. 将esp接着往下压，压到是4的倍数。

4. 把从argv[argc+1]一直到argv[0]的地址一个个写进去，这些地址就是我们刚刚压进去的那些位置

5. 再把4中放argv[0]的地址的那个地址放进去

6. 压进去一个argc

7. 压进去一个0，作为return address
   3. **现在就可以实现start_process和push_argument函数：**

```c
static void
start_process (void *file_name_)
{
    char *file_name = file_name_;
    struct intr_frame if_;
    bool success;

    char *fn_copy=malloc(strlen(file_name)+1);
    strlcpy(fn_copy,file_name,strlen(file_name)+1);

    /* Initialize interrupt frame */
    memset (&if_, 0, sizeof if_);
    if_.gs = if_.fs = if_.es = if_.ds = if_.ss = SEL_UDSEG;
    if_.cs = SEL_UCSEG;
    if_.eflags = FLAG_IF | FLAG_MBS;

    /*load executable. */
    //此处发生改变，需要传入文件名
    char *token, *save_ptr;
    file_name = strtok_r (file_name, " ", &save_ptr);
    success = load (file_name, &if_.eip, &if_.esp);

    if (success)
    {
        //计算参数的数量和参数的规格
        int argc = 0;
        //在测试用例中，参数的数量不能超过50
        int argv[50];
        for (token = strtok_r (fn_copy, " ", &save_ptr); token != NULL; token = strtok_r (NULL, " ", &save_ptr)){
            if_.esp -= (strlen(token)+1);//栈指针向下移动，留出token+'\0'的大小
            memcpy (if_.esp, token, strlen(token)+1);//token+'\0'复制进去
            argv[argc++] = (int) if_.esp;//存储参数的地址
        }
        push_argument (&if_.esp, argc, argv);//将参数的地址压入栈
        //记录父线程成功的exec_status和父线程的信号量
        thread_current ()->parent->success = true;
        sema_up (&thread_current ()->parent->sema);
    }
    //释放file_name
    palloc_free_page (file_name);
    free(fn_copy);
    if (!success)
    {
        thread_current ()->parent->success = false;
        sema_up (&thread_current ()->parent->sema);
        thread_exit ();
    }

    /* Start the user process by simulating a return from an
       interrupt, implemented by intr_exit (in
       threads/intr-stubs.S).  Because intr_exit takes all of its
       arguments on the stack in the form of a `struct intr_frame',
       we just point the stack pointer (%esp) to our stack frame
       and jump to it. */
    asm volatile ("movl %0, %%esp; jmp intr_exit" : : "g" (&if_) : "memory");
    NOT_REACHED ();
}

//将参数压进堆栈
void
push_argument (void **esp, int argc, int argv[]){
    *esp = (int)*esp & 0xfffffffc;
    *esp -= 4;//四位对齐（word-align）下压uint8_t大小
    *(int *) *esp = 0;
    /*下面这个for循环的意义是：按照argc的大小，循环压入argv数组，这也符合argc和argv之间的关系*/
    for (int i = argc - 1; i >= 0; i--)
    {
        *esp -= 4;
        *(int *) *esp = argv[i];
    }
    *esp -= 4;
    *(int *) *esp = (int) *esp + 4;//压入argv[0]的地址
    *esp -= 4;
    *(int *) *esp = argc;
    *esp -= 4;
    *(int *) *esp = 0;
}
```

​	函数说明：process_execute创建线程后，用户程序不会立即执行，而是进入start_process，该函数调用load，为用户程序分配内存。要设置堆栈，我们首先记住参数和命令名，并保存它们的地址以备将来使用，之前存储的argv地址也要确保argv[argc]是空指针。接下来添加argv、argc的地址，最后添加一个返回地址。push_argument函数的作用是在start_process中按argv分割参数。

4. **在thread_exit()中加入打印终止信息**

​	在thread_exit()函数中加入下列代码：

```c
printf ("%s: exit(%d)\n",thread_name(), thread_current()->st_exit);
```

​	说明：打印的名称应该是传递给`process_execute()` 的全名，省略命令行参数。

#### 2.2	进程控制系统调用

##### 2.2.1	数据结构

```c
struct list childs;                 /*创建的所有子线程*/
struct child * thread_child;        /*当前线程的子线程*/
int st_exit;                        /* 退出状态 */
struct semaphore sema;              /* 信号量 */
bool success;                       /* 判断子线程执行是否成功 */
struct thread* parent;              /* 当前进程的父进程*/
```

```c
struct child
{
    tid_t tid;                           /* 线程的tid */
    bool isrun;                          /* 子线程是否运行成功 */
    struct list_elem child_elem;         /* 子线程列表 */
    struct semaphore sema;               /* 控制等待的信号量 */
    int store_exit;                      /* 子线程的退出状态 */
};
```

##### 2.2.2	算法

###### （1）实现get_user函数

```c
/* 在用户虚拟地址 UADDR 读取一个字节。UADDR 必须低于 PHYS_BASE。 如果成功则返回字节值，如果发生段错误则返回 -1 。*/ 
static int 
get_user (const uint8_t *uaddr)
{
  int result;
  asm ("movl $1f, %0; movzbl %1, %0; 1:" : "=&a" (result) : "m" (*uaddr));
  return result;
}
```

​	函数说明：获取用户虚拟地址空间，用来判断用户指针是否有效。

###### （2）实现syscall_init()函数

```c
static void (*syscalls[max_syscall])(struct intr_frame *);

void sys_halt(struct intr_frame* f);
void sys_exit(struct intr_frame* f);
void sys_exec(struct intr_frame* f);

void sys_create(struct intr_frame* f);
void sys_remove(struct intr_frame* f);
void sys_open(struct intr_frame* f);
void sys_wait(struct intr_frame* f);
void sys_filesize(struct intr_frame* f);
void sys_read(struct intr_frame* f);
void sys_write(struct intr_frame* f);
void sys_seek(struct intr_frame* f);
void sys_tell(struct intr_frame* f);
void sys_close(struct intr_frame* f);

void
syscall_init (void) 
{
  intr_register_int (0x30, 3, INTR_ON, syscall_handler, "syscall");

  syscalls[SYS_HALT]=&sys_halt;
  syscalls[SYS_EXIT]=&sys_exit;
  syscalls[SYS_EXEC]=&sys_exec;

  syscalls[SYS_CREATE]=&sys_create;
  syscalls[SYS_REMOVE]=&sys_remove;
  syscalls[SYS_OPEN]=&sys_open;

  syscalls[SYS_WAIT]=&sys_wait;
  syscalls[SYS_FILESIZE]=&sys_filesize;
  syscalls[SYS_READ]=&sys_read;

  syscalls[SYS_WRITE]=&sys_write;
  syscalls[SYS_SEEK]=&sys_seek;
  syscalls[SYS_TELL]=&sys_tell;
  syscalls[SYS_CLOSE]=&sys_close;
}
```

​	函数说明：初始化系统调用，通过syscall数组来存储13个系统调用，在syscall_handler里通过识别数组的序号决定调用哪一个系统调用。

###### （3）修改syscall_handler函数

```c
static void
syscall_handler (struct intr_frame *f UNUSED) 
{
    //在其第一个参数上加上1，并打印结果
    int * p = f->esp;
    check_ptr2 (p + 1);//检验第一个参数
    int type = * (int *)f->esp;//检验系统调用号sys_code是否合法
    if(type <= 0 || type >= max_syscall){
        exit_special ();
    }
    syscalls[type](f);//无误则执行对应系统调用函数
}
```

​	函数说明：系统调用发生时，中断就会自动调用这个函数进行分发处理，调用对应的函数。

###### （4）实现sys_halt函数

```c
void sys_halt (struct intr_frame* f)
{
    shutdown_power_off();
}
```

​	函数说明：通过调用shutdown_power_off()来终止pintos。

###### （5）实现sys_exit函数

```c
void sys_exit (struct intr_frame* f)
{
    uint32_t *user_ptr = f->esp;
    check_ptr2 (user_ptr + 1);//检验第一个参数
    *user_ptr++;//指针指向第一个参数
    //记录进程的退出状态
    thread_current()->st_exit = *user_ptr;//保存exit_code
    thread_exit ();
}
```

​	函数说明：终止当前用户程序，将状态返回给内核。

###### （6）实现sys_exec 函数

```c
void sys_exec (struct intr_frame* f)
{
    uint32_t *user_ptr = f->esp;
    check_ptr2 (user_ptr + 1);//检查第一个参数的地址
    check_ptr2 (*(user_ptr + 1));//检查第一个参数的值，即const char *file指向的地址
    *user_ptr++;
    f->eax = process_execute((char*)* user_ptr);//使用process_execute完成pid的返回
}
```

​	函数说明：首先检查file_name引用的文件是否有效(指向内存地址、页面和页面内容的指针是否有效)。如果无效，返回-1，否则调用函数process_execute。

###### （7）实现check_ptr2函数

```c
void * check_ptr2(const void *vaddr)
{
    //判断地址
    if (!is_user_vaddr(vaddr))//是否为用户地址
    {
        exit_special ();
    }
    //判断页数
    void *ptr = pagedir_get_page (thread_current()->pagedir, vaddr);//是否为用户地址
    if (!ptr)
    {
        exit_special ();
    }
    //判断页面内容
    uint8_t *check_byteptr = (uint8_t *) vaddr;
    for (uint8_t i = 0; i < 4; i++)
    {
        if (get_user(check_byteptr + i) == -1)
        {
            exit_special ();
        }
    }

    return ptr;
}
```

​	函数说明：在该函数中可以完成返回pid等于-1。

###### （8）实现process_wait函数

1. 初始化线程

​	在thread_create 函数中，创建线程时对子线程进行处理，加入下列代码：

```c
init_thread (t, name, priority);
tid = t->tid = allocate_tid ();

t->thread_child = malloc(sizeof(struct child));
t->thread_child->tid = tid;//新线程的thread_child tid初始为新线程的tid
sema_init (&t->thread_child->sema, 0);//新线程的thread_child sema初始化
list_push_back (&thread_current()->childs, &t->thread_child->child_elem);

/* Initialize the  exit status by the MAX Fix Bug */
t->thread_child->store_exit = UINT32_MAX;
t->thread_child->isrun = false;
```

​	在初始化线程函数init_thread中新增对这些参数的初始化，新增代码如下所示：

```c
if (t==initial_thread)
      t->parent=NULL;
      //记录父线程
  else
      t->parent = thread_current ();
  //list的列表初始化
  list_init (&t->childs);
  list_init (&t->files);
  //list的信号量初始化
  sema_init (&t->sema, 0);
  t->success = true;
  //初始化退出状态为MAX
  t->st_exit = UINT32_MAX;
```

2. 实现

```c
int
process_wait (tid_t child_tid UNUSED) 
{
  //找到当前线程等待的子线程的ID，并删除子线程的信号量
  struct list *l = &thread_current()->childs;
  struct list_elem *child_elem_ptr;
  child_elem_ptr = list_begin (l);
  struct child *child_ptr = NULL;
  while (child_elem_ptr != list_end (l))//遍历当前线程的所有子线程
  {
    /* list_entry:将指向列表元素LIST_ELEM的指针转换为指向嵌入LIST_ELEM的结构的指针. 提供外部结构STRUCT的名称和list元素的成员名成员. */
    child_ptr = list_entry (child_elem_ptr, struct child, child_elem);//把child_elem的指针变成child的指针
    if (child_ptr->tid == child_tid)//找到child_tid
    {
        if (!child_ptr->isrun)//检查子线程之前是否已经等待过
        {
            child_ptr->isrun = true;
            sema_down (&child_ptr->sema);//线程阻塞，等待子进程结束
            break;
        }
        else 
        {
            return -1;
        }
    }
    child_elem_ptr = list_next (child_elem_ptr);
  }
  if (child_elem_ptr == list_end (l)) {//找不到child_tid
      return -1;
  }
  //执行到这里说明子进程正常退出
  list_remove (child_elem_ptr);//从子进程列表中删除该子进程，因为它已经没有在运行了，也就是说父进程重新抢占回了资源
  return child_ptr->store_exit;//返回子线程exit值
}
```

​	函数说明：内核终止时、子线程的tid不存在或其不是调用进程的子线程时、子进程成功运行结束时返回-1。

###### （9）修改thread_exit函数

​	进程等待的时候进行了sema_down操作，在子线程退出的时候要进行sema_up操作，新增的代码如下所示：

```c
//保存下来st_exit在process_wait中使用
thread_current ()->thread_child->store_exit = thread_current()->st_exit;
//子线程退出，把资源还给父线程
sema_up (&thread_current()->thread_child->sema);
```

###### （10）实现sys_wait 函数

```c
void sys_wait (struct intr_frame* f)
{
    uint32_t *user_ptr = f->esp;
    check_ptr2 (user_ptr + 1);
    *user_ptr++;
    f->eax = process_wait(*user_ptr);
}
```

​	函数说明：等待子进程pid并检索子进程的退出状态。如果pid没有终止，则等待直到它终止。然后，返回pid传递给exit的状态。如果pid没有调用exit()，而是被内核终止(例如由于异常而被杀死)，wait(pid)必须返回-1。当父进程调用wait时，父进程等待已经终止的子进程是合法的，但是内核仍然必须允许父进程检索其子进程的退出状态，或者了解到子进程已被内核终止。

#### 2.3	文件操作系统调用

##### 2.3.1	数据结构

​	在线程结构体中加入下列成员：

```c
struct list files; //打开的文件列表
int max_file_fd;//最大的文件描述符
```

​	文件结构体：

```c
/* File that the thread open */
struct thread_file
{
    int fd;//文件描述符
    struct file* file;
    struct list_elem file_elem;//文件列表
};
```

##### 2.3.2	算法

###### （1）acquire_lock_f 和release_lock_f 函数的实现

```c
static struct lock lock_f;
void acquire_lock_f()
{
    lock_acquire(&lock_f);
}

void release_lock_f()
{
    lock_release(&lock_f);
}
```

​	函数说明：在进行文件读写的时候，只允许一个线程进行操作，所以需要建立一个锁的机制。

###### （2）修改thread_exit函数

```c
 //关闭所有文件
struct list_elem *e;
struct list *files = &thread_current()->files;
while(!list_empty (files))
{
    e = list_pop_front (files);
    struct thread_file *f = list_entry (e, struct thread_file, file_elem);
    acquire_lock_f ();
    file_close (f->file);
    release_lock_f ();
    //删除列表中的文件
    list_remove (e);
    //释放文件获取的资源
    free (f);
}
```

​	函数说明：在线程退出的时候，把当前线程拥有的所有文件都释放掉。

###### （3）实现sys_write 函数

```c
void sys_write (struct intr_frame* f)
{
    uint32_t *user_ptr = f->esp;
    check_ptr2 (user_ptr + 7);
    check_ptr2 (*(user_ptr + 6));
    *user_ptr++;
    int fd = *user_ptr;
    const char * buffer = (const char *)*(user_ptr+1);
    off_t size = *(user_ptr+2);
    if (fd == 1) {//写入控制台
        //使用putbuf进行测试
        putbuf(buffer,size);
        f->eax = size;
    }
    else
    {
        //写入文件
        struct thread_file * thread_file_temp = find_file_id (*user_ptr);
        if (thread_file_temp)
        {
            acquire_lock_f ();//文件操作需要锁
            f->eax = file_write (thread_file_temp->file, buffer, size);
            release_lock_f ();
        }
        else
        {
            f->eax = 0;//返回0
        }
    }
}

//根据文件的ID查找文件
struct thread_file *find_file_id (int file_id)
{
    struct list_elem *e;
    struct thread_file * thread_file_temp = NULL;
    struct list *files = &thread_current ()->files;
    for (e = list_begin (files); e != list_end (files); e = list_next (e)){
        thread_file_temp = list_entry (e, struct thread_file, file_elem);
        if (file_id == thread_file_temp->fd)
            return thread_file_temp;
    }
    return false;
}
```

​	函数说明：首先我们需要判断是否要写入控制台或文件。如果它将被写入控制台，我们使用函数putbuf（const char \*buffer，size_t n）来完成。如果它将写入文件，首先我们通过当前线程中的id find_file_id（int file_id）查找文件，然后通过file_write（struct file\*file，const void*buffer）执行写操作。

###### （4）实现sys_create函数

```c
//获取文件操作的锁
void sys_create(struct intr_frame* f)
{
    uint32_t *user_ptr = f->esp;
    check_ptr2 (user_ptr + 5);
    check_ptr2 (*(user_ptr + 4));
    *user_ptr++;
    acquire_lock_f ();
    f->eax = filesys_create ((const char *)*user_ptr, *(user_ptr+1));
    release_lock_f ();
}
```

​	函数说明：调用filesys_create 函数，通过acquire_lock_f()获取锁，通过release_lock_f()完成释放锁，创建一个新的文件，成功返回true，失败返回false。

###### （5）实现sys_remove函数

```c
void sys_remove(struct intr_frame* f)
{
    uint32_t *user_ptr = f->esp;
    check_ptr2 (user_ptr + 1);//arg address
    check_ptr2 (*(user_ptr + 1));//file address
    *user_ptr++;
    acquire_lock_f ();
    f->eax = filesys_remove ((const char *)*user_ptr);
    release_lock_f ();
}
```

​	函数说明：调用filesys_remove()函数删除文件。

###### （6）实现sys_open 函数

```c
//打开文件
void sys_open (struct intr_frame* f)
{
    uint32_t *user_ptr = f->esp;
    check_ptr2 (user_ptr + 1);
    check_ptr2 (*(user_ptr + 1));
    *user_ptr++;
    acquire_lock_f ();
    struct file * file_opened = filesys_open((const char *)*user_ptr);
    release_lock_f ();
    struct thread * t = thread_current();
    if (file_opened)
    {
        struct thread_file *thread_file_temp = malloc(sizeof(struct thread_file));
        thread_file_temp->fd = t->max_file_fd++;
        thread_file_temp->file = file_opened;
        list_push_back (&t->files, &thread_file_temp->file_elem);//维护files列表
        f->eax = thread_file_temp->fd;
    }
    else// 文件打不开
    {
        f->eax = -1;
    }
}
```

​	函数说明：通过函数filesys_open (const char *name)打开文件。然后，将具有thread_file结构的文件推到线程打开的文件的列表文件中。当打开文件时返回一个不同的fd值，每个进程都有自己独立的fd集合，即使是同一个文件的不同打开也是不同的fd。

###### （7）实现sys_filesize 函数

```c
void sys_filesize (struct intr_frame* f){
    uint32_t *user_ptr = f->esp;
    check_ptr2 (user_ptr + 1);
    *user_ptr++;//fd
    struct thread_file * thread_file_temp = find_file_id (*user_ptr);
    if (thread_file_temp)
    {
        acquire_lock_f ();
        f->eax = file_length (thread_file_temp->file);//返回字节大小
        release_lock_f ();
    }
    else
    {
        f->eax = -1;
    }
}
```

​	函数说明：首先从find_file_id(int file_id)中找到文件。然后调用函数file_length (struct file *file)执行syscall filesize。

###### （8）实现sys_read 函数

```c
//检查用户指针是否有效
bool is_valid_pointer (void* esp,uint8_t argc){
    for (uint8_t i = 0; i < argc; ++i)
    {
        if((!is_user_vaddr (esp)) ||
           (pagedir_get_page (thread_current()->pagedir, esp)==NULL)){
            return false;
        }
    }
    return true;
}

void sys_read (struct intr_frame* f)
{
    uint32_t *user_ptr = f->esp;

    *user_ptr++;

    int fd = *user_ptr;
    uint8_t * buffer = (uint8_t*)*(user_ptr+1);
    off_t size = *(user_ptr+2);
    if (!is_valid_pointer (buffer, 1) || !is_valid_pointer (buffer + size,1)){
        exit_special ();
    }
    //获取文件缓冲区
    if (fd == 0) //stdin
    {
        for (int i = 0; i < size; i++)
            buffer[i] = input_getc();
        f->eax = size;
    }
    else
    {
        struct thread_file * thread_file_temp = find_file_id (*user_ptr);
        if (thread_file_temp)
        {
            acquire_lock_f ();
            f->eax = file_read (thread_file_temp->file, buffer, size);
            release_lock_f ();
        }
        else//can't read
        {
            f->eax = -1;
        }
    }
}
```

​	函数说明：将size字节大小的文件从fd读入缓冲区。返回实际读到的大小(文件结束时为0)，如果无法读取文件则返回-1。fd为0时使用input_getc()从键盘读取。

###### （9）实现sys_seek函数

```c
void sys_seek(struct intr_frame* f)
{
    uint32_t *user_ptr = f->esp;
    check_ptr2 (user_ptr + 5);
    *user_ptr++;
    struct thread_file *file_temp = find_file_id (*user_ptr);
    if (file_temp)
    {
        acquire_lock_f ();
        file_seek (file_temp->file, *(user_ptr+1));
        release_lock_f ();
    }
}
```

​	函数说明：通过调用文件系统中的file_seek（）函数来执行系统查找。

###### （10）实现sys_tell 函数

```c
void sys_tell (struct intr_frame* f)
{
    uint32_t *user_ptr = f->esp;
    check_ptr2 (user_ptr + 1);
    *user_ptr++;
    struct thread_file *thread_file_temp = find_file_id (*user_ptr);
    if (thread_file_temp)
    {
        acquire_lock_f ();
        f->eax = file_tell (thread_file_temp->file);
        release_lock_f ();
    }else{
        f->eax = -1;
    }
}
```

​	函数说明：返回下一个读写的字节在文件中的位置。

###### （11）实现sys_close 函数

```c
void sys_close (struct intr_frame* f)
{
    uint32_t *user_ptr = f->esp;
    check_ptr2 (user_ptr + 1);
    *user_ptr++;
    struct thread_file * opened_file = find_file_id (*user_ptr);
    if (opened_file)
    {
        acquire_lock_f ();
        file_close (opened_file->file);
        release_lock_f ();
        //从列表中删除打开的文件
        list_remove (&opened_file->file_elem);
        //释放打开的文件
        free (opened_file);
    }
}
```

​	函数说明：关闭文件，调用file_close,再把其从file列表里删除。

###### （12）修改load 函数

因为process_execute也会访问文件，所以只要进程在运行，就需要保持exec对应的文件处于打开状态，因此在load过程中，在对文件操作前需要给文件操作加锁，在操作完成后再释放锁，同时把exec()中的文件加入到file队列里，在thread_exit中自动删除，修改后的代码如下所示。

```c
bool
load (const char *file_name, void (**eip) (void), void **esp) 
{
···
  //给文件加锁
  acquire_lock_f();

  /* Open executable file. */
  file = filesys_open (file_name);

  struct thread_file *thread_file_temp = malloc(sizeof(struct thread_file));
  thread_file_temp->file = file;
  list_push_back (&thread_current()->files, &thread_file_temp->file_elem);
  file_deny_write(file);

 ···
  release_lock_f();
  return success;
}
```

### 三、实验测试结果

![image-20221227220834244](F:\md\typora-images\image-20221227220834244.png)

### 四、实验总结

​	通过此次实验，更加了解了用户程序调用的相关工作流程，进程控制系统调用和文件操作系统的调用，在操作文件之前一定要申请锁，在操作完成之后释放锁，加深了对互斥同步信号量机制的理解，由于是第一次做操作系统有关的大型试验，所以做起来不是那么的一帆风顺，包括对c语言掌握的不够好，对Linux系统的了解不够充分，不过好在在坚持不懈的努力下成功的攻克了难题，成功跑通了所有的测试，还是很有成就感的。希望在未来的努力下可以对操作系统知识掌握的更加充分，为将来工作就业打下坚实的基础。





