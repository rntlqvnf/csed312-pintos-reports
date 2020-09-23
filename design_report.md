# [CSED312] OS project 1 Design Report

- 20180673 하재현
- 20180501 최진수

## Analysis of the current thread system

------

### Thread structure

- 구조체 thread는 threads/thread.h에 아래 코드와 같이 정의되어 있다. 멤버를 하나하나 살펴보자면, tid는 thread id를 나타내고, status는 thread의 state를 나타낸다. name은 debugging을 목적으로 thread의 이름을 저장하고 있고, stack은 현재 thread의 stack pointer을 의미한다. priority는 아래에서 더 구체적으로 다루겠지만 thread의 우선 순위를 나타낸다. 
- elem member은 중의적인 역할을 띠고 있는데, 하나는 run queue에서의 element로서의 역할, 다른 하나는 semaphore wati list에서의 element로서의 역할이다. elem이 이런 두 가지의 역할을 할 수 있는 것은 그 두 역할이 mutually exclusive하기 때문이다. 만약 thread가 ready state에 있다면 그것은 run queue에 있을 것이고, blocked state에 있다면 semaphore wait list에 있을 것이다.

```c
struct thread
  {
    /* Owned by thread.c. */
    tid_t tid;                          /* Thread identifier. */
    enum thread_status status;          /* Thread state. */
    char name[16];                      /* Name (for debugging purposes). */
    uint8_t *stack;                     /* Saved stack pointer. */
    int priority;                       /* Priority. */
    struct list_elem allelem;           /* List element for all threads list. */

    /* Shared between thread.c and synch.c. */
    struct list_elem elem;   
```

- 모든 thread structure은 각자 4kB page를 차지한다. 위의 코드에서 보이는 구조체 내 원소들은(structure 자체는) page offset 0에 저장되고, 나머지 page는 offset 4kB(top of the page)로부터 시작해서 아래쪽으로 자라는, thread의 kernel stack에 위치하게 된다. 



### initializing threading system

- ``` void thread_init(void)``` 함수는 threading system을 initializing하는 역할을 한다. (첫 번째 kernel thread를 형성하는 역할을 한다)
- thread를 initializing하는 방법은 현재 돌아가고 있는 code를 thread로 transforming하면서이다.
- ``` void thread_init(void)``` 함수는 run queue와 tid lock을 initializing하며, 이 함수를 호출한 이후에는 아래에서 다룰 ``` thread_create() ``` 함수를 이용해 thread를 생성하기 전에 page allocator를 initialize해야만 한다.

```c
void thread_init (void) 
{
  ASSERT (intr_get_level () == INTR_OFF);

  lock_init (&tid_lock);
  list_init (&ready_list);
  list_init (&all_list);

  /* Set up a thread structure for the running thread. */
  initial_thread = running_thread ();
  init_thread (initial_thread, "main", PRI_DEFAULT);
  initial_thread->status = THREAD_RUNNING;
  initial_thread->tid = allocate_tid ();
}
```



### Thread creation

- ```thread_create()```는 새로운 kernel thread를 생성하는 함수이다. 주요 인자로 name(이름), priority(우선 순위),  수행할 function pointer을 넘겨준다. return value는 creation이 성공적으로 완료되면 thread identifier, creation이 완료되지 못했을 때에는 TID_ERROR이다.
- thread가 creating된다는 것은 scheduling이 이루어질 새로운 context를 creating한다는 의미이다. 처음 thread가 scheduled되고 돌아가기 시작하면 인자로 넘겨지는 function으로부터 시작해서 그 context에서 execute된다.
- 만약 인자로 넘겨진 function이 return 한다면 ```thread_create()``` 함수도 terminate하게 된다. 
- Pintos에서 돌아가는 모든 thread는 pintos 내에서 돌아가는 일종의 mini program으로 볼 수 있다.
- ```thread_create()``` 함수의 동작 방법은 다음과 같다. 일단 ```struct thread```를 위한 공간을 할당하고, ```init_thread()```를 이용해 그 멤버를 초기화한다. context switching이 일어날 수 있도록 3개의 stack frame(kernel thread, switch entry, switch thread를 위한)을 할당한다. 생성된 thread는 blocked state로 초기화되는데, return 전에 unblocked 상태로 만들어 새로운 thread가 schedule 될 수 있게 해준다.

```c
tid_t thread_create (const char *name, int priority,
               thread_func *function, void *aux) 
{
  struct thread *t;
  struct kernel_thread_frame *kf;
  struct switch_entry_frame *ef;
  struct switch_threads_frame *sf;
  tid_t tid;

  ASSERT (function != NULL);

  /* Allocate thread. */
  t = palloc_get_page (PAL_ZERO);
  if (t == NULL)
    return TID_ERROR;

  /* Initialize thread. */
  init_thread (t, name, priority);
  tid = t->tid = allocate_tid ();

  /* Stack frame for kernel_thread(). */
  kf = alloc_frame (t, sizeof *kf);
  kf->eip = NULL;
  kf->function = function;
  kf->aux = aux;

  /* Stack frame for switch_entry(). */
  ef = alloc_frame (t, sizeof *ef);
  ef->eip = (void (*) (void)) kernel_thread;

  /* Stack frame for switch_threads(). */
  sf = alloc_frame (t, sizeof *sf);
  sf->eip = switch_entry;
  sf->ebp = 0;

  /* Add to run queue. */
  thread_unblock (t);

  return tid;
}
```



### Thread scheduler

- 프로그램이 시작할 때 ```thread_start()``` 함수에 의해 scheduler가 시작된다. ```thread_start()```는 idle thread를 생성하며, 이것은 다른 어떤 thread도 준비되지 않았을 때 schedule되는 thread이다. 그리고 interrupt가 enable되고, 이것은 scheduler를 enable하는 결과를 낳는데 이것은 scheduler가 timer interrupt으로부터의 return에 ```intr_yield_on_return()``` 함수를 이용해 돌아가기 때문이다. 

- 임의의 시간에 돌아가는 thread는 항상 단 하나여야 한다. 돌아가는 하나의 thread를 제외한 나머지 thread는 inactivate된 상태를 유지해야만 한다. 그리고 다음으로 돌아갈 thread를 결정해주는 것이 thread scheduler이다. 만약 어느 thread도 돌아갈 준비가 되지 않았다면 idle thread가 돌아가게 된다. 즉, thread 사이의 switch가 일어나게 하는 것이 scheduler의 역할이 된다.

- ```thread_tick()``` 함수는 모든 timer tick마다 timer interrupt마다 호출되는데, thread statistics를 tracking하고 time slice가 만료되었을 때 scheduler을 trigger하는 역할을 한다.

- ```into_yield_on_return()``` 함수는 interrupt context로, ```thread_yield()``` 함수를 interrupt return 직전에 호출한다. 이것은 Timer interrupt handler에서 thread의 time slice가 만료되었을 때 새로운 thread가 호출되도록 하는 역할을 한다.

  

### Thread completion

- ```thread_exit()``` 함수의 호출에 의해 일어난다. thread가 task 수행을 완료할 때 ```thread_exit()``` 함수의 호출을 통해 thread state를 THREAD_DYING으로 설정하고 thread switching을 일어나게 한다. 그리고 page를 free하게 된다.



## Analysis of the current synchronization

## Solutions

### 1. Alarm Clock

### 2. Priority scheduling

### 3. Advanced scheduler