用户态定时器API
======================

nanosleep
-------------------

nanosleep基于hrtimer来实现纳秒级延时，内核中提供hrtimer_nanosleep接口，并且封装成系统调用nanosleep给用户空间使用。其核心是do_nanosleep，会将线程设置为TASK_INTERRUPTIBLE|TASK_FREEZABLE状态，然后调度出去，当定时时间到期后，定时器中断唤醒该task。从2.3时钟源精度52ns就可以看出，在加上这里会有任务调度，和代码执行耗时，真想实现ns级延时是不现实的，微秒级应该是可以的。

.. code-block:: c

  static int __sched do_nanosleep(struct hrtimer_sleeper *t, enum hrtimer_mode mode)
  {
      struct restart_block *restart;

      do {
          set_current_state(TASK_INTERRUPTIBLE|TASK_FREEZABLE);
          hrtimer_sleeper_start_expires(t, mode);

          if (likely(t->task))
              schedule();

          hrtimer_cancel(&t->timer);
          mode = HRTIMER_MODE_ABS;

      } while (t->task && !signal_pending(current));

      __set_current_state(TASK_RUNNING);
  }

itimer
-------------------

itimer基于hrtimer和信号配合实现，通过hrtimer设置高精度超时时间，到期后通过信号方式通知进程。itimer提供下面三种定时方式，收到的信号也不同。
定时类型	含义	到期信号
ITIMER_REAL	真实时间	SIGALRM
ITIMER_VIRTUAL	当前进程在用户态实际执行时间	SIGVTALRM
ITIMER_PROF	当前进程在用户态和内核态实际执行时间	SIGPROF

如下signal_struct结构体定义中包含的定时器相关的成员，与itimer相关的有，real_timer是该线程对应真实时间hrtimer定时器，it[2]是2种cpu执行时间定时的cpu_itimer定时器。

.. code-block:: c

  struct signal_struct {

  #ifdef CONFIG_POSIX_TIMERS

          /* POSIX.1b Interval Timers */
          int                        posix_timer_id;
          struct list_head        posix_timers;

          /* ITIMER_REAL timer for the process */
          struct hrtimer real_timer;
          ktime_t it_real_incr;

          /*
          * ITIMER_PROF and ITIMER_VIRTUAL timers for the process, we use
          * CPUCLOCK_PROF and CPUCLOCK_VIRT for indexing array as these
          * values are defined to 0 and 1 respectively
          */
          struct cpu_itimer it[2];

          /*
          * Thread group totals for process CPU timers.
          * See thread_group_cputimer(), et al, for details.
          */
          struct thread_group_cputimer cputimer;

  #endif
          /* Empty if CONFIG_POSIX_TIMERS=n */
          struct posix_cputimers posix_cputimers;

  } __randomize_layout;

ITIMER_REAL是真实时间定时，也是最常规的，在线程被fork出来时，会在copy_signal中初始化real_timer，设置超时函数it_real_fn。在超时函数中发送SIGALRM给当前线程所属的进程tgid，所以同一个进程内，同时只能有一个线程使用ITIMER_REAL。

.. code-block:: c

  static int copy_signal(unsigned long clone_flags, struct task_struct *tsk)
  {
  #ifdef CONFIG_POSIX_TIMERS
      INIT_LIST_HEAD(&sig->posix_timers);
      hrtimer_init(&sig->real_timer, CLOCK_MONOTONIC, HRTIMER_MODE_REL);
      sig->real_timer.function = it_real_fn;
  #endif
  }

  enum hrtimer_restart it_real_fn(struct hrtimer *timer)
  {
      struct signal_struct *sig =
          container_of(timer, struct signal_struct, real_timer);
      struct pid *leader_pid = sig->pids[PIDTYPE_TGID];

      trace_itimer_expire(ITIMER_REAL, leader_pid, 0);
      kill_pid_info(SIGALRM, SEND_SIG_PRIV, leader_pid);

      return HRTIMER_NORESTART;
  }

ITIMER_REAL使用示例

.. code-block:: c

  #include <unistd.h>
  #include <signal.h>
  #include <sys/time.h>
  #include <iostream>

  static int itimer_count = 0;
  struct itimerval timer_set;
  void sig_handler(int signo)
  {
      if(++itimer_count>=5){
          timer_set.it_value.tv_sec = 0;
          timer_set.it_value.tv_usec = 0;
          timer_set.it_interval.tv_sec = 0;
          timer_set.it_interval.tv_usec = 0;
          //5*2=10s后，将timer_set所有成员清0，然后调用setitimer停止定时器
          setitimer(ITIMER_REAL, &timer_set, NULL);
      }
      std::cout<<"recieve sigal: "<<signo<<std::endl;
  }

  int main()
  {
      signal(SIGALRM, sig_handler);

      //2s后启动
      timer_set.it_value.tv_sec = 2;
      timer_set.it_value.tv_usec = 0;

      //定时器间隔：2s
      timer_set.it_interval.tv_sec = 2;
      timer_set.it_interval.tv_usec = 0;

      //设置定时器
      if(setitimer(ITIMER_REAL, &timer_set, NULL) < 0)
      {
          std::cout<<"start timer failed..."<<std::endl;
          return 0;
      }

      int temp;
      std::cin>>temp;

      return 0;
  }

