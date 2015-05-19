# OS15x156_157_164
project1
            +--------------------+
            | PROJECT 1: THREADS |
            |   DESIGN DOCUMENT  |
            +--------------------+
                   
---- GROUP ----

>> Fill in the names and email addresses of your group members.

顾泉松 <———— >
周刚 <email@domain.example>
侯杰 <email@domain.example>

---- PRELIMINARIES ----

>> If you have any preliminary comments on your submission, notes for the
>> TAs, or extra credit, please give them here.

>> Please cite any offline or online sources you consulted while
>> preparing your submission, other than the Pintos documentation, course
>> text, lecture notes, and course staff.

                 ALARM CLOCK
                 ===========

---- DATA STRUCTURES ----

>> A1: Copy here the declaration of each new or changed `struct' or
>> `struct' member, global or static variable, `typedef', or
>> enumeration.  Identify the purpose of each in 25 words or less.

The members added in Struct Thread in thread.h:
int64_t ticks_blocked;   // To record how long the thread is slept

---- ALGORITHMS ----

>> A2: Briefly describe what happens in a call to timer_sleep(),
>> including the effects of the timer interrupt handler.

In original function timer_sleep(), there is a while loop to test whether it is time to run the thread ,which will make busy waiting happen. So I change the function timer_sleep() to block the thread at the beginning and add the member ticks_blocked in Struct Thread to record how long the thread is slept. Then using the timer interrupt of OS to observe the thread status.(each time do ticks_blocked-1 ).When ticks_blocked = 0  ,wake this thread(use the new Function blocked_thread_check ()).

>> A3: What steps are taken to minimize the amount of time spent in
>> the timer interrupt handler?

Delete a while loop to test whether it is time to run the thread.
And use the new Function blocked_thread_check () to wake the slept thread whose ticks_blocked = 0.

---- SYNCHRONIZATION ----

>> A4: How are race conditions avoided when multiple threads call
>> timer_sleep() simultaneously?

Blocking the thread and record how long the thread is slept.When the thread is waked, putting it into ready list. By this way can handle multiple timer sleep calls.

>> A5: How are race conditions avoided when a timer interrupt occurs
>> during a call to timer_sleep()?

When the timer interrupt occurs, all the blocked threads are checked for ticks_blocked times which is less than the current timer_ticks. this way we can make sure that only one thread are woken up in the timer interrupt this avoiding race condition.

---- RATIONALE ----

>> A6: Why did you choose this design? In what ways is it superior to
>> another design you considered?

Initially I tried to implement the threads of waiting list by comparing with their wake time. But since different threads has different wake time, this did not work out. I also tried to implement this with seperate array of wake times and comparing the wake time with the current timer ticks. This approach needs a seperate mechanism to point to their individual threads. Finally I use the way block threads at the beginning,this way solve the Problem busying waiting and avoid race conditions . And the code changed is more less then other ways.

             PRIORITY SCHEDULING
             ===================

---- DATA STRUCTURES ----

>> B1: Copy here the declaration of each new or changed `struct' or
>> `struct' member, global or static variable, `typedef', or
>> enumeration.  Identify the purpose of each in 25 words or less.

The members added in Struct Thread in thread.h:
int base_priority;            // Base priority
struct list locks;            // Locks that the thread is holding
struct lock *lock_waiting;    // The lock that the thread is waiting for

The members added in Struct Lock in synch.h:
struct list_elem elem;      //List element for priority donation. 
int max_priority;           //Max priority among the threads acquiring the lock

>> B2: Explain the data structure used to track priority donation.
>> Use ASCII art to diagram a nested donation. (Alternately, submit a
>> .png file.)

H		M
p:33		p:32
\		 /
	L
donated_priority:32
is_donated: true
second_donated_priority:33
is_second_donated:true

---- ALGORITHMS ----

>> B3: How do you ensure that the highest priority thread waiting for
>> a lock, semaphore, or condition variable wakes up first?

The current implementation of adding a thread to synchronisation waiting queue is by using list_push_back function which pushback the thread at the end of the list. So in that case the higher priority thread is not invoked first. So I changed my implementation of adding the elements to the list based on the priority. compare_by_priority and compare_by_sema_priority are the two functions which are invoked to determine the ordering of the list items. So everytime the first element in the list would be higher order. The current thread accessing the synchronisation primitive is checked with the front thread in the waiting queue of the list. If the current thread has lower priority than the front thread of the list, it is yielded immediately. By this way we can make sure that the highest priority waiting thread is woken up first.

>> B4: Describe the sequence of events when a call to lock_acquire()
>> causes a priority donation.  How is nested donation handled?

The first thread which calls the lock acquire gets hold on the lock and it is stored in lock->holder variable. Every thread's donated priority and second donated priority are initialised to the thread's priority at the time of creation itself(handled in inti_thread function).  When a higher priority thread(say M) calls the lock acquire, it first checks lock->holder donated priority is less than the M's priority. If so it sets the lock->holder's donated_priority to M's priority and also lock->holder's is_donated to true. When the second thread access lock acquire(say H) with a priority higher than both M and lock->holder's priority,then lock_acquire function checks whether the lock->holder has already donated or not and the H's priroity is higher that lock->holer's priority, if so then it sets the lock->holder's second_donated_priority to H's priority and also lock->holder's is_second_donated to true.

>> B5: Describe the sequence of events when lock_release() is called
>> on a lock that a higher-priority thread is waiting for.

When the lock_release function is called, first it checks whether the there are any waites for the lock->semaphore. then it checks whether it is is_second_donated, if so then lock->holder second_donated_priority is set to its donated priority. By this way we make sure that it lowers its priority priority to second priority. (this condition is made sure because the implementation is such a way that only after setting the is_donated, is_second_donated can be checked.). When the medium prioirty thread access the lock acquire then lock->holder is_donated is already set to true, so it sets the donated priority to its original priority.thread_get_priority function is handled such a way to support this implementation.

---- SYNCHRONIZATION ----

>> B6: Describe a potential race in thread_set_priority() and explain
>> how your implementation avoids it.  Can you use a lock to avoid
>> this race?

When a thread access the function thread_set_priority to set its priority to a new value, if first checks whether the new_priority is less than the priority of first thread. If so it yields. By this way, we make sure that the highest priority thread runs everytime we change the priority.

---- RATIONALE ----

>> B7: Why did you choose this design? In what ways is it superior to
>> another design you considered?

Initially I tried to recurse through all the list elements in order to get the maximum prioirty in the list. however everytime for everythread the while loop is executed, so it is of order O(N) to get the maximum priority element in the list.By this implementation, getting the maximum priority element from the list is of order O(1).
