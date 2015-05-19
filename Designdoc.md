    +--------------------+
            |       CS 326       |
            | PROJECT 1: THREADS |
            |   DESIGN DOCUMENT  |
            +--------------------+
                   
---- GROUP ----

>> Fill in the names and email addresses of your group members.

顾泉松 <email@domain.example>
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

  ADVANCED SCHEDULER               ==================  ---- DATA STRUCTURES ----  >> C1: Copy here the declaration of each new or changed `struct' or >> `struct' member, global or static variable, `typedef', or >> enumeration.  Identify the purpose of each in 25 words or less. 
struct thread 
int nice;                        //stores the nice value of each thread 
int recent_cpu;                 //stores the CPU time of each thread.

int load_avg;	          //Used to store the system load_avg.
extern struct list ready_list;
extern struct list all_list;
	//Changed the implementation to extern because it has to be used in the timer.c to refresh the priorities adn the recent CPU of all the threads in the lists

void refresh_load_avg(void);  //function to update the load average 
int refresh_priority(struct thread* a);  //refreshing the priority of the thread 'a' to a new value and returns it.
int refresh_priority_cur(void);  //refreshing the priority of the current thread and returns it.
int refresh_recent_cpu(struct thread* t);    //Refeshes the recent CPU of thread t and returns it.
---- ALGORITHMS ----  >> C2: Suppose threads A, B, and C have nice values 0, 1, and 2.  Each >> has a recent_cpu value of 0.  Fill in the table below showing the >> scheduling decision and the priority and recent_cpu values for each >> thread after each given number of timer ticks:timer  recent_cpu    priority   thread
ticks   A   B   C   A   B   C   to run
-----  --  --  --  --  --  --   ------
 0	0   0   0  63  61  59   A
 4	4   0   0  62  61  59	A
 8	8   0   0  61  61  59   B
12	8   4   0  61  60  59   A
16     12   4   0  60  60  59   B
20     12   8   0  60  59  59   A
24     16   8   0  59  59  59   C
28     16   8   4  59  59  58   B
32     16  12   4  59  58  58   A
36     20  12   4  58  58  58   C
>> C3: Did any ambiguities in the scheduler specification make values >> in the table uncertain?  If so, what rule did you use to resolve >> them?  Does this match the behavior of your scheduler?
Assume that timer interrupt occured right after thread A . By this way the recent CPU of A gets increased and priority of this thread is recalculated. Then the process goes on to all threads.
>> C4: How is the way you divided the cost of scheduling between code >> inside and outside interrupt context likely to affect performance?
---- RATIONALE ----  >> C5: Briefly critique your design, pointing out advantages and >> disadvantages in your design choices.  If you were to have extra >> time to work on this part of the project, how might you choose to >> refine or improve your design?

I read line by line in the document and coded the functions then and there. However I did make small mistake somewhere, which makes my mlfqs-load-60 to start to zero. If i had given extra time i would have defined an abtraction layer for all the operations that i implemented with this method. I would have also changed the implementation of the  refresh_recent_cpu,refresh_priority function to a better way.
>> C6: The assignment explains arithmetic for fixed-point math in >> detail, but it leaves it open to you to implement it.  Why did you >> decide to implement it the way you did?  If you created an >> abstraction layer for fixed-point math, that is, an abstract data >> type and/or a set of functions or macros to manipulate fixed-point >> numbers, why did you do so?  If not, why not?

Initally i thought of using the fixed point math as it is given by the doc. Like to convert a number a number into floating point i just multiply it by 2^14. Latter i realised that it may lead to many errors when we manually perform every operation then and there. So i declared all the operation in macro as given in the document. It saves lot of space which is a must for kernel level programming.