对于另外2种cpu运行时间相关定时器，设置接口也是setitimer，但是clock类型选择ITIMER_VIRTUAL或者ITIMER_PROF。cpu timer实现方式比较复杂，流程是在tick事件处理时，通过run_posix_cpu_timers来检查cpu_timer，并向到期的timer发送对应的信号。
.. code-block::

  run_posix_cpu_timers
  |-->struct task_struct *tsk = current
  |-->__run_posix_cpu_timers(tsk)
      |-->handle_posix_cpu_timers(tsk)
          |-->check_thread_timers(tsk, &firing);  //检查线程时间tsk->cpu_timers[N]
          |-->check_process_timers(tsk, &firing); //检查进程时间tsk->signal->cpu_timers[N]
              |-->collect_posix_cputimers(pct, samples, firing);
              |-->check_cpu_itimer(SIGPROF)   //检查用户态和内核态总时间
                  |-->send_signal_locked(SIGPROF, SEND_SIG_PRIV, tsk, PIDTYPE_TGID);//如果到期则发送SIGPROF信号
              |-->check_cpu_itimer(SIGVTALRM) //检查用户态时间
                  |-->send_signal_locked(SIGVTALRM, SEND_SIG_PRIV, tsk, PIDTYPE_TGID);//如果到期则发送SIGVTALRM信号

alarm
-------------------

alarm基于itimer实现来定时，并且以秒为单位，时间到内核会给该进程发送SIGALRM信号。

.. code-block:: c

  SYSCALL_DEFINE1(alarm, unsigned int, seconds)
  {
      return alarm_setitimer(seconds);
  }

  static unsigned int alarm_setitimer(unsigned int seconds)
  {
      struct itimerspec64 it_new, it_old;

  #if BITS_PER_LONG < 64
      if (seconds > INT_MAX)
          seconds = INT_MAX;
  #endif
      it_new.it_value.tv_sec = seconds;
      it_new.it_value.tv_nsec = 0;
      it_new.it_interval.tv_sec = it_new.it_interval.tv_nsec = 0;

      do_setitimer(ITIMER_REAL, &it_new, &it_old);

      /*
      * We can't return 0 if we have an alarm pending ...  And we'd
      * better return too much than too little anyway
      */
      if ((!it_old.it_value.tv_sec && it_old.it_value.tv_nsec) ||
            it_old.it_value.tv_nsec >= (NSEC_PER_SEC / 2))
          it_old.it_value.tv_sec++;

      return it_old.it_value.tv_sec;
  }

进程在调用alarm定时之前，需要设置SIGALRM信号的处理函数。参数很简单，只有一个秒，使用代码示例如下

.. code-block:: c

  #include <stdio.h>
  #include <unistd.h>
  #include <signal.h>

  void sig_handler(int signum) {
      printf("Received SIGALRM, timer expired!\n");
  }

  int main() {
      signal(SIGALRM, sig_handler);
      alarm(5);
      printf("Waiting for alarm...\n");
      pause(); // Suspend the process until a signal is received
      printf("Exiting...\n");
      return 0;
  }

posix Timer
-------------------

Posix timer大大扩展了itimer的功能，一个进程可以同时创建任意个timer，并且可以指定到期信号。Posix timer封装了多个syscall接口：

创建定时器：timer_create
删除定时器：timer_delete
设置定时器时间：timer_settime
获取定时器剩余：timer_gettime

通过which_clock参数来区分使用哪种时间类型来计时，下面是支持的时间类型。此外还提供了clock_gettime()、clock_settime()、clock_adjtime()、clock_getres()系统调用来获取和设置各种类型时间信息，属于大一统的接口。

.. code-block:: c

  static const struct k_clock * const posix_clocks[] = {
      [CLOCK_REALTIME]        = &clock_realtime,
      [CLOCK_MONOTONIC]       = &clock_monotonic,
      [CLOCK_PROCESS_CPUTIME_ID]  = &clock_process,
      [CLOCK_THREAD_CPUTIME_ID]   = &clock_thread,
      [CLOCK_MONOTONIC_RAW]       = &clock_monotonic_raw,
      [CLOCK_REALTIME_COARSE]     = &clock_realtime_coarse,
      [CLOCK_MONOTONIC_COARSE]    = &clock_monotonic_coarse,
      [CLOCK_BOOTTIME]        = &clock_boottime,
      [CLOCK_REALTIME_ALARM]      = &alarm_clock,
      [CLOCK_BOOTTIME_ALARM]      = &alarm_clock,
      [CLOCK_TAI]         = &clock_tai,
  };

timer_fd
-------------------

