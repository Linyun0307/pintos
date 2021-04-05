# Report for Pintos

SID:11812131

Name: 李琳云

## Data structures and functions


### Data structures:

```c
int64_t ticks_blocked;/* The remaining ticks should be blocked. */

int base_priority;/* Base priority before donated. */

struct lock *lock_waiting;/* Lock blocking this thread. */
struct list locks_holding;/* Lcoks held by this thread. */

/* For mlfqs. */
int nice;
fixed_t recent_cpu;
```
``struct thread`` is the data structure to record the information of all information about a thread. 
The above varibles should be added to ``struct thread``.

``ticks_blocks`` is the remaining ticks that the thread should be blocked, which is used to design the alar clock.

``base_priority``, ``lock_waiting``, ``lock_holding`` are use to mantain the donate of priority.

``nice`` and ``recent_cpu`` are the nice value of the thread and the CPU time each process have received recently.
These variables are used to design multi-level feedback queue scheduler.

---

```c
int priority;
struct list_elem elem;
```
These two variables are added to ``struct lock`` to implement priority donation. 
``priority`` is the priority that the thread waiting for the lock can donate.
``elem`` is used to implement the list ``lock_holding`` in ``struct thread``.

---

```c
fixed_t load_avg;
```
This is the golable varriable added to thread.c. It is the estimated average number of threads ready to run over the past minute.

---


### Functions

The functions added in thread.c:
```c
void blocked_thread_check(struct thread *t, void *aux UNUSED);
```
This is the function used to check whether a blocked thread can be unblock in this time trick.

```c
bool thread_cmp_priority (struct list_elem *x, struct list_elem *y,void *aux UNUSED);
void thread_donate_priority(struct thread *t);
void thread_update_priority(struct thread *t);
```
``thread_cmp_priority`` is used to maintain a sorted list with the maximum priority thread in the front.

``thread_donate_priority`` is to calculate the priority that a thread could have after donation.

``thread_update_priority`` is to calculate the priority of thread from its lock and base priority.

```c
void mlfqs_increase_recent_cpu_by_one (void);
void mlfqs_update_load_avg_and_recent_cpu (void);
void mlfqs_update_priority (struct thread *t);
```
Functions that used in mlfqs.

---

The functions implemented in sysch.c:
```c
bool lock_cmp_priority (struct list_elem *x, struct list_elem *y,void *aux UNUSED);
bool cond_sema_cmp_priority (struct list_elem *x, struct list_elem *y,void *aux UNUSED);

```
Function to compare the priority in lock and the priority in conditional sema.

---
## Alogrithm


### Task1 Efficent Alarm clock
- block the thread when ``time_sleep`` is called:

```c
void timer_sleep (int64_t ticks) 
{
  if(ticks <= 0) return;
  ASSERT (intr_get_level () == INTR_ON);

  enum intr_level old_level = intr_disable();

  struct thread *current_thread = thread_current();
  current_thread -> ticks_blocked = ticks;
  thread_block();
  
  intr_set_level(old_level);
}
```
Record the time a thread should block and bolck it.


