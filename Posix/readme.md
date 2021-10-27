# Posix cheatsheet 
Welcome to the POSIX C cheatsheet!
## Topics you will encounter in this sheet
* <a href="#tc">Thread creation </a>
* <a href="#jt">Joining threads </a>
* <a href="#apt">Assigning priorities to threads </a>
* <a href="#asst">Assigning Scheduling strategies to threads </a>
* <a href="#me">Mutual Exclusion </a>
* <a href="#cme">Conditional Mutual Exclusion </a>
* <a href="#ip">Inversion of Priority </a>
* <a href="#pt"> Periodic tasks</a>
## libraries
All the functions, structs and constants exist within the pthread module.

## <p id="tc">Thread Creation</p>
``` int pthread_create(pthread_t *thread, pthread_attr_t *attr, void* (*start_routine)(void*), void* arg);```
 
 ### Explaining the parameters:
 * thread: pointer of type pthread_t
 * attr : variable of type pthread_attr_t. this is a structure containing configuration constants (representing a manifest of the thread) to determine specefic functionalities.Click <a href="#attr">here</a> to view explanation.
 * start_routine: function to be executed
 * arg: argument passed to the start_routine function, generally void*(equivalent to any in typescript).

## <p id="jt">Joining Threads</p>
If the use case requires your current function to wait on a thread to finish its execution. You can implement this by calling the pthread_join function with this signature:

``` int pthread_join(pthread_t *thread, void** thread_return) ```
  ### Explaining the paramters
  * thread: pointer of type pthread_t to the thread that's to be waited upon.
  * thread_return: return value of the waited upon thread upon finishing its execution.

> NB: void ** means pointer of any possible return type. ~ pointer to Any

## Example using the last 2 sections:
```
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <pthread.h>

void* fonc(void* arg){
int i;
for(i=0;i<7;i++){
printf("Tache %d : %d\n", (int) arg, i);
usleep(1000000); //attendre 1 seconde
}
}
int main(void)
{
pthread_t tache1, tache2; 
pthread_create(&tache1, NULL, fonc, (void*) 1); 
pthread_create(&tache2, NULL, fonc, (void*) 2);
pthread_join(tache1, NULL); 
pthread_join(tache2, NULL);
```

## <p id="attr">Detaching a thread from its creator</p>
The need for this approach rises from 1 main issue:
* The main process no longer needs to wait for its child, when the jobs of the main process and the child thread diverge, we can release the child thread by creating the thread differently.
This requires using <b>attr</b> parameter.

The Posix approach:

1. constructing the structure with the function ``` pthread_attr_init(&attr); ```
2. Setting the <b>detachstate</b> attribute with ``` pthread_attr_setdetachstate(&attr, PTHREAD_CREATE_DETACHED); ```
3. Creating the detachable thread with ``` pthread_create(&tache1, &attr, fonc, 1); ```
4. De-allocating the attr variable with ``` pthread_attr_destroy(&attr); ```

## <p id="apt">Assigning Priorities to threads</p>
When tasks require different priorities to maintain the program semantics, the <b>attr</b> variable can be used to achieve this.
This is done with the following function:

``` int pthread_attr_setschedparam(pthread_attr_t *attr, const struct sched_param *param) ```
>  To achieve this we need to define a variable of type sched_param which contains an int denoting the sched_priority.

## <p id="asst">Assigning scheduling strategies to threads</p>
There are 2 ways to achieve this:

1. Inherint the parent process scheduling strategy:
  ``` int pthread_attr_setinheritsched(pthread_attr_t *attr, PTHREAD_INHERIT_SCHED) ```

2. Explicity defining the scheduling strategy for the thread:
``` 
int pthread_attr_setinheritsched(pthread_attr_t *attr, PTHREAD_EXPLICIT_SCHED);
int pthread_attr_setschedpolicy(pthread_attr_t *attr, SCHED_RR | SCHED_OTHER | SCHED_FIFO)
```
### Explaining the policy constants
1. <b>SCHED_FIFO</b>: Premptive scheduling with fixed priorities. tasks with the same priorities are scheduling as First In First Out
2. <b>SCHED_RR</b>: Task consumes a constant amount of time on the CPU then gets appended to the Queue containing all other tasks having the same priority (Different Queue for every priority)
3. <b>SCHED_OTHER</b>: generally points to SCHED_FIFO

> It's possible to change the scheduling settings of a process during its execution by using the following function 

``` int pthread_setschedparam(pthread_t thread, int policy, const struct sched_param *param) ```

## Priority & Scheduling example:

``` 
#include <stdio.h>
#include <stdlib.h>
#include <pthread.h>

void* fonc(void* arg){
int i;
for(i=0;i<7;i++){
printf("Tache %d : %d\n", (int) arg, i);
usleep(1000000);
}
}
int main(void)
{
pthread_t tache1, tache2;
pthread_attr_t attr;
struct sched_param param;
pthread_attr_init(&attr);
param.sched_priority = 12;
pthread_setschedparam(pthread_self(), SCHED_FIFO, &param); // pthread_self() is a stat is a static method pointing to the thread executing the current context where it was called
pthread_attr_setinheritsched(&attr, PTHREAD_EXPLICIT_SCHED);

pthread_attr_setschedpolicy(&attr, SCHED_FIFO);
param.sched_priority = 10;
pthread_attr_setschedparam(&attr, &param);
pthread_create(&tache1, &attr, fonc, 1);
param.sched_priority = 7;
pthread_attr_setschedparam(&attr, &param);
pthread_create(&tache2, &attr, fonc, 2);
pthread_attr_destroy(&attr);
pthread_join(tache1, NULL);
pthread_join(tache2, NULL);
return 0;
```
## <p id="me">Mutual Exclusion</p>
I will not be explaining what Mutual exclusion is in this section. This section with showcase Posix implementation of Mut-Ex.
1. Declare ur mutex variable 
``` pthread_mutex_t key ```
2. Register your mutex with the init function, can be further configured with a variable of type <b>pthread_mutextattr_t</b>. In this course we'll use NULL for this to use the default configuration of the mutex. But the function looks as follows:
```  pthread_mutex_init(pthread_mutex_t *key, const pthread_mutextattr_t *m_attr) ```