timer_fd是一个基于文件描述符的定时器接口，精度为纳秒级，直接基于hrtimer实现。提供三个接口函数，通过文件描述符的可读事件进行超时通知。通过timerfd_create在内核创建一个定时器实例，并返回一个文件描述符fd，通过timerfd_settime(fd)设置超时时间。由于不是基于信号通知，进程需要通过select、poll、epoll等io机制，监听fd的事件，来实现异步事件通知。

.. code-block:: c

  // 内核源码路径：fs/timerfd.c
  SYSCALL_DEFINE2(timerfd_create, int, clockid, int, flags)
  {
    int ufd;
    struct timerfd_ctx *ctx;

    /* Check the TFD_* constants for consistency.  */
    BUILD_BUG_ON(TFD_CLOEXEC != O_CLOEXEC);
    BUILD_BUG_ON(TFD_NONBLOCK != O_NONBLOCK);

    if ((flags & ~TFD_CREATE_FLAGS) ||
        (clockid != CLOCK_MONOTONIC &&
        clockid != CLOCK_REALTIME &&
        clockid != CLOCK_REALTIME_ALARM &&
        clockid != CLOCK_BOOTTIME &&
        clockid != CLOCK_BOOTTIME_ALARM))
      return -EINVAL;

    if ((clockid == CLOCK_REALTIME_ALARM ||
        clockid == CLOCK_BOOTTIME_ALARM) &&
        !capable(CAP_WAKE_ALARM))
      return -EPERM;

    ctx = kzalloc(sizeof(*ctx), GFP_KERNEL);
    if (!ctx)
      return -ENOMEM;

    init_waitqueue_head(&ctx->wqh);
    spin_lock_init(&ctx->cancel_lock);
    ctx->clockid = clockid;

    if (isalarm(ctx))
      alarm_init(&ctx->t.alarm,
          ctx->clockid == CLOCK_REALTIME_ALARM ?
          ALARM_REALTIME : ALARM_BOOTTIME,
          timerfd_alarmproc);
    else
      hrtimer_init(&ctx->t.tmr, clockid, HRTIMER_MODE_ABS);

    ctx->moffs = ktime_mono_to_real(0);

    ufd = anon_inode_getfd("[timerfd]", &timerfd_fops, ctx,
              O_RDWR | (flags & TFD_SHARED_FCNTL_FLAGS));
    if (ufd < 0)
      kfree(ctx);

    return ufd;
  }

  // 用户态接口
  #include <sys/timerfd.h>
  int timerfd_create(int clockid, int flags);
  int timerfd_settime(int fd, int flags, const struct itimerspec *new_value, struct itimerspec *old_value);
  int timerfd_gettime(int fd, struct itimerspec *curr_value);

以下是timer_fd的使用示例代码，每隔1s触发一次，每次触发都要重新设置，并且设置的时间都是绝对时间：

.. code-block:: c

  #include <stdio.h>
  #include <errno.h>
  #include <string.h>
  #include <utils/Timers.h>
  #include <sys/epoll.h>
  #include <sys/timerfd.h>
  static int mTimerFd, mEpollFd;

  void my_set_timer(int TimerFd, int64_t time)
  {
      struct itimerspec old_timer;
      struct itimerspec new_timer {
          .it_interval = {.tv_sec = 0, .tv_nsec = 0},
          .it_value = {.tv_sec = (long)(time / 1000000000),
                      .tv_nsec = (long)(time % 1000000000)},
      };

      if(timerfd_settime(TimerFd, TFD_TIMER_ABSTIME, &new_timer, &old_timer))
              printf("Failed to set timerfd %s (%i)", strerror(errno), errno);
  }

  void timer_loop(int mTimerFd, int mEpollFd){
      epoll_event timerEvent;
      timerEvent.events = EPOLLIN;
      if (epoll_ctl(mEpollFd, EPOLL_CTL_ADD, mTimerFd, &timerEvent) == -1) {
          printf("Error adding timer fd to epoll dispatch loop");
          return;
      }

      while (true) {
          epoll_event events;
          uint64_t mIgnored = 0;
          epoll_wait(mEpollFd, &events, 2, -1);
          read(mTimerFd, &mIgnored, sizeof(mIgnored));

          my_set_timer(mTimerFd, systemTime(SYSTEM_TIME_MONOTONIC) + 1000*1000*1000);
          printf("timer_fd triggerd, do something\n");
      }
  }

  int main(void)
  {
      int mTimerFd, mEpollFd;
      mTimerFd = timerfd_create(CLOCK_MONOTONIC, TFD_CLOEXEC | TFD_NONBLOCK);
      mEpollFd = epoll_create1(EPOLL_CLOEXEC);
      printf("create mTimerFd=%d, mEpollFd=%d\n", mTimerFd, mEpollFd);

      my_set_timer(mTimerFd, systemTime(SYSTEM_TIME_MONOTONIC) + 1000*1000*1000); //1s

      timer_loop(mTimerFd, mEpollFd);
      return 0;
  }