- Check every time silce and unblock the thread when it should wake up.
```c
static void timer_interrupt (struct intr_frame *args UNUSED)
{
  ticks++;
  thread_foreach(blocked_thread_check,NULL);
  thread_tick ();
}

void blocked_thread_check (struct thread *t, void *aux UNUSED){
  if(t->status == THREAD_BLOCKED && t->ticks_blocked > 0){
    t->ticks_blocked--;
    if(t->ticks_blocked == 0)
      thread_unblock(t);
  }
}
````

### Task2 Priority scheduler
- Chosing the next thread to run:

  Chose the thread in ready list with the highest priority.

- Acquiring a Lock

  First check whether the lock has a holder. Once the lock has holder, try to donate the thread's priority to the waiting lock, and the lock's priority to the thread holding the lock in the chain until the thread in the end of the chain.

  Then after the thread can have the lock, push the lock into its holding list and update the lock's and the thread's information.

- Releasing a Lock
  Update the lock's information of its holder.

  Update the thread's information. Remove the lock from the ``lock_holdong`` list and update the thread priority.

- Computing the effective priority

  The priority of a thread should came from its base priority or the max priority of all the list it holds.

  So, first get its base priority from ``base_priority``.
  Then get the max priority from all member in ``lock_holding`` and compare it to base priority.

- Priority scheduling for locks and semaphores.
  
  The locks held by a thread are record in a list ``locks_holding`` as a variable in ``struct thread``. Every time we push a lock into the back of a list. When we want to use the lock's priority to upadate the thread's, we use ``list_max ()`` (in list.c) and ``lock_cmp_priority()`` (in synch.c) to get the maximum prioprity of locks in the list. 

  In sema, we use a list ``waiters`` as the list of waiting threads. Each time if we want to ``sema_down`` check its value and while the sema's ``value`` equals to zero, put the thread into the back of list ``waiters``.
  Each time we call ``sema_up``, while the list ``waiters`` is not empty, we find the thread with maximun priority and unblock it.
  Thus the thread with highest priority of all can get the lock.

- Priority for condition variables

  We make list ``cond_waiters`` as a ordered list. Each time we call ``cond_signal``, sort the list and ``sema_up`` the thread that got the highest priority's semaphore.

  Implement ``cond_sema_cmp_priority`` to compare the priority of thread in list ``cond_waters``

- Changing thread's priority
  First change ``base_priority`` of the thread.
  If the new priority is larger than its tmp priority or the thread is not in a donating process, we'll change its ``priority``.
  Then do ``thread_yield`` to reshecdule the program.

---

### Task3 Muti-level Feedback Queue Scheduler

The mlfqs was implemented according to guidance. Use macro in thread/fixed_point.h to do floa
- Niceness

  Implent ``thread_get_nice(void)`` and ``thread_set_nice(int new_nice)`` to get or set a nice value of a thread
- Calculating and Updating Priority
```
priority = PRI_MAX - (recent_cpu / 4) - (nice * 2)
```
  Then check if it is out of change and update the correct priority to the thread.

- Calculate Recent CPU and Load Average
```
recent_cpu = (2 * load_avg) / (2 * load_avg + 1) * recent_cpu + nice
load_avg = (59 / 60) * recent_cpu + (1 / 60) * ready_threads
```
First calculate ready_threads, then get load_avg, finally get rencet_cpu.

---

## Synchronization

### List of all potential concurrent access rescource:
- lock: use a semaphore to prevent a lock be used by two thread at the same time.
- lists: ``all_list``, ``ready_list`` in threas.c. ``sema_waiters`` in sema: We disable the interrupt before changing the list and set the interruput back after the change.

---
### Rationale
In my code, 242 lines was added an 22 lines are deleted. Though the implement is a simple an direct way, it may cause more overhead as the list of ``cond_waiters`` have been sorted many times.

---
### Additoinal Questions
| timer ticks | R(A) | R(B) | R(C) | P(A) | P(B) | P(C) | thread to run |
| ----------- | ---- | ---- | ---- | ---- | ---- | ---- | ------------- |
| 0           | 0    | 0    | 0    | 63   | 61   | 59   | A             |
| 4           | 4    | 0    | 0    | 62   | 61   | 59   | A             |
| 8           | 8    | 0    | 0    | 61   | 61   | 59   | B             |
| 12          | 8    | 4    | 0    | 61   | 60   | 59   | A             |
| 16          | 12   | 4    | 0    | 60   | 60   | 59   | B             |
| 20          | 12   | 8    | 0    | 60   | 59   | 59   | A             |
| 24          | 16   | 8    | 0    | 59   | 59   | 59   | C             |
| 28          | 16   | 8    | 4    | 59   | 59   | 58   | B             |
| 32          | 16   | 12   | 4    | 59   | 58   | 58   | A             |
| 36          | 20   | 12   | 4    | 58   | 58   | 58   | C             |