To lock/unlock the mutex, we can use these 2 functions
```
pthread_mutex_lock(pthread_mutex_t *key);
pthread_mutex_unlock(pthread_mutex_t *key);
 ```
 ### Mutex Code example
Credits to [Amine Haj Ali](https://github.com/hajali-amine).

 ```
 #include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <pthread.h>

typedef struct
{
    float taille;
    float poids;
} type_donneePartagee;

pthread_mutex_t key;
type_donneePartagee donneePartagee;

void *tache1(void *arg)
{
    type_donneePartagee ma_donneePartagee;
    int i = 0;
    while (i < 10)
    {
        pthread_mutex_lock(&key);
        ma_donneePartagee = donneePartagee;
        pthread_mutex_unlock(&key);
        printf("La tache %s vient de lire la donnee partagee\n", (char *)arg);
        usleep(1000000);
        i++;
    }
}

void *tache2(void *arg)
{
    int i = 0;
    while (i < 10)
    {
        pthread_mutex_lock(&key);
        donneePartagee.taille = 100 + rand() % 101;
        donneePartagee.poids = 10 + rand() % 101;
        pthread_mutex_unlock(&key);
        printf("La tache %s vient de modifier la donnee partagee\n", (char *)arg);
        usleep(1000000);
        i++;
    }
}

int main(void)
{
    pthread_t th1, th2;
    pthread_mutex_init(&key, NULL);
    donneePartagee.taille = 100 + rand() % 101;
    donneePartagee.poids = 10 + rand() % 101;
    pthread_create(&th1, NULL, tache1, "1");
    pthread_create(&th2, NULL, tache2, "2");
    pthread_join(th1, NULL);
    pthread_join(th2, NULL);
    return 0;
}
```
## <p id="cme">Conditional Mutual Exclusion</p>
Refer to this [link](https://www.youtube.com/watch?v=DTHoiQreroE) for a good explanation.
You can see the problem that this section solves in the following manner:
Imagine we have a producer and a consumer sharing a Queue between them. If by some order of chance the consumer gets hold of the mutex when the Queue is empty, we fall into the pit of unefficient multi-threadding. So we need another lock alongside the mutex to determine wether it's okay for the consumer to consume.
* We start by initializing a conditional variable like this: 
```
pthread_cond_t cond = PTHREAD_COND_INITIALIZER; 
```
* To make a thread possessing the mutex key but unable to consume due to the condition mismatch go to sleep and let go of its mutex, we call the following method:
```
int pthread_cond_wait(pthread_cond_t *cond, pthread_mutex_t *key)
```
* To send a wake up signal to <b> ONE THREAD</b> who slept on this conditional variable, we use 
```
 int pthread_cond_signal(pthread_cond_t *cond) 
 ```
 * If you wish to wake up all the threads waiting on that conditional use this instead:
 ```
 int pthread_cond_broadcast(pthread_cond_t *cond)
 ```

 ## <p id="ip">Inversion of Priority</p>
Soon™
## <p id="pt">Periodic Tasks</p>
To create periodic tasks, one should follow these steps:
1. Delcare a time struct called timespec
```
struct timespec time;
```
2. To retrieve the time in the variable we declared, we call the following function:
```
int clock_gettime(clockid_t CLOCK_REALTIME | CLOCK_MONOTONIC |CLOCK_PROCESS_CPUTIME_ID | CLOCK_THREAD_CPUTIME_ID, struct timespec *time);
```
> Note that many of these clock_id variables do not exist on windows Hosts.

The Posix module takes the following approach:
Since the thread has a time condition to meet before waking up and executing its task, the implementation has to a conditional mutex key seen in the section above.
So to wake up a thread after a <b>time</b> amount of time, use the following :
```
int pthread_cond_timedwait(pthread_cond_t *cond, pthread_mutex_t *key, struct
timespec *time)
```
### Code example of Periodic tasks
```
#include <stdio.h>
#include <stdlib.h>
#include <pthread.h>
#include <time.h>
void* tachePeridique(void* periode){
pthread_cond_t cond;
pthread_mutex_t key;
struct timespec time;

pthread_cond_init(&cond, NULL);
pthread_mutex_init(&key, NULL);
int i=0;
clock_gettime(CLOCK_REALTIME, &time);
while(i<10){
pthread_mutex_lock(&key);
time.tv_sec = time.tv_sec + (int) periode;
printf("La tache %s s'execute periodiquement à l'instant %d secondes\n", "t1", (int)
time.tv_sec);
//suite du code
pthread_cond_timedwait(&cond, &key, &time);
pthread_mutex_unlock(&key);
i++;
}
}
int main(void)
{
pthread_t tache1;
pthread_create(&tache1, NULL, tachePeridique, (void*) 5); //la tache1 est périodique de
periode 5s
pthread_join(tache1, NULL);
return 0;
}
```

* Thank you for your attention!
* Please support [Mellywins](https://github.com/Mellywins).











