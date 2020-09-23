# [CSED312] OS project 1 Design Report

- 20180673 하재현
- 20180501 최진수

## Analysis of the current thread system

## Analysis of the current synchronization

multi thread 환경을 구현할 때에는 thread간의 자원 공유에 대해서 매우 신경을 써야 합니다.

만약 공유 자원에 여러 thread가 동시에 접근하게 되면 race condition이 발생하여 원치 않은 결과를 만들어낼 수 있고, 때로는 시스템 전체를 붕괴시키기도 합니다.

이런 불상사를 막기 위해 Synchronization, 즉 스레드들이 수행되는 시점을 적절히 조절하는 것이 필요합니다.

### 1. Disabling Interrupts

### 2. Semaphores

**Semaphore** 란 아래 두 개의 연산자를 이용해서 atomically 수정 가능한 nonnegative integer을 의미한다.

- **"Down" or "P"**: 값이 양수가 될 때까지 기다리고, 감소시킴
- **"Up" or "V"** : 값을 증가시킴 (그리고 waiting thread가 있으면 그 중 하나를 실행시킴)

간단한 사용법은 다음과 같다.

thread는 critical section에 진입하기 전 down을 해서 semaphore 값을 0으로 만든다.

이후 이 critical section에 진입하는 다른 thread는 down을 하지만 양수가 아니므로 기다려야 한다.

첫 번째로 진입한 thread가 끝나면, up을 하면서 대기중인 thread를 깨운다.

깨워진 thread는 semaphore 값을 확인하여 양수라면 critical section에 진입한다.

만약 초기값을 1보다 크게 설정하면 그만큼 여러 스레드를 동시에 진입시킬 수 있다.

현재 Pintos에서는 이 semaphore가 synchronization의 핵심이다.

자세한 원리를 알아보기 위해, Semaphore의 구조체를 알아보자.

```c++
struct semaphore 
  {
    unsigned value;             /* Current value. */
    struct list waiters;        /* List of waiting threads. */
  };
```

Semaphore은 그 값과, waiting thread list가 있다.

```c++
void
sema_init (struct semaphore *sema, unsigned value) 
{
  ASSERT (sema != NULL);

  sema->value = value;
  list_init (&sema->waiters);
}
```

Semaphore의 초기화는 초기값의 설정과 list의 초기화로 이루어져있다.

연산자의 동작을 알아보자.

```c++
void
sema_down (struct semaphore *sema) 
{
  enum intr_level old_level;

  ASSERT (sema != NULL);
  ASSERT (!intr_context ());

  old_level = intr_disable ();
  while (sema->value == 0) 
    {
      list_push_back (&sema->waiters, &thread_current ()->elem);
      thread_block ();
    }
  sema->value--;
  intr_set_level (old_level);
}
```

down은 semaphore의 값을 확인하여 0이 아니라면 곧바로 값을 1 감소시키지만, 0이라면 waiting list에 넣고, thread를 block한다. 

나중에 block이 풀리면, semaphore의 값이 양수임을 확인하고, 값을 1 감소시킨다.

```c++
void
sema_up (struct semaphore *sema) 
{
  enum intr_level old_level;

  ASSERT (sema != NULL);

  old_level = intr_disable ();
  if (!list_empty (&sema->waiters)) 
    thread_unblock (list_entry (list_pop_front (&sema->waiters),
                                struct thread, elem));
  sema->value++;
  intr_set_level (old_level);
}
```

up은 priority에 상관없이, FIFO로 waiting thread를 선택하여 unblock 한 후 값을 1 증가시킨다.

### 3. Locks



### 4. Monitors

## Solutions

### 1. Alarm Clock

### 2. Priority scheduling

### 3. Advanced scheduler