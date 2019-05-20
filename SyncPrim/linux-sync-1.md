리눅스 커널의 동기화 기본기능. Part 1.
================================================================================

소개
--------------------------------------------------------------------------------

이 파트는 [linux-insides](https://0xax.gitbooks.io/linux-insides/content/) 책의
새로운 챕터를 시작합니다. 앞의
[chapter](https://0xax.gitbooks.io/linux-insides/content/Timers/index.html)
에서는 타이머와 시간 관리에 대한 내용을 다뤘습니다. 이제 다음으로 넘어갑시다.
이 파트의 제목에서 이미 이해하셨겠지만, 이 챕터는 리눅스 커널의
[동기화](https://en.wikipedia.org/wiki/Synchronization_%28computer_science%29)
기본기능들을 설명합니다.

항상 그랬듯, 동기화에 관련된 뭔가를 고려하기 전에, `동기화 기본기능` 이
일반적으로 무엇인가에 대해 알아보겠습니다. 사실, 동기화 기본기능은 두개 이상의
[병렬](https://en.wikipedia.org/wiki/Parallel_computing) 프로세스나 쓰레드가
특정 코드 영역을 동시에 수행하지 못하게 하는 소프트웨어 메커니즘입니다. 예를
들어,
[kernel/time/clocksource.c](https://github.com/torvalds/linux/master/kernel/time/clocksource.c)
파일의 다음 코드를 봅시다:

```C
mutex_lock(&clocksource_mutex);
...
...
...
clocksource_enqueue(cs);
clocksource_enqueue_watchdog(cs);
clocksource_select();
...
...
...
mutex_unlock(&clocksource_mutex);
```

이 코드는 특정
[clocksource](https://0xax.gitbooks.io/linux-insides/content/Timers/linux-timers-2.html)
를 clock source 리스트에 추가하는 `__clocksource_register_scale` 함수에서
가져온 겁니다. 이 함수는 등록된 clock source 를 가지고 있는 리스트에 여러
연산을 수행합니다. 예를 들어, `clocksource_enqueue` 함수는 주어진 clock source
를 등록된 clocksource를 가지고 있는 리스트 - `clocksource_list` 에 추가합니다.
이 코드는 두 함수로 싸여져 있음을 알아두시기 바랍니다: 하나의 패러미터 (여기선
`clocksource_mutex`) 를 받는 `mutex_lock` 과 `mutex_unlock` 입니다.

이 함수들은 [mutex](https://en.wikipedia.org/wiki/Mutual_exclusion) 동기화
기본기능에 기반한 locking 과 unlocking 을 나타냅니다.  `mutex_lock` 이
수행되면, 이 함수는 우리가 두개 이상의 쓰레드가 이 mutex 소유자가
`mutex_unlock` 을 수행하기 전까지는 이 코드를 동시에 수행하는 걸 막을 수 있게
해줍니다. 달리 말하면, 우리는 `clocksource_list` 의 병렬 연산을 방지합니다.
여기서 `mutex` 가 필요한 이유가 뭘까요? 두개의 병렬 프로세스가 하나의 clock
source 를 등록하려 하면 어떻게 될까요. 우리가 이미 알고 있듯,
`clocksource_enqueue` 함수는 주어진 clock source 를 `clocksource_list` 리스트에
가장 큰 rating 을 갖는 clock source (시스템에서 가장 높은 frequency 를 갖는
등록된 clock source) 바로 뒤에 추가시킵니다:

```C
static void clocksource_enqueue(struct clocksource *cs)
{
	struct list_head *entry = &clocksource_list;
	struct clocksource *tmp;

	list_for_each_entry(tmp, &clocksource_list, list) {
		if (tmp->rating < cs->rating)
			break;
		entry = &tmp->list;
	}
	list_add(&cs->list, entry);
}
```

만약 두개의 병렬 프로세스가 이걸 동시에 수행하면, 두 프로세스 모두 같은 `entry`
를 보게 되어 [race condition](https://en.wikipedia.org/wiki/Race_condition) 을
일으킬 수 있는데 이를 달리 말하면, 두번째 프로세스가 `list_add` 를 수행함으로써
첫번째 쓰레드의 clock source 를 덮어쓰게 될겁니다.

이 간단한 예제 외에, 동기화 기본기능은 리눅스 커널의 모든 곳에 있습니다. 앞의
[chapter](https://0xax.gitbooks.io/linux-insides/content/Timers/index.html)
또는 다른 챕터를 다시 보거나 일반적인 리눅스 커널 솟 크도르르 보게 되면 이런
것들을 많이 볼 수 있을 겁니다. 우린 리눅스 커널에서 `mutex` 가 어떻게 구현되어
있는지는 고려하지 않겠습니다. 사실, 리눅스 커널은 다양한 동기화 기본기능들을
제공합니다:

* `mutex`;
* `semaphore`;
* `seqlock`;
* `atomic operation`;
* 기타 등등.

우린 이 챕터를 `spinlock` 으로 시작하겠습니다.

Spinlocks in the Linux kernel.
--------------------------------------------------------------------------------

The `spinlock` is a low-level synchronization mechanism which in simple words, represents a variable which can be in two states:

* `acquired`;
* `released`.

Each process which wants to acquire a `spinlock`, must write a value which represents `spinlock acquired` state to this variable and write `spinlock released` state to the variable. If a process tries to execute code which is protected by a `spinlock`, it will be locked while a process which holds this lock will release it. In this case all related operations must be [atomic](https://en.wikipedia.org/wiki/Linearizability) to prevent [race conditions](https://en.wikipedia.org/wiki/Race_condition) state. The `spinlock` is represented by the `spinlock_t` type in the Linux kernel. If we will look at the Linux kernel code, we will see that this type is [widely](http://lxr.free-electrons.com/ident?i=spinlock_t) used. The `spinlock_t` is defined as:

```C
typedef struct spinlock {
        union {
              struct raw_spinlock rlock;
 
#ifdef CONFIG_DEBUG_LOCK_ALLOC
# define LOCK_PADSIZE (offsetof(struct raw_spinlock, dep_map))
                struct {
                        u8 __padding[LOCK_PADSIZE];
                        struct lockdep_map dep_map;
                };
#endif
        };
} spinlock_t;
```

and located in the [include/linux/spinlock_types.h](https://github.com/torvalds/linux/master/include/linux/spinlock_types.h) header file. We may see that its implementation depends on the state of the `CONFIG_DEBUG_LOCK_ALLOC` kernel configuration option. We will skip this now, because all debugging related stuff will be in the end of this part. So, if the `CONFIG_DEBUG_LOCK_ALLOC` kernel configuration option is disabled, the `spinlock_t` contains [union](https://en.wikipedia.org/wiki/Union_type#C.2FC.2B.2B) with one field which is - `raw_spinlock`:

```C
typedef struct spinlock {
        union {
              struct raw_spinlock rlock;
        };
} spinlock_t;
```

The `raw_spinlock` structure defined in the [same](https://github.com/torvalds/linux/master/include/linux/spinlock_types.h) header file represents the implementation of `normal` spinlock. Let's look how the `raw_spinlock` structure is defined:

```C
typedef struct raw_spinlock {
        arch_spinlock_t raw_lock;
#ifdef CONFIG_DEBUG_SPINLOCK
	unsigned int magic, owner_cpu;
	void *owner;
#endif
#ifdef CONFIG_DEBUG_LOCK_ALLOC
	struct lockdep_map dep_map;
#endif
} raw_spinlock_t;
```

where the `arch_spinlock_t` represents architecture-specific `spinlock` implementation. As we mentioned above, we will skip debugging kernel configuration options. As we focus on [x86_64](https://en.wikipedia.org/wiki/X86-64) architecture in this book, the `arch_spinlock_t` that we will consider is defined in the [include/asm-generic/qspinlock_types.h](https://github.com/torvalds/linux/master/include/asm-generic/qspinlock_types.h) header file and looks:

```C
typedef struct qspinlock {
        union {
		atomic_t val;
		struct {
			u8	locked;
			u8	pending;
		};
		struct {
			u16	locked_pending;
			u16	tail;
		};
        };
} arch_spinlock_t;
```

We will not stop on this structures for now. Let's look at the operations on a spinlock. The Linux kernel provides following main operations on a `spinlock`:

* `spin_lock_init` - produces initialization of the given `spinlock`;
* `spin_lock` - acquires given `spinlock`;
* `spin_lock_bh` - disables software [interrupts](https://en.wikipedia.org/wiki/Interrupt) and acquire given `spinlock`.
* `spin_lock_irqsave` and `spin_lock_irq` - disable interrupts on local processor and preserve/not preserve previous interrupt state in the `flags`;
* `spin_unlock` - releases given `spinlock`;
* `spin_unlock_bh` - releases given `spinlock` and enables software interrupts;
* `spin_is_locked` - returns the state of the given `spinlock`;
* and etc.

Let's look on the implementation of the `spin_lock_init` macro. As I already wrote, this and other macro are defined in the [include/linux/spinlock.h](https://github.com/torvalds/linux/master/include/linux/spinlock.h) header file and the `spin_lock_init` macro looks:

```C
#define spin_lock_init(_lock)			\
do {						\
	spinlock_check(_lock);		        \
	raw_spin_lock_init(&(_lock)->rlock);	\
} while (0)
```

As we may see, the `spin_lock_init` macro takes a `spinlock` and executes two operations: check the given `spinlock` and execute the `raw_spin_lock_init`. The implementation of the `spinlock_check` is pretty easy, this function just returns the `raw_spinlock_t` of the given `spinlock` to be sure that we got exactly `normal` raw spinlock:

```C
static __always_inline raw_spinlock_t *spinlock_check(spinlock_t *lock)
{
	return &lock->rlock;
}
```

The `raw_spin_lock_init` macro:

```C
# define raw_spin_lock_init(lock)		\
do {						\
    *(lock) = __RAW_SPIN_LOCK_UNLOCKED(lock);	\
} while (0)					\
```

assigns the value of the `__RAW_SPIN_LOCK_UNLOCKED` with the given `spinlock` to the given `raw_spinlock_t`. As we may understand from the name of the `__RAW_SPIN_LOCK_UNLOCKED` macro, this macro does initialization of the given `spinlock` and set it to `released` state. This macro is defined in the [include/linux/spinlock_types.h](https://github.com/torvalds/linux/master/include/linux/spinlock_types.h) header file and expands to the following macros:

```C
#define __RAW_SPIN_LOCK_UNLOCKED(lockname)      \
         (raw_spinlock_t) __RAW_SPIN_LOCK_INITIALIZER(lockname)

#define __RAW_SPIN_LOCK_INITIALIZER(lockname)			\
         {                                                      \
             .raw_lock = __ARCH_SPIN_LOCK_UNLOCKED,             \
             SPIN_DEBUG_INIT(lockname)                          \
             SPIN_DEP_MAP_INIT(lockname)                        \
         }
```

As I already wrote above, we will not consider stuff which is related to debugging of synchronization primitives. In this case we will not consider the `SPIN_DEBUG_INIT` and the `SPIN_DEP_MAP_INIT` macros. So the `__RAW_SPINLOCK_UNLOCKED` macro will be expanded to the:

```C
*(&(_lock)->rlock) = __ARCH_SPIN_LOCK_UNLOCKED;
```

where the `__ARCH_SPIN_LOCK_UNLOCKED` is:

```C
#define __ARCH_SPIN_LOCK_UNLOCKED       { { .val = ATOMIC_INIT(0) } }
```

for the [x86_64](https://en.wikipedia.org/wiki/X86-64) architecture. So, after the expansion of the `spin_lock_init` macro, a given `spinlock` will be initialized and its state will be - `unlocked`.

From this moment we know how to initialize a `spinlock`, now let's consider [API](https://en.wikipedia.org/wiki/Application_programming_interface) which Linux kernel provides for manipulations of `spinlocks`. The first is:

```C
static __always_inline void spin_lock(spinlock_t *lock)
{
	raw_spin_lock(&lock->rlock);
}
```

function which allows us to `acquire` a spinlock. The `raw_spin_lock` macro is defined in the same header file and expnads to the call of `_raw_spin_lock`:

```C
#define raw_spin_lock(lock)	_raw_spin_lock(lock)
```

Where `_raw_spin_lock` is defined depends on whether `CONFIG_SMP` option is set and `CONFIG_INLINE_SPIN_LOCK` option is set. If the [SMP](https://en.wikipedia.org/wiki/Symmetric_multiprocessing) is disabled, `_raw_spin_lock` is defined in the [include/linux/spinlock_api_up.h](https://github.com/torvalds/linux/blob/master/include/linux/spinlock_api_up.h) header file as a macro and looks like:

```C
#define _raw_spin_lock(lock)	__LOCK(lock)
```

If the SMP is enabled and `CONFIG_INLINE_SPIN_LOCK` is set, it is defined in [include/linux/spinlock_api_smp.h](https://github.com/torvalds/linux/blob/master/include/linux/spinlock_api_smp.h) header file as the following:

```C
#define _raw_spin_lock(lock) __raw_spin_lock(lock)
```

If the SMP is enabled and `CONFIG_INLINE_SPIN_LOCK` is not set, it is defined in [kernel/locking/spinlock.c](https://github.com/torvalds/linux/blob/master/kernel/locking/spinlock.c) source code file as the following:

```C
void __lockfunc _raw_spin_lock(raw_spinlock_t *lock)
{
	__raw_spin_lock(lock);
}
```

Here we will consider the latter form of `_raw_spin_lock`. The `__raw_spin_lock` function looks:

```C
static inline void __raw_spin_lock(raw_spinlock_t *lock)
{
        preempt_disable();
        spin_acquire(&lock->dep_map, 0, 0, _RET_IP_);
        LOCK_CONTENDED(lock, do_raw_spin_trylock, do_raw_spin_lock);
}
```

As you may see, first of all we disable [preemption](https://en.wikipedia.org/wiki/Preemption_%28computing%29) by the call of the `preempt_disable` macro from the [include/linux/preempt.h](https://github.com/torvalds/linux/blob/master/include/linux/preempt.h) (more about this you may read in the ninth [part](https://0xax.gitbooks.io/linux-insides/content/Initialization/linux-initialization-9.html) of the Linux kernel initialization process chapter). When we unlock the given `spinlock`, preemption will be enabled again:

```C
static inline void __raw_spin_unlock(raw_spinlock_t *lock)
{
        ...
        ...
        ...
        preempt_enable();
}
```

We need to do this to prevent the process from other processes to preempt it while it is spinning on a lock. The `spin_acquire` macro which through a chain of other macros expands to the call of the:

```C
#define spin_acquire(l, s, t, i)                lock_acquire_exclusive(l, s, t, NULL, i)
#define lock_acquire_exclusive(l, s, t, n, i)           lock_acquire(l, s, t, 0, 1, n, i)
```

The `lock_acquire` function:

```C
void lock_acquire(struct lockdep_map *lock, unsigned int subclass,
                  int trylock, int read, int check,
                  struct lockdep_map *nest_lock, unsigned long ip)
{
         unsigned long flags;

         if (unlikely(current->lockdep_recursion))
                return;
 
         raw_local_irq_save(flags);
         check_flags(flags);
 
         current->lockdep_recursion = 1;
         trace_lock_acquire(lock, subclass, trylock, read, check, nest_lock, ip);
         __lock_acquire(lock, subclass, trylock, read, check,
                        irqs_disabled_flags(flags), nest_lock, ip, 0, 0);
         current->lockdep_recursion = 0;
         raw_local_irq_restore(flags);
}
```

As I wrote above, we will not consider stuff here which is related to debugging or tracing. The main point of the `lock_acquire` function is to disable hardware interrupts by the call of the `raw_local_irq_save` macro, because the given spinlock might be acquired with enabled hardware interrupts. In this way the process will not be preempted. Note that in the end of the `lock_acquire` function we will enable hardware interrupts again with the help of the `raw_local_irq_restore` macro. As you already may guess, the main work will be in the `__lock_acquire` function which is defined in the [kernel/locking/lockdep.c](https://github.com/torvalds/linux/blob/master/kernel/locking/lockdep.c) source code file.

The `__lock_acquire` function looks big. We will try to understand what this function does, but not in this part. Actually this function is mostly related to the Linux kernel [lock validator](https://www.kernel.org/doc/Documentation/locking/lockdep-design.txt) and it is not topic of this part. If we will return to the definition of the `__raw_spin_lock` function, we will see that it contains the following definition in the end:

```C
LOCK_CONTENDED(lock, do_raw_spin_trylock, do_raw_spin_lock);
```

The `LOCK_CONTENDED` macro is defined in the [include/linux/lockdep.h](https://github.com/torvalds/linux/blob/master/include/linux/lockdep.h) header file and just calls the given function with the given `spinlock`:

```C
#define LOCK_CONTENDED(_lock, try, lock) \
         lock(_lock)
```

In our case, the `lock` is `do_raw_spin_lock` function from the [include/linux/spinlock.h](https://github.com/torvalds/linux/blob/master/include/linux/spnlock.h) header file and the `_lock` is the given `raw_spinlock_t`:

```C
static inline void do_raw_spin_lock(raw_spinlock_t *lock) __acquires(lock)
{
        __acquire(lock);
         arch_spin_lock(&lock->raw_lock);
}
```

The `__acquire` here is just [Sparse](https://en.wikipedia.org/wiki/Sparse) related macro and we are not interested in it in this moment. The `arch_spin_lock` macro is defined in the [include/asm-generic/qspinlock.h](https://github.com/torvalds/linux/blob/master/include/asm-generic/qspinlocks.h) header file as the following:

```C
#define arch_spin_lock(l)               queued_spin_lock(l)
```

We stop here for this part. In the next part, we'll dive into how queued spinlocks works and related concepts.

Conclusion
--------------------------------------------------------------------------------

This concludes the first part covering synchronization primitives in the Linux kernel. In this part, we met first synchronization primitive `spinlock` provided by the Linux kernel. In the next part we will continue to dive into this interesting theme and will see other `synchronization` related stuff.

If you have questions or suggestions, feel free to ping me in twitter [0xAX](https://twitter.com/0xAX), drop me [email](anotherworldofworld@gmail.com) or just create [issue](https://github.com/0xAX/linux-insides/issues/new).

**Please note that English is not my first language and I am really sorry for any inconvenience. If you found any mistakes please send me PR to [linux-insides](https://github.com/0xAX/linux-insides).**

Links
--------------------------------------------------------------------------------

* [Concurrent computing](https://en.wikipedia.org/wiki/Concurrent_computing)
* [Synchronization](https://en.wikipedia.org/wiki/Synchronization_%28computer_science%29)
* [Clocksource framework](https://0xax.gitbooks.io/linux-insides/content/Timers/linux-timers-2.html)
* [Mutex](https://en.wikipedia.org/wiki/Mutual_exclusion)
* [Race condition](https://en.wikipedia.org/wiki/Race_condition)
* [Atomic operations](https://en.wikipedia.org/wiki/Linearizability)
* [SMP](https://en.wikipedia.org/wiki/Symmetric_multiprocessing)
* [x86_64](https://en.wikipedia.org/wiki/X86-64) 
* [Interrupts](https://en.wikipedia.org/wiki/Interrupt)
* [Preemption](https://en.wikipedia.org/wiki/Preemption_%28computing%29) 
* [Linux kernel lock validator](https://www.kernel.org/doc/Documentation/locking/lockdep-design.txt)
* [Sparse](https://en.wikipedia.org/wiki/Sparse)
* [xadd instruction](http://x86.renejeschke.de/html/file_module_x86_id_327.html)
* [NOP](https://en.wikipedia.org/wiki/NOP)
* [Memory barriers](https://www.kernel.org/doc/Documentation/memory-barriers.txt)
* [Previous chapter](https://0xax.gitbooks.io/linux-insides/content/Timers/index.html)
